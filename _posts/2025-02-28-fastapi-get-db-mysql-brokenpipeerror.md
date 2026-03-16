---
layout: post
title: "FastAPI에서 잘못된 get_db 사용법이 초래한 MySQL BrokenPipeError 해결 방안"
date: 2025-02-28
description: "FastAPI로 구축된 서버에서 MySQL BrokenPipeError 오류가 발생한 원인과 제너레이터의 올바른 사용법을 정리한 글입니다."
tags: [PYTHON]
tistory_id: 59
---

# 개요

FastAPI로 구축된 서버에서 "MySQL server has gone away (BrokenPipeError(32, 'Broken pipe'))" 오류가 발생했습니다. 이 오류의 로그를 추적하여 발견한 문제점과 해결 방법을 정리했습니다.

## 문제 배경

다음은 데이터베이스 세션 객체를 생성하고 관리하는 코드입니다. FastAPI 예제와 본 서버에서도 유사하게 구현되었습니다.

```
def get_db():
    session = Session()
    try:
        yield session
    except Exception as e:
		    log.exception("Session Error!!", e)
		    raise e
    finally:
		    log.debug("Session Close")
        session.close()
```

FastAPI의 예제에서도 그렇고, 본 서버에서는 DB에 접근하는 Session 객체를 생성하고 관리하는 함수를

이렇게 정의했습니다.

```
# Controller + Persistence
@api.get("/test")
def test():
	log.debug("API Start")
	db = next(get_db())
	data = db.query(DataModel).all()
	log.debug("API End")
	return data
```

이 코드에서 로그의 출력 순서는 다음과 같이 예상했습니다.

```
1. API Start
2. API End
3. Session Close
```

그러나 실제 실행 결과는 아래와 같이 나타났습니다.

```
1. API Start
2. Session Close
3. API End
```

Session Close가 API End보다 먼저 실행되었습니다. 이는 세션(트랜잭션)에서 오류가 발생했을 때 Exception 블록이 실행되지 않는 문제를 야기합니다. 이미 Finally 블록의 session.close()가 실행되었기 때문입니다.

## 문제 원인

제너레이터를 잘못된 방식으로 호출했기 때문입니다. Python에서 제너레이터 함수는 yield 키워드를 사용하여 값을 하나씩 반환하는 특별한 종류의 이터레이터입니다. 제너레이터를 호출하면 제너레이터 객체가 반환되고, next() 함수를 호출할 때마다 다음 yield 지점까지 실행됩니다.

문제의 원인이 되는 코드는 다음과 같습니다.

```
db = next(get_db())
```

이 코드에서 get_db()는 제너레이터 객체를 반환하고, next()는 첫 번째 yield 값을 가져오는 즉시 finally 블록을 실행하여 세션을 닫습니다.

> Yield expressions are allowed anywhere in a try construct. If the generator is not resumed before it is finalized (by reaching a zero reference count or by being garbage collected), the generator-iterator's close() method will be called, allowing any pending finally clauses to execute.

위 Python 문서에서 finally 블록이 즉시 실행되는 이유를 설명합니다.

## 올바른 사용법

FastAPI에서는 의존성 주입 시스템을 통해 제너레이터를 올바르게 관리할 수 있습니다:

```
@contextlib.contextmanger
def get_db():
    session = Session()
    try:
        yield session
    except Exception as e:
		    log.exception("Session Error!!", e)
		    raise e
    finally:
		    log.debug("Session Close")
        session.close()

# Controller + Persistence
@api.get("/test")
def test():
	log.debug("API Start")
	with get_db() as db:
		data = db.query(DataModel).all()
	log.debug("API End")
	return data
```

Python의 컨텍스트 관리자(contextlib.contextmanager)를 사용하면 제너레이터의 생명주기를 관리하여 함수가 완전히 실행된 후에만 finally 블록이 실행되도록 보장합니다.
