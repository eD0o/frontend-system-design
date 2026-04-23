# 2 - DOM API

## 2.1 - DOM & Querying

The DOM API is the `interface with tools to manipulate the page`. It is exposed through a `hierarchy of browser objects`, not just HTML elements.

### Global entry points

The browser exposes two main objects:

- window → global execution context
- document → representation of the current page

![](assets/images/2026-04-22-08-07-11.png)

Window is not part of the DOM tree.

It:

- Represents the browser environment (not just viewport)
- Holds global APIs (timers, events, etc.)
- Contains document

Important:

> window does not provide DOM manipulation directly. It only exposes document, which is the entry point to the DOM.

### Where the DOM API actually lives

The DOM API is not attached to a single place.

Instead, `it is distributed across the prototype chain`:

- Element.prototype → element-specific APIs (querySelector, attributes, etc.)
- Document.prototype → document-specific APIs (createElement, querySelector)
- Node.prototype → tree traversal (parentNode, childNodes)
- EventTarget.prototype → events (addEventListener)

The reasoin is that API must work across different node types:

-HTML elements
-SVG elements
-Document
-Text nodes (partially)

Cause → effect:

APIs are split by responsibility (tree, elements, document, events)
Prototype chain composes the full behavior

![](assets/images/2026-04-22-08-07-35.png)

### Core hierarchy

The DOM is built on a class hierarchy that represents a tree structure.

Simplified structure:

- Node
  - Element
    - HTMLElement
  - TextNode

Behavior:

- Node → provides tree structure (parent, children, traversal)
- Element → adds DOM manipulation capabilities
- HTMLElement → represents actual HTML elements
- TextNode → represents raw text inside elements

> Text is not part of an element directly. It is wrapped inside a TextNode, do not support querying and only support basic Node APIs.

Example:

```html
<div>Hello</div>
```

Internally:

- div → HTMLElement
- "Hello" → TextNode (child of div)

### HTMLDocument exception

HTMLDocument behaves differently: it `it extends Node but also exposes DOM APIs, acting as both a node and an entry point.`

This makes it a special case in the hierarchy.

Practical implication:

- document can act like both:
  - a node in the tree
  - an entry point for DOM operations

### Querying

| Method                 | Query Mechanism              | Query Cost | Read Cost | Return Type    | Live? | Memory Cost | Notes                                                          |
| ---------------------- | ---------------------------- | ---------- | --------- | -------------- | ----- | ----------- | -------------------------------------------------------------- |
| getElementById         | Hashmap lookup               | O(1)       | O(1)      | Single Element | No    | Low         | Fastest; relies on global ID uniqueness                        |
| getElementsByClassName | DFS traversal (+ cache hint) | O(n)\*     | O(n)      | HTMLCollection | Yes   | Low         | Live collection; re-evaluates on access                        |
| getElementsByTagName   | DFS traversal (+ cache hint) | O(n)\*     | O(n)      | HTMLCollection | Yes   | Low         | Same behavior as className                                     |
| querySelector          | CSS selector (compiled)      | O(n)\*\*   | O(1)      | Single Element | No    | Low         | ID selectors may optimize to O(1)                              |
| querySelectorAll       | CSS selector (compiled)      | O(n)\*\*   | O(1)      | NodeList       | No    | Higher      | Snapshot; allocates new structure (proxy-like copies of nodes) |

Observations:

O(n)\* - First query may traverse the tree; repeated queries may be faster (engine-dependent, not guaranteed)

O(n)\*\* - Depends on selector complexity: simple selectors → faster / complex selectors → slower (compile + match).

Quick mental comparison:

- Fastest lookup → getElementById
- Lowest memory → getElementsByClassName / TagName
- Most flexible → querySelector
- Best for stable iteration → querySelectorAll
