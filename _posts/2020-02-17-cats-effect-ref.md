---
layout: post
title:  "Cats Effect Ref"
author: jeff
categories: [ scala, cats-effect, ref, fp, concurrency, mutability, state ]
tags: []
image: assets/images/koushik-chowdavarapu-2xUdMrorXjc-unsplash.jpg
description: "Ref is a great choice for holding concurrent application state."
featured: true
hidden: false
---

As a newcomer to functional programming, I thought mutability was prohibited. Although I figured out how to use immutability in most situations, I found there are obvious cases where it doesn’t work such as modeling and updating in-memory state (e.g. in-memory caches and counters). I was wrong to think that functional programming mandates only using immutable values. Functional programming requires referential transparency, but as long as that holds, mutability is fine.

In some cases, in-memory state can be modeled using the [state monad](https://typelevel.org/cats/datatypes/state.html). It provides a way to represent state that is updated sequentially. If you have a case where state moves serially from one value to another by taking the previous state as an input, then you should look at the state monad. However, if your use case needs to update state concurrently rather than sequentially, look no further than Cats Effect’s `Ref` (Actually, you can also look at Zio for another `Ref` implementation if you prefer).

## Ref: An Overview
`Ref` is a mutable reference which:

1. Is non-blocking
2. Is accessed and modified concurrently
3. Is modified atomically
4. Is dealt with using only pure functions

`Ref` provides these features by combining Java’s `AtomicReference` and Cats Effect’s `Sync` type class. To show how these building blocks combine to create `Ref`, I am going to walk through each of the major operations that are available on Ref:

1. `of`
2. `get`
3. `set`
4. `update`
5. `modify`

Note: I have changed some of the implementations below to simplify them for this post. They are mostly the same. If you wish to see the real implementation, check it out [here](https://github.com/typelevel/cats-effect/blob/master/core/shared/src/main/scala/cats/effect/concurrent/Ref.scala). 

## Ref#of
The `of` method is a constructor for `Ref`.

### Implementation

```scala
def of[F[_], A](a: A)(implicit F: Sync[F]): F[Ref[F, A]] = F.delay(new Ref[F, A](new AtomicReference[A](a)))
```

This constructor creates a `Ref` filled with an `AtomicReference` containing an initial value `a`. It does this within the `F` context to maintain purity.

This operation is pure since it is lifted into `F` using `delay` from the `Sync` type class. No effects are executed by calling this function. It merely returns an effect  `F` with the ability to run at some future point to create the `Ref`. The same is true for all of the functions within `Ref`. They are pure because they are taking advantage of the delay behavior provided by the `Sync` type class.

### Example

```scala
val ref: IO[Ref[IO, Int]] = Ref.of[IO, Int](42)
```

Note: This is equivalent to `Ref[IO].of(42)` because `Ref` takes advantage of [partially applied types](https://typelevel.org/cats/guidelines.html#partially-applied-type-params).

## Ref#get
This operation is the simplest of all operations on `Ref`. It gets the value that the `AtomicReference` currently contains.

Here, `ar` is the `AtomicReference` supplied within the `of` constructor. 

### Implementation

```scala
def get: F[A] = F.delay(ar.get)
```

The `get` operation on an `AtomicReference` happens in a single machine instruction. This means that any number of threads can be performing a `get `on an `AtomicReference` at the same time without any contention. As the name `AtomicReference` implies, thread-safety holds for all of its operations. This is because `AtomicReference` cleverly performs each of its operations in a single machine instruction. 

Another key thing to note about  `AtomicReference` is that it is backed by a `volatile` variable. `volatile` variables will be read from and written to the host computer’s main memory rather than a CPU cache. This means that even if our program is running on different CPUs it will behave as though it is running on a single CPU. For more on volatile variables, see [here](http://tutorials.jenkov.com/java-concurrency/volatile.html).

### Example

{% scalafiddle template="CatsEffect" %}
```scala
val printRef = for {
  ref <- Ref[IO].of(42)
  contents <- ref.get
} yield println(contents)
printRef.unsafeRunSync()
```
{% endscalafiddle %}

## Ref#set
This operation is structured the same as `get` except we are performing a `set` operation this time. This operation relies on the same `volatile` and thread-safe nature of `AtomicReference`.

### Implementation

```scala
def set(a: A): F[Unit] = F.delay(ar.set(a))
```

### Example

{% scalafiddle template="CatsEffect" %}
```scala
val printRef = for {
  ref <- Ref[IO].of(42)
  _ <- ref.set(21)
  contents <- ref.get
} yield println(contents)
printRef.unsafeRunSync()
```
{% endscalafiddle %}

## Ref#update
Having `get` and `set` operations is great, but they alone don’t allow us to atomically update the value contained inside of the reference. For example, if we wanted to take whatever value is stored in our `Ref` and increment it by one, we might be tempted to do something like:

```scala
def getThenSet(ref: Ref[IO, Int]): IO[Unit] = {
  ref.get.flatMap { contents =>
    ref.set(contents + 1)
  }
}

val printRef = for {
  ref <- Ref[IO].of(42)
  _ <- NonEmptyList.of(getThenSet(ref), getThenSet(ref)).parSequence
  contents <- ref.get
} yield println(contents)
```

The issue here is that we are going to get inconsistent outputs when running this code. We have two separate processes working in parallel to perform a `getThenSet` operation. Both of these processes may `get` the value in the `Ref` before the other process has a chance to increment the value using the `set` operation. This means that the execution could look like:

```
process1: Ref#get -> returns 42
process2: Ref#get -> returns 42
process1: Ref#set -> sets to 43
process2: Ref#set -> sets to 43
```

We would have expected the output of this program to be `44`, but due to `get` and `set` happening in two different operations, we end up potentially incrementing the same value twice.

This is where update comes in. 

### Implementation

```scala
def update(f: A => A): F[Unit] =
  modify(a => (f(a), ()))
```

This operation is logically more simple than `modify`, which is why I have included it before `modify` in this post. Don’t worry too much about the implementation here until after you have read the section about `modify`. For now the important thing to note is what `update` does.

`update` takes a function `f` as an argument that will be called with whatever value is currently contained inside of the `Ref`. Then whatever value the function `f` returns will be the new value inside of the `Ref`.

How this all works will make more sense after we cover `modify` below. 

### Example

{% scalafiddle template="CatsEffect" %}
```scala
val printRef = for {
  ref <- Ref[IO].of(42)
  _ <- ref.update(_ + 1)
  _ <- ref.update(_ + 1)
  contents <- ref.get
} yield println(contents)
printRef.unsafeRunSync()
```
{% endscalafiddle %}

## Ref#modify
We’ve shown that update can perform a “get and set” operation (although we have yet to explain how), but we haven’t yet seen how to perform a “get and set and get” operation. Although it may seem unlikely that such an operation is needed, it is a common one. Take the example above using `update` to increment a counter. This method works unless we need to atomically access the value after it’s been incremented. For example, if we want to keep track of the number of requests that an HTTP server has seen over time, we could use a `Ref` with the `update` function to add one to a counter for every request. However, the `update` function wouldn’t work if we wanted to return the new request number after updating it. We could use `update` followed by a `get`, but this is not atomic because the `get` is performed as a separate machine instruction. This could lead to two requests getting the same ID. 

This is the problem that modify solves. 

### Implementation

```scala
def modify[B](f: A => (A, B)): F[B] = {
  @tailrec
  def spin: B = {
    val c = ar.get
    val (u, b) = f(c)
    if (!ar.compareAndSet(c, u)) spin
    else b
  }
  F.delay(spin)
}
```

Before I dive into explaining the `modify` method, we need to understand what the `AtomicReference#compareAndSet` method does. Compare and set (CAS) takes two parameters: 1) the value `c` that we expect to be contained in the AtomicReference and 2) the value `u` to which we want to update the `AtomicReference`. CAS will update the `AtomicReference` to the value of `u` if and only if the value currently held within the `AtomicReference` is equal to the value of `c`. If the value of `c` is not equal to the value currently held in the AtomicReference, then compareAndSet does not update the reference and returns `false` to indicate such. The CAS operation is atomic because both the compare and the set functionality happen in a single machine instruction.

Now let’s look at the `Ref#modify` method. This method takes a function `f` as a parameter. This function accepts the value that is currently stored in the `Ref` and returns a tuple containing: 1) the value with which to update the `Ref` and 2) the value to return from the `modify` method.

The `modify` method contains a tail-recursive (stack-safe) inner method called `spin`. The `spin` method forms what is called a compare and set loop. The reason this loop is needed is that `modify` requires two operations to work: `AtomicReference#get` and `AtomicReference#compareAndSet`. Each of these operations on its own is atomic, but when we call both of them in sequence the resulting operation is not atomic. Thus the `AtomicReference` may get updated between the call to `get` and the call to `compareAndSet` causing the `compareAndSet` operation to fail (return false). For this reason, we need to retry this operation repeatedly until it finally succeeds. It will succeed when there are no updates to the underlying `AtomicReference` between the calls to `get` and `compareAndSet`.

### Example

{% scalafiddle template="CatsEffect" %}
```scala
val printRef = for {
  ref <- Ref[IO].of(42)
  contents <- ref.modify(current => (current + 20, current + 20))
} yield println(contents)
printRef.unsafeRunSync()
```
{% endscalafiddle %}

## Full Ref Example
Here is a full example using `Ref` to manage the state of a set of bank accounts.

{% scalafiddle template="CatsEffect" %}
```scala
import BankAccounts._

final class BankAccounts(ref: Ref[IO, Map[String, BankAccount]]) {

  def alterAmount(accountNumber: String, amount: Int): IO[Option[Balance]] = {
    ref.modify { allBankAccounts =>
      val maybeBankAccount = allBankAccounts.get(accountNumber).map { bankAccount =>
        bankAccount.copy(balance = bankAccount.balance + amount)
      }
      val newBankAccounts = allBankAccounts ++ maybeBankAccount.map(m => (m.number, m))
      val maybeNewBalance = maybeBankAccount.map(_.balance)
      (newBankAccounts, maybeNewBalance)
    }
  }

  def getBalance(accountNumber: String): IO[Option[Balance]] = ref.get.map(_.get(accountNumber).map(_.balance))

  def addAccount(account: BankAccount): IO[Unit] =
    ref.update(_ + (account.number -> account))
}

object BankAccounts {
  type Balance = Int
  final case class BankAccount(number: String, balance: Balance)
}

val example = for {
	ref <- Ref[IO].of(Map.empty[String, BankAccount])
  bankAccounts = new BankAccounts(ref)
  _ <- bankAccounts.addAccount(BankAccount("1", 0))
  _ <- bankAccounts.alterAmount("1", 50)
  _ <- bankAccounts.alterAmount("1", -25)
  endingBalance <- bankAccounts.getBalance("1")
} yield println(endingBalance)

example.unsafeRunSync()
```
{% endscalafiddle %}

This example is still fairly basic and is certainly not intended to show how you would implement real banking software. However, it gives you an idea of what is possible using `Ref`.

## Regions of Sharing
Here is a final note about something that can be confusing when first working with `Ref`. Whenever we are working with `Ref` or its operations, we are using a for-comprehension. As you may know, for-comprehensions are syntactic sugar that represent a series of `flatMap` calls followed by a `map` at the end. Because `Ref`’s operations are contained within the `F` effect context, we need to `map` or `flatMap` over the results of those operations to access the values. This is referred to as a region of sharing.

```scala
Ref[IO].of(42).flatMap { i =>
  // This is the region of sharing
}
// We can't access the value the Ref contains from out here
```

Regions of sharing are convenient because they give us the ability to reason about our code locally. In other words, we can see every possible operation that could be modifying our `Ref` from right inside its region of sharing. We don’t have to look at anything outside of that region since it can’t do anything to access the `Ref`.

## Conclusion
I hope that this overview has given you a better intuition for what `Ref` is and how you can use it within your code. `Ref` is a super powerful construct that can give you an easy way to manage application state in a way that is both atomic and pure. For more on `Ref` I would recommend checking out these resources:

* [Cats Effect Documentation](https://typelevel.org/cats-effect/concurrency/ref.html)
* [Composable Concurrency with Ref + Deferred](https://vimeo.com/366191463)

If you have any questions or comments about this article, please reach out to me on Twitter. Thanks for reading!