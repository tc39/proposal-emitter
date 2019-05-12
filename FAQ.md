# FAQ

### How does this relate to Transducers?

The composition model is very similar to and inspired by transducers' approach of creating composable algorithmic transformations that can work across different contexts.

The main difference revolves around the higher-order aspect. This was actually where earlier experiments started. However, this turns out to be too slow. In addition, whilst the reducing signature is quite powerful, the limited model does break down quickly. 

To understand the difference in code terms, instead of using a reducing function like this:

```js
const map = fn => send => (result, input) => { .. }
const filter = fn => send => (result, input) => { .. }
```

The `send` function is passed as the third argument:

```js
const map = fn => (input, i, send) => ..
const filter = fn => (input, i, send) => ..
```

Rather it's more like `{ send }`, as we also have a way to make other things accesible, like `resolve` and `reject`. The lack of these things, as well as other quirks in correctly implementing operators, means basic things like early termination start becoming awkward if you try to just fit everything into reducing functions.

The concept of serialising new output collections from the stream of values using `into` also exists in the ["default reducers"](/API.md#reduce) API.

### How does this relate to Iterators/Generators? 

This was demonstrated above, you can use iteration to step through an Emitter where applicable, e.g:

```js
for (const v of map(d => d + 1, [1,2,3]))
  console.log(v)
// or
[...map(d => d + 1, [1,2,3])]
```

An additional point to highlight is that iterators are objects that look like `{ next, return, throw }`. Emitters look like `{ next, resolve, reject }`. Due to the similiarity, it may seem reasonable to think about enhancing the existing object. This was explored and decided against. It would conflate things in a confusing way e.g. Emitter has a `done` property, and vends `value`, iterator has no `done` property and vends `{ done, value }` tuples. Accordingly, it's a trivial thin wrapping to go from one to the other (see below), and best to consistently align with the Promise-terminology instead (e.g. `resolve` vs `return`). 

```js
[Symbol.iterator](){
  return {
    next: () => ({ value: this.next(), done: this[done] })
  , return: () => ({ value: this.resolve(), done: this[done] })
  , throw: () => ({ value: this.reject(), done: this[done] })
  }
}
```

### How does this relate to `ReadableStream`/`WritableStream`?

There are some similarities, particularly in the API e.g. the way the controller is exposed to functions passed in the constructor. However, these are very specialised for [particular I/O use cases and unsuitable for more general cases](https://github.com/whatwg/streams/blob/master/FAQ.md). As a part of the Emitter design, there are specificially no proliferation of base Emitters like readable/transform/writable Emitters (or other [fragmentations like sources/sinks/pullables/pullers/listenables/listeners](https://github.com/staltz/callbag-basics#terminology)) - only one type to minimise cognitive overhead. They are point-to-point, whereas Emitter handles multiple consumers. They are pull-based, whereas Emitter is push-based. Emitter does have a backpressure mechanism, but it's not as intricate as the options these streams provide (e.g. custom queuing strategy). The `buffer` operator is probably closer, in that nexting a value onto a buffer node, does not immediately send it but queues it up to a watermark. Hence the closest analogy would be Emitter with in-built buffering. It may be possible to further subclass it to provide similar API, but those specific use cases are probably better served by separate primitives. 

Since these, as well as existing Node streams, implement `Symbol.asyncIterator`, they interoperate pretty fine already. That is, you could pull from a readable stream, compose them with the operators, or reduce into a writable stream.

```js
run(response.body, map(..), d => console.log('chunk', d))
run([1,2,3], map(..), reduce(fs.createWriteStream('file.txt')))
```

### How does this relate to Observables?

* Observables are similarly push-based primitives
* Observables do not handle multiple consumers (by design)
* Observables invent new error-handling semantics ("complete/error/done"), which becomes a barrier/challenge in integrating it with the rest of the language
* Observables can be thought of as a higher level abstration than Emitter, one that internally generates a new Emitter on every subscription. 
* Emitter would be closer to what is called a Subject in Observable-land. However, this is also not quite accurate as when you `next` onto an Emitter it's not just directly broadcasting to it's children, but rather lets the Emitter process the value. 
* Emitter is _unidirectional_, all signals go downwards (whether that's values, or an Emitter resolving/rejecting like a Promise would). A child can never affect a parent by design, especially not implicitly. This gives developers guarantees which make them easier to reason about. Observables, and other libraries, are built on a _bidirectional_ architecture. When a subscription is made, a signal shoots upwards, and then values start coming back down. This in part contributes to making them harder to understand/implement. It seems this design is also a by-product of the original method-chaining API: when you have `A.op().op().subscribe()` you're subscribing on the last node, which means a signal has to travel back up somehow. However, with more modern forms of the API using `pipe` or the pipeline operator, as well as the API here, you are explicitly running A into B top-down, rather than from B to A. `compose` also returns something that represents the entire group, not just the last node in the chain, making it unnecessary for signals to travel up as well as down. 

  One use-case that does perhaps become less ergonomic is `until` not automatically tearing down an Emitter somewhere higher up in the chain. However, this seems like a worthy trade-off as it manifests as the constraint of having to be explicit with what you are tearing down by having the condition first (a composed emitter is a single thing, hence tears down .
  
  ```js
  // these will both return an array of three, however the first will not stop the array emitter
  run([...], map(), map(), until(3), reduce([]))
  run(until(3, [...], map(), map(), reduce([])))
  ```

### How does this relate to the Standard Library?

This would be a great candidate to expose via the new standard library, although it does not necessarily need to be blocked by that either. It could also neatly be a new global from which you can destructure the operators from i.e. `const { map, filter, run } = Emitter`.

One example that gets mentioned a lot in the standard library proposal is [the generics](https://github.com/tc39/proposal-javascript-standard-library/blob/master/slides-JS-std-lib-July-2018.pdf)). The purpose of that proposal is about developing the infrastructure rather than the contents of the library, whereas this is a deeper design for that particular space inside it. However, it might be useful to highlight a few differences with the examples in those slides and explain why, so people develop the right expectations and mental model. 

* `map(predicate, arr)` rather than `map(arr, predicate)` means we can use `map(predicate)` and `map(predicate)(arr)`.
* Operators like `map` do _not_ serialise any new representation, they only operate on a value. Otherwise we'd end up with a lot of unnecessary intermediate representations, less composable operators and much slower performance. `reduce` is the usual way to create a new representation. Hence if you wanted to transform a new array it would be closer to `reduce([], map(fn, arr))`. If that's a common enough use case, another form can be added that combines them, i.e. `map([], fn, arr)`.
* Emitters created via `from` are all lazy. This means `map(fn, arr)` does not start immediately running itself till completion. Otherwise you couldn't do things like `for (const d of map(fn, arr))` which steps through the transformation for each value one by one. If you wanted that you can use `val` to _evaluate_ it and return the value.

### Should this be done in X instead?

The goal of the standard library proposal is specifically to move common functionality from userland into the runtime. These common operators for which people use libraries like lodash, Rx, etc and covered here are the main candidates. There are also [long term and performance concerns](https://github.com/whatwg/dom/issues/544#issuecomment-352499976) if this gets specified outside of TC39. In addition, once this lands, we then have a consistent and interoperable foundation for other parts of the standard library to build upon (e.g. stats libraries would heavily benefit) as well as platform API (browsers/node) to converge upon and leverage. The current design is ergonomic to use, but the committee could also think about dedicated syntax in the longer term, for things like running, chaining, or evaluating an Emitter. Extensions to this proposal like signal-based programming which would allow reactive programming with sync-like code would certainly benefit from dedicated syntax.
