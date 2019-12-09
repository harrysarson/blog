---
title: "Why Platform Must Define Task"
date: 2019-11-21T20:00:00Z
draft: false
---

<style>

span:before {
  display: block;
  position: absolute;
  content: " ";
  margin-top: -285px;
  height: 285px;
  visibility: hidden;
  pointer-events: none;
}

</style>

# Why `Platform` must define `Task`

> The examples in this post uses `elm 0.19.1` and `elm/core 1.0.2`.

I have been investing the runtime of the [elm programming language](https://elm-lang.org), mainly to see if it is possible to write more of the logic of the runtime in elm code, rather than in JavaScript "kernel code".
As part of this investigation I replaced the definition of the `Task` type documented [here](https://package.elm-lang.org/packages/elm/core/latest/Task) and [here](https://package.elm-lang.org/packages/elm/core/latest/Platform#Task).
Currently, the `Platform` module defiens the `Task` type as follows
```elm
type Task err ok = Task
```

<span id="a1"></span>
This is possibly the most important type in the elm runtime and yet, as far as elm code can tell, contains no data.
This is a stub definition, elm's typechecker can use it to verify that elm code using `Task`s is written correctly&nbsp;[[1]](#f1).
<span id="a2"></span>

My goal is to create a runtime for elm mostly written in elm, and so the definition of the `Task` type seemed a good place to start.
After much iteration&nbsp;[[2]](#f2) I settled on the following definition of a `Task` which I placed in a new module `Platform.Scheduler`.

```elm
module Platform.Scheduler exposing (Task(..), DoneCallback, TryAbortAction)

type Task val
    = Value val
    | AndThen (HiddenVal -> Task val) (Task HiddenVal)
    | AsyncAction (DoneCallback -> TryAbortAction) TryAbortAction


type HiddenVal
    = HiddenVal Never


type alias DoneCallback =
    Task val -> ()


type alias TryAbortAction =
    () -> ()
```

Two details of this definition are worth noting:

1. The `HiddenVal` type allows me to write down the definition of the `Task` type  which would be immposible to do in elm code.
  I will need to use a kernel function to convert values of some generic type `a`   into `HiddenVal` in the body of the `Task.andThen` function.
2. There is only one type parameter.
    This simplifies the defintion and the implemention of `Task` related functions.

I can then define the `Platform.Task` type as an alias to this definition.

```elm
module Platform exposing (Task, ...)

import Platform.Scheduler

type alias Task err ok =
    Platform.Scheduler.Task (Result err ok)
```

Brilliant!
Only there are two problems:

1. This is a MAJOR api change according to `elm diff` as it considers type aliases to a type and that type as different.
2. When compiling effect managers the elm compiler checks the type of the special functions that an event manager must define.
    It checks that the return type is `Platform.Task err ok`.
    However, with the changes I tried above the return types of these special effect manager functions become `Platform.Scheduler.Task err ok`.
    The compiler does not see that the special functions use a type alias from the `Platform` module as the canonicalisation performed by the compiler removes this information.

The obvious solution is to move the type definition back into the `Platform` module.
Next time I will explain the issues I faced when trying to do just that.

---

* <b id="f1">[1](#a1)</b> With this definition, the compiler can not provide any b to the code implementing elm's runtime.
* <b id="f2">[2](#a2)</b> Maybe that iteration will be the subject of a future post.
