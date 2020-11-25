## `Object.{pick,pickBy}` / `Object.{omit,omitBy}`

ECMAScript Proposal, specs, and reference implementation for `Object.pick`, `Object.pickBy`, `Object.omit`, and `Object.omitBy`.

Spec drafted by [@Aleen](https://github.com/aleen42).

### Motivation

- To operate an object convenient by picking or omitting its properties, described in [the group](https://es.discourse.group/t/object-prototype-pick-object-prototype-omit/515).

### Syntax

```
Object.pick(obj, pickedKeys)
Object.pickBy(obj, predictedFunction(currentValue[, key[, object]])[, thisArg])

Object.omit(obj, omittedKeys)
Object.omitBy(obj, predictedFunction(currentValue[, key[, object]])[, thisArg])
```

#### Parameters

- `pickedKeys`: keys of properties you want to pick from the object.
- `omittedKeys`: keys of properties you want to pick from the object.
- `predictedFunction`: the function to predicted whether the property should be picked or omitted.
  - `currentValue`: the current value processed in the object.
  - `key`: the key of the `currentValue` in the object.
  - `object`: the object `pickBy` or `omitBy` was called upon.
- `thisArg`: the object used as `this` inside the predicted function.

#### Returns

Returns a new object, which has picked or omitted properties from the object.

### Usage

```js
Object.pick({a : 1, b : 2}, ['a']); // => {a: 1}
Object.omit({a : 1, b : 2}, ['b']); // => {b: 1}

Object.pick({a : 1, b : 2}, ['c']); // => {}
Object.omit({a : 1, b : 2}, ['c']); // => {a: 1, b: 2}

Object.pick([], [Symbol.iterator]); // => {Symbol(Symbol.iterator): f}
Object.pick([], ['length']); // => {length: 0}

Object.pickBy({a : 1, b : 2}, v => v === 1); // => {a: 1}
Object.omitBy({a : 1, b : 2}, v => v === 2); // => {a: 1}
Object.pickBy({a : 1, b : 2}, (v, k) => k === 'a'); // => {a: 1}
Object.omitBy({a : 1, b : 2}, (v, k) => k === 'b'); // => {a: 1}
```

### FAQ

1. When it comes to the prototype chain of an object, should the method pick or omit it?

    A: The implementation of [`_.pick`](https://lodash.com/docs/4.17.15#pick) and [`_.omit`](https://lodash.com/docs/4.17.15#omit) by Lodash has taken care about the chain:

    ```js
    _.pick({a : 1}, ['toString']); // => {toString: f}
    _.omit({a : 1}, ['toString']).toString; // => ƒ toString() { [native code] }
    ```
2. If some properties of an object is not accessible like throwing an error, can `Object.pick` or `Object.omit` operate such an object?

    A: I suggest throwing the error wrapped by `Object.pick` or `Object.omit`, but it is **NOT the final choice**:

    ```js
    Object.pick(Object.defineProperty({}, 'key', {
	    get() { throw new Error() }
    }), ['key']);
    ```

    The error stack will look like:

    ```
    Uncaught Error
        at Object.get (<anonymous>:2:20)
        at Object.pick (<anonymous>:2:10)
        at <anonymous>:1:8 
    ```

3. In comparison with [**proposal-shorthand-improvements**](https://github.com/rbuckton/proposal-shorthand-improvements), when should we use these two methods?

    A: Multiple properties. Assume that we need to ensure an object without any side-effected keys except `key1` and `key2`:

    ```js
    postData({[key1] : o[key1], [key2] : o[key2]});
    postData(Object.pick(o, [key1, key2])); 
    ```

4. Why can't be defined on the `Object.prototype` directly?

    A: As `Object` is especially fundamental, and both of them will result in conflicts of properties of any other objects. In shorthand, if defined, any objects inherited from `Object` with `pick` or `omit` defined in its prototype should break.

