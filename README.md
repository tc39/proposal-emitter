# ECMAScript proposal: `Emitter`

* JavaScript's most general container, over single and multiple, sync and async values
* Operators form the Generics, a large part of the eventual standard library
* Composable algorithmic transformations, decoupled from their input and outputs
* Well-integrated with language rather than inventing lots of new concepts, (i.e. Promises, iteration, async iteration, etc)
* Backwards-compatible unification of platform API (interfaces like EventTarget and EventEmitter)
* [Ultra-High Performance (faster than most.js)](/PERFORMANCE.md) and memory efficient 

<br> 

## Reactive programming examples

Log XY coordinates of click events on `<button>` elements:

```js
const { on, map, filter, run } = Emitter

run(
  on(document, 'click')
, filter(ev => ev.target.tagName === 'BUTTON')
, map(ev => ({ x: ev.clientX, y: ev.clientY }))
, coords => console.log(coords)
)
```

Map clicks from two buttons to a stream of +1 and -1, merge them and render the result:

```js
const { on, map, tap, reduce, flatten, render } = Emitter

const inc = map(() => 1, on(buttonA, 'click')) 
const dec = map(() => -1, on(buttonB, 'click'))
const counter = reduce(0, flatten(inc, dec))
const render = tap(total => { output.innerText = String(total) }, counter)
run(render)
```

Fork 10 peers, wait till they've all connected, return a reference to the processes:

```js
const { run, until, once, flatten, reduce } = Emitter
const { fork } = require('child_process')

const processes = await run(
  10
, map(() => fork('peer.js'))
, map(async peer => {
    await until(d => d == 'connected', on(peer, 'message'))
    return peer
  })
, flatten()
, reduce([])
)
```

### Iterable programming examples

From a range of numbers, divide them by 4, remove any greater than 12, then start collecting those one by one:

```js
function* range(from, to) { 
  for (let i = from; i <= to; i++) yield i  
}

const arr = run(
  range(40, 60)
, map(d => d / 4)
, filter(d => d < 12)
, reduce([])
)
```

Transform a series of numbers one by one and log the result:

```js
for (const d of map(d => d + 1, [1, 2, 3]))
  console.log('transformed output', d)
```

Log whenever the user presses CTRL+C

```js
for await (const d of filter(([n]) => n == 3, process.stdin))
  console.log('SIGINT!')
```

<br>

## API

### Transformation

  * [`map`](/API.md#map)
  * [`reduce`](/API.md#reduce)
  * [`flatten`](/API.md#flatten)

### Filtering

  * [`filter`](/API.md#filter)
  * [`until`](/API.md#until)

### Running

  * [`run`](/API.md#run)

### Creating

  * [`on`](/API.md#on)
  * [`once`](/API.md#once)
  * [`from`](/API.md#from)
  * [`compose`](/API.md#compose)

<br>

## Background

For general background into this space, especially from more of a language perspective, [Rich Hickey's talk on the motivation, design and use of Clojure's core.async library](https://www.infoq.com/presentations/clojure-core-async) is a must.

For comparison, here would be Rob Pike's original example of [reducing tail latency using replicated search servers](https://talks.golang.org/2012/concurrency.slide#50) discussed in the talk:

```js
const { race, timeout, until, reduce, flatten } = Emitter 

const results = await race(
  timeout(80)
, until(3, reduce([], flatten(
    race(web1(query), web2(query))
  , race(img1(query), img2(query))
  , race(vid1(query), vid2(query))
  )))
)
```

It's useful to also familiarise with existing JavaScript libraries in this space:

* [ramda](https://github.com/ramda/ramda)
* [lodash](https://github.com/lodash/lodash)
* [most.js](https://github.com/mostjs/core)
* [rxjs](https://github.com/ReactiveX/rxjs)
* [callbags](https://github.com/staltz/callbag-basics/)

More details are included and will be documented in the [FAQ](FAQ.md).

* [How does this relate to Transducers?](/FAQ.md#how-does-this-relate-to-transducers)
* [How does this relate to Iterators/Generators? ](/FAQ.md#how-does-this-relate-to-iteratorsgenerators)
* [How does this relate to `ReadableStream`/`WritableStream`](/FAQ.md#how-does-this-relate-to-readablestreamwritablestream)?
* [How does this relate to Observables?](/FAQ.md#how-does-this-relate-to-observables)
* [How does this relate to the Standard Library?](/FAQ.md#how-does-this-relate-to-the-standard-library)
* [Should this be done in X instead?](/FAQ.md#should-this-be-done-in-x-instead)
