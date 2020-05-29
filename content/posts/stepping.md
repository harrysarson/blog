---
title: "Steppers"
date: 2020-05-29
---

# Steppers

If you have ever wondered how the elm runtime works you may have traced the
function calls required to create an application (for example
`elm/core:Platform.worker` or `elm/browser:Browser.element`). If you do so you
will end up at `elm/core:Elm.Kernel.Platform.intitialize`.

 The `initialize` function in the `elm/core` package is defined like
this:

```js
function _Platform_initialize(flagDecoder, args, init, update, subscriptions, stepperBuilder) {
   // ...
}
```

`flagDecoder` sounds a bit scary but we ignore it for now. `args` is the
javascript object you pass to init: `Elm.Main.init({ flags: {}, node: {}})`.
Whilst tracing the functions call you can tell that `init`, `update` and
`subscriptions` are the elm functions that you, as a seasoned elm programmer,
have come to know and love. But what is the `stepperBuilder` parameter?

We have the following clues to help us work out the mystery:

1. When creating a `elm/core:Platform.Worker` we pass the `initialize` function
   [this][stepper-worker] as its `stepperBuilder`:

   ```js
   function() { return function() {} }
   ```

2. `elm/core:Elm.Kernel.Platform.initialize` uses the `stepperBuilder`
   parameter like [this][init-builder]:

   ```js
   var initPair = init(result.a);
	 var model = initPair.a;
   var stepper = stepperBuilder(sendToApp, model);
   ```

   and then uses the created `stepper` like [this][init-stepper]:

   ```js
   var pair = A2(update, msg, model);
   stepper(model = pair.a, viewMetadata);
   ```

3. When creating a HTML element, `elm/browser:Elm.Kernel.Browser.element`
   passes [this monster][stepper-element] as its `stepperBuilder`:

   ```js
   function(sendToApp, initialModel) {
      var view = impl.__$view;
      /**__PROD/
      var domNode = args['node'];
      //*/
      /**__DEBUG/
      var domNode = args && args['node'] ? args['node'] : __Debug_crash(0);
      //*/
      var currNode = _VirtualDom_virtualize(domNode);

      return _Browser_makeAnimator(initialModel, function(model)
      {
         var nextNode = view(model);
         var patches = __VirtualDom_diff(currNode, nextNode);
         domNode = __VirtualDom_applyPatches(domNode, currNode, patches, sendToApp);
         currNode = nextNode;
      });
   }
   ```

It is crystal clear now? I doubt it, it certainly is not crystal clear for me.
Given my three clues I think all we can work out is:

- complex code without static typing quickly becomes an unintelligible mess
  that is very hard to read or maintain.
- the runtime calls `stepperBuilder` with the model returned by init.
- `stepperBuilder` returns a function (here called `stepper`) that the runtime
  calls everytime the model is updated.
- `stepperBuilder` and its offspring `stepper` do nothing when the program is
  headless, but come into their own when the runtime starts needing to
  manipulate the DOM.

## A small step for a man

Let us try to make some sense of this with the help of static typing. A
`stepperBuilder` "builds" a stepper. If were trying to rewrite the runtime in
elm we might have&nbsp;[[1]](####1):

```elm
type alias StepperBuilder =
      SendToApp msg -> model -> Stepper model
```

We can say that `StepperBuilder` takes a `SendToApp` and the initial model and
creates `Stepper`. It is very important to note that this function **is not
pure**. When you call `stepperBuilder` stuff may happen (concretely the runtime
renders your initial view and draws it to the DOM).

"What is `Stepper` and what is `SendToApp msg`?" I hear you ask. Well:

```elm
type alias Stepper model =
      model -> ViewMetadata -> ()

type alias SendToApp msg =
      msg -> ViewMetadata -> ()
```

So `Stepper` takes the model's new value and some `ViewMetadata` and returns
nothing. We can tell that `Stepper` must be an impure function (with side
effects) otherwise calling `stepper` would be pointless. Every time the model
updates, the elm runtime will call this function and it do its virtual DOM
diffing magic and updates the DOM accordingly.

Yet there is  another type: `ViewMetadata`! Before you ask "What is
`ViewMetadata`?" I must confess that I do not really know. I believe it
controls whether the runtime instantly renders changes to the DOM or queues the
update using `requestAnimationFrame`. I believe `ViewMetadata` is the solution
to the issues in elm `0.18` where port subscriptions would sometimes fire
before the DOM had been updated and other times they would not fire instantly
in response to DOM events. For now, we can make `ViewMetadata` a placeholder
type and fill it in future when I have a better idea about what is going on.

```elm
type ViewMetadata
   = ViewMetadata ViewMetadata
```

Finally we come to `SendToApp`. The DOM may produce events which need to be
passed to the application's `update` function. The runtime gives the
`StepperBuilder` a `SendToApp` function to do just that. Event listeners
attached to the DOM call `SendToApp` with a message (and a value for the
mysterious `ViewMetadata`). The runtime will then take care of passing that
message to the app via `update`.

### A giant leap for mankind

So what can we say. Well `stepperBuilder` could be named
`createOnUpdateHandler` and creates a function `onUpdateHandler` that will be
called by the runtime every time the model updates. `onUpdateHandler` takes
care of all the changes to the DOM needed to display your webapp. Tidy.

### Notes

#### 1

Elm curries functions. The js definition of `stepperBuilder` is a function that
takes two arguments so this curried version isn't truely the type ignature of
`stepperBuilder`. We cannot write the true type signature in elm.

[stepper-worker]: https://github.com/elm/core/blob/22eefd207e7a63daab215ae497f683ff2319c2ca/src/Elm/Kernel/Platform.js#L26
[init-builder]: https://github.com/elm/core/blob/22eefd207e7a63daab215ae497f683ff2319c2ca/src/Elm/Kernel/Platform.js#L42
[init-stepper]: https://github.com/elm/core/blob/22eefd207e7a63daab215ae497f683ff2319c2ca/src/Elm/Kernel/Platform.js#L48
[stepper-element]:https://github.com/elm/browser/blob/1d28cd625b3ce07be6dfad51660bea6de2c905f2/src/Elm/Kernel/Browser.js#L37-L54
