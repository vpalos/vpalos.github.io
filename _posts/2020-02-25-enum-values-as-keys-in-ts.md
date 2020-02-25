---
layout: post
title: Using `enum` values as strictly typed object keys
categories: [programming]
tags: [typescript, enum]
---

In TypeScript, it's often useful to define interfaces or complex (structured)
types whose properties (or keys) may only be the values of a previously defined
`enum` type.

Here's a good example: an object declaring a set of buttons for a modal dialog.

## Instead of this...

```ts
type DialogButtons = {
  yes?: boolean,
  no?: boolean,
  cancel?: boolean
}

interface IDialog {
  buttons: DialogButtons
}

const dialog: IDialog = {
  buttons: {
    yes: true,
    no: false
  }
}

console.log("yes" in dialog.buttons) // true
console.log(dialog.buttons.yes) // true

console.log("no" in dialog.buttons) // true
console.log(dialog.buttons.no) // false

console.log("cancel" in dialog.buttons) // false
console.log(dialog.buttons.cancel) // undefined
```
This works, but once we try to use these values in other contexts, the approach becomes difficult to use.

> What if we want to pass which button was pressed to a callback function? It would be cumbersome.

Therefore...

## ...we can use `enum` values.

```ts
enum DialogButton {
  YES = "yes",
  NO = "no",
  CANCEL = "cancel"
}

interface IDialog {
  buttons: { [B in DialogButton]?: boolean },
  callback: (button: DialogButton) => void
}

const dialog: IDialog = {
  buttons: {
    [DialogButton.YES]: true,
    [DialogButton.NO]: false
  },
  callback(button) {
    console.log(button)
  }
}

console.log(DialogButton.YES in dialog.buttons) // true
console.log(DialogButton.NO in dialog.buttons) // true
console.log(DialogButton.CANCEL in dialog.buttons) // false

console.log(dialog.buttons[DialogButton.YES]) // true
console.log(dialog.buttons[DialogButton.NO]) // false
console.log(dialog.buttons[DialogButton.CANCEL]) // undefined

dialog.callback(DialogButton.YES) // yes
dialog.callback(DialogButton.NO) // no
dialog.callback(DialogButton.CANCEL) // cancel
```
This uses the a TypeScript feature called [Mapped Types](https://www.typescriptlang.org/docs/handbook/advanced-types.html#mapped-types) 
and allows us to play around with the actual button values with a lot more flexibility, for example we could
define an array of buttons.

```ts
function renderButtons(buttons: [DialogButton]): [HTMLElement] {
  // ...
}
```

Code gist: [https://gist.github.com/vpalos/0aab903ef607d6da31229b957d91d888](https://gist.github.com/vpalos/0aab903ef607d6da31229b957d91d888).

Enjoy!
