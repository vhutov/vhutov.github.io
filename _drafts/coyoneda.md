Co-Yoneda helps you benefit from Functor composition law:
`fa.map(f).map(g) = fa.map(f.andThen(g))`.
They do exactly what's written - fuse subsequent computations into a single one.
Let's say you have logic:
```scala
def check[F[_]: Functor](fi: F[Int]) =
  fi.map(_ + 1)
    .map(_ * 2)
    .map(_.toString)
```
If you apply this to a huge List, you'll end up having 2 intermediate unnecessary lists with impact on GC and running time.
With Co-Yoneda it will be like:
```scala
def checkYo[F[_]: Functor](fi: F[Int]) = {
  check(Yoneda(fi))
    .run
}

def checkCoyo[F[_]: Functor](fi: F[Int]) = {
  check(Coyoneda.lift(fi))
    .run
}
```
Both of them will fuse subsequent maps into a single map, meaning you will not produce intermediate lists.