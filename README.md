# 4 - Virtualization

A rendering optimization technique that `keeps data in memory while only rendering` the visible portion of the UI.

The main goal is to `reduce DOM size, mutations, CPU work, and memory` usage. Large DOM trees are expensive for the browser to maintain, layout, paint, and update.

## 4.1 - Technique

Instead of rendering the entire dataset::

- keep the full dataset in memory
- render only a small visible window
- recycle DOM nodes as the user scrolls

Mental model:

- the data grows
- the DOM stays almost the same size

Cause → effect:

- fewer DOM nodes → less layout/repaint work
- fewer mutations → lower CPU usage
- stable DOM size → more predictable performance

| Aspect                | Normal Rendering             | Virtualization          |
| --------------------- | ---------------------------- | ----------------------- |
| DOM size              | grows continuously           | stays relatively stable |
| CPU work              | more layout/repaint work     | reduced rendering work  |
| Scrolling performance | degrades with large lists    | more consistent         |
| Element lifecycle     | constantly creates new nodes | reuses existing nodes   |

This is especially important for:

- infinite feeds
- chat applications
- tables
- large lists
- log viewers

### 4.1.1 - High-Level Structure

A virtualized list usually contains:

- top observer
- viewport
- virtual list container
- bottom observer

Simplified structure:

```html
<div class="container">
  <div id="top-observer"></div>

  <div class="virtual-list">
    <!-- rendered items -->
  </div>

  <div id="bottom-observer"></div>
</div>
```

| Element         | Responsibility             |
| --------------- | -------------------------- |
| top observer    | detects upward scrolling   |
| viewport        | defines the visible area   |
| virtual list    | renders visible items      |
| bottom observer | detects downward scrolling |

The observers are typically implemented with IntersectionObserver.

### 4.1.2 - Flow

Initial render:

```txt
[item 1]
[item 2]
```

Example with page size = 2:

When the viewport reaches the bottom observer:

- load next data chunk
- do not create unlimited DOM elements
- recycle elements that are no longer visible

Instead of creating new nodes:

```txt
[item 1] -> recycled
[item 2] -> recycled
```

Those elements are moved and updated to represent:

```txt
[item 5]
[item 6]
```

Important detail:

- the user cannot see recycled elements during the move
- recycling happens outside the visible viewport

This creates the illusion of an infinitely growing list while keeping the DOM size stable.

### 4.1.3 - Positioning Strategy

Virtualized lists commonly reposition elements using transforms.

Example:

```ts
const translateY = (y: number) => `transform: translateY(${y}px)`;
```

Transforms are preferred because they:

- avoid expensive layout recalculations
- are usually GPU-accelerated
- allow smooth movement

The implementation also stores positional metadata using attributes like:

```html
<div data-y="120"></div>
```

This helps track where recycled elements should be placed.

### 4.1.4 - Observer-Based Rendering

IntersectionObserver is used to detect when observers enter the viewport.

Example:

```ts
const observer = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      if (!entry.isIntersecting) continue;

      if (entry.target.id === "top-observer") {
        handleTopObserver();
      }

      if (entry.target.id === "bottom-observer") {
        handleBottomObserver();
      }
    }
  },
  {
    threshold: 0.2,
  },
);
```

Behavior:

- observer intersects viewport
- callback fires
- list updates
- elements get recycled/repositioned

Threshold controls how much of the target must be visible before triggering.

Example:

```txt
threshold: 0.2
```

Means:

```txt
20% visible → callback executes
```

### 4.1.5 - Rendering Model

The implementation separates:

- template generation
- DOM rendering
- side effects

Simplified flow:

```txt
HTML template → render → register observers/effects
```

Example render pattern:

```ts
render() {
  root.innerHTML = this.toHTML();
  this.effect();
}
```

Mental model:

- toHTML() describes structure
- effect() attaches browser behavior

This resembles React's render + effect lifecycle.

### 4.2 - Virtualization in React

In React, virtualization is usually `implemented declaratively through libraries instead of manual DOM` recycling.

Common libraries:

- react-window
- react-virtualized
- TanStack Virtual

Instead of manually moving DOM nodes, React virtualization typically works by:

- calculating the visible range
- rendering only visible items
- updating the rendered subset during scroll

Example:

```ts
const startIndex = Math.floor(scrollTop / itemHeight);
const endIndex = startIndex + visibleCount;
```

Then only the visible slice is rendered:

```ts
items.slice(startIndex, endIndex);
```

| Vanilla JS                 | React Virtualization     |
| -------------------------- | ------------------------ |
| manual DOM recycling       | declarative rendering    |
| direct DOM mutations       | state-driven updates     |
| explicit node movement     | abstracted by libraries  |
| IntersectionObserver-heavy | scroll calculation-heavy |
| low-level control          | easier integration       |

Unlike low-level vanilla JS implementations:

- React usually abstracts DOM recycling
- libraries manage positioning internally
- rendering becomes state-driven instead of mutation-driven

Many React virtualization libraries also use:

- absolute positioning
- translateY
- spacer elements
- overscanning

Overscanning renders a small buffer outside the viewport to avoid visible pop-in during fast scrolling.

Example:

```txt
visible items: 10
overscan: 3
actual rendered: 16
```

Modern React virtualization often `relies more on scroll position calculations than IntersectionObserver because it provides more deterministic rendering behavior` for large lists.

IntersectionObserver is still commonly used for:

- infinite loading
- lazy loading
- sentinel detection
