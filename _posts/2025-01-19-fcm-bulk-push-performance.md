---
layout: post
title: "FCM 서버 Bulk Push 성능 개선 과정(+FastAPI, HTTP v1 API)"
date: 2025-01-19
description: "FCM Bulk Push에서 Thread 과다 생산과 Memory Leak 문제를 Lambda와 비동기 처리로 해결하여 70분에서 4분으로 성능을 개선한 과정을 공유합니다."
tags: [PYTHON]
tistory_id: 53
---

# FCM 서버 Bulk Push 성능 개선 과정(+FastAPI, HTTP v1 API)

## 개요

현재 회사에서는 중,고등학생들 내신을 위한 문제 풀이 앱을 운영하고 있습니다. 해당 앱에서 유저들에게 FCM을 개인 맞춤화 메시지로 보내는 기획이 생겨 급하게 FastAPI 프레임워크를 사용해 만들었습니다. 하지만 시간이 점점 지날수록 유저들에게 보낼 메시지 수가 많아지고 점점 보내는데 오래 걸려 Timeout이 발생했고 Request가 비정상으로 끊겨 Memory Leak 현상이 발생했습니다.

다음과 같이 정리해서 설명하겠습니다:

1. 문제에 대한 설명 (What)
2. 문제의 원인 (Why)
3. 어떻게 문제를 풀었는지 (How)
4. 구현 및 결과 (Implement and Result)

## 1. 문제에 대한 설명(What)

유저들에게 개인 맞춤화 메시지를 보낼 때 처리하는 로직:

```python
# 보내고 싶은 Target 에 대한 User Data 가져온다
target_user_data = user_repo.find(target)

# 데이터 전처리 후 list로 형 변환
df = pd.DataFrame(....)
send_data : list = [...]

# 전처리한 데이터를 Firebase SDK를 사용해 Send
def send_fcm_by_token(data):
  noti = messaging.Notification(title=..., body=..., image=...)

  message = messaging.MulticastMessage(
      tokens=...,
      notification=noti,
      fcm_options=messaging.FCMOptions(analytics_label=...),
  )
  data = messaging.send_multicast(message, dry_run=...)
  return data

with concurrent.futures.ThreadPoolExecutor(
    max_workes= (os.cpu_count() or 1 ) +2
) as executor:
    results = list(executor.map(self.send_fcm_by_token, send_data))

# 보낸 결과 값을 Slack 전송
failure_count = 0
success_count = 0
if results.__iter__:
    for result in results:
        failure_count += result.failure_count
        success_count += result.success_count
send_slack(f"FCM Send Success: {success_count}, Failure: {failure_count}")
```

## 2. 문제에 대한 원인(Why)

### 1. Send 하는 과정 중에 발생하는 CPU 사용율 증가

**원인:**

현재 구조가 ThreadPool을 열어 함수를 실행하고 해당 함수(`send_each_for_multicast`)는 또 ThreadPool을 열어 실행합니다. 이렇게 한 이유는 Request가 Sync I/O이기 때문입니다.

Firebase SDK에서 제공하는 `send_each_for_multicast` 함수의 소스 코드:

```python
def send_multicast(multicast_message, dry_run=False, app=None):
    messages = [Message(
        data=multicast_message.data,
        notification=multicast_message.notification,
        android=multicast_message.android,
        webpush=multicast_message.webpush,
        apns=multicast_message.apns,
        fcm_options=multicast_message.fcm_options,
        token=token
    ) for token in multicast_message.tokens]
    return _get_messaging_service(app).send_each(messages, dry_run)

def send_each(self, messages, dry_run=False):
    """Sends the given messages to FCM via the FCM v1 API."""
    if not isinstance(messages, list):
        raise ValueError('messages must be a list of messaging.Message instances.')
    if len(messages) > 500:
        raise ValueError('messages must not contain more than 500 elements.')

    def send_data(data):
        try:
            resp = self._client.body(
                'post',
                url=self._fcm_url,
                headers=self._fcm_headers,
                json=data)
        except requests.exceptions.RequestException as exception:
            return SendResponse(resp=None, exception=self._handle_fcm_error(exception))
        else:
            return SendResponse(resp, exception=None)

    message_data = [self._message_data(message, dry_run) for message in messages]
    try:
        with concurrent.futures.ThreadPoolExecutor(max_workers=len(message_data)) as executor:
            responses = [resp for resp in executor.map(send_data, message_data)]
            return BatchResponse(responses)
    except Exception as error:
        raise exceptions.UnknownError(
            message='Unknown error while making remote service calls: {0}'.format(error),
            cause=error)
```

주요 문제 부분:

```python
concurrent.futures.ThreadPoolExecutor(max_workers=len(message_data)) as executor:
```

ThreadPool의 Max Thread 개수는 보낼 메시지의 양입니다. 30만 개의 List를 500개씩 Chunk 하면 Thread 개수만 500개씩 생성됩니다.

### 2. Memory Leak

![메모리 사용 패턴](https://blog.kakaocdn.net/dna/cFemZ8/btsLTH7bUvG/AAAAAAAAAAAAAAAAAAAAAMV__vf5UW2OFeNjNSjLjtxUgoVq3dQ3Vjp9SdlTC_zk/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=fFS3icRFezb91nnw85KiTps1Q0c%3D)

- 주황색: Max Memory
- 초록색: Min Memory
- 파란색: Avg Memory

FCM 서버는 ECS 서비스에서 운영되고 있습니다. 메모리가 점점 차올라서 MemoryOverFlow가 발생하여 ECS에서 컨테이너를 다시 띄우는 현상이 발생합니다.

AWS ELB의 Request Timeout 설정으로 인해, Target이 적으면 Max Timeout 내에 보낼 수 있지만 Target이 많을수록 1시간을 넘어가는 경우도 있습니다. Request Timeout이 발생하면 비정상적인 Connection Loss로 인해 TCP Socket이 제대로 Close되지 않아 Memory Leak이 발생합니다.

## 3. 어떻게 문제를 풀었는지(How)

문제의 원인 정리:

- Thread 과다 생산으로 인한 CPU 사용율 증가
- Socket Loss로 인한 Memory Leak 발생

### Thread 과다 생산으로 인한 CPU 사용율 증가

1. `send_each_for_multicast` 함수 사용 제외
2. Python Event Loop를 사용하여 Thread 1개로 처리

### Socket Loss로 인한 Memory Leak 발생

1. Request Timeout이 발생하기 전에 처리 완료
2. Background 또는 Celery, Lambda 같은 외부 작업자에 일임

## 4. 구현 및 결과(Implement and Result)

최종 아키텍처: FCM 서버에서 모든 것을 처리하는 것이 아닌 다음과 같이 구성:

**FCM 서버 데이터 전처리 -> Request -> Lambda -> FCM Send -> FCM 서버 결과 처리**

### FCM 서버

**AS-IS:**

```python
def send_fcm_by_token(data):
  noti = messaging.Notification(title=..., body=..., image=...)

  message = messaging.MulticastMessage(
      tokens=...,
      notification=noti,
      fcm_options=messaging.FCMOptions(analytics_label=...),
  )
  data = messaging.send_multicast(message, dry_run=...)
  return data

with concurrent.futures.ThreadPoolExecutor(
    max_workes= (os.cpu_count() or 1 ) +2
) as executor:
    results = list(executor.map(self.send_fcm_by_token, send_data))
```

**TO-BE:**

```python
def find_optimal_split(self, lst, split_number):
    '''
    Lambda Request Body Max Size : 6MB로 인해
    Request Data를 5MB 단위로 Chunk함
    '''
    max_size_mb = 5

    def spliter():
        nonlocal lst
        nonlocal split_number

        splits = np.array_split(lst, split_number)
        first_split_size = sys.getsizeof(splits[0]) / 1024 / 1024

        if first_split_size > max_size_mb:
            split_number += 10
            return spliter(split_number)
        else:
            return splits

    return spliter()

def send_to_lambda(self, data):
    client = config.lambda_client
    response = client.invoke(
        FunctionName=lambda_send_fcm,
        Payload=json.dumps({"data": data})
    )
    return response

def send_fcm_by_token(
    self,
    target,
    data: list,
    cnt_send_fcm_data,
):
    data = self.list_append(data)
    data = self.find_optimal_split(data, 50)

    def temp_func(jobs):
        jobs = [list(job) for job in jobs]
        self.send_to_lambda(jobs)

    with concurrent.futures.ThreadPoolExecutor(
        max_workers=(os.cpu_count() or 1) + 3
    ) as executor:
        executor.map(temp_func, data)
```

### 람다

```python
async def post_data(session, headers, url, data, semaphore):
    async with semaphore:
        async with session.post(url, headers=headers, json=data) as response:
            return await response.text()

def list_convert_fcm_message(datas):
    return [
        {
            "validate_only": data[5],
            "message": {
                "notification": {
                    "title": data[1],
                    "body": data[2],
                },
                "fcm_options": {"analytics_label": data[3]},
                "token": data[0],
            },
        }
        for data in datas
    ]

async def main(event):
    FCM_URL = "https://fcm.googleapis.com/v1/projects/{Project Name}/messages:send"
    cred = credentials.Certificate("./firebase.json")

    headers = {
        "Authorization": "Bearer " + cred.get_access_token().access_token,
        "Content-Type": "application/json; UTF-8",
    }

    data = event["data"]
    data = list_convert_fcm_message(data)
    data_cnt = len(data)
    urls = [FCM_URL] * data_cnt
    total_request = data_cnt
    batch_size = 15_000
    semaphore = asyncio.Semaphore(batch_size)

    async with aiohttp.ClientSession() as session:
        tasks = []
        try:
            for i in range(total_request):
                task = post_data(session, headers, urls[i], data[i], semaphore)
                tasks.append(task)
                if len(tasks) >= batch_size:
                    await asyncio.gather(*tasks)
                    tasks = []
            if tasks:
                await asyncio.gather(*tasks)
        except Exception as e:
            print(e)

def lambda_handler(event, context):
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main(event))
    return {"statusCode": 200}
```

### Semaphore 사용한 이유

Semaphore를 설정하지 않으면 List의 길이만큼 데이터를 보냅니다. 예를 들어 등록된 Task가 20,000개라면 동시에 20,000개가 실행되면 연결할 수 있는 Port의 수가 부족하여 에러가 발생합니다.

Semaphore 설정으로 동시 실행되는 Task 수가 제한되어 Port 개수 문제가 해결됩니다.

## 정리

Token 약 40만개를 보낼 때 성능:

- **AS-IS**: 70분
- **TO-BE**: 4분

10배~20배 정도 성능 개선을 달성했습니다.

성능 개선 과정에서 Semaphore, TCP Socket, Buffer Size, Port 개수 등 책으로만 공부했던 개념들을 실제 코드에 구현하면서 그 의미를 더욱 깊이 있게 이해할 수 있었습니다.

### 추가 고려사항

면접에서 해당 이슈 처리 시 단기간에 처리량을 이렇게 늘리면 서버 과부하가 발생하지 않냐는 질문을 받았습니다. 이 때는 개발자 관점에서 성능 개선에만 집중했던 것으로, 다른 사이드 이펙트를 고려하지 못했습니다.
