# Structs

A `struct` declaration shares many similarities with a `class` declaration, with respect to syntax, and can have
elements such as constructors, fields, methods, getters, and setters:
```
struct S {
  foo;

  constructor() { }

  bar() { }

  get baz() { }
  set baz(value) { }
}
```

Methods and accessors are attached to a realm-local prototype reachable via the `struct`'s constructor. Unlike a
`class`, the fields of a `struct` have a fixed layout. You cannot `delete` fields from a `struct` instance, and
cannot attach new ones.

## Static Elements

In addition, fields, methods, getters, and setters can also be declared `static`, and you may also use `static {}`
blocks:

```
struct S {
  static foo = 1;

  static {
  }

  static bar() {}
  static get baz() { }
  static set baz(value) { }
}
```

As with a `class`, any `static` elements are attached to the constructor.

## Computed Property Names

Members of a `struct` may have computed property names, and the names of methods, getters, setters, and _static_ fields
can be either strings or symbols. However, it is unclear as to whether symbols will be allowed for the names of
fixed-layout fields.

```
const sym = Symbol();

struct S {
  [sym] = 1; // probably not ok?

  [Symbol.iterator]() { } // ok
}
```

## Private Names

Ideally, a `struct` may also have private-named elements, just like a `class`. Private-named fields would also be
part of the `struct`'s fixed field layout.

```
struct S {
  #bar;
  get bar() { return this.#bar; }
  set bar(value) { this.#bar = value; }
}
```

## Subclassing

A `struct` can inherit from another `struct` but cannot inherit from a regular function or `class` due to the nature of
how field layouts are established.

```
struct Sup {
  x;
  constructor(x) {
    this.x = x;
  }
}
struct Sub extends Sup {
  y;
  constructor(x, y) {
    super(x);
    this.y = y;
  }
}
```

Unlike with `class`, however, returning a different object from the superclass constructor does not change `this`,
and *all* fields of the constructed instance are installed on the `this` value when it is allocated so as to maintain
the fixed field layout inherent in `struct` values.

## `null` Prototypes

A `struct` can inherit from `null` instead of another `struct`, which produces a data-only `struct` with no prototypal
behavior. As such, a `struct` that extends `null` may not have non-static methods, getters, or setters.

```
struct S extends null {
  foo;
  bar;
}
```

# Shared Structs

Shared structs borrow the same syntax as non-shared struct, and are indicated with a leading `shared` keyword:

```
shared struct S {
  foo;

  constructor() { }

  bar() { }

  get baz() { }
  set baz(value) { }
}
```

As with non-shared structs, methods, getters, and setters are attached to a realm-local prototype. The constructor
function produced for a shared struct is also local to the realm, but is capable of producing shared struct instances.

When a shared struct is passed to a `Worker`, it is given a different realm-local prototype in that `Worker`'s realm.
If that `Worker` happens to load the same `shared struct` declaration from the same path/position, the prototype from 
that declaration is used, and that declaration's constructor can be used to create new instances of a `shared struct`
that, for all intents and purposes, is essentially the same as an instance from the main thread.

## Private Names

Ideally, a `shared struct` may also have private-named elements, just like a `class`. Private-named fields would also be
part of the `shared struct`'s fixed field layout.

```
shared struct S {
  #bar;
  get bar() { return this.#bar; }
  set bar(value) { this.#bar = value; }
}
```

However, this may not guarantee quite the same level of "hard privacy" that a private name on a class might have,
depending on what final mechanisms we might choose for correlating `shared struct` declarations. If malfeasant code is
able to construct a `Worker` that can forge an alternative definition for `S`, they would be able to craft replacement
methods that can directly access shared private state. For that reason, it would be recommended to avoid granting
untrusted code the ability to load an alternative definition for the `shared struct`, possibly through the use of CSP.

# Future Syntax Support: Decorators

Struct syntax can be further extended in the future for additional consistency with `class`, such as through the use
of decorators. Decorators have the potential to be useful for a number of scenarios, such as supplying runtime
annotations for WASM-GC types to fixed layout fields, or reusing existing decorators for methods and fields (including
auto-`accessor` fields):

```
shared struct S {
  @wasm_type("i8") x;
  @wasm_type("i32") y;

  @logged method() {}
}
```

# Future Syntax Support: References

While not yet proposed for Stage 1, some form of the [`ref` proposal](https://github.com/rbuckton/proposal-refs) would
also benefit `shared struct` declarations that utilize private-named fields as a mechanism to perform atomic updates:

```
shared struct S {
  #count = 0;

  get value() { return this.#count; }

  increment() {
    return Atomics.add(ref this.#count, 1);
  }

  decrement() {
    return Atomics.sub(ref this.#count, 1);

  }
}
```
