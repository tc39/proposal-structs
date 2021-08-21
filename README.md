# JavaScript Structs: Fixed Layout Objects

Stage: 0

Author: Shu-yu Guo (@syg) and others TBD

Champion: Shu-yu Guo (@syg) and others TBD

## Introduction

Structs are declarative sealed objects. There are two kinds of structs: plain structs and shared structs. Plain structs behave as if they were sealed objects. Shared structs have additional restrictions and can be concurrently accessed from different agents.

All structs have the following properties:
- Opaque storage like plain objects. Not aliasable via `ArrayBuffer` or `SharedArrayBuffer`.
- Sealed instances. The engine must be able to fix a layout that is unchanging.
- Immutable prototype chain. Prototypes themselves must be sealed, frozen, or `null`.

Shared structs have the following additional properties:
- Only data fields are allowed. Getters, setters, and methods are disallowed.
- Shared structs only reference other primitives or shared structs. One consequence is that the prototype chain must be transitively shared structs or `null`.
- Shared structs default to `null` prototype.
- Shared structs do not have a `.constructor` property.

This proposal is intended to be minimal.

Structs can be designed with or without novel syntax. For brevity of presentation, examples are given using `class` syntax with a `struct` or `shared struct` qualifier.

A minimal plain struct example.

```javascript
struct class Box {
  constructor(x) { this.x = x; }
  x;
}

let box = new Box();
box.x = 42;  // x is declared
assertThrows(() => { box.y = 8.8; });       // structs are sealed
assertThrows(() => { box.__proto__ = {} }); // structs are sealed
```

A minimal shared struct example.

```javascript
// main.js
shared struct class SharedBox {
  constructor(x) { this.x = x; }
  x;
}

let sharedBox = new SharedBox();
let sharedBox2 = new SharedBox();

sharedBox.x = 42;          // x is declared and rhs is primitive
sharedBox.x = sharedBox2;  // x is declared and rhs is shared
assertThrows(() => { sharedBox.x = {}; }) // rhs is not a shared struct

let worker = new Worker('worker.js');
worker.postMessage({ sharedBox });

sharedBox.x = "main";      // x is declared and rhs is primitive
console.log(sharedBox.x);
```

```javascript
// worker.js
onmessage = function(e) {
  let sharedBox = e.data.sharedBox;
  sharedBox.x = "worker";  // x is declared and rhs is primitive
  console.log(sharedBox.x);
};
```

The above program is permitted to print out any interleaving:
- main main
- main worker
- worker worker
- worker main

## Motivation and requirements

### Shared memory for concurrency

This proposal seeks to enable to enable more shared memory concurrency for a more concurrent future. Like other shared memory features in JavaScript, it is high in expressive power and high in difficulty to use correctly. This proposal is both intended as an incremental step towards higher-level, easier-to-use (e.g. data-race free by construction) concurrency abstractions as well as an escape hatch for expert programmers who need the expressivity.

The two design principles that this proposal follows are:
1. Syntax that looks atomic ought to be atomic. (For example, the dot operator on shared structs should only access an existing field and does not tear.)
1. There are no references from shared objects to non-shared objects.

### WasmGC interoperability

The [WasmGC proposal](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md) adds fixed layout, garbage-collected objects to Wasm. While the details of the type system of these objects are yet to be nailed down, interoperability with JavaScript will be important.

WasmGC objects have opaque storage and are not aliased by linear memory, so they cannot be exposed as all Wasm memory is exposed today via `ArrayBuffer`s. We propose structs to be the reflection of WasmGC objects in JS.

WasmGC objects exposed to JS should behave the same as structs, modulo extra type checking that WasmGC require that JS structs do not. JS structs is also good foundation for reflecting into Wasm as WasmGC objects, but that is currently left as future work as it may need a typed field extensions to be worthwhile.

Further, WasmGC itself will eventually have multithreading. It behooves us to maintain a single memory model between JavaScript and Wasm as we have today, even with higher-level object abstractions.

### Predictable performance

Objects that are declared with a fixed layout help engines to have more predictable performance. A fixed layout object also lays groundwork for future refinement, such as typed fields.

## Possibly in-scope

The following are not currently in scope to maintain the proposal's minimalism. However, they may be brought into the scope of the proposal if they are deemed requirements for ergonomics.

### Arrays

For structured data, ergonomic ways to declare inline fixed-length fields seems desirable. Additionally, shared arrays also seem similarly desirable.

### Code sharing

Code sharing is not part of this proposal and thus shared structs cannot have methods. This may prove to be unergonomic enough that we bring code sharing in scope. Doing so would likely require a new kind of function cannot close over non-shared objects.

## Out-of-scope

### Value semantics, immutability, and operator overloading

This proposal does not intend to explore the space of objects with value semantics, including immutability and operator overloading. Structs have identity like other objects and are designed to be used like other objects. Value semantics is a sufficient departure that it may be better solved with other proposals that focus on that space.

### Sophisticated type systems

This proposal does not intend to explore sophisticated type and runtime guard systems. It is even more minimal than the closest spiritual ancestor, the [Typed Objects proposal](https://github.com/tschneidereit/proposal-typed-objects/blob/main/explainer.md), in that we do not propose integral types for sized fields. (Typed and sized fields are reserved for future work.)

### Binary data overlay views

This proposal does not intend to explore the space of overlaying structured views on binary data in an `ArrayBuffer`. This is a requirement arising from the desire for WasmGC integration, and WasmGC objects are similarly opaque.

Structured overlays are fundamentally about aliasing memory, which we feel is both a different problem domain and sufficiently solvable today in userland. For example, see [buffer-backed objects](https://github.com/GoogleChromeLabs/buffer-backed-object).

## Proposal

TODO

### Extending Atomics

By default, field accesses on shared structs are unordered. The various Atomics methods will be extended to accept shared structs as a first argument and a field name as the second argument to support sequentially consistent accesses.

## Implementation guidance

### Immutable shapes

Structs are declared with fixed layout up front. Engines should make an immutable shape for such objects. Optimizers can optimize field accesses without worrying about deopts.

### Shared structs: make sure fields are pointer-width and aligned

Shared structs should store fields such that underlying architectures can perform atomic stores and loads. This usually means the fields should be at least pointer-width and aligned.

### Strings will be difficult

Except for strings, sharing primitives in the engine is usually trivial, especially for NaN-boxing implementations.

Strings in production engines have in-place mutation to transition representation in order to optimize for different use cases (e.g. ropes, slices, canonicalized, etc). Sharing strings will likely be the most challenging part of the implementation.

It is possible to support sharing strings by copying-on-sharing, but may be too slow. If possible, lockfree implementations of in-place mutations above is ideal.

## Future work

### Typed fields

In the future, it may be sensible for more efficient memory representation ("packing") to also declare the type and size of fields. It is omitted from this proposal in that it is not a requirement for any of the primary motivations. At the same time, starting without types lets us add them incrementally in the future.
