---
layout: post
title: Remember last value in React functional components
categories: [programming]
tags: [react, typescript, useRef, remember]
comments: true
---

Sometimes it can be very useful to save previous values of props (or state) in a React component.
Perhaps this is easier to be imagined in a class component, but in a functional component, less so.

However, we can use the `useRef()` React hook to achieve that quite efficiently.

Less talk, more code (typescript)...

```ts
const someFunctionalComponent = (p: {index: number}) => {
  // Keep the last positive value around.
  const lastPositiveIndexRef = React.useRef<number>(Math.max(p.index, 0));
  if (p.index >= 0) {
    lastPositiveIndexRef.current = p.index;
  }
  
  // Here, the ref will always have the last positive input.
  const positiveIndex = p.index >= 0 ? p.index : lastPositiveIndexRef.current;
  return <span>I'm only allowing positive numbers to be shown: {positiveIndex}</span>
}
```

Here's a [working sample](https://jsfiddle.net/valeriupalos/znoqju5d/12/) (JSFiddle).

Enjoy!
