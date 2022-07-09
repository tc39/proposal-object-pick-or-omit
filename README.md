## `Object.{pick, omit}`
> ECMAScript Proposal, specs, and reference implementation for `Object.pick`, `Object.omit`.

Spec drafted by [@Aleen](https://github.com/aleen42).

### Motivation

Let us consider a few scenarios from the real world to understand what we are trying to solve in this proposal. 

* On `MouseEvent` we are intreseted on `'ctrlKey', 'shiftKey', 'altKey', 'metaKey'` events only.
* We have a `configObject` and we need `['dependencies', 'devDependencies', 'peerDependencies']` from it. 
* We have an `optionsBag`and we would allow on `['shell', 'env', 'extendEnv', 'uid', 'gid']` on it.
* From a `req.body` we want to extract `['name', 'company', 'email', 'password']`
* Checking if a component `shouldReload` by extracting `compareKeys` from `props` and compare it with `prevProps`. 
* Say we have a `depsObject` and we need to ignore all `@internal/packages` from it.
* We have `props` from which we need to remove `[‘_csrf’, ‘_method’]`
* We need to construct a `newModelData` by removing `action.deleted` from `({ ...state.models, ...action.update })`
* Filtering configuration objects when the filter list is given by a `CLI` argument.

Well, you see life is all about `pick`ing what we want and `omit`ing what we don't! 

Would life be easier if the language provided a convenient method to help us during similar scenarios?

Now, one might argue saying we can implement `pick` and `omit` as below:

```js
const pick = (obj, keys) => Object.fromEntries(
    keys.map(k => obj.hasOwnProperty(k) && [k, obj[k]]).filter(x => x)
);

/*
We can also use a Destructuring assignment
const { authKey, ...toLog } = userInfo;
*/
```

```js
const omit = (obj, keys) => Object.fromEntries(
    keys.map(k => !obj.hasOwnProperty(k) && [k, obj[k]]).filter(x => x)
);
```

The major challenges we see with the above implementations:

* It is not ergonomic!
* If we opt for the destructuring way it doesn't work at all for `pick`, or for `omit` with dynamic values.

We can read more about such use-cases and challenges from `es.discourse` below:

* [Object.{pick,omit}](https://es.discourse.group/t/object-prototype-pick-object-prototype-omit/515).
* [Object restructuring syntax](https://es.discourse.group/t/object-restructuring-syntax/651)
* [Object Array Pick](https://es.discourse.group/t/object-array-pick/992)
* [js-pick-notation](https://github.com/rtm/js-pick-notation)
* [slect multiple object values](https://es.discourse.group/t/set-multiple-object-values-a-b-undefined/1052/3)

With that in mind would it not be easier if we had `Object.pick` and `Object.omit` static methods?!

Let us now discuss what the API of such a helpful method would be?

### Syntax

```
Object.pick(obj[, pickedKeys | predictedFunction(currentValue[, key[, object]])[, thisArg])
Object.omit(obj[, omittedKeys | predictedFunction(currentValue[, key[, object]])[, thisArg])
```

#### Parameters

- `obj`: which object you want to pick or omit.
- `pickedKeys` (**optional**): keys of properties you want to pick from the object. The default value is an empty array.
- `omittedKeys` (**optional**): keys of properties you want to pick from the object. The default value is an empty array.
- `predictedFunction` (**optional**): the function to predicted whether the property should be picked or omitted. The default value is an identity: `x => x`.
  - `currentValue`: the current value processed in the object.
  - `key`: the key of the `currentValue` in the object.
  - `object`: the object `pick` was called upon.
- `thisArg` (**optional**): the object used as `this` inside the predicted function.

#### Returns

- Returns a new object, which has picked or omitted properties from the object.

### Usage

```js
// default
Object.pick({a : 1}); // => {}
Object.omit({a : 1}); // => {a: 1}
Object.pick({a : 0, b : 1}, v => v); // => {b: 1}
Object.pick({a : 0, b : 1}, v => !v); // => {a: 0}
Object.pick({}, function () { console.log(this) }); // => the object itself
Object.pick({}, function () { console.log(this) }, window); // => Window

Object.pick({a : 1, b : 2}, ['a']); // => {a: 1}
Object.omit({a : 1, b : 2}, ['b']); // => {a: 1}

Object.pick({a : 1, b : 2}, ['c']); // => {}
Object.omit({a : 1, b : 2}, ['c']); // => {a: 1, b: 2}

Object.pick([], [Symbol.iterator]); // => {Symbol(Symbol.iterator): f}
Object.pick([], ['length']); // => {length: 0}

Object.pick({a : 1, b : 2}, v => v === 1); // => {a: 1}
Object.pick({a : 1, b : 2}, v => v !== 2); // => {a: 1}
Object.pick({a : 1, b : 2}, (v, k) => k === 'a'); // => {a: 1}
Object.pick({a : 1, b : 2}, (v, k) => k !== 'b'); // => {a: 1}
```

### Visions

1. A syntax sugar in the case of picking:

    To extend the motivation of this proposal, there may be some syntax notations as an alternative of picking properties from objects, like the proposal, [proposal-slice-notation](https://github.com/tc39/proposal-slice-notation):

    There are two ideas around how to wrap picking keys:

    - square brackets:

        ```js
        ({a : 1, b : 2, c : 3}).['a', 'b']; // => {a : 1, b : 2}

        const keys = ['a', 'b'];
        ({a : 1, b : 2, c : 3}).[keys[0], keys[1]]; // => {a : 1, b : 2}
        ({a : 1, b : 2, c : 3}).[...keys]; // => {a : 1, b : 2}
        ```

    - curly brackets

        ```js
        ({a : 1, b : 2, c : 3}).{a, b} // => {a : 1, b : 2}

        const keys = ['a', 'b'];
        ({a : 1, b : 2, c : 3}).{[keys[0]], b}; // => {a : 1}
        ({a : 1, b : 2, c : 3}).{[...keys]}; // => {a : 1, b : 2}

        // Similar to destructuring
        ({a : 1, b : 2, c : 3}).{a, b : B}; // => {a : 1, B : 2}
        ```

        Currently, there is a disagreement on whether properties with default assignment values should be picked.

        ```js
        // If considering the meaning of picking, the initial value has no meanings
        ({a : 1, b : 2, c : 3}).{a, d = 2}; // => {a : 1}

        // If considering as "restructuring", the shortcut has its reason to pick
        ({a : 1, b : 2, c : 3}).{a, d = 2}; // => {a : 1, d : 2}
        ```

    Nevertheless, it is just a simple vision, and feel free to discuss.

### FAQ

1. When it comes to the prototype chain of an object, should the method pick or omit it? (The answer may change)

    A: The implementation of [`_.pick`](https://lodash.com/docs/4.17.15#pick) and [`_.omit`](https://lodash.com/docs/4.17.15#omit) by Lodash has taken care about the chain. To keep the rule, we can pick off properties of prototype, but can't omit them:

    ```js
    Object.pick({a : 1}, ['toString']); // => {toString: f}
    Object.omit({a : 1}, ['toString']).toString; // => ƒ toString() { [native code] }
    ```

    The same rule applies to `__proto__ event if it has been [deprecated](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto), because the proposal should be pure enough to not specify a special logic to eliminate deprecated properties:

    ```js
    Object.pick({}, ['__proto__']); // => {__proto__: {...}}
    Object.omit({}, ['__proto__']).__proto__; // => {...} 
    ```

    In some opinions, picking off or omitting properties from the prototype chain should make the method more extendable:

    ```js
    const pickOwn = (obj, keys) => Object.pick(obj, keys.filter(key => obj.hasOwnProperty(key)));
    const omitOwn = (obj, keys) => Object.omit(obj, keys.filter(key => obj.hasOwnProperty(key)));
    ```

2. What is the type of the returned value?

    A: All these methods should return plain objects:

    ```js
    Object.pick([]); // => {} 
    Object.omit([]); // => {}
    Object.pick(new Map()); // => {} 
    Object.omit(new Map()); // => {}
    ```

3. How to handle `Symbol`?

    A: `Symbol` should just be considered as properties within `Symbol` keys, and they should obey the rules mentioned above:

    ```js
    Object.pick([], [Symbol.iterator]); // => {Symbol(Symbol.iterator): f}, pick off from the prototype
    Object.omit([], [Symbol.iterator]); // => {}, plain objects
    
    const symbol = Symbol('key');
    Object.omit({a : 1, [symbol]: 2}, [symbol]); // => {a : 1}
    
    Object.prototype[symbol] = 'test'; // override prototype
    Object.pick({}, [symbol]); // => {Symbol(key): "test"}, pick off from the prototype
    Object.omit({}, [symbol])[symbol]; // => "test", cannot omit properties from the prototype
    ```

4. If some properties of an object are not accessible like throwing an error, can `Object.pick` or `Object.omit` operate such an object?

    A: I suggest throwing the error wrapped by `Object.pick` or `Object.omit`, but it is **NOT the final choice**:

    ```js
    Object.pick(Object.defineProperty({}, 'key', {
       get() { throw new Error() }
    }), ['key']);
    ```

    The error stack will look like this:

    ```
    Uncaught Error
        at Object.get (<anonymous>:2:20)
        at Object.pick (<anonymous>:2:10)
        at <anonymous>:1:8 
    ```

5. In comparison with [**proposal-shorthand-improvements**](https://github.com/rbuckton/proposal-shorthand-improvements), when should we use these two methods?

    A: Multiple properties. Assume that we need to ensure an object without any side-effected keys except `key1` and `key2`:

    ```js
    postData({[key1] : o[key1], [key2] : o[key2]});
    postData(Object.pick(o, [key1, key2])); 
    ```

6. Why can't be defined on the `Object.prototype` directly?

    A: As `Object` is especially fundamental, and both of them will result in conflicts of properties of any other objects. In shorthand, if defined, any objects inherited from `Object` with `pick` or `omit` defined in its prototype should break.

7. Why not define filtered methods corresponding to two actions: [`pickBy`](https://lodash.com/docs/4.17.15#pickBy) and [`omitBy`](https://lodash.com/docs/4.17.15#omitBy) like Lodash?

    A: It is [unnecessary](https://github.com/aleen42/proposal-object-pick-or-omit/issues/2) to double two methods, because it can be combined into the argument instead:

    Besides, the passing filtered method can be easily reversed with equal meaning, and it means that `omitBy` can be easily defined as `pickBy`'s inverse.

    ```js
    Object.pick({a : 1, b : 2}, v => v);

    // Equivalent to the following:
    Object.omitBy({a: 1, b : 2}, v => !v);
    ```

***Notice: If you have any suggestions or ideas about this proposal? Appreciate your discussions via issues.***
