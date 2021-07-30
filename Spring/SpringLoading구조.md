## 프로젝트의 로딩 구조
* 스프링 프로젝트의 구동시 관여하는 XML은 web, root-context, servlet-context 파일이다.  
이 파일 중 web.xml은 Tomcat구동과 관련된 설정, 나머지 파일은 스프링과 관련된 설정으로 되어있다.

###  web.xml
```XML
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee https://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">

	<!-- The definition of the Root Spring Container shared by all Servlets and Filters -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/spring/root-context.xml</param-value>
	</context-param>
	
	<!-- Creates the Spring Container shared by all Servlets and Filters -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- Processes application requests -->
	<servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
		
	<servlet-mapping>
		<servlet-name>appServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

</web-app>
```
<center><strong> Eclipse로 SpringMVC 프로젝트 생성시 자동 생성되는 web.xml</strong></center>
  
* 프로젝트의 구동 시작시 해당 xml 파일에서 시작한다.  
해당 파일의 상단에 가장 먼저 구동되는 Context Listener가 등록되어 있다.  

* context-param에는 root-context.xml의 경로, listener에는 SpringMVC의 ContextLoaderListener가 등록되어 있다. 해당 ContextLoaderListener는 Web Application 구동 시 같이 동작, 그러므로 가장 먼저 실행된다.  

* root-context.xml이 처리된다면 root-context.xml에 있는 객체(Bean) 설정들이 동작하게 된다.  
그리고 Bean들이 Spring의 영역에 생성되고, 객체들 간의 의존성(Dependency)이 처리된다.  

* 이후 SpringMVC에서 사용하는 DispatcherServlet이라는 서블릿과 관련된 설정들이 동작하게 된다.  
해당 코드에서는 org.springframework.web.servlet.DispatcherServlet 클래스로 설정되어 있는데, SpringMVC 구조에서 가장 핵심적인 역할을 하는 클래스로, 내부적으로 웹 관련 처리의 준비작업을 진행하는데 이때 사용하는 파일이 servlet-context.xml 파일이다.

* DispatcherServlet에서 XmlWebApplicationContext를 이용해서 servlet-context.xml을 로딩하고 해석한다. 이 과정에서 등록된 Bean들은 기존 Bean들과 같이 연동되게 된다.

## SpringMVC의 기본 구조
![](https://blog.kakaocdn.net/dn/HVTFH/btqCqNXb9ba/GyuCRrJfVEet9nxcCpzbJk/img.png)
1. 사용자 Request는 Front-Controller인 Dispatcher Servlet을 통해서 처리
```XML
	<servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
		
	<servlet-mapping>
		<servlet-name>appServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
```
해당 코드를 보면 모든 Request를 DispatcherServlet이 받도록 처리되어있다.  

2. ,3) HandlerMapping은 Request의 인터페이스를 구현한 여러 객체들 중 RequestMappingHandlerMapping 같은 경우는 @RequestMapping 어노테이션이 적용된 것을 기준으로 판단한다. 맞는 컨트롤러를 찾았다면 HandlerAdapter를 이용하여 컨트롤러 동작

4) 실제 개발자의 Request 처리 로직, 이때 View(JSP,html...)에 전달되는 데이터는 주로 Model이라는 객체에 담아서 전달, 컨트롤러의 반환 타입 처리는 ViewResolver를 이용

5) ViewResolver는 Controller가 반환한 결과를 어떤 View를 통해서 처리하는 것이 좋을지 해석한다. 가장 많이 사용되는 설정은 servlet-context.xml의 InternalResourceViewResolver를 사용 
```XML
<beans:bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<beans:property name="prefix" value="/WEB-INF/views/" />
		<beans:property name="suffix" value=".jsp" />
	</beans:bean>
	
	<context:component-scan base-package="org.zerock.controller" />
```
6) ,7) View는 실제로 응답 보내야 하는 데이터를 Jsp 등을 이용해서 생성하는 역할 만들어진 응답은 DispatcherServlet을 통해서 전송

## 요약

### Spring 프로젝트의 로딩 구조
1. Spring Framework의 프로젝트 구동 시 첫 수행되는 작업들이 web.xml에 기술되어있다.
    * 가장 먼저 ContextLoaderListener가 구동
    * 프로젝트 구동 시 WebApplicationContext라는 프로젝트에서 사용되는 메모리가 할당된다.
2. 그 다음 root-context.xml 파일의 설정들이 동작하게 된다.
    * 해당 파일 내에 정의된 Bean들이 WebApplicationContext 안에 생성, 의존성 처리
3. DispatcherServlet, 서블릿 관련 설정이 동작, 이 과정에서 등록된 Bean들은 기존에 만들어진 Bean들과 같이 연동

### SpringMVC의 기본 구조
1. 사용자 Request는 Front-Controller인 Dispatcher Servlet을 통해서 처리
2. ,3) HandlerMapping - RequestMappingHandlerMapping @RequestMapping 어노테이션 적용된 것 기준으로 찾음 --> 완료시 HandlerAdapter 이용하여 컨트롤러 조작
4) 실제 개발자의 Request 처리 로직
5) ViewResolver는 Controller가 반환한 결과를 어떤 View를 통해서 처리하는 것이 좋을지 해석