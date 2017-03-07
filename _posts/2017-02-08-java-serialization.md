---
layout: post
title: "java 직렬화(serialization)"
date: 2017-02-08 01:53:00 +0900
categories: java
---

# 직렬화(Serialization)

## 정의

- 객체를 바이트 스트림으로 인코딩, 인코딩된 바이트 스트림으로부터 객체를 복원하는 프레임워크를 제공하는 api
- 원격 통신을 위한 표준 통신 회선 수준의 객체 표현 제공, 자바빈 컴포넌트 아키텍처의 표준 영속 데이터 포맷을 제공
- 직렬화(serializing) : 객체를 바이트 스트림으로 인코딩
	>
	- 인코딩 : 하나의 포맷에서 다른 포맷으로 정보를 변환하는 것
	- 직렬화 과정에서는 객체가 갖는 모든 정보, 즉 자신의 상태 값 및 관련있는 다른 객체들의 정보까지 인코딩
- 역직렬화(deserializing) : 바이트 스트림을 객체로 디코딩


## 조건
- 기본 자료형(boolean, char, byte, short, int, long, float, double)
	- 정해진 바이트의 변수이기 때문에 바이트 단위로 분해하여 전송한 후 조립하는데 문제가 없음
- Serializable 인터페이스를 구현한 객체
	- 기본 자료형과는 달리 객체의 크기는 가변적이며, 객체를 구성하는 자료형들의 종류와 수에 따라 객체의 크기는 다양하게 바뀔수 있기에, 객체의 직렬화를 위해 Serializable 인터페이스를 구현해야 한다.


## 주의점
- 직렬화는 클래스 명 옆에 `implements Serializable` 만 추가하면 클래스의 인스턴스가 직렬화한다는 걸 명시할수있지만, UID를 명시적으로 작성하지 않으면 클래스 내부를 수정할 때마다 UID를 새로 생성하게 되어 java.io.InvalidClassException 이 발생함.
	
	```
java.io.InvalidClassException: com.github.gogleowner.TestClass; 
local class incompatible: 
stream classdesc serialVersionUID = 2876855513509316737, local class serialVersionUID = -95977783010344940
	```



- serialVersionUID : 스트림 고유 식별자(stream unique identifier)
 	- serialVersionUID가 선언하지 않을 경우 실행시점에 serialization을 담당하는 모듈이 디폴트 값을 산정하게 되는데, 그 알고리즘은 [Java(TM) Object Serialization Specification](https://docs.oracle.com/javase/8/docs/platform/serialization/spec/protocol.html)에 정의된 것을 따른다.

## Reference 
- Effective Java 책 내용
	> 직렬화가 가능한 모든 클래스는 고유 식별번호를 갖는다. 만일 static final long 필드인 serialVersionUID를 명시적으로 선언하여 그 번호를 지정하지 않으면, 시스템에서 해당 클래스에 대해 복잡한 절차를 적용하여 런타임시에 자동으로 생성해준다. 이때 그 클래스 이름, 그 클래스가 구현하는 인터페이스들의 이름, 그 클래스의 모든 public과 private 멤버들이 자동으로 값을 생성할 때 영향을 준다. **만일 그것들 중 어느 것이라도 변경하면, 예를 들어, 평범하고 편리한 메소드를 하나 추가하면, 자동으로 생성된 직렬화 버전 UID가 변경된다.(이 경우 클래스 변경 전에 직렬화된 인스턴스의 직렬화 형태는 버전 호환이 안되므로 역직렬화가 안될 것이다.)** 따라서 만일 명시적으로 직렬화 버전 UID를 선언하는데 실패하면(선언을 안하거나 잘못하면), 클래스의 내부 구현 변경 시 호환성이 깨질 것이고 런타임시에 InvalidClassException 예외 발생을 초래하게 된다.
- [스택오버플로우 java-io-invalidclassexception-local-class-incompatible](http://stackoverflow.com/questions/10378855/java-io-invalidclassexception-local-class-incompatible)
- [스택오버플로우 what-is-serialVersionUID](http://stackoverflow.com/questions/285793/what-is-a-serialversionuid-and-why-should-i-use-it)
- [serialVesrsionUID-선언-이유](http://enxxstory.tistory.com/entry/serialVersionUID-선언이유에-대한-포스팅)
- [IntelliJ에서 serialVersionUID생성방법](http://stackoverflow.com/questions/12912287/intellij-idea-generating-serialversionuid)
