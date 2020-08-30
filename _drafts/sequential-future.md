Here is how you can run a list of asynchronous computations one after another in non-blocking fashion:
```scala
def sequential[A, B]
  (as: List[A])
  (f: A => Future[B])
  (implicit ec: ExecutionContext): Future[List[B]] =
  as.foldLeft(successful(List())) {
    case (previousFuture, a) =>
      for {
        results <- previousFuture
        b <- f(a)
      } yield b :: results
  }.map(_.reverse)
```