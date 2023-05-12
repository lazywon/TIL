## FE 협업을 위한 준비 (preparations for effective collaboration in react project)

### git 정책

- git-flow 브랜칭 전략 정하기
  - 브랜치 (master - develop - feature)
    - master - 운영 서버에 배포된 소스 관리
    - develop - 개발 서버에 배포된 소스 관리
    - feature - 지라 이슈 단위
- 커밋 컨벤션
  - commit 메시지 컨벤션 참고
    - [좋은 git commit 메시지를 위한 영어 사전](https://blog.ull.im/engineering/2019/03/10/logs-on-git.html)
    - [GIT](https://danalfintech.atlassian.net/wiki/spaces/R/pages/169148429)

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

- ESLint, Prettier 포맷터 자동 적용을 통해 코드 스타일 및 포맷 통일
- 네이밍 컨벤션
  - component: PascalCase
  - non-component: camelCase
  - 속성 (className, onClick..): camelCase
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

- package.json

  - 설치하려는 모듈에 대한 의존성 목록을 갖는다.
  - 의존성 패키지 버전을 `range`로 관리.

  ```json
  "react": "^17.0.2",      // ^, ~, <=
  ```

  - `yarn install` 시 팀원끼리 서로 다른 버전의 node_modules를 생성할 수 있음. 즉, `yarn install` 혹은 `npm install` 을 실행하면 서로 다른 버전을 가지는 모듈을 가지는 경우가 생길 수 있다.

- yarn.lock

  - npm을 사용하는 경우, package-lock.json 파일이 생성됨.
  - 위 문제 해결을 위해 해당 파일이 존재한다.
  - 정확한 버전이 명시되어 있어 package.json을 기준으로 설치할때 보다 정확한 의존성 모듈을 설치할 수 있다.

  ```json
  react@^17.0.2:
  version "17.0.2"
  ```

  - 이러한 이유로 package-lock.json, yarn.lock을 git에 커밋해 두어야 한다.
  - 전체를 대상으로 install하며 개별적으로는 불가능
  - 명령어 실행 시 lock 파일이 무조건 존재해야하고 만약 없으면 에러를 뱉는다.
  - 이미 node_modules가 존재하는 경우 명령어 실행 전에 node_modules를 지우고 실행해야 함

  - 패키지 설치 시 `yarn install` 보다는 아래 명령어 이용하여 lock 파일을 통해 정확한 버전의 의존성 패키지 설치하도록 하기

    ```shell
    yarn install --immutable --immutable-cache --check-cache
    ```

  - npm의 경우
    ```shell
    npm ci
    ```
  - `npm install`의 경우 packag.json과 package-lock.json에 쓰기 권한을 갖지만 npm ci 는 없다. 오직 lock파일을 읽고 의존성 목록을 설치하기 때문에 쓰기 권한이 없는 것.
  - 그래서 개발 환경이 아닌 CI 환경에서는 npm install 보다는 적합한 방안으로 여겨진다.

- package-lock.json 파일을 기반으로 의존성을 설치하고, package.json 은 버전 매칭 용도로 사용
