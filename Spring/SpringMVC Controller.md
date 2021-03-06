## @Controller, @RequestMapping
```java
@Controller
@RequestMapping("/sample/*")
public class SampleController {

}
```
* 해당 소스코드를 살펴보면 <code>@Controller</code> <code>@RequestMapping</code> 어노테이션이 적용되어 있는데, servlet-context.xml 파일에 등록되어 있는 component-scan의 패키지의 경우 해당 클래스는 Spring의 Bean으로 등록이 된다.

#### component-scan?
```xml
<?xml version="1.0" encoding="UTF-8"?>
    ...
	<context:component-scan base-package="org.zerock.controller" />
	<context:component-scan base-package="org.zerock.testing"/>
    ...
</beans:beans>
```
* 해당 내용은 servlet-context.xml 파일에 있는 내용이다. <code>\<context:component-scan base-package="...."></code>의 내용은 해당 태그를 이용하여 지정된 패키지를 스캔하도록 한다.   
즉, 해당 패키지 내의 Spring의 Bean 설정에 해당되는 클래스들을 파악하고 생성, 관리하게 된다.  
  
보통 Spring Framework에서 클래스 선언부에는 <code>@Controller</code> <code>@RequestMapping</code>을 주로 사용한다.

예로 하나의 샘플 컨트롤러를 만들어서 실행시킨다면
```java
package org.zerock.controller;

import ...

@Controller
@RequestMapping("/sample...*")
public class SampleController {
    @RequestMapping("")
    public void basic() {
        log.info("basic.................");
    }
    ..
}
```
Spring 프로젝트를 실행시킨다면 해당 정보가 출력된다.
>LINE: 547 INFO : org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping - Mapped "{[/sample/*]}" onto public void org.zerock.controller.SampleController.basic()

해당 경로로 접근이 가능하도록 매핑되어있다는 것을 알려준다.

## @RequestMapping의 변화 및 파라미터 수집 방법

### @RequestMapping의 변화
```java
	@RequestMapping(value="/basic",method= {RequestMethod.GET,RequestMethod.POST})
	public void basicGet() {
		 log.info("basic get........................");
	}
    @GetMapping("/basicOnlyGet")
	public void basicGet2() {
		log.info("basic get only get.............................");
	}
```
위의 두개의 매핑은 같은 동작을 한다. Spring 4.3 버전 이전에는 위의 RequestMapping을 사용했으나, HTTP의 Method 속성에 맞춰 개별적으로 Mapping을 사용할 수 있도록 되어있다.  

간편하나, 기능에 대한 제약이 있다.

### Controller의 파라미터 수집 방법
```java
package org.zerock.domain;

import lombok.Data;

@Data
public class SampleDTO {
	private String name;
	private int age;
}
```
<center><strong>파라미터 수집 방법을 위한 클래스(DTO)</strong></center>

```java
@GetMapping("/ex01")
	public String ex01(SampleDTO dto) {
		log.info(""+dto);
		return "ex01";	
	}
```
* 해당 파라미터로 들어가있는 SampleDTO는 <code>@Data</code> 어노테이션이 들어가있어서 Getter/Setter가 자동으로 설정되어있다.  
Get 방식으로 작동하므로 URL 뒤에 ?name=OOO&age=1234 이런 방식으로 입력하면 자동으로 Setter가 작동되어 처리된다.
  
```java
	@GetMapping("/ex02")
	public String ex02(@RequestParam("name") String name, @RequestParam("age") int age) {
		log.info("name: "+name);
		log.info("age: "+age);
		return "ex02";
	}
```
* 파라미터를 클래스가 아닌 기본 자료형을 이용하는 방법또한 있다.
* @RequestParam() 어노테이션을 이용하여 사용
```java
	@GetMapping("/ex02List")
	public String ex02List(@RequestParam("ids")ArrayList<String> ids) {
		log.info("ids :" + ids);
		return "ex02List";
	}
```
```java
	@GetMapping("/ex02Array")
	public String ex02Array(@RequestParam("ids") String[] ids) {
		log.info("array ids: " + Arrays.toString(ids));
		return "ex02Array";
	}
```
* 리스트와 배열을 이용한 방법
* /sample/ex02(Array & List)?ids=111&ids=222 이런 방식으로 사용
```java
	@GetMapping("/ex02Bean")
	public String ex02Bean(SampleDTOList list) {
		log.info("list dtos: " + list);
		return "ex02Bean";
	}
```
* 객체리스트 이용

#### @InitBinder
특정 입력 받은 문자열을 자바에서 사용하는 특정 데이터 타입으로 변환하는 작업
```java
package org.zerock.domain;

import java.util.Date;

import lombok.Data;

@Data
public class TodoDTO {
	private String title;
	private Date dueDate;
}
```
```java
    @InitBinder
	public void initBinder(WebDataBinder binder) {
		SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
		binder.registerCustomEditor(java.util.Date.class, new CustomDateEditor(dateFormat, false));
	}
```
```java
	@GetMapping("/ex03")
	public String ex03(TodoDTO todo) {
		log.info("todo: "+ todo);
		return "ex03";
	}
```
dueDate=yyyy-mm-dd 방식으로 입력 받는다면 java.util.Date 형식에 맞춰서 저장된다.

#### Model 데이터 전달자
컨트롤러에서 값을 입력 받았으면, 입력 받은 값을 출력하는 방법 또한 있다.  
Spring에서는 기본적으로 Beans의 규칙에 맞는 객체들은 별다른 처리 없이 출력할 수 있도록 되어있다. 하지만 기본 자료형의 경우는 선언하더라도 기본적으로 전달이 되지 않는다. 이때 @ModelAttribute 어노테이션을 이용하여 전달할 수 있다.
```java
	@GetMapping("/ex04")
	public String ex04(SampleDTO dto, @ModelAttribute("page") int page) {
		log.info("dto: "+dto);
		log.info("page: "+page);
		return "/sample/ex04"; //views/sample/ex04.jsp
	}
```
