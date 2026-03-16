---
layout: post
title: "요즘 핫한 Claude Desktop에 MCP 적용하기"
date: 2025-03-26
description: "Claude Desktop에 MCP(Model Context Protocol)를 적용하는 방법과 command not found 에러 해결 방법을 정리한 글입니다."
tags: [ai]
tistory_id: 60
---

# 요즘 핫한 Claude Desktop에 MCP 적용하기(+트러블 슈팅)

# MCP란?

Model Context Protocol 의 약자이다.

모델 컨텍스트 프로토콜은 개발자가 데이터 소스와 AI 기반 도구 간에 안전한 양방향 연결을 구축할 수 있도록 하는 개방형 표준

### 주요 구성 요소

- 모델 컨텍스트 프로토콜 [사양 및 SDK](https://github.com/modelcontextprotocol)
- [Claude Desktop 앱](https://claude.ai/download) 에서 로컬 MCP 서버 지원
- MCP 서버의 오픈 [소스 저장소](https://github.com/modelcontextprotocol/servers)

## Claude DeskTop + MCP 적용

### 1. Claude Desktop 앱 다운로드

[https://claude.ai/download](https://claude.ai/download)

### 2. 앱 메인 화면 진입

![](https://blog.kakaocdn.net/dna/qj6Ch/btsMVrRo7Bb/AAAAAAAAAAAAAAAAAAAAAMc74hwuVMIoTDIkWZD6osaPbxWkdweeZu5YJTwL9UIw/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=v0XDTL7kyhSxpjdFJ2Uv2fKeQo8%3D)

### 3. 앱 상단 Claude - 설정 클릭

![](https://blog.kakaocdn.net/dna/tWoKf/btsMXaABGQ0/AAAAAAAAAAAAAAAAAAAAABcXxHoxrscj5fbrjgJee04cNYtbAXO7jggrrTUGRaVS/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=wiyu600AxxbD2iZAlgJw2uoDCG8%3D)

### 4. 개발자 선택 후 - 설정 편집 클릭합니다.

![](https://blog.kakaocdn.net/dna/MZVA2/btsMXdKSKHk/AAAAAAAAAAAAAAAAAAAAALWzkemq2h55k2Gg6PBFq7WO73eo12hx_btd8vLWnwZa/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=RX50GJiro2Q8ZBFUbQnLt61Hooc%3D)

### 5. claude_desktop_config.json 생성 확인 후 수정

경로 : /Users/{user_name}/Library/Application Support/Claude

![](https://blog.kakaocdn.net/dna/bdXcif/btsMW0EY8qX/AAAAAAAAAAAAAAAAAAAAABCKuDFQy9HJtmxmLEeAH_3_eKPdoiWJh2CJQ_QAx7ME/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=0BJxo4VDxt86MWDikUtKT7IQXdA%3D)

### 6. 해당 파일 다음 내용 복사

```
{
"mcpServers": {
    "filesystem": {
        "command": "npx",
        "args": [
            "-y",
            "@modelcontextprotocol/server-filesystem",
            "/Users/user_name/Desktop"
            ]
        }
        }
}
```

user_name은 사용자 이름으로 변경해주세요.

### 7. 저장 후 Claude Desktop 종료 후 실행

![](https://blog.kakaocdn.net/dna/KiuBx/btsMVFV8KK2/AAAAAAAAAAAAAAAAAAAAANLi1XotS02YNGGVxIepw6GswvMXqEk3kaccA7B5PH9c/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=UUY7FpoC9vKjlcvMSwoAiw%2FuFYw%3D)

-> 정상적으로 running 확인 가능함.

![](https://blog.kakaocdn.net/dna/pnlxM/btsMWnAJOMG/AAAAAAAAAAAAAAAAAAAAAI2CN8PxYkcp8me15mD2FiDUgQKVB4k_--RylCxxSwZ0/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=iwtMevlcDM4o2kvttiFx5RpaDxQ%3D)

-> 해당 부분 생긴 걸 확인할 수 있음.

### 8. MCP filte-system Test

![](https://blog.kakaocdn.net/dna/diAQCJ/btsMXyOMsHf/AAAAAAAAAAAAAAAAAAAAAKoQ3mzAwM1t9tlm3OUQFmqaPcqhu_6tBdWxLEXxuCBF/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=GC%2FMK7nNJZtlHqIKEymEfLrcwCI%3D)

## 에러 슈팅

### 1. NPX 실행이 안될 때

-> node.js를 다운 받으셔야 합니다.

### 2. npx 명령어가 실행이 됐는데도 command not found 에러 발생 시

-> NVM로 여러 노드 버전을 관리하면 환경에선 해당 에러가 발생합니다. 노드 버전 1개만 남겨놓으면 됩니다.

(Why? I don't know, 관련 글이 하나도 없음.)
