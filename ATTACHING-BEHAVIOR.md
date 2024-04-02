# Attaching Behavior to Shared Structs

The shared part of shared structs refers to data. Because functions in JS are closures that are deeply tied to their creation Realm, they are not shareable in the same way data is. For this reason, methods are not allowed in shared struct declarations by default. In the fullness of time, it is desirable to add truly shareable functions to JS (see [CODE-SHARING-IDEAS.md](CODE-SHARING-IDEAS.md) for a collection of ideas).

Nevertheless, it is crucial to the developer experience of shared structs that one _can_ attach behavior to them.

For example, a codebase may want to incrementally adopt multithreading in an existing codebase by converting some existing `class` declarations. These existing `class` declarations are likely to have methods. Without the ability to attach methods to shared structs, the whole codebase would need to be refactored such that uses of shared structs instances use free functions instead of method calls. This is poor DX that should be solved.

## Realm-local prototypes

In the short term, the crux of the idea to attach behavior to shared structs is the ability to have realm-local prototype objects, analogous to the per-realm primitive prototype wrappers (e.g. `Number.prototype`). A shared struct that declares its prototype to be realm-local will have a plain object lazily created in the current realm upon access. Since the prototype is a plain object and realm-local, it can reference any value, including ordinary functions.

Pending bikeshedding, the proposed syntax is as follows.

```javascript
shared struct SharedThing {
  // Incantation for saying SharedThing has an realm-local prototype.
  //
  // Open to bikeshedding.
  with nonshared prototype;
}
let s = new SharedThing;

// Inside worker A:
Object.getPrototypeOf(s).whereAmI = () => { return "local to A" };

// Inside worker B:
Object.getPrototypeOf(s).whereAmI = () => { return "local to B" };
```

Semantically, realm-local prototypes is equivalent to a per-realm `WeakMap` keyed by some abstract "type identifier" corresponding to the shared struct (e.g. the underlying `map` in V8's object representation). This key is not exposed to user code.

## Coordinating identity continuity among workers

Shared struct declarations evaluate to new constructors per evaluation, just like ordinary `class` declarations. In other words, they are nominally typed instead of structurally typed. However, this poses an identity discontinuity issue in a multi-worker setting.

Consider the following:

```javascript
// Inside worker A:
shared struct SharedThing {
  with nonshared prototype;

  x;
  y;
}

SharedThing.prototype = {
  foo() { console.log("do a foo"); }
};
let thing1 = new SharedThing;

workerB.postMessage(thing1);

// Inside worker B:
shared struct SharedThing {
  with nonshared prototype;

  x;
  y;
}

SharedThing.prototype = {
  foo() { console.log("do a foo"); }
};

onmessage = (thing) => {
  // undefined is not a function!
  thing.foo();
};
```

In the code snippet above, `thing.foo()` doesn't work because worker A and worker B have different evaluations of `SharedThing`. While the two `SharedThing`s have identical layout, they are nevertheless different "types". So, `thing` from worker A is an instance of worker A's `SharedThing`, whose realm-local prototype was never set up by worker B. Worker B only set up the realm-local prototype of its own evaluation of `SharedThing`.

The proposed solution is to use an agent cluster-wide registry to correlate multiple evaluations of shared struct declarations. Shared struct declarations that are declared to be **registered** have the following semantics:

- When evaluated, if the source location does not exist in the registry, insert a new entry whose key is the source location, and whose value is a description of the layout (i.e. order and names of instance fields, and whether the prototype is realm-local).
- When evaluated, if the source location already exists in the registry, check if the layout exactly matches the current evaluation's layout. If not, either throw or do nothing (open design question).
- Evaluation of the declaration returns a constructor function that always creates instances recognizable by the engine as the same layout. In other words, the registry deduplicates layout.
- Lookup and insertion into the registry are synchronized and threadsafe.
- The registry is not programatically accessible to user code.

The primary purpose of the registry is to coordinate the setting up of realm-local prototypes without additional communication.

Consider again the above example with registered shared structs.

```javascript
// Inside worker A:
shared struct SharedThing {
  // Incantation to perform auto-correlation.
  with registered;
  with nonshared prototype;

  x;
  y;
}

SharedThing.prototype = {
  foo() { console.log("do a foo"); }
};

let thing1 = new SharedThing;

workerB.postMessage(thing1);

// Inside worker B:
shared struct SharedThing {
  // Incantation to perform auto-correlation.
  with registered;
  with nonshared prototype;

  x;
  y;
}

SharedThing.prototype = {
  foo() { console.log("do a foo"); }
};

onmessage = (thing) => {
  thing.foo(); // do a foo
};
```

Because the two shared struct declarations have the same name, `SharedThing`, and are both declared registered, the VM is able to connect the types under the hood.

### Incompatible layouts

If two registered shared struct declarations share the same name, their layout must match. This means the order and name of their fields must match, and whether the prototype is realm-local must match. A non-match might throw or do nothing (open design question). If the chosen behavior is to do nothing (i.e. do not deduplicate), the console ought to warn.

```javascript
{
  registered shared struct SharedThing {
    x;
    y;
  }
}

{
  registered shared struct SharedThing {
    y;
    x;
  } // either throws or silently do not deduplicate
}
```

### Realm-local prototype "surprise"

While each registered shared struct declaration continues to evaluate to a new constructor function, the layout deduplication means that a registered shared struct declaration may evaluate to a function with an already populated realm-local prototype.

```javascript
{
  shared struct SharedThing {
    with nonshared prototype;

    x;
    y;
  }

  SharedThing.prototype.foo = "bar";
}

{
  shared struct SharedThing {
    with nonshared prototype;

    x;
    y;
  }

  console.log(SharedThing.prototype.foo); // "bar"
}
```

While this may be surprising, this pattern seems unlikely to occur in the wild. It is unlikely that the same registered shared struct declarations will be evaluated multiple times. Declaring something registered signals intent that it ought to be behave the same across worker boundaries.

## Communication channel

Since the registry key is source location, the key is unforgeable. If on layout mismatch, the chosen behavior is to do nothing, then this is not an observable communication channel. If instead the chosen behavior is to throw, then to exploit the registry as a communication channel requires being able to modify source text and triggering re-evaluations of said source text, which seems unlikely.

## Should this be the default?

It is arguable that the registered behavior ought to be the default behavior for all shared structs. Making it the default would save on modifier keyword soup and also makes the currently proposed default less surprising.

## Bundler guidance

Because the source location is now meaningful, duplication of shared struct declarations in source text is no longer a semantics-preserving transformation. Bundlers and program transforms must take care during e.g., tree shaking on specific worker threads, to duplicate shared struct declarations.

## Upshot

The combination of agent-local prototypes and an agent cluster-wide registry means worker threads can independently evaluate the same shared struct definitions (i.e. same source text) without additional coordination, and things more or less work as expected.

### Performance advantages

Additionally, it has the performance advantage of performing deduplication of layouts at declaration time. This means more monomorphic inline caches and better sharing. Without deduplication of layout, the number of morally equivalent shared struct types scales with the number of threads, which means a multithreaded program becomes more polymorphic with more threads, which is highly undesirable.
