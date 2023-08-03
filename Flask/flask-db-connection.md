# Flask에서 데이터베이스 연동하기

기존 Flask 프로젝트를 리팩토링하던 중 MySQL DB 연동 시, pymysql을 이용한 코드와 SQLAlchemy를 이용한 코드가 각각 따로 있어 어느 상황에서 어떤 방식을 이용하는 것인지 파악하고 둘의 차이점에 대해 알아보고자 한다.

## 데이터베이스 연동 및 작업 처리 방식

프로젝트 내에 두가지 방식이 있다.

1. 직접적으로 pymysql 드라이버를 사용하여 데이터베이스 커서(Cursor)객체를 이용하여 쿼리를 다루는 방식
   - 데이터베이스 커서를 이용한 방식은 더 저수준의 데이터베이스 작업을 위해 필요할 때에 선택한다.
   - 직접적으로 데이터베이스 드라이버를 사용하기 때문에, SQLAlchemy ORM 기능을 사용하지 않는다.
   - 대신 SQL문을 직접 작성하여 데이터베이스 쿼리를 수행한다.
   - 데이터베이스 연결 관리
     - 데이터베이스 커서를 사용하는 방식은 직접적으로 데이터베이스 연결을 다루기 때문에, 연결 관리에 대한 책임이 프로그래머에게 있다. 이로 인해 연결 관리를 제대로 처리하지 않으면 불필요한 자원 소모가 발생할 수 있다.
2. SQLAlchemy 세션과 ORM 기능을 사용하여 처리하는 방식
   - 일반적으로 데이터베이스 작업을 더 편리하고 가독성 좋게 처리하기 위해서는 SQLAlchemy를 사용한다.
   - 데이터베이스 연결 관리
     - SQLAlchemy 세션을 사용하는 방식은 SQLAlchemy에 내장된 연결 관리 기능을 활용하여 데이터베이스 연결을 효율적으로 관리한다. 세션은 필요할 때 연결을 생성하고, 작업이 끝나면 연결을 자동으로 반환한다.

### SQLAlchemy

- ORM(객체-관계 매핑) 기능과 데이터베이스 추상화 도구를 제공하는 라이브러리
- 여러 종류의 데이터베이스 시스템과 호환되며, pymysql을 포함하여 다양한 드라이버를 지원한다. 그렇기 때문에 데이터베이스 종류에 따라 다른 드라이버를 사용하여 여러 데이터베이스 시스템과 호환될 수 있다.
- SQLAlchemy를 사용하면 데이터베이스 연결, 쿼리 작성, 테이블 생성, 데이터 조작 등을 쉽게 할 수 있다.

### pymysql

- MySQL 데이터베이스에 접속하기 위한 파이썬 드라이버
- SQLAlchemy는 데이터베이스와 상호작용하기 위해 내부적으로 pymysql과 같은 드라이버를 사용한다.
- 즉, SQLAlchemy를 사용할 때에도 pymysql 라이브러리가 필요하며, 일반적으로 SQLAlchemy를 사용하면서 pymysql을 따로 사용할 필요는 없다.

## SQLAlchemy에서 데이터베이스 작업을 처리하는 방식

- ORM (Object-Relational Mapping)
  - 데이터베이스 테이블과 Python 클래스를 매핑하여 객체로 데이터를 다루는 방식
  - ORM을 사용하면 SQL을 직접 작성하지 않고도 데이터베이스 작업을 수행할 수 있어 코드 가독성과 유지보수성 높일 수 있다.
  - ORM을 통해 정의한 model을 가지고 Query 객체를 사용하여 데이터베이스 쿼리를 작성하고, 필터 적용, 결과 정렬, 조인 등을 수행할 수 있다.
- Raw SQL Queries
  - SQL 문을 직접 작성하여 데이터베이스에 질의하는 방식
  - 복잡한 쿼리에서 ORM을 사용하기 어려운 경우에 유용한 방식
- Stored Procedure 호출
  - `session.execute()` 메서드를 사용하여 직접 SQL문을 작성하여 Stored Procedure를 호출하는 방식

## 결론

- 굳이 기존에 SQLAlchemy를 사용하면서 pymysql로 데이터베이스 커서 객체를 통해 db 작업 처리하는 코드를 작성할 필요가 없다고 판단
- SQLAlchemy로 통일하고, db bind 연결 설정과 트랜잭션 관리를 효율적으로 할 수 있는 공통 클래스를 만들어 사용하면 좋을것 같다.

### SQLAlchemy를 이용한 db 작업 처리 예제

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
