## FE 협업을 위한 준비 (preparations for effective collaboration in react project)

### git 정책

- git-flow 브랜칭 전략 정하기
- 커밋 컨벤션

### 버전 통일

- 팀원간 node, yarn 버전 통일
- 현재 본인이 설치한 버전 확인해보기
  ```shell
  $ node -v
  $ yarn -v
  ```
- 23.05.12 기준
  - node.js LTS 버전: 18.16.0
  - yarn Classic Stable: v1.22.19

### 리액트 코딩 컨벤션

- ESLint, Prettier 포맷터 적용을 통해 코드 스타일 통일
- 네이밍 컨벤션
  - component: PascalCase
  - non-component: camelCase
  - unit tests: 파일명과 동일하게
- 함수 선언 방식 통일
  - 함수 선언문
    ```javascript
    export default function 함수선언식() {}
    ```
  - 함수 표현식
    ```javascript
    const 함수표현식 = () => {};
    ```
  - airbnb 코드스타일 가이드에서는 함수표현식으로 작성
- 이벤트 핸들러 네이밍
  - props: on\* 접두사
  - function: handle\* 접두사
  ```jsx
  <MyComponent
    onAlertClick={this.handleAlertClick}
    onclick={this.handleClick}
    onFormSubmit={this.handleFormSubmit}
  />
  ```
- 버그 방지를 위해 null 또는 undefined 일 수 있는 값은 optional chaning 연산자 사용하기
  ```javascript
  obj?.prop;
  ```
- 불필요한 주석과 console.log() 제거
- 코드 효율성과 가독성을 위해 ES6 문법 사용
  - spread 연산자
  - 구조분해할당
  - var 대신 let, const
  - 화살표 함수
- CSS
  - 되도록 inline css 사용하지 않기
  - 색상 코드 상수화
- TypeScript
  - type alias
    - 모든 타입 선언 가능
    - 확장 불가능한 타입 선언
  - interface
    - 객체 타입만 선언 가능
    - 확장 가능한 타입 선언 가능 (선언 병합)
      - 서로 다른 프로퍼티를 갖는 동일한 이름의 타입을 선언하면 에러가 나지 않고 나중에 선언된 타입의 프로퍼티를 가짐
  - 타입스크립트 공식 문서에 따르면 type alias 보다는 interface를 사용할 것을 권장한다.

### 의존성 패키지 설치

- package.json - 의존성 패키지 버전을 range로 관리. yarn install 시 팀원 간 서로 다른 버전의 패키지 설치할 가능성 있음.
- yarn.lock - 정확한 버전 명시
- 패키지 설치 시 yarn install 보다는 아래 명령어 이용하여 lock 파일을 통해 정확한 버전의 의존성 패키지 설치하도록 하기

  ```shell
  yarn install --immutable --immutable-cache --check-cache
  ```
