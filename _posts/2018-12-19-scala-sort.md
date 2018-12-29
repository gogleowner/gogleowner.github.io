---
layout: post
title: scala 정렬함수 비교
date: 2018-12-19 21:05:00 +0900
categories: scala
---

# scala 정렬함수 비교

스칼라에서 리스트를 정렬하는 메소드는 세가지가 있다. 이 세가지의 용법과 차이에 대해 정리를 해놓아야겠다.

- `sortBy`
- `sortWith`
- `sorted`

# scala 공식 문서 내용

- `sortBy`
    - 암시적으로 지정된 `Ordering`을 통해서 `Seq`를 정렬한다.
    - `String, BigDecimal, Int`등의 정렬 순서를 결정하는 `Ordering` 은 `scala.math.Ordering`에 미리 정의 되어있다.
    - `String` 비교를 위한 `Ordering`의 구현 내용을 살펴보면 익히알듯, `java` 로 구현된 `String`의 비교 연산을 하도록 `compareTo` 를 호출하게 되어있다.

            trait Ordering[T] extends Comparator[T] with PartialOrdering[T] with Serializable {
              ...
            	trait StringOrdering extends Ordering[String] {
                def compare(x: String, y: String) = x.compareTo(y)
              }
              implicit object String extends StringOrdering
              ...
            }

    - 스칼라 홈페이지에 나와있는 예시

            val words = "The quick brown fox jumped over the lazy dog".split(' ')
            // this works because scala.Ordering will implicitly provide an Ordering[Tuple2[Int, Char]]
            words.sortBy(x => (x.length, x.head))
            res0: Array[String] = Array(The, dog, fox, the, lazy, over, brown, quick, jumped)

        - 위의 예시에서 보면 `Ordering[Tuple2[Int, Char]]` 으로 변환된다고 나와있는데 이에 대한 실제 정렬 구현체도 `Ordering` 안에 구현되어있다.

                implicit def Tuple2[T1, T2](implicit ord1: Ordering[T1], ord2: Ordering[T2]): Ordering[(T1, T2)] =
                  new Ordering[(T1, T2)]{
                    def compare(x: (T1, T2), y: (T1, T2)): Int = {
                      val compare1 = ord1.compare(x._1, y._1)
                      if (compare1 != 0) return compare1
                      val compare2 = ord2.compare(x._2, y._2)
                      if (compare2 != 0) return compare2
                      0
                    }
                  }

        - 첫번째 값으로 비교가 되면 두번째 값은 보지 않게 되어있다. 위의 예시로 보면, 문자열의 길이는 오름차순으로, 길이가 같다면 더 빠른 알파벳이 앞으로 오게 되어있다.(대문자는 소문자가 더 앞에 있다.)

- `sortWith`

    비교 함수를 통해서 순서를 정렬한다. (크기가 정해져있지 않은 콜렉션에서는 종료되지 않는다.)

        List("Steve", "Tom", "John", "Bob").sortWith(_.compareTo(_) < 0) == List("Bob", "John", "Steve", "Tom")

    - 위의 정렬 코드는 `List("Steve", ..).sortWith((o1, o2) => o1.compareTo(o2) < 0)`으로 이렇게 좀더 풀어서 표현할 수 있다.
    - `scala`에서는 앞의 변수와 뒤의 변수는 `_` 를 통해 구분할 수 있어서 위와 같이 표현해도 된다.
    - `java`의 `Comparator`를 lambda식으로 표현한 문법이 같다.
- `sorted`
    - 객체의 특정 값으로 비교할 수 있도록 한다.

            List("Steve", "Tom", "John", "Bob").sortBy(str => str.length)

# Reference

- [https://www.scala-lang.org/api/2.12.x/scala/collection/immutable/List.html](https://www.scala-lang.org/api/2.12.x/scala/collection/immutable/List.html)
- [http://techie-notebook.blogspot.com/2014/07/difference-between-sorted-sortwith-and.html](http://techie-notebook.blogspot.com/2014/07/difference-between-sorted-sortwith-and.html)
- 역순 정렬 : [https://stackoverflow.com/questions/7802851/whats-the-best-way-to-inverse-sort-in-scala](https://stackoverflow.com/questions/7802851/whats-the-best-way-to-inverse-sort-in-scala)
- [https://gist.githubusercontent.com/gsluthra/80555ed4af24bea244b5/raw/a8fdbf5df47a1329f199d1e8eca317ccb775c236/Sorting_Lists_In_Scala](https://gist.githubusercontent.com/gsluthra/80555ed4af24bea244b5/raw/a8fdbf5df47a1329f199d1e8eca317ccb775c236/Sorting_Lists_In_Scala)
