## Controller Return Type
* String: jsp 파일경로의 이름 리턴
* void: 호출하는 URL의 동일한 이름의 jsp

### 객체 타입
주로 JSON 데이터를 만들어내는 용도로 사용한다.  
Jackson-databind 라이브러리를 이용
```java
@GetMapping("/ex06")
public @ResponseBody SampleDTO ex06() {
    log.info("/ex06..........");
    SampleDTO dto = new SampleDTO(); //Bean 객체
    dto.setAge(10);
    dto.setName("홍길동");
    return dto
}
```

### ResponseEntity 타입
HTTP Header를 다루는 경우, ResponseEntity를 이용하여 데이터 전달 가능
```java
	@GetMapping("/ex07")
	public ResponseEntity<String> ex07() {
		log.info("/ex07.........................");
		String msg = "{\"name\": \"홍길동\"}";
		HttpHeaders header = new HttpHeaders();
		header.add("Content-Type", "application/json;charset=UTF-8");
		return new ResponseEntity<>(msg,header,HttpStatus.OK);
	}
```

### File Upload 처리
#### commons-fileupload 사용
* dependency
```xml
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.3.3</version>
</dependency>
```
```java
@GetMapping("/exUpload")
public void exUpload() {		
    log.info("/exUpload..................");
}
	
@PostMapping("/exUploadPost")
public void exUploadPost(ArrayList<MultipartFile> files) {
	files.forEach(file -> {
		log.info("------------------------------");
		log.info("name:" + file.getOriginalFilename());
		log.info("size:"+file.getSize());
	});
}
```
```JSP
<%@ page language="java" contentType="text/html; charset=EUC-KR"
    pageEncoding="EUC-KR"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR">
<title>Insert title here</title>
</head>
<body>
<form action="/sample/exUploadPost" method="post" enctype="multiPART/form-data">
	<div><input type='file' name='files'></div>
	<div><input type='file' name='files'></div>
	<div><input type='file' name='files'></div>
	<div><input type='file' name='files'></div>
	<div><input type='file' name='files'></div>
	<div><input type='submit'></div>
</form>
</body>
</html>
```

## Controller 예외처리
SpringMVC에서는 @ExceptionHandler와 @ControllerAdvice를 이용한 처리  
@ResponseEntity를 이용하는 예외 메시지 구성으로 처리 가능하다.

### @ControllerAdvice
AOP(Aspect-Oriented-Programming)를 이용하는 방식,  
즉, 공통적인 관심사를 분리한다. 예를들어 웹에서 가장 공통적인 상황은 무엇이 있을까? 아마 404 not found 같은 잘못된 접근에 대한 처리가 가장 공통적인 관심사중 하나이지 않을까 생각한다.
#### ex)
```java
@ControllerAdvice
@Log4j
public class CommonExceptionAdvice {
	@ExceptionHandler(Exception.class)
	public String except(Exception ex, Model model) {
		log.error("Exception ...." + ex.getMessage());
		model.addAttribute("exception",ex);
		log.error(model);
		return "error_page";
	}
}
```
```JSP
<%@ page language="java" contentType="text/html; charset=EUC-KR"
    pageEncoding="EUC-KR"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR">
<title>Insert title here</title>
</head>
<body>
	<h4><c:out value="${exception.getMessage() }"></c:out></h4>
	<ul>
		<c:forEach items="${exception.getStackTrace() }" var="stack">
			<li><c:out value="${stack }"></c:out></li>
		</c:forEach>
	</ul>
</body>
</html>
```
컨트롤러의 어노테이션에서는 <code>@ControllerAdvice</code> 컨트롤러에서 발생하는 예외상황을 처리하는 컨트롤러? 같은 역할을 해준다. 클래스 내부에서는 
<code>@ExceptionHandler</code>가 적용되어 있는데, 특정 예외 타입을 처리한다는 내용이다. 해당 어노테이션 내부에서는 Exception 클래스가 적용되므로 모든 예외상황에 대해서 처리한다는 내용이다.  

보통 예외처리에 대한 컨트롤러는 다른 패키지를 만들어서 처리하므로, servlet-context.xml에서 컴포넌트 스캔을 추가해줘야 한다.

### 404 Not Found
404 Not Found의 경우도, DispatcherServlet을 이용한 처리를 404 에러 또한 처리되도록 web.xml을 수정한 후, 위의 예외처리처럼 처리하면 된다.

```xml
        ...
		<init-param>
			<param-name>throwExceptionIfNoHandlerFound</param-name>
			<param-value>true</param-value>
		</init-param>
        ...
```
```java
    ...
	@ExceptionHandler(NoHandlerFoundException.class)
	@ResponseStatus(HttpStatus.NOT_FOUND)
	public String handle404(NoHandlerFoundException ex) {
		log.error("404 Not Found....................... " + ex.getMessage());
		return "custom404";
	}
    ...
```
```jsp
<%@ page language="java" contentType="text/html; charset=EUC-KR"
    pageEncoding="EUC-KR"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="EUC-KR">
<title>Insert title here</title>
</head>
<body>
	<h1>해당 URL은 존재하지 않습니다.</h1>
</body>
</html>
```