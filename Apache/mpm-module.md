## Apache MPM 모듈

- MPM(Multi-Processing Module)
  - 웹 요청을 처리하는 방식을 제어하는 모듈

### MPM 종류

- Prefork MPM
  - 기본적으로 설치된 MPM이며, 가장 전통적인 모델
  - 하나의 프로세스당 하나의 스레드를 사용하는 방식
  - 각 프로세스는 독립적으로 요청을 처리하고, 동시에 여러 요청을 처리할 수 있다.
  - 스레드 간의 공유 리소스 없이 독립적으로 동작하므로 안정성이 높다.
  - 하지만 메모리 사용량이 높고, 동시에 처리할 수 있는 요청의 수가 제한적이다.
- Worker MPM
  - 멀티 프로세스 및 멀티 스레드 기반의 MPM
  - 여러 프로세스를 생성하고, 각 프로세스는 여러 개의 스레드를 가질 수 있다.
  - 프로세스 간의 리소스 공유를 통해 메모리 사용량을 줄이고 성능을 향상시킨다.
  - 하지만 스레드 간의 안정성을 고려해야 하며, 일부 모듈은 멀티 스레드에서 작동하지 않을 수 있다.
- Event MPM
  - Worker MPM을 기반으로 개선된 MPM
  - 비동기 I/O 처리를 사용하여 웹 서버의 성능을 향상시킨다.
  - 웹 요청의 대부분은 비동기적으로 처리되며, 오직 블로킹 작업에 대해서만 스레드를 사용한다.
  - Worker MPM에 비해 더 높은 처리량과 저렴한 리소스 사용을 제공한다.
  - 하지만 일부 모듈은 비동기 I/O를 지원하지 않을 수 있으며, 구성이 복잡할 수 있다.
- 결론
  - Prefork MPM은 안정성을 우선시하는 경우 유용
  - Worker MPM은 성능을 개선하는 데 적합
  - Event MPM은 많은 동시 연결이 발생하는 경우에 효과적

### 현재 설정

- 아파치 웹 서버가 사용중인 MPM(Multi-Processing Module) 확인
  ```
  httpd -V
  ```
- `httpd-mpm.conf` 파일 내 설정
  ```
  <IfModule mpm_worker_module>
      StartServers            100
      ServerLimit             200
      MinSpareThreads         1000
      MaxSpareThreads         3000
      ThreadsPerChild         100
      ThreadLimit             100
      MaxRequestWorkers       5000
      MaxConnectionsPerChild  0
  </IfModule>
  ```
  - StartServers
    - 웹 서버가 시작될 때 생성되는 worker 프로세스의 초기 개수를 나타낸다. 이 값은 일반적으로 사용자의 웹 트래픽과 서버의 리소스에 따라 조정된다.
  - ServerLimit
    - 웹 서버에서 동시에 처리할 수 있는 worker 프로세스의 최대 개수를 제한한다. 이 값은 시스템의 리소스 및 성능에 따라 조정되어야 한다.
  - MinSpareThreads
    - worker 프로세스가 유휴 상태일 때 유지되는 최소 스레드 개수를 나타낸다. 이 값은 웹 서버의 응답 시간을 유지하기 위해 설정되는 것이 일반적이다.
  - MaxSpareThreads
    - worker 프로세스가 유휴 상태일 때 허용되는 최대 스레드 개수를 나타낸다. 이 값은 서버 리소스 사용을 제어하고 메모리를 관리하는 데 도움이 된다.
  - ThreadsPerChild
    - 하나의 worker 프로세스에서 생성될 수 있는 최대 스레드 개수를 나타낸다. 이 값은 웹 서버의 응답 처리 능력을 결정하는 중요한 요소이다.
  - ThreadLimit
    - worker 프로세스 당 생성할 수 있는 최대 스레드 개수를 제한한다. 이 값은 `ThreadsPerChild`보다 작거나 같아야 한다.
  - MaxRequestWorkers (MaxClients)
    - 전체 웹 서버에서 동시에 처리할 수 있는 최대 요청 개수를 나타낸다. 이 값은 `ServerLimit`와 `ThreadsPerChild`를 곱한 것보다 작거나 같아야 한다.
  - MaxConnectionsPerChild
    - worker 프로세스가 다시 시작되기 전에 처리할 수 있는 최대 요청 개수를 제한한다. 이 값을 설정하면 worker 프로세스의 메모리 누수를 방지하고 성능을 향상시킬 수 있다.
