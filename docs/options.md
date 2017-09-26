# Options
Sentinel DOM library has a list of declarative options which grant precise control over the tracking logic.

* [Root options](#root-options)
  * [Targets](#targets)
  * [Custom boundaries](#bounds)
  * [Throttling](#throttle)
  * [Unique impressions](#once)
  * [Debug mode](#debug)
  * [Snapshots](#snapshots)

* [Snapshot options](#snapshot-options)
  * [Named snapshots](#named-snapshots)
  * [Unique snapshots](#unique-snapshots)
  * [Offsets](#offsets)
  * [Bleeding edges](#bleeding-edges)
  * [Thresholds (delta areas)](#thresholds)
  * [Callback function](#callback)

## Root options
### Targets
* `targets: Array<HTMLElement> | HTMLElement`

One or mutliple DOM elements to be tracked.

```js
new Tracker({
  targets: document.getElementsByClassName('box'),
  // ...
});
```

### Bounds
* `bounds?: HTMLElement` (default: `window`)

Boundaries against which the tracking is performed. You may want to provide custom boundaries to make the tracking more performant.

```js
new Tracker({
  // ...
  bounds: document.getElementById('custom-bounds'),
  // ...
})
```

### Throttle
* `throttle?: number` (default: `250`)

Throttle amount (**ms**) between each tracking attempt.

```js
new Tracker({
  // ...
  throttle: 1000, // attempt tracking no more than once per second
  // ...
});
```

### Once
* `once?: boolean` (default: `true`)

Should each snapshot invoke its callback only once, after the first successful tracking. Setting this to `false` will trigger snapshot's callback function each time the target becomes visible within the bounds.

```js
new Tracker({
  // ...
  once: false, // allow callback functions to be called mutliple times
  // ...
});
```

Setting `once` on a root level will apply to each snapshot, unless it has its own `once` specified.

### Debug
* `debug?: boolean` (default: `false`)

Enable/disable debug mode.

```js
new Tracker({
  debug: true,
  // ...
});
```

Debug mode is meant for monitoring the steps of tracking attempts in the console. It is useful during the investigation of the tracking behavior.

> **Note:** Debug mode may decrease the tracking performance due to adding extra operations (logging) after each tracking step. Consider **not to turn it on** unless needed.

### Snapshots
* `snapshots: Array<Snapshot>`

A list of the snapshots to take per each tracking attempt.

```js
new Tracker({
  // ...
  snapshots: [
    {
      edgeY: 10, // once scroll position is at 10% of target's height or more
      callback() { ... }
    },
    {
      thresholdY: 50, // once exactly 50% of the target's height becomes visible
      callback() { ... }
    }
  ]
});
```

The benefit of a snapshot system is that you are able to perform multiple tracking operations against the same target/bounds within a single declaration.

## Snapshot options
Most of the tracking logic is defined through the snapshot options.

Provided snapshot options have higher priority and overwrite the root options (i.e. `once`). Consider this for a granular control over each individual snapshot. **Remember** that you can combine snapshot options to achieve the logic you need.

### Named snapshots
* `name?: string`

The name of a snapshot. Useful primarily for debugging purposes. When performing multiple snapshots they will appear named in debug mode if they have `name` option set.
```js
new Tracker({
  // ...
  snapshots: [
    {
      name: 'Box has appeared',
      callback() { ... }
    }
  ]
})
```

### Unique snapshots
* `once?: boolean`

Similar to the [root option](#once), `once` allows/forbids to perform the callback function multiple times, after the target has appear the first time. Setting this option on a certain snapshot will overwrite the `once` option provided in the root options.
```js
new Tracker({
  // ...
  once: false, // allow for callbacks to fire multiple times
  snapshots: [
    {
      callback() { ... }
    },
    {
      once: true, // this snapshot's callback will fire only once
      callback() { ... }
    }
  ]
})
```

### Offsets
* `offsetX?: number` (default: `0`)
* `offsetY?: number` (default: `0`)

*Offset* - is an absolute amount of pixels (**px**) to add to the current bleeding edge/threshold.

For example, a callback function should be called once there is still 10 pixels left to the actual top edge of the target:
```js
new Tracker({
  // ...
  snapshots: [
    {
      edgeY: 0, // expect the very top coordinate of the target to appear
      offsetY: -10, // negative value because the top position should be less than actual
      callback() { ... }
    }
  ]
});
```

<div align="center">
  <img src="./media/offset-y.png" style="margin-right:1rem">
  <p>Setting vertical offset to 10px.</p>
</div>

One the blue line appear in the viewport, a snapshot will be considered successful. Similarly, offsets affect [Bleeding edges](#bleeding-edges) or [Thresholds](#thresholds) if the latter are specified. The same logic applies to the horizontal offset (`offsetX`).

> **Note:** If your tracking logic relies on the percentage of the target's width/height see [Threshold options](#thresholds) respectively. Do **not** use offset option for this purpose.

### Bleeding edges
* `edgeX?: number`
* `edgeY?: number`

*Bleeding edge* - is an imaginary line drawn at a relative coordinate on one of the target's axis.

It is possible to set expected horizontal (`edgeX`) and vertical (`edgeY`) bleeding edges. Bleeding edges are set in percentages (**%**) relatively to the target's width or height respectively.

```js
new Tracker({
  // ...
  snapshots: [
    {
      edgeX: 25,
      callback() { ... }
    }
  ]
});
```

<div align="center">
  <img src="./media/edge-x.png" style="margin-right:1rem">
  <p>Setting horizontal bleeding edge to 25%.</p>
</div>

Setting `edgeX: 25` will draw a line at 25 percent of the target's width. The blue line (bleeding edge) on a picture should appear in the viewport <b>and</b> bounds (in case of custom bounds) in order to fire a callback function.

> Note: Bleeding edges are unaware of scroll direction. That means that the **exact** coordinate should be met. Using the current example (`edgeX: 25`), if you were scrolling from right to left, you would need to scroll to **75%** of the target's width to meet the bleeding edge.

```js
new Tracker({
  // ...
  snapshots: [
    {
      edgeY: 25,
      callback() { ... }
    }
  ]
});
```

<div align="center">
  <img src="./media/edge-y.png" style="margin-right:1rem">
  <p>Setting vertical bleeding edge to 25%.</p>
</div>

<p>In this example we set <code>edgeY: 25</code>. The blue line represents a vertical bleeding edge which should be in viewport <b>and</b> bounds (in case of custom bounds) in order to trigger a callback.</p>

Generally, using `edgeX` and `edgeY` is recommended when your visibility logic relies on *uniderictional* scroll behavior. For example, when you need to determine if user is reading something. It is obvious that reading happens from top to bottom, so setting `edgeY` at low percentage would ensure user has started reading, while at high percentage - that he has read an article competely.

> **Note:** You should **not** combine any of the edge options with threshold options. They are mutually exclusive.

### Thresholds
* `thresholdX?: number` (default: `100`)
* `thresholdY?: number` (default: `100`)

*Threshold* - is a percentage of the target's width/height which should appear in the viewport and bounds in order for a snapshot to be successful.


Lets say you would like to execute a certain callback only when at least **75%** of the element's height is in the viewport. You can achieve this by setting `thresholdY: 75` as a snapshot option.

```js
const Tracker({
  // ...
  snapshots: [
    {
      thresholdY: 75,
      callback() { ... }
    }
  ]
});
```

This would render a *delta area* demonstrated as striped rectangles below:

<div align="center">
  <img src="./media/threshold-y.png" alt="Threshold Y">
  <p>Setting vertical threshold to 75%.</p>
</div>

As you can see, delta areas are *omnidirectional*, meaning that they are expect to appear regardless of a scroll direction. This is the main difference between the thresholds and [bleeding edges](#bleeding-edges). The same rules apply to the horizontal threshold.

One of the powerful features of thresholds is the ability to combine them:
```js
new Tracker({
  // ...
  snapshots: [
    {
      thresholdX: 75,
      thresholdY: 75,
      callback() { ... }
    }
  ]
});
```

<div align="center">
  <img src="./media/threshold-xy.png" alt="Setting both thresholds">
  <p>Setting vertical and horizontal thresholds to 75%.</p>
</div>

Once any of these delta areas fully appear in the viewport/bounds, the snapshot would be considered successsful, and a callback function would be called.

> **Note:** Treshold options **do not** accept negative values.

### Callback
* `callback: Function(args: TCallbackArgs): any`

Callback function which is called immediately once snapshot is considered successful (the target is visible within the bounds).

Callback function has the following arguments:
```js
type TCallbackArgs = {
  DOMElement: HTMLElement // a reference to the visible element in the DOM
}
```

Let's say we would like to add a certian class name to the element once it becomes visible:
```js
new Tracker({
  target: document.getElementsByClassName('box'),
  once: true,
  snapshots: [
    {
      callback({ DOMElement }) {
        /* Append class name "tracked" once the snapshot it resolved */
        DOMElement.classList.add('tracked');
      }
    }
  ]
});
```