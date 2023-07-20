# JavaScript Structs: Fixed Layout Objects and Some Synchronization Primitives

Stage: 1

Author: Shu-yu Guo (@syg)

Champion: Shu-yu Guo (@syg), Ron Buckton (@rbuckton), Asumu Takikawa (@takikawa), Keith Miller (@kmiller68)

## Introduction

### Structs

Structs are declarative sealed objects. There are two kinds of structs: plain structs and shared structs. Plain structs behave as if they were sealed objects. Shared structs have additional restrictions and can be concurrently accessed from different agents.

All structs have the following properties:
- Opaque storage like plain objects. Not aliasable via `ArrayBuffer` or `SharedArrayBuffer`.
- Instances have all fields initialized in one shot, then sealed. The engine must be able to fix a layout that is unchanging. This implies that all superclasses must also be structs.
- Transitively immutable [[Prototype]] slot. A struct instance's [[Prototype]] slot is immutable, as are the [[Prototype]] slot of every object on its prototype chain.

Shared structs have the following additional properties:
- Only data fields are allowed. Getters, setters, and methods are disallowed.
- All superclasses must be shared structs.
- Shared structs only reference other primitives or shared objects.
- Shared structs have a null [[Prototype]].
- Shared structs do not have a `.constructor` property.
- Shared struct constructors have `[Symbol.hasInstance]` to support `instanceof`.

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

### Shared fixed-length arrays

Shared fixed-length arrays are the closed counterpart to Arrays, as structs are the closed counterpart to ordinary JS objects.

Shared fixed-length arrays are always shared. Structured data sharing requires _some_ primitive notion of collections, on top of which more sophisticated collections can be built.

While there is nothing in principle preventing addition of a non-shared fixed-length array, the use case is unclear. Where sharing across agents is not needed, ordinary Array instances are more flexible and already performant.

Shared fixed-length arrays have the following property:
- Length is required at construction time.
- Instances cannot be resized.
- Elements can only be other primitives other shared objects.
- Shared arrays have a null [[Prototype]].
- Shared arrays do not have a `.constructor` property.
- The shared array constructor has `[Symbol.hasInstance]` to support `instanceof`.

Shared fixed-length arrays do not need novel syntax. For brevity of presentation, examples are given using the `SharedFixedArray` constructor.

```javascript
// main.js
let sharedArray = new SharedFixedArray(10);
assert(sharedArray.length === 10);

let worker = new Worker('worker.js');
worker.postMessage({ sharedArray });

sharedArray[0] = "main";
console.log(sharedArray[0]);
```

```javascript
// worker.js
onmessage = function(e) {
  let sharedArray = e.data.sharedArray;
  sharedArray[0] = "worker";
  console.log(sharedArray[0]);
};
```

Just like the struct example, above program is permitted to print out any interleaving:
- main main
- main worker
- worker worker
- worker main

### Synchronization primitives

Non-recursive mutexes and conditional variables are well-understood synchronization primitives. Structured data sharing are well served by these higher-level synchronization primitives beyond `Atomics.wait` and `Atomics.notify`.

Pending future work on method and code sharing, mutexes and conditional variables are currently proposed as opaque, prototypeless shared objects to be used with static methods.

Minimal examples below.

```javascript
// Creates a new mutex.
let mutex = new Atomics.Mutex;

// This would block the current agent if the lock is held
// by another thread. This cannot be used on the agents
// whose [[CanBlock]] is false.
Atomics.Mutex.lock(mutex, function runsUnderLock() {
  // Do critical section stuff
});

// tryLock is like lock but returns true if the lock was acquired
// without blocking, and false is the lock is held by another
// thread.
Atomics.Mutex.tryLock(mutex, function runsUnderLock() {
  // Do critical section stuff
});
```

```javascript
let cv = new Atomics.Condition;

Atomics.Mutex.lock(mutex, () => {
  // This blocks the current agent, and cannot be used on the agents
  // whose [[CanBlock]] is false. The passed in mutex must be locked.
  Atomics.Condition.wait(cv, mutex);
});

// Waiters can notified with a count. A count of undefined or
// +Infinity means "all waiters".
let count = 1;
let numWaitersWokenUp = Atomics.Condition.notify(cv, count);
```

#### Extending `Atomics.Mutex` to support `using`

The [Explicit Resource Management](https://github.com/tc39/proposal-explicit-resource-management) proposal adds `using` declarations that perform lexically scoped resource management. The `Atomics.Mutex` API can be extended to better support `using`. For example,

- `Atomics.Mutex.lock(mutex)` can be overloaded to lock `mutex`
- `Atomics.Mutex.unlock(mutex)` can be added
- The `[Symbol.dispose]` own property can be added to all `Atomics.Mutex` instances

#### Asynchronous locking and waiting

See [ASYNC-LOCKING-WAITING.md](ASYNC-LOCKING-WAITING.md) for `lockAsync` and `waitAsync`.

## Motivation and requirements

### Shared memory for parallelism

This proposal seeks to enable more shared memory parallelism for a more parallel future. Like other shared memory features in JavaScript, it is high in expressive power and high in difficulty to use correctly. This proposal is both intended as an incremental step towards higher-level, easier-to-use (e.g. data-race free by construction) parallelism abstractions as well as an escape hatch for expert programmers who need the expressivity.

The two design principles that this proposal follows are:
1. Syntax that looks atomic ought to be atomic. (For example, the dot operator on shared structs should only access an existing field and does not tear.)
1. There are no references from shared objects to non-shared objects. The shared and non-shared heaps are conceptually separate, with references only going one way.

### WasmGC interoperability

The [WasmGC proposal](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md) adds fixed layout, garbage-collected objects to Wasm. While the details of the type system of these objects are yet to be nailed down, interoperability with JavaScript will be important.

WasmGC objects have opaque storage and are not aliased by linear memory, so they cannot be exposed as all Wasm memory is exposed today via `ArrayBuffer`s. We propose structs to be the reflection of WasmGC objects in JS.

WasmGC objects exposed to JS should behave the same as structs, modulo extra type checking that WasmGC require that JS structs do not. JS structs is also a good foundation for reflecting into Wasm as WasmGC objects, but that is currently left as future work as it may need a typed field extensions to be worthwhile.

Further, WasmGC itself will eventually have multithreading. It behooves us to maintain a single memory model between JavaScript and Wasm as we have today, even with higher-level object abstractions.

### Predictable instance performance

Objects that are declared with a fixed layout help engines to have more predictable performance. A fixed layout object also lays groundwork for future refinement, such as typed fields.

## Out-of-scope

### Value semantics, immutability, and operator overloading

This proposal does not intend to explore the space of objects with value semantics, including immutability and operator overloading. Structs have identity like other objects and are designed to be used like other objects. Value semantics is a sufficient departure that it may be better solved with other proposals that focus on that space.

### Sophisticated type systems

This proposal does not intend to explore sophisticated type and runtime guard systems. It is even more minimal than the closest spiritual ancestor, the [Typed Objects proposal](https://github.com/tschneidereit/proposal-typed-objects/blob/main/explainer.md), in that we do not propose integral types for sized fields. (Typed and sized fields are reserved for future work.)

### Binary data overlay views

This proposal does not intend to explore the space of overlaying structured views on binary data in an `ArrayBuffer`. This is a requirement arising from the desire for WasmGC integration, and WasmGC objects are similarly opaque.

Structured overlays are fundamentally about aliasing memory, which we feel is both a different problem domain and sufficiently solvable today in userland. For example, see [buffer-backed objects](https://github.com/GoogleChromeLabs/buffer-backed-object).

## Proposal

This proposal can be developed with or without novel syntax. It is presented with novel syntax below.

### Plain structs

Plain structs are declared with the contextual `struct` keyword in front of a class declaration.

`struct class` expressions parse the same as plain `class` expressions.

During evaluation of a `struct class` expression, the following checks are performed.
- If there is an `extends` clause and the superclass is not a `struct class`, throw a `TypeError`

When a `struct class` constructor is invoked, it creates instances with the following properties:
- All instance fields, including those from any superclasses, are defined and initialized to `undefined` before the newly constructed instance escapes to the constructor function (aka "one-shot")
- Instances are sealed
- Instances' [[Prototype]] slot is immutable after initialization

### Shared structs

Shared structs are declared with the contextual `shared struct` keywords in front of a class declaration. Intuitively, they behave as very restricted structs.

`shared struct class` expressions throw an early error if the following forms are encountered.
- method
- getter
- setter
- field initializers

During evaluation of a `struct class` expression, the following checks are performed.
- If there is an `extends` clause and the superclass is not a `shared struct class`, throw a `TypeError`

When a `struct class` constructor is invoked, it creates instances with the following properties:
- All instance fields are defined and initialized to `undefined` before the newly constructed instance escapes to the constructor function (aka "one-shot")
- Constructors have `[Symbol.hasInstance]` to support `instanceof`
- Instances are sealed
- Instances' [[Prototype]] slot contains null
- Instances do not have a `.constructor` property
- Instances are shared with instead of copied to other agents
- Instances' identities are preserved when communicated between agents
- When a value is assigned to an instance field, if it is neither a primitive nor a shared object or is a `Symbol`, throw a `TypeError`
- Instances' field accesses are unordered shared memory accesses and must not tear (i.e. another agent must not observe a partially written value)

The following `Atomics` methods will be extended to accept shared struct instances as the first argument and a field name as the second argument, to support sequentially consistent accesses.
- `Atomics.load`
- `Atomics.store`
- `Atomics.exchange`
- `Atomics.compareExchange`

Note that the arithmetic `Atomics` methods like `Atomics.add` are not included because there isn't widespread ISA support for atomic read-modify-write of floating point values. The workaround is to compute locally and store with `Atomics.compareExchange`.

### Shared fixed-length arrays

Shared fixed-length arrays are constructed using the `SharedFixedArray(len)` constructor. They are considered shared objects.
- Constructors have `[Symbol.hasInstance]` to support `instanceof`
- Instances are sealed
- Instances' [[Prototype]] slot contains null
- Instances do not have a `.constructor` property
- Instances have an immutable `.length` property
- Instances are shared with instead of copied to other agents
- Instances' identities are preserved when communicated between agents
- When a value is assigned to an instance field, if it is neither a primitive nor a shared object or is a `Symbol`, throw a `TypeError`
- Instances' element accesses are unordered shared memory accesses and must not tear (i.e. another agent must not observe a partially written value)

The following `Atomics` methods will be extended to accept `SharedFixedArray` instances as the first argument and an index as the second argument, to support sequentially consistent accesses.
- `Atomics.load`
- `Atomics.store`
- `Atomics.exchange`
- `Atomics.compareExchange`

Note that the arithmetic `Atomics` methods like `Atomics.add` are not included because there isn't widespread ISA support for atomic read-modify-write of floating point values. The workaround is to compute locally and store with `Atomics.compareExchange`.

### `Atomics.Mutex`

Non-recursive mutexes are constructed using the `Atomics.Mutex` constructor. They are considered shared objects.
- `Atomics.Mutex` has a `[Symbol.hasInstance]` to support `instanceof`
- Instances are sealed
- Instances have no properties
- Instances are shared with instead of copied to other agents
- Instances' identities are preserved when communicated between agents

The following static methods exist on `Atomics.Mutex`
- `Atomics.Mutex.lock(mutex, funToRunUnderLock)`: Acquire `mutex` by blocking. Once acquired, invoke `funToRunUnderLock`, then release the lock.
- `Atomics.Mutex.lockAsync(mutex, funToRunUnderLock)`: Acquire `mutex` asynchronously. Once acquired, enqueue `funToRunUnderLock` as a task to run. Release the lock after the enqueued task is finished.
- `Atomics.Mutex.tryLock(mutex, funToRunUnderLock)`: Try to acquire `mutex`, returning `undefined` if already locked. Once acquired, invoke `funToRunUnderLock`, then release the lock.

### `Atomics.Condition`

Condition variables are constructed using the `Atomics.Condition` constructor. They are considered shared objects.
- `Atomics.Condition` has a `[Symbol.hasInstance]` to support `instanceof`
- Instances are sealed
- Instances have no properties
- Instances are shared with instead of copied to other agents
- Instances' identities are preserved when communicated between agents

The following static methods exist on `Atomics.Condition`
- `Atomics.Condition.wait(cv, mutex, [ timeout ])`: Block the agent until `cv` is notified or until `timeout` milliseconds have passed. `mutex` must be locked. It is atomically released when the agent blocks and is reacquired after notification. `timeout` defaults to `Infinity`. Throw a `TypeError` of the agent's [[CanBlock]] is false.
- `Atomics.Condition.waitAsync(cv, mutex, [ timeout ])`: Returns a promise that is fulfilled when `cv` is notified or until `timeout` milliseconds have passed. `mutex` must be locked. It is released after the promise is constructed.
- `Atomics.Condition.notify(cv, count)`: Notify `count` number of `cv`'s waiters. A `count` of `Infinity` notifies all waiters.

## Implementation guidance

### Immutable shapes

Structs are declared with fixed layout up front. Engines should make an immutable shape for such objects. Optimizers can optimize field accesses without worrying about deopts.

### Shared structs: make sure fields are pointer-width and aligned

Shared structs should store fields such that underlying architectures can perform atomic stores and loads. This usually means the fields should be at least pointer-width and aligned.

### Shared structs: strings will be difficult

Except for strings, sharing primitives in the engine is usually trivial, especially for NaN-boxing implementations.

Strings in production engines have in-place mutation to transition representation in order to optimize for different use cases (e.g. ropes, slices, canonicalized, etc). Sharing strings will likely be the most challenging part of the implementation.

It is possible to support sharing strings by copying-on-sharing, but may be too slow. If possible, lockfree implementations of in-place mutations above is ideal.

### Synchronization primitives: they must be moving GC-safe

Production engines use moving garbage collectors, such as generational collectors and compacting collectors. If JS synchronization primitives are implemented under the hood as OS-level synchronization primitives, those primitives most likely depend on an unchanging address in memory and are _not_ moving GC-safe.

Engines can choose to pin these objects and make them immovable.

Engines can also choose to implement synchronization primitives entirely in userspace. For example, WebKit's [`ParkingLot`](https://webkit.org/blog/6161/locking-in-webkit/) is a userspace implementation of Linux futexes. This may have other benefits, such as improved and tuneable performance.

## Future work

### Code sharing

Code sharing is not part of this proposal and thus shared structs cannot have methods. This may prove to be unergonomic enough that we bring code sharing in scope. Doing so would likely require a new kind of function cannot close over non-shared objects.

This proposal future-proofs by having shared struct instances throw when touching the [[Prototype]] slot. This will be relaxed with a future proposal when code sharing becomes possible.

See [CODE-SHARING-IDEAS.md](CODE-SHARING-IDEAS.md) for a collection of ideas.

### Typed fields

In the future, it may be sensible for more efficient memory representation ("packing") to also declare the type and size of fields. It is omitted from this proposal in that it is not a requirement for none of the primary motivations. At the same time, starting without types lets us add them incrementally in the future.

### Shared structs: fast cloning

Since structs have fixed layout with an immutable [[Prototype]] slot and no accessors, they are amenable to fast cloning.
