---
layout: post
title: "Claude Code를 활용한 Github 작업"
date: 2025-06-16
description: "Claude Code를 활용하여 GitHub 브랜치 생성, 커밋, PR 작성 등 반복적인 Git 작업을 자동화하는 방법을 정리한 글입니다."
tags: [ai]
tistory_id: 63
---

## 개요

간단한 작업을 하든, 복잡한 작업을 하든 협업을 위해 Gitflow 전략에 따라 Branch를 나누고 합치고 있습니다.

시간이 많이 소요되는 부분:
1. 브랜치 이름 생성
2. PR(Pull Request) 작성 (제목과 설명)

이 글에서는 Claude Code를 활용하여 이러한 프로세스를 간소화하는 방법을 다룹니다.

## Getting Started

### GitHub CLI Setup

- Homebrew로 설치: `brew install gh`
- 인증: `gh auth login`

### Claude Code 설치

Claude Code Pro plan은 일정 한도까지 무료로 사용 가능합니다.

설치 명령어:
```
npm install -g @anthropic-ai/claude-code
```

요구사항: Node v18 이상

### IntelliJ IDE 연동

1. Claude 아이콘 클릭 (좌측 상단)
2. 터미널에서 `/memory` 입력
3. "User Memory" 선택
4. CLAUDE.md에 커스텀 지시사항 편집 및 저장

## User Memory 설정

Git 전문가 요구사항을 시스템 프롬프트에 지정:

**브랜치 네이밍**: `bugfix/JH2-` 또는 `feature/JH2-` 접두사 사용 (기본값: `feature/JH2-`)

**커밋**: "변경 이력을 기반으로 메시지를 작성"

**Pull Requests**:
- 제목 형식: `feat: [15자 요약]`
- Base: dev 브랜치
- Compare: 현재 작업 브랜치
- 설명: 수정된 파일별로 변경 사항 정리

## GitHub 테스트

![](https://blog.kakaocdn.net/dna/bMs24e/btsODHdjMUE/AAAAAAAAAAAAAAAAAAAAAJbi3KUpO-ds-MnmVv4WXXmGbu7mQAlCSKibWsPkOFja/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1750085999&allow_ip=&allow_referer=&signature=7gllhLA1OsY8GoYUMxI%2FTGWYsrw%3D)

![](https://blog.kakaocdn.net/dna/Uu0C6/btsOD2OPJMx/AAAAAAAAAAAAAAAAAAAAAK-BCig2DMhdxQ-X8SoyRQSW9iWEtS51Wt0-WxL-qHtZ/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1750085999&allow_ip=&allow_referer=&signature=WlFtljGA2cM41kSwY5LkneVRlFQ%3D)

![](https://blog.kakaocdn.net/dna/PXkS3/btsOEhLHKjv/AAAAAAAAAAAAAAAAAAAAAHvXC_j-AzclsyIyUpcOdzc8IqgqzqpXJlOfxDxcKqbA/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1750085999&allow_ip=&allow_referer=&signature=aotIKUdlkfnIBwLDN0LnKMe9AYc%3D)

![](https://blog.kakaocdn.net/dna/t0xPY/btsODmN3W7i/AAAAAAAAAAAAAAAAAAAAAMm4dBofxZjoyHfiHD8zF_h8F6vN76NpcCOdclDaswg0/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1750085999&allow_ip=&allow_referer=&signature=Eo4W8IRYa3VbY86T3dZGnCzsnfg%3D)

![](https://blog.kakaocdn.net/dna/dNGrsn/btsOEhrm0UE/AAAAAAAAAAAAAAAAAAAAAF_xPLNtTNxfGbgv8f9trMZIgGRRa-4Ee9T42obPNYAW/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1750085999&allow_ip=&allow_referer=&signature=6EhgpXwTCCg6VYetUH8rk4Bbtmg%3D)

![](https://blog.kakaocdn.net/dna/di6Zrw/btsOEd3zSxg/AAAAAAAAAAAAAAAAAAAAAGlDNjl2yFuAqEHa6y1uSG4uJfpp6xy3RQ-uWFJEA2ec/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1750085999&allow_ip=&allow_referer=&signature=Vw1lIBsE5fKQ4lamcREu6FrSqYY%3D)

API 생성부터 Claude Code 실행, GitHub PR 검증까지의 워크플로우를 스크린샷으로 확인할 수 있습니다.
