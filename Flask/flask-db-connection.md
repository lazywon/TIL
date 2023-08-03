# Flask에서 데이터베이스 연동하기

기존 Flask 프로젝트를 리팩토링하던 중 MySQL DB 연동 시, pymysql을 이용한 코드와 SQLAlchemy를 이용한 코드가 각각 따로 존재하는 것을 발견했다. 어느 상황에서 어떤 방식을 이용하는 것인지 파악하고 둘의 차이점과 어떤 방식을 이용하는 것이 더 나은 방법인지 알아보고자 한다. 더불어, 다양한 방식으로 산재되어 있는 데이터베이스 관련 처리 코드를 상황에 따라 통일할 수 있도록 리팩토링을 목표로 한다.

## 데이터베이스 연동 및 작업 처리 방식

프로젝트 내에서는 MySQL과 연동하고 있는데, 두가지 방식이 코드에 산재되어 있다.

1. 직접적으로 `pymysql 드라이버`를 사용하여 데이터베이스 커서(Cursor)객체를 이용하여 쿼리를 다루는 방식
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

- ORM(객체-관계 매핑) 기능과 데이터베이스 추상화 도구를 제공하는 라이브러리
- 여러 종류의 데이터베이스 시스템과 호환되며, pymysql을 포함하여 다양한 드라이버를 지원한다. 그렇기 때문에 데이터베이스 종류에 따라 다른 드라이버를 사용하여 여러 데이터베이스 시스템과 호환될 수 있다.
  - ex)
    ```python
    SQLALCHEMY_BINDS = {
        'mysql_db': 'mysql+pymysql://user1:password1@localhost/db1',
        'postgres_db': 'postgresql+psycopg2://user2:password2@localhost/db2'
    }
    ```
- SQLAlchemy를 사용하면 데이터베이스 연결, 쿼리 작성, 테이블 생성, 데이터 조작 등을 쉽게 할 수 있다.

### pymysql

- MySQL 데이터베이스에 접속하기 위한 파이썬 드라이버
- SQLAlchemy는 데이터베이스와 상호작용하기 위해 내부적으로 pymysql과 같은 드라이버를 사용한다.
- 즉, SQLAlchemy를 사용할 때에도 pymysql 라이브러리가 필요하며, 일반적으로 SQLAlchemy를 사용하면서 pymysql을 따로 사용할 필요는 없다.

## SQLAlchemy에서 데이터베이스 작업을 처리하는 방식

### SQLAlchemy ORM (Object-Relational Mapping)

- 데이터베이스 테이블과 Python 클래스를 매핑하여 객체로 데이터를 다루는 방식
- ORM을 사용하면 SQL을 직접 작성하지 않고도 데이터베이스 작업을 수행할 수 있어 코드 가독성과 유지보수성 높일 수 있다.
- ORM을 통해 정의한 데이터베이스 model을 가지고 Query 객체를 사용하여 데이터베이스 쿼리를 작성하고, 필터 적용, 결과 정렬, 조인 등을 수행
- 데이터베이스 바인딩은 모델의 **bind_key** 속성을 사용하여 정의하므로, 각 모델을 특정 데이터베이스에 바인딩할 수 있다. 기본적으로 **bind_key**를 정의하지 않은 모델은 `SQLALCHEMY_DATABASE_URI`에 설정된 기본 데이터베이스와 연결된다.
  - ex)
    ```python
    class User(db.Model):
    __bind_key__ = 'db1'  # 'db1'에 바인딩
    # 모델 정의
    ```
  - 기본적으로 config.py에 설정된 `SQLALCHEMY_DATABASE_URI` 값에 따라 설정한 디폴트 데이터베이스에 연결되지만,
  - ORM을 통해 정의한 Model을 사용하여 쿼리 수행 후 `db.session.commit()` 을 호출하면 디폴트로 설정된 데이터베이스가 아닌, 해당 모델을 통해 바인딩 된 데이터베이스 연결 정보에 변경 사항이 커밋된다.
    - 다시말해, 각 모델 클래스에 bind_key가 정의되어 있다면, 해당 데이터베이스와 연결된다.
- SQLAlchemy의 쿼리를 작성하는 두 가지 방법
  - SQLAlchemy의 db.session을 사용하여 쿼리를 작성하는 방법
    ```python
    db.session.query(User).filter()
    ```
    - db.session은 SQLAlchemy의 세션 객체를 의미하며, 이 세션은 기본적으로 SQLALCHEMY_DATABASE_URI에 설정한 데이터베이스와 연결된다.
    - 세션을 직접 사용하기 때문에 세션 관리에 더 많은 제어를 할 수 있다.
  - SQLAlchemy의 모델 클래스를 사용하여 쿼리를 작성하는 방법
    ```python
    User.query.filter()
    ```
    - User는 Base 클래스를 상속받은 SQLAlchemy 모델 클래스이며, 이를 사용하여 쿼리를 작성할 수 있다.
    - 좀 더 직관적이고 가독성이 좋으며, 세션 관리를 명시적으로 하지 않아도 된다.

### Raw SQL Queries

- SQL 문을 직접 작성하여 데이터베이스에 질의하는 방식
- 복잡한 쿼리에서 ORM을 사용하기 어려운 경우에 유용한 방식

### Stored Procedure 호출

- `session.execute()` 메서드를 사용하여 직접 SQL문을 작성하여 Stored Procedure를 호출하는 방식
- 일반적으로 프로시저 호출은 ORM으로는 처리하기 어려운 복잡한 데이터베이스 작업에 사용된다.

## SQLAlchemy의 `Session` 객체를 이용한 db 작업 처리 예제

```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

class CommonDB:
    def __init__(self, bind_name):
        if not bind_name:
            raise ValueError("Invalid bind_name. Please provide a valid database connection string.")

        self.engine = create_engine(bind_name)
        self.connect = self.engine.connect()
        self.session = Session(bind=self.connect)

    def close(self):
        try:
            self.session.close()
            self.connect.close()
        except SQLAlchemyError as e:
            raise e

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
# example
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

finally:
    db.close()

```

- 위 CommonDB를 이용하면 여러 종류의 데이터베이스 드라이버를 사용하는 데에도 문제가 없다.
- `SQLALCHEMY_BINDS` 설정에 원하는 드라이버를 추가하고 CommonDB 클래스를 사용할 때 해당 드라이버를 지정하면 됨.

## 결론

- 기존에 이미 사용중에 있기도 하며, 추후 다른 데이터베이스 드라이버가 추가될 것을 고려하여 여러 종류의 데이터베이스 드라이버 사용이 가능한 SQLAlchemy를 활용하고, pymysql 드라이버를 이용한 연동 코드는 제거하는 것이 좋을것 같다.
- 데이터베이스 연결 관리와 트랜잭션 관리를 효율적으로 할 수 있는 공통 클래스(CommonDB)를 만들어 사용하면 좋을것 같다.
- 프로시저를 호출하는 경우와 ORM으로 쿼리를 수행하는 경우를 나누어 상황에 따라 적절한 방식으로 처리하도록 통일하는것이 좋을것 같다.
  - 프로시저를 호출하는 경우, SQLAlchemy의 Session 객체를 이용한 CommonDB를 활용하여 데이터베이스 바인딩과 트랜잭션 처리를 간편하게 한다.
  - ORM으로 쿼리 수행하는 경우, SQLAlchemy의 ORM 기능을 활용하여 모델을 정의하고 모델에 정의된 데이터베이스 바인딩을 사용하여 처리하도록 한다.
