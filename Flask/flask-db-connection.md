# Flask에서 데이터베이스 연동하기

## 들어가면서

먼저 관련 공부를 하게 된 계기는, 사내 Flask 프로젝트의 DB Replication 관련 이슈 확인을 위함이었다. 문제가 될 만한 부분을 찾아보고자 DB 연동 관련 코드를 보게 되었고, 생각보다 코드 내에 문제의 소지가 될만한 부분들이 많아 보였다.

다양한 방식으로 산재되어 있는 데이터베이스 관련 처리 코드를 상황에 따라 통일할 수 있도록 리팩토링을 목표로 한다.
또한 SQLAlchemy 관련 오류를 마주치면 원인 파악에 조금이라도 도움될 수 있도록 최소한 라이브러리에 대해 공부하고 코드를 작성하고자 한다.

## 사전 지식

### PyMySQL

- 파이썬에서 MySQL을 사용하기 위한 라이브러리

#### 사용법

```python
import pymysql

# Connection 객체 생성
conn = pymysql.connect(host='localhost', user='root', password='password', charset='utf8')

# Connection 객체로부터 Cursor객체 생성
cursor = conn.cursor()

# 새 데이터베이스 생성
sql = "CREATE DATABASE test"
cursor.execute(sql)

# 트랜잭션 커밋 및 db 연결 종료
conn.commit()
conn.close()
```

#### Database Connection

- 데이터베이스와의 연결을 나타내는 객체. 커넥션은 커서를 생성하고 관리하는 역할을 수행
- 주요 작업 : 커서 생성, 트랜잭션 관리, 커넥션 풀 관리하여 재사용 가능한 연결 유지

#### Database Cursor

- 하나의 DB connection에 대하여 독립적으로 SQL 문을 실행할 수 있는 작업환경을 제공하는 객체
- 즉, 데이터베이스 연결을 통해 쿼리를 실행하고 결과를 조회하는데 사용되는 객체
- 주요 작업 : 쿼리 실행, 쿼리 실행 결과 조회 및 수정, 트랜잭션의 커밋과 롤백 관리

### SQLAlchemy

- 파이썬에서 사용할 수 있는 ORM 라이브러리로
- ORM(객체-관계 매핑) 기능과 데이터베이스 추상화 도구를 제공
- SQLAlchemy를 사용하면 데이터베이스 연결, 쿼리 작성, 테이블 생성, 데이터 조작 등을 쉽게 할 수 있다.

#### SQLAlchemy 사용법

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# SQLAlchemy 엔진 생성
engine = create_engine('mysql+pymysql://root:password@localhost', encoding='utf8', echo=True)

# Session 클래스 생성
Session = sessionmaker(bind=engine)
session = Session()

# 새 데이터베이스 생성
sql = "CREATE DATABASE test"
session.execute(sql)

# 트랜잭션 커밋 및 세션 종료
session.commit()
session.close()
```

#### SQLAlchemy의 Session

- SQLAlchemy의 세션은 ORM 작업 수행 시 사용되는 핵심 개념 중 하나
- 주요 작업 :
  - 세션을 통해 ORM으로 정의된 클래스의 객체를 생성, 수정, 삭제 및 데이터베이스와 동기화
  - 트랜잭션 관리
  - 조회된 객체 캐싱하고, 필요 시 연관된 객체를 로드하는 지연 로딩 기능 제공
  - 데이터베이스와의 동시 작업 관리 및 병렬 처리 지원
- sessionmaker
  - session을 만들어주는 팩토리 클래스

#### SQLAlchemy vs Flask SQLAlchemy

일반 SQLAlchemy 예제와 조금씩 다른 프로젝트 내 코드를 보다가 일반 SQLAlchemy가 아닌 Flask SQLAlchemy를 사용하고 있었다는 것을 깨닫게 되었다.
일반적으로 python으로 개발 시 ORM은 SQLAlchemy밖에 선택권이 없으나, Flask로 개발 시에는 일반 SQLAlchemy와 Flask에 최적화된 Flask-SQLAlchemy 중 선택할 수 있다.
이 두 라이브러리는 완전히 다른 라이브러리는 아니고, Flask-SQLAlchemy를 사용하면 SQLAlchemy의 강력한 기능을 더 쉽고 편리하게 활용할 수 있다.

- `일반 SQLAlchemy`
  - 자체적으로 세션을 생성하고 관리해야 한다.
  - SQLAlchemy는 SQLAlchemy ORM 및 Core 기능을 포함하고 있다.
- `Flask-SQLAlchemy`
  - Flask 프레임워크와 통합되어 더 편리하게 사용할 수 있도록 도와주는 도구.
  - SQLAlchemy의 `db` 객체를 생성하여 애플리케이션과 연결한 뒤, `db.session`을 통해 세션을 관리하며, 모델 클래스와 연동하여 데이터베이스 작업을 처리할 수 있다.
    - db.session은 데이터베이스와 연결된 세션, 즉 접속된 상태를 의미함. 이 세션을 통해 데이터 저장, 수정, 삭제 작업 진행

Flask-SQLAlchemy는 세션 관리를 app 컨텍스트에 맞춰 관리해 주기 때문에 편리하지만, 일반 SQLAlchemy를 사용하는 경우 개발자가 직접 세션을 관리해야 한다.

#### Flask-SQLAlchemy 사용법

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

# Flask 앱 생성 및 DB 연결 정보 설정
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:password@localhost'
db = SQLAlchemy(app)

# 새 데이터베이스 생성
sql = "CREATE DATABASE test"
db.session.execute(sql)

# 트랜잭션 커밋 및 세션 종료
db.session.commit()
db.session.close()
```

## 데이터베이스 연동 및 작업 처리 방식

프로젝트 내에서는 MySQL과 연동하고 있는데, 두가지 방식이 코드에 산재되어 있다.

1. 직접적으로 `pymysql 드라이버`를 사용하여 데이터베이스 `커서(Cursor)객체`를 이용하여 쿼리를 다루는 방식
   - 데이터베이스 커서를 이용한 방식은 더 저수준의 데이터베이스 작업을 위해 필요할 때에 선택한다.
   - 직접적으로 데이터베이스 드라이버를 사용하기 때문에, SQLAlchemy ORM 기능을 사용하지 않는다.
   - 대신 SQL문을 직접 작성하여 데이터베이스 쿼리를 수행한다.
   - 데이터베이스 연결 관리
     - 데이터베이스 커서를 사용하는 방식은 직접적으로 데이터베이스 연결을 다루기 때문에, 연결 관리에 대한 책임이 프로그래머에게 있다.
     - 이로 인해 연결 관리를 제대로 처리하지 않으면 불필요한 자원 소모가 발생할 수 있다.
2. `SQLAlchemy의 Session과 ORM 기능`을 사용하여 처리하는 방식

   - 일반적으로 데이터베이스 작업을 더 편리하고 가독성 좋게 처리하기 위해서는 SQLAlchemy를 사용한다.
   - 데이터베이스 연결 관리

     - SQLAlchemy 세션을 사용하는 방식은 SQLAlchemy에 내장된 연결 관리 기능을 활용하여 데이터베이스 연결을 효율적으로 관리한다.
     - SQLAlchemy 세션은 기본적으로 필요할 때 연결을 생성하고, 작업이 끝나면 연결을 자동으로 반환하는 기능을 제공한다. 이러한 자동 관리 기능은 개발자가 직접 세션을 열거나 닫을 필요가 없어 편리하다.

       - 세션 자동 관리는 주로 `with` 문을 사용하여 구현된다.

       ```python
       from sqlalchemy.orm import sessionmaker

       Session = sessionmaker(bind=engine)
       with Session() as session:
           # 세션을 사용한 데이터베이스 작업 수행
           # ...

       # with 블록이 끝나면 세션은 자동으로 닫힌다.
       ```

       - 하지만, 세션 내에서 오류가 발생했을 때는 해당 세션을 롤백하고 세션을 닫는 것이 좋고, 세션을 사용한 데이터베이스 작업이 길거나 복잡할 경우에도 수동으로 세션을 닫아 리소스를 더 효율적으로 관리할 수 있다는 장점도 있기 때문에 상황에 맞게 결정하면 된다.

### SQLAlchemy

- 여러 종류의 데이터베이스 시스템과 호환되며, pymysql을 포함하여 다양한 드라이버를 지원한다. 그렇기 때문에 데이터베이스 종류에 따라 다른 드라이버를 사용하여 `여러 데이터베이스 시스템과 호환`될 수 있다.
  - ex)
    ```python
    SQLALCHEMY_BINDS = {
        'mysql_db': 'mysql+pymysql://user1:password1@localhost/db1',
        'postgres_db': 'postgresql+psycopg2://user2:password2@localhost/db2'
    }
    ```

### pymysql

- MySQL 데이터베이스에 접속하기 위한 파이썬 드라이버
- SQLAlchemy는 데이터베이스와 상호작용하기 위해 내부적으로 pymysql과 같은 드라이버를 사용한다.
- 즉, SQLAlchemy를 사용할 때에도 pymysql 라이브러리가 필요하며, 일반적으로 SQLAlchemy를 사용하면서 pymysql을 따로 사용할 필요는 없다.

### 정리

- SQLAlchemy는 여러 종류의 데이터베이스 시스템과 호환되며, pymysql을 포함하여 `다양한 드라이버를 지원`하기 때문에 데이터베이스 종류에 따라 다른 드라이버를 사용할 수 있다. (SQLAlchemy 내부적으로 mysql 연동 시에 pymysql 드라이버 사용)

## SQLAlchemy에서 데이터베이스 작업을 처리하는 방식

### SQLAlchemy ORM (Object-Relational Mapping)

- 데이터베이스 테이블과 Python 클래스를 매핑하여 객체로 데이터를 다루는 방식
- ORM을 사용하면 SQL을 직접 작성하지 않고도 데이터베이스 작업을 수행할 수 있어 코드 가독성과 유지보수성 높일 수 있다.
- ORM을 통해 정의한 데이터베이스 model을 가지고 Query 객체를 사용하여 데이터베이스 쿼리를 작성하고, 필터 적용, 결과 정렬, 조인 등을 수행한다.
- 데이터베이스 바인딩은 모델의 **bind_key** 속성을 사용하여 정의하므로, 각 모델을 특정 데이터베이스에 바인딩할 수 있다. 기본적으로 **bind_key**를 정의하지 않은 모델은 `SQLALCHEMY_DATABASE_URI`에 설정된 기본 데이터베이스와 연결된다.
  - ex)
    ```python
    class User(db.Model):
    __bind_key__ = 'db1'  # 'db1'에 바인딩
    __tablename__ = 'TB_USER'
    # 모델 정의
    ```
  - 기본적으로 config.py에 설정된 `SQLALCHEMY_DATABASE_URI` 값에 따라 설정한 디폴트 데이터베이스에 연결되지만, ORM을 통해 정의한 Model을 사용하여 쿼리 수행 후 `db.session.commit()` 을 호출하면 디폴트로 설정된 데이터베이스가 아닌, 해당 모델을 통해 바인딩 된 데이터베이스 연결 정보에 변경 사항이 커밋된다.
- SQLAlchemy의 쿼리를 작성하는 두 가지 방법
  - Flask-SQLAlchemy의 db.session 사용
    ```python
    db.session.query(User).filter()
    ```
    - db.session은 기본적으로 SQLALCHEMY_DATABASE_URI에 설정한 데이터베이스와 연결된다.
    - 세션을 직접 사용하기 때문에 세션 관리에 더 많은 제어를 할 수 있다.
  - SQLAlchemy의 모델 클래스 사용
    ```python
    User.query.filter()
    ```
    - User는 Base 클래스를 상속받은 SQLAlchemy 모델 클래스이며, 이를 사용하여 쿼리를 작성할 수 있다.
    - 좀 더 직관적이고 가독성이 좋으며, 세션 관리를 명시적으로 하지 않아도 된다.

### Raw SQL Queries

- SQL 문을 직접 작성하여 데이터베이스에 질의하는 방식
- 복잡한 쿼리에서 ORM을 사용하기 어려운 경우에 유용한 방식

### Stored Procedure 호출

- 일반적으로 프로시저 호출은 ORM으로는 처리하기 어려운 복잡한 데이터베이스 작업에 사용된다.
- `cursor.callproc()`
  ```python
  result = cursor.callproc('SP_GET_USER', [user_id])
  ```
- `session.execute()`
  - 프로시저 호출은 주로 `session.execute()`를 통해 수행된다.
  - **`callproc()`** 메서드는 SQLAlchemy의 **`Session`** 객체에는 포함되어 있지 않다. 따라서 **`Session`** 객체에서 **`callproc()`**를 직접 호출할 수 없기 때문에 SQLAlchemy를 이용하여 프로시저를 호출할 때는 **`execute()`** 메서드를 사용해야 한다.
    ```python
    result = session.execute(text("CALL SP_GET_USER(:param1, :param2)"), param1=value1, param2=value2)
    ```
- 문제
  - SQLAlchemy의 `session.execute()` 함수를 사용해서 다중 결과 집합을 처리하려 했는데, SQLAlchemy에서 직접적으로 제공하지 않는다고 한다.
  - 일반적으로 `다중 결과 집합을 조회`하는 경우, DBAPI 드라이버 기능을 사용하여 처리한다. SQLAlchemy도 DBAPI 드라이버를 활용하며, `connection.execute()` 함수를 통해 다중 결과 집합을 처리할 수 있다.
    - `DBAPI`란, 데이터베이스와 상호 작용하기 위한 프로그래밍 인터페이스를 제공하는 라이브러리나 드라이버의 모음을 말한다. pymysql(MySQL), psycopg1(PostgreSQL), sqlite3(SQLite) 등이 있다.

## Flask-SQLAlchemy를 이용한 CommonDB 클래스

- DB 연결 설정과 트랜잭션 관리를 추상화하고 중복 코드를 줄이기 위해 다음과 같은 팩토리 클래스를 구현해 보았다.

```python
from sqlalchemy import create_engine, text
from app import db

class CommonDB:
    def __init__(self, bind_name):
        if not bind_name:
            bind_name = db.engine.url

        self.session = Session(bind=self.connect)
				self.session.bind = create_engine(bind_name)

    def commit(self):
        try:
            self.session.commit()
        except SQLAlchemyError as e:
            self.session.rollback()
            raise e

    def rollback(self):
        try:
            self.session.rollback()
        except SQLAlchemyError as e:
            raise e
```

```python
# 사용 예제
try:
    BIND = current_app.config["SQLALCHEMY_BINDS"]["TEST"]
    db = CommonDB(BIND)

    # 추가적인 데이터베이스 작업 수행
    # ...

    db.commit()

except Exception as e:
    print(f"An error occurred: {str(e)}")
    logger.exception(traceback.format_exc())
    db.rollback()
    raise
```

- Flask-SQLAlchemy 사용 시 session.close()를 따로 하지 않아도 앱 종료 시 자동으로 세션을 닫아주므로 작성할 필요가 없다.
- 예외 발생 시 데이터베이스 연결이 종료되지 않는다면 데이터베이스 리소스가 계속해서 낭비되는 상황이 발생할 수 있으므로, Flask-SQLAlchemy가 아닌 다른 방식으로 커넥션을 맺었을 시에 finally문에 연결을 종료하는 코드를 추가해 주어야 한다.
- 위 CommonDB를 이용하면 여러 종류의 데이터베이스 드라이버를 사용하는 데에도 문제가 없다. `SQLALCHEMY_BINDS` 설정에 원하는 드라이버를 추가하고 CommonDB 클래스를 사용할 때 해당 드라이버를 지정하면 된다.

## 결론

- 기존에 이미 사용중에 있기도 하며, 추후 다른 데이터베이스 드라이버가 추가될 것을 고려하여 여러 종류의 데이터베이스 드라이버 사용이 가능한 `SQLAlchemy를 활용`하고, pymysql 드라이버를 이용한 연동 코드는 가능한 제거하는 것이 좋을것 같다.
- 데이터베이스 연결 관리와 트랜잭션 관리를 효율적으로 할 수 있는 공통 클래스(CommonDB)를 만들어 사용하면 좋을것 같다.
- 기존 프로젝트에서 try문 내에서 db 연결을 close 하는 코드를 종종 보았는데, 자동으로 세션 관리를 해주는 Flask-SQLAlchemy를 이용한 방식이 아니라 직접 세션을 관리해야 하는 경우, finally문에서 연결을 닫아주도록 변경하는 것이 좋겠다.
- 프로시저를 호출하는 경우와 ORM으로 쿼리를 수행하는 경우를 나누어 상황에 따라 적절한 방식으로 처리하도록 통일하는것이 좋을것 같다.
  - 프로시저를 호출하는 경우
    - 다중 결과 집합을 조회(cursor.nextset())해야 하는 특수한 경우는 DBAPI 드라이버를 이용하되, 그 외의 경우 session.execute()를 이용하여 통일성을 유지하도록 하자.
    - output 파라미터를 조회해야 하는 경우, cursor.call_proc()은 제공하지 않으므로 session.execute() 사용하자.
  - ORM으로 쿼리 수행하는 경우, SQLAlchemy의 ORM 기능을 활용하여 모델을 정의하고 모델에 정의된 데이터베이스 바인딩을 사용하여 처리하도록 한다.
