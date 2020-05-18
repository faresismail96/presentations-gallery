# PureConfig

#### Reading ADTs as Env. Vars



## Presentation

PureConfig allows us to quickly and easily load configurations from config files in a type safe and boiler plate free manner.



## Quick Note

I am using the following version of pureconfig

```scala mdoc
"com.github.pureconfig" %% "pureconfig" % "0.9.1"
```



### Requirement

A given job will be executed daily. The job can either be executed in FullMode (where the entire history will be treated)
Or it can be executed in DeltaMode where only data between two given dates will be treated.



### One possible Impl

```scala

import java.time.LocalDate
sealed trait AuditMode

object AuditMode {

  final case object FullMode extends AuditMode

  final case class DeltaMode(startDate: LocalDate, endDate: LocalDate)
      extends AuditMode
}
```



Conf Case class: 

```scala
import com.audit.model.AuditMode

final case class AuditConf(name: String, auditMode: AuditMode)
```

reference.conf

```hocon
app1 {
  name: "First Test"
  audit-mode: fullmode
}
```

To Load Conf

```scala
import cats.syntax.either._
import com.audit.config.AuditConf
Either.catchNonFatal(pureconfig.loadConfigOrThrow[AuditConf]("app1"))

// Compile error
```



## Why?

```text
could not derive a ConfigReader instance for type AuditConf,
 because:
  - missing a ConfigReader instance for type AuditMode,
 because:
    - missing a ConfigReader instance for type DeltaMode,
 because:
      - missing a ConfigReader instance for type LocalDate
    Either.catchNonFatal(pureconfig.loadConfigOrThrow[AuditConf]("app1"))
```

again, why?

```text
We cannot provide a [[ConfigConvert]] for 
[[java.time.LocalDate]] because traditionally
 there are many different
[[java.time.format.DateTimeFormatter]]s 
to parse a [[java.time.LocalDate]] 
from a [[java.lang.String]].
```



## Solution?

```scala
import java.time.LocalDate
import pureconfig._
import java.time.format.DateTimeFormatter

sealed trait AuditMode
object AuditMode {
  final case object FullMode extends AuditMode

  final case class DeltaMode(startDate: LocalDate, endDate: LocalDate)
      extends AuditMode

  implicit val localDateConfigConverter: ConfigConvert[LocalDate] = 
    pureconfig.configurable.localDateConfigConvert(
    DateTimeFormatter.ofPattern("yyyy-MM-dd")
    )
}
```



## Result?

```text
Left(pureconfig.error.ConfigReaderException: 
Cannot convert configuration to a AuditConf. Failures are:
  at 'app1.audit-mode':
    - reference.conf:3 Expected type OBJECT. Found STRING instead.
)
```

```hocon
app1 {
  name: "First Test"
  audit-mode: fullmode
}
```

in comes the `EnumCoproductHint`



```scala
sealed trait AuditMode
object AuditMode {

  final case object FullMode extends AuditMode

  final case class DeltaMode(startDate: LocalDate, endDate: LocalDate)
      extends AuditMode

  implicit val coproductHint: EnumCoproductHint[AuditMode] =
    new EnumCoproductHint[AuditMode]

  implicit val localDateConfigConverter: ConfigConvert[LocalDate] =
    pureconfig.configurable.localDateConfigConvert(
      DateTimeFormatter.ofPattern("yyyy-MM-dd"))
}
```



#### Result

```text
Right(AuditConf(First Test,FullMode))
```



#### What about with DeltaMode?

```hocon
app1 {
  name: "First Test"
  audit-mode: fullmode
}
```

Replace `EnumCoproductHint` with `FieldCoproductHint`



```scala
sealed trait AuditMode
object AuditMode {

  final case object FullMode extends AuditMode
  final case class DeltaMode(startDate: LocalDate, endDate: LocalDate)
      extends AuditMode

  implicit val coproductHint: FieldCoproductHint[AuditMode] =
    new FieldCoproductHint[AuditMode]("mode")
  implicit val localDateConfigConverter: ConfigConvert[LocalDate] =
    pureconfig.configurable.localDateConfigConvert(
      DateTimeFormatter.ofPattern("yyyy-MM-dd"))
}
```



```hocon

app1 {
  name: "First Test",
  audit-mode: {
    mode: deltamode
    start-date: 2020-04-25
    end-date: 2020-04-28
  }
}
```

//Right(AuditConf(First Test,DeltaMode(2020-04-25,2020-04-28)))

Ok Cool...



## Now the tricky part

Environment Variables:

```hocon
app1 {
  name: "First Test",
  audit-mode: ${AUDIT_MODE}
}
```

```scala
import com.audit.config.AuditConfigLoader

object Main extends App with AuditConfigLoader {

  System.setProperty("AUDIT_MODE", "{ mode : fullmode }")

  println(conf)

}
//Left(pureconfig.error.ConfigReaderException: Cannot convert configuration to a com.pureconfig.config.AuditConf. Failures are:
//  at 'app1.audit-mode':
//    - Expected type OBJECT. Found STRING instead.
//)
```



## Why?

<https://github.com/lightbend/config/blob/master/HOCON.md#substitution-fallback-to-environment-variables>

"environment variables always become a string value, though if an app asks for another type automatic type conversion would kick in"

Apparently not that well...

Luckily we have a helpful method: `getConfigConvert`



...



```scala
sealed trait AuditMode
object AuditMode {
  final case object FullMode extends AuditMode
  final case class DeltaMode(startDate: LocalDate, endDate: LocalDate)
      extends AuditMode
  val fch: FieldCoproductHint[AuditMode] =
    new FieldCoproductHint[AuditMode]("mode")
  val auditModeConfigConverter: ConfigConvert[AuditMode] = {
    implicit val coproductHint: FieldCoproductHint[AuditMode] = fch
    implicit val ldc = pureconfig.configurable.localDateConfigConvert(
      DateTimeFormatter.ofPattern("yyyy-MM-dd"))
    Configs.getConfigConvert[AuditMode]("AuditMode")
  }
  object implicits {
    implicit val coproductHint: FieldCoproductHint[AuditMode] = fch
    implicit val modeConfigConverter: ConfigConvert[AuditMode] =
      auditModeConfigConverter
  }
}
```



```scala

System.setProperty("AUDIT_MODE", """{ "mode" : "fullmode"}""")

// Right(AuditConf(First Test,FullMode))

System.setProperty(
"AUDIT_MODE",
"""{ "mode" : "deltamode", "start-date": 2020-04-25, "end-date": 2020-04-28}""")

//Right(AuditConf(First Test,DeltaMode(2020-04-25,2020-04-28)))
```
