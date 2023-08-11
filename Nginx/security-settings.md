# Nginx 보안 취약점 조치 - 서버 정보 숨기기

## Issue

- ISMS 취약점 조치사항으로, PHP로 개발된 페이지 요청 시 응답 헤더 내 PHP 버전 정보가 노출되지 않도록 조치해야 했다.
  - HTTP header에서 노출되는 정보값들의 노출을 제한하는 것이 보안 상 좋다.
- 운영 서버의 경우 DNS => L4 => server의 구조이지만 개발 서버의 경우 DNS => server의 구조라서 각 서비스별로 포트 포워딩을 따로 해주어야 할 필요성이 있었다.
  - 개발 서버의 443번 포트를 nginx가 점유하고 있으며, 443번 포트로 들어온 요청을 도메인에 따라 proxy 처리하도록

## 조치 방법

### 취약 여부 판별하기

```
curl -I 'IP 주소 또는 도메인 주소'
```

### 웹 서버 설정

- nginx.conf 설정 파일 내 server_tokens 옵션을 off로 주어 헤더 내 버전 정보 노출 제한. default 설정은 on이다.
  ```
  server_tokens off
  ```
- 위 server_tokens 설정을 주어도 php 버전 정보는 노출되는 현상이 발생했다.

### 프록시 서버 설정

- proxy에서 PHP 버전 정보를 노출시킬 수 있다고 판단.
- Nginx proxy 설정 proxy.conf 파일 내 아래 옵션 추가하여 서버 정보가 노출될 수 있는 헤더 제거
  ```
  server {
      ...
      proxy_hide_header X-Powered-By;
      ...
  }
  ```
  - X-Powered-By 헤더(어떤 기술로 개발되어 있는지)를 숨기기.

### Nginx 재구동

- 아래와 같이 재구동하여 더이상 PHP 버전이 노출되지 않는 것을 확인할 수 있다.
  ```
  sudo /nginx경로/nginx -s stop
  sudo /nginx경로/nginx
  ```
- Nginx Reload
  - Nginx 서버를 다시 시작하지만, 현재 실행중인 작업에는 영향을 미치지 않음
  - Nginx의 설정 파일을 다시 로드하여 변경 사항을 적용한다.
  - 기존 연결은 유지되며, 새로운 연결은 새로운 설정으로 처리
  - 변경 사항이 제대로 반영되지 않는 경우가 있을 수 있음
- Nginx Restart
  - Nginx 서버를 완전히 중지한 후 다시 시작
  - 현재 실행중인 연결과 작업을 중단하고, 모든 리소스를 다시 시작
  - 모든 연결이 끊기고 잠시 서비스가 중단될 수 있으므로 L4 분리 후 진행해야 함

### php.ini 수정

- apache의 경우, php설정 파일 내에서도 수정 필요하다.

* php.ini 위치 찾기
  ```
  sudo find / -name php.ini
  ```
* php.ini 파일 내 expose_php off로 설정
  ```
  expose_php = Off
  ```

### 정리

- 운영 서버의 경우 해당 이슈 발생하지 않았다. proxy 서버가 필요하지 않기 때문에 Nginx 웹 서버 설정만 server_tokens off 해주면 적용되었다.
- 개발 서버의 경우 proxy 설정 내에서도 header 숨기는 설정을 따로 해주어야 했다.
- nginx 설정이 먹히지 않는 것 같을 땐 reload 말고 stop/start 를 해보자.
- apache를 이용할 경우, php.ini 설정파일에서도 따로 옵션 주어야 php 버전 정보가 노출되지 않는다.
