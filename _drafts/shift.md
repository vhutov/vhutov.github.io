Beware when using Async.
Given function:
```scala
def doAsync(): F[Unit] = Async[F].async[Unit] { cb =>
  global.execute(() => cb(Right()))
}
```
The `cb` function will continue running the IO loop in the callback thread, which is not what you likely to want:
```scala
def wrong =
  for {
    res <- doAsync()
    _ <- doStuff()
  } yield res
```
The fix is simply ask IO to shift to the correct context:
```scala
def correct =
  for {
    res <- doAsync()
    _ <- ContextShift[F].shift
    _ <- doStuff()
  } yield res
```