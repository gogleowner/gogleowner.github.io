---
layout: post
title: "스프링 배치 프로젝트를 올리다."
date: 2017-03-31 23:15:00 +0900
categories: java
---

# 스프링 배치 프로젝트를 올리다.

## [csvToDmlGenerator](https://github.com/gogleowner/dml_query_generator)

### 개발 동기

- 분리망 의 데이터를 csv 로 받아서 개발망으로(수작업 ㅠ) [sequel pro](https://www.sequelpro.com) 로 옮기던 와중.. csv 데이터 매핑이 이상하게 되는 게 있었다. 도대체 뭐지 ?! 알고보니 컬럼값이 json 형식었다는!
- 이건 어떻게 해야할까. 구분자를 `|` 로 바꿔서 할까? 앗, 그런데 컬럼내의 값이 | 로 구분되어있는 데이터도 있었다. json 형식은 다른 구분자로 하면 됬지만, 다른 여러 상황이 발생할 수 있는 것이었다.
- 그래서 만들었다. **구분자, 테이블명(파일명)** 만 유저가 정해놓으면 알아서 insert dml 구문을 만들어주는 프로그램을..

### 막상 만들고 보니 지금 이 프로그램은 한계가 있다. 
1. 테이블, 구분자를 변경할 때마다 작업을 해야한다.
	- [`FileInfoContainable`](https://github.com/gogleowner/dml_query_generator/blob/master/src/main/java/io/github/gogleowner/container/FileInfoContainable.java) 인터페이스의 구현체를 만들어서
	- [`CsvToDmlJobConfiguration`](https://github.com/gogleowner/dml_query_generator/blob/master/src/main/java/io/github/gogleowner/configuration/CsvToDmlGenerateJobConfiguration.java#L68) 에 구현체를 리턴해줘야한다는 것.
	2. 그래서, [`JobParameter`](http://docs.spring.io/spring-batch/apidocs/org/springframework/batch/core/JobParameters.html) 의 형식으로 넘기는 작업이 필요하다.
		- 자바 특성상 컴파일 해서 실행해야하는 점은 어쩔 수없지만 파라미터 형태로 한다면 1번항목은 극복할 수 있을 듯 싶다.
		- 파라미터 : delimeter=, csvFilePath=sample_data tableName=table_name 요런 형태로!

## 푸념

- 공부, 시험삼아 만들어본 것들을 정리해서 github 에 올려보고자 한다.
- 이정도 만들었으면 됐지~ 라고 생각했던 프로젝트를 다시 열어서 README.md 를 작성하여 올리려고 보니.. 수정할게 하나씩 더 생겨난다.
- 결국 요프로그램도 수정해서 올린건데 아직도 부족한 점이 많다. 조만간 수정해야지!