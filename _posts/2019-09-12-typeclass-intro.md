---
layout: post
title:  "Functional Programming in Scala Part 1: Typeclasses"
author: jeff
categories: [ scala, fp, typeclass ]
tags: []
image: assets/images/11.jpg
description: "An introduction into typeclasses - what they are and how to use them."
featured: true
hidden: false
---

## Introduction
This is the first post in a series that I will be doing about functional programming in Scala. I’ve spent the bulk of my career skirting around “scary” functional programming features, but now I am taking the time to learn and incorporate them into my everyday coding.

Hopefully these posts about my learnings will help you out with your understanding of these concepts too.

## Definition of a Typeclass
The definition of typeclasses that I like the most is the following:

> A typeclass is a sort of interface that defines some behavior. If a type is a part of a typeclass, that means that it supports and implements the behavior the typeclass describes. [2]  

To break this definition down a bit, a typeclass essentially represents a set of behaviors. Any type that is defined which is given those behaviors is a part of that typeclass.

It is important to note that those behaviors are not given to a type within the type itself or within the definition of the typeclass. Rather, they are defined separately in implementations of the typeclass. This means that a type can be added to a typeclass without any changes to the type or the typeclass. This makes it so that types can be added to a typeclass without any planning done beforehand and without any need to update the original definitions. This is also referred to as ad-hoc polymorphism because it gives you the ability to create polymorphic types on the fly.

In short, typeclasses have several distinctions and benefits over traditional OOP classes:

1. They decouple data from the behaviors capable using that data
2. They allow for ad-hoc polymorphism
3. More reasons that I will cover in my next post 😃


## Simple Typeclass Example
For this next part, I am going to show a simple example of a typeclass and break down all of its parts. Don’t worry, I realize that this example is contrived.I will show a more realistic example in the next section.

## Realistic Typeclass Example

Here I am going to show how typeclasses can be used to convert regular model case classes into JSON. There are several open source frameworks that actually use this approach for building JSON serializer/deserializer functionality.

The first few sections of code are just setup and have nothing to do with typeclasses. To start out, I pulled the algebraic data type (ADT) that is used by the [json4s](https://github.com/json4s/json4s/blob/fe1164b75679a7db439289e13f1feee19fc66b8f/README.md#guide) library.

{% scalafiddle template="Result" %}

```scala
sealed abstract class JValue
case object JNothing extends JValue // 'zero' for JValue
case object JNull extends JValue
case class JString(s: String) extends JValue
case class JDouble(num: Double) extends JValue
case class JDecimal(num: BigDecimal) extends JValue
case class JInt(num: BigInt) extends JValue
case class JLong(num: Long) extends JValue
case class JBool(value: Boolean) extends JValue
case class JObject(obj: List[JField]) extends JValue
case class JArray(arr: List[JValue]) extends JValue

type JField = (String, JValue)
```

This implicit class provides a method for stringifying the JSON ADTs. Still no typeclasses.
```scala
implicit class JValueStringify(value: JValue) {
  def stringify: String = value match {
    case JNothing => ""
    case JNull => "null"
    case JString(s) => s""""$s""""
    case JDouble(num) => num.toString
    case JDecimal(num) => num.toString
    case JInt(num) => num.toString
    case JLong(num) => num.toString
    case JBool(value) => value.toString
    case JObject(obj) => "{" + obj.map { case (key, value) => s""""$key":${value.stringify}""" }.mkString(",") + "}"
    case JArray(arr) => s"[${arr.map(_.stringify).mkString(",")}]"
  }
}
```

Finally, we define our typeclass. `JsonConverter` is a typeclass that defines a behavior `toJson`. Implementations of this typeclass will take a value such as `String`, `Boolean`, or even `List[String]` and convert them into the appropriate `JValue` types.
```scala
trait JsonConverter[T] {
  def toJson(value: T): JValue
}
```

Here is the companion object for our typeclass. Within it you will find two helper methods.
```scala
object JsonConverter {
  def apply[T: JsonConverter]: JsonConverter[T] = implicitly[JsonConverter[T]]
  
  implicit class JsonConverterOps[T](value: T) {
    def convertToJson(implicit converter: JsonConverter[T]): JValue =
      converter.toJson(value)
  }
  
}

// Json typeclass members

object DefaultJsonProtocol {
  
  implicit object StringJsonConverter extends JsonConverter[String] {
    def toJson(value: String): JValue = JString(value)
  }
  
  implicit object IntJsonConverter extends JsonConverter[BigInt] {
    def toJson(value: BigInt): JValue = JInt(value)
  }
  
  implicit object LongJsonConverter extends JsonConverter[Long] {
    def toJson(value: Long): JValue = JLong(value)
  }
  
  implicit object DoubleJsonConverter extends JsonConverter[Double] {
    def toJson(value: Double): JValue = JDouble(value)
  }
  
  implicit object DecimalJsonConverter extends JsonConverter[BigDecimal] {
    def toJson(value: BigDecimal): JValue = JDecimal(value)
  }
  
  implicit object BoolJsonConverter extends JsonConverter[Boolean] {
    def toJson(value: Boolean): JValue = JBool(value)
  }
  
  implicit def OptionJsonConverter[T: JsonConverter]: JsonConverter[Option[T]] =
    new JsonConverter[Option[T]] {
      def toJson(value: Option[T]): JValue = value match {
        case Some(v) => JsonConverter[T].toJson(v)
        case None => JNothing
      }
    }
    
  implicit def ListJsonConverter[T: JsonConverter]: JsonConverter[List[T]] =
    new JsonConverter[List[T]] {
      def toJson(values: List[T]): JValue = JArray(values.map(v => JsonConverter[T].toJson(v)))
    }
    
  implicit def MapJsonConverter[T: JsonConverter]: JsonConverter[Map[String, T]] =
    new JsonConverter[Map[String, T]] {
      def toJson(values: Map[String, T]): JValue = JObject(values.toList.map { case (k, v) => (k, JsonConverter[T].toJson(v)) })
    }
  
}

// Tests

val field: JObject = JObject(List(("fieldName", JString("field value here"))))

println(field.stringify)

import JsonConverter._
import DefaultJsonProtocol._

println(List("hello").convertToJson.stringify)

println(Map("key" -> "value").convertToJson.stringify)

case class Address(street1: String, aptNum: Long)

case class Person(name: String, address: Address)

object ApplicationJsonProtocol {
  implicit object AddressWriter extends JsonConverter[Address] {
    def toJson(value: Address) = JObject(
      List(
        "street1" -> value.street1.convertToJson,
        "aptNum" -> value.aptNum.convertToJson
      )
    )
  }

  implicit object PersonWriter extends JsonConverter[Person] {
    def toJson(value: Person) = JObject(
      List(
        "name" -> value.name.convertToJson,
        "address" -> value.address.convertToJson
      )
    )
  }
}

import ApplicationJsonProtocol._

val person = Person("Name", Address("streetAddress", 111))

person.convertToJson.stringify
```
{% endscalafiddle %}


## References
Here are some fantastic references that I used in building my understanding of typeclasses. If you are looking for more depth on this subject, these are some great places to start.

1. https://scalac.io/typeclasses-in-scala/
2. http://learnyouahaskell.com/types-and-typeclasses
3. http://learnyouahaskell.com/making-our-own-types-and-typeclasses
4. https://medium.com/@kolemannix/an-introduction-to-typeclasses-in-scala-26d4dc5fdf58
5. https://medium.com/@olxc/type-classes-explained-a9767f64ed2c
