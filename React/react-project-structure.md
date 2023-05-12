## React 프로젝트 구조

여러 다양한 방식이 있지만, 프로젝트 생성 시 참고용으로 대체적으로 많이 사용되는 구조로 잡아보았다.

```bash
├── src
    ├── assets
        ├── fonts
        ├── images
    ├── components
        ├── common
        ├── footer
        ├── header
        ├── nav
    ├── constants
    ├── hooks (hoc)
    ├── pages
    ├── services (api)
    ├── stores (or contexts)
    └── utils
```

### 설명

- assets
  - 이미지 혹은 폰트 같은 파일
  - public에 직접 넣는 경우와 assets 폴더에 넣는 경우의 차이는 컴파일시에 필요한지 여부이다. favicon과 같이 `index.html` 내부에서 직접 사용하여 컴파일 단계에서 필요하지 않은 파일들은 public 폴더 아래에 넣고, 컴포넌트 내부에서 사용하는 이미지 파일인 경우 assets 폴더에 위치시킨다.
- components
  - 재사용 가능한 컴포넌트들
  - 매우 많아질 수 있기에 하위 폴더로 추가 분류하는 경우가 많다.
- constants
  - 공통적으로 사용하는 상수들 정의한 파일들
- hooks
  - 커스텀 훅
- pages
  - 라우팅 적용하는 페이지 컴포넌트들
  - 모달 창들을 보여주는 역할도 여기서 함
- services
  - api 관련 로직 모듈 파일
- stores
  - redux 사용하는 경우 store, contextAPI 사용하는 경우 contexts라고 지음. 상태 관리를 위한 폴더
  - store를 구성하는 모든 slice와 reducer를 정의한 폴더
- utils
  - 공통으로 사용되는 유틸 파일들
