# OffscreenCanvas Animation Proposals

This proposal aims to provide a reliable mechanism for driving animations using OffscreenCanvas in a Worker.

## Use Case Description

1. An OffscreenCanvas is used in a worker to produce an animation sequence. The animation frames are propagated via commit() to a placeholder canvas element that is in a visible document.
2. An OffscreenCanvas is used in a worker to produce an animation sequence. The web application displays multiple views of a 3D scene. To avoid resource duplication, all views must be rendered from the same WebGL context. The display updates of the different views may be required to be synchronised.
3. An OffscreenCanvas is used in a worker to produce an animation sequence that is displayed to a WebVR device.
4. An OffscreenCanvas is used in a worker to produce an animation sequence that is displayed simultaneously to a WebVR device and to a placeholder canvas element that is in a visible document. The VR device and the main display run at different refresh rates.
5. An OffscreenCanvas is used to perform non-graphics computations (e.g. physics simulation) that need to run at the same rate as display refresh because the results of the computation are consumed to drive an animation on the main thread
6. An OffscreenCanvas is used to produce an animation sequence. The animation frames are propagated via commit() to a placeholder canvas that is in a document that is visible in a different window on a different display device from the window of the browsing context where the OffscreenCanvas object is used.  This may occur by transferring an OffscreenCanvas object via a MessageChannel.

Use Case Requirements:
* Animation frames need to be produced at regular intervals that match the frame rate of the display device.
* If the graphics sub-system (e.g. GPU driver) is asynchronous and cannot execute rendering work fast enough to keep up with the display's frame rate, it must be possible for user agents to trigger a backpressure mechanism to throttle the animation to avoid accumulation a rendering backlog. Otherwise the app may experience high display latency and run-away resource consumption.
* The API must be extensible in order to accommodate future improvements, such as an option to specify a frame rate.
* Prevent overdraw: every rendered frame must be displayed. If frames were dropped, that would mean that compute power was wasted.

## Current Usage and Workarounds
Currently, it is possible to use setTimeout or setInterval in a worker to invoke an animation callback at a regular interval. However, since this mechanism is not driven by the display, the following issues arise:
* The display device's refresh rate needs to be guessed.
* Even if the correct rate is guessed, drift in the timing will cause dropped frames and skipped frames.
* In GPU-accelerated use cases, it is possible for the rendering script to run at a rate that the GPU cannot keep-up with, which may cause the accumulation of a multi-frame rendering backlog, resulting in high latency and eventually OOM crashes.  A possible solution to this is for the user agent to implement a throttling mechanism, in which case the worker's event loop may be periodically de-scheduled while the GPU catches up.  Such de-scheduling is bad because it prevents the worker from doing other work.

Another solution is to have a requestAnimationFrame loop in the browsing context's event loop that posts a message to the worker at each animation iteration.
* This mechanism may add undue latency to the signal, especially when the browsing context's event loop is busy, which completely destroys one of the key advantages of using OffscreenCanvas in a worker.
* The the frame rate in the browsing context's event loop may be higher than the worker can keep up which which will require a throttling mechanism to be implemented in script
* As with setTimeout/setInterval, it is possible for the rendering script to run at a rate that the GPU cannot keep-up with.

## Considered Solutions

### WorkerGlobalScope.requestAnimationFrame

This would replicate the behavior of window.requestAnimationFrame, but in a worker.

#### Problems:
* It is not clear which display device to synchronize with in this context, unlike with the window interface which represents a view that is displayed on a specific display device. In the case of a dedicated worker, we could assume that the display that the worker is connected to is the display of the window of the browsing context that owns the dedicated worker.  For shared workers however, relation to a display is less clear.
* This approach would require adding the notion of a graphics update phase to the worker event loop processing model. It would be unfortunate to add this complexity to workers, which are not inherently designed for graphics.

### OffscreenCanvas.requestAnimationFrame

Works much like window.requestAnimationFrame, except that the scheduling of callbacks is per object. This allows multiple Offscreen canvas objects in the same worker to be synchronized with different displays.  The display that the OffscreenCanvas would synchronize with is the device where its placeholder canvas is displayed.

#### Processing Model

The requestAnimationFrame() and cancelAnimationFrame() methods shall be spec'ed almost identically to their Window interface counterparts, except that the callback list would be stored in the OffscreenCanvas object.

The main difference with respect to the Window.requestAnimationFrame processing model is in how the callbacks are scheduled. In a browsing context, the animation callbacks are coordinated with the graphics update in the [https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model event loop processing model]. Such coordination shall not exist in workers.  

For OffscreenCanvases, the user agent will schedule an "animation task" to run all of the OffscreenCanvas's pending animation callbacks. in a way the respects the following constraints:
* No overdraw: An animation frame committed from an animation task shall not replace an animation frame from a previous animation task until that previous frame has been rendered to the display device.
* animation tasks shall be scheduled at the highest possible rate that can be maintained without going into overdraw and without accumulating a backlog of more than one pending frame.

##### Special Cases
* When rAF is invoked on an OffscreenCanvas that does not have a placeholder canvas and is not linked to a VRLayer, throw an InvalidStateError. The OffscreenCanvas content must be composited for the rAF processing model to make sense.
* Attempting to transfer an OffscreenCanvas object with a non-empty animation callback list throws an InvalidStateError.
* Attempting to construct a VRLayer using an OffscreenCanvas object with a non-empty animation callback list throws an InvalidStateError.
* When the OffscreenCanvas is associated with a VRLayer, all calls to {request|cancel}AnimationFrame must be forwarded to the VRLayer's VRDisplay's {request|cancel}AnimationFrame methods.  This implies that when the OffscreenCanvas simultaneously is visible through a placeholder canvas and a VR device, the animation loop is driven by the VR device.
* The animations tasks for different OffscreenCanvas objects that live in the same event loop are not necessarily synchronized. 

#### Issues
Calling commit() on a given OffscreenCanvas multiple times in the same animation frame is problematic.  Possible way of handling the situation:
* Drop all commits but the last one (or the first one?)
* Queue multiple frames and wait for all of them to have be displayed before scheduling the next animation task
* Throw an exception

What to do if commit() is not called from within the animation callback?  This is problematic because the scheduling of the next animation frame is usually queued behing the completion of the current frame. Possible courses of action:
* Do an implicit commit() at end of script task.
* Repeat the previous frame
* Schedule the next animation frame immediately.
* Prevent this from ever happening: Let OffscreenCanvas object have a needsCommit flag that is initially false. Set needsCommit to true at the beginning of an animation task. Set needsCommit to false when commit is called. When requestAnimationFrame is called, throw an exception if needsCommit is true.

Should it be possible to commit() the contents of other canvases from within a rAF callback? 

The design of requestAnimationFrame was mean to allow multiple independent rendering loops in parallel which is why there is a notion of a callback list.  This aspect of the processing model is overkill in the case where requestAnimationFrame is per-object instead of global.

For new async Web APIs, it is preferred to use Promises rather than callbacks.

## Proposed Solution

### OffscreenCanvas.commit() to return a promise
An alternate solution would be to have commit() return a promise that gets resolved when it is time to begin rendering the next frame.  This single API entry-point provides the necessary flexibility to handle continuous animations as well as sporadic updates.  Functionally, this API solves all the same use cases as OffscreenCanvas.requestAnimationFrame, with the following added benefits:
* Not possible to schedule an animation frame without comitting (implementation no longers needs to worry about that case).
* Simpler processing model (no callback queue)
* The use of promises allows more modern syntaxes and code constructs. For example: async/await.

Continuous animation example:

```
function animationLoop() {
  // draw stuff
  (...)
  ctx.commit().then(animationLoop);
  // do post commit work
  (...)
}
```

Another possibility is to use the async/await syntax:

```
async function animationLoop() {
  var promise;
  do {
    //draw stuff
    (...)
    promise = ctx.commit()
    // do post commit work
    (...)
  } while (await promise);
}
```

To animate multiple canvases in lock-step, one could do this, for example:
 
```
function animationLoop() {
  // draw stuff
  (...)
  Promise.all([ctx1.commit(), ctx2.commit()]).then(animationLoop);
}
```

For occasional update use cases, it is just a matter of ignoring the promise returned by commit() and to drive the animation using another signal, for example a network event.  In the case where multiple calls to commit are made in the same frame interval, the user agent skips frames in order to avoid accumulating a multi-frame backlog, as described in the processing model below.

#### Processing Model

An OffscreenCanvas object has a ''pendingFrame'' internal slot that stores a reference to the frame that was captured by the last call to commit(). The reference is held until the frame is actually committed. 'pendingFrame' is initially unset.

An OffscreenCanvas object has a ''pendingPromise'' internal slot that stores a reference to the promise that was returned by the last call to commit(). ''pendingPromise'' is initially unset, and its reference is only retained while the promise is in the unresolved state.

The ''BeginFrame'' signal is a signal that is dispatched by the UserAgent to a specific OffscreenCanvas when it is time to render the next animation frame for the OffscreenCanvas.

When commit() is called:
* Let ''frame'' be a copy of the current contents of the canvas.
* Set the OffscreenCanvas object's ''pendingFrame'' to be a reference to ''frame''.
* If ''pendingPromise'' is set, then return ''pendingPromise''.
* Set ''pendingPromise'' to be a newly created unresolved promise object.
* Create a commit task that runs the steps to '''commit a frame''' for this OffscreenCanvas and add the newly created task to a global ''commit queue''. There is a single global ''commit queue'' per event loop. The tasks in the ''commit queue'' are to be executed, and the queue flushed, at the end of each script task (to be executed before returning control to the event loop), and whenever the "await" statement is invoked. When this commit task is created, ''pendingFrame'' is unset.
* Return ''pendingPromise''.

When the ''BeginFrame'' signal is to be dispatched to an OffscreenCanvas object, the UserAgent must queue a task on the OffscreenCanvas object's event loop that runs the following steps: 
* If the OffscreenCanvas object's ''pendingFrame'' is set, then run the following substeps:
    * Run the steps to '''commit a frame''' for the OffscreenCanvas object.
    * Return.
* If ''pendingPromise'' is not set then return.
* Resolve the promise referenced by the OffscreenCanvas object's ''pendingPromise''.
* Unset ''pendingPromise''.

When the user agent is required to run the steps to '''commit a frame''', it must do what is currently spec'ed as the steps for commit().

When the tasks in a commit queue are executed, if there are multiple pending commits, they must all be executed atomically, such that committed frames that are displayed on the same device are guaranteed to appear on the display during the same display refresh cycle.

This processing model takes care of the unresolved issues with the OffscreenCanvas.requestAnimationFrame solution because it makes it safe to call commit at any time by providing the following guarantees:
* In cases of overdraw (commit() called at a rate higher than can be displayed), frames may be dropped to ensure low latency (no more than one frame of backlog).
* The frame captured by the last call to commit after the end of an animation sequence is never dropped. In other words, when animation stops, it is always the most recent frame that is displayed.

### Document history

This document was forked from a [proposal](https://wiki.whatwg.org/wiki/OffscreenCanvas.requestAnimationFrame) that was first posted to the WHATWG wiki, which incorporated feedback from the associated [thread](https://github.com/whatwg/html/issues/2139) on the WHATWG Github issue tracker.
