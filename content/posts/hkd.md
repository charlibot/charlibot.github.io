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

To get started, let's break down some of the concepts in that first sentence. A data class in Scala is best represented with a case class. Let's say a person with a name, age, date of birth and possibly a nickname:

```scala
case class Person(
    name: String, 
    age: Int, 
    dob: LocalDate, 
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

