## Walkthrough

> **Composability arises from the regularity of shapes**

This section aims to build up deeper intuition and familiarity step-by-step. The details here do _not_ represent common usage, or what users will be conscious of when using it regularly - composing, running, and evaluating Emitters will all become quickly intuitive after just playing with a few examples. This section is more useful for implementors to understand how the full processing model is built from a very small core.

### Multiple Values

With `Promise`, you can send a single value down a chain like this:

```js
new Promise(..)
  .then(..)
  .then(..)
  .then(..)
```

With `Emitter`, you can send multiple values down a chain like this:

```js
new Emitter(..)
  .each(..) 
  .each(..) 
  .each(..) 
```

### Error-handling 

Because an Emitter extends a Promise, we inherit the same familiar error-handling mechanisms rather than inventing new ones i.e. it can be resolved, representing it's final value, or rejected, and finally can be used for any cleanup.

```js
new Emitter(..)
  .each(..)
  .each(..)
  .each(..)
  .then(..)
  .catch(..)
  .finally(..)
```

### Sending Values

You can use `.next` to send values through an Emitter.

```js
const emitter = new Emitter()
emitter.each(d => console.log('d', d)) // d 42
emitter.next(42)
```

However, in some cases it might be a little hacky to let anybody who gets hold of the emitter to send values to it's children from the outside. 

That's why Emitter lets the author control the behaviour, by giving access to the ability to send values via the revealing constructor pattern.

```js
// only send true if an odd number was received
new Emitter((d, i, { send }) => { if (d % 2) send(true) })

// delay sending the value by 1 second
new Emitter((d, i, { send }) => { setTimeout(() => send(d), 1000) })

// send 3 values, then resolve the emitter
new Emitter((d, i, { send, resolve, reject }) => { 
  send(1)
  send(2)
  send(3)
  resolve('done')
})
```

Note on the signature: This reflects the familiar pattern in JavaScript used by the Array methods:

* `d` - The first parameter is the _datum_, a single item from a set.
* `i` - The second parameter is _implicit data_. This is usually the index of the element in an array, but it could be any useful metadata: for example, the key of a property in an object, or something more exotic like the HTTP headers for a server response.
* `n` - The last parameter represents the current _context_. For array operators, this is the entire array. Similarly, this is the actual controller for the Emitter which you can send/resolve/reject/etc. For convenience, methods can be lazily destructured here.

### Operators

`.each` connects one Emitter to another, and returns the last one.

When you do: 

```js
emitter.each(d => console.log('d', d))
```

This is actually equivalent to: 

```js
emitter.each(new Emitter(d => console.log('d', d)))
```

So you could also write an Emitter that does some processing before sending like this:

```js
emitter.each((d, i, n) => n.send(d + 1))
```

You can implement the usual `map`/`filter` suspects using higher-order functions:

```js
const map = fn => (d, i, n) => n.send(fn(d))
const filter = fn => (d, i, n) => fn(d) && n.send(d)

emitter
  .each(map(add1))
  .each(filter(even))
```

Note that `.each` is also variadic, so the above would also be equivalent to:

```js
emitter.each(map(add1), filter(even))
```

Implementation note: the actual `map` and `filter` operators are not implemented using higher-order functions, since capturing the predicate function that way would crash performance. Rather, they each create a new Emitter and take the predicate in the constructor: 

```js
Emitter.map = fn => new MapEmitter(fn)
```

### Hooks

Passing a function configures the `next` behaviour - what happens when the Emitter receives a new value. There are other hooks besides `next` you can configure too, for example what happens when you `resolve` or `reject` an Emitter.

```js
new Emitter({ next, finally, resolve, reject, value, run })
```

We'll come back to the `value` slot and `run` later. 

These hooks can be installed via the constructor, but also via the class itself e.g:

```js
class UDPMulticaster extends Emitter {
  static finally(){ 
    return new Promise(resolve => this[socket].close(resolve))
  }
}
```

In this case we have an UDP Multicaster which extends an Emitter. When it's resolved it performs some clean-up first, tearing down the the internal `dgram` socket, before finishing resolving.

### Creating

A more common way of creating an Emitter is using the static helper `Emitter.from` which is used to lift other values to the Emitter type. `from` is very powerful and can accept lots of things like functions, sync/async generators/iterables, etc (see API for full details).

```js
from([1,2,3])
  .each(
    map(..)
  , filter(..)
  , reduce(..)
  )
```

### Lazily Running

`.each` just connects Emitters, it does not trigger any behaviour. You can use `.run()` to "run" an Emitter, which just means it calls the internal `run` hook (by default, a noop). You can also use the form `.run(...args)` to more easily connect a chain of Emitters and then run the source in one go i.e.: 

```js
A.each(B, C, D)
A.run()
// is equivalent to:
A.run(B, C, D)
```

The expression `from([1, 2, 3])` creates an Emitter, that _when run_, will send 1, 2, 3 and then resolve itself. Expressed in code, this is equivalent to:

```js
new Emitter({
  run(me){
    me.send(1)
    me.send(2)
    me.send(3)
    me.resolve()
  }  
})
```

In addition, instead of writing the following:

```js
from([1,2,3])
  .run(
    map(..)
  , filter(..)
  , reduce(..)
  )
```

You can use the static form `Emitter.run` which is one-line of sugar (`Emitter.run = (input, ...args) => from(input).run(...args)`), so you can write the more modern form:

```js
run(
  [1,2,3]
, map(..)
, filter(..)
, reduce(..)
)
```

### Composition

You can compose any set of Emitters into another Emitter and use it anywhere else you'd expect to use an Emitter. 

```js
const transform = compose(
  map(..)
, filter(..)
, reduce(..)
)

// these two are semantically equivalent:
run([1,2,3], transform)
run([1,2,3], ...[map(..), filter(..), reduce(..)])
```

It's safe to never have to think about the `ComposedEmitter` as something other than it's components, but always just visualise in terms of a group of Emitters as a single block.

### Short-hand Composition

The following would create a `ComposedEmitter` grouping an `ArrayEmitter` and a `MapEmitter`:  

```js
compose([1,2,3], map(d => d + 1))
```

However, all of the operators take their trailing arguments and compose them i.e.: 

```js
Emitter.map = (fn, ...args) => compose(...args, new MapEmitter(fn))
```
 
Which means you could write things like: 

```js
map(d => d + 1, [1,2,3])
```

This also naturally works recursively, which gives rise to the ability to write:

```js
reduce(0, filter(even, map(add1, [1,2,3])))
```

Or things like ([from the modern refactor of most.js](https://github.com/mostjs/core/blob/master/examples/counter/src/index.js)):

```js
const inc = map(() => 1, buttonA) 
const dec = map(() => -1, buttonB)
const counter = reduce((acc, d) => acc + d, 0, flatten(inc, dec))
const render = tap(total => { output.innerText = String(total) }, counter)
run(render)
```

### Iteration

How does this relate with the iteration protocol? You can think of `.run()` as one way to run an Emitter. The iteration protocol is another way to turn an Emitter, "one-by-one", by just calling `.next()` till it's done. This means you can do things like this:

```js
for (const v of map(d => d + 1, [1,2,3]))
  console.log(v)
```

### Named Channels

Emitter allows creating an Emitter on a named channel using `.on()`:

```js
emitter.on('click')
```

This is similar to platform API that accepts a string and function. By omitting the second callback parameter, it returns an Emitter. This provides a unifying mental model with existing callback code and makes it easier to transition/upgrade, as the following are equivalent: 

```js
emitter.on('click', fun)
emitter.on('click').each(fun)
```

In fact, since the trailing arguments are just passed to `.each`, you can do the following:

```js
emitter.on('click', ...args)
emitter.on('click').each(...args)
emitter.on('click', filter(..), fun)
```

### Backward-Compatibility

Instead of having to change existing interfaces at all, an even better approach is to use the static `Emitter.on` helper: 

```js
on(element, 'click').each(...args)
on(element, 'click', ...args)

on(fork('./child'), 'message').each(...args)
on(fork('./child'), 'message', ...args)
```

Since the channels are linked using Symbols (could also be WeakMap), it means we don't pollute the public interface of objects with methods like `.on`, `.emit`, etc. 

You can create a named channel with any arbitrary object. If an EventTarget or EventEmitter is used, it calls `addListener`/`addEventListener` when created and `removeListener`/`removeEventListener` when resolved/rejected.

```js
// logs three messages from the parent process then cleanly tears down
tap(log, until(3, on(process, 'message')))
```

### Awaiting Till Done

Since `each`/`run` chains Emitters and returns the last one, and Emitter extends Promise - you can chain, run and await a pipeline of Emitters till that process completes:

```js
// arr == [3]
const arr = await run(
  [1,2,3]
, map(d => d + 1)
, filter(d => d % 2)
, reduce([])
)
```
```js
await until(status => status == ‘connected’, on(peer, 'status'))
```

Emitter resolves to its `value`, which can be set internally. Operators like `reduce` do this for example, as they run an accumulator function and update their value.

### Evaluation

In many cases, you may want to write transformations for which you do not want to pollute your code with `async`/`await` unnecessarily. In this case, you can use `val`, which stands for _evaluate_. Evaluate simply runs an Emitter, and returns the value of the last one i.e:

```js
Emitter.val = (...args) = run(...args).value
```

Using `await` in the previous `map`/`filter`/`reduce` with an array is redundant since it finishes immediately. We can use `val` instead here:

```js
// arr == [3]
const arr = val(
  [1,2,3]
, map(d => d + 1)
, filter(d => d % 2)
, reduce([])
)
```

This means we have a higher power-to-weight ratio single library of operators that covers synchronous and asynchronous use cases.

### Backpressure

What is the return value of `.next(d)`? Rather, what would be _useful_ or _meaningful_ for its return value to be?

The return value actually composes the return values of the children for that item. 

```js
emitter.each(() => 1)
emitter.each(() => 2)
emitter.next(42) // returns [1,2] 
```

This works recursively too, plus where operators chain their send calls, we can successfully elide intermediate arrays to always have a flat list of the return values from the leaf nodes:

```js
emitter.each((d, i, n) => n.send(d, i)).each(() => 1)
emitter.each(() => 2)
emitter.next(42) // returns [1,2] rather than [[1], 2]
```

This is useful for a number of reasons, chiefly though, because it allows communicating backpressure without any separate signalling mechanism. An Emitter can simply return a Promise to signal when it's done consuming that value. The caller then has the fine-grained information to decide to wait for all - or however many consumers it wants - to have finished processing before moving on:

```js
emitter.each(async () => { .. })
emitter.each(async () => { .. })
await Promise.all(emitter.next(1)) // wait till both have processed 1
await Promise.all(emitter.next(2)) // wait till both have processed 2
await Promise.all(emitter.next(3)) // wait till both have processed 3
```

### Concurrency Control

Previously we showed `.run()` as one way to run an Emitter, as well as the iteration protocol to step through an Emitter until it's exhausted. `.run()` however does not care about the return values of the Emitter. Hence we have `run.limit(N)` which leverages that information, and will run an Emitter with a max of `N` inflight values. 

For example, let's say we have a framework that composes a pipeline that will search a predefined port space for a free server and then create an agent from that server:

```js
const vendAgents = () => compose(
  range(8000, 9000)
, (port, i, { send }) => new Promise(resolve => createServer()
    .on('error', () => resolve()) // EADDRINUSE, port in use, move on..
    .listen(port, function(){ send(this), resolve() }) // created server
  )
, map(createAgentFromServer)
)
```

Then let's say a consumer wants to pull 3 new agents, concurrently trying upto 2 ports at a time:

```js
const [agent1, agent2, agent3] = await run.limit(2)(reduce([], until(3, vendAgents())))
```

If we didn't use `run.limit`, we'd overshoot and eagerly create too many servers. Notably, other consumers are also able to obtain however many new agents they want, at whatever rate they like (e.g. give me 5 new agents, trying up to 3 ports at a time).

`run.await` is an alias for `run.limit(1)`, meaning await the results before moving on to the next value. This is conceptually the same as the iteration and async iteration protocol, but in a functional form. 

For example, let's say we have some test cases that we want to step through one-by-one - in each case we create a new environment, test the scenario, tear down, before moving on to the next: 

```js
await run.await({
  case1: { body: 1, expected: 200 }
, case2: { body: 2, expected: 404 }
, case3: { body: 3, expected: 500 }
}, async ({ expected, body }, sessionID) => {
  const { server, destroy } = await startup(...)
  same(await server.recv(body), expected, sessionID)
  await destroy()
})
```

`run.all` is an alias for `run.limit(Infinity)`, meaning you don't care when all the values have finished or in what order, but you just want to know when they are all done. 

For example, let's say we have a cluster of peers. We want to teardown the entire cluster as fast possible, destroy all the peers in any order, but just await till they've all been destroyed:

```js
await run.all(cluster, peer => peer.destroy())
```