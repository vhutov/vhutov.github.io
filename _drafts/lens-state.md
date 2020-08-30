Did you know that you can easily convert `Lens[S, A]` to a useful `State[S, B]` modification?
In short, Lens allows you to drill down to a specific value within a complex immutable structure, whereas State describes a computation on immutable structure `S => (S, A)`.
Here are two examples of such conventions:

```scala
def inspectState[S, A, B](L: Lens[S, A])(f: A => B): State[S, B] =
  State.inspect(f.compose(L.get))
def modifyState[S, A](L: Lens[S, A])(f: A => A): State[S, Unit] =
  State.modify(L.modify(f))
```

And how you can use it:

```scala
case class MyState(a: String)
val L: Lens[MyState, String] = GenLens[MyState](_.a)

val update: State[MyState, Unit] = modifyState(L)(_ + " world!")

val initial: MyState = MyState("hello")
val newState: MyState = update.runS(initial).value
require(newState == MyState("hello world!"))
```
