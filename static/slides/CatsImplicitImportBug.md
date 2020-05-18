# Bug Due to cats Implicit



## Recap on Semigroups

A ```Semigroup``` for a given type ``A`` has a single operation that takes two values of type ``A`` and returns a single value of the same type. This operation needs to be associative.
We will call this operation combine for simplicity.

```scala mdoc
trait Semigroup[A] {
  def combine(x: A, y: A): A
}
```



### Some Examples

```scala
import cats.kernel.Semigroup

implicit val minStringSemigroup: Semigroup[String] =
    implicitly[Ordering[String]].min
Semigroup[String].combine("1", "2")
```




```scala
import cats.kernel.Semigroup

implicit val minStringSemigroup: Semigroup[String] =
    implicitly[Ordering[String]].min
Semigroup[String].combine("1", "2")
// res0: String = 1
```



### Other Examples

```scala

import cats.kernel.Semigroup
import cats.instances.option._

implicit val minStringSemigroup: Semigroup[String] =
    implicitly[Ordering[String]].min
Semigroup[Option[String]].combine(None, Some("1"))
//==> Some("1")
Semigroup[Option[String]].combine(None, None)
//==> None
Semigroup[Option[String]].combine(Some("1"), Some("2"))
//==> Some("1")
```



## Later On In Time

Someone added the following line in the code for a dev that was completely unrelated to our function:

```scala
import cats.implicit._
```

Code is put in prod and all is fine... no one notices anything...



# Except

It was not fine...



### When running the same code above

```scala

import cats.kernel.Semigroup
import cats.instances.option._

implicit val minStringSemigroup: Semigroup[String] =
    implicitly[Ordering[String]].min
Semigroup[Option[String]].combine(None, Some("1"))
//==> Some("1")
Semigroup[Option[String]].combine(None, None)
//==> None
Semigroup[Option[String]].combine(Some("1"), Some("2"))
//==> Some("12")!!!
```

Why?



### Lets look at implicit hints

Before:

```scala
import cats.kernel.Semigroup
import cats.instances.option._

implicit val minStringSemigroup: Semigroup[String] =
    implicitly[Ordering[String]].min
Semigroup[Option[String]](catsKernelStdMonoidForOption(minStringSemigroup))
    .combine(Some("1"), Some("2"))
// ==> Some("1")
```




With Cats Implicit in Scope:

```scala
import cats.kernel.Semigroup
import cats.instances.option._

implicit val minStringSemigroup: Semigroup[String] =
    implicitly[Ordering[String]].min
Semigroup[Option[String]](catsKernelStdMonoidForOption(catsKernelStdMonoidForString))
    .combine(Some("1"), Some("2"))
// ==> Some("12")
```



```scala
implicit val catsKernelStdMonoidForString: Monoid[String]
 = new StringMonoid
```

```scala
class StringMonoid extends Monoid[String] {
  def empty: String = ""
  def combine(x: String, y: String): String = x + y

  ...
}
```



### Best Practice?

Do not import cats implicit. Instead, only import what you need.
