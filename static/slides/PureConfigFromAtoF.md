# PureConfig From A to F



## Presentation

PureConfig allows us to quickly and easily load configurations from config files in a type safe and boiler plate free manner.



## Quick Note

I am using the following verion of pureconfig

```scala mdoc
"com.github.pureconfig" %% "pureconfig" % "0.12.3"
```



### A Very Basic Example

```conf
person {
  name: "Fares"
  age: 23
  field-of-work: "DataEngineer"
  hobbies: ["Biking", "Hiking"]
}
```

```scala mdoc
import pureconfig._
import pureconfig.generic.auto._

final case class Person(name: String,
                        age: Int,
                        fieldOfWork: String,
                        hobbies: List[String])
val person = ConfigSource.default.at("person").loadOrThrow[Person]

println(person)
//Person(Fares,23,DataEngineer,List(Biking, Hiking))
```



One note, even if IntelliJ says that ``import pureconfig.generic.auto._`` is an unused import, its actually used. Its just that IntelliJ is bad at dealing with ``Macros``



## Lets Make It Better

`fieldOfWork` doesn't have to be a ``String``. We could instead represent it in an ADT:

```scala
sealed abstract class FieldOfWork
object FieldOfWork {
  final case object DataEngineer extends FieldOfWork
  final case object Business extends FieldOfWork
}
```



```conf
person {
  name: "Fares"
  age: 23
  field-of-work {
    type: data-engineer
  }
  hobbies: ["Biking", "Hiking"]
}
```

```scala
final case class Person(name: String,
                        age: Int,
                        fieldOfWork: FieldOfWork,
                        hobbies: List[String])
val person = ConfigSource.default.at("person").loadOrThrow[Person]
println(person)
// Person(Fares,23,DataEngineer,List(Biking, Hiking))
```



We had to specify the type of "field-of-work" in the additional fields. Because by default, pureconfig will not know.
As specified in the documentation: [Sealed Families](https://pureconfig.github.io/docs/overriding-behavior-for-sealed-families.html)

But there is a way to change this default behavior, and thats by defining an implicit ``EnumCoproductHint[FieldOfWork]``.



Note that as of version 0.11, this is deprecated and instead we have are asked to use the following:



```scala
import pureconfig.generic.semiauto._

implicit val fieldOfWorkConvert: ConfigReader[FieldOfWork] =
    deriveEnumerationReader[FieldOfWork]
```



we could then write the conf in the following manner:

```conf
person {
  name: "Fares"
  age: 23
  field-of-work: data-engineer
  hobbies: ["Biking", "Hiking"]
}
```



Noticed how we have to write the config in Kebab Case? that is because by default in pureconfig, enumerations are encored as string with the ``kebab-case`` representation of the class name.



### Quick Recap

- Camel Case: Is how you would normally define variable in scala, example: ``val myName: String = "Fares"``.
- Pascal Case: Is how you would normally name your case classes and objects... ``MyExample``



- Kebab/Spinal Case: Is how pureconfig encodes enumerations 😛 ``my-example-is-self-explanatory``
- Snake/Underscore Case: Is how we often name our env variable ``SOME_EXAMPLE_HERE``
- Alternating Caps: Is HoW yOu WoUlD wRiTe If YoU'Re A MoRrOn.



For info ``Alternating Caps`` is not supported out of the box in pureconfig (I wonder why... :roll_eyes:)



## Going Back

If we wanted to change the default representation, we could do it in the implicit:

```scala
  implicit val fieldOfWorkConvert: ConfigReader[FieldOfWork] =
    deriveEnumerationReader[FieldOfWork](
      ConfigFieldMapping(PascalCase, PascalCase))
```



First param to ``ConfigFieldMapping``, is the naming convention used by the case class, second is how the representation in the conf file.

This would allow us to define data engineer in the conf file as: `DataEngineer`

We can define our own instance of `NamingConvention` and pass it to the ``ConfigFieldMapping``



## Overriding Behavior

The same way we overrode the behavior of Sealed Families, we can override the behavior when loading config from a case class by providing a ``ProductHint[T]``.

Here is what we can override:



1. FieldName Mapping to Case Classes (Default is `Kebab Case`)
2. Default Values (Default is to take value from cases class if they exist)
3. Unknown Keys (Default is to ignore)
4. Missing Keys (Ok if optional, otherwise return key not found)



### Mapping FieldNames to Case Classes

```scala
implicit val productHint = ProductHint[SampleConf](new ConfigFieldMapping {
  def apply(fieldName: String) = fieldName.toUpperCase
})
```

Would allow us to load conf where all the keys in the config file are in uppercase.




### Default Values

To no longer use the default values defined in the case classes:

```scala
implicit val hint = ProductHint[SampleConf](useDefaultArgs = false)
```



### Unknown Keys

To no longer ignore unknown keys:

```scala
implicit val hint = ProductHint[SampleConf](allowUnknownKeys = false)
```



### Missing Keys

If for some reason, we would like pureconfig to handle missing keys, we would have to extends ReadsMissingKeys and define a default behavior and return a config reader.

```scala
implicit val maybeIntReader = new ConfigReader[Int] with ReadsMissingKeys {
  override def from(cur: ConfigCursor) =
    if (cur.isUndefined) Right(42) else ConfigReader[Int].from(cur)
}
```



## Loading Classes

PureConfig is based on shapeless and shapeless does not support non case classes. So:

While case classes are supported by pureconfig out of the box, ``class`` is not. For those, we would need to provide an implicit instance of ``ConfigReader[T]``

To do that we can follow one of three options:



1. Modify an existing ConfigReader

2. Use a ConfigReader Factory Method

3. Create an implementation of ConfigReader from scratch




### Modifying an Existing ConfigReader

Assume we have a MyInt that takes an int Value, we can modify the existing ConfigReader[Int] to work on MyInt class.

```scala
implicit val myIntReader = ConfigReader[Int].map(n => new MyInt(n))
```



### Using a FactoryMethod

If we were to use a factory method for the same example above:

```scala
implicit val myIntReader = ConfigReader.fromString[MyInt](
  ConvertHelpers.catchReadError(s => new MyInt(s.toInt)))
```



### Implementing ConfigReader from Scratch

To implement a config reader from scratch we would need to define a from function that takes in the `ConfigCursor` and returns an Either of ``T`` or a list of errors:

```scala
implicit val myIntReader = new ConfigReader[MyInt] {
  def from(cur: ConfigCursor) = cur.asString.map(s => new MyInt(s.toInt))
}

ConfigSource.string("{ n: 1 }").load[MyInt]

```



If it looks easy, its because the example Ive used is rather trivial. In most cases we'd have a more complex class at hand so lets take a shallow dive into those cases.



## Combinators

Combinators provide an easy way to transform existing ConfigReaders to support new types. We've seen map in our first example.

1. Emap
2. orElse



### emap

Allows us to validate the input and provide detailed errors in the case of failure.

From the official doc:

```scala
import pureconfig.error._

case class Port(number: Int)
case class PortConf(port: Port)

// reads a TCP port, validating the number range
implicit val portReader = ConfigReader[Int].emap {
  case n if n >= 0 && n < 65536 => Right(Port(n))
  case n => Left(CannotConvert(n.toString, "Port", "Invalid port number"))
}
```



```scala
ConfigSource.string("{ port = 8080 }").load[PortConf]
// res1: ConfigReader.Result[PortConf] = Right(PortConf(Port(8080)))
ConfigSource.string("{ port = -1 }").load[PortConf]
// res2: ConfigReader.Result[PortConf] = Left(
//   ConfigReaderFailures(
//     ConvertFailure(
//       CannotConvert("-1", "Port", "Invalid port number"),
//       None,
//       "port"
//     ),
//     List()
//   )
// )
```



### orElse

orElse can be used to provide multiple ways to load a config



```scala
val csvIntListReader = ConfigReader[String]
                        .map(_.split(",")
                        .map(_.toInt).toList)
implicit val intListReader = ConfigReader[List[Int]]
                              .orElse(csvIntListReader)

case class IntListConf(list: List[Int])
```

```scala
ConfigSource.string("""{ list = [1,2,3] }""")
            .load[IntListConf]
// res3: ConfigReader.Result[IntListConf] =
//        Right(IntListConf(List(1, 2, 3)))
ConfigSource.string("""{ list = "4,5,6" }""")
            .load[IntListConf]
// res4: ConfigReader.Result[IntListConf] =
//        Right(IntListConf(List(4, 5, 6)))
```



## Config Cursors

In the third method to create a configreader, we talked about defining a `from` method that takes a config cursor and returns a type `T`.

So what are config cursors?



Config Cursor is a wrapper of the class ``ConfigValue`` by Typesafe Config. The added value of using `ConfigCursor` is that most errors are handled automatically and enriched with information regarding the location of the error.

Let us turn our `Person` case class into a class and try to write our own ConfigReader[Person].



The result would look something like this:

```scala
import pureconfig._
import pureconfig.generic.auto._
import cats.syntax.either._ // Because Im working with scala 2.11 (not yet right biased)
implicit val fieldOfWorkConvert: ConfigReader[FieldOfWork] =
  deriveEnumerationReader[FieldOfWork](
    ConfigFieldMapping(PascalCase, PascalCase))
class Person(name: String,
             age: Int,
             fieldOfWork: FieldOfWork,
             hobbies: List[String]) {
  override def toString: String =
    s"""Person($name, $age years old, works as a $fieldOfWork and has the following hobbies: ${hobbies
      .mkString(",")})"""
}
```



```scala

implicit val personReader = ConfigReader.fromCursor[Person] { cur =>
  for {
    objCur <- cur.asObjectCursor // Right if it points to an object left is a list of errors
    nameCur <- objCur.atKey("name")
    name <- nameCur.asString

    ageCur <- objCur.atKey("age")
    age <- ageCur.asInt
    seasonCur <- objCur.atKey("field-of-work")
    season <- fieldOfWorkConvert.from(seasonCur)

    hobbiesCur <- objCur.atKey("hobbies")
    hobbiesList <- hobbiesCur.asList.map(_.map(_.asString.getOrElse("")))

  } yield new Person(name, age, season, hobbiesList)
}

val person = ConfigSource.default.at("person2").loadOrThrow[Person]
println(person)

```



with the following config file:

```conf
person2 {
  name: "Fares"
  age: 23
  field-of-work: DataEngineer
  hobbies: ["Biking", "Hiking"]
}
```

We would get:

``Person(Fares, 23 years old, works as a DataEngineer and has the following hobbies: Biking,Hiking)``

So it is doable, but... its a lot more effort.



## Things I have yet to understand

1. Materialized Derivations
2. Macros in Scala
3. TypeTags
