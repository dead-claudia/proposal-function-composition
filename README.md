# Function composition operators for JS

Currently, there's a lot of talk about a [possible pipeline operator](https://github.com/tc39/proposal-pipeline-operator) being added. My experience says that it's also quite often we need to compose functions, too. And I'm not alone - there's mountains of library precedent:

- [`lodash.flow` sees over 500K downloads a week](https://npm-stat.com/charts.html?package=lodash.flow), more than even [`lodash.flatmap`](https://npm-stat.com/charts.html?package=lodash.flatmap).
- Underscore has had [`_.compose`](http://underscorejs.org/#compose) since 0.2.0 (released 2009).
- Ramda has had [`R.compose`](https://ramdajs.com/docs/#compose) since the very begninning (0.1.0 was released 2013).
- Redux has had [`compose`](https://redux.js.org/api/compose) since [the 1.0.0 release candidate](https://github.com/reduxjs/redux/commit/6a2730deaf91cc53f358a56e981b31f740e58806).

And of course, there's quite a bit of precedent in other languages, too:

- [Haskell has a built-in `g . f` function that's sugar for `\x -> g (f x)`](https://stackoverflow.com/questions/1475896/haskell-function-composition), and it also has a flipped version `f & g` that's equivalent to the above.
- [It's a common question in OCaml, and all major standard library replacements have something equivalent to Haskell's version](https://stackoverflow.com/questions/16637015/is-there-an-infix-function-composition-operator-in-ocaml)
- Clojure has [`(comp f & fs)`](https://clojuredocs.org/clojure.core/comp) that's nearly identical to most JS library implementations.
- Java's had [`java.util.Function.compose`](https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html#compose-java.util.function.Function-) and its reverse [`java.util.Function.andThen`](https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html#andThen-java.util.function.Function-) since Java 8.
- Julia has [`f ∘ g`](https://docs.julialang.org/en/v1/manual/functions/#Function-composition-and-piping) built into its standard library, too.
- Wikipedia has more examples in other various languages, too: https://en.wikipedia.org/wiki/Function_composition_%28computer_science%29

## Proposal

I propose a single left-associative operator for composition: `a |>> b`. This is roughly equivalent to  `(...args) => g(f(...args))`, but to give it added flexibility and power, you can also construct the result and it invokes the left side accordingly. (The result is still coerced to an object in the end in this case.) So in reality, the actual desugaring would be this:

```js
const _apply = Reflect.apply
const _construct = Reflect.construct
const _defineProperty = Object.defineProperty

function _compose(a, b) {
  if (!%IsCallable(a) && !%IsConstructible(a)) {
    throw new TypeError("Expected left hand side to be callable or constructible.")
  }

  if (!%IsCallable(b)) {
    throw new TypeError("Expected right hand side to be callable.")
  }

  const f = (function () {
    return b(
      new.target === void 0
        ? _apply(a, this, arguments)
        : _construct(a, new.target, arguments)
    )
  })

  _defineProperty(f, "name", {value: ""})
  _defineProperty(f, "length", {value: a.length})
  _markAsNativeSomehow(f)

  return f
}

// const composed = a |>> b |>> c
const composed = _compose(_compose(a, b), c)
```

## Syntax concerns

The syntax is meant to be based on https://github.com/tc39/proposal-pipeline-operator, so any discrepancy from that is not intended, and any concerns with it also carry over to this (like `async`/`await` and optional chaining).
