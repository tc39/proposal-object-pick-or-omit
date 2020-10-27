## `Object.prototype.pick` / `Object.prototype.omit`

ECMAScript Proposal, specs, and reference implementation for `Object.prototype.pick` and `Object.prototype.omit`.

Spec drafted by [@Aleen](https://github.com/aleen42).

This proposal is currently [stage 1](https://github.com/tc39/proposals/) of the [process](https://tc39.github.io/process-document/).

### Motivation

- To operate an object convenient by picking or omitting its properties, describe in [the group](https://es.discourse.group/t/object-prototype-pick-object-prototype-omit/515).

### Syntax

```
obj.pick(pickedKeys)
obj.omit(omittedKeys) 
```

#### Parameters

- `pickedKeys`: keys of properties you want to pick from the object.
- `omittedKeys`: keys of properties you want to pick from the object.

#### Returns

Returns a new object, which has picked or omitted properties from the original one.

### Usage

```js
({a : 1, b : 2}).pick(['a']); // => {a : 1}
({a : 1, b : 2}).omit(['b']); // => {b : 1}

({a : 1, b : 2}).pick(['c']); // => {}
({a : 1, b : 2}).omit(['c']); // => {a : 1, b : 2}
```
