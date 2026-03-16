---
layout: post
title: "간단히 적어보는 claude code 1.0 -> 2.0 적용 방법"
date: 2025-10-02
description: "Claude Code 2.0 업그레이드 시 발생하는 설치 경로 충돌 문제와 해결 방법을 간단히 정리한 글입니다."
tags: [Python]
tistory_id: 67
---

## 개요

Anthropic의 새로운 Claude Sonnet 4.5 모델과 함께 출시된 Claude Code 2.0으로의 업그레이드 방법을 정리합니다. 업데이트를 시도하면서 설치 관련 이슈를 겪었습니다.

## 환경

MacBook M3에서 진행했습니다.

## 해결 방법

다음 설치 명령을 실행합니다:

```bash
rm -rf ${where claude}  # 기존 Claude 설치 삭제

npm install -g @anthropic-ai/claude-code  # Claude Code 설치
```

## 핵심 이슈

기존 설치 경로와 새로운 경로가 달라서 시스템이 업데이트된 버전을 인식하지 못하는 문제가 발생합니다.

기존 설치를 먼저 제거한 후 새로 설치하면 정상적으로 2.0 버전이 적용됩니다.
