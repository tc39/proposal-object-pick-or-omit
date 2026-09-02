## `Object.{pick, omit}`
> ECMAScript Proposal, specs, and reference implementation for `Object.pick`, `Object.omit`.

__Authors:__ [@Aleen](https://github.com/aleen42) & [Hemanth HM](https://github.com/hemanth)

__Champions:__ [Hemanth HM](https://github.com/hemanth), [Jordan Harband](https://github.com/ljharb), [@js-choi](https://github.com/js-choi)

This proposal is currently [Stage 1](https://github.com/tc39/proposals/blob/master/README.md) of the [TC39 Process](https://tc39.es/process-document/), presented at the [July 2022 TC39 meeting](https://github.com/tc39/notes/blob/main/meetings/2022-07/jul-21.md#ergonomic-dynamic-object-restructuring).

---

### Motivation

Creating new objects by picking or omitting specific properties from an existing object is one of the most common operations in modern JavaScript:

* **Event handling:** On a `MouseEvent`, extracting only `['ctrlKey', 'shiftKey', 'altKey', 'metaKey']`.
* **Config extraction:** From a `package.json` config object, picking `['dependencies', 'devDependencies', 'peerDependencies']`.
* **Options bags:** Whitelisting `['shell', 'env', 'extendEnv', 'uid', 'gid']` from user input.
* **HTTP / API payload filtering:** From `req.body`, extracting `['name', 'company', 'email', 'password']`.
* **Component memoization / props diffing:** Extracting `compareKeys` from `props` and comparing them against `prevProps`.
* **Sanitization:** Stripping sensitive or internal fields (e.g. `['_csrf', '_method']`, passwords, or tokens) before logging or persisting.
* **Dynamic CLI flags:** Filtering configuration objects when the key list is supplied dynamically at runtime.

Today, developers frequently reach for utility libraries such as Lodash (`_.pick`, `_.omit`, `_.pickBy`, `_.omitBy`), or hand-roll ad-hoc implementations:

```js
const pick = (obj, keys) => Object.fromEntries(
  keys.filter(k => k in obj).map(k => [k, obj[k]])
);

const omit = (obj, keys) => Object.fromEntries(
  Object.entries(obj).filter(([k]) => !keys.includes(k))
);
```

These ad-hoc approaches suffer from notable shortcomings:
- **Ergonomics & boilerplate:** Verbose, repetitive, and error-prone.
- **Performance:** Inefficient intermediate array allocations (`Object.entries`, `filter`, `fromEntries`).
- **Edge cases:** Inconsistent handling of prototype properties, non-enumerable properties, symbol keys, and getters.

---

### The Problem Space: Static vs. Dynamic Restructuring

Feedback from the TC39 committee (July 2022 plenary) highlighted that object restructuring spans two distinct categories:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Object Restructuring                              │
├──────────────────────────────────────┬──────────────────────────────────────┤
│ 1. Static Restructuring              │ 2. Dynamic Restructuring & Predicates│
├──────────────────────────────────────┼──────────────────────────────────────┤
│ • Known, fixed property keys at      │ • Keys computed dynamically at       │
│   authoring time.                    │   runtime (arrays, iterables, sets). │
│ • Ergonomics gap:                    │ • Predicate filtering functions:     │
│   const {a, b} = o; return {a, b};   │   (value, key, obj) => boolean       │
│ • Best addressed by syntax:          │ • Capability gap: cannot be solved   │
│   o.{a, b}                           │   by static syntax alone.            │
│ • Engine & JIT friendly (ICs, shape  │ • Natural fit for static methods:    │
│   transitions, static analysis, TS). │   Object.pick(o, keysOrPredicate)    │
│                                      │   Object.omit(o, keysOrPredicate)    │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

#### 1. Static Restructuring (Ergonomics Gap)
When the keys are statically known, destructuring today requires repeating identifiers:
```js
const { a, b, c } = options;
const selected = { a, b, c };
```
For static omissions, rest destructuring already provides a native solution:
```js
const { secret, token, ...safeUser } = user;
```
Explorations for syntax-level static picking (e.g. `obj.{a, b}`) directly target this ergonomic gap while preserving inline caches (ICs) and static analyzability.

#### 2. Dynamic Restructuring and Predicate Filtering (Capability Gap)
When the keys are dynamically provided at runtime (e.g., from a variable, CLI flags, configuration arrays) or determined by a predicate function, static syntax cannot help. An expression like `o.{[...keys]}` would merely be a runtime method in disguise.

Dynamic property extraction is a true capability gap in JavaScript. Static methods `Object.pick` and `Object.omit` provide the canonical, standard library solution for this space.

---

### Syntax & API

```ts
Object.pick(obj, keysOrPredicate)
Object.omit(obj, keysOrPredicate)
```

#### Parameters

- **`obj`**: The source object. Coerced via `ToObject(obj)` (throws `TypeError` if `null` or `undefined`).
- **`keysOrPredicate`** *(optional)*:
  - **Iterable of keys**: An iterable (`Array`, `Set`, etc.) yielding property keys (`string` or `symbol`).
  - **Predicate function**: A callable `(value, key, object) => boolean` invoked for each enumerable own property.

#### Returns

A new plain object (`%Object.prototype%`) containing the picked or non-omitted properties.

---

### Usage Examples

#### Picking and Omitting with Key Lists

```js
const user = { id: 1, name: 'Alice', email: 'alice@example.com', role: 'admin' };

// Pick specific keys
Object.pick(user, ['id', 'name']);
// => { id: 1, name: 'Alice' }

// Omit specific keys
Object.omit(user, ['email', 'role']);
// => { id: 1, name: 'Alice' }

// Symbols are fully supported
const sym = Symbol('meta');
const record = { a: 1, [sym]: 2 };
Object.pick(record, [sym]);
// => { [sym]: 2 }
```

#### Picking and Omitting with Predicates

```js
const scores = { a: 10, b: 25, c: 5, d: 40 };

// Pick properties satisfying a condition
Object.pick(scores, (val, key) => val >= 20);
// => { b: 25, d: 40 }

// Omit properties satisfying a condition
Object.omit(scores, (val, key) => key === 'c' || val < 10);
// => { a: 10, b: 25, d: 40 }
```

---

### Syntax Exploration (Visions)

As discussed during Stage 1, syntax notations could complement static methods for compile-time/static property extraction:

1. **Dot-brace syntax (`.{}`)**:
   ```js
   const subset = obj.{ a, b };
   // Equivalent to: { a: obj.a, b: obj.b }

   // Renaming and defaults:
   const renamed = obj.{ a, b: alias, c = defaultValue };
   ```

2. **Index pick notation (`.[]`)**:
   ```js
   const subset = obj.['a', 'b'];
   ```

---

### FAQ & Semantics

#### 1. How are prototype properties handled?
Consistent with destructuring, `Object.pick` with explicit keys reads properties via `[[Get]]` (allowing inherited properties to be picked), while `Object.omit` operates on own enumerable properties (you cannot omit/delete an inherited prototype property on a newly cloned object):

```js
Object.pick([1, 2, 3], ['length']); // => { length: 3 }
// Equivalent to destructuring:
const { length } = [1, 2, 3]; // length === 3

// Omitting cannot delete inherited prototype properties:
const arr = [1, 2, 3];
Object.omit(arr, ['length']); // => {} (length is non-enumerable own on Array, not copied)

// __proto__ follows standard property access rules:
Object.pick({}, ['__proto__']); // => { __proto__: Object.prototype }
const { __proto__ } = {};       // __proto__ === Object.prototype
```

#### 2. What is the return value type?
Consistent with destructuring and object spread, `Object.pick` and `Object.omit` always return plain objects inheriting from `Object.prototype`:

```js
Object.pick([]);        // => {}
Object.omit([]);        // => {}
Object.pick(new Map()); // => {}

// Equivalent to object spread / destructuring behavior:
const { ...res } = [];
res instanceof Array;   // => false
```

#### 3. How are `Symbol` keys handled?
`Symbol` keys are first-class property keys in ECMAScript and are handled identically to string keys:

```js
const sym = Symbol('key');
Object.pick({ a: 1, [sym]: 2 }, [sym]); // => { [sym]: 2 }
Object.omit({ a: 1, [sym]: 2 }, [sym]); // => { a: 1 }
```

#### 4. What happens when property access throws an error (e.g., throwing getter)?
Consistent with destructuring and `Object.assign`, property getters are executed during property access; any thrown exception propagates immediately:

```js
const o = Object.defineProperty({}, 'key', {
  get() { throw new Error('custom error'); }
});

// Throws Error: 'custom error'
Object.pick(o, ['key']);
```

#### 5. Why are these static methods on `Object` rather than `Object.prototype`?
Defining methods on `Object.prototype` risks breaking existing web code with name collisions (e.g. objects or libraries already using `.pick` or `.omit`). Additionally, `Object.create(null)` objects would be unable to use prototype methods.

#### 6. Why accept both key lists and predicate functions in `Object.pick`/`Object.omit`?
Combining key lists and predicates under `Object.pick` and `Object.omit` keeps the API surface minimal and intuitive. Predicates cleanly handle filtering by value or dynamic conditions without requiring separate `pickBy`/`omitBy` methods.

---

### Related Specifications & Prior Art

* [TC39 July 2022 Meeting Notes](https://github.com/tc39/notes/blob/main/meetings/2022-07/jul-21.md#ergonomic-dynamic-object-restructuring)
* [TypeScript Utility Types: `Pick<T, Keys>` and `Omit<T, Keys>`](https://www.typescriptlang.org/docs/handbook/utility-types.html)
* [Lodash `_.pick`, `_.omit`, `_.pickBy`, `_.omitBy`](https://lodash.com/docs/4.17.15)
* [TC39 Proposal: Shorthand Improvements](https://github.com/rbuckton/proposal-shorthand-improvements)
* [TC39 Proposal: Pattern Matching](https://github.com/tc39/proposal-pattern-matching)
* [TC39 Proposal: Extractors](https://github.com/tc39/proposal-extractors)

