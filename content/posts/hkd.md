---
title: "Higher-kinded data in Scala 3"
date: 2021-05-07T14:11:50+02:00
draft: true
tags: [
    "fp",
    "scala3",
]
---

Higher-kinded data (HKD) takes your existing data classes and wraps each field in a higher-kinded type, allowing you to do some funky things. I have recently seen a few presentations of this which piqued my interest sufficiently to try it out for myself. Given Scala 3 is just around the corner, I will use this as a chance to also try that out.

I will link the talks, along with source code and blogs I found along the way, at the end - they are great resources to go deeper than I have here.

## Terminology

To get started, let's break down some of the concepts in that first sentence. A data class in Scala is best represented with a case class. Let's say a person with a name, age, postcode and possibly a nickname:

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

An improvement over using `null`s or `Map[String, Any]` but now we have a class almost identical to `Person` that needs to be kept in sync.

Next, we are asked to provide an API that can stream revisions for each field. Let's try to define a streaming case class:

```scala
case class StreamingPerson(
    name: Stream[String], 
    age: Stream[Int], 
    postcode: Stream[String], 
    nickname: Stream[Option[String]]
)
```

Our ScalaJS people display page will now be able to update each person's field in real-time by subscribing to each of these streams.

At this point we can see a pattern. The `MaybePerson` and `StreamingPerson` case classes use the same fields and underlying types as the `Person` class but with a different wrapper in each case.

Using higher-kinded types, we can make reduce this boilerplate substantially:

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

Anytime we add a new field to PersonF, we need not touch the other types. Our work will focus on implementing the other update call for that field or streaming the other field or whatever it may be.

## Validation

Focusing on the `addPerson` and `updatePerson` functions from before, it might be best to validate the values passed to these functions before committing the changes to the database. Furthermore, if those API requests originate from some user-facing UI, getting back an error message alongside each field would be helpful.

Ordinarily, we would then write a function `validatePerson` which then might call `validateName`, `validateAge`, `validatePostcode` and `validateNickname`. We quickly run into the same problem as before where new fields means more validations.

Let's define two new type aliases that can help us out:

```scala
type Validation[T] = T => Either[String, T]
type ValidatedPerson = PersonF[Validation]
```

What we are saying here is each field of `ValidatedPerson` will be a function that returns either the value passed into it or an error message. Let's take a look at an example with some silly validations:

```scala
val personValidations = ValidatedPerson(
    name => Either.cond(name.startsWith("C"), name, "Name does not being with 'C'"),
    age => Either.cond(age > 0, age, "Age must be greater than 0"),
    postcode => Either.cond(postcode.matches(postcodeRegex), postcode, "Postcode doesn't look right"),
    nickname => Right(nickname)
)
```

Now the question is, how do we apply these validations to an instance of `Person` or `MaybePerson`?


## Testing

`type ListPerson = PersonF[List]`