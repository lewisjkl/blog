---
layout: post
title:  "Require is an Anti-Pattern"
author: jeff
categories: [ scala, refined ]
tags: []
image: assets/images/kinsey-Gk-MMZu9rmE-unsplash.jpg
description: "Scala's require method is frequently used where better options are available."
featured: true
hidden: false
---

Scala's `require` method is frequently used where better options are available.

To get started, I will show an example that demonstrates what `require` is frequently used for.

```scala
def transferMoney(amount: Int): TransferResult = {
  require(amount > 0, "amount to transfer must be greater than zero")
  // transfer money operations here
}
```

When I first encountered this pattern, I thought it was great. I loved being able to define up front in a function what its inputs should look like. Another nicety from this was that for the rest of the function I could assume that my required conditions were true. This often allowed the logic in functions to be simpler. So what’s the issue then? Well, I see several. 

1. The function is not _actually_ covering the entire domain of its inputs. If a function doesn’t do this, it should really be a `PartialFunction` so the caller knows it doesn’t cover the entire domain.
2. The caller has no way of knowing these restrictions exist without looking at the function directly.
3. There’s a good chance a caller won’t “safely” call this function (e.g. wrapped in a `Try`) so the thrown exception could actually crash your program 😬.

The more I analyze this approach, the more I realize that I do like the _idea_ of locking down my types more. What I don’t like about this is the way it is accomplished.

## A Better Way
An open-source library called Refined has support for what I believe to be the ideal way to restrict types beyond their native definitions.

{% scalafiddle template="Result" %}
```scala
import eu.timepit.refined.api.Refined
import eu.timepit.refined.auto._
import eu.timepit.refined.numeric._

type PositiveNonZeroInt = Int Refined Positive
final case class TransferResult(amountTransferred: PositiveNonZeroInt)

def transferMoney(amount: PositiveNonZeroInt): TransferResult = {
  // transfer money operations here
  TransferResult(amount)
}

transferMoney(100)
```
{% endscalafiddle %}

What Refined allowed me to do in the above example is create a new type that holds only positive non-zero integers. This means that what was previously being checked by a `require` statement is now built in to the type itself. This gives us the same benefits of `require` without any of the drawbacks enumerated above. Our function is now defined over its inputs’ entire domain and the caller has full visibility into the restrictions being imposed. 

## Conclusion
Using refined types is an easy way to lock down your types in a way that is transparent to the caller.

Reach out on Twitter or Github if you have any questions, comments, or corrections.
