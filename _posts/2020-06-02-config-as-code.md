---
layout: post
title:  "Configuration as Code with Ciris"
author: jeff
categories: [ config, configuration, ciris ]
tags: []
image: assets/images/markus-spiske-qjnAnF0jIGk-unsplash.jpg
description: "Configuration as code allows you to avoid common runtime errors by providing compile-time safety."
featured: true
hidden: false
---

Traditionally, working with configuration means you define sets of values without type-safety and look them up in your application at runtime. What could go wrong? Turns out, a lot of things. You may not realize that you put a configuration at the wrong path. You may load the configuration as the incorrect type. You may compile multiple sub-projects together and have configurations overwritten in unexpected ways. It can be a very frustrating experience.

This is where configuration as code comes in. Configuration as code gives you compile-time safety, while still giving you the option of overriding configuration values at runtime. It is the best of both worlds. My favorite configuration as code library is called Ciris. Here I will show you the basics of how I work with Ciris.

## Configuration as Code with Ciris
Personally, I treat configuration as code as the place where I define the structure of my configuration along with default values. From there, I override any default values at runtime using environment variables. There are other approaches that you could take such as reading in configurations from files instead of environment variables, but this best fits my use case. Here is a small example using Ciris.

```scala
import cats.implicits._
import ciris._

object Config {
    final case class AppConfig(
        host: String,
        port: Int
    )

    val appConfig: ConfigValue[AppConfig] =
        (
            env("HOST").as[String].default("0.0.0.0"),
            env("PORT").as[Int].default(8080)
        ).parMapN(AppConfig)
}
```

This setup gives me a value `appConfig` that I can reference from other code to access the host and port that this application should bind to. `appConfig` can be kept as is with default values or `HOST` and `PORT` can be provided as environment variables at runtime to override those defaults.

You may notice that this setup still allows for runtime failures. For example, the environment variable `PORT` is expected to be an integer, but you can provide any type you want there. To account for this scenario, I resolve all of my configurations inside of my main method. This way, any runtime failures happen right when the application boots and not later on in the runtime.

```scala
object Main extends IOApp {

  val myApp: IO[Unit] = for {
      cfg <- Config.appConfig.load[IO]
      _ <- new MyProgram(cfg)
  } yield ()

  def run(args: List[String]): IO[ExitCode] = {
      myApp.as(ExitCode.Success)
  }
}
```

Any runtime configuration failures that are going to happen in the above program will happen right when I call `Config.appConfig.load[IO]`.

If you were to use a more traditional configuration approach, such as Typesafe Config, you could still do all config resolution at the time your application boots to isolate runtime errors to application startup. However, this is going to require a lot more manual work that Ciris takes care of for you with the DSL it provides for constructing a `ConfigValue`. Further, this approach would still leave you with uncertainty around the way configurations from separate sub-projects are merged together.

The good news is that you have now seen basically everything you need to in order to work with Ciris. It is a simple library with few methods that you need to know. That being said, there are a few other features that I want to mention.

## Other Ciris Features

#### Secret

Ciris allows you to load configurations wrapped in a `Secret` class if they contain sensitive values that you don't want to end up in application logs.

```scala
env("DATABASE_PASSWORD").as[DatabasePassword].secret // gives you a Secret[DatabasePassword]
```

#### Refined Support

Ciris provides a module for parsing configurations directly into Refined types.

```scala
env("DATABASE_USERNAME").as[NonEmptyString].default("username")
```

#### Custom Decoders

Ciris allows you to decode your own custom types. You do this by implementing `ConfigDecoder` for your custom types. Take for example a type that you have defined which needs to run validation on its constructor arguments.

```scala
final case class SlackChannel private (value: String)

object SlackChannel {
    def create(value: String): Option[SlackChannel] {
        if (value.startsWith("#")) {
            value.some
        } else {
            None
        }
    }
}

private implicit val slackChannelDecoder: ConfigDecoder[String, SlackChannel] =
    ConfigDecoder
      .identity[String]
      .mapOption("SlackChannel")(SlackChannel.create)
```

## Conclusion

Hopefully this article has shown you how useful configuration as code can be. I have found it to be a huge improvement over traditional configuration resolution methods. I appreciate the ability it gives me to define my configuration structure in type-checked Scala code. Ciris is my favorite library that I have used for configuration as code. It provides a super simple DSL for resolving configurations and providing default values. It is also completely extensible so you can add whatever functionality you want to it.