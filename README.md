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