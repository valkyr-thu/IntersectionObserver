# Intersection Observers Explained

## What's All This About?

This repo outlines an API that can be used to understand movement of DOM elements relative to another element or the browser top level viewport. Changes are delivered asynchronously and are useful for understanding the visibility of elements, managing pre-loading of DOM and data, as well as deferred loading of "below the fold" page content.

## Observing Position

The web's traditional position calculation mechanisms rely on explicit queries of DOM state. Some of these are known to cause style recalcuation and layout and, frequently, are redundant thanks to the requirement that scripts poll for this information.

A body of common practice has evolved that relies on these behaviors, however, including (but not limited to):

  * Observing the location of "below the fold" sections of content in order to lazy-load content.
  * Implementing data-bound high-performance scrolling lists which load and render subsets of data sets. These lists are a central mobile interaction idiom.
  * Calculating element visibility. In particular, [ad networks now require reporting of ad "visibility" for monetizing impressions](http://www.iab.net/iablog/2014/03/viewability-has-arrived-what-you-need-to-know-to-see-through-this-sea-change.html). This has led to many sites abusing scroll handlers, [synchronous layout invoking readbacks](http://gent.ilcore.com/2011/03/how-not-to-trigger-layout-in-webkit.html), and resorting to exotic plugin-based solutions for computing "true" element visibility (as a fraction of the element's intended size).

These use-cases have several common properties:

  1. They can be represented as passive "queries" about the state of individual elements with respect to some other element (or the global viewport).
  1. They do not impose hard latency requirements; that is to say, the information can be delayed somewhat asynchronously (e.g. from another thread) without penalty.
  1. They are poorly supported by nearly all combinations of existing web platform features, requiring extraordinary developer effort despite their widespread use.

A notable non-goal is pixel-accurate information about what was actually displayed (which can be quite difficult to obtain efficiently in certain browser architectures in the face of filters, webgl, and other features). In all of these scenarios the information is useful even when delivered at a slight delay and without perfect compositing-result data.

Given the opportunity to reduce CPU use, increase battery life, and eliminate jank it seems like a new API to simplify answering these queries is a prudent addition to the web platform.

### Proposed API

```js
function callback(entries) {
  entries.forEach(function(entry) {
    if (isInterseting(entry.intersectionRect))
      doSomething();
  });
};
var root = document.getElementById('root');
var target = document.getElementById('target');
var options_dict = {
  thresholds: [0.0, 0.3, 0.7, 1.0],
  root: root
};
var observer = new IntersectionObserver(callback, options_dict);
observer.observe(target);
```

The expected use of this API is that you create an IntersectionObserver with a root element; then observe one or more elements that are descendants of the root.  The callback will be fired whenever any of the observed elements' ratio of (area of observed element's intersection with root / total area of observed element) crosses any of the observer's thresholds (i.e., transitions from less than the threshold to greater, or vice versa).  The callback includes change records for all observed elements for which a threshold crossing has occurred since the last callback.

## Element Visibility

The information provided by this API, combined with the default viewport query, allows a developer to easily understand when an element comes into, or passes out of, view. Here's how one might implement validation of input events for an embedded element, to ensure that the target element has been fully visible for a minimum duration when the event is received.  Note that all of the code runs in the embedded iframe's execution context; it's not necessary to run any code in the top-level embedding document.

```html
<!DOCTYPE html>
<button id="payButton" onclick="payButtonClicked()">Pay Money Now</button>
```

```js
// An element is considered visible if it's at least 80% unclipped.
var minimumUnclippedArea = 0.8;

// An element must be visible for at least 0.5 seconds prior to an input event
// for the input event to be considered valid.
var minimumVisibleDuration = 500;

function payButtonClicked(evt) {
  if (evt.currentTarget.visibleSince &&
      performance.now() - evt.currentTarget.visibleSince >= minimumVisibleDuation) {
    acceptClick();
  } else {
    rejectClick();
  }
}

function targetIsVisible(change) {
  return change.isIntersecting &&
         change.isVisible &&
         change.intersectionRatio >= minimumUnclippedArea;
}

function processChanges(changes) {
  let lastChange = changes.pop();  // We're only interested in the most recent change.
  if (targetIsVisible(lastChange)) {
    if (!lastChange.target.visibleSince) {
      lastChange.target.visibleSince = lastChange.time;
    }
  } else {
    delete lastChange.target.visibleSince;
  }
}

var observer = new IntersectionObserver(
  processChanges,
  { threshold: [minimumUnclippedArea] } 
);

var theButton = document.querySelector('#payButton);
observer.observe(theButton);
```

If more granular information about visibility is needed, the above code may be modified to use a sequence of threshold values.  This higher rate of delivery might seem expensive at first glance, but note the power and performance advantages over current practice:

  - No scroll handlers need be installed/run (a frequent source of jank).
  - Off-screen ads do not deliver any events or set any timers until they come into view.
  - No polling, synchronous layouts, or plugins are required; only a single timeout to record the completed ad impression.

## Data Scrollers

Many systems use data-bound lists which manage their in-view contents, recycling DOM to remain memory and layout-efficient while triggering loading of data that will be needed at some point in the near future.

These systems frequently want to use different queries on the same scroll-containing viewport. Data loading can take a long time, so it is advantageous to pre-populate data stores with significantly more information than is visible. The rendered element count may display a much smaller subset of available data; only the "skirt" on each side of a scrolling area necessary to keep up with scrolling velocity (to avoid "blank" or "checkerboard" data).

We can use an `IntersectionObserver` on child elements of a parent scrolling element to inform the system when to load data and recycle scrolled-out-of-view elements and stamp new content into them for rendering at the "end" of the list:

```html
<style>
  .container {
    overflow: auto;
    width: 10em;
    height: 30em;
    position: relative;
  }

  .inner-scroll-surface {
    position: absolute;
    left: 0px;
    top: 0px;
    width: 100%;
    /* proportional to the # of expected items in the list */
    height: 1000px;
  }

  .scroll-item {
    position: absolute;
    height: 2em;
    left: 0px;
    right: 0px;
  }
</style>

<div class="container">
  <div class="inner-scroll-surface">
    <div class="scroll-item" style="top: 0em;">item 1</div>
    <div class="scroll-item" style="top: 2em;">item 2</div>
    <div class="scroll-item" style="top: 4em;">item 3</div>
    <!-- ... -->
  </div>
</div>
```

As the user moves the `container`, the children can be observed and as they cross the threshold of the scrollable area, a manager can recycle them and fill them with new data instead of needing to re-create the items from scratch.

```js
function query(selector) {
  return Array.from(document.querySelectorAll(selector));
}

function init() {
  // Notify when a scroll-item gets within, or moves beyond, 500px from the visible scroll surface.
  var opts = { 
    root: document.querySelector(".container"),
    rootMargin: "500px 0px" 
  };
  var observer = new IntersectionObserver(manageItemPositionChanges, opts);
  // Set up observer on the items
  query(".inner-scroll-surface > .scroll-item")
    .forEach(function(scrollItem) {
      observer.observe(scrollItem);
    });
}

function manageItemPositionChanges(changes) {
  // ...
},
```

Many scrollers also want to fetch even more data than what's displayed in the list. We can create a second observer with a much larger "skirt" outside the viewport which will allow us to fetch a larger data set to account for latency.

## Delay Loading

Many sites like to avoid loading certain resources until they're near the viewport. This is easy to do with an IntersectionObserver:

```html
<!-- index.html -->
<div class="lazy-loaded">
  <template>
    ...
  </template>
</div>
```

```js
function query(selector) {
  return Array.from(document.querySelectorAll(selector));
}

var observer = new IntersectionObserver(
  // Pre-load items that are within 2 multiples of the visible viewport height.
  function(changes) {
    changes.forEach(function(change) {
      var container = change.target;
      var content = container.querySelector("template").content;
      container.appendChild(content);
      observer.unobserve(container);
    });
  },
  { rootMargin: "200% 0" }
);

// Set up lazy loading
query(".lazy-loaded").forEach(function(item) {
  observer.observe(item);
});
```

## Open Design Questions

This is a work in progress! We've tried to pattern the initial design after [`Object.observe()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/observe) and [DOM's Mutation Observers](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver).

The specific timing of of change record delivery is also TBD.

Is it meaningful to have overdraw queries against the default viewport?
