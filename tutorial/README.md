# safety-pass Beginner Tutorial

Tutorial walking through inspecting, repairing, transforming, and testing a circuit using safety-pass

## Prerequisites

- **Git** - download the repository
- **Rust and Cargo** - compile and run safety-pass
- **Graphviz** - turn a DOT graph into an image
- **Verilator** - simulate & test the Verilog circuit

Installation commands (macOS, Homebrew):

```bash
brew install graphviz
brew install verilator
```

Verify everything is available:

```bash
git --version
cargo --version
dot -V
verilator --version
```

### Common setup errors

If you see:

```text
dot: command not found
```

install Graphviz. The Rust `Broken pipe` message is only a consequence of the missing `dot` command.

If you see:

```text
make: verilator: No such file or directory
```

install Verilator and run the command again.

If a program is installed but your terminal still cannot find it, check your `PATH` environment variable. `PATH` is the list of directories your shell searches when you type a command.

```bash
echo $PATH
command -v dot
command -v verilator
```

If `command -v` prints nothing, the executable is not currently visible through your `PATH`. If you installed the program with Homebrew, you can also run:

```bash
brew --prefix
```

and verify that Homebrew's `bin` directory is included in your `PATH`.

## 1. Download the tutorial, create a work branch

Clone the repository:

```bash
git clone https://github.com/matth2k/safety-pass.git
```

Enter the repository:

```bash
cd safety-pass
```

The entire repo is required because the tutorial files depend on `safety-net`, `safety-pass`, and `nl_opt` code elsewhere in the project.

Before making any changes, create a separate branch:

```bash
git switch -c tutorial-pass-work
```

Then enter the tutorial directory:

```bash
cd tutorial
```

## 2. Visualize the broken circuit

Generate an image of the ripple-carry adder:

```bash
make rca.png
```

Open the image on macOS:

```bash
open rca.png
```

This command:

1. Parses `rca.v`
2. Converts the Verilog into a safety-net netlist
3. Runs the existing `dot-graph` pass
4. Uses Graphviz to create `rca.png`

The first three full adders form a carry chain:

```text
fa_0 --carry[0]--> fa_1 --carry[1]--> fa_2
```

However, `fa_3` is not connected to that chain. There is no wire from `carry[2]` entering the `CI` port of `fa_3`.

The complete chain should instead be:

```text
fa_0 --carry[0]--> fa_1 --carry[1]--> fa_2 --carry[2]--> fa_3
```


## 3. Test the broken circuit 

Run the provided Verilator test before fixing anything:

```bash
make test
```

The first failing case should be:

```text
ERROR: 1 + 7 != 0
```

Why does `1 + 7` expose the bug?

```text
1 + 7 = 8 = 1000₂
```

Producing the `1` in the highest output position requires a carry to propagate from `fa_2` into `fa_3`. Because that connection is missing, the final full adder never receives the carry.

Earlier input pairs do not require that particular carry connection, which is why this is the first failing case.

## 4. Repair the broken circuit

Open `rca.v` and find the `fa_3` full adder. Its carry-input connection is commented out:

```verilog
// .CI(carry[2]),
```

Remove the two slashes:

```verilog
.CI(carry[2]),
```

This connects the carry output from `fa_2` to the carry input of `fa_3`.

Regenerate the image:

```bash
make -B rca.png
```

The `-B` forces `make` to rebuild the image. Open it again:

```bash
open rca.png
```

The graph should now show `carry[2]` connecting `fa_2` to the `CI` port of `fa_3`

## 5. Test the repaired circuit

Run a provided Verilator test:

```bash
make test
```

A four-bit input can represent the numbers 0 through 15; the test tries every pair of inputs giving:

```text
16 × 16 = 256 test cases
```

Every line should say `OK` and the final line should be:

```text
OK: 15 + 15 = 30
```

At this point, we know the original circuit works correctly. This gives us a baseline. If the circuit stops working after our transformation, the transformation introduced the problem.


## 6. Apply the starter pass

Before implementing the transformation, it helps to understand what a **compiler pass** is.

A compiler generally represents its input using an internal data structure called an **intermediate representation**, or IR. Here, the IR is a `safety-net` **netlist**: a graph containing hardware cells and the wires connecting them.

A **compiler pass** performs one operation over that representation. A pass might:

- inspect cells and connections
- collect information about the circuit
- modify the circuit
- produce another representation such as a graph or Verilog

`safety-pass` contains these analyses and transformations, and `nl_opt` allows passes to be run from the command line.

For this tutorial, you will implement a pass called `MyPass`.

Rather than manually writing all the boilerplate required to add a new pass, the tutorial provides:

```text
pass_template.patch
```

This patch contains the **starter code needed to add and register a new compiler pass with `nl_opt`**. The actual transformation is intentionally left unfinished for you to implement.

Apply it with:

```bash
make patch
```

This applies `pass_template.patch` which adds a new pass named `MyPass` to:

```text
safety-pass/src/passes.rs
```

Only run `make patch` once. Running it again will fail because the changes have already been applied.

The important unfinished portion is:

```rust
for cell in netlist.matches(|p| p.get_type() == CellType::FA) {
    todo!("Do something with this full adder cell! Swap A and B?")
}
```

> Find every cell in the netlist whose type is `FA` then perform some operation on it.

The patch also registers `MyPass` with the command-line program. This allows it to be selected using

```bash
-p my-pass
```

## 7. Implement the transformation

Your goal is to modify every full adder so that its `A` and `B` inputs are swapped.

Before:

```text
a[i] -> A
b[i] -> B
```

After:

```text
b[i] -> A
a[i] -> B
```

This should not change the behavior of the circuit because the full-adder logic treats `A` and `B` symmetrically.

Implement the transformation inside `MyPass`.

Some useful methods are:

```rust
find_input(...)
```

Find an input port belonging to a cell.

```rust
get_driver()
```

Find the net currently driving that input.

```rust
connect(...)
```

Connect an input to a different driver.

A useful approach is:

1. Find the `A` input
2. Find the `B` input
3. Save both original drivers
4. Connect `A` to the original `B` driver
5. Connect `B` to the original `A` driver
6. Count how many full adders were modified

It is important to save **both original drivers before changing either connection**.

Check and run the implementation:

```bash
cargo check
cargo run --release --quiet -- rca.v -p my-pass
```

Should report:

```text
Swapped A and B inputs on 4 full adders
```

A reference solution is provided separately in:

```text
pass_solution.patch
```

It is intentionally separate from `pass_template.patch` so that the solution itself is not part of the normal `safety-pass` source code.

The solution patch assumes that `pass_template.patch` has already been applied and that `MyPass` is still in its original starter state.

If you want to apply the reference solution immediately after `make patch`, run:

```bash
make solution
```


## 8. Verify the structural change

Run `MyPass` followed by `dot-graph` in the same pipeline:

```bash
cargo run --release --quiet -- rca.v \
    -p my-pass \
    -p dot-graph |
    dot -Tpng > rca_swapped.png
```

`MyPass` first modifies the in memory netlist and `dot-graph` then visualizes the modified version

In `rca_swapped.png` verify:

- Each `b[i]` wire now connects to port `A`
- Each `a[i]` wire now connects to port `B`
- The carry chain remains unchanged

## 9. Verify that behavior is preserved

Emit the transformed netlist as Verilog:

```bash
cargo run --release --quiet -- rca.v \
    -p my-pass \
    -p print-verilog \
    > rca_swapped.v
```

Compile the transformed circuit in a separate Verilator build directory:

```bash
verilator --cc --exe --build \
    -Wno-PINMISSING \
    --top-module rca \
    --Mdir obj_dir_swapped \
    rca_main.cpp cells.v rca_swapped.v
```

`cells.v` is included because `rca_swapped.v` contains the transformed `rca` module but still relies on the separate `FA` module definition

Run the transformed simulation:

```bash
./obj_dir_swapped/Vrca
```

All 256 cases should pass.

## Summary of the workflow

Congratulations! Here's what you accomplished:

1. Compile Verilog into a safety-net netlist
2. Find cells of a specific type
3. Inspect and modify their connections
4. Visualize the transformed structure
5. Emit the transformed netlist as Verilog
6. Verify that the transformation preserves behavior
