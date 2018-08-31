---
layout: post
title: 쿼리 파라미터 값과 percentage
date: 2018-08-31 16:10:00 +0900
categories: http
---

# 쿼리 파라미터 값과 %

## 상황
- Spring MVC로 구현되어있는 웹서버에 HTTP 요청시에 쿼리 파라미터 값 '%'가 들어갈 경우.. 해당 파라미터의 키와 값을 무시한다.
- example
	- request
		- HTTP Method : GET
		- url : `http://localhost:{포트번호}/foo?bar=%`
			- 이때 `bar` 이 필수 파라미터라고 가정하면...
	- message
		- HTTP statusCode : 400 bad request

## 원인
- 스프링에서 쿼리 파라미터를 파싱할때 '%'가 있으면 이를 인코딩이 된 코드로 인식을 하여 파싱을 시작한다. 하지만 utf-8 인코드는 %20 부터 시작한다. %[20미만]의 경우 없는 코드이므로 해당 파라미터 키와 값을 무시한다. 
- 위 예제의 경우 `bar`는 필수 파라미터라고 했으니, 들어온 값이 없기 때문에 400 bad request 메시지를 반환한다.


## 재현
- 코드

	```
package my.controller;
import org.springframework.web.bind.annotation.*;
@RestController
@RequestMapping(path = "/")
public class FooBarController {
        @GetMapping("/foo")
        public String fooBar(@RequestParam("bar") String bar) {
            return bar + " ping pong";
        }
}
	```
- 요청, 응답
	```
	$ curl -XGET http://localhost:10001/foo?bar=%
	{"timestamp":1535698762377,"status":400,"error":"Bad Request","message":"Required String parameter 'bar' is not present","path":"/foo"}%
	```

- 서버 로그

	```
2018-08-31 15:59:22.371  INFO 52680 --- [io-10001-exec-2] org.apache.tomcat.util.http.Parameters   : 
Character decoding failed. Parameter [bar] with value [%] has been ignored. Note that the name and value quoted here may be corrupted due to the failed decoding. 
Use debug level logging to see the original, non-corrupted values.
Note: further occurrences of Parameter errors will be logged at DEBUG level.
	```

## 해결방법
- 파라미터 값을 utf-8로 인코딩 해서 요청하면 된다.
	- `%` -> `%25`
- 코드값은 [https://www.utf8-chartable.de/unicode-utf8-table.pl](https://www.utf8-chartable.de/unicode-utf8-table.pl)을 참고.

## 분석
- `org.apache.tomcat.util.http.Parameters` 로그 메시지는 스프링의 [UnsatisfiedServletRequestParameterException](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/UnsatisfiedServletRequestParameterException.html)의 73번 라인의 메소드인 `getMessage()`에서 나온 메시지이다.
	- `UnsatisfiedServletRequestParameterException.getMessage()`
	
		```
		@Override
		public String getMessage() {
				StringBuilder sb = new StringBuilder("Parameter conditions ");
				int i = 0;
				for (String[] conditions : this.paramConditions) {
					if (i > 0) {
						sb.append(" OR ");
					}
					sb.append("\"");
					sb.append(StringUtils.arrayToDelimitedString(conditions, ", "));
					sb.append("\"");
					i++;
				}
				sb.append(" not met for actual request parameters: ");
				sb.append(requestParameterMapToString(this.actualParams));
				return sb.toString();
			}
		```
	- `ServletRequestBindingException` 클래스를 상속하는 클래스인것으로 보아 요청을 바인딩할때 발생하는 오류임도 확인할 수 있다.

## 결론
- utf-8 중 %로 시작하는 코드는 25부터이다.
- 이걸 기억하는것보다 `GET` 요청 파라미터 값은 반드시 **인코딩** 해야함..!!