## SQLAlchemy Lost Connection 에러

### SQLAlchemy conection pool

- 연결이 요청될 때마다 생성되는 것이 아니라, 재사용됨을 의미함
- api가 db에 요청 시 모든 요청에 대해 새 mysql db 연결을 생성하고 작업하고 연결을 닫는 과정을 반복하게 된다. 매번 연결을 생성하고 닫는 이 과정과 비용을 줄이기 위해 connection pool에 반납을 하게 되는 것.
- 즉, 차후에 발생할 데이터베이스 요청에 대비하여 데이터베이스 연결을 캐싱하는 기법
- 짧은 요청이 빈번하게 발생하는 웹 서비스와 같은 곳에 연결 풀 궁합이 잘 맞음

### 설정 값

- pool size = 5 (default)
  - 최대 세션 연결 가능 수
- pool recycle (default = 2H)
  - pool에 세션을 끊지 않고 유지하는 시간 (초). pool_recycle을 설정하면 해당 시간이후에는 연결을 끊지 않고 유지하여 재사용
  - SQLAlchemy 에서는 쿼리 요청이 없더라도 pool recycle로 설정한 일정 주기로 갱신함
  - pool_recycle의 주기가 wait_timeout 시간보다 작아야 한다.
    - wait_timeout 시간보다 짧게 설정해야 timeout전에 재사용 처리를 하기 때문에 사용하는 의미가 있다.
    - 근데 또 pool_recycle을 너무 짧게 설정하면 데이터 처리 중에 Echo를 넣게 되어 문제가 될 수 있음.
    - 따라서 유효 세션일때 사용하도록 wait timeout과 적절히 조정해야 문제가 발생하지 않는다.
- mysql에 적용된 wait_timeout = 288800 (default 8시간)
  - 활동하지 않는 커넥션을 끊을때까지 서버가 대기하는 시간
  - 위의 recycle이 3600이면, 1시간마다 갱신.
- max overflow = 10 (default)
  - 커넥션 요청 pool size개수 + max overflow개수 까지는 'sleep'되어있는 session이 사라지기를 기다림
    - 기본값으로 30초를 기다림.
    - 30초 기다려도 반환되는 연결이 없다면 TimeoutError 예외 발생시킴

## 오류 발생

- 연결이 끊어진 후 요청을 하게되면 연결이 끊어 졌다는("mysql+pymysql://"로 연결 한 경우에는 'Lost connection to MySQL server during query', "mysql://"로 연결한 경우에는 'MySQL server has gone away') exception 에러가 발생
- session이 큐 풀에 연결을 받기 위해 기다리다 30초 이후에 timeoutError 내는 것.
- 큐 풀의 기본 timeout 30초 동안 풀의 모든 연결이 점유된 상태에서 아무것도 받지 못한 상태가 된 것
- --> 기존에 점유한 session에서 제대로 연결을 반환해주지 않아서 발생하는 문제임.
  - 특히 flask 웹 서비스에서 요청시마다 세션이 연결을 불러다 써놓고 pool에 돌려주는 일을 빼먹는 실수가 잦음. 요청이 끝나는 시점에 맞춰 session.close()를 적절히 호출해주어야 함
- 혹은 flask 생애주기 중 before_request 구현에서 데이터베이스에 접근할때, 오류 발생.
  - 본래 디비 연결 필요한 엔드포인트에서만 접속 발생하던 것이 before_request에 붙으면서 모든 엔드포인트가 데이터베이스 연결을 하게 되면 사용량이 폭증하게 됨.
  - 이처럼 전역적인 영역에서 DB 접근을 하는 시나리오는 최소화 하도록하는 것이 좋음.

## 해결방법

- 세션 반납
  - try - catch 에러핸들링
    - finally 구문으로 어떤 경우라도 세션 반납하도록
      - db.session.close()
- config 수정
  - SQLALCHEMY_POOL_SIZE = 10
  - SQLALCHEMY_MAX_OVERFLOW = 10
  - SQLALCHEMY_POOL_RECYCLE = 30
- 결정적으로, 프로젝트 내에서 Flask-SQLAlchemy를 사용하고 있었는데, config내 수정한 sqlalchemy 설정은 기본 SQLAlchemy 패키지 문법으로 사용하고 있었기 때문에 설정이 먹히지 않았던 것. 기본 pool_recycle 시간이 2시간이기 때문에 mysql wait timeout 8시간에서 1시간으로 변경한 뒤로부터 문제가 발생했던 것. 커넥션을 DB에서 먼저 끊었기 때문에 lost connection 에러 발생한 것.
  - Flask-SQLAlchemy 설정주어 문제 해결
