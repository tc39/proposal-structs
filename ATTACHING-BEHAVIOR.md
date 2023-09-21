# Attaching Behavior to Shared Structs

The shared part of shared structs refers to data. Because functions in JS are closures that are deeply tied to their creation Realm, they are not shareable in the same way data is. For this reason, methods are not allowed in shared struct declarations by default. In the fullness of time, it is desirable to add truly shareable functions to JS (see [CODE-SHARING-IDEAS.md](CODE-SHARING-IDEAS.md) for a collection of ideas).

Nevertheless, it is crucial to the developer experience of shared structs that one _can_ attach behavior to them.

For example, a codebase may want to incrementally adopt multithreading in an existing codebase by converting some existing `class` declarations. These existing `class` declarations are likely to have methods. Without the ability to attach methods to shared structs, the whole codebase would need to be refactored such that uses of shared structs instances use free functions instead of method calls. This is poor DX that should be solved.

## Agent-local prototypes

In the short term, the crux of the idea to attach behavior to shared structs is the ability to have agent-local (i.e. thread-local) prototype objects. A shared struct that declares its prototype to be agent-local will have a plain object lazily created in the current agent upon access. Since the prototype is a plain object and agent-local, it can reference any value, including ordinary functions.

Pending bikeshedding, the proposed syntax is as follows.

```javascript
shared struct class SharedThing {
  // Incantation for saying SharedThing has an agent-local prototype.
  static nonshared prototype;
}
let s = new SharedThing;

// Inside worker A:
Object.getPrototypeOf(s).whereAmI = () => { return "local to A" };

// Inside worker B:
Object.getPrototypeOf(s).whereAmI = () => { return "local to B" };
```

Semantically, agent-local prototypes is equivalent to a per-agent `WeakMap` keyed by some abstract "type identifier" corresponding to the shared struct (e.g. the underlying `map` in V8's object representation).

### Realm-local?

It is an open design question whether the right granularity is agent-local or Realm-local.

The concept of an agent is not currently reified in the language, so a Realm may be the more natural choice. On the other hand, requiring prototype setup per-Realm may be onerous and unnecessary for most applications.

## Coordinating identity continuity among workers

Shared struct declarations evaluate to new constructors per evaluation, just like ordinary `class` declarations. In other words, they are nominally typed instead of structurally typed. However, this poses an identity discontinuity issue in a multi-worker setting.

Consider the following:

```javascript
// Inside worker A:
shared struct class SharedThing {
  static nonshared prototype;

  x;
  y;
}

SharedThing.prototype.foo =
  () => { console.log("do a foo"); };
let thing1 = new SharedThing;

workerB.postMessage(thing1);

// Inside worker B:
shared struct class SharedThing {
  static nonshared prototype;

  x;
  y;
}

SharedThing.prototype.foo =
  () => { console.log("do a foo"); };

onmessage = (thing) => {
  // undefined is not a function!
  thing.foo();
};
```

In the code snippet above, `thing.foo()` doesn't work because worker A and worker B have different evaluations of `SharedThing`. While the two `SharedThing`s have identical layout, they are nevertheless different "types". So, `thing` from worker A is an instance of worker A's `SharedThing`, whose agent-local prototype was never set up by worker B. Worker B only set up the agent-local prototype of its own evaluation of `SharedThing`.

The proposed solution is to use a agent cluster-wide registry to correlate multiple evaluations of shared struct declarations. Shared struct declarations that are declared to be **registered** have the following semantics:

- When evaluated, if the class name does not exist in the registry, insert a new entry whose key is the class name, and whose value is a description of the layout (i.e. order and names of instance fields, and whether the prototype is agent-local).
- When evaluated, if the class name already exists in the registry, check if the layout exactly matches the current evaluation's layout. If not, throw.
- Evaluation of the declaration returns a constructor function that always creates instances recognizable by the engine as the same layout. In other words, the registry deduplicates layout.
- Lookup and insertion into the registry are synchronized and threadsafe.

The primary purpose of the registry is to coordinate the setting up of agent-local prototypes without additional communication.

Consider again the above example with registered shared structs.

```javascript
// Inside worker A:
registered shared struct class SharedThing {
  static nonshared prototype;

  x;
  y;
}

SharedThing.prototype.foo =
  () => { console.log("do a foo"); };
let thing1 = new SharedThing;

workerB.postMessage(thing1);

// Inside worker B:
registered shared struct class SharedThing {
  static nonshared prototype;

  x;
  y;
}

SharedThing.prototype.foo =
  () => { console.log("do a foo"); };

onmessage = (thing) => {
  thing.foo(); // do a foo
};
```

Because the two shared struct declarations have the same name, `SharedThing`, and are both declared registered, the VM is able to connect the types under the hood.

### Incompatible layouts

If two registered shared struct declarations share the same name, their layout must match. This means the order and name of their fields must match, and whether the prototype is agent-local must match. A non-match throws.

```javascript
{
  registered shared struct class SharedThing {
    x;
    y;
  }
}

{
  registered shared struct class SharedThing {
    y;
    x;
  } // throws
}
```

### Agent-local prototype "surprise"

While each registered shared struct declaration continues to evaluate to a new constructor function, the layout deduplication means that a registered shared struct declaration may evaluate to a function with an already populated agent-local prototype.

```javascript
{
  registered shared struct class SharedThing {
    static nonshared prototype;

    x;
    y;
  }

  SharedThing.prototype.foo = "bar";
}

{
  registered shared struct class SharedThing {
    static nonshared prototype;

    x;
    y;
  }

  console.log(SharedThing.prototype.foo); // "bar"
}
```

While this may be surprising, this pattern seems unlikely to occur in the wild. It is unlikely that the same registered shared struct declarations will be evaluated multiple times. Declaring something registered signals intent that it ought to be behave the same across worker boundaries.

## Communication channel

The registry proposed above is a communication channel because of the throwing semantics if incompatible shared structs are registered with the same name.

Communication channels in themselves are not a problem so long as they are _deniable_. To that end, we also propose a method to freeze the registry such that any additional `registered shared struct` declarations will always throw. This registry is not reified otherwise in the language.

```javascript
// SharedStructRegistry is a namespace object.
SharedStructRegistry.freeze();

assertThrows(() => {
  registered shared struct class SharedThing {} // throws
});
```

## Should this be the default?

It is arguable that the `registered` behavior ought to be the default behavior for all shared structs. Making it the default would save on modifier keyword soup and also makes the currently proposed default less surprising.

## Upshot

The combination of agent-local prototypes and an agent cluster-wide registry means worker threads can independently evaluate the same shared struct definitions (i.e. same source text) without additional coordination, and things more or less work as expected.

Additionally, it has the performance advantage of performing deduplication of layouts at declaration time. This means more monomorphic inline caches and better sharing.
