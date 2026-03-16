---
layout: post
title: "Web Architechure 고찰"
date: 2025-01-19
description: "Internal ELB 생성 및 테스트 과정에서 겪은 트러블슈팅을 통해 웹 아키텍처에 대해 다시 고민해보고 정리한 글입니다."
tags: [ARCHITECTURE]
tistory_id: 49
---

# Web Architecture Error 슈팅

안녕하세요. 이번에 Internal(내부) ELB 생성하고 테스트, 적용하는 과정 중에 어떤 트러블을 겪었는지 적어볼려고 합니다. 해당 트러블로 인해 웹 아키텍처에 대해서 다시 고민해보는 시간을 가졌습니다. 혼자 고민하면서 저만의 결론을 내린 내용을 토대로 글을 작성합니다.

# Web Architecture

### AWS에서 제공하는 Web Architecture

![](https://blog.kakaocdn.net/dna/caid9j/btsLSBmAlk3/AAAAAAAAAAAAAAAAAAAAAOJgZBfdIKGKS_T-firWgJE84bsqDYWg5ZeFW1DFSRx0/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=2QvI2HgkLwaaZPI4bGuFk1Nm5j8%3D)

### 필자가 하는 Web Architecture

![](https://blog.kakaocdn.net/dna/bmupHY/btsLSS9okYc/AAAAAAAAAAAAAAAAAAAAAHCKld913aXJ-G8BgFFc2zxeVaioly0hn6VpkOOHq6QL/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=b9RzauviZmasVqlFpUJ6ZtpcYxo%3D)

# AWS Service

해당 그림에서 제공해주는 용어들 중에서 모르는 사람이 있을 수도 있어 간단하게 설명을 하고자 합니다.

## VPC

Virtual Private Cloud, 이름 그대로 독립된 가상 Cloud 이다. VPC는 RFC1918이라는 사설 IP 대역에 맞추어 설계해야함.

## Subnet

VPC의 IP 주소 범위이다. Subnet은 단일 가용영역(Availability Zone= AZ)에 상주함.

CIDR의 주소 범위를 Subnet으로 맛깔나게 표현한 거다.

### Private 와 Public을 나누는 기준이 뭔가요??

인터넷에 접근할 수 있냐 없냐로 구분함. 즉 Internet Gateway를 연결했냐 안했냐로 구분하면 됩니다

### Internet Gateway, NAT Gateway

Internet Gateway는 인터넷과 통신하기 위한 경로이다.

NAT Gateway 의 NAT은 Network Address Translation의 약자이다

패킷의 IP 주소, 포트 등을 변환하는 기술을 의미 함

Private Subnet에서 외부 Public Networ와 통신하기 위해서 사용한다.

![](https://blog.kakaocdn.net/dna/Kup59/btsLRNt8wV1/AAAAAAAAAAAAAAAAAAAAAFcfMV3HfkeFJmZHTPg_FJmJOJsQFgtIXXfs_KysZzYZ/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=zfHg%2FoD4SHhvzu68nUo%2F37Oj6CU%3D)

해당 그림처럼 Private 인스턴스는 Public IP이 아닌 Private IP만 가지고 있음

ex) Private Subnet에서 Slack 알람 보내기

1. A Instance Slack API를 보냄
2. NAT Gateway에 의해 Elastic Public IP에 A 트래픽 랩핑됨
3. 랩핑된 트래픽이 Internet(Slack API) 통신
4. 통신한 결과 값이 NAT Gateyay 도착
5. Nat Gateway에 의해 랩핑된 트래픽 분해
6. A instance 에 트래픽 도착

# 해당 Architecture 에 따른 Test

## AWS Architecture

이 부분은 넘어가겠습니다.

## 필자 Architecure

여러 Test 케이스를 할려고 합니다

Public EC2

- Curl
- Next js 서버
  - Server Component
  - Client Component

### 1. Curl Test (실행 환경 : 해당 서버 터미널)

![](https://blog.kakaocdn.net/dna/bCJZ8h/btsLSBmAlkZ/AAAAAAAAAAAAAAAAAAAAAPBMbA-pyD0ZxVtDOVquxyv25UaTa70i0EjpxKBLMlSp/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=HIxbv043Yitf%2BXxQO9h7sWJA4JM%3D)

- CluodWatch

![](https://blog.kakaocdn.net/dna/dxfexl/btsLSNtpu0T/AAAAAAAAAAAAAAAAAAAAAKmttiHQKzje0fQ2DMPLzX-ZQkGVksuJjRhQpS7O8v-6/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=yIgBvEGf8Vdi4HEzxICX8XbxxnQ%3D)

True 확인, Log 확인

### 2. Next JS 서버(실행 환경 : 필자 컴퓨터 브라우저)

- Client Component (Axios)

```
"use client" // directive for client components
import axios from 'axios';

const MyComponent = () => {
  const jaeha = axios.get("http://internal-jaeha-test-1659558992.ap-northeast-2.elb.amazonaws.com/health")
  console.log(jaeha)
  return <div>Hello</div>
}
```

- CloudWatch

![](https://blog.kakaocdn.net/dna/b2IJIP/btsLTsI40Tp/AAAAAAAAAAAAAAAAAAAAAGCXrA2_NpnYpoDMcPwYX3BDANI3qI6CM9JhxpyYsNUF/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=0aK6ORayB2Ymjq99jaayq%2BnX41w%3D)

- Client Component (fetch)

```
const getData = async () => {
  const gg = await fetch("http://internal-jaeha-test-1659558992.ap-northeast-2.elb.amazonaws.com/health")
  const jaeha_json = await gg.json();
  console.log(jaeha_json)
  return <div>jaeasdha</div>
}

export default MyComponent;
```

- CloudWatch

Timeout Error 에러 발생, Log 확인

- Server Component (fetch)

```
const fetchPosts = async () => {
  const response = await fetch("http://internal-jaeha-test-1659558992.ap-northeast-2.elb.amazonaws.com/health", {
    cache: "no-store",
  });
  return await response.json();
};

const Posts = async () => {
  const data = await fetchPosts();
  return JSON.stringify(data)
};

export default Posts;
```

- CloudWatch

True 확인, Log 확인

### Test 도출할 수 있는 결론

1. 모든 테스트 같은 경우에는 Private Subnet에 신호를 보냄
2. Client Component 같은 경우에는 Request는 가지만 Response 안 옴

### 생겨나는 Why??

- 같은 HTTP 프로토콜인데 왜 서로 결과 값이 다를 까?
- CORS 에러인가?

### 필자가 내린 결론

![](https://blog.kakaocdn.net/dna/PDUZM/btsLTwdD3ba/AAAAAAAAAAAAAAAAAAAAALdAlOfQq8qUBVV8NyB6W5ylbfKlTDWt07Hc1hZuuoOw/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=85WHgOnzy4m9MzJDrv%2FB5kfwqRQ%3D)

트래픽이 나가는 경로를 중심으로 생각을 해봤다.

- Curl: Public EC2 → Curl → Request → Internal ELB → Request → Private EC2 → Response → Internal ELB → Response → Public EC2
- Nextjs Client Component: Web Browser → Request → Request → Internal ELB → Request → Private EC2 → Response → NAT Gateway → Response → Web Browser
- Nextjs Server Component: Web Browser 접속 → Nextjs node server → Request → Internal ELB → Request → Private EC2 → Response → Nextjs node server → Response → Web Browser

해당 경로를 정리해서 보면 잘못된 것이 보인다

- Client Component 같은 경우에는 왜 NAT Gateway인가??
  - 필자는 Private Subnet은 인터넷과 통신을 할려면 NAT Gateway가 필요해서 이 쪽 부분을 들렀다 나오는 줄 알았음

잘못된 부분을 정리하자면

Nextjs Client Component

1. Web Browser → Request → Internal ELB → Request → Private EC2 → Response → Internal ELB → Response → Web Browser

Internal ELB 에선 인터넷에 트래픽을 못 보낸다. 하지만 공용 IP 가 존재하지 않기 떄문에 Response 트래픽을 못 보낸다. 고로 TimeOut이 발생함
