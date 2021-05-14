---
title: "Higher-kinded data in Scala 3"
date: 2021-05-07T14:11:50+02:00
tags: [
    "fp",
    "scala3",
]
---

Higher-kinded data (HKD) takes your existing data classes and wraps each field in a higher-kinded type, allowing you to do some funky things. I have recently seen a few presentations of this which piqued my interest sufficiently to try it out for myself. Given Scala 3 is just around the corner, I will use this as a chance to also try that out.

I will link the talks, along with source code and blogs I found along the way, at the end. They are great resources to go deeper than I will do here.

## Terminology

To get started, let's break down some of the terms in that first sentence. A data class in Scala is best represented with a case class. Let's say a person with a name, age, postcode and possibly a nickname:

```scala
case class Person(
    name: String, 
    age: Int, 
    postcode: String, 
    nickname: Option[String]
)
```

I find higher-kinded types easiest to understand with an example. They are usually used in the contexts of type classes. A functor is the most recognisable example:

```scala
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]
```

`F[_]` here is a higher-kinded type because it takes another type to return a proper type. Given this, we can implement `Functor` for many concrete data types that fit, such as `Option`, `List` and `IO`. For example, we can write the Option instance as:

```scala
given Functor[Option] with 
  def map[A, B](fa: Option[A])(f: A => B): Option[B] =
    fa.map(f) 
```

This delegates to the underlying `Option`'s `map` method which performs the function `f` if the value is present, otherwise it does nothing.

Now we can look at the motivation for HKD.

## Motivation

Say we had an API to add a person using the case class above which does some IO in doing so (e.g. inserting to a database):

```scala
def addPerson(person: Person): IO[PersonId]
```

Now we need to add another API to update this person. However, not all the fields would be updated. We could use a `Map[String, Any]` in our update function but that is not ideal. We could pass a `Person` instance with `null` or values representing no update for fields we do not want to be part of the update call but, again, this is not great.

A nicer solution is to create a person where each field is optional:

```scala
case class MaybePerson(
    name: Option[String], 
    age: Option[Int], 
    postcode: Option[String], 
    nickname: Option[Option[String]]
)
```

Now we would have 

```scala
def updatePerson(id: PersonId, person: MaybePerson): IO[Unit]
```

An improvement over using `null`s or `Map[String, Any]` but now we have a class almost identical to `Person` that needs to be kept in sync if any fields are updated, removed or added.

Next, we are asked to provide an API that can stream revisions for each field. Let's try to define a streaming case class:

```scala
case class StreamingPerson(
    name: Stream[String], 
    age: Stream[Int], 
    postcode: Stream[String], 
    nickname: Stream[Option[String]]
)
```

Our ScalaJS page that display a list of people will now be able to update each person's field in real-time by subscribing to each of these streams.

At this point we can see a pattern. The `MaybePerson` and `StreamingPerson` case classes use the same fields and underlying types as the `Person` class but with a different wrapper in each case.

Using higher-kinded types, we can reduce this boilerplate substantially:

```scala
case class PersonF[F[_]](
    name: F[String], 
    age: F[Int], 
    postcode: F[String], 
    nickname: F[Option[String]]
)

type Identity[A] = A
type Person = PersonF[Identity]
type MaybePerson = PersonF[Option]
type StreamingPerson = PersonF[Stream]
```

Anytime we add a new field to PersonF, we need not touch the other types. Our work will focus on implementing the update call for that field or streaming the other field or whatever it may be.

## Validation

Focusing on the `addPerson` and `updatePerson` functions from before, it might be best to validate the values passed to these functions before committing the changes to the database. Furthermore, if those API requests originate from some user-facing UI, getting back an error message alongside each field would be helpful.

Ordinarily, we would write a function `validatePerson` which then might call `validateName`, `validateAge`, `validatePostcode` and `validateNickname`. We quickly run into the same problem as before where new fields means more validations that might fall out of sync with the underlying data class.

Let's define two new type aliases that can help us out:

```scala
type Validation[T] = T => Either[String, T]
type ValidatedPerson = PersonF[Validation]
```

What we are saying here is each field of `ValidatedPerson` will be a function that returns either the value passed into it or an error message. Let's take a look at an example with some silly validations:

```scala
val personValidations = PersonF[Validation](
    name => Either.cond(name.startsWith("C"), name, "Name does not begin with 'C'"),
    age => Either.cond(age > 0, age, "Age must be greater than 0"),
    postcode => Either.cond(postcode.matches(postcodeRegex), postcode, "Postcode doesn't look right"),
    nickname => Right(nickname)
)
```

How do we apply these validations to an instance of `Person` or `MaybePerson`? The operative word being "apply", which we will explore more later. We previously introduced `Functor` without much ceremony. This is a type class that injects a function into a data type. We also provided an instance of `Functor` for `Option`. Let's rework that a little to make use of Scala 3's extension methods and add a `Functor` instance for `Identity`:

```scala
trait Functor[F[_]]:
  extension [A](fa: F[A]) 
    def map[B](f: A => B): F[B]

given Functor[Identity] with
  extension [A](fa: Identity[A]) 
    def map[B](f: A => B): Identity[B] =
      f(fa) 

given Functor[Option] with
  extension [A](fa: Option[A])
    def map[B](f: A => B): Option[B] =
      fa.map(f) 
```

Now we can write a function that relies on the the Functor capability that `Option` and `Identity` has to give us a validated person:

```scala
type Validated[F[_]] = [A] =>> F[Either[String, A]]
def validatePersonF[F[_]: Functor](personF: PersonF[F]): PersonF[Validated[F]] = 
  PersonF(
    personF.name.map(personValidations.name),
    personF.age.map(personValidations.age),
    personF.postcode.map(personValidations.postcode),
    personF.nickname.map(personValidations.nickname)
  )
```

And if we wanted to return whether the validation completed as a whole where we fail with the validated person if at least one of the validations did not pass or returning the person otherwise:

```scala
def validatePersonFComplete[F[_]: Traverse](personF: PersonF[F]): Either[PersonF[Validated[F]], PersonF[F]] = 
  val v = validatePersonF(personF)
  val p = for 
    name <- v.name.sequence
    age <- v.age.sequence
    postcode <- v.postcode.sequence
    nickname <- v.nickname.sequence
  yield PersonF(name, age, postcode, nickname)
  p.fold(_ => Left(v), Right(_))
```

There's a bit to unpack here before moving on. If you're already familiar with `Traverse` and what those `.sequence` calls are doing, feel free to skip ahead to the next section, otherwise, let's dive in.

Our `v.age` from the snippet above is of type `F[Either[String, Int]]`, however we want to get the `Either` on the outside. That's where `.sequence` comes in, this essentially turns the types inside out, moving the `F` inside so we end up with `Either[String, F[Int]]`. The for comprehension helps us then construct our person on the right side if everything went well.

To be able to call `.sequence`, the `F` needs to be traversable, hence the type class `Traverse`:

```scala
trait Traverse[F[_]] extends Functor[F]:
  extension [A](fa: F[A])
    def traverse[G[_]: Applicative, B](f: A => G[B]): G[F[B]]
    def map[B](f: A => B): F[B] = fa.traverse[Identity, B](f)
  extension [G[_]: Applicative, A](fga: F[G[A]])
    def sequence: G[F[A]] = fga.traverse(ga => ga)
```

In fact, `sequence` is written in terms of `traverse` which is a more generic function so let's focus on that. It is often used to perform some effect, for example taking a name and looking up an ID in a database, `String => IO[Int]`, on a `List` of items, names in this case, returning the list within a single effect, `IO[List[Int]]` instead of `List[IO[Int]]`. However, this ability to switch the place of the effect can be helpful in other places like ours.

Looking at the signature a little closer, we see `G` is required to have an instance of the `Applicative` type class:

```scala
trait Applicative[F[_]] extends Functor[F]:
  def pure[A](x: A): F[A]
  extension [A](fa: F[A])
    def ap[B](ff: F[A => B]): F[B]
    def map[B](f: A => B): F[B] =
      fa.ap(pure(f))
    def map2[B, C](fb: F[B])(f: (A, B) => C): F[C] =
      fb.ap(fa.map(a => (b: B) => f(a, b)))
```

`Applicative` gives us the ability to lift an object into the `F` type with `pure`. It also gives us the ability to take two `F` objects and apply a function on both of their internal values together, producing a new `F` with `map2`. Note that `map2` is written in terms of `ap` which I find to be less intuitive than `map2` so will forego explanation. 

Mapping these type classes back to our validation function, we need our `F[_]` to have `Traverse` instances and `Either[String, _]` needs to be applicative. On the latter:

```scala
given [A]: Applicative[[B] =>> Either[A, B]] with
  def pure[B](x: B): Either[A, B] = Right(x)
  extension [B](fa: Either[A, B])
    def ap[C](ff: Either[A, B => C]): Either[A, C] =
      ff match 
        case Right(f) => fa.map(f)
        case Left(l) => Left(l)
```

And our `Traverse` instances for `Identity` and `Option`:

```scala
given Traverse[Identity] with
  extension [A](fa: Identity[A])
    def traverse[G[_]: Applicative, B](f: A => G[B]): G[Identity[B]] =
      f(fa)

given Traverse[Option] with
  extension [A](fa: Option[A])
    def traverse[G[_]: Applicative, B](f: A => G[B]): G[Option[B]] =
      fa match 
        case Some(a) => summon[Applicative[G]].map(f(a))(Some(_))
        case None => summon[Applicative[G]].pure(None)
```

## Do it all over again

To recap, we have redefined our class representing a person to use a higher-kinded type `F[_]`. This gave us the ability to reuse our data class to represent one with optional fields, streamable fields and let us define a set of validations for each field. We then created a helpful function to return our validated person or the set of errors if any were found.

If we had to start do the process again with another case class, we would find ourselves writing the same boilerplate to validate each field of the class and then sequencing and combining all the fields. Let's see if we can remove some of that boilerplate.

We previously discussed the `Applicative` type class which gives us the `map2` method:

```scala
def map2[B, C](fa: F[A])(fb: F[B])(f: (A, B) => C): F[C]
```

If we squint a little, we could see that our person object could map into `fa` and validation object into `fb` then the function `f` would somehow map over each field and run the validation across the corresponding value from the person object, ultimately returning a new validated person object.

Using the `Applicative` type class directly is not possible though since the type signature of `PersonF[F[_]]` does not quite match the required `F[A]`. We need to go one level higher which is usually denoted with a `K` suffix on the type classes:

```scala
trait FunctorK[U[_[_]]]:
  extension [F[_]](u: U[F])
    def mapK[G[_]](f: [T] => F[T] => G[T]): U[G]

trait ApplyK[U[_[_]]] extends FunctorK[U]:
  extension [F[_]](u: U[F])
    def map2K[G[_], H[_]](v: U[G])(f: [T] => (F[T], G[T]) => H[T]): U[H]
    def mapK[G[_]](f: [T] => F[T] => G[T]): U[G] = 
      u.map2K(u)([T] => (t: F[T], _: F[T]) => f(t))
```

Note: we do not need the full power of `Applicative` with `pure`. Instead, we need only `map2` coming from `Apply` which sits between `Functor` and `Applicative` in the type class hierarchy.

The `mapK` function takes a higher-kinded data class, `U[_[_]]`, with a higher-kinded type, `F[_]`, and a polymorphic function that can convert our `F[T]`s into `G[T]`s for any type `T`. An example function could be `[T] => (t: Identity[T]) => Some(t)` which would wrap all the fields of a data class in a `Some`. The `map2K` function is similar but takes an additional data class with a different higher-kinded type and thus the polymorphic function takes an additional argument. If we can derive `ApplyK` for our case classes we will be on the way to our goal of cutting down the boilerplate.

We can leverage the fact case classes are products in Scala to derive `ApplyK`:

```scala
import scala.compiletime.summonFrom
import scala.deriving.Mirror

inline def deriveApplyKForCaseClass[U[_[_]] <: Product]: ApplyK[U] = 
  new ApplyK[U] {
    extension [F[_]](u: U[F])
      def map2K[G[_], H[_]](v: U[G])(f: [T] => (F[T], G[T]) => H[T]): U[H] = summonFrom {
        case p: Mirror.ProductOf[U[F]] =>
          p.fromProduct(new Product {
            def canEqual(that: Any): Boolean = true
            def productArity: Int = u.productArity
            def productElement(n: Int): Any =
              f(u.productElement(n).asInstanceOf[F[Any]], v.productElement(n).asInstanceOf[G[Any]])
          }).asInstanceOf[U[H]]
        case _ => sys.error("cannot handle this")
      }
  }
```

To briefly summarise `map2K`'s implementation: 
  - we start with `summonFrom` which does an implicit search for the types in the pattern match, in this case a `Mirror.ProductOf[U[F]]`. Use of this function means we must make the function `inline`
  - `Mirror.ProductOf[U[F]]` provides type level information on our product `U`. In this case, we use it to build our case class after applying the function. There is a lot more to mirrors so if interested check out the docs
  - `fromProduct` takes a product that can be used to build an instance of `U`. We create a new product with the same arity as the original and where the nth element is the application of the function `f` on the nth element of `u` and `v` case classes. We have to do some ugly casts but I'm not sure if that's avoidable

Now we can rewrite our validation function from before:

```scala
given ApplyK[PersonF] = deriveApplyKForCaseClass[PersonF]

def validatePersonF[F[_]: Functor](personF: PersonF[F]): PersonF[Validated[F]] = 
  personF.map2K(personValidations)(
    [T] => (value: F[T], validation: Validation[T]) => value.map(validation)
  )
```

Any new fields added to the `PersonF` would not require any updates to this function now. We could even make this more generic, giving us the ability to validate any case class that has an instance of `ApplyK`:

```scala
def validateU[U[_[_]]: ApplyK, F[_]: Functor](u: U[F], validations: U[Validation]): U[Validated[F]] =
  u.map2K(validations)(
    [T] => (value: F[T], validation: Validation[T]) => value.map(validation)
  )
```

The last step in validating our case class is to return either our original case class if all validations succeeded or return the one with the error messages otherwise. We are aiming for `Either[U[Validation[F]], U[F]]`.

Looking back at the implementation in `validatePersonFComplete`, we can see the bulk of the work is in the for comprehension where each field has the same `sequence` function applied, producing `Either[String, F[T]]` for each field type `T`, and then a new `PersonF` is constructed within the context of this for comprehension, producing an `Either[String, PersonF[F]]`. This looks like traverse: performing the same function on each item, in this case each field, producing an `Either`, that ends up wrapping the `PersonF`. Let's take a look at `TraverseK`:

```scala
trait TraverseK[U[_[_]]] extends FunctorK[U]:
  extension [F[_]](uf: U[F])
    def traverseK[V[_]: Applicative, G[_]](f: [T] => F[T] => V[G[T]]): V[U[G]]
    def mapK[G[_]](f: [T] => F[T] => G[T]): U[G] = 
      traverseK[Identity, G]([T] => (ft: F[T]) => f(ft))
```

We will have our `uf` containing our validated `PersonF`, `PersonF[[T] =>> F[Either[String, T]]]`, and so need to define the function `f` that will return an `Applicative`. We already provided an instance of `Applicative` for `[T] =>> Either[A, T]` and so it looks like we can use that and the `traverseK` will then return `Either[String, PersonF[F]]` which is what we're looking for. It can be a little confusing to map the types we have to this function because the `F` in our desired returned type is actually the `G` in the function's context bounds and the `[T] =>> F[Either[String, T]]` is the `F`.

The `f` we will provide to this function is then `[T] => (x: F[Either[String, T]]) => x.sequence`. Given a type `T` and value `F[Either[String, T]]`, if we flip the `F` and `Either` with `sequence` we will get the type we want of `Either[String, F[T]]`.

Before going any further, let's see if we can derive `TraverseK` for our case class using a similar process as in `ApplyK`:

```scala
inline def deriveTraverseKForCaseClass[U[_[_]] <: Product]: TraverseK[U] = 
  new TraverseK[U] {
    extension [F[_]](u: U[F])
      def traverseK[V[_]: Applicative, G[_]](f: [T] => F[T] => V[G[T]]): V[U[G]] = summonFrom {
        case p: Mirror.ProductOf[U[F]] =>
          val appliedF = u.productIterator.toList.map(t => f(t.asInstanceOf[F[Any]]))
          val vTuple: V[Tuple] = appliedF.foldRight(summon[Applicative[V]].pure(EmptyTuple)) { (vgs, x) =>
            x.map2(vgs)((tuple, a) => (a *: tuple) )
          }

          vTuple.map { tuple =>
            p.fromProduct(new Product {
              def canEqual(that: Any): Boolean = true
              def productArity: Int = tuple.productArity
              def productElement(n: Int): Any =
                tuple.productElement(n)
            }).asInstanceOf[U[G]]
          }
      }
  }
```

As before, we require `U` to be a product so we can summon the mirror. The first thing we do after summoning is apply the function `f` to each of the fields producing a list of `V[G[Any]]`. Since `V` is applicative, we can build a tuple within the context of `V` by making use of `map2` whilst folding over the list. This results in `V[(G[Any], G[Any], G[Any], ..., G[Any])]` which we can then `map` into and create our case class with the mirror.

Rewriting `validatePersonFComplete` will then be:

```scala
given TraverseK[PersonF] = deriveTraverseKForCaseClass[PersonF]

def validatePersonFComplete[F[_]: Traverse](personF: PersonF[F]): Either[PersonF[Validated[F]], PersonF[F]] = 
  val v = validatePersonF(personF)
  val e = v.traverseK([T] => (x: F[Either[String, T]]) => x.sequence)
  e.fold(_ => Left(v), Right(_))
```

As before with `validateU`, we can make this more generic:

```scala
def validateUComplete[U[_[_]]: TraverseK: ApplyK, F[_]: Traverse](u: U[F], validations: U[Validation]): Either[U[Validated[F]], U[F]] =
  val v = validateU(u, validations)
  val e = v.traverseK([T] => (x: F[Either[String, T]]) => x.sequence)
  e.fold(_ => Left(v), Right(_))
```

And there we have it, we can validate any flat higher-kinded data given the higher-kinded type is traversable.

### Derivation

One final useful thing we can do is implement [type class derivation](https://dotty.epfl.ch/docs/reference/contextual/derivation.html). Then, when we define our HKD case class, we can write something like `A[F[_]](a: F[Int]) derives ApplyK` to automatically have an `ApplyK` instance for `A`.

Since we want to derive both `ApplyK` and `TraverseK`, and they both inherit `mapK` from `FunctorK`, we will need to define a new type class that is the union of them both:

```scala
trait ApplyTraverseK[U[_[_]]] extends ApplyK[U], TraverseK[U]:
  extension [F[_]](u: U[F])
    override def mapK[G[_]](f: [T] => F[T] => G[T]): U[G] = 
      u.traverseK[Identity, G]([T] => (ft: F[T]) => f(ft))
```

Then we define a companion object with the method `derived` which copies the `traverseK` and `map2K` implementations from before:

```scala
object ApplyTraverseK:
  inline def derived[U[_[_]] <: Product]: ApplyTraverseK[U] = new ApplyTraverseK[U] {
    extension [F[_]](u: U[F])
      def traverseK[V[_]: Applicative, G[_]](f: [T] => F[T] => V[G[T]]): V[U[G]] = summonFrom {
        case p: Mirror.ProductOf[U[F]] =>
          val appliedF = u.productIterator.toList.map(t => f(t.asInstanceOf[F[Any]]))
          val vTuple: V[Tuple] = appliedF.foldRight(summon[Applicative[V]].pure(EmptyTuple)) { (vgs, x) =>
            x.map2(vgs)((tuple, a) => (a *: tuple) )
          }
          vTuple.map { tuple =>
            p.fromProduct(new Product {
              def canEqual(that: Any): Boolean = true
              def productArity: Int = tuple.productArity
              def productElement(n: Int): Any =
                tuple.productElement(n)
            }).asInstanceOf[U[G]]
          }
        case _ => sys.error("cannot handle")
      }
      def map2K[G[_], H[_]](v: U[G])(f: [T] => (F[T], G[T]) => H[T]): U[H] =
        summonFrom {
          case p: Mirror.ProductOf[U[F]] =>
            p.fromProduct(new Product {
              def canEqual(that: Any): Boolean = true
              def productArity: Int = u.productArity
              def productElement(n: Int): Any =
                f(u.productElement(n).asInstanceOf[F[Any]], v.productElement(n).asInstanceOf[G[Any]])
            }).asInstanceOf[U[H]]
          case _ => sys.error("cannot handle")
        }
  }
```

And now, we can go back to our `PersonF` case class and append `derives ApplyTraverseK`:

```scala
case class PersonF[F[_]](
    name: F[String], 
    age: F[Int], 
    postcode: F[String], 
    nickname: F[Option[String]]
) derives ApplyTraverseK
```

An instance of `PersonF` will have the `mapK`, `map2K` and `traverseK` methods available to it now. 

## Summary

That was a pretty long journey but I wanted to capture the whole process of going from our `Person` case class definition to providing a generic way of validating HKD and deriving the required type classes. All of the code snippets, with some examples that pass and fail validations, are available on [Github](https://github.com/charlibot/hkd). Some of what is here can be found in existing libraries and need not be reimplemented. In particular, [cats](https://github.com/typelevel/cats) can provide our non-K type classes (`Functor`, `Applicative`, `Traverse`) and instances for `Option`, `Either` and `Identity` (in Cats just `Id`). 

[Oleg Nizhnikov - HKD: stem cells for data](https://www.youtube.com/watch?v=k8IgRayL4vo) was the spark that started this write-up. The [associated source](https://github.com/Odomontois/data-lab) also greatly helped. There, he provides derivations for `RepresentableK` for case classes amongst other things which I did not need here. I also looked at the following which helped wrap my head around this topic:
- [Katrix's perspective library](https://github.com/Katrix/perspective)
- [Chris Penner's talk](https://www.youtube.com/watch?v=sIqZEmnFer8)
- Michael Thomas' two repos:
  - [Higher-kinded data](https://github.com/Michaelt293/higher-kinded-data)
  - [Higher-kinded aggregations](https://github.com/Michaelt293/higher-kinded-aggregations)
- [Philipp Martini's blog post](https://blog.philipp-martini.de/blog/magic-mirror-scala3/) was also helpful to understand the mirror usage in the deriving functions.

Taking a glimpse at any of those links shows I've barely scratched the surface here. Although for now, I think we can leave it there and call it a day.

Thanks for reading!
