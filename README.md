<a href="http://promisesaplus.com/">
    <img src="http://promisesaplus.com/assets/logo-small.png" alt="Promises/A+ logo"
         title="Promises/A+ 1.0 compliant" align="right" />
</a>

#Introduction

Bluebird is a full featured [promise](#what-are-promises-and-why-should-i-use-them) library with unmatched performance.

Features:

- [Promises A+ 3.x.x](https://github.com/promises-aplus/promises-spec)
- [Promises A+ 2.x.x](https://github.com/domenic/promises-unwrapping)
- [Cancellation](https://github.com/promises-aplus)
- [Progression](https://github.com/promises-aplus/progress-spec)
- [Synchronous inspection](https://github.com/promises-aplus/synchronous-inspection-spec)
- All, any, some, settle, map, reduce, spread, join...
- [Unhandled rejections and long, relevant stack traces](#error-handling)
- [Sick performance](https://github.com/petkaantonov/bluebird/tree/master/benchmark/stats)

Passes [AP2](https://github.com/petkaantonov/bluebird/tree/master/test/mocha), [AP3](https://github.com/petkaantonov/bluebird/tree/master/test/mocha), [Cancellation](https://github.com/petkaantonov/bluebird/blob/master/test/mocha/cancel.js), [Progress](https://github.com/petkaantonov/bluebird/blob/master/test/mocha/q_progress.js), [promises_unwrapping](https://github.com/petkaantonov/bluebird/blob/master/test/mocha/promises_unwrapping.js) (Just in time thenables), [Q](https://github.com/petkaantonov/bluebird/tree/master/test/mocha) and [When.js](https://github.com/petkaantonov/bluebird/tree/master/test) tests. See [testing](#testing).

#Topics

- [Quick start](#quick-start)
- [API Reference and examples](https://github.com/petkaantonov/bluebird/blob/master/API.md)
- [What are promises and why should I use them?](#what-are-promises-and-why-should-i-use-them)
- [Error handling](#error-handling)
- [Development](#development)
    - [Testing](#testing)
    - [Benchmarking](#benchmarks)
- [What is the sync build?](#what-is-the-sync-build)
- [License](#license)
- [Snippets for common problems](https://github.com/petkaantonov/bluebird/wiki/Snippets)
- [Promise anti-patterns](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns)
- [Changelog](https://github.com/petkaantonov/bluebird/blob/master/changelog.md)

#Quick start

##Node.js

    npm install bluebird


Then:

```js
var Promise = require("bluebird");
```

If you want to ensure you get your own fresh copy of bluebird, do instead:

```js
                                        //Note the extra function call
var Promise = require("bluebird/js/main/promise")();
```

##Browsers

Download the [bluebird.js](https://github.com/petkaantonov/bluebird/tree/master/js/browser) file. And then use a script tag:

```html
<script type="text/javascript" src="/scripts/bluebird.js"></script>
```

The global variable `Promise` becomes available after the above script tag.

After quick start, see [API Reference and examples](https://github.com/petkaantonov/bluebird/blob/master/API.md)

###Browser support

Browsers that [implement ECMA-262, edition 5](http://en.wikipedia.org/wiki/Ecmascript#Implementations) and later are supported.

IE8 (ECMAS-262, edition 3) is supported if you include [es5-shim.js](https://github.com/kriskowal/es5-shim/blob/master/es5-shim.js) and [es5-sham.js](https://github.com/kriskowal/es5-shim/blob/master/es5-sham.js).

#What are promises and why should I use them?

You should use promises to turn this:

```js
readFile("file.json", function(err, val) {
    if( err ) {
        console.error("unable to read file");
    }
    else {
        try {
            val = JSON.parse(val);
            console.log(val.success);
        }
        catch( e ) {
            console.error("invalid json in file");
        }
    }
});
```

Into this:

```js
readFile("file.json").then(JSON.parse).then(function(val) {
    console.log(val.success);
})
.catch(SyntaxError, function(e) {
    console.error("invalid json in file");
})
.catch(function(e){
    console.error("unable to read file")
});
```

Actually you might notice the latter has a lot in common with code that would do the same using synchronous I/O:

```js
try {
    var val = JSON.parse(readFile("file.json"));
    console.log(val.success);
}
//Syntax actually not supported in JS but drives the point
catch(SyntaxError e) {
    console.error("invalid json in file");
}
catch(Error e) {
    console.error("unable to read file")
}
```

And that is the point - being able to have something that is a lot like `return` and `throw` in synchronous code.

You can also use promises to improve code that was written with callback helpers:


```js
//Copyright Plato http://stackoverflow.com/a/19385911/995876
//CC BY-SA 2.5
mapSeries(URLs, function (URL, done) {
    var options = {};
    needle.get(URL, options, function (error, response, body) {
        if (error) {
            return done(error)
        }
        try {
            var ret = JSON.parse(body);
            return done(null, ret);
        }
        catch (e) {
            done(e);
        }
    });
}, function (err, results) {
    if (err) {
        console.log(err)
    } else {
        console.log('All Needle requests successful');
        // results is a 1 to 1 mapping in order of URLs > needle.body
        processAndSaveAllInDB(results, function (err) {
            if (err) {
                return done(err)
            }
            console.log('All Needle requests saved');
            done(null);
        });
    }
});
```

Is more pleasing to the eye when done with promises:

```js
Promise.promisifyAll(needle);
var options = {};

var current = Promise.fulfilled();
Promise.map(URLs, function(URL) {
    current = current.then(function () {
        return needle.getAsync(URL, options);
    });
    return current;
}).map(function(responseAndBody){
    return JSON.parse(responseAndBody[1]);
}).then(function (results) {
    return processAndSaveAllInDB(results);
}).then(function(){
    console.log('All Needle requests saved');
}).catch(function (e) {
    console.log(e);
});
```

Also promises don't just give you correspondences for synchronous features but can also be used as limited event emitters or callback aggregators.

More reading:

 - [Promise nuggets](http://spion.github.io/promise-nuggets/)
 - [Why I am switching to promises](http://spion.github.io/posts/why-i-am-switching-to-promises.html)
 - [What is the the point of promises](http://domenic.me/2012/10/14/youre-missing-the-point-of-promises/#toc_1)
 - [Snippets for common problems](https://github.com/petkaantonov/bluebird/wiki/Snippets)
 - [Promise anti-patterns](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns)

#Error handling

This is a problem every promise library needs to handle in some way. Unhandled rejections/exceptions don't really have a good agreed-on asynchronous correspondence. The problem is that it is impossible to predict the future and know if a rejected promise will eventually be handled.

There are two common pragmatic attempts at solving the problem that promise libraries do.

The more popular one is to have the user explicitly communicate that they are done and any unhandled rejections should be thrown, like so:

```js
download().then(...).then(...).done();
```

For handling this problem, in my opinion, this is completely unacceptable and pointless. The user must remember to explicitly call `.done` and that cannot be justified when the problem is forgetting to create an error handler in the first place.

The second approach, which is what bluebird by default takes, is to call a registered handler if a rejection is unhandled by the start of a second turn. The default handler is to write the stack trace to stderr or `console.error` in browsers. This is close to what happens with synchronous code - your code doens't work as expected and you open console and see a stack trace. Nice.

Of course this is not perfect, if your code for some reason needs to swoop in and attach error handler to some promise after the promise has been hanging around a while then you will see annoying messages. In that case you can use the `.done()` method to signal that any hanging exceptions should be thrown.

If you want to override the default handler for these possibly unhandled rejections, you can pass yours like so:

```js
Promise.onPossiblyUnhandledRejection(function(error){
    throw error;
});
```

If you want to also enable long stack traces, call:

```js
Promise.longStackTraces();
```

right after the library is loaded.

In node.js use the environment flag `BLUEBIRD_DEBUG`:

```
BLUEBIRD_DEBUG=1 node server.js
```

to enable long stack traces in all instances of bluebird.

Long stack traces cannot be disabled after being enabled, and cannot be enabled after promises have alread been created. Long stack traces imply a substantial performance penalty, even after using every trick to optimize them.

Long stack traces are enabled by default in the debug build.

####How do long stack traces differ from e.g. Q?

Bluebird attempts to have more elaborate traces. Consider:

```js
Error.stackTraceLimit = 25;
Q.longStackSupport = true;
Q().then(function outer() {
    return Q().then(function inner() {
        return Q().then(function evenMoreInner() {
            a.b.c.d();
        }).catch(function catcher(e){
            console.error(e.stack);
        });
    })
});
```

You will see

    ReferenceError: a is not defined
        at evenMoreInner (<anonymous>:7:13)
    From previous event:
        at inner (<anonymous>:6:20)

Compare to:

```js
Error.stackTraceLimit = 25;
Promise.longStackTraces();
Promise.fulfilled().then(function outer() {
    return Promise.fulfilled().then(function inner() {
        return Promise.fulfilled().then(function evenMoreInner() {
            a.b.c.d()
        }).catch(function catcher(e){
            console.error(e.stack);
        });
    });
});
```

    ReferenceError: a is not defined
        at evenMoreInner (<anonymous>:7:13)
    From previous event:
        at inner (<anonymous>:6:36)
    From previous event:
        at outer (<anonymous>:5:32)
    From previous event:
        at <anonymous>:4:21
        at Object.InjectedScript._evaluateOn (<anonymous>:572:39)
        at Object.InjectedScript._evaluateAndWrap (<anonymous>:531:52)
        at Object.InjectedScript.evaluate (<anonymous>:450:21)


A better and more practical example of the differences can be seen in gorgikosev's [debuggability competition](https://github.com/spion/async-compare#debuggability).

####Can I use long stack traces in production?

Probably yes. Bluebird uses multiple innovative techniques to optimize long stack traces. Even with long stack traces, it is still way faster than similarly featured implementations that don't have long stack traces enabled and about same speed as minimal implementations. A slowdown of 4-5x is expected, not 50x.

What techniques are used?

#####V8 API second argument

This technique utilizes the [slightly misleadingly documented](https://code.google.com/p/v8/wiki/JavaScriptStackTraceApi#Stack_trace_collection_for_custom_exceptions) second argument of V8 `Error.captureStackTrace`. It turns out you can that the second argument can actually be used to make V8 skip all library internal stack frames [for free](https://github.com/v8/v8/blob/b5fabb9225e1eb1c20fd527b037e3f877296e52a/src/isolate.cc#L665). It only requires propagation of callers manually in library internals but this is not visible to you as user at all.

Without this technique, every promise (well not every, see second technique) created would have to waste time creating and collecting library internal frames which will just be thrown away anyway. It also allows one to use smaller stack trace limits because skipped frames are not counted towards the limit whereas with collecting everything upfront and filtering afterwards would likely have to use higher limits to get more user stack frames in.

#####Sharing stack traces

Consider:

```js
function getSomethingAsync(fileName) {
    return readFileAsync(fileName).then(function(){
        //...
    }).then(function() {
        //...
    }).then(function() {
        //...
    });
}
```

Everytime you call this function it creates 4 promises and in a straight-forward long stack traces implementation it would collect 4 almost identical stack traces. Bluebird has a light weight internal data-structure (kcnown as context stack in the source code) to help tracking when traces can be re-used and this example would only collect one trace.

#####Lazy formatting

After a stack trace has been collected on an object, one must be careful not to reference the `.stack` property until necessary. Referencing the property causes
an expensive format call and the stack property is turned into a string which uses much more memory.

What about [Q #111](https://github.com/kriskowal/q/issues/111)?

Long stack traces is not inherently the problem. For example with latest Q with stack traces disabled:

```js
var Q = require("q");


function test(i){
    if (i <= 0){
       return Q.when('done')
   } else {
       return Q.when(i-1).then(test)
   }
}
test(1000000000).then(function(output){console.log(output) });
```

After 2 minutes of running this, it will give:

```js
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - process out of memory
```

So the problem with this is how much absolute memory is used per promise - not whether long traces are enabled or not.

For some purpose, let's say 100000 parallel pending promises in memory at the same time is the maximum. You would then roughly use 100MB for them instead of 10MB with stack traces disabled.For comparison, just creating 100000 functions alone will use 14MB if they're closures. All numbers can be halved for 32-bit node.

#Development

For development tasks such as running benchmarks or testing, you need to clone the repository and install dev-dependencies.

Install [node](http://nodejs.org/), [npm](https://npmjs.org/), and [grunt](http://gruntjs.com/).

    git clone git@github.com:petkaantonov/bluebird.git
    cd bluebird
    npm install

##Testing

To run all tests, run `grunt test`. Note that 10 processes are created to run the tests in parallel. The stdout of tests is ignored by default and everything will stop at the first failure.

Individual files can be run with `grunt test --run=filename` where `filename` is a test file name in `/test` folder or `/test/mocha` folder. The `.js` prefix is not needed. The dots for AP compliance tests are not needed, so to run `/test/mocha/2.3.3.js` for instance:

    grunt test --run=233

When trying to get a test to pass, run only that individual test file with `--verbose` to see the output from that test:

    grunt test --run=233 --verbose

The reason for the unusual way of testing is because the majority of tests are from different libraries using different testing frameworks and because it takes forever to test sequentially.


###Testing in browsers

To test in browsers:

    cd browser
    setup

Then open the `index.html` in your browser. Requires bash (on windows the mingw32 that comes with git works fine too).

You may also [visit the github hosted page](http://petkaantonov.github.io/bluebird/browser/).

Keep the test tab active because some tests are timing-sensitive and will fail if the browser is throttling timeouts. Chrome will do this for example when the tab is not active.

##Benchmarks

Currently the most relevant benchmark is @gorkikosev's benchmark in the article [Analysis of generators and other async patterns in node](http://spion.github.io/posts/analysis-generators-and-other-async-patterns-node.html). The benchmark emulates a situation where n amount of users are making a request in parallel to execute some mixed async/sync action. The benchmark has been modified to include a warm-up phase to minimize any JITing during timed sections.

You can run the benchmark with:

    bench spion

While on the project root. Requires bash (on windows the mingw32 that comes with git works fine too).

Node 0.11.2+ is required to run the generator examples.

Another benchmark to run is the [When.js benchmarks by CujoJS](https://github.com/cujojs/promise-perf-tests). The reduce and map have been modified from the original. The benchmarks also include warmup-phases.

    bench cujojs

While on the project root. Requires bash (on windows the mingw32 that comes with git works fine too).

##What is the sync build?

You may now use sync build by:

    var Promise = require("bluebird/zalgo");

The sync build is provided to see how forced asynchronity affects benchmarks. It should not be used in real code due to the implied hazards.

The normal async build gives Promises/A+ guarantees about asynchronous resolution of promises. Some people think this affects performance or just plain love their code having a possibility
of stack overflow errors and non-deterministic behavior.

The sync build skips the async call trampoline completely, e.g code like:

    async.invoke( this.fn, this, val );

Appears as this in the sync build:

    this.fn(val);

This should pressure the CPU slightly less and thus the sync build should perform better. Indeed it does, but only marginally. The biggest performance boosts are from writing efficient Javascript, not from compromising determinism.

Note that while some benchmarks are waiting for the next event tick, the CPU is actually not in use during that time. So the resulting benchmark result is not completely accurate because on node.js you only care about how much the CPU is taxed. Any time spent on CPU is time the whole process (or server) is paralyzed. And it is not graceful like it would be with threads.


```js
var cache = new Map(); //ES6 Map or DataStructures/Map or whatever...
function getResult(url) {
    var resolver = Promise.pending();
    if (cache.has(url)) {
        resolver.fulfill(cache.get(url));
    }
    else {
        http.get(url, function(err, content) {
            if (err) resolver.reject(err);
            else {
                cache.set(url, content);
                resolver.fulfill(content);
            }
        });
    }
    return resolver.promise;
}



//The result of console.log is truly random without async guarantees
function guessWhatItPrints( url ) {
    var i = 3;
    getResult(url).then(function(){
        i = 4;
    });
    console.log(i);
}
```

#Optimization guide

todo

#License

Copyright (c) 2013 Petka Antonov

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:</p>

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
