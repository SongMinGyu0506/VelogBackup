
![](https://velog.velcdn.com/images/alsrb5606/post/8185b579-9037-4450-9a13-ac1afe064b1f/image.png)

## 접근
스프링부트 프로젝트를 진행하던 도중 테스트코드를 작성하는데 개발한 다양한 기능들을 테스트하기 위해서는 로그인을 우선적으로 수행해야 기능을 사용 할 수 있다.
회원가입, 로그인 기능을 테스트 할 때 마다 작성하는 것은 매우 비효율적이라고 생각하여 방법을 찾던 도중 Security Context를 테스트 코드에 적용하는 방법을 찾게 되었다.

## WithSecurityContextFactory
해당 인터페이스를 사용하고, 구현하여 테스트 코드 작성 중에 Security Context를 추가한 테스트를 구현 할 수 있다.
WithSecurityContextFactory 인터페이스의 경우 특정한 Spring Security에서 사용하는 security context중 특정한 security context를 테스트 코드 중에 적용하고 실행하는 기능을 수행한다.

즉, 해당 인터페이스 구현을 통해 테스트용 계정을 생성하고 로그인하여 security context에 인증을 추가하는 방법을 이용한다. 프로젝트에서 개발한 로그인 기능은 JWT 인증을 통하여 로그인을 수행한다.
테스트 코드에서 토큰을 발급받고 인증하는 기능을 수행한 후 받는 결과값을 security context에 저장하여 반환하여 테스트를 수행한다.

### annotation
WithSecurityContextFactory의 경우 해당 security context를 적용시킬 범위가 필요한데, 이 범위를 지정하기 위해서 커스텀 어노테이션을 이용한다.
```java
import org.springframework.security.test.context.support.WithSecurityContext;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithAccountSecurityContextFactory.class)
public @interface WithAccount {
    String value();
}
```
---

### implementation
해당 어노테이션을 사용하면 적용될 메서드를 정의한다.
```java
...
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.AuthorityUtils;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.test.context.support.WithSecurityContextFactory;
import org.springframework.test.annotation.Commit;

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
여기서 사용하는 MemberService, TokenProvider, Member 객체들은 각각 회원관리 서비스, JWT 토큰 관리기능, 회원 엔티티이다.

테스트용 계정을 생성하고 해당 계정의 토큰을 생성, 검증후 나온 결과를 context에 추가한다.

### 테스트 코드 적용
```java
    @Test
    @WithAccount("test@test.com")
    @DisplayName("지원서 ZIP 다운로드 성공 테스트")
    void test2() {
        try {
            MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get("/a/f")
            ).andExpect(status().isOk()).andReturn();
            assertAll(
                    ()->assertThat(result.getResponse().getContentType()).isEqualTo(MediaType.APPLICATION_OCTET_STREAM_VALUE),
                    ()->assertThat(result.getResponse().getHeader(HttpHeaders.CONTENT_DISPOSITION)).contains("attachment")
            );
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
원래 테스트코드의 경우 로그인하여 받은 JWT 토큰을 헤더에 첨부시켜야 하지만 이미 Security Context에 내용을 등록하고 진행하기 때문에 별도의 토큰 없이 테스트를 수행 할 수 있다.

### 참고
https://velog.io/@jungnoeun/%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C%EB%A5%BC-%EC%9E%91%EC%84%B1%ED%95%98%EB%A9%B4%EC%84%9C
