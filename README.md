# JavaScript Structs: Fixed Layout Objects and Some Synchronization Primitives

Stage: 2

Author: Shu-yu Guo (@syg)

Champion: Shu-yu Guo (@syg), Ron Buckton (@rbuckton)

## Introduction

This proposal introduces four (4) logical features:

- [**Structs**](#structs), or unshared structs, which are fixed-layout objects. They behave like `class` instances, with more restrictions that are beneficial for optimizations and analysis.
- [**Shared Structs**](#shared-structs), which are further restricted structs that can be shared and accessed in parallel by multiple agents. They enable shared memory multithreading.
- [**Mutex and Condition**](#mutex-and-condition), which are higher level abstractions for synchronizing access to shared memory.
- [**Unsafe Blocks**](#unsafe-blocks), which are syntactic guardrails to lexically scope where racy accesses to shared structs are allowed.

This proposal is intended to be minimal, but still useful by itself, without follow-up proposals.

The motivations are:
- Enable new, high-performance applications to be written in JavaScript and on the web by unlocking shared memory multithreading.
- Give developers an alternative to `class`es that favors a higher performance ceiling and statically analyzability over flexbility.

Like other shared memory features in JavaScript, it is high in expressive power and high in difficulty to use correctly. This proposal is both intended as an incremental step towards higher-level, easier-to-use (e.g. data-race free by construction) parallelism abstractions as well as an escape hatch for expert programmers who need the expressivity.

The two design principles that this proposal follows for shared memory are:
1. Syntax that looks atomic ought to be atomic. (For example, the dot operator on shared structs should only access an existing field and does not tear.)
1. There are no references from shared objects to non-shared objects. The shared and non-shared heaps are conceptually separate, with direct references only going one way.

### Structs

Unshared structs are a refinement on JavaScript `class`es. They are declarative, like classes, but layout of struct instances are fixed.

Structs have the following properties:
- They are created with [integrity level sealed](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/seal). In other words, they have a fixed layout. Specifically, they cannot be extended with new properties. The value in their [[Prototype]] cannot change. Every syntactically declared field is writable, enumerable, and non-configurable.
- They have "one-shot initialization". Once a struct instance is accessible to JavaScript code, all their fields, including those of their superclasses, are already initialized to `undefined`.
- Struct `constructor` methods have a usable `this` value, which is the one-shot initialize instance, on entry. As a result, return override is not expressible. `super()` is still allowed, but not required.
- They can only extend other structs.
- The struct constructor itself is also sealed.
- Struct methods are non-generic. Their `this` value must be an instance of the struct or of a subclass.

Struct declarations use the `struct` keyword, and share their syntax with `class` declarations.

Some examples:

```javascript
struct Box {
  constructor(x) { this.x = x; }
  x;
}

let box = new Box(0);
box.x = 42;  // x is declared
assertThrows(() => { box.y = 8.8; });        // structs are sealed
assertThrows(() => { box.__proto__ = {}; }); // structs are sealed
```

```javascript
struct Point extends Box { // allowed because Box is also a struct
  constructor(x, y) {
    this.y = y;  // the this value is immediately usable
    super(x);    // calls the super constructor
    return {};   // the return value is discarded, no return override
  }

  distance(other) {
    return Math.sqrt((other.x - this.x) ** 2 +
                     (other.y - this.y) ** 2);
  }

  y;
}

let p = new Point(1, 2);
let fake = { x: 4, y: 5 };
// methods are non-generic
assertThrows(() => Point.prototype.distance.call(fake, p));
p.distance(fake); // allowed, the receiver is a Point
```

### Shared Structs

Shared structs are structs with even more restricted behavior, so as to make them amenable to be shared between different agents. Their fields can be accessed in parallel by multiple agents.

Shared structs, in addition to the properties listed for structs above, have the following additional properties:
- They can only extend other shared structs.
- They have a `null` prototype.
- They cannot contain instance methods or instance private names.
- Their instances can be communicated to other agents without copying.
- Their fields can only reference primitives or other shared structs. That is, they cannot point to unshared values.
- They cannot be frozen, because that would change their shape, which must be immutable to be amenable for sharing.

```javascript
// main.js
shared struct SharedBox {
  x;
}

let sharedBox = new SharedBox();
let sharedBox2 = new SharedBox();

unsafe {
  sharedBox.x = 42;          // x is declared and rhs is primitive
  sharedBox.x = sharedBox2;  // x is declared and rhs is shared
  assertThrows(() => { sharedBox.x = {}; }) // rhs is not a shared struct
}

// can programmatically test if a value can be shared
assert(Reflect.canBeShared(sharedBox2));
assert(!Reflect.canBeShared({}));

let worker = new Worker('worker.js');
worker.postMessage({ sharedBox });

unsafe {
  sharedBox.x = "main";      // x is declared and rhs is primitive
  console.log(sharedBox.x);
}
```

```javascript
// worker.js
onmessage = function(e) {
  let sharedBox = e.data.sharedBox;
  unsafe {
    sharedBox.x = "worker";  // x is declared and rhs is primitive
    console.log(sharedBox.x);
  }
};
```

The above program is permitted to print out any interleaving:
- main main
- main worker
- worker worker
- worker main

#### Shared Arrays

Shared Arrays are a fixed-length arrays that may be shared across agents. They are a special case of shared structs. There is no special syntax for Shared Arrays. They have a read-only `length` property.

```javascript
let sharedArray = new SharedArray(100);
unsafe {
  assert(sharedArray.length === 100);
  for (i = 0; i < sharedArray.length; i++) {
    // like shared structs, all elements are initialized to undefined
    assert(sharedArray[i] === undefined);
  }
}

let worker = new Worker('worker_array.js');
worker.postMessage({ sharedArray });

unsafe {
  sharedArray[0] = "main";
  console.log(sharedArray[0]);
}
```

```javascript
// worker_array.js
onmessage = function(e) {
  let sharedArray = e.data.sharedArray;
  unsafe {
    sharedArray[0] = "worker";
    console.log(sharedArray[0]);
  }
};
```

Like the shared struct example, the above program is also permitted to print out any interleaving:
- main main
- main worker
- worker worker
- worker main

#### Memory Model

By default, field accesses on shared structs are unordered. Sequentially consistent accesses are performed via the following new overloads on existing `Atomics` static methods.

The following pseudocode describes the new overloads.

```javascript
class Atomics {
  // ... existing stuff

  // Performs a seq-cst load on struct[fieldName].
  static load(struct, fieldName);

  // Performs a seq-cst store of value in struct[fieldName].
  static store(struct, fieldName, value);

  // Performs a seq-cst exchange on struct[fieldName] with newValue.
  //
  // newValue must be a primitive or a shared struct.
  //
  // Returns the old value.
  static exchange(struct, fieldName, newValue);

  // Performs a seq-cst compare-exchange on struct[fieldName].
  //
  // If the existing value in struct[fieldName] is expected, replace it with
  // replacement.
  //
  // replacement must be a primitive or a shared struct.
  //
  // Returns the value in struct[fieldName] before the replacement,
  // regardless of whether the value was exchanged.
  static compareExchange(struct, fieldName, expected, replacement);
}
```

Shared struct field accesses can never tear, regardless of the memory order. That is, a read of a shared struct field sees exactly one write of a shared struct field, never a mix of multiple writes.

### Mutex and Condition

Higher-level synchronization primitives are needed to aid in writing threadsafe code. This proposal adds `Atomics.Mutex` and `Atomics.Condition`.

#### Atomics.Mutex

`Atomics.Mutex` is a non-recursive mutex. The mutex itself is a shared struct with no fields. It is used via static methods on `Atomics.Mutex`. (If [per-Realm prototypes](#attaching-methods-to-shared-structs) become part of this proposal, those static methods will become prototype methods.)

The following pseudocode describes the API.

```javascript
shared struct Atomics.Mutex {
  // Creates a new mutex.
  constructor();

  // Attempt to acquire the lock on mutex.
  //
  // If the mutex is already locked, this blocks the agent until the lock is
  // acquired.
  //
  // If unlockToken is not undefined, it must be an empty UnlockToken.
  //
  // If unlockToken is undefined, returns a new UnlockToken.
  // Otherwise return the unlockToken that was passed in.
  static lock(mutex: Mutex,
              unlockToken: UnlockToken|undefined = undefined)
             : UnlockedToken;

  // Attempt to acquire the lock on mutex.
  //
  // If timeout is not Infinity, only block for timeout milliseconds. If
  // the operation timed out without acquiring the lock, returns
  // null.
  //
  // If timeout is 0, returns immediately without blocking if the lock
  // cannot be acquired.
  //
  // unlockToken behaves as it does in the lock method.
  static lockIfAvailable(mutex: Mutex,
                         timeout: Number,
                         unlockToken: UnlockToken|undefined = undefined)
                        : UnlockToken|null;
}
```

A mutex can only be unlocked via `UnlockToken`s, which are unlock capabilities. These tokens are ordinary objects, not shared structs. For high-performance applications, an application can allocate an empty `UnlockToken` and reuse it. This API is inspired by Rust and aims to minimize misuse (e.g. double unlocks).

```javascript
class Atomics.Mutex.UnlockToken {
  // Creates an empty UnlockToken.
  constructor();

  // Returns true if this token is non-empty.
  get locked(): bool;

  // If this token is non-empty, unlock the underlying mutex and returns true.
  // Otherwise returns false.
  unlock(): bool;

  // Like unlock, but returns undefined instead of bool.
  [Symbol.dispose](): undefined;
}
```

For example,

```javascript
shared struct MicrosoftSharePoint {
  x;
  y;
  mutex;
}

let point = new MicrosoftSharePoint();
point.mutex = new Atomics.Mutex();

let worker = new Worker('worker_mutex.js');
worker.postMessage({ point });

// assume this agent can block
unsafe {
  using lock = Atomics.Mutex.lock(point.mutex);
  point.x = "main";
  point.y = "main";
}

unsafe {
  using lock = Atomics.Mutex.lock(point.mutex);
  console.log(point.x, point.y);
}
```

```javascript
// worker_mutex.js
onmessage = function(e) {
  let point = e.data.point;
  unsafe {
    using lock = Atomics.Mutex.lock(point.mutex);
    point.x = "worker";
    point.y = "worker";
  }
};
```

The above program prints one of the following:
- main main
- worker worker

That is, because point the `x` and `y` fields are accessed under lock, no agent can observe a mix of main and worker values.

#### Atomics.Condition

`Atomics.Condition` is a condition variable. It is a shared struct with no fields. It is used via static methods on `Atomics.Condition`. (If [per-Realm prototypes](#attaching-methods-to-shared-structs) become part of this proposal, those static methods will become prototype methods.)

The following pseudocode describes the API.

```javascript
shared struct Atomics.Condition {
  // Creates a new condition variable.
  constructor();

  // Atomically unlocks the unlockToken and blocks the current agent until cv
  // is notified.
  //
  // unlockToken must not be empty.
  //
  // When the agent is resumed, the lock underlying mutex in unlockToken is
  // reacquired, blocking if necessary.
  //
  // Returns undefined.
  static wait(cv: Condition,
              unlockToken: UnlockToken): undefined

  // Atomically unlocks the unlockToken and blocks the current agent until cv
  // is notified or timed out.
  //
  // unlockToken must not be empty.
  //
  // timeout is in milliseconds.
  //
  // If predicate is not undefined, this method returns after predicate returns
  // true, or if timed out. If timed out, the final return value of the
  // predicate is returned. Whenever the predicate is executing, the lock on the
  // underlying mutex of unlockToken is acquired.
  //
  // If predicate is undefined, returns true if the wait was notified, and false
  // if timed out.
  static waitFor(cv: Condition,
                 unlockToken: UnlockToken,
                 timeout: Number,
                 predicate: Callable|undefined): bool

  // Notifies count waiters waiting on the condition variable cv.
  //
  // Returns the number of waiters that were notified.
  static notify(cv: Condition,
                count: Number = Infinity)
               : Number;
}
```

### Unsafe Blocks

Correct multithreaded programs are difficult to write, because races are subtle and difficult to reason about. To decrease the likelihood of incorrect programs, accesses to shared structs are only allowed when lexically contained with in an `unsafe { }` block. Note that SharedArrayBuffer access remains allowed in all contexts for backwards compatibilty.

The `unsafe` keyword is a clear signal of intent that a developer is choosing to work with shared memory multithreaded code. The presence of an `unsafe` block is an indication to code reviewers that special care must be taken during review. It also is acts as a syntactic marker that future tooling (linters, type checkers, etc.) could use to identify data races.

An `unsafe {}` block is otherwise treated the same as a normal Block. Its only distinction is that it explicitly labels code within the block as potentially containing non-thread-safe (e.g., "unsafe") code. The general expectation is that any thread safety concerns should be addressed by the developer as control flow exits the unsafe block. For example, you could utilize using to synchronize access to a shared struct via a lock:

```javascript
shared struct Counter {
  value = 0;
}

// normal JS code, outside of an "unsafe" context
const ctr = new Counter(); // allocations allowed
assertThrows(() => ctr.value = 1); // error (writes shared memory)
assertThrows(() => ctr.value);     // error (reads shared memory)

// "unsafe" JS code
unsafe {
  ctr.value = 1; // ok
  ctr.value;     // ok
}

function incrementCounter(ctr, mutex) {
  unsafe {
    using lck = Atomics.Mutex.lock(mutex);
    ctr.value++;
  }
}
```

Here, when the control enters the `unsafe` block, we allocate a lock against the provided mutex via a `using` declaration. As control exits the `unsafe` block, the lock tracked by using is released.

#### Lexically Scoped

The `unsafe` keyword is a syntactic marker that applies to lexically scoped reads and writes of the fields of a shared struct instance. Within an `unsafe` block, any lexically scoped accesses are permitted, even if they are nested within another function declared in the same block. This special lexical context shares some surface level similarities with the lexical scoping rules for private names, or the runtime semantics of "use strict".

Since `unsafe` is lexically scoped, it does not carry over to the invocation of functions declared outside of an `unsafe` context:

```javascript
function increment(ctr) {
  ctr.value++; // error due to illegal read/write of `ctr.value` outside of `unsafe`
}
unsafe {
  const ctr = new Counter();
  increment(ctr);
}
```

Thread-safe code may execute `unsafe` code without restriction, and `unsafe` code may do likewise. As `unsafe` already indicates a transition boundary between thread-safe and `unsafe` code, there is no need to declare all calling code `unsafe` as you might need to do for `async`/`await`. The `unsafe` keyword itself does not entail any implicit synchronization or coordination as that would be in opposition to our performance goals. Instead, the onus is on developers to be cognizant of thread safety concerns when they define an `unsafe` block. As such, a developer can choose the coordination mechanism that best suits the needs of their application, be that a `Mutex`, a lock-free concurrent deque, etc.

## Open Questions

### Attaching methods to shared structs

Because functions are deeply unshareable, shared structs currently do not have methods. However, this is a severe ergonomic pain point. It is also counter to encapsulation, which may have real harm in encouraging more thread-unsafe code. The current direction being explored to enable methods, which is undergoing discussion, is to give shared structs the following additional features:
1. A per-Realm prototype object, which is an ordinary object and thus can contain methods. This corresponds to making the [[Prototype]] internal field on shared structs thread-local storage.
1. A correlation mechanism to correlate evaluations of the same "logical" shared struct declaration across different agents.

This is an involved topic and has its own document. See [ATTACHING-BEHAVIOR.md](ATTACHING-BEHAVIOR.md).

## Future Work

### Asynchronous locking and waiting

Asynchronous locking is planned upcoming work but is out of scope of this proposal. See [ASYNC-LOCKING-WAITING.md](ASYNC-LOCKING-WAITING.md) for `lockAsync` and `waitAsync`.

## WasmGC interoperability

The [WasmGC proposal](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md) adds fixed layout, garbage-collected objects to Wasm. While the details of the type system of these objects are yet to be nailed down, interoperability with JavaScript is a requirement.

WasmGC objects have opaque storage and are not aliased by linear memory, so they cannot be exposed as all Wasm memory is exposed today via `ArrayBuffer`s. We propose structs to be the reflection of WasmGC objects in JS.

WasmGC objects exposed to JS should behave the same as structs, modulo extra type checking that WasmGC require that JS structs do not. JS structs is also a good foundation for reflecting into Wasm as WasmGC objects, but that is currently left as future work as it may need a typed field extensions to be worthwhile.

Further, WasmGC itself will eventually have multithreading. It behooves us to maintain a single memory model between JavaScript and Wasm as we have today, even with higher-level object abstractions.

## Out-of-Scope

### Value semantics, immutability, and operator overloading

This proposal does not intend to explore the space of objects with value semantics, including immutability and operator overloading. Structs have identity like other objects and are designed to be used like other objects. Value semantics is a sufficient departure that it may be better solved with other proposals that focus on that space.

### Sophisticated type systems

This proposal does not intend to explore sophisticated type and runtime guard systems. It is even more minimal than the closest spiritual ancestor, the [Typed Objects proposal](https://github.com/tschneidereit/proposal-typed-objects/blob/main/explainer.md), in that we do not propose integral types for sized fields. (Typed and sized fields are reserved for future work.)

### Binary data overlay views

This proposal does not intend to explore the space of overlaying structured views on binary data in an `ArrayBuffer`. This is a requirement arising from the desire for WasmGC integration, and WasmGC objects are similarly opaque.

Structured overlays are fundamentally about aliasing memory, which we feel is both a different problem domain, has significant performance downsides, and sufficiently solvable today in userland. For example, see [buffer-backed objects](https://github.com/GoogleChromeLabs/buffer-backed-object).

Notably, structured overlays in JavaScript essentially involves allocating unshared wrappers *per agent*. If an application has shared state with a complex structure, such as a large object graph, recreating that structure via a set of wrappers per agent negates the memory use benefits of *shared* memory. Structured overlays would work for specific application architectures where the structure of the shared state itself is simple, like a byte buffer.

## Implementation guidance

### Immutable shapes

Structs are declared with fixed layout up front. Engines should make an immutable shape for such objects. Optimizers can optimize field accesses without worrying about deopts.

### Shared structs: make sure fields are pointer-width and aligned

Shared structs should store fields such that underlying architectures can perform atomic stores and loads. This usually eans the fields should be at least pointer-width and aligned.

### Shared structs: strings will be difficult

Except for strings, sharing primitives in the engine is usually trivial, especially for NaN-boxing implementations.

Strings in production engines have in-place mutation to transition representation in order to optimize for different use ases (e.g. ropes, slices, canonicalized, etc). Sharing strings will likely be the most challenging part of the mplementation.

It is possible to support sharing strings by copying-on-sharing, but may be too slow. If possible, lockfree mplementations of in-place mutations above is ideal.

### Synchronization primitives: they must be moving GC-safe

Production engines use moving garbage collectors, such as generational collectors and compacting collectors. If JS ynchronization primitives are implemented under the hood as OS-level synchronization primitives, those primitives most ikely depend on an unchanging address in memory and are _not_ moving GC-safe.

Engines can choose to pin these objects and make them immovable.

Engines can also choose to implement synchronization primitives entirely in userspace. For example, WebKit's `ParkingLot`](https://webkit.org/blog/6161/locking-in-webkit/) is a userspace implementation of Linux futexes. This may have other benefits, such as improved and tuneable performance.
