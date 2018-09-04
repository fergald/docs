# Explainer: queueMicrotask


## Motivation

There are several existing ways
to queue a microtask
([introduction to microtasks](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)):



*   `Promise.result().then(callback)`
*   Register a mutation observer and trigger it with a mutation

The library [asap](https://github.com/kriskowal/asap) uses these tricks
to provide a cross-browser way of queueing a microtask.

Developers and frameworks are already using these methods to queue microtasks.
The platform should support queueing microtasks directly as a primitive operation.

Reasons to expose this API directly

*   Roughly the same semantics are available via Promise etc
    but we should make this lower level primitive directly available
    rather than making people emulate it with via higher level APIs
*   Emulating this via Promise causes exceptions thrown in the callback
    to be converted into rejected promises.
*   Emulating via Promise causes unnecessary objects to be created and detroyed.
    Other methods of emulating this also cause wasted effort.
*   The other semantically distinct async callback queue APIs are available directly,
    microtask not being available directly is an anomaly
*   Providing a direct API could lead to more widespread understanding of the distinction between
    *   setTimeout(callback,0) - task
    *   requestAnimationFrame(callback) - RAF
    *   queueMicrotask(callback) - microtask


## Proposal

This [proposal](https://github.com/whatwg/html/issues/512) would add an API
to directly queue a microtask
without the need for any tricks. E.g.


```
queueMicrotask(callback);
```


This  API would be available to JS in web pages via `Window`
and to JS in workers via `WorkerGlobalScope`.


## Usage

Microtasks are useful when you need async ordering behaviour,
but want the work done as soon as possible.
Beware that doing large amounts of work in microtasks
is just as bad for jank and interactivity
as doing that work in-place in your JS.
Depending on the desired effect,
`setTimeout(someFunction, 0)` or `requestAnimationFrame(someFunction)`
may be the correct choice.

Example

```
function microtask() {
  console.log("microtask");
}

function task() {
  console.log("task");
}

setTimeout(task, 0);
queueMicrotask(microtask);
```

Will result in the following order of operations

1.  Log "microtask"
1.  Return to the event loop and allow rendering/input callbacks to occur if appropriate
1.  Log "task"
