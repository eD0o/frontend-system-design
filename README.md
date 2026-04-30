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
  }
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
check()
```

Think:

“notify me when visibility changes”