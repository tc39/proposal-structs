# Code Sharing Ideas for Shared Structs

This is a companion document collecting ideas for future proposals to share code to make shared structs more ergonomic.

## Shared function

### Introduction

Intuitively, shared functions are functions that are sufficiently restricted to be shareable across agents. At a high level, this means they cannot close over local state, globals are looked up according to the caller's Realm, and have a prototype separate from `Function.prototype`.

Shared functions are a new kind of exotic callable object with the following properties:
- Shared functions form a separate lexical environment chain than plain functions; shared function environments are immutable
- During evaluation of a shared function expression or declaration, if there is a free variable that would capture a value that is neither a primitive nor a shared struct or is a `Symbol`, throw a `ReferenceError`
- During evaluation of a shared function expression or declaration, if there is a free variable with the same name as a binding in an outer plain function or as a top-level lexical binding, throw a `ReferenceError`
- During execution of a shared function, free identifiers are looked up in the caller's Realm
- During execution of a shared function, new objects are allocated in the caller's Realm
- During execution of a shared function, they use their caller's Realm whenever a Realm is needed
- Shared functions are sealed
- Shared functions have as their prototype `SharedFunction.prototype`, a sealed object that is itself shared across agents; there is a single `SharedFunction.prototype` per agent cluster

### Example

```javascript
// main.js
shared struct SharedBox { x; }
shared function mkSharedFun() {
  let sharedBox = new SharedBox;
  sharedBox.x = "main";
  let prim = 42;
  return shared function(arg) {
    // Console is a free variable, it'll be looked up in the caller's Realm.
    console.log(sharedBox.x, prim, arg);
  };
}
let worker = new Worker('worker.js');
worker.postMessage({ sharedFun: mkSharedFun() });
```

```javascript
// worker.js
onmessage = function(e) {
  let sharedFun = e.data.sharedFun;

  // Logs "main 42 worker" using the worker's console
  sharedFun('worker');
};
```

```javascript
// Cannot accidentally close over a local binding.
function outerLocal() {
  let local;
  return shared function() { console.log(local); }
}

// Unreachable due to SyntaxError
```

```javascript
// Cannot close over a non-shared value.
assertThrows(() => { (shared function outer1() {
  let x = {}; // Object literal syntax allocates a local object.
  return shared function() { x; }
})(); });

// Shared functions are sealed and not extensible.
shared function s() { }
assertThrows(() => { s.myProp = 42; });

// Cannot mutate closed-over values
shared function mkBad() {
  let x = 42;
  return shared function() { x++; }
}
assertThrows(() => { mkBad()(); });
```

### Extending shared structs

With shared functions, shared structs could be extended to support methods and prototype chains. All instance methods in a `shared struct` expression would be shared functions.

Shared struct constructors themselves are not shared by default. They can be made shared by prefixing `shared`. A shared constructor will also expose the `.constructor` property on its instances.

A combined example is shown below.

```javascript
// main.js
shared struct SharedPoint {
  shared constructor(x, y) { this.x = x; this.y = y; }
  x;
  y;
  add(p) {
    // Warning: these are racy!
    this.x += p.x;
    this.y += p.y;
  }
}

let p = new SharedPoint(1, 1);
let worker = new Worker('worker.js');
worker.postMessage({ p });

// Depends on timing, this could log 1 or 2.
console.log(p.x);
```

```javascript
// worker.js
onmessage = (e) => {
  let p = e.data.p;
  let SharedPoint = p.constructor;
  console.log(p.add(new SharedPointer(1, 1)));
};
```
