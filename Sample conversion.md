# Sample conversion to use get-originals

This shows what some code that uses get-originals would look like, by taking existing "normal" JS code and converting it to use get-originals so we can do a before/after comparisons.

## Async local storage (partial)

This example compares some parts of an [async local storage](https://github.com/domenic/async-local-storage) JS implementation.

Note that some of the `WeakMap` manipulation would be shorter, in both versions, by using [private fields](https://tc39.github.io/proposal-class-fields/). But, we leave them as-is for now.

### Before

```js
const databaseName = new WeakMap();
const databasePromise = new WeakMap();

if (!self.isSecureContext) {
  throw new DOMException("Async local storage is only available in secure contexts", "SecurityError");
}

export class StorageArea {
  constructor(name) {
    databasePromise.set(this, null);
    databaseName.set(this, `async-local-storage:${name}`);
  }

  async entries() {
    return performDatabaseOperation(this, "readonly", (transaction, store) => {
      const keysRequest = store.getAllKeys();
      const valuesRequest = store.getAll();

      return new Promise((resolve, reject) => {
        keysRequest.onerror = () => reject(keysRequest.error);
        valuesRequest.onerror = () => reject(valuesRequest.error);

        valuesRequest.onsuccess = () => {
          resolve(zip(keysRequest.result, valuesRequest.result));
        };
      });
    });
  }

  // ... other methods elided ...
}

function performDatabaseOperation(area, mode, steps) {
  // ... elided ...
}

function zip(a, b) {
  const result = [];
  for (let i = 0; i < a.length; ++i) {
    result.push([a[i], b[i]]);
  }

  return result;
}
```

### After

```js
const WeakMap = getOriginalConstructor("WeakMap");
const Promise = getOriginalConstructor("Promise");
const DOMException = getOriginalConstructor("DOMException");

const databaseName = new WeakMap();
const databasePromise = new WeakMap();

if (!getOriginalProperty(originalSelf, "isSecureContext")) {
  throw new DOMException("Async local storage is only available in secure contexts", "SecurityError");
}

export class StorageArea {
  constructor(name) {
    callOriginalMethod(databasePromise, "set", this, null);
    callOriginalMethod(databaseName, "set", this, `async-local-storage:${name}`);
  }

  async entries() {
    return performDatabaseOperation(this, "readonly", (transaction, store) => {
      const keysRequest = callOriginalMethod(store, "getAllKeys");
      const valuesRequest = callOriginalMethod(store, "getAll");

      return new Promise((resolve, reject) => {
        setOriginalProperty(keysRequest, "onerror", () => reject(getOriginalProperty(keysRequest, "error")));
        setOriginalProperty(valuesRequest, "onerror", () => reject(getOriginalProperty(valuesRequest, "error")));

        setOriginalProperty(valuesRequest, "onsuccess", () => {
          resolve(zip(getOriginalProperty(keysRequest, "result"), getOriginalProperty(valuesRequest, "result")));
        });
      });
    });
  }

  // ... other methods elided ...
}

function performDatabaseOperation(area, mode, steps) {
  // ... elided ...
}

function zip(a, b) {
  const result = [];
  for (let i = 0; i < a.length; ++i) {
    callOriginalStaticMethod("Object", "defineProperty", result, i, {
      value: [a[i], b[i]],
      enumerable: true,
      configurable: true,
      writable: true
    });
  }

  return result;
}
```

### Array manipulation analysis

The most interesting conversion above was the `zip()` function. Some notes:

- We took advantage of the fact that the inputs, `a` and `b`, were known to be real, user-agent-provided `Array` instances, with no holes and with a truthful `length` property. This means that any indexed access to them, within the range 0 through length, is safe from prototype pollution.

- Pushing onto the result array is a fairly arduous process, necessitating the use of (the original version of) `Object.defineProperty()`. Simple indexed assignment (i.e. `result[i] = [a[i], b[i]]`) would not do, since that could trigger setters that someone installed on `Array.prototype`.

- Array literals, such as `[a[i], b[i]]` are safe to use; they do not trigger any prototype setters.

### Alternate `zip()` using an "internal array"

Consider the following helper:

```js
const internalArrayProto = callOriginalStaticMethod(Object, "create", null);
internalArrayProto.toRealArray = function () {
  const result = [];
  for (let i = 0; i < this.length; ++i) {
    callOriginalStaticMethod("Object", "defineProperty", result, i, {
      value: this[i],
      enumerable: true,
      configurable: true,
      writable: true
    });
  }

  return result;
};

function createInternalArray() {
  const arr = [];
  callOriginalStaticMethod("Object", "setPrototypeOf", arr, internalArrayProto);
  return arr;
}
```

This would allow us to write the following `zip()` implementation:

```js
function zip(a, b) {
  const resultInternal = createInternalArray();
  for (let i = 0; i < a.length; ++i) {
    resultInternal[i] = [a[i], b[i]];
  }

  return resultInternal.toRealArray();
}
```

This is nicer to read and write, although it is definitely more expensive as it results in two iterations (one being the copy from `resultInternal` to the return real array).

This may point to a missing language-level primitive, similar to V8's [%MoveArrayContents](https://cs.chromium.org/chromium/src/v8/src/js/intl.js?l=627&rcl=4fe3de13fc5c2592559bb5e49e1a81a6d09a2efd), that would allow us to "move" `resultInternal` into `result`. But, that's quite tricky: e.g. we'd need a notion of what happens to `resultInternal` after it gets moved.

## Iterable processing

Consider a toy example to illustate iterable processing. Note that, [like all web platform features](https://heycam.github.io/webidl/#create-sequence-from-iterable), we intentionally accept any input iterable, and call the possibly-monkeypatched iteration protocol methods (e.g. `[Symbol.iterator]()`, `next()`) on it. So usage of `for`-`of` on the _input_ side is not problematic. Instead, as we'll see, the issue is usage of iteration features as part of our internal implementation.

Note that in both of the below "After" examples, we use the verbose original-`Object.defineProperty()` call instead of using internal arrays. A similar transformation as was shown for `zip()` above can be applied, in both cases.

### Before

```js
function unique(array) {
  const set = new Set(array); // iteration here is OK per the above
  return [...set];
}
```

The danger to keep in mind here while converting is that we cannot spread a `Set` syntactically, as in `[...set]`, without opening ourselves up to tampering. Syntactic spread always calls the (possibly non-original) `Set.prototype[Symbol.iterator]()`.

### After, manual iteration

A straightforward translation that unrolls the iteration would be as follows:

```js
const Set = getOriginalConstructor("Set");

function unique(array) {
  const set = new Set(array); // iteration here is OK per the above

  const valuesIter = callOriginalMethod(set, "values");
  let current = callOriginalMethod(valuesIter, "next");

  const result = [];
  while (current.done !== true) {
    callOriginalStaticMethod("Object", "defineProperty", result, i, {
      value: current.value,
      enumerable: true,
      configurable: true,
      writable: true
    });
    current = callOriginalMethod(valuesIter, "next");
  }
  return result;
}
```

### After, another approach

Another approach is to rethink the algorithm entirely and try to do things in one pass, by using lots of set-membership testing:

```js
const Set = getOriginalConstructor("Set");

function unique(array) {
  const seenSoFar = new Set();

  const result = [];
  let currentResultIndex = 0;
  for (const arrayItem of array) {  // iteration here is OK per the above
    if (!callOriginalMethod(seenSoFar, "has", arrayItem)) {
      callOriginalStaticMethod("Object", "defineProperty", result, currentResultIndex, {
        value: arrayItem,
        enumerable: true,
        configurable: true,
        writable: true
      });
      ++currentResultIndex;
    }
  }

  return result;
}
```

As an aside, I'm not sure whether this would be more or less efficient in practice. Its worst-case computational complexity is O(<var>n</var> log(<var>n</var>)), whereas the previous version is O(<var>n</var>), but it only does a single pass, and in practice `has()` may be closer to O(1) than O(log(<var>n</var>)).
