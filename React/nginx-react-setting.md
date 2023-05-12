# FE Settings

### FE 개발 정책

- React.js + Typescript (CRA 사용)
- style: PostCSS
- JS 패키지 매니저: yarn
- 컴포넌트 선언 방식: 함수형 컴포넌트
- 전역 상태 관리
  - ContextAPI
- 데이터 페칭 관련 라이브러리
  - react-query
  - graphQL?
- ESLint + Prettier 설정
  - airbnb 컨벤션 참고
- 테스트: Jest

## React 프로젝트 세팅

- 사전에 설치 작업
  - node.js 설치 (https://nodejs.org/ko/download)
  - 패키지 매니저 yarn 설치
    ```shell
    $ npm install -g yarn
    ```

### 프로젝트 생성 과정

- `create-react-app` 이용하여 React + Typescript 프로젝트 생성

  ```shell
  $ npx create-react-app my-app --template typescript
  ```

- ESLint, Prettier 패키지 설치

  ```shell
  $ yarn add -D eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser

  $ yarn add -D prettier eslint-config-prettier eslint-plugin-prettier
  ```

- 저장 시 자동 format 및 자동 lint 설정
  - vscode 설정 - format on save 체크
  - 자동 lint
    `   "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
},`

* vscode plugin 설치
  - Prettier - Code formatter
  - ESLint
  - reactjs code snippets
* vscode PostCSS emmet 단축키 설정
  - vscode settings.json 수정
    ```json
    "emmet.includeLanguages": {
        "javascript": "javascriptreact",
        "postcss": "css"
    },
    "emmet.syntaxProfiles": {
        "postcss": "css"
    }
    ```
* Typescript에서 CSS모듈 파일 임포트 가능하도록 설정. 이 설정 하지 않을 경우, 타입스크립트가 CSS 모듈 파일에 대한 타입 선언 파일(`module.css.d.ts`)을 찾을 수 없어 오류 발생.

  ```
  $ yarn add -D typescript-plugin-css-modules
  ```

  - tsconfig.json 추가

    ```json
    {
        "compilerOptions": {
            "plugins": [{ "name": "typescript-plugin-css-modules" }]
        }

            ...

        "include": [
            "global.d.ts",
            ...
        ]
    }
    ```

  - global.d.ts 파일 프로젝트 루트 경로에 생성
    ```typescript
    declare module "*.module.css" {
      const classes: { [key: string]: string };
      export default classes;
    }
    ```

- 크롬 익스텐션 설치
  - react developer tools

### 프로젝트 실행 방법

```shell
$ yarn && yarn start
```

---

## React 앱 Nginx 웹 서버 배포

- React 앱 빌드
  ```shell
  $ yarn build
  ```
- nginx 설치 (root 권한 로그인)

  ```shell
  $ yum update
  $ yum install nginx

  // nginx 서비스 시작 및 부팅 시 자동으로 시작하도록 설정
  $ systemctl start nginx
  $ systemctl enable nginx

  // 방화벽 설정 업데이트 - http, https 트래픽 허용
  $ firewall-cmd --permanent --add-service=http
  $ firewall-cmd --permanent --add-service=https
  $ firewall-cmd --reload
  ```

- /etc/nginx/nginx.conf 설정 - 8090포트 리액트 앱 띄우기

  ```conf
   server {
      listen       8090 default_server;
      listen       [::]:8090 default_server;
      server_name  _;

      # react app build 경로
      root         /home/service/wasd/
      upi-global-test/build;
      index index.html;

      # Load configuration files for the default server block.
      include /etc/nginx/default.d/*.conf;

      location / {
          try_files $uri $uri/ /index.html;
      }

      ...
  }
  ```
