# Accessing the originals

A proposal for a web platform API that allows access to the "original" versions of the global built-in objects' properties and methods, via a [built-in module](https://github.com/tc39/proposal-javascript-standard-library/).

## Motivations

### Writing robust code

An important technique when writing robust APIs is to call the underlying operation, not the publicly-modifiable API. This is normally quite difficult in JavaScript. A simple own-property check, such as

```js
const result = Object.prototype.hasOwnProperty.call(someObject, "someProp");
```

is subject to interference in any of the following ways:

* `globalThis.Object` could be overridden
* `Object.prototype` could be overridden
* `Object.prototype.hasOwnProperty` could be overridden
* `Object.prototype.hasOwnProperty.call` could be set (shadowing `Function.prototype.call`)
* `Function.Prototype.call` could be overridden

The standard way of avoiding this is by practicing [safe meta-programming](https://web.archive.org/web/20160805225710/http://wiki.ecmascript.org/doku.php?id=conventions:safe_meta_programming), potentially assisted by tools or type systems. This relies on grabbing the "original" values of the globals ahead of time; see the linked page for more examples.

However, conventional safe meta-programming techniques only work if you can guarantee that your script is the first on the page to run. (Otherwise, previously-run script may have interfered with the globals before you can grab the original values.) This is increasingly infeasible in a library-heavy world; only one library can run first, after all.

Additionally, the requirement to run first (or at least as early as possible) in the loading cycle is tension with the movement toward more lazy-loading of non-essential code. Third-party scripts, such as embeds, analytics, or ads, are perfect candidates for lazy-loading. However, the authors of these third party scripts need to be able to run in as many environments as possible, and as such greatly benefit from safe meta-programming.

The get-originals proposal provides a first-class way to get the original values needed for safe meta-programming, even for scripts that are loaded later in the page's lifecycle.

### Bridging the module and global worlds

There are many benefits from obtaining as many standard-library features as possible from the module system, instead of the global object. Developers often prefer the explicitness of such `import`s to global dereferences, and tooling appreciates the static analyzability. As such, by transposing global built-ins into the module system, the get-originals proposal allows writing code that makes its standard library dependencies explicit intead of implicit.

This benefit is a minor nice-to-have for JavaScript developers. But it is a significant upgrade for WebAssembly developers, which rely exclusively on their module system to be able to import functionality into their programs.

Right now, any attempts to access the platform's standard library need to be explicitly proxied via a JavaScript wrapper:

```js
WebAssembly.instantiateStreaming(fetch("simple.wasm"), {
  console: {
    log: console.log
  },
  TextEncoder: {
    default: TextEncoder,
    encodeInto: TextEncoder.prototype.encodeInto
  },
  /* ... */
});
```

The WebAssembly module system then can then import these provided function values.

The get-originals proposal provides direct, first-class access to the built-in globals (normally inaccessible to WebAssembly) via the module system ([soon](https://github.com/WebAssembly/esm-integration/blob/master/proposals/esm-integration/README.md) accessible to WebAssembly). This obviates the need to bridge all used functionality through JavaScript.

TODO: explain connection with Web IDL bindings, anyref, etc. for standard library functionality that doesn't take just integer types. Perhaps change above example if it relies too heavily on those to be workable.

### Platform layering

All web specifications, and many parts of the JavaScript specification, use the original functionality, instead of going through web-developer-modifiable APIs.

For example, [the algorithm for `JSON.stringify()`](https://tc39.github.io/ecma262/#sec-json.stringify) uses the original [IsArray](https://tc39.github.io/ecma262/#sec-isarray) abstract operation, and not the publicly-modifiable [`Array.isArray()`](https://tc39.github.io/ecma262/#sec-array.isarray) function, to determine the serialization of the values it is passed. The [toggleAttribute()](https://dom.spec.whatwg.org/#dom-element-toggleattribute) method directly operates on the underlying attributes collection of the element in question, instead of calling `this.setAttribute()` or `this.removeAttribute()`.

It is currently impossible to emulate these semantics in JavaScript. This means that we have discovered a missing low-level capability—access to the original, unmodified platform operations—which our higher-level features are built on top of. The [Extensible Web Manifesto](https://extensiblewebmanifesto.org/) calls on us to prioritize efforts which expose such low-level capabilities to web developers, and in doing so, explain the existing features built on top of them.

In addition to the general principles of the Extensible Web Manifesto, we'll note that access to the originals is required for full-fidelity polyfilling. Just like the real implementation of `toggleAttribute()`, a polyfill should be robust in the face of tampering with `setAttribute()` and `removeAttribute()`.

## Example usage

### Basic example

Consider our introductory example from above:

```js
const result = Object.prototype.hasOwnProperty.call(someObject, "someProp");
```

To make this robust, and emulate the way that the web platform would check for own properties, we could do the following:

```js
import { apply } from "std:global/Reflect";
import { hasOwnProperty } from "std:global/Object";

const result = apply(hasOwnProperty, someObject, ["someProp"]);
```

TODO WebAssembly example.

### More complex example

Consider what it would take to make the following code robust:

```js
function storeIt(value) {
  if (!Array.isArray(value)) {                     // (1)
    throw new TypeError("Must be an array!");      // (2)
  }

  const request = self.indexedDB.open("my-db", 1); // (3) (4) (5)

  request.onsuccess = () => {                      // (6)
    const database = request.result;               // (4)
    // ...
  });
}
```

The numbered lines point to the various operations we need to be able to do without triggering any potential monkeypatches:

1. Calling a static method (`Array.isArray()`)
2. Invoking a global constructor (`TypeError`)
3. Referencing the global (`self`)
4. Getting a property (`self.indexedDB`, `request.result`)
5. Calling a method (`indexedDB.open()`)
6. Setting a property (`request.onsuccess`)

Here is how we would rewrite the above code to be robust, while using the proposed get-originals API:

```js
import oSelf from "std:global";
import { apply } from "std:global/Reflect";
import { isArray_static } from "std:global/Array";
import TypeError as oTypeError from "std:global/TypeError";

import { indexedDB_get } from "std:global/Window";
import { open as IDBFactory_open } from "std:global/IDBFactory";
import { onsuccess_set as IDBRequest_onsuccess_set,
         result_get as IDBRequest_result_get } from "std:global/IDBRequest";

const oIndexedDB = apply(indexedDB_get, oSelf);                     // (3) (4)

function storeIt(value) {
  if (!isArray_static(value)) {                                     // (1)
    throw new oTypeError("Must be an array!");                      // (2)
  }

  const request = apply(IDBFactory_open, oIndexedDB, ["my-db", 1]); // (5)

  apply(IDBRequest_onsuccess_set, request, [() => {                 // (6)
    const database = apply(IDBRequest_result_get, request);         // (4)
    // ...
  }]);
}
```

Obviously, as with all safe meta-programming, this code is not that pleasant to write by hand, and hard to get correct. We are [exploring tools](#automatic-rewritingenforcement-of-robustness) to automate this rewriting process.

## The API

We propose adding the a new [built-in module](https://github.com/tc39/proposal-javascript-standard-library/) for every exposed global constructor and namespace object, of the form `"std:global/X"`. We would also have a special `"std:global"` built-in module which default-exports the global object itself (with no other exports).

Roughly speaking, each such module would have a series of exports:

- For classes:
  - The default export would be the constructor function.
  - The methods of the associated class would be exported under their original names.
  - Any static methods of the associated class would be exposed with a `_static` suffix.
  - Any getters and setters on the associated class's prototype would be exposed with `_get` and `_set` suffixes.
- For namespaces:
  - The functions stored on the namespace would be exported under their original names.
  - If a namespace includes a class, that class is exposed based on its qualified name.

(See [below](#name-mangling) for more discussion on these suffixes.)

_Open question: should we include data properties, i.e. "constants"? For example, should you be able to get the original value of `Math.PI` or `Event.AT_TARGET`? We omit them for now. Discuss in [#9](https://github.com/domenic/get-originals/issues/9)._

So, for example:

- `"std:global/Event"` (based on the [`Event` class](https://dom.spec.whatwg.org/#interface-event)) would expose:
  - Method exports `composedPath`, `stopPropagation`, `stopImmediatePropagation`, `preventDefault`,  `initEvent`.
  - Getter/setter exports `type_get`, `target_get`, `srcElement_get`, `currentTarget_get`, `eventPhase_get`, `bubbles_get`, `cancelable_get`, `returnValue_get`, `returnValue_set`, `defaultPrevented_get`, `composed_get`, `isTrusted_get`, `timeStamp_get`
- `"std:global/Reflect"` (based on the [`Reflect` namespace](https://tc39.github.io/ecma262/#sec-reflect-object)) would expose:
  - Function exports `apply`, `construct`, `defineProperty`, `deleteProperty`, `get`, `getOwnPropertyDescriptor`, `getPrototypeOf`, `has`, `isExtensible`, `ownKeys`, `preventExtensions`, `set`, `setPrototypeOf`
- `"std:global/Number"` (based on the [`Number` class](https://tc39.github.io/ecma262/#sec-number-objects)) would expose:
  - Method exports: `toExponential`, `toFixed`, `toLocaleString`, `toPrecision`, `toString`, `valueOf`
  - Static method exports: `isFinite_static`, `isInteger_static`, `isNaN_static`, `isSafeInteger_static`, `parseFloat_static`, `parseInt_static`
- `"std:global/WebAssembly.Memory"` (based on the [`WebAssembly.Memory` class](https://webassembly.github.io/spec/js-api/index.html#memories)) would expose:
  - Method export: `grow`
  - Getter export: `buffer_get`

### Identity is preserved

Each of these exports would be the same function object as can be accessed via the "normal" route. So, for example, assuming nobody had messed with the built-ins yet, we would have

```js
import { toExponential } from "std:global/Number";
console.assert(toExponential === Number.toExponential);
```

and even

```js
import { type_get } from "std:global/Event";
console.assert(type_get === Object.getOwnPropertyDescriptor(Event.prototype, "type").get);
```

This is especially important for constructors; consider a case like

```js
// Library code:
import oPromise from "std:global/Promise";
import { setTimeout as oSetTimeout } from "std:global/Window";
import global from "std:global";

function delay(ms) {
  return new oPromise(resolve => {
    oSetTimeout(resolve, ms);
  });
}
```

```js
// Consumer code:
console.assert(delay(500) instanceof Promise);
```


### Exposure restrictions are preserved

The exports and existence of these modules will vary per type of global, according to what is [exposed](https://heycam.github.io/webidl/#dfn-exposed) in that realm. So for example,

- `"std:global/Event"` will exist in windows, workers, and audio worklet realms, but not in paint worklet realms
- `"std:global/Node"` will only exist in window realms
- `"std:global/Math"` will exist in all realms
- The `setMatrixValue` export of `"std:global/DOMMatrix"` will only exist in window realms' versions of `"std:global/DOMMatrix`

Note that because the properties of [module namespace objects](https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects) are immutable, they provide a natural way to share access to the originals without allowing multiple consumers to interfere with each other.

## API discussion

### Motivation for using modules

TODO WebAssembly, import maps, import maps allow virtualization (#12, #17, #18)

### Name mangling

To accomplish this proposal's goals, all functions (static methods, prototype methods, getters, and setters) needs to be provided as non-configurable, non-writable properties. The API above does this by making them all module exports, where each global has a flattened namespace which all four types of functions share. This sharing is made possibly by mangling the names of static methods, getters, and setters.

In particular, the proposal assumes we will never have prototype methods whose names end in `_get`, `_set`, or `_static`. We think this asssumption is safe; such names are [prohibited by the W3C TAG design principles](https://w3ctag.github.io/design-principles/#casing-rules), and we don't anticipate any standards body introducing any methods that violate them. We could enforce this in Web IDL, for all specs that depend on that infrastructure.

If folks think this kind of constraint on the future is bad, we could consider alternatives, such as:

- different, more-esoteric suffixes, such as `$get` or `_____get` or `$_$_$get`, or
- separate modules `"std:global/X/statics"`, `"std:global/X/methods"`, and `"std:global/X/accessors"`

### Safe usage

As emphasized a few times already, safe meta-programming is not easy. Here are two techniques that users of this proposal need to keep in mind especially:

#### Use (the original) `Reflect.apply` extensively

All methods, getters, and setters imported from the `"std:global/X"` modules will need to be supplied with a correct thisArg. It may be tempting to write code such as the following:

```js
import { hasOwnProperty } from "std:global/Object";

// *** BAD: THIS WILL NOT WORK ***
const result = hasOwnProperty(someObject, "someProp");
```

which forgets to set the thisArg, or code such as this:

```js
import { hasOwnProperty } from "std:global/Object";

// *** BAD: THIS IS UNSAFE ***
const result = hasOwnProperty.call(someObject, "someProp");
```

which unsafely accesses the `.call` property of `hasOwnProperty`, which is not necessarily the original `Function.prototype.call` you'd be expecting.

Instead, you want to use the original version of `Reflect.apply`:

```js
import { apply } from "std:global/Reflect";
import { hasOwnProperty } from "std:global/Object";

// *** CORRECT: THIS IS SAFE ***
const result = apply(hasOwnProperty, someObject, ["someProp"]);
```

An alternate technique is to define an [uncurry thisArg](http://2ality.com/2011/11/uncurrying-this.html) function; doing so using get-originals APIs as the foundation is left as an exercise to the reader.

#### Do not grab properties from exports of the originals modules

A tempting footgun is to import the minimal number of originals, and then attempt to use their properties:

```js
import Array from "std:global/Array";

// *** BAD: THIS IS UNSAFE ***
const result = Array.isArray(someObject);
```

The get-originals API does nothing to prevent this; it can't, since the imported `Array` is [the same](#identity-is-preserved) as the normal global `Array`, which has an `isArray` static method. But of course, this code is suspectible to tampering. Instead, you need to do

```js
import { isArray_static } from "std:global/Array";

// *** CORRECT: THIS IS SAFE ***
const reuslt = isArray_static(someObject);
```

### Movement between prototypes

In the normal course of writing code, developers are not usually aware of what prototypes their methods and accessors come from. For example, when they do `document.body.firstElementChild`, they don't need to know or care whether `firstElementChild` is defined on `Node` or `Element`. It is observable:

```js
console.assert(!("firstElementChild" in Node.prototype));
console.assert("firstElementChild" in Element.prototype);
```

but normally it isn't observed. This has historically given the spec ecosystem some wiggle room in moving methods and accessors between prototypes. It has also led to some interoperability issues due to different placement, with [`HTMLDocument` vs. `Document`](https://github.com/whatwg/dom/issues/221) being the main example—but these interoperability issues have not been so bad to date, precisely because web developers so rarely observe the differences.

In contrast, any users of the get-originals APIs need to know exactly where in the prototype chain their methods and accessors are located:

```js
import { firstElementChild_get } from "std:global/Node";    // throws!
import { firstElementChild_get } from "std:global/Element"; // works
```

This reduces our flexibility in moving things between prototypes, and makes any interop issues immediately apparent to web developers.

We believe this issue is surmountable, through a few mechanisms:

- Rigorous tests should accompany any initial implementations of get-originals, which test the _per spec_ locations of all methods and accessors. If any implementation fails to conform to the spec's location for a method or accessor placement, they should refrain from shipping the relevant `"std:global/X"` module until we get either the spec or implementation situation straightened out.
- If we do need to move methods in the future, we can leave the exports from the `"std:global/X"` module in place. For example, if we moved `firstElementChild` from `Element.prototype` to `Node.prototype`, we could still export `firstElementChild_get` from the `"std:global/Element"` module to mantain backward compatibility. This could be maintained via Web IDL annotations, e.g. `[OriginalAlsoExportedFrom=Element] readonly attribute Element? firstElementChild`.

_Open question: we could also consider just exporting the entire prototype chain in each module, so that e.g. `"std:global/Element"`'s exports would be a superset of `"std:global/Node"`'s. We worry a bit about the bloat there, though._

### Limitations

Are there originals that programs may want to access, but which this proposal does not provide? Our answer so far is "theoretically yes, and in practice no." We list the cases here, and discuss what it would take to add them at a later time.

- **Constants** ([#9](https://github.com/domenic/get-originals/issues/9)). We could easily add constants, but have chosen not too, as they are easily replicated by user code just using the constant values directly. We are open to revisiting this.

- **Static getters/setters**. We do not know of any of these in existence in either the web or TC39 specifications. Adding them would be a simple matter of deciding whether the suffix is `_static_get` or `_get_static`.

- **Symbol-named methods**. Methods like `Date.prototype[Symbol.toPrimitive]()` or `NodeList.prototype[Symbol.iterator]()` are not included in this proposal. In most cases symbol-named properties provide esoteric functionality or functionality that can easily be achieved through other means (such as iteration). Adding these later would involve a name-mangling scheme for transating built-in symbol names to export names (e.g. `Symbol.toPrimitive` → `toPrimitive_symbol` or similar).

- **Defined-in-JavaScript global functions**. The functions in the [Function Properties of the Global Object](https://tc39.github.io/ecma262/#sec-function-properties-of-the-global-object) section of the JavaScript specification are not accounted for in the API presented here. (Functions defined via web specifications, such as `window.alert()`, are retrievable via e.g. `"std:global/Window"`.) They are all of dubious utility or have better replacements: for example, the `URL` class replaces `encodeURIComponent()`, and `Number.parseInt()` replaces `parseInt()`. So, they are omitted for now. If we added them later, we could add them as named exports of `"std:global"`.

- **Not-on-the-global classes and objects** ([#15](https://github.com/domenic/get-originals/issues/15)). The deprecated [`[NoInterfaceObject]`](https://heycam.github.io/webidl/#NoInterfaceObject) Web IDL extended attribute allows the creation of classes that are not exposed on the global object. Similarly, the ECMAScript spec, as well as Web IDL's binding layer, specify a variety of classes and prototype objects which are not exposed anywhere: examples include the [`GeneratorFunction` class](https://tc39.github.io/ecma262/#sec-generatorfunction-constructor), [`%IteratorPrototype%`](https://tc39.github.io/ecma262/#sec-%iteratorprototype%-object), or [iterator prototype objects](https://heycam.github.io/webidl/#dfn-iterator-prototype-object).

  In most cases, the reasons these have not been exposed is because they are not very useful directly, so the motivation for including them in get-originals is low. If we did want to expose them, the easiest way would be to give them global names and have them flow into get-originals as normal. ([Example](https://github.com/tc39/proposal-iterator-helpers#iterator-helpers).) If that is not an option, we could consider one-off "modulifications", e.g. introducing `"std:hidden/GeneratorFunction"`, and reusing the get-originals spec infrastructure to the extent possible.

## Specification plans

We expect that a formal specification for this API, after its initial incubation, will be included as part of [Web IDL](https://heycam.github.io/webidl/).

In particular, during the steps where Web IDL currently [defines the global property references](https://heycam.github.io/webidl/#define-the-global-property-references), we would insert steps which define new [synthetic module records](https://proposal-javascript-standard-library-domenic.now.sh/#sec-synthetic-module-records) to install into the realm's [module map](https://html.spec.whatwg.org/multipage/webappapis.html#concept-settings-object-module-map). This would take care of all Web IDL-defined globals.

There are two main non-Web IDL-defined specifications that also contribute to the web platform's globals: the [Streams Standard](https://streams.spec.whatwg.org/) and, of course, the [JavaScript specification](https://tc39.github.io/ecma262/) itself.

For Streams, [switching to Web IDL](https://github.com/whatwg/streams/issues/963) is already agreed upon, and we think it's best to block any exposure of Streams Standard originals on that work completing.

For JavaScript, there are ambitions [to use an IDL](https://github.com/tc39/proposal-idl), probably Web IDL, to reframe all existing globals. This would be the optimal outcome. However, we'll probably need an interim solution while that work continues. Such a solution would be specified in a separate section of the Web IDL spec, and would probably operate by iterating over clause titles from the JavaScript specification: for example, the clauses on [Constructor Properties of the Global Object](https://tc39.github.io/ecma262/#sec-constructor-properties-of-the-global-object) and [Other Properties of the Global Object](https://tc39.github.io/ecma262/#sec-other-properties-of-the-global-object).

Note that in the specifications, it will probably prove convenient to eagerly fill the module map with all of these modules, during realm creation. Implementations will likely choose a more lazy approach, but the observable consequences will be identical either way.

## Automatic rewriting/enforcement of robustness

Given how safe meta-programming is so difficult to do by hand, we think that most usage of the get-originals API will be via tooling. For WebAssembly, this will presumably be part of their usual compilation toolchain. For JavaScript, we anticipate working on tools such as:

- A transpiler, which takes idiomatic code and converts all method and property access of built-in objects into get-originals usage. This could be greatly helped by a type system, so e.g. this may work well as a [TypeScript](https://www.typescriptlang.org/) plugin. (But, note that an unsound type system like TypeScript would prevent us from getting 100% correctness guarantees.)

- A checker, which uses heuristics to scan source code for dangerous patterns, such as the use of the `.` operator for property access. This could be done as e.g. an [ESLint](https://eslint.org/) plugin.

- A chaos monkey, which mutates as many built-ins as it can then attempts to run your tests in the mutated environment, to verify resilience.

## Exposing other originals

### Built-in module originals

The proposal here is so far focused entirely on the existing global standard library. What about exposing the originals for parts of the standard library which are placed in [built-in modules](https://github.com/tc39/proposal-javascript-standard-library/)?

We think that this could be done as a fairly straightforward extension of this proposal's mechanisms, with slightly different entry points. For example, [`"std:kv-storage"`](https://github.com/WICG/kv-storage) could expose its originals via a dedicated `"std:kv-storage/originals/X"` namespace, leading to code such as

```js
import { apply } from "std:global/Reflect";
import { storage } from "std:kv-storage";
import { get as StorageArea_get } from "std:kv-storage/originals/StorageArea";

const promise = apply(StorageArea_get, storage, ["someKey"]);
```

We'll focus on globals first, but especially as the work on [defining built-in modules via Web IDL](https://github.com/heycam/webidl/pull/675) continues, we anticipate extending get-originals to built-in modules to be a straightforward extension, should the need arise.

### Author code originals

Similarly, author code may wish to provide robust and WebAssembly-accessible entry points, in the same way that get-originals does for the web platform. This can be done via the same conventional pattern:

TODO this doesn't work due to circular dependency ordering being imperfect. Do something a little more complicated.

```js
// my-library/index.mjs

import "./originals/AwesomeStuff.mjs"; // (*)

export class AwesomeStuff {
  doAwesome() { }
  static awesomeness() { }
}
```

```js
// my-library/originals/AwesomeStuff.mjs
import { AwesomeStuff } from "../index.mjs";

export default AwesomeStuff;
export const doAwesome = AwesomeStuff.prototype.doAwesome;
export const awesomeness_static = AwesomeStuff.awesomeness;
```

Here the line marked with a (*) is included so that any consumers of `my-library/index.mjs` will automatically trigger the... no wait this won't work.
