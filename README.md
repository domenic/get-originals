# Accessing the originals

A proposal for a web platform API that allows access to the "original" versions of built-in objects' properties and methods.

## Motivation

The [layered APIs](https://github.com/drufball/layered-apis) proposal, for creating new higher-level APIs, imposes a requirement that such APIs use no "magic" capabilities that are not exposed to JavaScript developers.

One important technique used when writing robust APIs is to call the underlying operation, not the publicly-modifiable API. For example, the [async local storage](https://domenic.github.io/async-local-storage/) layered API specification says:

> Let _request_ be the result of performing the steps listed in the description of [`IDBObjectStore`](https://w3c.github.io/IndexedDB/#idbobjectstore)'s [`get()`](https://w3c.github.io/IndexedDB/#dom-idbobjectstore-get) method on _store_, given the argument _key_.

and

> If [IsArray](https://tc39.github.io/ecma262/#sec-isarray)(_value_) is true, return true.

This phrasing ensures that, even if the layered API is being invoked in a context where `IDBObjectStore.prototype.get` or `Array.isArray` have been mucked with, the specification is still able to run normally. This is important for allowing implementations more internal flexibility in how they implement the specification, without their every move being interceptable by JavaScript code runnign in the page.

However, calling these underlying operations is not an ability currently available to web developers. Thus, it violates the layered API constraint of not using any un-layered magic.

This proposal provides a series of methods for getting the "original" values of JavaScript and web platform built-ins, thus exposing this capability to web developers. In turn, that allows layered API specifications to use such phrases as the above, without violating their layering constraint.

In addition to unlocking the layered API work, we envision this capability being generally useful for libraries interested in being robust against diverse environments. Beyond the web platform, Node.js may also be interested in implementing this API, so that they can write their built-in modules in a robust fashion.

_This problem space was previously discussed in [drufball/layered-apis#6](https://github.com/drufball/layered-apis/issues/6), which you may enjoy reading for more background._

## Example usage

Consider what it would take to make the following code robust:

```js
function storeIt(value) {
  if (!Array.isArray(value)) {                // (1) (2)
    throw new TypeError("Must be an array!"); // (1)
  }

  const request = indexedDB.open("my-db", 1); // (1) (3)

  request.onsuccess = () => {                 // (5)
    const database = request.result;          // (4)
    // ...
  });
}
```

The numbered lines point to the various operations we need to be able to do without triggering any potential monkeypatches:

1. Getting/using a global property (`Array`, `TypeError`, `indexedDB`)
2. Getting/calling a static method (`isArray`)
3. Getting/calling an instance method (`open`, `addEventListener`)
4. Invoking an instance getter (`result`)
5. Invoking an instance setter (`onsuccess`)

Here is how we would rewrite the above code to be robust, while using the proposed get-originals API:

```js
const Array = getOriginalGlobal("Array");
const TypeError = getOriginalGlobal("TypeError");
const indexedDB = getOriginalGlobal("indexedDB");

function storeIt(value) {
  if (!callOriginalMethod(Array, "isArray", value)) {
    throw new TypeError("Must be an array!");
  }

  const request = callOriginalMethod(indexedDB, "open", "my-db", 1);

  setOriginalProperty(request, "onsuccess", () => {
    const database = getOriginalProperty(request, "result");
    // ...
  });
}
```

## The API

We propose adding the following APIs to the global:

- `getOriginalGlobal("name")`
- `getOriginalProperty(obj, "propertyName")`
- `setOriginalProperty(obj, "propertyName", newValue)`
- `callOriginalMethod(obj, "methodName", ...args)`

All of these are non-writable and non-configurable (i.e. unforgeable), and available in all globals. In Web IDL:

```webidl
interface mixin GetOriginals {
  [Unforgeable] any getOriginalGlobal(DOMString name);
  [Unforgeable] any getOriginalProperty(any target, DOMString propertyName);
  [Unforgeable] void setOriginalProperty(any target, DOMString propertyName, any newValue);
  [Unforgeable] any callOriginalMethod(any target, DOMString methodName, any... args);
}

Window includes GetOriginals;
WorkerGlobalScope includes GetOriginals;
WorkletGlobalScope includes GetOriginals;
```

The intention is these APIs work equally well for the ECMAScript specification built-ins as they do for web platform built-ins. Web developers using these APIs should not need to know the difference. (This has some implementation complexities; see below.)

We discuss the details more in the following sections:

### `getOriginalGlobal()`

The supplied name here can be any property of the global object. This includes:

* All Web IDL interface and namespace names that are [exposed](https://heycam.github.io/webidl/#dfn-exposed) in the current realm;
* The [properties of the global object](https://tc39.github.io/ecma262/#sec-global-object) listed in the JavaScript specification;
* The [global properties](https://streams.spec.whatwg.org/#globals) listed in the Streams Standard;
* The various Web IDL attributes and methods listed on the current global object (e.g. `close()` and `document` in `Window` globals), and in its prototype chain (e.g. `addEventListener` for globals that inherit from `EventTarget`).

Essentially, everything a web developer could access is available here.

Examples:

```js
const o_Array = getOriginalGlobal("Array");
const o_Text = getOriginalGlobal("Text");
const o_ReadableStream = getOriginalGlobal("ReadableStream");
const o_JSON = getOriginalGlobal("JSON");
const o_CSS = getOriginalGlobal("CSS");
const o_document = getOriginalGlobal("document");
const o_postMessage = getOriginalGlobal("postMessage");
const o_addEventListener = getOriginalGlobal("addEventListener");
```

### `getOriginalProperty()`/`setOriginalProperty()`/`callOriginalMethod()`

These methods allow the following target objects:

* Any of the class or namespace objects returned by `getOriginalGlobal()`, so that you can get their static properties;
* Any Web IDL platform object, except the global object;
* Any of the classes defined in the JavaScript specification which perform brand checks (see [tc39/ecma262#354](https://github.com/tc39/ecma262/issues/354) for formalization);
* All object types defined in the Streams Standard which perform brand checks.

They allow the following target property/method names:

* All static attributes and operations of Web IDL interfaces or namespaces, when applied to the corresponding interface or namespace object;
* All "properties of the X constructor" defined in the JavaScript specification, when applied to one of the classes defined there;
* All function properties of the namespaces defined in the JavaScript specification;
* All regular attributes and operations of Web IDL interfaces, when applied to the corresponding Web IDL platform object
* All "properties of the X prototype object" defined in the JavaScript specification, when applied to an instance of one of the classes defined there.

If the property/method name is not in this list, the result will be `undefined`.

`callOriginalMethod()` accepts the same list of properties as `getOriginalProperty()`/`setOriginalProperty()`, but as usual trying to call a non-function will throw: e.g. `callOriginalMethod(window, "status")` will try to call a string and throw.

We recognize this isn't fully formalized, especially for JavaScript. We look forward to turning it into a full specification.

Note that we explicitly disallow the global object, as it has many properties (e.g. constructors) that are not from conventional sources. You should use `getOriginalGlobal()` for those. But, see the discussion below.

Examples:

```js
// Assume we have already gotten original globals:
// o_console, o_Response, o_Array, o_document

const o_log = getOriginalProperty(o_console, "log");
const o_redirect = getOriginalProperty(o_Response, "redirect");

const isAnArray = callOriginalMethod(o_Array, "isArray", [1, 2, 3]);

const o_body = getOriginalProperty(o_document, "body");
callOriginalMethod(o_body, "setAttribute", "bgcolor", "red");

const target = getOriginalProperty(someEvent, "target");

setOriginalProperty(o_document, "cookie", "foo=bar");
```

## API discussion

### Implementation strategies

For `getOriginalGlobal()`, the intention is for implementations to store the originals in parallel to installing them on the global object, or to snapshot them after setting up the global object. This should be done for every property of the global object.

_Open question:_ what kind of overhead would this add, either in memory or in startup time? Is there an alternative implementation strategy that would cost less resources? If so, would it require changing the API in some way? (For example, you could imagine that if the API were restricted to Web IDL interface and namespace objects, an implementation might already have a table of those available.)

For the methods that operate on properties of various objects, the implementation strategy branches depending on the target:

* For Web IDL platform objects, simply unwrap the JavaScript object to get at the underlying C++/Rust/etc. property/method, and get/set/call it.
* For JavaScript built-in types, perform a series of brand checks? Can this be done as a switch statement that jumps to the right branch, or would this require O(number of JS spec objects) tests? Then, once you know what type it is, is it easy to call the original getter/setter/method?
* For accessing static properties/methods of constructors and namespaces, can we even identify these objects? They won't have any brand installed on them...

_Open questions:_ As you can see, the above points to potential implementation difficulties. Consultation with implementers is necessary. It seems plausible that static properties/methods might need a separate dedicated API, e.g. `getOriginalStatic("Array", "isArray")`.

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

### Same values?

One important question is whether the "originals" returned by these methods have the same _object identity_ as the originals, or just the same behavior. That is, assuming the web developer has not modified any built-ins, are the following true?

```js
getOriginalGlobal("Promise") === Promise;
getOriginalProperty(Promise, "resolve") === Promise.resolve;
```

If this is not required, an implementation would be able to generate new JS "wrappers" for a given global/property/function on the fly, which could reduce the need to save all the originals in memory.

At least for `getOriginalGlobal()`, we think it's required to preserve identity. The reason is the following:

```js
// Library code:
const o_Promise = getOriginalGlobal("Promise");
const o_setTimeout = getOriginalGlobal("setTimeout");

function delay(ms) {
  return new o_Promise(resolve => o_setTimeout(resolve, ms));
}
```

```js
// Consumer code:
console.assert(delay(500) instanceof Promise);
```

That is, if `getOriginalGlobal("Promise")` did not have the same identity as the existing global `Promise`, consumers would be able to easily observe the difference, in a way that would be undesireable. This example generalizes to any constructor.

It's less clear how important identity preservation is for `getOriginalProperty()`. As long as `getOriginalProperty(Promise, "resolve")` behaves the same as `Promise.resolve`—including returning instances of the global `Promise` constructor—then we don't really care if it is exactly the same function. But, it would probably be pretty confusing for developers to allow such divergence?

### Do we need...?

The current API is somewhat of a mid-point between minimizing the number of methods and being easier to use each method. The below two sections explore ways we could go toward a more truly minimal API, and explain why we've currently landed on the above.

#### Do we need `getOriginalGlobal()`?

If we added a way to get the original global this value, and changed the definition of `getOriginalProperty()`, we could eliminate `getOriginalGlobal()`. For example, instead of

```js
const Array = getOriginalGlobal("Array");
```

you would do

```js
const Array = getOriginalProperty(originalSelf, "Array");
```

where `originalSelf` is a new non-configurable non-writable property on all globals that returns the current global this value. Or, we could make `getOriginalProperty(null, "Array")` mean to get it from the global.

The complexity here is that the set of properties of the global object is assembled in a rather different way from other objects. Essentially, it has a lot of non-method data properties, installed by the JavaScript and Web IDL specifications. Thus, for example, the implementation strategy outlined for Web IDL platform objects above would not work, even though the global is a Web IDL platform object. As such we've tentatively kept `getOriginalGlobal()` as a separate method.

#### Do we need `callOriginalMethod()`?

We could eliminate `callOriginalMethod()` in favor of making people use (the original version of) `Reflect.call`, e.g. instead of

```js
const request = callOriginalMethod(indexedDB, "open", "my-db", 1);
```

you would do

```js
const Reflect = getOriginalGlobal("Reflect");
const Reflect_call = getOriginalProperty(Reflect, "call");

const request = Reflect_call(indexedDB, "open", ["my-db", 1]);
```

Actually you would need to do more than that, because array literals are not safe:

```js
const Object = getOriginalGlobal("Object");
const Object_setPrototypeOf = getOriginalProperty(Object, "setPrototypeOf");

const safeArray(arr) {
  Object_setPrototypeOf(arr, null);
  return arr;
}

const request = Reflect_call(indexedDB, safeArray(["open", 1]));
```

This is kind of terrible though, so it'd be nice to give people `callOriginalMethod()`. Specifically, we designed `callOriginalMethod()` to take the method arguments as positional parameters, instead of an array, to avoid the extra hassle there.

## Automatic rewriting/enforcement of robustness

It looks like you would do something like:

- Don't allow any direct access to globals: must use `getOriginalGlobal()`. Easy to detect by checking for unknown bindings.
- First pass: don't allow any `x.y`-based property access
  - It's never safe on browser-provided objects
  - Even for author-created objects you should probably be using their private state (that you control), not public API
  - If you really need an escape hatch, we could allow `x["y"]`, or some kind of `// I know what I'm doing` comment
- Can we track which objects are browser-created and automatically rewrite them to use this pattern?

Looking at the [non-robust async local storage implementation](https://github.com/domenic/async-local-storage/blob/312d0fa31e6864b41df82119a9468db18c54ac19/prototype/implementation.mjs), it seems like every `x.y` property access should be converted to use the originals... except maybe `.length` on arrays. The array manipulation in general needs a bit more care and may not be doable without observable property access.

## Future extension: let author code expose its own originals

TODO: Figure out how/whether this can work, especially with the instance-based design.
