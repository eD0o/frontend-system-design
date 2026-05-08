# 3 - Observer APIs

Changes without polling by letting the browser handle detection.

Core idea:

- subscribe instead of polling
- runs only on change
- async + batched

Cause → effect:

- polling → constant work
- observer → work only when needed

Handled at browser level (not JS loop)

Types

| API                  | Observes   | Main use              | Examples                  |
| -------------------- | ---------- | --------------------- | ------------------------- |
| IntersectionObserver | Visibility | Lazy load, scroll UI  | Virtualization, analytics |
| MutationObserver     | DOM        | Dynamic UI            | Editors, drawing tools    |
| ResizeObserver       | Size       | Responsive components | Charts, adaptive layout   |

## 3.1 - IntersectionObserver

`Detects when an element enters or leaves the visible area of a container`.

Concepts:

- target → observed element
- root → container used as reference (default: viewport)
- threshold → minimum visibility ratio required to trigger

One observer can track multiple elements.

Callback:

- async
- fires only on state change
- receives entries[] and observer

Fires when:

- target starts intersecting
- target stops intersecting

Each entry contains:

- isIntersecting → whether the target is currently visible/intersecting

> entries[] exists because multiple observed elements can update together.

Usage:

```ts
const observer = new IntersectionObserver(
  (entries, observer) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        // logic here
      }
    });
  },
  {
    root: document.getElementById("container"),
    threshold: 0.1,
  },
);

observer.observe(document.getElementById("target"));
```

Manual approach before IntersectionObserver:

```ts
setInterval(() => {
  const rectA = A.getBoundingClientRect();
  const rectB = B.getBoundingClientRect();

  const isIntersecting =
    rectA.bottom > rectB.top &&
    rectA.right > rectB.left &&
    rectA.top < rectB.bottom &&
    rectA.left < rectB.right;

  if (isIntersecting) {
    // logic here
  }
}, 50);
```

Manual polling runs repeatedly.
IntersectionObserver runs only when intersection state changes.

Performance:

- no polling
- no layout loops
- batched updates

Result:

- lower CPU usage
- scales well

Mental model:

Instead of repeatedly checking:

```ts
check();
```

Think:

“notify me when visibility changes”

## 3.2 - MutationObserver

`Reacts to DOM changes by subscribing to mutations` instead of manual polling (setTimeout / setInterval).

The browser `batches DOM changes and delivers them asynchronously`.

### Core behavior

- runs natively inside the browser
- asynchronous and batched
- triggered only when configured conditions are met

Cause → effect:

- manual checks → constant work → CPU overhead
- observer → runs on change → minimal work

### What to observe

| Option        | What it detects        | Scope                | When to use                   |
| ------------- | ---------------------- | -------------------- | ----------------------------- |
| childList     | added or removed nodes | direct children only | track immediate structure     |
| subtree       | descendant DOM changes | entire subtree       | track deep DOM changes        |
| attributes    | attribute changes      | target element       | class, style, data-\* updates |
| characterData | text content changes   | text nodes           | inputs, editable content      |

Key distinction:

- childList → direct children only
- subtree → includes all descendants

### Configuration

The observer only reacts to what you enable.

- more flags → more callback executions
- fewer, precise flags → better performance

Guideline:

- enable only what you need
- avoid setting everything to true

### Mutation record

Each callback receives a list of changes, not the full state.

Common fields:

- type → what changed
- target → where it happened
- addedNodes / removedNodes → structural changes
- oldValue → previous value (if enabled)

Think of it as a diff, not a snapshot.

### Example

```ts
const observer = new MutationObserver((mutations) => {
  for (const m of mutations) {
    if (m.type === "childList") {
      // handle node changes
    }
  }
});

observer.observe(targetNode, {
  childList: true,
  subtree: true,
});
```

### Performance

- native → faster than JS-based tracking
- scales better than manual observation
- large observed subtrees can still be expensive

Constraint:

- broad configs → too many callbacks
- heavy callbacks → main bottleneck

### Flow

1. create observer with callback
2. configure what to track
3. attach to target node
4. handle mutations selectively

> Define what matters, don't try observing everything

### Mutation Observer in React

`React already reacts to state changes and controls DOM updates` internally, so observing DOM mutations is usually redundant.

| React                      | MutationObserver             |
| -------------------------- | ---------------------------- |
| observes application state | observes DOM mutations       |
| declarative                | imperative                   |
| knows what should change   | detects what already changed |

## 3.3 - ResizeObserver

Different resize problems require different tools.

The key distinction is:

- viewport-based adaptation
- element-based adaptation
- style-only reactions
- JavaScript-driven reactions

Modern browser APIs try to avoid manual resize tracking because `resize calculations can become extremely expensive during continuous layout updates`.

### Comparing Resize Approaches

| Tool                | Tracks            | JS Callback | Performance | Typical Use                   |
| ------------------- | ----------------- | ----------- | ----------- | ----------------------------- |
| CSS Media Query     | viewport/window   | no          | excellent   | responsive layouts            |
| CSS Container Query | element container | no          | excellent   | component-based responsive UI |
| ResizeObserver      | specific elements | yes         | very good   | reactive element measurements |
| resize event        | window            | yes         | poor        | legacy/manual resize logic    |

### CSS Media Queries

Best option for adaptive layouts when JavaScript execution is unnecessary.

The browser evaluates breakpoints internally during layout calculation, so `no DOM event traversal or JS scheduling is involved`.

Cause → effect:

- CSS-only adaptation
- browser handles layout natively
- minimal runtime overhead

Limitations:

- cannot execute JS callbacks
- cannot observe individual element sizes
- only reacts to viewport conditions

Example:

```css
@media (max-width: 768px) {
  .sidebar {
    display: none;
  }
}
```

### CSS Container Queries

Container queries solve a major limitation of media queries.

Instead of reacting to the viewport, `components react to the size of their parent container`.

Cause → effect:

- component size changes
- browser recalculates query conditions
- children adapt automatically

Still CSS-only:

- no JS callback support
- no measurement logic
- no side effects

Example:

```css
.card-container {
  container-type: inline-size;
}

@container (max-width: 400px) {
  .card {
    flex-direction: column;
  }
}
```

Useful for:

- reusable UI systems
- cards/grids
- dashboards
- nested layouts

### resize Event

The resize event is `one of the slowest resize mechanisms`.

It relies on the standard DOM event system, which means the `browser must propagate the event through the DOM tree`.

Internally:

- event travels down the tree
- reaches target
- bubbles back upward

This propagation happens repeatedly during continuous resizing.

Another major issue:

- resize fires extremely often
- thousands of callbacks may execute during a small drag resize

Cause → effect:

- continuous window resize
- excessive event firing
- repeated layout work
- performance degradation

Example:

```ts
window.addEventListener("resize", () => {
  console.log(window.innerWidth);
});
```

In practice, resize handlers are commonly debounced:

```ts
let timeout: number;

window.addEventListener("resize", () => {
  clearTimeout(timeout);

  timeout = window.setTimeout(() => {
    console.log("resize finished");
  }, 200);
});
```

Limitations:

- tracks only viewport/window
- cannot observe arbitrary elements
- high callback frequency
- expensive under heavy layouts

Prefer avoiding it unless:

- supporting legacy environments
- reacting specifically to viewport changes
- ResizeObserver is unavailable

### ResizeObserver

ResizeObserver was designed specifically for element resize tracking.

Unlike resize events, it `avoids expensive DOM event propagation and works closer to the rendering/layout` engine.

Mental model:

- browser detects element size changes during layout
- observer batches notifications
- callback runs after changes are collected

Cause → effect:

- element dimensions change
- browser schedules observer entries
- callback receives size snapshots

Advantages:

- tracks individual elements
- supports multiple observed elements
- much lower overhead than resize events
- callback-based API

### ResizeObserver API

The API is intentionally minimal.

Constructor:

```ts
const observer = new ResizeObserver((entries) => {
  for (const entry of entries) {
    console.log(entry);
  }
});
```

Observation:

```ts
observer.observe(element);
```

Multiple elements can share the same observer:

```ts
observer.observe(card1);
observer.observe(card2);
observer.observe(card3);
```

### Box Types

ResizeObserver can measure different box models.

Options:

| Box         | Includes                   |
| ----------- | -------------------------- |
| content-box | content only               |
| border-box  | content + padding + border |

Example:

```ts
observer.observe(element, {
  box: "border-box",
});
```

Behavior difference:

- content-box ignores padding/border changes
- border-box reacts to total rendered size changes

This matters when layout depends on visual dimensions rather than content area.

### ResizeObserverEntry

Each ResizeObserver callback receives entry objects describing resized elements.

Important properties:

| Property       | Purpose                  |
| -------------- | ------------------------ |
| target         | resized element          |
| contentBoxSize | content dimensions       |
| borderBoxSize  | total visible dimensions |

Example:

```ts
const observer = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const size = entry.borderBoxSize[0];

    console.log(size.inlineSize); // width
    console.log(size.blockSize); // height
  }
});
```

> The [0] exists because these values are arrays internally, even though today they usually contain only one item.