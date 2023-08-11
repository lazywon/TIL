# Javascript HTTP 요청 방식

### Ajax (Asynchronous JavaScript And XML)

- 전통적인 비동기 http 요청 방식
- XMLHttpRequest(XHR) 객체 사용
- Javascript로 직접 호출할 수 있지만, jQuery를 사용하면 더 쉽게 구현 가능하고 jQuery를 사용할 때 훨씬 코드가 간결해지고 직관적임
- Promise 기반이 아님

### Fetch

- ES6부터 생긴 JavaScript 내장 라이브러리
- XMLHttpRequest와 비슷하지만 더 강력하고 유연한 조작 가능
- Promise 기반이기 때문에 데이터 다루기 쉽고, 내장 라이브러리이므로 별도로 설치할 필요 없음
- 나름 최신 기술이라 익스플로러 미지원

### Axios

- Node.js와 브라우저를 위한 Promise API를 활용하는 HTTP 통신 라이브러리
- Promise 객체를 반환하기 때문에 response 데이터를 다루기 쉽다.
- fetch에는 없는 response timeout 처리 방법이 존재
- 브라우저 호환성 우수
- 사용을 위해 모듈 설치 필요
