## 실제 서비스 로그인 구현 시 고려사항

- 어떤 방식으로 로그인 구현할 것인가?
  - JWT
  - Session
- 토큰 혹은 세션id를 어디에 저장할 것인가?

  - 쿠키
  - 로컬 스토리지
  - 세션 스토리지

- 토큰을 매번 직접 넣어주어야 하는가?
  - 토큰 값을 보관했다가 재사용하는 로직 필요
  - 반복되는 로그인 여부 확인 로직 모듈화
- 유저 정보를 다른곳에서도 사용할 수 있도록
  - 전역 상태 관리 - context API, Redux ..
  - 페이지를 감싸는 _레이아웃 컴포넌트_ 만들기
    - 라우터 정보를 한군데에 넣은 별도 라우터 객체 생성
      - 모든 정보를 한 곳에서 관리하면 유지보수성, 가독성이 좋아지며 휴먼에러 방지할 수 있음
      - nav bar와 authorization 등 활용 가능
- 로그인 페이지와 서비스 페이지 분리

  - 로그인이 불필요한 페이지들과 로그인이 필요한 페이지들 구분

- 인증을 위한 token에는 어떤 것들이 있는가?

- 실제 서비스에 필요한 기능들
  - 자동 로그인
    - 로그인 로직에서 응답받은 토큰 로컬 스토리지에 저장
    - 로그인 페이지가 아닌 다른 페이지에서 로컬 스토리지 값을 이용하여 로그인
  - 로그인 후 로그인 상태 유지
  - 로그아웃
  - 권한 관리

## 토큰 방식 로그인 구현

- 로그인에 토큰을 사용하는 이유?
  - http는 상태가 없기 때문에 (stateless) 각각 누구로부터 온 요청인지 식별할 방법이 없어 식별자로서 토큰을 이용하여 기능을 요청.
  - 인증 시 토큰을 같이 넘겨주어 서버에서 검증 후 응답을 내려줌
- JWT (Json Web Token)
  - RFC에 정의되어 있는 토큰 만드는 웹 표준 기술
  - 내부 데이터 json 형식으로 담고 있는 웹 토큰
  - 내부 데이터
    - header
      - 암호화 규칙
      - 토큰 타입
    - payload
      - 토큰 데이터
    - signature
      - 암호화를 위한 데이터 규격
      - 서버만 알고 있는 secret key로 해싱된 데이터 덩어리
  - JWT 유효성 검증은 서버에서 한다.
    - 서버에서 header와 payload 데이터에 secret key를 더해 똑같이 해싱한 결과를 토큰과 비교
  - 사용자가 갖고 있는 토큰을 HTTP의 Authorization 헤더에 실어 보냄.
  - JWT 장점
    - DB에서 따로 세션을 관리하지 않고, 검사하지 않아도 되어서 더욱 빠르고 DB에 부담이 가지 않는 기능 제공
  - 보안 문제
    - secret key 노출되면 같은 해시값을 얻을 수 있음
    - payload 안에 민감하거나 중요한 값을 넣는다면 탈취당할 시 데이터 복호화로 인한 정보 유출될 수 있다.
    - 토큰 탈취. 이 경우 할 수 있는 일이 없음

### 토큰 보관 방식

- 재접근할 수 없는 런타임 메모리에 저장하는 경우
  - 매번 어떤 동작할 떄마다 로그인 필요
- 로컬 스토리지 혹은 쿠키에 저장
  - 보안 문제
    - XSS(Cross Site Scripting)
    - 쿠키에 저장 시
      - CSRF(Cross-site Request Forgery)
    - 프론트 단독으로는 할 수 있는 일이 없다..
    - 시스템 설계로 해결
      - Access token, Refresh token 사용 전략
        - Access token은 짧은 유효시간을 주고 메모리에 발급
        - Refresh token은 HttpOnly Cookie로 발급(JS로 접근 불가하도록)
          - 여기서 Access token에 직접 httpOnly로 하지 않는 이유는 Refresh token 때문이다. Refresh token은 CSRF(쿠키가 담긴 페이지를 납치하여 보내는) 공격 당할 수 있다. 하지만 Access token은 악성 사이트에서 토큰을 재발급하려하면, 모든 토큰을 무효화시키기 때문에, CSRF 공격으로 탈취당하지 않는다. 로그인을 새로해야하는 불편함이 있지만 보안 문제는 사라짐.
        - Access token이 만료되면 Refresh token을 요청하고, 새로운 Access token을 발급 받는 방식
      - 하지만 하드웨어 탈취로 Refresh token 자체를 탈취될 수 있는 보안 위협이 여전히 남아있다.
- 로그아웃
  - 이 토큰은 사용할 수 없다는 리스트를 따로 만들어두어야 함
- Refresh Token
  - 가장 많이 사용하는 방식
  - 인증 서버에 가서 Access token을 재발급 받아오는 용도로 사용
  - refresh token rotation 보안 전략
- OAuth
  - Open Authorization
  - 대리 인증

## Session 방식 로그인 구현

- 세션
  - 사용자 로그인 이후 로그아웃 혹은 로그인 만료까지의 시간
- 세션 방식 로그인
  - 사용자 로그인이 유효한 시간동안 서버에 session id를 기록해 두고 인증에 사용하는 방식
- 서버가 세션 아이디를 저장하는 방법
  - 쿠키
    - 서버에서 사용자에게 http 통해 전송하는 데이터 조각
    - 서버가 브라우저에 값을 세팅
    - 쿠키에 세션 id만 저장해두고, 쿠키를 통해 이 sid를 브라우저에 전달. 클라이언트에서 서버에 요청할 때마다 sid와 함께 전송.
    - 하지만, 서버에서 모든 세션 id를 저장하기 때문에 서버/백엔드 비용 대폭 증가.
    - 보안면에서 JWT 방식보다는 안전 함. 모든 인증 정보를 서버에서 관리하기 때문.
    - 쿠키 보안 정책 (백엔드에서 설정)
      - SameSite 옵션: None, Lax, Strict
        - 동일 출처. 같은 도메인에서만 저장하고 전송할 것이다는 정책
      - HttpOnly
        - 자바스크립트로 접근 불가능하도록 하는 옵션
      - Secure
        - HTTPS 요청
  - 발급된 Session ID는 브라우저에 쿠키 형태로 저장되지만, 실제 인증 정보는 서버에 저장되어 있음
- 로그아웃
  - 백엔드에서 세션 만료시간 만료시켜버리던가, 삭제 처리

## 로그인 방식 선택 기준

- 서비스 상황에 따라 판단
  - 동시접속자 수
  - 서비스 규모
  - 앱/웹 동시운용 여부
  - 팀 내 인력 구성
  - 일정
  - 보안 중요도
- 보안 중요도가 높다면

  - 동시 로그인(중복 로그인) 제한 기능도 필요

    - JWT 활용한 최선의 방법

      - access token의 유효시간을 짧게 잡고, refresh 토큰을 통한 access token 갱신 때 중복 로그인을 체크하는 방법
      - 혹은 매 요청 시마다 DB에 저장해두고 중복 로그인 체크
      - Redis, 경량화된 비관계형 DB를 이용하여 토큰 혹은 세션만 저장 -> 최대한 DB 부담 덜고 캐시 정도의 기능 이용

    - Session 방식 이용
    - 세션, 토큰 두가지 방식 병행하는 방법
