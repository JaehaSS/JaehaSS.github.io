---
layout: post
title: "GitHub Spec Kid이란?(SDD)"
date: 2025-09-22
description: "GitHub Spec Kit의 개념과 Spec-Driven Development(SDD) 방법론, 설치 및 사용법을 정리한 글입니다."
tags: [Python]
tistory_id: 66
---

## GitHub Spec Kit 상세 분석

### Spec Kit이란 무엇인가?

GitHub Spec Kit은 CSS 프레임워크로 오해받기도 하지만, 실제로는 "Spec-Driven Development (SDD)를 위한 오픈소스 툴킷"입니다. GitHub이 2024년에 공개했으며, GitHub Copilot, Claude Code, Gemini CLI 같은 AI 코딩 에이전트와 협업할 때 체계적인 개발 프로세스를 제공합니다.

**핵심 정의:** "Vibe Coding"(비체계적인 AI 프롬프팅)을 구조화된 스펙 기반 개발로 대체하는 Python 기반 CLI 도구이자 방법론 프레임워크.

### Why - 왜 Spec Kit이 필요한가?

#### "Vibe Coding"의 문제점

GitHub은 AI 기반 개발의 근본적인 문제를 다음과 같이 설명합니다:

> "As coding agents get more powerful, a pattern emerged. You describe a goal, get a code block, and it often looks right but doesn't actually work."

**기존 방식의 한계:**

1. **일관성 없는 AI 출력** - 모호한 프롬프트는 임의적인 가정으로 이어짐
2. **누적되는 기술 부채** - 코드가 사실상 스펙이 되어 유지보수 악몽 초래
3. **누락된 컨텍스트** - 핵심 요구사항이 대화, 위키, 개발자 머릿속에만 존재
4. **아키텍처 드리프트** - 의도와 구현 간 정렬을 유지할 체계적 방법 부재

#### 패러다임 전환: Code-First에서 Specification-First로

Spec Kit은 오랜 "코드가 왕"이라는 패러다임을 뒤집습니다. 스펙은 더 이상 예비 단계가 아니라 직접 작동하는 구현을 생성하는 "실행 가능한 문서"가 됩니다.

### What - Spec Kit의 핵심 기능과 구성 요소

#### 주요 구성 요소

**1. Specify CLI**
- Python 기반 커맨드라인 도구
- 크로스 플랫폼 지원 (Linux, macOS, Windows, WSL2)
- 다중 AI 에이전트 호환

**2. 템플릿 시스템**

```
.specify/templates/
├── agent-file-template.md
├── plan-template.md
├── spec-template.md
└── tasks-template.md
```

**3. Constitution Framework**

`constitution.md` 파일이 프로젝트 원칙을 정의합니다:

- 모든 기능은 독립 라이브러리로 시작
- 엄격한 TDD 필수
- 초기 프로젝트 구조는 최대 3개 컴포넌트로 제한
- 커스텀 추상화보다 프레임워크 기능 우선

#### 슬래시 명령 시스템

AI 에이전트를 위한 사전 설정 명령:

```
/constitution  # 거버넌스 원칙 수립
/specify       # 기능 스펙 생성
/plan          # 기술 구현 계획 작성
/tasks         # 실행 가능한 단위로 분해
/implement     # 구현 실행
```

### How - 실제 사용 방법과 구현 예제

#### 설치 방법

**기본 설치 (uvx 사용):**

```bash
uvx --from git+https://github.com/github/spec-kit.git specify init <PROJECT_NAME>
```

**NPM을 통한 설치:**

```bash
npm install -g @spec-kit/cli
npx @spec-kit/cli init my-project
```

**AI 도구별 초기화:**

```bash
specify init my-project --ai claude
specify init my-project --ai copilot
specify init my-project --ai gemini
specify init my-project --ai cursor
```

#### 프로젝트 구조

```
my-project/
├── .github/
│   └── prompts/
│       ├── plan.prompt.md
│       ├── specify.prompt.md
│       └── tasks.prompt.md
├── .specify/
│   ├── memory/
│   │   └── constitution.md
│   ├── scripts/
│   └── templates/
└── specs/
    └── 001-feature-name/
        ├── spec.md
        ├── plan.md
        └── tasks.md
```

#### Constitution 예제 코드

```markdown
# 프로젝트 헌법

## 코드 품질 표준
- Strict TDD for all implementations
- All features start as independent libraries
- Maximum 3 projects for initial implementation

## 테스트 요구사항
- Unit tests written before implementation
- User validation and approval
- Integration tests for all API endpoints

## 아키텍처 원칙
- Use framework features directly
- Modular design with clear boundaries
- CLI-first approach
```

### 장점과 현재 한계점

#### 강점

- **체계적 접근** - 비체계적인 "Vibe Coding" 제거
- **다중 AI 지원** - 다양한 AI 도구와 호환
- **조직 확장성** - Constitution을 통한 팀 일관성
- **반복적 개선** - 스펙 업데이트와 재생성 용이

#### 현재 한계점

- **초기 단계** - 아직 실험적
- **설정 오버헤드** - 간단한 프로젝트에는 과도할 수 있음
- **학습 곡선** - 스펙 작성에 규율 필요
- **도구 전환 어려움** - 초기 설정 이후 AI 도구 변경 곤란

### 결론

GitHub Spec Kit은 AI 코딩 에이전트 시대에 적합한 방법론적 진화를 대표합니다. Code-First에서 Specification-First 접근으로 전환함으로써, AI를 "검색 엔진"이 아닌 "진정한 페어 프로그래머"로 변환합니다.

**이상적인 사용 사례:**

1. 명확한 아키텍처 방향이 있는 그린필드 개발
2. 복잡한 기존 코드베이스에서의 기능 개발
3. 재구축 전 레거시 시스템 현대화
