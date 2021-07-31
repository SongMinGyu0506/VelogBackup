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
	@GetMapping("/ex02")
	public String ex02(@RequestParam("name") String name, @RequestParam("age") int age) {
		log.info("name: "+name);
		log.info("age: "+age);
		return "ex02";
	}
	@GetMapping("/ex02List")
	public String ex02List(@RequestParam("ids")ArrayList<String> ids) {
		log.info("ids :" + ids);
		return "ex02List";
	}
	@GetMapping("/ex02Array")
	public String ex02Array(@RequestParam("ids") String[] ids) {
		log.info("array ids: " + Arrays.toString(ids));
		return "ex02Array";
	}
	
	@GetMapping("/ex02Bean")
	public String ex02Bean(SampleDTOList list) {
		log.info("list dtos: " + list);
		return "ex02Bean";
	}
	@GetMapping("/ex03")
	public String ex03(TodoDTO todo) {
		log.info("todo: "+ todo);
		return "ex03";
	}
	@GetMapping("/ex04")
	public String ex04(SampleDTO dto, @ModelAttribute("page") int page) {
		log.info("dto: "+dto);
		log.info("page: "+page);
		return "/sample/ex04";
	}
	
	@GetMapping("/ex06")
	public @ResponseBody SampleDTO ex06() {
		log.info("/ex06................");
		SampleDTO dto = new SampleDTO();
		dto.setAge(10);
		dto.setName("홍길동");
		return dto;
	}
```
