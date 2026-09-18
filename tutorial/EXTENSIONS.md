# Extensions: why `safety-net` is designed this way

The RCA tutorial treated a netlist as a graph you could query and rewrite. This extension looks one layer lower that that, looking into why `safety-net` uses reference-counted graph objects and a type parameter such as `Netlist<I>`, and what does that buy a pass author?

The short answer is that netlist editing combines shared graph relationships with representation specific cell data. `safety-net` keeps the connectivity machinery reusable while making unsafe graph changes harder to perform accidentally.

## 1. Reference counting as a safeguard

A netlist is not a tree. A cell output may feed several inputs, a pass may retain a handle to a cell while traversing, and analyses may hold more handles of their own. Careless deletion can produce:

> object deleted\
> → another part of the graph still refers to it\
> → invalid reference

[`NetRef<I>`](https://matth2k.github.io/safety-net/safety_net/struct.NetRef.html) is a cloneable, reference-counted handle. Destructive operations inspect those live references. In particular, [`Netlist::clean`](https://matth2k.github.io/safety-net/safety_net/struct.Netlist.html#method.clean) checks that removing an unused object will not strand a live handle and returns [`Error::DanglingReference`](https://matth2k.github.io/safety-net/safety_net/enum.Error.html#variant.DanglingReference) before mutating the graph if it would.

The extension patch contains a complete small program, including setup and a minimal custom cell type. Apply it from `tutorial/`:

```bash
make extensions-patch
cargo run --quiet --package safety-pass --example extensions
```

The important part of the first demo is only this sequence:

> `let held_reference = candidate.clone();`\
> `let blocked = netlist.clean().unwrap_err();`\
> `drop(candidate);`\
> `drop(held_reference);`\
> `let removed = netlist.clean().unwrap();`

Expected output begins with:

> `live reference: Attempted to create a dangling reference to nets [Net { ... }]`\
> `released references: removed 1 object`

This is an actual guard in the current API and not just a benefit of Rust ownership in the abstract. The first `clean` sees that the unused cell still has handles outside the netlist and refuses to remove it. Once those handles are released, the same operation succeeds. In a larger pass, that turns a potentially stale reference into a local reported error before graph mutation begins.

## 2. One graph API, different cell representations

`safety-net`'s core graph algorithms are not hard-coded to one exact cell representation. A [`Netlist<I>`](https://matth2k.github.io/safety-net/safety_net/struct.Netlist.html) operates over an instantiable type that exposes the capabilities required by the API through the [`Instantiable`](https://matth2k.github.io/safety-net/safety_net/trait.Instantiable.html) trait.

The patch hides the 70 or so lines that define `MiniCell`, declare its ports, and implement `Instantiable`. The conceptual payload is much smaller:

> `let netlist: Rc<Netlist<MiniCell>> = Netlist::new("custom_cells".into());`\
> `... insert two MiniCell values and connect them ...`\
> `let pass = CellStats::<MiniCell>(PhantomData);`\
> `println!("{}", pass.run(&netlist).unwrap());`

The final output includes:

> `MINI_BUF: 2`\
> `Total: 2`

This uses the existing generic [`CellStats<I>` pass](../safety-pass/src/passes.rs#L168-L203), unchanged, on a representation that is neither the tutorial's [`Cell`](../safety-pass/src/cells.rs#L359-L366) nor `safety-net`'s built-in `Gate`. The same `Netlist` insertion, connection, traversal, output, and pass APIs work because `MiniCell` supplies the `Instantiable` contract.

When finished remove only the injected example:

```bash
make extensions-clean
```

### Why the type parameter matters

1. **A pass needs less representation knowledge.** [`CellStats<I>`](../safety-pass/src/passes.rs#L168-L203) asks the graph for objects and asks each instance for its name; it does not know how `MiniCell` stores that name or what other fields a production cell contains. Restricting what a pass knows restricts what it can accidentally depend on or break. Representation-specific passes can still choose a concrete type, as the RCA tutorial's `MyPass` does.

2. **Capabilities can grow through traits.** The graph abstraction does not have to absorb every cell-specific feature. This repository already adds a separate [`Primitive`](../safety-pass/src/cells.rs#L765-L816) trait for primitive type and sizing queries. A future representation could use the same pattern for a truth table or clock-domain capability; those are examples of extension points, not features claimed by the current API. Passes that need such a capability can require it, while graph-only passes remain generic.

3. **The graph machinery is reused.** Traversal, connectivity, guarded cleanup, replacement, and verification live on the generic netlist instead of being rebuilt for every representation.

> hard-coded approach: representation A traversal | representation B traversal\
> `safety-net`: representation A → `Netlist<I>` common graph API ← representation B

The library demonstrates this reuse beyond the patch: generic passes such as [`Clean<I>`](../safety-pass/src/passes.rs#L114-L139), [`RenameNets<I>`](../safety-pass/src/passes.rs#L142-L165), and [`RemapCells<I>`](../safety-pass/src/passes.rs#L562-L611) all operate on any compatible `I`.

## 3. What the RCA tutorial was already using

The RCA pass used [`Netlist::matches`](https://matth2k.github.io/safety-net/safety_net/struct.Netlist.html#method.matches), [`NetRef::find_input`](https://matth2k.github.io/safety-net/safety_net/struct.NetRef.html#method.find_input), [`InputPort::get_driver`](https://matth2k.github.io/safety-net/safety_net/struct.InputPort.html#method.get_driver), and [`InputPort::connect`](https://matth2k.github.io/safety-net/safety_net/struct.InputPort.html#method.connect). It did not manipulate graph pointers or rewrite Verilog text.

That is the design payoff. Pass code can describe a transformation in netlist terms while reusable typed infrastructure owns more of the connectivity, representation adaptation, and mutation checks. As the tool grows, new cell representations can share that infrastructure instead of bringing a new graph API with them.
