
# Blick

Blick is a JavaScript component animation controller based on the lower level
animation frame handler.
The animator batches reads and writes by coordinating measurement, transitions,
animation, drawing, and redrawing.
Each component retains its own part of the animation controller so there is no
garbage collector churn frame to frame.
This coordinated draw cycle enables a wide variety of cooperative scenarios:

- Components that need to measure their bounding boxes before drawing.
- Components that need to redraw on every animation frame because their model
  changes continuously.
- Components that need to draw initially then redraw in response to discrete
  model changes.
- Components that need to set their initial position (with transitions
  disabled) in one animation frame, and set their target position (with
  transitions enabled) in a subsequent animation frame.
- Components that need to resize text until it fits the bounding box exactly.


## Installation

```
npm install --save blick
```

## Usage

Just to get started, create an animation controller and add a component.
This will create and return an animation controller for your component.

```js
var Animator = require("blick");
var animator = new AnimationController();
component.animator = animator.add(component);
```

The animation controller expects that the component will request that the
animator call certain methods to perform animation, transitions, or rendered
document queries.

```js
animator.requestMeasure();
animator.requestTransition();
animator.requestAnimation();
animator.requestDraw();
animator.requestRedraw();

animator.cancelMeasure();
animator.cancelTransition();
animator.cancelAnimation();
animator.cancelDraw();
animator.cancelRedraw();

component.measure();
component.transition();
component.animate();
component.draw();
component.redraw();
```

In each animation frame, each of the following phases will be executed.
In each phase, every component that has requested that phase will execute.

-   **measure:** The animation controller will call `measure()` on every
    component that has requested an opportunity to measure its rendered state on
    the document in the first phase of the next animation frame.
    This may include measuring its size, position, or computed styles.
    The component may at this point decide whether to request or cancel any
    subsequent phases depending on whether the measured state reflects the
    intended state.
    The component will need to request another measurement if it wishes to do so
    on a subsequent animation frame.
-   **transition:** The animation controller will call `transition()` on every
    component that has requested a transition on the next animation frame.
    If a component has requested that it have an opportunity to draw or redraw
    before it starts a transition, this phase will be skipped for this animation
    frame and the transition will remain scheduled.
    Once the transition has been initiated, the component will need to request
    another opportunity to transition if it wishes to make a change in a
    subsequent animation frame.
-   **animate:** The animation controller will call `animate()` on every
    component that has requested an opportunity to animate on every animation
    frame.
    The component must explicitly cancel animation if it wishes to stop
    animating.
-   **draw:** The animation controller will call `draw()` on every component
    that has requested an opportunity to draw itself onto the document on the
    next animation frame.
    Drawing is intended to occur only once when a component enters the document.
-   **redraw:** The animation controller will call `redraw()` on every component
    that has requested an opportunity to redraw itself onto the document on the
    next animation frame.
    If the component wishes to redraw, it must request another opportunity to
    redraw.

Typically, the animator will be shared by all of the components in a scope
through some form of dependency injection.
For example, if you are using [Gutentag][], you would add the animator to the
scope shared by all or some components.

[Gutentag]: https://github.com/gutentags/gutentag

```js
var Scope = require("gutentag/scope");
var Blick = require("blick");
var scope = new Scope();
scope.animator = new Blick();
```

This component requests a color change each time its value changes.

```js
function ColorComponent(body, scope) {
    this.element = body.ownerDocument.createElement("div");
    body.appendChild(this.element);
    this.animationController = scope.animator.add(this);
    this._value = null;
}

ColorComponent.prototype.destroy = function destroy() {
    this.animationController.destroy();
};

Object.defineProperty(ColorComponent.prototype, "value", {
    get: function () {
        return this._value;
    },
    set: function (value) {
        this._value = value;
        this.animationController.requestDraw();
    }
});

ColorComponent.prototype.draw = function () {
    this.element.actualNode.style.color = this._value;
};
```

<!-- TODO show a component that has a separate firstDraw and draw -->

<!-- TODO show a component that resizes its text to fill a view by measuring
its content either before or after drawing -->

## Explanation

On one level, it is useful for components to modify the document no more
frequently than those changes can be perceived.
At this level of consideration, the developer trades time spent altering the
document for time and memory spent planning to alter the document.

Browsers will typically avoid rendering changes to a page more frequently than
60 frames per second.
Regardless, the user has at most 17 miliseconds of time on the CPU in the main
JavaScript worker to prepare and apply each frame of animation on a page.

Certain combinations of alterations to a document and queries to the state of
the rendered document can force the document to render more frequently than
necessary.
As such, on a second level, it is necessary for components to avoid causing the
page to render (reflow or redraw) more frequently than those changes can be
perceived.
This can be achieved by coordinating changes and queries to the state of the
rendered document, such that all components query together either before or
after all components have an opportunity to draw themselves.

On a third level, the design of CSS animations pose a certain challenge.
While an ideal animation API in JavaScript would permit each component to set
its initial state, final state, and the duration of the change as an atomic
operation, CSS transitions require the component to yield to the renderer
between when they set the initial state and when they initiate a transition.

This is only ever a problem if a component needs to start a transition from a
different position than its current position.
This is often necessary for games or visualizations of live data.

The following component receives periodic updates about its position and
velocity at the current time (not shown: computing the current position and
velocity based on the last known position and velocity reported by a remote
authority).
The depicted vectors come from [ndim][].

[ndim]: https://github.com/kriskowal/ndim

```js
Component.prototype.updateFrequency = 1000;
Component.prototype.update = function (position, velocityPerMs) {
    this.initialPosition.become(position);
    this.finalPosition
        .become(velocity)
        .scale(this.updateFrequency)
        .addThis(position);
    this.animationController.requestRedraw();
    this.animationController.requestTransition();
};

Component.prototype.redraw = function () {
    this.element.style.left = initialPosition.x + 'px';
    this.element.style.top = initialPosition.y + 'px'
    this.element.style.transition = 'none';
};

Component.prototype.transition = function () {
    this.element.style.left = finalPosition.x + 'px';
    this.element.style.top = finalPosition.y + 'px';
    this.element.style.transition = 'all ' + this.updateFrquency + 'ms linear');
};
```

To this end, Blick provides a system where components can coordinate when they
alter or query the rendered document, as well separating "redraw" and
"transition" animation frames for each component.
Also, because animation is sensitive to garbage collection churn, the animation
library associates each component with a reusable animation controller object
that is reused for the life of that component.

This library does nothing to ensure that components animated in any specific
relative order since information about rendering should only flow from phase to
phase.

## Copyright and License

Copyright (c) 2015 by Kris Kowal and contributors.
All rights reserved.
MIT License.

