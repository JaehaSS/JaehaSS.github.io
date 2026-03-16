---
layout: post
title: "FastAPI + Sqlalchemy를 활용한 Pytest( + async)"
date: 2025-01-19
description: "FastAPI와 Sqlalchemy를 사용하여 동기/비동기 Pytest를 작성하고, 중첩 트랜잭션을 활용해 테스트 데이터를 자동 롤백하는 방법을 정리합니다."
tags: [Python]
tistory_id: 51
---

# FastAPI + Sqlalchemy를 활용한 Pytest( + async)

## 개요

FastAPI Pytest를 활용한 유닛 테스트 및 통합 테스트를 정리하고자 합니다. 해당 기능을 활용해 Github CI까지 하는게 목표입니다.

## 간략한 소개

### FastAPI

현대적이고 빠르며(고성능), 파이썬 표준 힌트에 기초한 Python의 API 빌드하기 위한 웹 프레임워크이다. 프레임워크 단에서 비동기 지원을 해준다.

### Pytest

Python에서 널리 사용되는 테스트 프레임워크, 간단하고 확장 가능한 테스트를 작성할 수 있다. 다양한 플러그인과 기능을 제공하여 테스트 작성 및 실행을 편리하게 해줌

### Sqlalchemy

Python에서 SQL 데이터베이스와 상호작용하기 위한 ORM이다. 주로 FastAPI에선 Sqlalchemy 라이브러리를 사용한다.

## 설정

### 라이브러리

```bash
# FastAPI
pip install fastapi
pip install uvicorn

# Sqlalchemy
pip install sqlalchemy
pip install pymysql # Mysql Driver
pip install asyncmy # Mysql Async Driver

# Pytest
pip install pytest
pip install pytest-aio
pip install pytest-dotenv
```

### 폴더 구조

```
.
├── Dockerfile
├── README.md
├── app
│   ├── api
│   │   ├── common
│   │   │   └── dto
│   │   │       └── base_response_dto.py
│   │   ├── dependency.py
│   │   └── v1
│   │       ├── endpoint.py
│   │       └── user
│   │           ├── dto
│   │           │   ├── request_user_profile_dto.py
│   │           │   ├── request_user_remove_data.py
│   │           │   ├── request_user_sign.py
│   │           │   ├── request_user_sign_dto.py
│   │           │   └── request_user_sign_out_dto.py
│   │           ├── endpoint.py
│   │           ├── entity
│   │           │   └── user.py
│   │           ├── repository.py
│   │           ├── service.py
│   │           └── utils.py
│   ├── config
│   │   ├── config.py
│   │   ├── db
│   │   │   ├── database.py
│   │   │   └── time_stamp_mixin.py
│   │   └── redis_config.py
│   ├── main.py
│   └── models
├── db_script
│   ├── product.sql
│   └── user.sql
├── docker-compose.yml
├── env
│   ├── local.env
│   └── test.env
├── requirement.txt
└── tests
    ├── __init__.py
    ├── api
    │   └── test_user.py
    └── conftest.py
```

### DataBase 설정

```python
# config/db/database.py
class DataBaseManager:
    def __init__(self):
        sync_database_url = "url"
        async_database_url = "async_url"

        self.sync_engine = create_engine(
            sync_database_url,
            pool_recycle=3600,
            pool_size=10,
            max_overflow=10,
        )

        self.async_engine = create_async_engine(
            async_database_url,
            pool_recycle=3600,
            pool_size=5,
            max_overflow=5,
        )

        self.sync_session_maker = sessionmaker(
            autocommit=False,
            autoflush=True,
            bind=self.sync_engine,
        )

        self.async_session_maker = async_sessionmaker(
            self.async_engine,
            expire_on_commit=False,
        )

    @contextmanager
    def get_sync_session(self) -> Generator:
        session = self.sync_session_maker()
        try:
            yield session
        except SQLAlchemyError as e:
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=str(e)
            )
        finally:
            session.close()

    @asynccontextmanager
    async def get_async_session(self) -> AsyncGenerator:
        async_session = self.async_session_maker()
        try:
            yield async_session
        except SQLAlchemyError as e:
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=str(e)
            )
        finally:
            await async_session.close()

    @asynccontextmanager
    async def mocking_async_session(self) -> AsyncGenerator:
        async with self.async_engine.connect() as conn:
            await conn.begin_nested()
            async with AsyncSession(
                bind=conn,
                expire_on_commit=False,
            ) as session:
                yield session
            await conn.close()


db_manager = DataBaseManager()

db_with = db_manager.get_sync_session

def db():
    with db_manager.get_sync_session() as session:
        yield session

async_db = (
    db_manager.get_async_session
    if config.ENV != "test"
    else db_manager.mocking_async_session
)
```

**중첩 트랜잭션을 Test에 사용한 이유**

사내에서는 따로 테스트 DB가 없습니다. 테스트 시 SQLite 또는 로컬 테스트 DB 사용 옵션이 있으나, 전자는 MySQL 호환성 문제, 후자는 성능 문제로 배제했습니다.

**중첩 트랜잭션의 이점**

중첩 트랜잭션을 사용하여 API의 DB 쿼리 에러 발생 여부를 체크하고 자동 롤백을 수행합니다.

## Pytest

### Sync Test

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture(scope="module")
def test_app():
    with TestClient(app) as client:
        yield client

# tests/api/test_user.py
from starlette.testclient import TestClient

def test_sign_up(test_app: TestClient):
    response = test_app.post(
        "/v1/user/sign-up",
        json={
            "user_phone": "01012345699",
            "password": "password",
        },
    )
    assert response.json()["code"] == 200
```

이 방식은 테스트 성공을 확인할 수 있지만, DB에 테스트 데이터가 남게 됩니다.

### Sync Test - Nested Transaction

```python
@contextmanager
def get_sync_session(self) -> Generator:
    session = self.sync_session_maker()
    try:
        yield session
    except SQLAlchemyError as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=str(e)
        )
    finally:
        session.close()

# 해당 코드를 다음으로 변경
@contextmanager
def mocking_sync_db():
    with sync_engine.connect() as connection:
        trans = connection.begin()
        with Session(
            bind=connection, join_transaction_mode="create_savepoint"
        ) as session:
            yield session
            session.close()
        trans.rollback()
        connection.close()
```

중첩 트랜잭션 적용 후 DB에 데이터가 남지 않습니다.

### Async Test

Sync와 달리 AsyncTest는 기존 TestClient를 AsyncClient로 변경합니다.

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.fixture(scope="session")
async def test_app():
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://localhost:8000"
    ) as client:
        yield client

# tests/conftest.py
async def test_async_sign_up(test_app: AsyncClient):
    response = await test_app.post(
        "/v1/user/async/sign-up",
        json={
            "user_phone": "01012345699",
            "password": "password",
        },
    )
    assert response.json()["code"] == 200
```

### Async Test - Nested Transaction

```python
# config/db/database.py

# ENV = test 일 때 해당 코드가 실행됨.
@asynccontextmanager
async def mocking_async_session(self) -> AsyncGenerator:
    async with self.async_engine.connect() as conn:
        await conn.begin_nested()
        async with AsyncSession(
            bind=conn,
            expire_on_commit=False,
        ) as session:
            yield session
        await conn.close()

async_db = (
    db_manager.get_async_session
    if config.ENV != "test"
    else db_manager.mocking_async_session
)
```

비동기 테스트에서도 DB에 데이터가 쌓이지 않습니다.

## 정리

해당 글에서는 FastAPI와 Sqlalchemy를 사용하여 동기 및 비동기 데이터베이스 작업을 설정하고 Pytest를 사용하여 테스트를 작성하는 방법을 다루었습니다. Sqlalchemy에서 제공하는 중첩 트랜잭션을 사용해 로직이 DB 단에 질의할 때 에러가 나는지 체크만 하고 데이터를 남기지 않는 방법을 소개했습니다.
