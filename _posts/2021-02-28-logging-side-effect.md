---
layout: post
title:  "Logging: It's a Side Effect"
author: jeff
categories: [ scala, fp, logging, side-effects ]
tags: []
description: "Although it may feel like logging is exempt from being considered a side effect, treating it like any other operation provides benefits."
featured: false
hidden: false
---

Logging is often not thought of as something that can be a side-effect. I think this is because we tend to only think about side-effects as operations that have some impact on the end user. However, treating logging as different from other effects comes at the detriment of reasoning and leads to unexpected behavior.

## Motivating Example

Here is an example of logging being treated as **not** being a side effect when it should be:

```scala
def checkExternalServiceUp(workerId: String): IO[Boolean] = {
  logger.info(s"Worker $workerId checking whether external service is up.")
  Get("http://www.example.com/health").sendRequest[IO]
}
```

In this example, we have a function that returns an IO which sends a health check request to an external API. Now imagine that in our application, we call this function once on startup with the workerId that has been assigned to our service. From there, we schedule the returned effect to run once every n minutes and then save the result to a database.

The issue here is that our log is only going to get logged out once rather than each time we run the effect. This means that we are going to be mislead in our logs to think that the health checks are not being sent when they really are. Further, the one time that our application does log that line out, it will not even be correlated to the sending of a health check request. Rather it is actually logged out when the effect is created.

## Solution

Luckily, our solution here is quite simple. First we need to select a logging library that supports logging without side effects. My personal choice is usually to use [log4cats](https://github.com/typelevel/log4cats).

```scala
def checkExternalServiceUp(workerId: String): IO[Boolean] = {
  logger.info(s"Worker $workerId checking whether external service is up.") *>
  Get("http://www.example.com/health").sendRequest[IO]
}
```

Notice here that the only difference in this function is the presence of the `*>` operator between the two lines of code. This operator comes from the Cats `Apply` type class and is an alias for the `Apply#productR` method. The other difference is that `logger` would have a different type than it did in the first example (see the log4cats [readme](https://github.com/typelevel/log4cats) for more info).

The `*>` operator here makes it so that the info log is returned along with the get request as part of the `IO`. Now, every time that the returned effect is run, the log will be displayed as expected.