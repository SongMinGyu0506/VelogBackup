# jUnit Security Context 설정
Springboot 등에서 테스트코드를 작성할 때, 로그인 이후의 기능을 테스트 해야하는 경우가 상당히 많다. 이러한 상황에서는 회원가입 - 로그인 기능을 수행 후 테스트해야할 코드들을 작성해야하는데, 테스트 코드에 반복이 많아지고, 코드가 복잡해질 수 있다.

이 상황을 해결하기 위해 ```WithSecurityContextFactory```를 이용하여 문제를 해결할 수 있다.

## 기존 로그인 구성
```WithSecurityContextFactory```를 이용한 프로젝트에서 사용하는 로그인의 경우 JWT 로그인 방식을 이용하여 JWT 토큰을 확인하고 확인이 가능하다면 Security Context를 생성하여 Spring Core 내부로 진입하는 방식을 사용하고 있다.
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter{

    private final TokenProvider tokenProvider;

    @Autowired
    public JwtAuthenticationFilter(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    public String parseBearerToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken)&&bearerToken.startsWith("Bearer")) {
            return bearerToken.substring(7);
        }
        return null;
    }

    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            String token = parseBearerToken(request);
            if (token != null && !token.equalsIgnoreCase("null")) {
                int userId = tokenProvider.validateAndGetUserId(token);
                AbstractAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userId,null, AuthorityUtils.NO_AUTHORITIES);
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContext securityContext = SecurityContextHolder.createEmptyContext();
                securityContext.setAuthentication(authentication);
                SecurityContextHolder.setContext(securityContext);
            }

        } catch (Exception e) {
            logger.error("Could not set user authentication in security context",e);
        }
        filterChain.doFilter(request,response);
    }
}
```
if문 내부 구조가 토큰 통과가 된다면 Security Context를 생성하고 있다.

## WithSecurityContextFactory 구현
### 1. Annotation
우선 ```WithSecurityContextFactory```를 구현하기 위해서는 해당 기능을 사용할 Annotation이 필요하다.
```java
package com.webkit640.backend.controller;

import org.springframework.security.test.context.support.WithSecurityContext;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithAccountSecurityContextFactory.class)
public @interface WithAccount {
    String value();
}
```
### 2. Implementation
해당 어노테이션이 있다면 어떤 동작을 수행할 것인지 지정한다.
```java
public class WithAccountSecurityContextFactory implements WithSecurityContextFactory<WithAccount> {

    @Autowired
    MemberService memberService;
    @Autowired
    TokenProvider tokenProvider;


    @Override
    @Commit
    public SecurityContext createSecurityContext(WithAccount annotation) {
        Member member = Member.builder()
                .applicant(null)
                .boards(null)
                .counsels(null)
                .email(annotation.value())
                .memberBelong("Belong")
                .memberType("Type")
                .name("Name")
                .isAdmin(true)
                .password("1234")
                .build();

        memberService.create(member);

        int userId = tokenProvider.validateAndGetUserId(tokenProvider.create(member));

        AbstractAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userId,null, AuthorityUtils.NO_AUTHORITIES);
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        return context;
    }
}
```
해당 코드는 WithSecurityContextFactory<(Annotation)> 인터페이스를 받아 구현하게 된다.
createSecurityContext 메소드 내부의 경우 테스트용 사용자 계정을 만들고, JWT 토큰 발급
그리고 Security Context를 생성하고 Security Context를 반환하고 테스트코드를 수행하게 된다.
### 3. 테스트코드 예시
```java
@Test
@WithAccount("test@test.com")
void viewMember() throws Exception {
    signup();
   	mvc.perform(MockMvcRequestBuilders.get("/auth/member/2")).andExpect(status().isOk()).andDo(print());
}
```
해당 테스트코드는 특정 회원을 검색하는 테스트코드로 기능의 사용대상은 관리자만 사용가능하게끔 구현했다.
하지만 해당 기능의 사용자에 대한 소스코드는 ```viewMember()``` 메소드 내에서는 찾을 수 없고 ```@WithAccount(~~~)```에서 처리를 수행하게 된다.

## 좀 더 찾아보기
### 1. WithSecurityContext & WithSecurityContextFactory
>We have seen that @WithMockUser is an excellent choice if we are not using a custom Authentication principal. Next we discovered that @WithUserDetails would allow us to use a custom UserDetailsService to create our Authentication principal but required the user to exist. We will now see an option that allows the most flexibility.
We can create our own annotation that uses the @WithSecurityContext to create any SecurityContext we want.

스프링 부트 공식 문서에서는 위와 같은 언급이 되어있다. ```@WithMockUser```의 경우 커스텀 인증 주체를 사용하지 않는다면 뛰어난 선택이 될 수 있지만 그렇지 않은 경우 유연한 사용이 어렵기 때문에 ```@WithSecurityContext```를 이용하여 유연한 사용이 가능하도록 한다.
따라서 ```@WithSecurityContext```를 사용하기 위해 ```WithSecurityContextFactory```에서 구현 후 사용하는 방식을 채택하고 있다.

[원문 및 구현 예제코드]
https://docs.spring.io/spring-security/site/docs/4.2.x/reference/html/test-method.html#test-method-withsecuritycontext

### 2. AbstractAuthenticationToken
인증 토큰을 나타내는 추상 클래스, 여기서 사용되는 인증 토큰은 사용자 인증을 포함한 인증 프로세스에서 지속적으로 사용된다.

AbstractAuthenticationToken의 내부 코드는 아래와 같다
```java
public abstract class AbstractAuthenticationToken implements Authentication, CredentialsContainer {
	private final Collection<GrantedAuthority> authorities;
	private Object details;
	private boolean authenticated = false;
    
    public AbstractAuthenticationToken(Collection<? extends GrantedAuthority> authorities) {...}
    @Override
	public Collection<GrantedAuthority> getAuthorities() {...}
    @Override
	public String getName() {...}
    @Override
	public boolean isAuthenticated() {...}
	@Override
	public void setAuthenticated(boolean authenticated) {...}
	@Override
	public Object getDetails() {...}
	public void setDetails(Object details) {...}
	@Override
	public void eraseCredentials() {...}
	private void eraseSecret(Object secret) {...}
	@Override
	public boolean equals(Object obj) {...}
	@Override
	public int hashCode() {...}
	@Override
	public String toString() {...}
}
```
Authentication, CredentialsContainer 인터페이스를 상속받는 것을 볼 수 있다.
기본적으로 추상클래스이므로, 직접적인 인스턴스화는 불가능하지만 해당 클래스를 이용한 다른 구현 인스턴스를 통해 사용할 수 있다.

### 3. UsernamePasswordAuthenticationToken
Spring Security에서 사용자의 이름과 비밀번호를 기반으로 한 인증을 나타내는 구체적인 AbstractAuthenticationToken의 서브 클래스중 하나
이름과 같이 사용자의 인증정보를 나타내는데 사용된다. 사용자의 이름과 비밀번호를 수집하고 인증을 수행

### 4. SecurityContext & SecurityContextHolder
#### SecurityContext
Spring Security에서 사용자의 보안 정보를 저장하고 관리하는 인터페이스, 보안 관련 작업에서 사용자에 대한 정보를 액세스하거나 수정하는데 사용
```setAuthentication()```, ```getAuthentication()```을 사용하여 ```Authentication``` 객체를 관리할 수 있다.
#### SecurityContextHolder
```SecurityContext```는 ```SecurityContextHolder```를 통해 관리 할 수 있다.
주요 메소드로는 아래와 같다.
* getContext(): 현재 스레드의 SecurityContext를 반환합니다.
* setContext(context): 현재 스레드에 SecurityContext를 설정합니다.
* clearContext(): 현재 스레드의 SecurityContext를 제거합니다.
