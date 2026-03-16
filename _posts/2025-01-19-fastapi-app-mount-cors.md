---
layout: post
title: "FastAPI에서 APP Mount + CORSMiddleware"
date: 2025-01-19
description: "FastAPI에서 App Mount 시 CORSMiddleware가 기대대로 동작하지 않는 문제를 분석하고, Middleware 등록/실행 순서와 해결 방법을 정리합니다."
tags: [PYTHON]
tistory_id: 54
---

# FastAPI에서 APP Mount + CORSMiddleware

## FastAPI App Mount CORS

### 개요

최근 QB 개발 서버에 Prometheus Client 기능을 붙여 HTTP 트래픽 횟수, API 호출 횟수 등 데이터 추적을 할려고 한다. Prometheus 서버는 QB 개발 서버의 특정 API 주소(/metrics)를 호출해 API 데이터를 파싱 후 Prometheus 서버에 저장한다. 해당 metrics 주소를 Origin을 적용해 private하게 접근할려고 한다.

### 문제 사항

```python
# starlette/aplications.py
class Starlette:
    def add_middleware(
        self,
        middleware_class: type[_MiddlewareClass[P]],
        *args: P.args,
        **kwargs: P.kwargs,
    ) -> None:
        if self.middleware_stack is not None:
            raise RuntimeError("Cannot add middleware after an application has started")
        self.user_middleware.insert(0, Middleware(middleware_class, *args, **kwargs))
```

```python
# ./test_main.py
app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://main.co.kr"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(LoggiMiddleware)
app.add_middleware(MetricsMiddleware)

class CORSMiddlewareTest(CORSMiddleware): ...

metrics_app = FastAPI()

metrics_app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://metrics.co.kr"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.mount("/metrics", metrics_app)

@metrics_app.get("/")
def get_prometheus_metrics():
    return "Test"
```

해당 구조에서 http://localhost:3000 서버에서 ~/metrics API를 요청했으면 CORS 에러가 발생해야 하지만, CORS 에러가 발생하지 않고 정상적으로 데이터를 가져온다.

### FastAPI Middleware 등록 순서

코드에서 middleware를 선언한 순서대로 `user_middleware`에 insert(0, ...)으로 추가된다. 따라서 등록 순서는 선언 순서의 역순이다.

### FastAPI Middleware 실행 순서

선언한 순서가 아닌 역순으로 호출된다. 다만 starlette 프레임워크에서는 선언한 순서대로 호출된다고 한다.

참조: https://github.com/encode/starlette/issues/479

### CORSMiddleware 분석

```python
# starlette/middleware/cors
class CORSMiddleware:
    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        method = scope["method"]
        headers = Headers(scope=scope)
        origin = headers.get("origin")
        if origin is None:
            await self.app(scope, receive, send)
            return
        if method == "OPTIONS" and "access-control-request-method" in headers:
            response = self.preflight_response(request_headers=headers)
            await response(scope, receive, send)
            return
        await self.simple_response(scope, receive, send, request_headers=headers)

    async def simple_response(
        self, scope: Scope, receive: Receive, send: Send, request_headers: Headers
    ) -> None:
        send = functools.partial(self.send, send=send, request_headers=request_headers)
        await self.app(scope, receive, send)

    async def send(
        self, message: Message, send: Send, request_headers: Headers
    ) -> None:
        if message["type"] != "http.response.start":
            await send(message)
            return

        message.setdefault("headers", [])
        headers = MutableHeaders(scope=message)
        headers.update(self.simple_headers)
        origin = request_headers["Origin"]
        has_cookie = "cookie" in request_headers

        if self.allow_all_origins and has_cookie:
            self.allow_explicit_origin(headers, origin)

        elif not self.allow_all_origins and self.is_allowed_origin(origin=origin):
            self.allow_explicit_origin(headers, origin)

        await send(message)
```

## 문제 개선

### 1. CORSMiddleware 단일로 처리

CORSMiddleware가 여러 개 붙어도 결국에는 기본 앱인 app의 CORSMiddleware 정책을 따르게 된다. metrics_app의 CORSMiddleware를 삭제한다.

```python
# ./test_main.py

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://main.co.kr"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(LoggiMiddleware)
app.add_middleware(MetricsMiddleware)

class CORSMiddlewareTest(CORSMiddleware): ...

metrics_app = FastAPI()

app.mount("/metrics", metrics_app)

@metrics_app.get("/")
def get_prometheus_metrics():
    return "Test"
```

### 2. Mount -> API Router로 개선

```python
# ./test_main.py

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://main.co.kr"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(LoggiMiddleware)
app.add_middleware(MetricsMiddleware)

@app.get("/metrics")
def get_prometheus_metrics():
    return "Test"
```

결국에는 ~/metrics을 사용하는 거니 APIRouter.get("/metrics")로 수정한다. 하지만 ~/metrics에 특별한 정책을 사용하고 싶다면 APIRouter.middleware 기능을 사용하면 된다.

참조: https://github.com/fastapi/fastapi/discussions/7691
