## Spring Security Config class 구조
```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled=true)
@EnableWebSecurity
public class WebSecurityConfig {
	@Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    	http....
    	...
        http.exceptionHandling()...
        http.addFilterAfter(...)...
        return http.build();
    }
```
기존에는 ```WebScurityConfigurerAdapter```를 상속받아 오버라이드 형태로 구성했으나, ```WebSecurityConfigurerAdapter```가  deprecated 된 이후로 시큐리티 설정은 해당 방식을 이용하고 있다.
SecurityFilterChain 메서드 내부의 설정은 위의 예제보다 더 많으나 가장 간단하게 설명한다.
## http config chain
### 예제
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
	http.cors()
    	.and()
        .csrf()
        .disable()
        .httpBasic()
        .disable()
        .sessionManagement()
 		.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .and()
        .authorizeRequests()
        .antMatchers("/","/auth/**","/h2-console/**").permitAll()
        .anyRequest()
        .authenticated();
	return http.build();
}
```
스프링 부트 프로젝트를 진행 할 때 사용했던 시큐리티 설정의 일부분으로, 해당 방식처럼 이용하여 시큐리티 설정을 진행 할 수 있다.

### 설정 설명

* and() : 체이닝 방식을 사용하여 진행 할 때 사용
* disable(): 사용 안함 설정
* cors(): __CorsFilter__ 추가, 추가하지 않을 시 corsConfigurationSource 정의
* csrf(): csrf보호 활성화 호출 후 enable, disable로 설정 https://ko.wikipedia.org/wiki/%EC%82%AC%EC%9D%B4%ED%8A%B8_%EA%B0%84_%EC%9A%94%EC%B2%AD_%EC%9C%84%EC%A1%B0
* sessionManagement: 한번에 단일 사용자 인스턴스만 인증되도록 하는 방법
- sessionCreationPolicy: 세션 정책
    - .ALWAYS: 항상 세션을 생성
    - .IF_REQUIRED: 필요시 생성(기본)
    - .NEVER: 생성하지 않지만 기존 존재시 사용
    - .STATELESS: 생성하지도 않고, 기존 것도 사용하지 않음 (JWT 토큰방식이용)
- authorizeRequests(): RequestMatcher를 기반으로 엑세스 제한 기능
- antMatcher: HttpSecurity에서 제공된 ant 패턴과 일치할 때만 호출 가능
- anyRequest ~ authenticated : 어떠한 요청도 인가받아야한다.


## http.exceptionHandling
```java
http.exceptionHandling()
                .authenticationEntryPoint((request,response,e)-> {
                    Map<String,Object> data = new HashMap<String,Object>();
                    data.put("status", HttpServletResponse.SC_FORBIDDEN);
                    data.put("message",e.getMessage());

                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);

                    objectMapper.writeValue(response.getOutputStream(),data);
                });
```
스프링 시큐리티 필터 접근에 예외를 처리하는 부분으로 ExceptionTranslationFilter에 구현된 인터페이스이다. Spring Security의 첫 진입점 필터로 사용자 인증을 확인하고 예외를 처리하는 기능을 수행한다.

## http.addFilterAfter
```java
http.addFilterAfter(jwtAuthenticationFilter,CorsFilter.class);
```
필터를 추가하는 메서드로 해당 예제에서는 앞의 설정에서 생성한 CorsFilter __앞에__ 임의로 생성한 JWT 토큰 인증 필터를 추가하였다. addFilterAfter를 사용하면 대상 필터 앞에 추가된다.
