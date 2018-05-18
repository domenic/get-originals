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

Here is how we would rewrite the above code to be robust, while using the get-originals API:

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

TODO flesh out

- `getOriginalGlobal("globalProperty")`
- `callOriginalMethod(obj, "methodName", ...args)` (but see below)
- `getOriginalProperty(obj, "propertyName")`
- `setOriginalProperty(obj, "propertyName", newValue)`

All `[Unforgeable]`. All available in both windows and workers (and worklets?).

Talk about JS spec objects vs. Web IDL spec objects, and how we don't want authors to have to know the difference.

Talk about what `"globalProperty"`, `"methodName"`, and `"propertyName"` values will work.

Talk about implementation strategies (unwrapping Web IDL objects... harder for JS objects)

Is the name "original" right? For example some properties might change their _values_ over time, so we're not getting the original _values_; maybe the word is misleading?

### Do we need `callOriginalMethod()`?

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
