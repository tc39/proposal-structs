# Asynchronous Locking and Waiting

In a web browser, the main thread cannot block, with `Atomics.Mutex.lock` and `Atomics.Condition.wait` throwing instead of blocking as it would it in a worker.

This document lays out the asynchronous versions of `Atomics.Mutex.lock` and `Atomics.Condition.wait` which behave fairly differently.

## `Atomics.Mutex.lockAsync`

The core design of `lockAsync` mirrors that of the [Web Locks API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Locks_API). A callback to run under lock is passed, and is scheduled to run on the event loop when the lock is acquired.

```javascript
let mutex = new Atomics.Mutex;

let releasedPromise = Atomics.Mutex.lockAsync(
  mutex,
  async () => {
    doCriticalSectionStuff();
    await doAsyncCriticalSectionStuff();
  });

// releasedPromise settles when the mutex is released.
await releasedPromise;
```

There are two promises:

1. A **waiting promise**, that when settled, releases the mutex. That is, this promise controls the unlock. This promise is resolved with the return value of callback to run under lock.
1. A **released promise**, that settles when the mutex is released. This is returned by `Atomics.Mutex.lockAsync` itself.

For most use cases, the waiting promise is invisible, as in the above example. Since it is resolved with the return value of the callback, passing in a callback that returns a promise gives the user explicit control of when to release the lock.

```javascript
let mutex = new Atomics.Mutex;

let resolveWaiting;
let myWaitingPromise = new Promise((resolve) => {
  resolveWaiting = resolve;
});

let resolveAcquired;
let myAcquiredPromise = new Promise((resolve) => {
  resolveAcquired = resolve;
});

let releasedPromise = Atomics.Mutex.lockAsync(
  mutex,
  () => {
    resolveAcquired();
    return myWaitingPromise;
  });

await myAcquiredPromise;
doCriticalSectionStuff();
resolveWaiting();

await releasedPromise;
```

### Performance and starvation

`lockAsync` has *drastically* different performance implications than `lock`.

- Because the callback to run under lock is queued on the event loop, the critical section both runs at some arbitrary time after the lock was acquired and takes much longer to execute.
- Implementations may choose to starve async waiters in favor of sync waiters.

## `Atomics.Condition.waitAsync`

Actually this one is easy. `waitAsync` returns a promise that's settled when the condition is notified.

```javascript
let mutex = new Atomics.Mutex;
let cv = new Atomics.Condition;

await Atomics.Mutex.lockAsync(mutex, async () => {
  // This can be called on any agent. But has *drastically* different
  // performance implications than sync counterparts.
  await Atomics.Condition.waitAsync(cv, mutex);
});
```
