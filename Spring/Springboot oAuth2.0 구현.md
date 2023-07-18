# Springboot oAuth 로그인 구현
Springboot로 백엔드 서버를 구현하던 중 카카오 oAuth를 구현하는데 어려움을 겪어 해결 방법에 대해 기록

## 1.동작과정
기본적으로 Kakao Developer에서 제공하는 동작과정은 다음과 같습니다.
![](https://velog.velcdn.com/images/alsrb5606/post/09ac0406-3783-4120-8d55-6cbce7fc4ccc/image.png)

하지만 해당 과정을 그대로 수행하여 UI 부분에서 카카오 로그인을 수행하면 백엔드 서버 부분에서 인가 코드 발급부터 사용자 로그인, JWT 토큰 발급까지 전체적으로 수행을 진행하였으나, 403 에러가 발생하며 로그인 과정을 진행하는데 실패했습니다.

다양한 방법을 시도했으나, 팀 프로젝트 회의에서 kakao developer에서 필요로하는 redirect_uri가 일치하지 않아 에러가 발생 할 가능성이 높다고 판단하여 oAuth 접근 순서를 바꿔서 진행해보기로 결정했습니다.

![](https://velog.velcdn.com/images/alsrb5606/post/3fd18ffa-26fe-4015-840c-61590d596746/image.png)

## 2.인가코드 처리
### Kakao API
인가코드에 대한 요청은 프론트엔드 부분에서 카카오 서버로 바로 요청하여 반환된 인가코드를 지정한 redirect_uri로 통해 추가 작업을 진행하도록 계획했습니다.

카카오에서 제공하는 인가코드 요청 API는 다음과 같습니다.
```
GET /oauth/authorize?client_id=${REST_API_KEY}&redirect_uri=${REDIRECT_URI}&response_type=code HTTP/1.1
Host: kauth.kakao.com
```
${REST_API_KEY}부분에는 Kakao Developer에서 제공하는 REST API 키를 입력하고, ${REDIRECT_URI}부분에는 인가코드를 반환하고 작업이 끝난 후 이동할 위치를 지정합니다.
인가코드 요청에 대한 반환 데이터는 다음과 같습니다.
![](https://velog.velcdn.com/images/alsrb5606/post/027617b7-2261-40bc-850e-2f6d237e3569/image.png)
'code' 부분이 인가코드 부분으로 해당부분을 이용하여 추후 과정을 진행합니다.
### Front-end Part
```html
<a href="https://kauth.kakao.com/oauth/authorize?client_id=${rest_api_key}&redirect_uri=${redirect_uri}&response_type=code">
                        <img className="kakao-btn" src="/images/kakao_login.png"/></a>
```
프로젝트 소스코드 중 소셜 로그인에 대한 프론트엔드 소스코드입니다. 발급받은 키와 반환받을 백엔드 서버의 주소는 현재 가려놨지만 해당 부분에 입력하여 사용 중에 있습니다.

### Back-end Part
백엔드 부분에서는 인가 코드 반환값의 이름이 'code'로 지정되어있기 때문에 요청 파라미터를 code로 받는 컨트롤러를 구성해준 후 백엔드 부분에서 필요한 다음 단계를 진행합니다.
![](https://velog.velcdn.com/images/alsrb5606/post/ddb7f526-6ce2-4588-bf07-64f2b1d87509/image.png)

## 3.계정 엑세스 토큰 처리
다음 과정으로는 계정에 대한 엑세스 토큰을 발급받고, 발급받은 엑세스 토큰을 이용하여 카카오 유저 데이터를 요청하여 기존 회원인지, 신규회원인지 구분하는 작업을 실시하게 됩니다.
백엔드 서버 부분에서는 2번에서 발급받은 인가 코드를 이용하기 때문에
![](https://velog.velcdn.com/images/alsrb5606/post/ddb7f526-6ce2-4588-bf07-64f2b1d87509/image.png)
해당 컨트롤러 내부에서 3번 이후 작업을 수행하게 됩니다.

인가 코드를 성공적으로 처리했다면 발급받은 인가코드를 이용해 계정 엑세스 토큰을 발급받아야합니다.
제가 진행한 프로젝트의 경우 계정 엑세스 토큰 처리를 백엔드 서버에서 진행했습니다.

Kakao Developer에서 제공하는 계정 엑세스 토큰 요청 API는 다음과 같습니다.
```
POST /oauth/token HTTP/1.1
Host: kauth.kakao.com
Content-type: application/x-www-form-urlencoded;charset=utf-8
```
![](https://velog.velcdn.com/images/alsrb5606/post/6806f781-5602-482c-9f8b-1b4d11c6e68d/image.png)

백엔드 서버에서 해당 양식과 파라미터를 맞추어 카카오 서버에 전송한다면 계정 엑세스 토큰이 발급됩니다. 
엑세스 토큰을 요청하는 소스코드는 다음과 같습니다.
```java
public String getKakaoAccessToken(String code) {
        String accessToken = "";
        String refreshToken;
        String reqUrl = "https://kauth.kakao.com/oauth/token";

        try {
            URL url = new URL(reqUrl);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();

            conn.setRequestMethod("POST");
            conn.setDoOutput(true);

            BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(conn.getOutputStream()));
            String sb = "grant_type=authorization_code" +
                    "&client_id=[키 보안]" +
                    "&redirect_uri=[URI 보안]" +
                    "&code=" + code;
            bw.write(sb);
            bw.flush();

            int responseCode = conn.getResponseCode();
            log.info(String.valueOf(responseCode));

            BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()));
            String line="";
            String result="";

            while((line = br.readLine()) != null) {
                result += line;
            }
            log.info(result);

            JsonParser parser = new JsonParser();
            JsonElement element = parser.parse(result);

            accessToken = element.getAsJsonObject().get("access_token").getAsString();
            refreshToken = element.getAsJsonObject().get("refresh_token").getAsString();

            br.close();
            bw.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return accessToken;
    }

```
백엔드 서버에서 카카오 서버로 요청을 전송하기 때문에 ```java.net.URL```을 이용하여 전송을 시도했습니다.
Kakao Developer에서 제공하는 API로 엑세스 토큰을 요청합니다. 그 후, 반환받는 데이터들 중 'accessToken'을 찾아 다음 단계에서 활용합니다.

## 4. 카카오 유저 정보 처리
엑세스 토큰 발급까지 성공했다면 엑세스 토큰을 이용하여 계정 정보를 요청합니다.
Kakao Developer에서 제공하는 API는 다음과 같습니다.
```
GET/POST /v2/user/me HTTP/1.1
Host: kapi.kakao.com
Authorization: Bearer ${ACCESS_TOKEN}/KakaoAK ${APP_ADMIN_KEY}
Content-type: application/x-www-form-urlencoded;charset=utf-8
```
![](https://velog.velcdn.com/images/alsrb5606/post/d197c506-9468-4d71-9fa1-476f9f41e022/image.png)
https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#kakaoaccount
여기서 인증 방식은 Bearer 토큰을 이용하며, 발급받은 엑세스토큰으로 진행합니다.

```java
public KakaoDTO createKakaoUser(String token) {
        String reqUrl = "https://kapi.kakao.com/v2/user/me";
        String email = "";
        try {
            URL url = new URL(reqUrl);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();

            conn.setRequestMethod("POST");
            conn.setDoOutput(true);
            conn.setRequestProperty("Authorization","Bearer "+token);

            int responseCode = conn.getResponseCode();

            BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()));
            String line = "";
            String result = "";

            while ((line = br.readLine()) != null) {
                result += line;
            }

            JsonParser parser = new JsonParser();
            JsonElement element = parser.parse(result);

            int id = element.getAsJsonObject().get("id").getAsInt();
            boolean hasEmail = element.getAsJsonObject().get("kakao_account").getAsJsonObject().get("has_email").getAsBoolean();

            if(hasEmail) {
                email = element.getAsJsonObject().get("kakao_account").getAsJsonObject().get("email").getAsString();
            }
            log.info(email);
        } catch (Exception e) {
            e.printStackTrace();
        }
        KakaoDTO dto = KakaoDTO.builder().email(email).build();
        return dto;
    }
```
해당 작업 이후 카카오 계정을 통해 로컬 유저로 등록이 되어있는지 아닌지 확인 작업이 필요하므로, 별도의 객체로 만들어 반환합니다.

## 5. 로컬 유저 데이터 처리
카카오 계정 정보를 반환받는데 성공했다면, 이제 해당 계정이 회원가입이 되어있는지 아닌지 확인하는 절차를 수행하면 끝입니다.
```java
 @GetMapping("***")
    public ResponseEntity<?> kakaoGetToken(@RequestParam String code) {
        log.info("ENTER ***");
        //Kakao Login 수행 과정
        String accessToken = kakaoOauthService.getKakaoAccessToken(code);
        KakaoDTO kakaoDTO = kakaoOauthService.createKakaoUser(accessToken);

        //로그인한 카카오 이메일이 로컬 계정으로 등록되어있다면
        if (userService.findByEmail(kakaoDTO.getEmail())) {

            //로컬 멤버 데이터 추출 후 JWT 토큰 추가하여 프론트엔드로 반환
            Member member = userService.getByEmail(kakaoDTO.getEmail());
            //멤버 못가져올경우 예외처리 조건문
            if (member != null) {
                final String token = tokenProvider.create(member); //토큰 생성
                final LoginDTO responseUserDTO = LoginDTO.builder() //프론트로 반환할 DTO 생성
                        .email(member.getEmail())
                        .name(member.getName())
                        .memberBelong(member.getMemberBelong())
                        .id(member.getId())
                        .token(token)
                        .memberType(member.getMemberType())
                        .build();

                List<LoginDTO> dtos = new ArrayList<>();
                dtos.add(responseUserDTO);
                ResponseDTO response = ResponseDTO.<LoginDTO>builder().data(dtos).build();
                log.info("LEAVE *** - LOGIN SUCCESS");
                return ResponseEntity.ok().body(response);
            } else {
                log.error("ERROR *** - LOGIN FAILED");
                ResponseDTO responseDTO = ResponseDTO.builder().error("Login failed").build();
                return ResponseEntity.badRequest().body(responseDTO);
            }
        } else {
            //로컬 미등록 계정
            List<KakaoDTO> dtos = new ArrayList<>();
            dtos.add(kakaoDTO);
            OauthResponseDTO responseDTO = OauthResponseDTO.<KakaoDTO>builder().isUser(false).error("No Local User").data(dtos).build();
            log.error("ERROR *** - NOT REGISTERED USER");
            return ResponseEntity.ok().body(responseDTO);
        }
    }
```
해당 과정은 만약 프로젝트 계정 데이터베이스에 존재하는 계정일 경우, 이미 로컬 회원가입을 완료한 사용자이므로, 프로젝트 계정 로그인으로 추가한 JWT 토큰 발급과정으로 넘어가게됩니다.
만약 데이터베이스에 없는 계정이라면, 프론트 부분에 미등록 사용자라고 메시지를 준 후, 프론트 부분에서 회원가입 페이지로 리다이렉트 하도록 진행했습니다.

## 해결 이후 개인적인 생각
oAuth 로그인 과정은 일반 로컬 로그인 과정보다 조금 더 복잡한 과정으로 이루어져있고, 토큰 발급과 엑세스코드 등 보안 인증 절차에 대한 흐름을 어느정도 이해하는 것이 필요하다고 생각됩니다.
따라서, 앞으로 보안 인증 절차와, 어떻게 데이터를 전송하는데 구성하는 방법과 다양한 방법을 생각하는 것이 필요하다고 느낍니다.
