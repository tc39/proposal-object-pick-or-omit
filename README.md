## `Object.pick` / `Object.omit`

ECMAScript Proposal, specs, and reference implementation for `Object.pick` and `Object.omit`.

Spec drafted by [@Aleen](https://github.com/aleen42).

### Motivation

- To operate an object convenient by picking or omitting its properties, described in [the group](https://es.discourse.group/t/object-prototype-pick-object-prototype-omit/515).

### Syntax

```
Object.pick(obj, pickedKeys)
Object.omit(obj, omittedKeys) 
```

#### Parameters

- `pickedKeys`: keys of properties you want to pick from the object.
- `omittedKeys`: keys of properties you want to pick from the object.

#### Returns

Returns a new object, which has picked or omitted properties from the object.

### Usage

```js
Object.pick({a : 1, b : 2}, ['a']); // => {a : 1}
Object.omit({a : 1, b : 2}, ['b']); // => {b : 1}

Object.pick({a : 1, b : 2}, ['c']); // => {}
Object.omit({a : 1, b : 2}, ['c']); // => {a : 1, b : 2}
```

### FAQ

1. When it comes to the prototype chain of an object, should the method pick or omit it?

    A: The implementation of [`_.pick`](https://lodash.com/docs/4.17.15#pick) and [`_.omit`](https://lodash.com/docs/4.17.15#omit) by Lodash has taken care about the chain:

    ```js
    _.pick({a : 1}, ['toString']); // => {toString: f}
    _.omit({a : 1}, ['toString']).toString; // => ƒ toString() { [native code] }
    ```

2. In comparison with [**proposal-shorthand-improvements**](https://github.com/rbuckton/proposal-shorthand-improvements), when should we use these two methods?

    A: Multiple properties. Assume that we need to ensure an object without any side-effected keys except `key1` and `key2`:

    ```js
    postData({[key1] : o[key1], [key2] : o[key2]});
    postData(Object.pick(o, [key1, key2])); 
    ```

3. Why can't be defined on the `Object.prototype` directly?

    A: As `Object` is especially fundamental, and both of them will result in conflicts of properties of any other objects. In shorthand, if defined, any objects inherited from `Object` with `pick` or `omit` defined in its prototype should break.
