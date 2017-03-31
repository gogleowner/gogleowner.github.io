---
layout: post
title: "Java8 in Action Part III 정리"
date: 2017-04-01 06:15:00 +0900
categories: java
---

# Java8 in Action Part III 정리
## Part III 리팩토링,테스팅,디버깅
### 목표
- 람다 표현식을 활용하여 코드의 가독성을 높일 수 있다.
- 람다 표현식을 활용하여 객체지향 디자인 패턴 간소화가 가능하다.
- 코드 테스트 및 디버깅 방법

### 1. 리팩토링
- 익명클래스 -> 람다 표현식으로 변경

	```
	// 익명클래스
	Runnable runnableAnnoymousClass = new Runnable() {
            @Override
            public void run() {
                System.out.println("some job");
            }
        };
// 람다 표현식
Runnable runnableLambda = () -> System.out.println("some job");
	```
	
	- 익명클래스와 람다표현식의 차이
		- this, super
			- 익명클래스에서의 this : 클래스 자신
			- 람다표현식 this : 람다를 감싸는 클래스를 가리킴
		- 변수 제어
			- 익명클래스는 메소드 밖에서 선언한 변수 명으로 동일하게 선언을 해도 사용이 가능하나, 람다 표현식에서는 컴파일에러가 발생한다.

			```
			int a = 1;
        Runnable runnableAnnoymousClass = new Runnable() {
            @Override
            public void run() {
                int a = 2; // 사용 가능
                System.out.println("some job");
            }
        };
        Runnable runnableLambda = () -> {
            int a = 3; // 컴파일 오류 발생
            System.out.println("some job");
        };
			```

	- 익명클래스와 람다표현식의 공통점
		- 블록 밖의 변수 조작이 불가능하다.
	
	- 시그니처가 같은 함수형 인터페이스를 가지고 있는 경우
		- 명시적 형변환을 활용하여 해결한다.
		- 이는 대부분의 IDE 에서 해결해준다.

- 람다 표현식 -> 메소드 레퍼런스로 변경
	- 메소드 레퍼런스 : 메소드를 참조해서 매개 변수의 정보 및 리턴 타입을 알아내어, 람다식에서 불필요한 매개 변수를 제거하는 것이 목적
	- 메소드 레퍼런스의 종류

	| Kind    |Example
	|-----------------------------------------------------------------------------|--------------------------------------
	| Reference to a static method                                                | ContainingClass::staticMethodName
	| Reference to an instance method of a particular object                      | containingObject::instanceMethodName
	| Reference to an instance method of an arbitrary object of a particular type | ContainingType::methodName
	| Reference to an instance method of an arbitrary object of a particular type | ClassName::new

- 코드로 표현	
	- 익명 클래스

		```
	    Comparator<Apple> appleComparatorByAnnomousClass = new Comparator<Apple>() {
	        @Override
	        public int compare(Apple o1, Apple o2) {
	            return o1.getWeight().compareTo(o2.getWeight());
	        }
	    };
		```
    
	- 람다 표현식
	
		```
	    Comparator<Apple> appleComparatorByLambda = (o1, o2) -> o1.getWeight().compareTo(o2.getWeight());
	    ```
    
    - 메소드 레퍼런스

	    ```
	    Comparator<Apple> appleComparatorByMethodReference = Comparator.comparing(Apple::getWeight);	
		```

### 2. 객체지향 디자인 패턴 리팩토링
- 디자인패턴에 적절하게 람다표현식을 더하면 코드가 간결해질 것

#### Strategy Pattern
- 예를들어 검증 로직에 대한 인터페이스가 존재하고 여러 검증 구현체가 있다고 했을때, Predicate<T> 함수형 인터페이스를 직접 넘기도록 하면 별도의 클래스 생성 없이 구현이 가능해진다.

#### Template Method Pattern
- 템플릿에 대한 여러 구현체를 구현했는데 어떤 구현체는 별도의 작업이 필요한 경우가 있을 것이다. 그 구현체를 위해서 탬플릿을 변경한다면 해당 템플릿에 대한 구현체는 그에 대한 작업을 해줘야할 것이다. 이 경우 Consumer<T> 함수형 인터페이스를 활용하여 해당 작업을 구현체에 맡겨버리면 다양한 동작을 추가할 수 있다.

#### Factory Mathod Pattern
- switch case 로 되어있던 팩토리 메소드를 static Map을 선언하여  key,value의 형태로 저장한다. 예를들어 switch case 의 키를 key로 switch case에서 리턴하고 있는 값을 value 로 지정하는 것이다. 
- case 가 많아지면 오히려 가독성에 안좋을 수도 있을 듯하다.


### 3. 람다 테스팅, 디버깅
- 람다 표현식의 테스트는 람다 자체를 테스트하는 경우가 애매한 경우이긴 하다. 복잡한 람다표현식이라면 해당 변수를 static으로 빼던지 해서 테스트하기 용이하도록 바꿔야한다. 그렇지 않은 경우에는 테스트만으로는 어렵고, 디버깅을 통해서 동작이 올바로 동작하는지 확인하자.

#### 보이는 람다표현식
- static 으로 Comparator<T>가 선언되어있다면 Comparator가 비교할 동작으로 받는 인스턴스가 있을 것이다.
- 해당 경우에는 테스트메소드에서 인스턴스를 생성하여 동작을 확인한다.

	```
	public static final Comparator<Apple> appleComparatorByMethodReference = Comparator.comparing(Apple::getWeight);
@Test
public void lambdaTesting() throws Exception {
    Apple a1 = new Apple(10);
    Apple a2 = new Apple(20);
    int compareResult = appleComparatorByMethodReference.compare(a1, a2);
    assertThat(compareResult, is(equals(-1)));
}
	```	
#### 디버깅
- 스택트레이스
	- {클래스명}$$Lambda$3
		- 람다 표현식 내부에서 발생한 익셉션임을 확인할 수 있다.
		- 람다 표현식은 이름이 없으므로 컴파일러가 람다를 참조하는 이름을 만들어낸 것이다.
	- 간혹 스택트레이스에서도 unknown 으로 나와서 람다 표현식의 어느부분인지 찾기 어려운 경우가 있는데, 이는 미래의 자바 컴파일러가 개선해야할 부분이다.
- 정보로깅
	- map, filter, limit 과정을 통해 데이터가 어떻게 바뀌었는지 확인해야할 경우가 있다. forEach는 종단연산이기 때문에 모든 스트림을 소비해버린다. 연산 후의 정보를 확인하고 싶은 경우 peek를 활용한다.
	- peek() 스트림 연산 활용

		```
		Stream.of("one", "two", "three", "four")
              .filter(e -> e.length() > 3)
              .peek(e -> System.out.println("Filtered value: " + e))
              .map(String::toUpperCase)
              .peek(e -> System.out.println("Mapped value: " + e))
              .collect(Collectors.toList());
		```

