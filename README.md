# Accessing the originals

A proposal for a web platform API that allows access to the "original" versions of built-in objects' properties and methods.

## Motivation

TODO explain. See https://github.com/drufball/layered-apis/issues/6.

## Example usage

Consider what it would take to make the following code robust:

```js
function storeIt(value) {
  if (!Array.isArray(value)) {                // (1) (2)
    throw new TypeError("Must be an array!"); // (1)
  }

  const request = indexedDB.open("my-db", 1); // (3)

  request.onsuccess = () => {                 // (5)
    const database = request.result;          // (4)
    // ...
  });
}
```

The numbered lines point to the various operations we need to be able to do without triggering any potential monkeypatches:

1. Getting/using a global constructor (`Array`, `TypeError`)
2. Getting/calling a static method (`isArray`)
3. Getting/calling an instance method (`open`, `addEventListener`)
4. Invoking an instance getter (`result`)
5. Invoking an instance setter (`indexedDB`, `onsuccess`)

Here is how we would rewrite the above code to be robust, while using the get-originals API:

```js
const Array = getOriginalGlobal("Array");
const TypeError = getOriginalGlobal("TypeError");
const indexedDB = getOriginalProperty(originalSelf, "indexedDB");

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

- `originalSelf`
- `getOriginalGlobal("name")`
- `getOriginalProperty(obj, "propertyName")`
- `setOriginalProperty(obj, "propertyName", newValue)`
- `callOriginalMethod(obj, "methodName", ...args)`

All of these are non-writable and non-configurable (i.e. unforgeable), and available in all globals. In Web IDL:

```webidl
[Exposed=(Window,Worker,Worklet)]
interface mixin GetOriginals {
  [Unforgeable] readonly attribute object originalSelf;
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

The separation into "globals", "object properties", and "object methods" is not based in the JavaScript object model. Rather, it is based on a logical separation. For example, even though in the JavaScript object model `composedPath` is a property of `Event` instances, it is not exposed through `getOriginalProperty()`, and you have to use `callOriginalMethod()` instead. Similarly, although `Node` is a property of the global object, you must use `getOriginalGlobal()` to access it.

We discuss the details more in the following sections:

### `originalSelf`

This is simply an unforgeable version of the global `self` property. In Window contexts, it is redundant with the global `window` property, which is already unforgeable.

It is mainly used in conjunction with `getOriginalProperty()` to access important objects stored as global properties, e.g. `document` via `getOriginalProperty(originalSelf, "document")`.

### `getOriginalGlobal()`

Roughly speaking, the name argument is meant to be a class or namespace name. Concretely, we would allow:

* All Web IDL interface and namespace names that are [exposed](https://heycam.github.io/webidl/#dfn-exposed) in the current realm;
* The [constructor properties of the global object](https://tc39.github.io/ecma262/#sec-constructor-properties-of-the-global-object) listed in the JavaScript specification;
* The [other properties of the global object](https://tc39.github.io/ecma262/#sec-other-properties-of-the-global-object) listed in the JavaScript specification;
* The [global properties](https://streams.spec.whatwg.org/#globals) listed in the Streams Standard.

### `getOriginalProperty()`/`setOriginalProperty()`

These methods allow the following target objects:

* Any of the class or namespace objects returned by `getOriginalGlobal()`, so that you can get their static properties;
* Any Web IDL platform object (including the global object);
* JavaScript `Date`, `RegExp`, `Map`, `Set`, `WeakMap`, `WeakSet` `ArrayBuffer`, `SharedArrayBuffer`, `DataView`, typed array, and `Promise` objects;
* All object types defined in the Streams Standard (including the non-globally-exposd ones).

They allow the following target property names:

* All static attributes of Web IDL interface or namespace objects;
* All non-function "properties of the X constructor" defined in the JavaScript specification, for the constructor/other properties of the global object defined there;
*

### Scratchwork

Talk about JS spec objects vs. Web IDL spec objects, and how we don't want authors to have to know the difference.

Talk about what `"globalProperty"`, `"methodName"`, and `"propertyName"` values will work.

Talk about implementation strategies (unwrapping Web IDL objects... harder for JS objects)

Is the name "original" right? For example some properties might change their _values_ over time, so we're not getting the original _values_; maybe the word is misleading?

### Open questions

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

- Don't allow any direct access to globals: must use `getOriginalGlobal`. Easy to detect by checking for unknown bindings.
- First pass: don't allow any `x.y`-based property access
  - It's never safe on browser-provided objects
  - Even for author-created objects you should probably be using their private state (that you control), not public API
  - If you really need an escape hatch, we could allow `x["y"]`, or some kind of `// I know what I'm doing` comment
- Can we track which objects are browser-created and automatically rewrite them to use this pattern?

Looking at the [non-robust async local storage implementation](https://github.com/domenic/async-local-storage/blob/312d0fa31e6864b41df82119a9468db18c54ac19/prototype/implementation.mjs), it seems like every `x.y` property access should be converted to use the originals... except maybe `.length` on arrays. The array manipulation in general needs a bit more care and may not be doable without observable property access.

## Future extension: let author code expose its own originals

Figure out how/whether this can work, especially with the instance-based design.

## Alternatives considered

Go through the various things considered in https://github.com/drufball/layered-apis/issues/6. Especially talk about using instances vs. requiring developers to know the prototype chain.
