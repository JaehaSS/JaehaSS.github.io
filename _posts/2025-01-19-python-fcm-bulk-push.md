---
layout: post
title: "Python을 활용한 FCM Bulk Push"
date: 2025-01-19
description: "사내에서 FCM 프로젝트를 맡게 되었는데 FastAPI + FCM으로 단일 유저에게 FCM Push하는 예제는 많지만 Bulk Push에 대해서는 한글 예제를 못 찾아서 공유하고자 한다."
tags: [PYTHON]
tistory_id: 48
---

사내에서 FCM 프로젝트를 맡게 되었는데 FastAPI + FCM 으로 단일 유저에게 FCM Push하는 예제는 많이 여기 저기 있지만 Bulk Push에 대해서는 한글로 된 예제를 못 찾아서 공유하고자 한다.

### 공식 문서

공식 문서에서 Bulk Push 보내는 방법에 대해서

![](https://blog.kakaocdn.net/dna/vR4Qz/btsLSxRPcOT/AAAAAAAAAAAAAAAAAAAAAH0WbAIDGja7hq3KeaurogEtUkLo80OCn3IuVV0YI7rV/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=pTdFnn34tknPKfXZnzhbcjoPKh0%3D)

send_multicast Function을 사용하라고 나온다.

바로 예제를 보자

```
from firebase_admin import messaging

def send_bulk_push(token_list : list[str]):
    noti = messaging.Notification(title="Title", body="Body")
    message = messaging.MulticastMessage(
        tokens=token_list
        notification=noti,
    )
    data = messaging.send_multicast(message, dry_run=False)

molk_data = ["token_test1", "token_test2"] * 100  # 유저의 FCM 토큰

send_bulk_push(molk_data)
```

해당 예제는 send_bulk_push 함수가 유저의 FCM 토큰 리스트인 ["token_test1", "token_test2"] * 100 을 한 번에 보낸다

ps. 한 번에 보낼 수 있는 리스트의 크기는 500이하여서 만약 500보다 크면 슬라이스 작업을 해줘서 여러 번 작업해야한다

만약 10만 개 이상을 보낼려면 어떻게 해야할 까??

500개씩 최소 20번을 실행을 해야하는데 거기에 걸리는 시간은??

보낼 때마다 I/O Bound 작업이 발생하는데 최소화 할려면 비동기로 해야하나?? 근데 공식 라이브러리에서 제공하는 함수가 비동기인가??

사내에서 파이썬으로 구축을 하는 데에 있어서 제일 고민이었다. 맨 처음 40만 토큰으로 테스트를 할 때 30분 이상 걸려서 강제 종료 했다. 결국은 Threadpool을 사용하여 2분 30초로 단축을 했다.

```
from firebase_admin import messaging
import concurrent.futures
import os

def send_bulk_push(token_list : list[str]):
    noti = messaging.Notification(title="Title", body="Body")
    message = messaging.MulticastMessage(
        tokens=token_list
        notification=noti,
    )
    data = messaging.send_multicast(message, dry_run=False)

molk_data = ["token_test1", "token_test2"] * 100

with concurrent.futures.ThreadPoolExecutor(
            (os.cpu_count() or 1) + 4
        ) as executor:
            results = list(executor.map(send_bulk_push), molk_data)
```

추가

해당 과정을 성능 개선하기 위해 어떻게 처리했는 지 [개선 과정](https://jaehashin.tistory.com/53) 참고해주세요.
