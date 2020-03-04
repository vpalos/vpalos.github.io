Sometimes it can be very useful to save previous values of props (or state) in a React component.
Perhaps this is easier to be imagined in a class component, but in a functional component, less so.

However, we can use the `useRef()` React hook to achieve that quite efficiently.

Less talk, more code...

```ts
const someFunctionalComponent = (p: {index: number}) => {
  /**
   * We're always keeping the last positive value around, so as to allow
   * the selector to re-appear from the same position it disappeared from.
   */
  const lastPositiveIndexRef = React.useRef<number>(Math.max(p.index, 0));
  if (p.index >= 0) {
    lastPositiveIndexRef.current = p.index;
  }
  
  /**
   * Here, ref will always have the last positive input, if we need it.
   */
  const positiveIndex = p.index >= 0 ? p.index : lastPositiveIndexRef.current;
  
  return <span>I'm only allowing positive numbers to be shown: {positiveIndex}</span>
}
```

Here's a [working sample](https://jsfiddle.net/valeriupalos/znoqju5d/11/) (JSFiddle).

Enjoy!
