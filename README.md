# Input For Workers
This repository contains a proposed solution for exposing user input events to the worker and worklets on the web.

# Problem Definition
Today on the Web Platform only the main thread has access to the DOM elements and receives their related input events. In particular, Web Workers and Worklets do not have access to these. If we allow workers to receive input events for a given DOM node we enable many latency sensitive applications to leverage workers.
As a result if developers want to do some event dependent logic like drawing on an off-screen canvas or using fetch in their worker thread they need to listen to those events on the main thread and postMessage them across to the worker thread. Here the worker is not only going to be blocked by the main thread to handle the events first but also suffers from the latency introduced by the extra thread hop. This is particularly unfortunate for the input events where the latency is important.

# Use Cases
- Low-Latency Drawing: Offscreen Canvas now provides workers with an off-main-thread drawing context. However any input driven drawing is still blocked by the main thread getting the user events/inputs and forwarding them to the worker to be drawn on the off screen canvas. With this latency paper-like drawing cannot be achieved on the web pages via workers.
- Gaming and XR: Similar to the drawing use case, low latency input handling and drawing is important for these applications and they can benefit from access to both input and canvas in worker threads.
- Page Analytics: Analytic scripts that process events without needing to access DOM. Some analytics scripts want to send user input over the network and only need access to the raw coordinates and state of the pointer including the buttons. In this case accessing DOM or knowing whether the default actions were prevented is not important.
- Interactive Animations: Animation Worklet enables scripted animation off-main-thread. By providing input events to Animation Worklet we enable rich interactive animation driven by the user input to be off-main thread to ensure smoothness and responsiveness. For many such effects passive access to pointer input events is sufficient.
- Low-Latency Interactive Audio: Audio Worklet provides a rich off-main-thread audio context which is being used to enable high-fidelity rich audio applications on the web. If audio worklet were able to process user input (including messages from Web MIDI API) directly it may help reduce the latency from user input to audio output which is important for interactive audio applications. 

# Event Delegation
Here we propose event delegation scheme that assumes there is no DOM access in workers. The proposal introduces a generic event delegation mechanism that in theory works for all DOM events. However to be practical we intend to first focus on enabling this for input events starting with PointerEvents. This addresses key use cases and gives us a chance to prove and perfect the model while also leaving the door open to enable delegation for more event types that can benefit from this capability.

Here we allow workers to observe events in parallel to main thread without the ability to stop their propagation or to prevent default. In this design the main thread continues to be the sole owner in control of the event and event flow which means there is no need to block main thread on the workers when propagating the event. Similarly, in most situations the worker can process and handle events without delay or coordination with main thread.

This is similar to allowing workers to register a passive event handler with the added restriction that the handler also cannot stop propagation. 

# Considerations on Fork Point
A key consideration in this approach is at which point in the propagation path the event is forked and forwarded to the worker. This fork point choice controls the trade-off between the degree of main/worker control in the delegation process and the performance isolation. Here are some of the most interesting options:
- a) Fork at root: Basically fork and delegate the event at the root of propagation path. Here main has no ability to prevent the delegation and the delegated listener always receives the event. It provides maximum performance isolation (no footguns) and as a minor benefit, the delegating agent does not need to compute propagation path before making a decision on whether the event should be delegated. Note that main still controls delegation at target granularity but not at the event granularity. Other solutions in the list provide more granular per event control.
- b) Fork at target node during capturing phase: Fork the event only when it would reach the target node. Gives parent nodes and the target node in DOM tree the ability to prevent delegation on the main thread by registering a capturing event listener and preventing propagation. This provides some controls to the main at the cost of making the worker performance only isolated from main when there are no capturing event listener in the propagation path before the target node. In other words, in this model the listeners in the worker are considered capturing listeners that run after main listeners.
- c) Fork at target node during bubbling Phase: Similar to model (b) but during bubbling phase. This gives main even more granular control by allowing parent and child nodes the ability to prevent delegations. This loses most of performance isolations since bubbling event listeners are prevalent on the web. In other words, in this model the listeners in the worker are considered bubbling listeners that run after main listeners.
- d) Worker’s choice: Allow the worker to choose between (b) and (c) based on the type of listener (i.e. capturing vs non-capturing) it registers.

(a) has the strongest performance isolation and perhaps simplest to implement however it provides least amount of control to main and deviates the most from the well-understood DOM Event propagation model. As we move from (a) to (d), we are trading performance isolation in favor of richer control at the main and more alignment with existing event propagation model. We recommend against (c) because it harms the performance isolation which is a key goal of this proposal. Both (a) and (b) are acceptable choices. (d) is interesting since it gives the power to developer but this also means it comes with all the same footguns. Particularly, if the delegetated target is the window node all of these proposals will be equivalent.

# Proposed API

```webidl
interface mixin EventDelegate {
  void addEventTarget(EventTarget target, EventDelegationOptions? option);
  void removeEventTarget(EventTarget target);
};

// This dictionary allows us to add options in the future if needed.
dictionary EventDelegationOptions {
  any context; // To accommodate for any additional context data from the other thread.
};

interface Worker includes EventDelegate;
interface WorkletAnimation includes EventDelegate;
interface AudioWorkletNode includes EventDelegate;

// The opaque proxy representing an event target in worker.
interface DelegatedEventTarget : EventTarget {};
```

and the API will be used like this in the main thread:

```javascript
var t1 = document.getElementById("div1");

var myWorker = new Worker("worker.js");
myWorker.addEventTarget(t1);
```
and like this in the worker:

``` javascript
self.addEventListener("eventtargetadded",(event) => {
    event.target.addEventListener(
           "pointermove",
           (e) => {
               // Handle event e
           }
});
```
The above APIs allow the worker to get the targets separately and add any particular event listener they want to for each of them. This enables the user agent to optimize for the events not required by a worker if they can. Note that the EventDelegationOptions is empty for now but it is there to let us add more options to the API in the future.

# Stripping DOM References from Events
As part of this proposal we are providing access to events and event targets in the worker context. To make this work, both the events and targets objects that are delivered to the worker need to have all of their DOM related properties and functions stripped off (or replaced). 

Here are some key properties and functions with suggested changes:
-  dispatchEvent: becomes a no-op for delegated targets in the worker. 
- Target, currentTarget, sourceElement, composedPath: These are set to null as they might point into other elements that worker doesn’t have any reference to. An alternative may be to have a simple opaque ID (See Element.prototype.uniqueID Proposal)
- stopPropagation(), stopImmediatePropagation(), preventDefault(): This become a no-op similar to how passive event handler make preventDefault a no-op.
- coordinates: Screen coordinates (screenX/Y) of the event will remain unchanged.  Other coordinates may be changed and/or outdated in certain cases, see the section below on computing coordinates.
relatedTarget, view: Set to null

The list above is focused on PointerEvents properties because as discussed earlier we intend to to start with delegating PointerEvents. However there are a large number of DOM Event types with a varying list of properties that may need to be stripped. One way to scale this is to use a similar approach to Object Serialization for sending objects to a worker. Basically every event type can indicate if it is “delegatable” and specify the process by which an event object can be processed so that it can be used in worker.

Note that the above forking scheme can also work for synthetic events but for simplicity sake our initial focus will be on delegating trusted events.

# Addressing Double Handling Problem
A general issue with only providing passive event handling is that the event handler on the worker side can not prevent propagation or the default action. This can lead to having multiple handlers processing the same event a.k.a double handling when we also have handlers on the main thread.

This is a special case of the more general problem of coordinating actions in the presence of concurrency that exists with any usage of workers. Ultimately dealing with this problem requires application specific logic that uses the underlying platform’s concurrency primitives such as message passing. While this proposal makes it easier to have more concurrency in certain area previously not possible but it does not introduce new mechanisms to make dealing with concurrency easier. We think such mechanisms are orthogonal to the proposal here.

A simple and common way to avoid double handling is to have a single handler. Note that this can be achieved fairly without additional syntax:
- Principal handler is main: Developer ensures no action is taken in the worker that interfere with main thread actions.
- Principal handler is worker: Install a handler on main thread that unconditionally prevents propagation or default action or make sure no other handler is run on the main thread.

Note that the latter pattern may benefit from being declarative. This is possible to do by allowing additional options for event delegation or even more broadly for all event handling

# Computing coordinates for dispatched events
Most use cases we mentioned above require event coordinate information, so it’s important to be highlight how a Worker can interpret the coordinates.  Because the container of the event target is maintained by a different thread possibly running at a different “speed”, in general it is impossible to guarantee that any coordinate that is relative/connected to the part of the DOM outside the event target would be sensible.  Therefore:
- Raw event coordinates are not dependent on the rest of the DOM, dispatched events will have screenX/Y unchanged without any problem,
- Other coordinates are all relative (clientX/Y is relative to viewport, offsetX/Y relative to top-left of event target, pageX/Y relative to top-left of page), so they would be outdated (if they are dispatched unchanged) when the position of event target changes during dispatch (e.g. through scrolling) and/or the involved threads are out-of-sync. 

# Code Examples

Low Latency Drawing with Off-Screen Canvas
Here is a code to draw user input on a canvas completely off the main thread

DOM and the main thread:
```javascript

// This canvas could be embedded within an iframe if only the root window events are allowed to be delegated to the worker.
<canvas id="canvas"></canvas>

<script>
  var worker = new Worker("worker.js");
  var canvas = document.getElementById("canvas")

  var handler = canvas.transferControlToOffscreen();
  worker.postMessage({canvas: handler}, [handler]);
  worker.addEventTarget(canvas);
</script>
```

Worker thread:
```javascript
var context;

addEventListener("message", (msg) => {
  if (msg.data.canvas)
    context = msg.data.canvas.getContext("2d");
});

addEventListener("eventtargetadded", ({target}) => {
  target.addEventListener("pointermove", onPointerMove);
});

addEventListener("eventtargetremoved", ({target}) => {
  target.removeEventListener("pointermove", onPointerMove);
});

function onPointerMove(event){
  // Use event.clientX/Y or offsetX/Y to draw things on the context.
  context.beginPath();
  context.arc(event.offsetX, event.offsetY, 5, 0, 2.0* Math.PI, false);
  context.closePath();
  context.fill();
  context.commit();
}
```

Scale/Rotate Gestures using AnimationWorklet

Here an animation worklet is used to create a component that allows an image to be directly manipulated (e.g., scaled and rotates) by multi-touch gestures. 

DOM and the main thread:
```javascript
<img id='target'>

<script>
await CSS.animationWorklet.addModule('worklet.js');
const target = document.getElementById('target');

const rotateEffect = new KeyFrameEffect(
   target, {rotate: ['rotate(0)', 'rotate(360deg)']}, {duration: 100, fill: 'both' }
);
const scaleEffect = new KeyFrameEffect(
  target, {scale: [0, 100]}, {duration: 100, fill: 'both' }
);

// Note the worklet animation has no timeline since the animation is not time-based.
const animation = new WorkletAnimation('scale_and_rotate', [rotateEffect, scaleEffect],  null);
animation.play();

// Delegate pointer events for target to worklet animation.
animation.addEventTarget(target);
</script>
```

Worker thread:
```javascript
// Made up library that given pointer event sequence can compute an active gesture similar to Hammer.js 
import {Recognizer} from 'gesture_recognizer.js'


registerAnimator('scale_and_rotate', class {
  constructor() {
    this.gestureRecognizer = new Recognizer();
    this.addEventListener("eventtargetadded", (event) => {
       for (type of ["pointerdown", "pointermove", "pointerup"])  {
          event.target.addEventListener(type, (e) => {
            this.gestureRecognizer.consumeEvent(e)));
          });
       }
   });
  }


  animate(currentTime, effects) {
    // Note that currentTime is undefined and unused.

    // Get current recognized gesture value and update rotation and scale effects accordingly.
    const gesture = this.gestureRecognizer.activeGesture;
    if (!gesture) return;
 
    effect.children[0].localTime = gesture.rotate * 100;
    effect.children[1].localTime = gesture.scale;
  }
});
```

## References
* [More detailed document](https://docs.google.com/document/d/1byDy6IZqvaci-FQoiRzkeAmTSVCyMF5UuaSeGJRHpJk/edit#) about this feature and how it fits in Chromium.

## Status
* This feature is being discussed with other browser vendors and developers..
