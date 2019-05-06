# API

## Contents

* [**Operators**](#operators)
  * [`map`](#map)
  * [`filter`](#filter)
  * [`tap`](#tap)
  * [`reduce`](#reduce)
  * [`until`](#until)
  * [`flatten`](#flatten)
  * [`debounce`](#debounce)
  * [`buffer`](#buffer)

* [**Static Methods**](#static-methods)
  * [`run`](#run)
  * [`val`](#val)
  * [`compose`](#compose)
  * [`from`](#from)
  * [`timeout`](#timeout)
  * [`interval`](#interval)
  * [`race`](#race)
  * [`on`](#on)
  * [`once`](#once)
  * [`emit`](#emit)
  * [`listeners`](#listeners)

<br>

## Operators

All operators are free functions which can be referenced from the constructor (e.g. `Emitter.map`). 

All operators compose their trailing arguments i.e.

```js
op = (a, b, ...args) => compose(...args, new OpEmitter(a, b))
```

No extra args means `compose(new OpEmitter(a, b))` which returns `new OpEmitter(a, b)`.

### `map`

The `map(fn)` operator creates a new Emitter sending the return value of the provided function invoked on every element received.

```js
m = map(d => d + 1)
m.each(d => console.log(d)) // logs 2
m.next(1)
```

### `filter`

The `filter(fn)` operator creates a new Emitter sending the value received, if the provided function invoked on it returns truthy.

```js
f = filter(d => d % 2)
f.each(d => console.log(d)) // logs 1
f.next(1)
f.next(2)
```

### `tap`

The `tap(fn)` operator creates a new Emitter invoking the provided function on every element received, and sending the original value it received. This is commonly used for executing side-effects on every element, without having to worry about the return value if you used `map` instead.

```js
t = tap(d => console.log('received', d))
t.each(d => console.log(d)) // logs 1
t.next(1)
```

### `reduce`

The `reduce` operator creates Emitters that updates it's internal value every time a new value is received, and also broadcasts it's new internal value. Besides the traditional method of using an accumulator function, other "default reducers" are included too i.e. for reducing into an array, object, string, number, generator, async generator, another emitter, etc.

* `reduce(function, value)`

  The `reduce(function, value)` form creates a new Emitter whose value is initialised to `value`. The provided `function` is invoked on every element received, along with the internal value of the Emitter. The value of the Emitter is updated to the return value of the provided function every time it's called. 

  ```js
  val([1, 2, 3] reduce((acc, v, i, n) => acc + v, 10)) // returns 16
  ```

* `reduce(number)`

  The `reduce(number)` form creates a new Emitter whose value is initialised to `number`. The value of the Emitter is incremented by every value it receives, or by 1 if it's not a number. `reduce(0)` is a common way to count things. 

   ```js
   val([1, 2, 3], reduce(0)) // returns 6
   val({ user1: {}, user2: {}, user2: {} }, reduce(0)) // returns 3
   ```

* `reduce(string)`
  
  The `reduce(string)` form creates a new Emitter whose value is initialised to `string`. The value of the Emitter is concatted by every value it receives.

   ```js
   val(['a', 'b', 'c'], map(capitalise), reduce('')) // returns 'ABC'
   ```  

* `reduce(array)`

  The `reduce(array)` form creates a new Emitter whose value is initialised to `array`. Every value the Emitter receives is pushed onto the array.

  ```js
  val([1, 2, 3], map(add1), reduce([])) // returns [2, 3, 4]
  ```

* `reduce(object)`
  
  The `reduce(object)` form creates a new Emitter whose value is initialised to `object`. Every value the Emitter receives is assigned to the object at the key specified 

  ```js
  // object-to-object transformation, without converting to arrays!
  const users = {
    user1: { city: 'london' }
    user2: { city: 'new york' }
  }

  // returns { user1: 'london', user2: 'new york' }
  val(users, map(d => d.city), reduce({})) 
  ```
  ```js
  // array-to-object transformation
  // returns { 0: 11, 2: 13 }
  val([11, 12, 13], filter(d => d % 2), reduce({}))
  ```

* `reduce(generator, value)`
  
  The `reduce(generator, value)` form creates a new Emitter whose value is initialised to `value`. The provided `generator` is primed with the specified `value` (invoked with it and `.next()`). Every value the Emitter receives gets sent to the generator (i.e. `.next(v)` and received internally via `yield`). The value of the Emitter is updated to what the generator yields.

  ```js
  // takes in a stream of bytes being uploaded and concats them onto a buffer
  // sends the total buffer size forward every time
  // resolves to and returns the buffer
  await run(
    byteStream
  , reduce(function*(buffer){ 
      while (buffer.byteLength < expectedSize) 
        buffer = buffer.concat([buffer, yield buffer.byteLength])
      return buffer
    }, Buffer.alloc(0))
  , tap(size => console.log("received bytes", size))
  )
  ```

  The Emitter resolves to what the generator returns, or rejects with what it throws. 

  In case the Emitter is resolved or rejected externally rather than on it's own, this will just call `.return(v)` or `.throw(e)` on the generator. You can handle and customise what value it settles with via a `try..catch..finally` block. 

  ```js
  // keeps pushing values onto an array
  // when resolved returns the array length rather than with what was passed in
  val([1,2,3], reduce(function*(acc){
    try { while (true) acc.push(yield) } 
    finally { return acc.length }
  }, []))
  ```

  You can similary use anything that implements `Symbol.iterator` here, there will just be no priming step required.

* `reduce(async generator, value)`

  The `reduce(async generator, value)` form creates a new Emitter whose value is initialised to `value`. The provided `async generator` is primed with the specified `value` (invoked with it and `.next()`). Every value the Emitter receives gets sent to the generator (i.e. `.next(v)` and received internally via `yield`). The value of the Emitter is updated to what the generator yields.

  ```js
  // will emit 1, then 2 after 1 second, then 3 after 2 seconds, 
  // then resolve and return after 3 seconds to [1, 2, 3]
  await run([1, 2, 3], reduce(async function*(acc){ 
    try { 
      while (true) {
        acc.push(yield) 
        await run(timeout(1000))
      }
    } finally { return acc }
  }, []))
  ```

  You can similary use anything that implements `Symbol.asyncIterator` here, there will just be no priming step required.

* `reduce(emitter)`
  
  The `reduce(emitter)` form creates a new Emitter whose value is always the value of `emitter`, and is sent forward every time it emits a new value. Every value the Emitter receives is redirected to `emitter` to handle. This is mostly useful when you want a stream of the value of another Emitter rather than it's actual stream. 

  
  ```js
  // server only sends deltas, but the subscription value combines them
  // converts this stream of deltas, into a stream of the full, latest set
  reduce(server.subscribe('trades'))
  ```

* `reduce(undefined | null | boolean | symbol)`

  For all other inputs, the Emitter created will be initialised with the specified value, and is then updated with the identity function i.e. every value it receives becomes it's new internal value. This is useful for collecting the last item sent

  ```js
  // this fails to find any odd numbers and resolves/returns false
  val([2,4,6], filter(d => d % 2), reduce(false))

  // this finds the first user in new york
  // return { name: 'bob', city: 'new york' }
  val({ 
    user1: { name: 'alice', city: 'london' }
  , user2: { name: 'bob', city: 'new york' } 
  }
  , filter(d => d.city == 'new york')
  , reduce()
  )
  ```

### `until`

The `until` operator creates Emitters that will resolve itself once a certain condition becomes true.

* `until(number)`

  The `until(number)` form creates a new Emitter that will resolve once it receives `number` values. 

  ```js
  // logs three messages from the child process then cleanly tears down
  tap(log, until(3, on(process, 'message')))
  ```

* `until(function)`

  The `until(function)` form creates a new Emitter that will invoke the provided `function` on every value it receives and resolve if it returns true. 

  ```js
  // will keep firing until the peer emit's a status with 'connected'
  await until(status => status == 'connected', on(peer, 'status'))
  ```

* `until(promise)`

  The `until(promise)` form creates a new Emitter that will resolve when the provided `promise` settles. 
  
  ```js
  // flatten B, C, D into A, until it's resolved
  const resolved = Promise.resolve(A)
  run(until(resolved, B, (v, i) => A.next(v, i))
  run(until(resolved, C, (v, i) => A.next(v, i))
  run(until(resolved, D, (v, i) => A.next(v, i))
  ```
  
* `until(emitter)`

  The `until(emitter)` form creates a new Emitter that will resolve when the provided `emitter` emits a value.

  ```js
  // every time on mouseenter, we start logging the mousemove events until mouseleave
  on(node, 'mouseenter', () => { 
    until(on(node, 'mouseleave'), on(node, 'mousemove'), event => console.log(event))
  })
  ```
  ```js
  // subscribe to some remote data, until component removed
  class Component extends HTMLElement { 
    connectedCallback(){
      until(node.once('disconnected'), server.subscribe('data'), data => this.render(data))
    }
  }
  ```

### `flatten`

* `flatten(...args)`

The `flatten` operator creates a new Emitter that will:

* Flatten the values it receives into itself using `run` 

  ```js
  // returns [1,2,3]
  val([[1], [2], [3]], flatten(), reduce([]))
  ```
  ```js
  // returns ['H10', 'H20', 'H30', 'I10', 'I20', 'I30']
  val(
    'HI'
  , map(char => map(num => char + num, [10, 20, 30]))
  , flatten()
  , reduce([])
  )
  ```

* If the `args` passed into `flatten` contains a function, that will be invoked on the value received before it's flattened into the Emitter. This means instead of having a `map` followed by a `flatten`, you can combine them i.e. "flatmap".

  ```js
  // returns ['H10', 'H20', 'H30', 'I10', 'I20', 'I30']
  val(
    'HI'
  , flatten(char => map(num => char + num, [10, 20, 30]))
  , reduce([])
  )
  ```

* `args` can contain other external sources you want to merge into the stream too. Again, these are just `run` into the Emitter.

  ```js
  // returns ['h', 'e', 'y', 'h', 'o']
  val(flatten('hey', 'ho'), reduce([]))
  ```
  ```js
  // create a stream that fires when first run, 
  // then also when the window position/size changes
  flatten(
    on(window, 'scroll')
  , on(window, 'resize')
  , [1]
  )
  ```

  Flattening an array of one element, is a common way of defining "starts with" behaviour.

* If a new value is received as an Emitter (or mapped to one), then the previous one will be resolved and the new one will be flattened into the Emitter (i.e. switchmap behaviour)

```js
// when then user scrolls, resolve the old subscription, 
// create a new one and rerender with the new contents whenever it changes
run(
  on(table, 'scroll')
, map(() => table.visibleRange)
, flatten(range => server.subscribe(range))
, rows => table.render(rows)
)
```

### `debounce`

The `debounce(number)` operator creates a new Emitter that will delay sending values it receives by at least `number` milliseconds since the last value received. If another value is received before that time has elapsed, the timer is reset. 

```js
// waits until the user has stopped scrolling for at least 200ms 
// before updating the subscription and contents
run(
  on(table, 'scroll')
, debounce(200)
, map(() => table.visibleRange)
, flatten(range => server.subscribe(range))
, rows => table.render(rows)
)
```

### `buffer`

The `buffer(lo, hi)` operator creates a new Emitter that will only send a maximum of `lo` values  concurrently forward. If more values than that are received, they will be queued in a backlog of size `hi` till the number of inflight values being processed drops backdown below `lo` again. 

If more values than `hi` are queued, they will be dropped. If `hi` is negative it means store the last `hi` most recent number of values, dropping earlier ones (i.e. sliding window)

```js
// composes a new pipeline that will only allow uploading one-by-one
// waiting for the previous to complete before processing the next one
const upload = compose(buffer(1), form => server.send(form))
upload.next(form1)
upload.next(form2)
upload.next(form3)
```
```js
// iterate over the last 10 clicks
for await (const click of buffer(1, -10, on(body, 'click')))
  console.log("click", click)
```

<br>

## Static Methods

### `run`

The `run` helper provides a number of common ways of running an Emitter

* [`run(...args)`](/WALKTHROUGH.md#lazily-running)
* [`run.limit(N)(...args)`](/WALKTHROUGH.md#concurrency-control)
* [`run.await(...args)`](/WALKTHROUGH.md#concurrency-control)
* [`run.all(...args)`](/WALKTHROUGH.md#concurrency-control)

### `val`

* [`val(...args)`](/WALKTHROUGH.md#evaluation)

### `compose`

* [`compose(...args)`](/WALKTHROUGH.md#composition)

### `from`

The `from` helper creates Emitters from other things

* `from(array)`

  The `from(array)` form creates an Emitter that when run will send the values of the array (with the respective index as the implicit data) and then resolve itself. 

  ```js
  // logs ['A', 0], ['B', 1], ['C', 2], 
  run(['A', 'B', 'C'], (d, i) => console.log([d, i]))
  ```

* `from(object)`

  The `from(object)` form creates an Emitter that when run will send the values of the object (with the respective key as the implicit data) and then resolve itself. 

  ```js
  // logs [0, 'A'], [1, 'B'], [2, 'C'], 
  run(from({ A: 0, B: 1, C: 2 }), (d, i) => console.log([d, i]))
  ```

* `from(function)`

  The `from(function)` form creates an Emitter that when run will invoke the provided `function` with the controller for the Emitter and by default connect the results into the Emitter. This allows deferring calculating the source, which could be another Emitter. Consider the following HTTP server: 

  ```js
  const server = fn => http.createServer((req, res) => {
    // the result of the user's function is connected to the HTTP side-effects
    // note that the function is completely isolated from these
    from(() => fn(req))
      .run(v => res.write(v))
      .then((v = 200) => res.writeHead(v))
      .catch((e = 500) => res.writeHead(e))
      .finally(() => res.end())
  })
  ```

  Then we can have different servers that respond with a value, promise or another Emitter

  ```js
  // this server will respond to requests with 'ok' and resolve
  server(req => 'ok')
  ```
  ```js
  // this server will wait for and respond with a single value and then resolve
  // note that if the function throws, the Emitter rejects
  server(async req =>
    await authenticated(req)
      ? await fetch('/users')
      : []
  )
  ```
  ```js
  // this is the most powerful option as Emitter is capable of representing any process
  // will emit 1, 2, 3 and then end with 304
  server(req => from({ send, resolve } => {
    send(1)
    send(2)
    send(3)
    resolve(304)
  })
  ```

* `from(number)`

  The `from(number)` form creates an Emitter that when run will emit `number` numbers starting from zero (i.e. similar to a range function)

  ```js
  // emitter that generates the ports 8000, 8001, 8002
  const ports = map(v => 8000 + v, 3)
  ```

* `from(generator)`

  The `from(generator)` form creates a new Emitter that primes `generator` (invoked and `.next()`) and then every value the Emitter receives gets sent to the generator (i.e. `.next(v)` and received internally via `yield`). 

  ```js
  // choreographs an Emitter that when run will emit blob.size
  // followed by 1024 byte chunks of the blob
  const chunk = blob => from(function*(i = 0) {
    yield blob.size
    while (i < blob.size)
      yield blob.slice(i, i += 1024)
  })

  // slice the binary and upload the chunks one by one
  // waits till they've all been uploaded
  await run.await(chunk(file), data => server.upload(data))
  ```

  The Emitter resolves to what the generator returns, or rejects with what it throws. 

  In case the Emitter is resolved or rejected externally rather than on it's own, this will just call `.return(v)` or `.throw(e)` on the generator. You can handle and customise what value it settles with via a `try..catch..finally` block. 

  ```js
  // keeps pushing values onto an array
  // when resolved returns the array length rather than with what was passed in
  val([1,2,3], from(function*(acc = []){
    try { while (true) acc.push(yield) } 
    finally { return acc.length }
  }))
  ```

  You can similary use anything that implements `Symbol.iterator` here, there will just be no priming step required.

  ```js
  // counts the number of bytes in the buffer
  val(reduce(0, map(() => 1, Buffer.alloc(10)))
  ```

* `from(async generator)`

  The `from(async generator)` form creates a new Emitter that primes the `async generator` (invoked and `.next()`) and then every value the Emitter receives gets sent to the generator (i.e. `.next(v)` and received internally via `yield`). 

  ```js
  // will log 5 numbers, one every second
  // then resolve and return after 5 seconds to 'done'
  await run(async function*(i = 0){ 
    while (i < 5) yield (await run(timeout(1000)), i)
    return 'done'
  }), d => console.log(d))
  ```

  You can similary use anything that implements `Symbol.asyncIterator` here, there will just be no priming step required.

  ```js
  // compose a new stream that will only emit when user presses ctrl+c
  process.SIGINT = filter(([v]) => v == 3, process.stdin)
  ```

* `from(promise)`

  The `from(promise)` form creates a new Emitter that when run will emit the value of the promise (if it's resolved, or whenever it does resolve). 

  ```js
  // will log 5 numbers, one every second
  // then resolve and return after 3 seconds to 'done'
  await run(async function*(i = 0){ 
    while (i < 5) yield (await run(timeout(1000)), i)
    return 'done'
  }), d => console.log(d))
  ```

  You can similary use anything that implements `Symbol.asyncIterator` here, there will just be no priming step required.

* `from(emitter)`

  The `from(emitter)` form returns `emitter`

  ```js
  from(emitter) === emitter
  ```

### `timeout`

The `timeout(number, value)` helper will create an Emitter that when run will emit `value` after `number` milliseconds (if `value` is a function it will be invoked and the return value will be used). The Emitter resolves after emitting it's value. Notably, since you can also resolve or reject an Emitter, it means you can also cancel the timeout.

```js
// waits for 10 ms
await run(timeout(10))
```
```js
// does not log anything
t = run(timeout(10), d => console.log('received', d))
t.resolve()
```

### `interval`

The `interval(number, value)` helper will create an Emitter that when run will emit `value` after every `number` milliseconds (if `value` is a function it will be invoked and the return value will be used). Notably, since you can also resolve or reject an Emitter, it means you can also cancel the interval.

```js
// logs 10 events before tearing down
run(until(timeout(100), interval(10), d => console.log('received', d))
```

### `race`

The `race(...args)` helper will create an Emitter that will run all it's arguments (using `run`) and the first one to emit a value will resolve the Emitter as well the other arguments.

```js
// wait for the server connect, but timeout after 10 seconds
// timer will be cancelled if server connects first
await race(server.connected, timeout(10000))
```
```js
// will keep checking if the page has a div, every 10ms, for upto 10 seconds
const waitFor = (fn, ms = 10000) => race(timeout(ms), until(fn, interval(10)))
await waitFor(() => document.body.querySelector('div'))
```

### `on`

* `on(obj, name, ...args)`

The `on(obj, name)` helper will create a new Emitter that listens on the named channel `name` of the specified object `obj`. If `obj` is an EventTarget or EventEmitter it will create a listener that nexts the events on the Emitter, that also gets cleaned when the Emitter resolves or rejects.

```js
clicks = on(document.body, 'click'), 
clicks.each(() => { console.log('clicked') })
clicks.resolve()
```

If there are any trailing arguments, they are passed to `.each()` i.e. equivalent to `on(obj, name).each(...args)`.

If `name` is not a string, then this reduces to connecting the trailing arguments on the main channel (i.e. just static form of `.each()`).

### `once`

* `once(obj, name, ...args)`

`once` creates an Emitter on a named channel similar to `on` but will resolve itself after the first time it emits. If the second parameter is not a string, it will do the same using the main channel. This is generally useful when you want a Promise from an Emitter. 

```js
// wait till the server is listening
await once(server, 'listening')
```
```js
// just wait for the first response from the server
await once(server.send(data))
```

### `emit`

The helper `emit(obj, name, v, i)` will emit the value `v` with metadata `i` to all the Emitters attached to the named channel `name` on the object `obj`.

```js
const obj = {}
on(obj, 'foo', (d, i) => console.log("A", d, i))
on(obj, 'foo', (d, i) => console.log("B", d, i))
emit(obj, 'foo', 10, 20)
```

### `listeners`

The `listeners(obj, name)` helper will return the Emitters attached on the named channel `name` to the object `obj`. Omitting the second parameter (`listeners(obj)`) will return the Emitters on the main channel.
