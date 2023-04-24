# Chromium Dev Trial

An experimental implementation of JS shared memory features is gated behind Chromium as of M108 with the the `JavaScriptExperimentalSharedMemory` flag. You can enable this by passing `--enable-features=JavaScriptExperimentalSharedMemory` to a chromium build or navigating to `chrome://flags/#enable-javascript-experimental-shared-memory`.

# New APIs

Enabling the experimental JS shared memory features enables the following APIs, which differ in syntax from the current proposal but have largely the same semantics.

## Shared structs

Shared structs can be made by first making shared struct types, and then making instances of those types. Unlike the proposal, there is no support for `struct` syntax and this must be done programmatically.

Shared structs can only store JS primitives and other shared objects. They have a `null` constructor and prototype. They are sealed, i.e. new fields cannot be added, thus the need to declare a type up front.

```javascript
// SharedStructType takes an array of field names.
let SharedBox = new SharedStructType(["x"]);

let sharedBox = new SharedBox();
let sharedBox2 = new SharedBox();

sharedBox.x = 42;          // x is declared and rhs is primitive
sharedBox.x = sharedBox2;  // x is declared and rhs is shared
assertThrows(() => { sharedBox.x = {}; }); // rhs is not shared
```

Shared structs may be brand checked with `SharedStructType.isSharedStruct`. Note that `isSharedStruct(o)` returns `true` if `o` is _any_ shared struct.

## Shared arrays

Shared arrays are fixed-length JS arrays.

```javascript
// Make a 1024-long shared array.
let sharedArray = new SharedArray(1024);

sharedArray[0] = 42;               // rhs is primitive
sharedArray[0] = new SharedBox();  // rhs is shared
assertThrows(() => { sharedArray[0] = {}; }); // rhs is not shared
```

Shared arrays may be brand checked with `SharedArray.isSharedArray`.

## Synchronization primitives

Two high-level synchronization primitives are provided: `Atomics.Mutex` and `Atomics.Condition`.

Because they are shared objects, and shared objects cannot have methods, they are used using free methods on the `Atomics` namespace object.

```javascript
let mutex = new Atomics.Mutex;

// This would block the current thread if the lock is held
// by another thread. On the web, this cannot be used on the
// main thread since the main thread cannot block.
Atomics.Mutex.lock(mutex, function runsUnderLock() {
  // Do critical section stuff
});

// tryLock is like lock but returns true if the lock was acquired
// without blocking, and false is the lock is held by another
// thread.
Atomics.Mutex.tryLock(mutex, function runsUnderLock() {
  // Do critical section stuff
});

sharedBox.x = mutex;   // rhs is shared
```

```javascript
let cv = new Atomics.Condition;

// This blocks the current thread, and cannot be used on the main
// thread. The passed in mutex must be locked.
Atomics.Mutex.lock(mutex, () => {
  Atomics.Condition.wait(cv, mutex);
});

// Waiters can notified with a count. A count of undefined or
// +Infinity means "all waiters".
let count = 1;
let numWaitersWokenUp = Atomics.Condition.notify(cv, count);
```

Mutexes and condition variables may be brand checked with `Atomics.Mutex.isMutex` and `Atomics.Condition.isCondition`, respectively.

# Sharing across threads

In a browsing context, shared objects created by the above APIs can be shared across threads if and only if SharedArrayBuffers can be shared across threads. That is, if and only if [`self.crossOriginIsolated`](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) is `true`, which is determined by setting [COOP and COEP headers](https://web.dev/coop-coep/).

```javascript
// main.js
let worker = new Worker("worker.js");

// Assume sharedBox from above.
assert(self.crossOriginIsolated);
worker.postMessage({ sharedBox });

// By default, property accesses are racy. This is a data race!
sharedBox.x = "race written by main";

// Lock-free accesses can be totally ordered (sequentially consistent)
Atomics.store(sharedBox, "x", "sequentially consistent");
```

```javascript
// worker.js
onmessage = function (e) {
  // This is the same sharedBox as the one the main thread has, not a copy
  let sharedBox = e.data;

  // This is a data race!
  console.log(sharedBox.x);

  // This is totally ordered
  console.log(Atomics.load(sharedBox, "x"));
};
```
