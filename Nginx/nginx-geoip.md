# Nginx GeoIP 이용한 IP 주소 접근 차단

## 들어가며

- 해외 IP 접근 차단 기능이 필요하다는 요구 사항
- 최대한 Docker-Nginx ISMS 취약점 심사를 마친 기존에 사용중인 이미지 및 Nginx 설정을 기반으로 변경사항은 최소화하고 새로운 설정만 추가하는 것을 목표로 했다.
  - 기존 베이스 이미지: [nginxinc/nginx-unprivileged](https://hub.docker.com/r/nginxinc/nginx-unprivileged)
    - root가 아닌 user로 컨테이너를 구동해야 한다는 보안 사항에 따라 사용하게 된 nginx 베이스 이미지

## Nginx GeoIP

- IP 주소를 국가 코드로 변환한 GeoIP 데이터베이스를 [MaxMind](https://www.maxmind.com/en/home)에서 제공한다.

### GeoLite2 선택 이유

- GeoIP.dat 파일은 19년도 기준으로 MaxMind에서 더 이상 지원하지 않음.

  - GeoIP.dat 파일을 사용하기 위해서는 `ngx_http_geoip_module` 모듈이 Nginx에 설치되어 있어야 한다.
    - 해당 모듈은 IP 주소를 GeoIP.dat 파일과 비교하여 해당 IP가 어느 국가에서 오는지 판별하는 역할을 함
    - 그러나 해당 모듈은 Nginx 상위 버전에서는 공식적으로 지원되지 않음. 따라서 해당 모듈 사용하려면 Nginx를 해당 모듈과 함께 빌드하거나, Nginx의 하위 버전을 사용해야 한다.
    - GeoIP2는 Nginx 상위 버전에서 잘 동작함. 현재 서비스에서 이용중인 Nginx 버전: 1.23.0

- GeoIP2는 유료 패키지이며, GeoLite2는 GeoIP2의 무료 버전
  - GeoIP2를 사용하기 위해서는 `ngx_http_geoip2_module` 모듈을 사용해야 한다.
  - 해당 모듈은 Nginx 상위 버전에서도 지원됨.

### GeoLite2 DB 파일 다운로드 절차

- MaxMind 회원가입 후, 라이센스 키를 생성한다.
- 생성한 라이센스 키 이용하여 GeoLite2 DB 파일 다운로드
  ```bash
  wget "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=YOUR_LICENSE_KEY&suffix=tar.gz" -O GeoLite2-Country.tar.gz
  ```
- 다운로드 받은 tar 파일 압축 해제하여 .mmdb 파일 얻기
  ```bash
  tar -xzvf GeoLite2-Country.tar.gz
  ```

### 파일 정상 작동 확인 방법

```bash
mmdblookup --file ./GeoLite2_Country.mmdb --ip <조회할 IP> country iso_code
```

- 출력 결과 국가 코드 확인
  ```bash
  "KR" <utf8_string>
  ```

### 추가로 정해야 하는 사항

- 데이터베이스 파일 갱신
  - 수동 or 별도 스크립트를 통해 주기적으로 업데이트 필요
    - 업데이트 주기
    - 업데이트 방식

---

## custom image 이용한 컨테이너 구동 절차

- 기존 `ginxinc/nginx-unprivileged` 이미지에서 geoip 추가한 새로운 커스텀 이미지 생성

### custom docker image 생성

#### GeoLite2 DB 파일 위치 아래 경로로 옮겨 놓기

```
/my-path/etc/nginx/conf.d/GeoLite2-Country.mmdb
```

#### Dockerfile 생성

```
FROM nginxinc/nginx-unprivileged:1.23.0

ARG NGINX_HOME=/etc/nginx

WORKDIR ${NGINX_HOME}

USER root

# 필요한 패키지 설치
RUN apt-get update && \
  apt-get install -y \
  libmaxminddb-dev \
  git \
  wget \
  build-essential \
  libpcre3-dev \
  zlib1g-dev

# ngx_http_geoip2_module 다운로드 및 설치
RUN git clone --recursive https://github.com/leev/ngx_http_geoip2_module.git && \
  wget http://nginx.org/download/nginx-1.23.0.tar.gz && \
  tar -xvzf nginx-1.23.0.tar.gz && \
  cd nginx-1.23.0 && \
  ./configure --with-compat --add-dynamic-module=../ngx_http_geoip2_module && \
  make modules && \
  cp objs/ngx_http_geoip2_module.so /etc/nginx/modules

# Nginx 설정 파일 복사
COPY nginx.conf ${NGINX_HOME}/nginx.conf

USER nginx

CMD ["nginx", "-g", "daemon off;"]
```

- 패키지 설치
  - `libmaxminddb-dev` : GeoIP2 데이터베이스 파일 읽는 데 사용
  - `build-essential` : Nginx 빌드 시 필요
  - `libpcre3-dev` : Nginx의 HTTP rewrite 모듈에서 PCRE 라이브러리 필요로 해서 추가 설치
  - `zlib1g-dev` : Nginx의 HTTP gzip 모듈에서 zlib 라이브러리 필요로 해서 추가 설치
- `ngx_http_geoip2_module` 은 동적 모듈로서, Nginx를 빌드할 때 해당 모듈을 추가하여 빌드해야 한다. 그래서 베이스 이미지의 Nginx 버전과 동일한 버전의 Nginx 소스 코드를 다운로드하여 모듈을 빌드하였다.
  - apt-get 으로 `ngx_http_geoip2_module` 설치하려 하면 해당 모듈이 빌드된 Nginx 버전과 현재 실행 환경의 Nginx 버전이 일치하지 않아 호환성 문제가 발생함.
- nginx.conf 설정 파일 생성 및 복사 이유
  - 최상단에 `ngx_http_geoip2_module` 로드 필요하여 기존 설정 파일 사용 불가

#### nginx.conf 생성

```
load_module modules/ngx_http_geoip2_module.so;

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /tmp/nginx.pid;


events {
    worker_connections  1024;
}

http {
    client_max_body_size 10M;
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

- 최상단에 `ngx_http_geoip2_module` 로드

#### default.conf

```
# If we receive X-Forwarded-Proto, pass it through; otherwise, pass along the
# scheme used to connect to this server
map $http_x_forwarded_proto $proxy_x_forwarded_proto {
  default $http_x_forwarded_proto;
  ''      $scheme;
}
# If we receive X-Forwarded-Port, pass it through; otherwise, pass along the
# server port the client connected to
map $http_x_forwarded_port $proxy_x_forwarded_port {
  default $http_x_forwarded_port;
  ''      $server_port;
}
# If we receive Upgrade, set Connection to "upgrade"; otherwise, delete any
# Connection header that may have been passed to this server
map $http_upgrade $proxy_connection {
  default upgrade;
  '' close;
}
# Apply fix for very long server names
server_names_hash_bucket_size 128;

# Default dhparam
ssl_dhparam /etc/nginx/certs/dhparam.pem;

# Set appropriate X-Forwarded-Ssl header based on $proxy_x_forwarded_proto
map $proxy_x_forwarded_proto $proxy_x_forwarded_ssl {
  default off;
  https on;
}

gzip_types text/plain text/css application/javascript application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
log_format vhost '$host $remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '"$upstream_addr" "$geoip2_data_country_code"';
access_log off;

#...(생략)

# HTTP 1.1 support
proxy_http_version 1.1;
proxy_buffering off;
proxy_set_header Host $http_host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $proxy_connection;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $proxy_x_forwarded_proto;
proxy_set_header X-Forwarded-Ssl $proxy_x_forwarded_ssl;
proxy_set_header X-Forwarded-Port $proxy_x_forwarded_port;
# Mitigate httpoxy attack (see README for details)
proxy_set_header Proxy "";
```

#### geoip.conf

```
# geoip2 설정
geoip2 /etc/nginx/conf.d/GeoLite2-Country.mmdb {
    auto_reload 5m;
    $geoip2_metadata_country_build metadata build_epoch;
    $geoip2_data_country_code country iso_code;
}

map $geoip2_data_country_code $allowed_country {
    default yes;
    KR yes;
    CH no; # 차단할 국가 코드 예시
}

upstream proxy.io-upstream {
    server $<SERVER_IP>:$<PORT> max_conns=4000;
    keepalive 100;
}
server {
    server_name proxy.io;

    listen 9440;

    # L4 없는 경우 설정
    # set_real_ip_from 172.17.0.1;
    # real_ip_header X-Real-IP;

    server_tokens off;
    proxy_pass_header server;
    keepalive_timeout 2;
    client_body_buffer_size 100k;

    access_log /var/log/nginx/access.log vhost;
        location / {
                proxy_connect_timeout 5;
                proxy_read_timeout 75;
                proxy_pass http://proxy.io-upstream;

                # geoip2 설정
                if ($allowed_country = no) {
                    return 444;
                }

        }
}
```

- geoip2 설정 부분
  - auto_reload 5m
    - 5분마다 데이터베이스 파일 변경 모니터링하여 변경 감지 시 자동으로 reload. nginx 서버 재시작하거나 리로드할 필요 없음.
  - $geoip2_metadata_country_build metadata build_epoch
    - 데이터베이스의 빌드 시간(에포크 시간)을 $geoip2_metadata_country_build 변수에 저장
  - `$geoip2_data_country_code` country iso_code
    - 사용자의 IP 주소를 국가 코드로 변환하고, 이를 `$geoip2_data_country_code` 변수에 저장
  - 접근 차단 국가 설정(KR yes; CH no;) 임의로 지정
  - location 블록 내에 차단 국가 확인 후 리턴 설정
    - 반환할 http status code 임의 지정

#### Makefile 생성

```
DOCKER_TAG = nginxinc/nginx-unprivileged-custom-geoip

dockerize:
  docker build -t $(DOCKER_TAG) .

.PHONY: dockerize
```

- TAG명 `nginxinc/nginx-unprivileged-custom-geoip`으로 생성

#### Makefile 실행하여 도커 이미지 빌드

```bash
$ make dockerize
```

### Docker Container 실행

- run.sh 실행하여 Docker Container 구동
- run.sh

  ```
  #!/bin/bash -e

  IMAGE=nginxinc/nginx-unprivileged-custom-geoip

  OPTS=(
    "--name unprivileged-proxy-geoip"
    "--log-opt max-size=100m"
    "--log-opt max-file=10"
    "-p 9440:9440"
    "-v /var/run/docker.sock:/tmp/docker.sock:ro"
    "-v /my-path/etc/nginx/certs:/etc/nginx/certs"
    "-v /my-path/etc/nginx/conf.d:/etc/nginx/conf.d:rw"
    "-v /my-path:/var/log/nginx"
    "--pids-limit=100"
    "--security-opt=no-new-privileges"
  )

  docker run ${OPTS[@]} -d $IMAGE
  ```

- nginx 설정 변경으로 reload
  ```
  docker exec -it unprivileged-txproxy.p512 nginx -s reload
  ```

---

## 이슈 history

- 이슈
  - 도커 컨테이너 내의 nginx 웹서버에서 로그 출력 시 remote_addr 값이 도커 내부 네트워크 주소인 '172.17.0.1'로 고정 출력되는 현상이 발생했다.
  - 이에 따라 remote_addr 값이 해당 값으로 출력되어 geoip 국가 코드에 일치하는 IP가 없는 것으로 판단되어 정상 작동하지 않는 문제가 발생했다.
- Docker 컨테이너 환경에서는 Docker 네트워킹 구조와 NAT 때문에 클라이언트 IP 주소가 변경된다.
- 이를 해결하려면 도커 앞단에 _프록시 서버_ 혹은 *로드 밸런서*를 사용하여 X-Real-IP 헤더에 클라이언트의 실제 원본 IP를 담아 전달하거나, 도커 네트워크 모드를 'host'로 설정하는 방법이 있다.
  - 단, 도커 네트워크 모드를 host로 설정하는 것은 보안상의 이유 및 ISMS 보안 취약점으로 도출될 가능성으로 인해 권장하지 않는다.
- 운영 서버의 경우 L4 통해 포트 포워딩 되므로, 도커 호스트 네트워크 ip가 아닌 $remote_addr가 원본 클라이언트 IP로 잘 찍히나, 개발 서버의 경우 L4가 없어서 호스트에 설치된 Nginx를 통해 포트 포워딩 하도록 해야 했다.

### Nginx 프록시 이용한 해결 방법

#### 호스트의 Nginx proxy 세팅

```
location / {
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_pass http://127.0.0.1:9440;
}
```

- _X-Real-IP, X-Forwarded-For_
  - 웹서버가 클라이언트의 실제 IP 주소를 알아내는 데 사용되는 HTTP 요청 헤더로, 주로 프록시 서버나 로드 밸런서를 통해 웹 서버에 전달된다.
  - X-Real-IP: 클라이언트의 원본 IP 주소
  - X-Forwarded-For: 클라이언트의 원본 IP 주소와 함께 중간에 위치한 모든 프록시 서버의 IP 주소를 포함하여, 전체 경로를 추적 가능

* `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
  - 'X-Forwarded-For' 헤더에 $proxy_add_x_forwarded_for의 값을 설정하여, 클라이언트의 원본 IP와 그 요청이 지나온 모든 프록시 서버들의 IP 주소를 담아서 프록시로 전달

- `proxy_set_header X-Real-IP $remote_addr;`
  - 'X-Real-IP' 헤더에 $remote_addr의 값을 설정하여, 바로 직전 클라이언트의 IP 주소를 담아서 프록시로 전달

#### unprivileged-proxy-geoip 컨테이너 내 Nginx 설정

- Nginx real_ip 모듈을 사용

```
set_real_ip_from 172.17.0.1; # or your docker network range
real_ip_header X-Real-IP;
real_ip_recursive on;
```

- set_real_ip_from에서 온 요청일 경우 $remote_addr를 real_ip_header에 명시되어있는 헤더 값으로 변경
