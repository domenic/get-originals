# Accessing the originals

A proposal for a web platform API that allows access to the "original" versions of built-in objects' properties and methods.

## Motivation

The [layered APIs](https://github.com/drufball/layered-apis) proposal, for creating new higher-level APIs, imposes a requirement that such APIs use no "magic" capabilities that are not exposed to JavaScript developers.

One important technique used when writing robust APIs is to call the underlying operation, not the publicly-modifiable API. For example, the [async local storage](https://domenic.github.io/async-local-storage/) layered API specification says:

> Let _request_ be the result of performing the steps listed in the description of [`IDBObjectStore`](https://w3c.github.io/IndexedDB/#idbobjectstore)'s [`get()`](https://w3c.github.io/IndexedDB/#dom-idbobjectstore-get) method on _store_, given the argument _key_.

and

> If [IsArray](https://tc39.github.io/ecma262/#sec-isarray)(_value_) is true, return true.

This phrasing ensures that, even if the layered API is being invoked in a context where `IDBObjectStore.prototype.get` or `Array.isArray` have been mucked with, the specification is still able to run normally. This is important for allowing implementations more internal flexibility in how they implement the specification, without their every move being interceptable by JavaScript code running in the page.

However, invoking these underlying operations is not an ability currently available to web developers. Thus, invoking them violates the layered API constraint of not using any un-layered magic.

This proposal provides a series of methods for accessing the "original" values of JavaScript and web platform built-ins, thus exposing this capability to web developers. In turn, that allows layered API specifications to use such phrases as the above, without violating their layering constraint.

In addition to unlocking the layered API work, we envision this capability being generally useful for libraries interested in being robust against diverse environments. Beyond the web platform, Node.js may also be interested in implementing this API, so that they can write their built-in modules in a robust fashion.

_This problem space was previously discussed in [drufball/layered-apis#6](https://github.com/drufball/layered-apis/issues/6), which you may enjoy reading for more background._

## Example usage

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
2. Getting a global constructor (`TypeError`)
3. Referencing the global (`self`)
4. Getting a property (`self.indexedDB`, `request.result`)
5. Calling a method (`indexedDB.open()`)
6. Setting a property (`request.onsuccess`)

Here is how we would rewrite the above code to be robust, while using the proposed get-originals API:

```js
const oArrayIsArray = getOriginalStatic("Array", "isArray");
const oTypeError = getOriginalConstructor("TypeError");
const oIndexedDB = getOriginalProperty(originalSelf, "indexedDB");

function storeIt(value) {
  if (!oArrayIsArray(value)) {
    throw new oTypeError("Must be an array!");
  }

  const request = callOriginalMethod(oIndexedDB, "open", "my-db", 1);

  setOriginalProperty(request, "onsuccess", () => {
    const database = getOriginalProperty(request, "result");
    // ...
  });
}
```

## The API

We propose adding the following APIs to the global:

- `originalSelf`
- `getOriginalConstructor("name")`
- `callOriginalStaticMethod("base", "name", ...args)`
- `getOriginalProperty(obj, "name")`
- `setOriginalProperty(obj, "name", newValue)`
- `callOriginalMethod(obj, "name", ...args)`

All of these are non-writable and non-configurable (i.e. unforgeable), and available in all globals. In Web IDL:

```webidl
interface mixin GetOriginals {
  [Unforgeable] readonly attribute object originalSelf;
  [Unforgeable] any getOriginalConstructor(DOMString name);
  [Unforgeable] any callOriginalStaticMethod(DOMString base, DOMString name, any... args);
  [Unforgeable] any getOriginalProperty(any target, DOMString name);
  [Unforgeable] void setOriginalProperty(any target, DOMString name, any newValue);
  [Unforgeable] any callOriginalMethod(any target, DOMString name, any... args);
}

Window includes GetOriginals;
WorkerGlobalScope includes GetOriginals;
WorkletGlobalScope includes GetOriginals;
```

The intention is these APIs work equally well for the ECMAScript specification built-ins as they do for web platform built-ins. Web developers using these APIs should not need to know the difference. This has some implementation complexities; the shape of the API was specifically chosen to try to mitigate this, as we will discuss below.

We discuss the details more in the following sections:

### `originalSelf`

This is simply an unforgeable version of the `self` global. It is mainly envisioned to be used in conjunction with `getOriginalProperty()`, `setOriginalProperty()`, and `callOriginalMethod()`. For example, a non-monkeypatchable way to open a popup window would be with

```js
callOriginalMethod(originalSelf, "open", "https://example.com/");
```

or you could get access to various globals via code like

```js
const oNavigator = getOriginalProperty(originalSelf, "navigator");
const oParent = getOriginalProperty(originalSelf, "parent");
// ...
```

### `getOriginalConstructor(name)`

This method retrieves the original versions of various "constructors" that are available on the global object.

Concretely, the supplied `name` here can be one of the following:

* All Web IDL interfaces that are [exposed](https://heycam.github.io/webidl/#dfn-exposed) in the current realm;
* The [constructor properties of the global object](https://tc39.github.io/ecma262/#sec-constructor-properties-of-the-global-object) listed in the JavaScript specification;
* The [global properties](https://streams.spec.whatwg.org/#globals) listed in the Streams Standard

If the given `name` does not identify one of these values, or the value identified is not implemented by the current user agent, the function must return `undefined`.

The return value must be identical (`===`) to the original value. For example, in a non-monkeypatched environment, it must be the case that

```js
getOriginalConstructor("TypeError") === TypeError;
getOriginalConstructor("Node") === Node;
// ...
```

Note that this will not retrieve _any_ global property: for example, `getOriginalConstructor("status")`, `getOriginalConstructor("NaN")`, or `getOriginalConstructor("JSON")` will all return `undefined`. It only retrieves a small set of constructors. For the rest, use `getOriginalProperty()`.

**Implementation strategy:** We anticipate this being implemented with a lookup table, assembled at the same time as the realm is being created. This lookup table would gather contributions from both JS engine startup code and Web IDL bindings code.

### `callOriginalStaticMethod(base, name, ...args)`

This method calls various "static" methods that are available on various global constructors and namespaces.

Concretely, the supplied `base` here is a string that can be one of the following:

* All Web IDL interfaces and namespaces that are [exposed](https://heycam.github.io/webidl/#dfn-exposed) in the current realm;
* The [constructor properties of the global object](https://tc39.github.io/ecma262/#sec-constructor-properties-of-the-global-object) listed in the JavaScript specification;
* The [other properties of the global object](https://tc39.github.io/ecma262/#sec-other-properties-of-the-global-object) listed in the JavaScript specification;
* The [global properties](https://streams.spec.whatwg.org/#globals) listed in the Streams Standard

And the supplied `name` can be one of the following:

* All static operations defined on such Web IDL interfaces;
* All regular operations defined on such Web IDL namespaces;
* All method "properties of the X constructor" defined in the corresponding section of the JavaScript specification (e.g., [`ArrayBuffer.isView()`](https://tc39.github.io/ecma262/#sec-arraybuffer.isview) is a method in a section titled "[Properties of the ArrayBuffer Constructor](https://tc39.github.io/ecma262/#sec-properties-of-the-arraybuffer-constructor));
* All method properties of these "other properties of the global object" listed in the JavaScript specification (e.g., [`JSON.parse()`](https://tc39.github.io/ecma262/#sec-json.parse) is a method in the "[The JSON Object](https://tc39.github.io/ecma262/#sec-json-object)" section, which is linked to from "other properties of the global object");
* All method "properties of the X constructor" defined in the corresponding section of the Streams Standard (none exist at this time)

If the given `name` or `base` do not identify one of these values, or the specified value is not implemented by the current user agent, the function must throw a `TypeError`.

**Implementation strategy**: We anticipate this being implemented with a combination of the lookup table used for `getOriginalConstructor()`, with implementations' existing abilities to find an execute the backing algorithm behind a given static method.

### `getOriginalProperty(target, name)`/`setOriginalProperty(target, name, value)`/`callOriginalMethod(target, name, ...args)`

These methods allow the following `target` values:

* Any Web IDL platform object, except the global object;
* Any of the classes defined in the JavaScript specification which perform brand checks (see [tc39/ecma262#354](https://github.com/tc39/ecma262/issues/354) for formalization);
* All object types defined in the Streams Standard which perform brand checks.

Calling any of these methods on an invalid `target` value must throw a `TypeError`.

These methods allow the following property/method `name` values:

* All regular attributes and operations of Web IDL interfaces, when applied to the corresponding Web IDL platform object;
* All "properties of the X prototype object" defined in the JavaScript specification (both methods and accessors), when applied to an instance of one of the classes defined there;
* All "properties of the X prototype object" defined in the Streams Standard (both methods and accessors), when applied to an instance of one of the classes defined there;
* For the global object, additionally, all "[function properties of the global object](https://tc39.github.io/ecma262/#sec-function-properties-of-the-global-object)" defined in the JavaScript specification.

Calling `getOriginalProperty()` with an invalid `name` value will return `undefined`. Calling `setOriginalProperty()` with an invalid `name` value will do nothing. Calling `callOriginalMethod()` with an invalid `name` value will throw a `TypeError`. Here, "invalid" includes being in the wrong category, e.g.

```js
getOriginalProperty(originalSelf, "close");    // returns undefined: Window's close() is an operation
setOriginalProperty(originalSelf, "toolbar");  // does nothing: Window's toolbar is readonly
callOriginalMethod(originalSelf, "status");    // throws a TypeError: Window's status is an attribute
```

**Implementation strategy**:

* For Web IDL platform objects, the idea is to unwrap to the underlying C++/Rust property/method, and get/set/call it. Validation that that property/method exists might be tricky?
* For JavaScript objects, the idea is to determine their brand and call the appropriate original method/getter/setter code. Hopefully this can be done with some sort of switch over the brand slot?
* The global object might need some special casing since it's both. If that's too hard, we can eliminate the "function properties of the global object" clause.

## API discussion

### Motivation for this API surface

The primary motivations for this API surface are:

1. Be implementable efficiently
2. Do not require authors to understand what specification a class/method/etc. is defined in.

These are somewhat in conflict, as the natural implementation strategies for acessing the originals of values provided by the JS engine are different than those provided by Web IDL bindings. We think we have threaded the needle relatively well, but welcome feedback.

In particular, the requirement to be efficiently implementable explains this API's focus on categorization. For example:

* Instead of treating statics as original properties or methods of their constructor, we carved out a separate `callOriginalStaticMethod()` method.
* Instead of treating all globals equally, we specifically selected the "constructors" subset to be accessible via `getOriginalConstructor()`.
* Instead of directly exposing the original function objects in all cases, e.g. allowing retrieval of the original value of `window.close`, or of the getter function for `window.status`, we only did that for the specific case of constructors, which need to create objects that will pass `instanceof` and similar checks.

An [earlier version](https://github.com/domenic/get-originals/tree/dab86957e9f736d0884bf1f3370962ec5a383108#the-api) was more minimal, but when we tried to envision how it would be implemented, we realized it would be quite difficult. The current version attempts to do better.

### Instance-based design rationale

Some might find APIs that identify a property or method using a string more natural than what we have proposed. For example,

```js
const sliced = callOriginalMethod(
  "Array.prototype.slice",
  /* thisArg = */[1, 2, 3],
  /* args... = */1, 5
  );
```

This was indeed part of our [original proposal](https://github.com/drufball/layered-apis/issues/6); however, @bzbarsky pointed out that this introduces excessive fragility into the ecosystem.

In particular, it prevents future refactoring in the platform where we move properties between prototypes. Imagine if we introduced some sort of `IndexedCollection` base class for `Array`, and moved `slice()` to `IndexedCollection`. Then, the above code, and any web sites that used it, would not work.

Another issue is with regard to the existing lack of interoperability around various prototype chains on the platform. For example, currently [it is not very clear what belongs on `HTMLDocument` vs. `Document`](https://github.com/whatwg/dom/issues/221), and different browsers have different answers (all of which differ from the spec). The above API would force web developers to know which of `HTMLDocument.prototype` or `Document.prototype` a given property or method is on, which could vary per browser. Whereas with the proposed API, they would just use the `document` instance, and it would work either way.

You can imagine various workarounds. For example, maintaining a separate "original prototype chain" which you can do lookups on, so that even in that hypothetical future, `"Array.prototype.slice"` can be treated the same way as `"IndexedCollection.prototype.slice"`. But this seems like a lot of work, for very little gain. Especially since the instance-based approach works so nicely, at least for Web IDL-backed objects.

### Method calls with variadic arguments

Both `callOriginalMethod()` and `callOriginalStaticMethod()` take the arguments to be passed variadically, after the identifying information for the method in question.

The reason for preferring this over using an arguments array (as seen in e.g. JavaScript's `Reflect.call`) is that array literals are not safe from monkeypatching. Pretend for a moment we used arrays for `callOriginalMethod()`. Then we have this unfortunate scenario:

```js
// Author code:
Object.defineProperty(Array.prototype, Symbol.iterator, {
  get() {
    throw new Error("boo!");
  }
});

// Library code, trying to be robust:
callOriginalMethod(originalSelf, "open", ["https://example.com/"]);
```

The call to `callOriginalMethod()` here will throw in the process of processing the `["https://example.com/"]` argument. The attempts to be robust have failed.

Avoiding this problem involves bootstrapping your way to a "safe array" constructor, whose iterator is hand-coded to return the array elements appropriately. This is quite difficult, not in the least because until you have this in hand, you can't effectively use `callOriginalMethod()` or `callOriginalStaticMethod()`, so the boostrap process itself becomes harder. We think it might be technically doable, but it certainly won't be easy. Instead, we think using variadic arguments for `callOriginalMethod()` and `callOriginalStaticMethod()` is much simpler.

### Same values

One important question is whether the "originals" returned by these methods have the same _object identity_ as the originals, or just the same behavior.

Because of our API design, this question only arises for one method: `getOriginalConstructor()`. In all other cases the "original" being used is not actually returned to script; it is simply invoked. This should allow easier implementation, by going directly to the backing algorithm, instead of having to save away the appropriate JavaScript function object.

But for `getOriginalConstructor()`, it's important to give the exact original constructor, because of cases like this:

```js
// Library code:
const o_Promise = getOriginalConstructor("Promise");

function delay(ms) {
  return new o_Promise(resolve => {
    callOriginalMethod(originalSelf, "setTimeout", resolve, ms)
  });
}
```

```js
// Consumer code:
console.assert(delay(500) instanceof Promise);
```

That is, if `getOriginalConstructor("Promise")` did not have the same identity as the existing global `Promise`, consumers would be able to easily observe the difference, in a way that would be undesireable. This example generalizes to any constructor.

### Things that do not work

The above sections explain how each part of this API is geared toward accessing specific originals. Here we give a series of examples of things that, according to those rules, do _not_ work. This should hopefully make the boundaries of the API clearer:

```js
// Constructor properties of the global: use getOriginalConstructor() instead.

getOriginalProperty(originalSelf, "TypeError");   // undefined
getOriginalProperty(originalSelf, "Node")         // undefined


// Methods: use callOriginalMethod() instead.

getOriginalProperty(originalSelf, "close");       // undefined
getOriginalProperty(originalSelf, "encodeURI");   // undefined
getOriginalProperty(someNode, "isEqualNode");     // undefined


// Static methods: use callOriginalStaticMethod() instead.

getOriginalProperty(Array, "isArray");            // undefined
callOriginalMethod(Number, "isNaN", 5);           // TypeError


// Non-method statics: use the actual constant values instead

getOriginalProperty(Number, "MAX_SAFE_INTEGER");  // undefined
getOriginalProperty(Math, "PI");                  // undefined
getOriginalProperty(Node, "COMMENT_NODE");        // undefined
getOriginalProperty(originalSelf, "NaN");         // undefined
```

## Automatic rewriting/enforcement of robustness

It looks like you would do something like:

- Don't allow any direct access to globals: must use `getOriginalConstructor()` or `getOriginalProperty(originalSelf, ...)`. Easy to detect by checking for unknown bindings.
- First pass: don't allow any `x.y`-based property access
  - It's never safe on browser-provided objects
  - Even for author-created objects you should probably be using their private state (that you control), not public API
  - If you really need an escape hatch, we could allow `x["y"]`, or some kind of `// I know what I'm doing` comment
- Can we track which objects are browser-created and automatically rewrite them to use this pattern?

Looking at the [non-robust async local storage implementation](https://github.com/domenic/async-local-storage/blob/312d0fa31e6864b41df82119a9468db18c54ac19/prototype/implementation.mjs), it seems like every `x.y` property access should be converted to use the originals... except maybe `.length` on arrays. The array manipulation in general needs a bit more care.

## Future extension: let author code expose its own originals

TODO: Figure out how/whether this can work, especially with the instance-based design.
