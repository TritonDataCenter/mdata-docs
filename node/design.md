# Design

Node.js is JavaScript, everything you already know about JavaScript can be
applied to your application. The patterns you use to write your frontend client
code can continue to be used when writing your server side application logic.
There are no language extensions or any other modifications to JavaScript for
Node to accomplish its goal of server side JavaScript.

There are however patterns used throughout Node and Joyent that are helpful to
know and can be useful while designing your application.


## <a name="EventEmitter"></a>EventEmitter

The first pattern to be aware of is the
[EventEmitter](http://nodejs.org/api/events.html#events_class_events_eventemitter)
pattern. Since you're primarily interacting with asynchronous actions, the
`EventEmitter` is more or less a base class that allows implementors to emit an
event, and consumers to subscribe to the events they are interested in.

When implementing, you simply need to `emit` the event and the arguments
associated with that event.

```javascript
var EventEmitter = require('events').EventEmitter;
var util = require('util');

function MyClass() {
  if (!(this instanceof MyClass)) return new MyClass();

  EventEmitter.call(this);

  var self = this;
  setTimeout(function timeoutCb() {
    self.emit('myEvent', 'hello world', 42);
  }, 1000);
}
util.inherits(MyClass, EventEmitter);
```

The constructor for `MyClass` creates a timer that will fire in 1 second, when
that timer fires it will `emit` the event `myEvent` with the string `'hello
world'` and the number `42`. To consume that event you need to use the `on()`
method which was added to the prototype of your class when you inherited it
from `EventEmitter`:

```javascript
var myObj = new MyClass();
var start = Date.now();
myObj.on('myEvent', function myEventCb(str, num) {
  console.log('myEvent triggered', str, num, Date.now() - start);
});
```

It's important to note that you subscribe `EventEmitter` events to be notified
of asynchronous events, the execution of the listeners themselves is
synchronous. So if the `myEvent` event has 10 listeners subscribed, all 10
listeners will be called sequentially without deferring to the event loop. With
that synchronicity in mind it's important to defer emission of events. This
allows consumers to subscribe to multiple events in the same turn of the event
loop, and have confidence that the callback will be fired sometime in the
future.

If a descendent of `EventEmitter` has cause to `emit('error')` and there are no
listeners subscribed to that event, the `EventEmitter` base class with `throw`
an exception, which will lead to the
[uncaughtException](http://nodejs.org/api/process.html#process_event_uncaughtexception)
event firing on the `process` object.

## <a name="streams"></a>Streams

Streams are another base pattern used extensively throughout Node. Along with
most core pieces implementing the [EventEmitter](#EventEmitter) pattern, many
will implement the
[Readable](http://nodejs.org/api/stream.html#stream_class_stream_readable),
[Writable](http://nodejs.org/api/stream.html#stream_class_stream_writable), or
both ([Duplex](http://nodejs.org/api/stream.html#stream_class_stream_duplex))
interfaces.

Streams are an abstract interface that provide common events from things like
`readable`, `writable`, `drain`, `data`, `end`, and `close`. These events in
and of themselves are powerful to interact with, but the most powerful part of
the streams interface is your ability to compose a series of streams into a
pipeline.

A pipeline itself is helpful to simplify the complexity, understandability, and
readability of your code. But also by using the `.pipe()` pattern allows Node
to communicate back-pressure through the pipeline. This back-pressure means
that you're only attempting to read what you're able to write, or write what
you're able to read. Therefore you're only keeping the amount of work you can
accomplish in memory at a given time.

Consider if you want to send data coming in from `stdin` to both a local file
and to a remote server.

```javascript
var fs = require('fs');
var net = require('net');

var localFile = fs.createWriteStream('localFile.tmp');

net.connect('255.255.255.255', 12345, function(client) {
  process.stdin.pipe(client);
  process.stdin.pipe(localFile);
});
```

Both `client` and `localFile` will read from `stdin` but the data will only as
fast as the slowest reader is able to consume.

`pipe()` always returns the destination stream. So if the destination is
Duplex, or if it is a special case of Duplex known as a
[Transform](http://nodejs.org/api/stream.html#stream_class_stream_transform)
you are able to chain the pipeline together.

Consider the previous example, but this time we only want to send to a local
file and want to
[gunzip](http://nodejs.org/api/zlib.html#zlib_zlib_creategunzip_options) the
stream before it's written to disk.

```javascript
var fs = require('fs');
var zlib = require('zlib');

process.stdin.pipe(zlib.createGunzip()).pipe(fs.createWriteStream('localFile.tar'));
```

## <a name="controlflow"></a>Control Flow

Since JavaScript has the concept of functions as first class objects, and
closures, it easy to define the callback right where it is needed. This is
convenient when prototyping your solution, as it allows you to consolidate the
logic right where it's needed. However it can quickly lead to unwieldy sets of
embedded functions, sometimes referred to as the christmas tree.

Consider that you want to read a series of files in succession, and perform a
common task on the results of those tasks:

```javascript
fs.readFile('firstFile', 'utf8', function firstCb(err, firstFile) {
  doSomething(firstFile);
  fs.readFile('secondFile', 'utf8', function secondCb(err, secondFile) {
    doSomething(secondFile);
    fs.readFile('thirdFile', 'utf8', function thirdCb(err, thirdFile) {
      doSomething(thirdFile);
    });
  });
});
```

This doesn't appear too bad, but there are some consequences to the pattern.

 1. If the logic around any of these pieces gets too complex it can be
difficult to understand the flow and order of operations.
 1. There's no error handling being done here, and in fact by the time we're in
the third level we've shadowed the first operations error twice.
 1. The result from reading the first file will stay active in the eyes of the
GC until the third operation completes.
  - Closure memory leaks are both the most predominate types of leaks found in
JavaScript applications, and often the most difficult to diagnose and detect.

If you find yourself needing to do a series of asynchronous operations on a set
of input, it's good to find a [control flow library](#vasync) that can simplify
the process for you. In general you want to find something that fits your
coding style, but provides you with insight when you're debugging your
application.

## <a name="codingstyle"></a>Coding Style

It's important to follow a style that is comfortable for you and your team.
That being said there are some techniques that can be used to make your life
developing with Node easier.

### Name your functions

Always name your functions. While V8 will try its best to identify by script
name and function location the source of an error, it can be difficult to
differentiate functions. The last thing you want to do when debugging while
building your application is waste time fixing the wrong function.

### No closures

Along with named functions, move your function definitions out side of other
functions. Doing this changes your thinking from closure based, to stack based.
This small switch in logic can help eliminate the majority of the unintended
memory leaks that closures can bring with them.

### More and smaller functions

The V8 JIT is a powerful engine, and can optimize a lot of patterns, but the
smaller and more concise you are with your functions the more likely the JIT
will be able to inline those functions. On the upside, if your function is
small (no more than a 100 or so lines of code for instance) it's likely to
increase the readability and understandability of your codebase, which should
decrease the amount of effort spent trying to maintain your application.

### Lint your codebase

While `lint`ing is often used to enforce a specific common style on a code
base, it can also be quite helpful to identify bad patterns. There are many
different tools available to achieve this, find one that fits your needs. Also,
strongly consider enabling strict mode on your codebase `'use strict';`, as
this can help your code fail fast if the JavaScript parser can identify a
leaked global or similar bad behavior.

## Logging

While designing and building your application, be sure to plan for the future.
Especially consider the tools you will need when it comes to debugging. An
excellent and obvious first step is to add appropriate amounts of logging to
your application. Make sure you pick a [logging library](#bunyan) that supports
the feature sets you need, some considerations would be: does it support the
destinations you care about, the output format you prefer, and is the API what
you expect from logging while at the same time not feeling foreign to Node and
your codebase.

It's important to identify the information that will be critical for debugging
and analyzing your application while it's running. But be mindful, including
excess information in your logs may have a detrimental effect on performance or
on storage. Be sure you're including the useful information, and in the
appropriate places, while not also slowing your application down for
unnecessary information.

## <a name="client-server"></a>Client Server

It can be advantageous to design your application in such a way that allows you
to build distributed systems. It's quite natural to describe this sort of
interface with a [REST like API](#restify) over HTTP, or even with raw [JSON
over TCP](#fast). Combine Node's expertise in asynchronous network
environments, and the use of [streams](#streams) into a powerful distributed
and scalable system.

## <a name="specific-software"></a>Specific Software

### <a name="bunyan"></a>Bunyan

[Bunyan](http://npmjs.org/package/bunyan) is a straight forward logging library
for Node.js applications. Bunyan's output is line delimited JSON, which makes
it easy to consume on the command line with your normal unix command line
utilities like `grep` and `sed`, also with its own CLI utility, or with the
[json cli utility](https://npmjs.org/package/jsontool).

Bunyan also comes with builtin support for DTrace, this support allows you to
keep your existing log level for your existing destinations (e.g. `INFO`) but
enable at runtime a more verbose level (e.g. `TRACE`) and be notified in user
space of those log entries, without them also going to your existing
destinations and potentially filling the disks. DTrace is completely safe to
use at runtime, so if enabling higher levels of logging would result in
detrimental effect your system DTrace will exit before doing your system harm.

### <a name="vasync"></a>vasync

[vasync](https://npmjs.org/package/vasync) is a control flow library, inspired
by the patterns found in the [async](https://npmjs.org/package/async) module.
However vasync is designed specifically to enable a consumer to be able to have
visibility and observability into their progress for a given task. Such
information is crucial when trying to determine how far along a task was before
an error occurred.

### <a name="restify"></a>restify

[Restify](https://npmjs.org/package/restify) is a module for creating and
consuming REST endpoints. Designed specifically to increase the observability
and debugability of your application, Restify comes with first class
[Bunyan](#bunyan) support as well support for DTrace. With Bunyan and DTrace
support, you're gaining the ability to see via the logs or at runtime routes
and their latencies over requests. 


### <a name="fast"></a>fast

[Fast](https://npmjs.org/package/fast) is a lightweight library for efficiently
handling JSON messaging across a TCP channel. At its base it was created to
enable message based RPC, where the result for a given command may induce a
series of related objects being streamed to the client. Designed with
observability in mind, fast also comes with DTrace support, allowing you to
quickly identify the performance characteristics of your server and clients.

### <a name="workflow"></a>workflow

[workflow](https://npmjs.org/package/node-workflow) built on top of
[restify](#restify), enables you to define orchestration among a series of
remote services and their APIs. This allows you to decompose a complex series
of operations down to a sequence of discreet tasks with a state machine.
Workflow is more than a mechanism to allow you to define a sequence of tasks
that are run in a specified order, it also enables you to handle failure
states, describe timeouts and define retries, as well as identify stuck
workflows.

### <a name="verror"></a>verror

[verror](https://npmjs.org/package/verror) extends the base `Error` class, and
allows you to define your messages using `printf` string formatting. The logic
for your application is often a composition of asynchronous methods, and when
adding error handling you often want to bubble that information up through your
system. verror has `VError` and `WError` which allow you to accumulate multiple
levels of errors through a chain, and either see the combination of errors in
the message (as in `VError`) or have a final message in `WError` but
programmatically be able to access the previous errors in the chain.

