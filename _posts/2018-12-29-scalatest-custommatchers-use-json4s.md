---
layout: post
title: ScalaTest의 Custom Matchers 구현
date: 2018-12-29 17:00:00 +0900
categories: scala
---

# ScalaTest의 Custom Matchers 구현

이번 글에서는 Scala에서 대표적인 테스트도구인 ScalaTest의 기능을 확장하여 두 Json의 비교를 위한 테스트케이스에서 실패시에 상세한 메시지를 보여줄 수 있는 기능을 구현해보겠습니다. 

## ScalaTest, Json4s

- 먼저 [ScalaTest](http://www.scalatest.org)는 Scala 생태계에서 가장 유연하고 널리 사용되는 테스트 도구입니다. Java의 대표적인 테스트 도구는 JUnit이듯이 Scala의 대표적인 테스트도구는 ScalaTest입니다. ScalaTest는 Scala 뿐만 아니라 Scala.js와 Java 코드도 테스트가 가능하다고 합니다.
- Json을 파싱하는 라이브러리로는 [json4s](https://github.com/json4s)를 활용하였습니다.

## 환경 구성
Scala의 빌드도구인 SBT로 ScalaTest, Json4s를 사용하기 위한 환경 구성을 합니다.

```
libraryDependencies ++= Seq(
  "org.json4s" % "json4s-jackson_2.12" % "3.6.3",
  // json4s를 구성하는 방법으로 native 또는 jackson 을 사용할 수 있는데 필자가 익숙한 jackson을 사용하였습니다.
  "org.scalactic" %% "scalactic" % "3.0.5",
  "org.scalatest" %% "scalatest" % "3.0.5" % "test"
)
```

## 테스트 작성
이번 글은 ScalaTest와 Json4s의 사용법이 아니라 ScalaTest의 Matchers를 확장하여 Json을 테스트하는 글이기 때문에 구체적인 사용방법에 대해서는 생략합니다..
일반적으로 Json4s를 이용한 테스트 작성은 아래와 같이 작성합니다.

```
import org.scalatest.FunSpec
class JsonTest extends FunSpec {
  it("compare two json") {
    import org.json4s._
    import org.json4s.jackson.JsonMethods._

    val a = parse(
      """
         |{
         |  "field": "value",
         |  "field1": "value1"
         |}
       """.stripMargin)

    val b = parse(
      """
         |{
         |  "field": "value1",
         |  "field2": "value2"
         |}
       """.stripMargin)

    assert(a == b)
  }
}
```

테스트 실패 메시지는 아래와 같습니다.

```
JObject(List((field,JString(value)), (field1,JString(value1)))) did not equal JObject(List((field,JString(value1)), (field2,JString(value2))))
ScalaTestFailureLocation: custommatchers.JsonTest at (JsonTest.scala:38)
Expected :JObject(List((field,JString(value1)), (field2,JString(value2))))
Actual   :JObject(List((field,JString(value)), (field1,JString(value1))))
(이하생략)
```

지금은 Element가 5개 이하라 어떤 Element가 잘못되어있는지 한눈에 보이지만, Json의 Element가 많아진다면 찾아내기 아주 어려울 것입니다.

## Json Diff
Json4s는 두 Json을 비교(Diff)하여 어느 Element가 추가되고, 변경되고, 삭제되었는지 확인할 수 있는 기능을 제공합니다.

먼저, Json을 비교하는 구문은 아래와 같습니다.

```
val diffResult = Diff.diff(a, b)

if (diffResult.changed != JNothing) {
  println(s"changed : ${compact(diffResult.changed)}")
}
if (diffResult.added != JNothing) {
  println(s"added : ${compact(diffResult.added)}")
}
if (diffResult.deleted != JNothing) {
  println(s"deleted : ${compact(diffResult.deleted)}")
}
```

변경된(changed) Element, 추가된(added) Element, 삭제된(deleted) Element를 확인해볼 수 있습니다. 

```
changed : {"field":"value1"}
added : {"field2":"value2"}
deleted : {"field1":"value1"}
```

## Custom Matchers

두 Json의 Diff 결과를 보여준다면 어떤 Element가 잘못되어있는지 한눈에 확인할 수 있을 것입니다. ScalaTest의 `Matchers`를 이용하여 작성해볼 수 있습니다.

```
import org.json4s.JsonAST.JValue
import org.scalatest.matchers.{MatchResult, Matcher}

trait JsonMatchers {

  def equals(right: JValue): Matcher[JValue] = new Matcher[JValue] {
    override def apply(left: JValue): MatchResult = {
		// 이부분을 작성하면 됩니다!
    }
  }

}
```

Diff 결과를 Readable하게 작성해봅시다.

```
package custommatchers

import org.json4s.Diff
import org.json4s.JsonAST.{JNothing, JValue}
import org.json4s.jackson.JsonMethods.compact
import org.scalatest.matchers.{MatchResult, Matcher}

import scala.collection.mutable.ListBuffer

trait JsonMatchers {
  def equals(right: JValue): Matcher[JValue] = new Matcher[JValue] {
    override def apply(left: JValue): MatchResult = {
      val diffResult = Diff.diff(left, right)

      val failureMessageBuf = new ListBuffer[String]
      failureMessageBuf.append("Json elements are Different.")

      if (diffResult.changed != JNothing) {
        failureMessageBuf.append(s"  changed : ${compact(diffResult.changed)}")
      }
      if (diffResult.added != JNothing) {
        failureMessageBuf.append(s"  added : ${compact(diffResult.added)}")
      }
      if (diffResult.deleted != JNothing) {
        failureMessageBuf.append(s"  deleted : ${compact(diffResult.deleted)}")
      }

      MatchResult(
        matches = false,
        failureMessageBuf.mkString("\n"),
        "Json elements are same.")
    }
  }
}
```
`MatchResult`는 테스트의 결과값과 실패시에 메시지를 담는 아규먼트를 받고 있습니다. Diff 결과를 넣어 테스트 실패 메시지를 작성했습니다.

이렇게 작성한 `JsonMatchers`를 실제 테스트 코드에 적용해보았습니다.

```
import org.scalatest.{FunSpec, Matchers}
import org.json4s._
import org.json4s.jackson.JsonMethods._

class JsonTest extends FunSpec with Matchers with JsonMatchers {
  it("compare two json") {
    val a = parse(
      """
         |{
         |  "field": "value",
         |  "field1": "value1"
         |}
       """.stripMargin)

    val b = parse(
      """
         |{
         |  "field": "value1",
         |  "field2": "value2"
         |}
       """.stripMargin)

    a should equals(b)
  }
}
```
`JsonMatchers trait`의 사용을 적용하고 `JsonMatchers`에 작성한 `equals` 메소드를 이용하여 테스트 구문을 작성합니다.

실패 메시지는 아래와 같습니다.

```
Json elements are Different.
  changed : {"field":"value1"}
  added : {"field2":"value2"}
  deleted : {"field1":"value1"}
ScalaTestFailureLocation: custommatchers.JsonTest at (JsonTest.scala:25)

(이하생략)
```

changed 메시지는 좀 불친절해보입니다. 실제 기대하는 값이 무엇인지 확인하기 어렵기에 역으로 우항과 좌항을 비교하여 어떤 값이 나와야하는지도 출력해봅시다.

```
if (diffResult.changed != JNothing) {
  failureMessageBuf.append(s"  changed : right -> ${compact(diffResult.changed)}")
  failureMessageBuf.append(s"            left  -> ${compact(Diff.diff(right, left).changed)}")
}
```

이제 실패 메시지는 이렇게 출력됩니다.

```
Json elements are Different.
  changed : right -> {"field":"value1"}
                    left  -> {"field":"value"}
  added : {"field2":"value2"}
  deleted : {"field1":"value1"}
ScalaTestFailureLocation: custommatchers.JsonTest at (JsonTest.scala:25)
(이하 생략)
```


이렇게 완성한 전체 코드는 아래와 같습니다.

```
package custommatchers

import org.json4s.Diff
import org.json4s.JsonAST.{JNothing, JValue}
import org.json4s.jackson.JsonMethods.compact
import org.scalatest.matchers.{MatchResult, Matcher}

import scala.collection.mutable.ListBuffer

trait JsonMatchers {

  def equals(right: JValue): Matcher[JValue] = new Matcher[JValue] {
    override def apply(left: JValue): MatchResult = {
      val diffResult = Diff.diff(left, right)

      val failureMessageBuf = new ListBuffer[String]
      failureMessageBuf.append("Json elements are Different.")

      if (diffResult.changed != JNothing) {
        failureMessageBuf.append(s"  changed : right -> ${compact(diffResult.changed)}")
        failureMessageBuf.append(s"            left  -> ${compact(Diff.diff(right, left).changed)}")
      }
      if (diffResult.added != JNothing) {
        failureMessageBuf.append(s"  added : ${compact(diffResult.added)}")
      }
      if (diffResult.deleted != JNothing) {
        failureMessageBuf.append(s"  deleted : ${compact(diffResult.deleted)}")
      }

      MatchResult(
        matches = false,
        failureMessageBuf.mkString("\n"),
        "Json elements are same.")
    }

    override def toString(): String = s"Json Matchers : `${compact(right)}`"
  }

}
```

지금까지 ScalaTest의 Matchers 를 customize한 Matchers를 작성하여 두 Json 의 비교를 할 수 있는 코드를 작성해보았습니다. 이 글을 통해 ScalaTest와 Json4s를 더욱 풍성하게 사용할 수 있길 바랍니다.

## Reference
- https://github.com/json4s/json4s
- http://www.scalatest.org/user_guide/using_matchers#usingCustomMatchers