# Claude Code 소스코드 분석 문서

## 프로젝트 소개

이 프로젝트는 Anthropic의 **Claude Code CLI** 소스코드를 체계적으로 분석한 한국어 기술 문서입니다. Claude Code는 1,900개 이상의 파일과 512,000줄 이상의 TypeScript 코드로 구성된 대규모 개발 도구입니다. npm 소스맵을 통해 공개된 원본 코드를 바탕으로, 각 시스템의 설계 원리, 구현 방식, 데이터 흐름을 상세히 기술합니다. 이 문서는 Claude Code의 내부 동작 방식을 이해하고자 하는 개발자, 아키텍처 설계자, 그리고 고급 사용자를 위해 작성되었습니다.

---

## 기술 스택 한눈에 보기

Claude Code는 다음과 같은 핵심 기술 스택으로 구성되어 있습니다:

| 계층 | 기술 | 용도 |
|------|------|------|
| **런타임** | Bun | JavaScript/TypeScript 실행 엔진 |
| **언어** | TypeScript | 타입 안전 및 대규모 코드베이스 관리 |
| **CLI 프레임워크** | Commander.js | 커맨드 라인 인터페이스 및 인자 파싱 |
| **UI 라이브러리** | React/Ink | 터미널 기반 사용자 인터페이스 |
| **검증** | Zod v4 | 런타임 데이터 스키마 검증 |
| **프로토콜** | MCP SDK | Machine Context Protocol 구현 |
| **통신** | LSP (Language Server Protocol) | IDE 브릿지 및 언어 서버 지원 |
| **모니터링** | OpenTelemetry | 분산 추적 및 성능 관찰 |
| **번들링** | esbuild | TypeScript 컴파일 및 번들 최적화 |

---

## 읽기 가이드

이 문서는 세 가지 학습 경로를 제공합니다. 목표와 시간 여건에 따라 선택하세요.

### 경로 1: 빠른 개요 (30분)
Claude Code의 전체 개념을 빠르게 파악하고 싶다면 이 경로를 추천합니다.
1. [architecture.md](./level-1-overview/architecture.md) - 전체 아키텍처 구조
2. [request-lifecycle.md](./level-1-overview/request-lifecycle.md) - 사용자 요청이 처리되는 전 과정

### 경로 2: 특정 시스템 이해 (1-2시간)
특정 기능이나 시스템에 깊게 들어가고 싶다면 이 경로를 추천합니다.
1. [key-concepts.md](./level-1-overview/key-concepts.md) - 핵심 개념 정리
2. Level 2 문서에서 관심 있는 시스템 선택

### 경로 3: 전체 정독 (4-8시간)
Claude Code 전체를 마스터하고 싶은 개발자를 위한 완전한 학습 경로입니다.
- Level 1 → Level 2 → Level 3 순서로 진행
- 각 문서는 이전 수준의 개념을 바탕으로 구성

---

## 문서 목차

### Level 1: 입문 (기초 개념)

개발자가 Claude Code를 이해하기 위해 필수적인 개념과 전체 구조를 다룹니다.

| 문서 | 설명 | 읽기 시간 |
|------|------|---------|
| [architecture.md](./level-1-overview/architecture.md) | Claude Code의 전체 아키텍처 조감도, 주요 컴포넌트 관계도, 계층 구조 | 15분 |
| [request-lifecycle.md](./level-1-overview/request-lifecycle.md) | 사용자 입력부터 응답까지의 전체 데이터 흐름, 각 단계별 처리 프로세스 | 20분 |
| [key-concepts.md](./level-1-overview/key-concepts.md) | QueryEngine, Tool, Command, Permission, Agent 등 핵심 개념 정리 | 15분 |

### Level 2: 서브시스템 분석 (시스템 이해)

각 주요 기능과 시스템의 설계 원리, 구현 방식, 상호 작용을 상세히 분석합니다.

| 문서 | 설명 | 읽기 시간 |
|------|------|---------|
| [query-engine.md](./level-2-systems/query-engine.md) | QueryEngine의 구조, 쿼리 파싱, 실행 엔진, 최적화 기법 | 25분 |
| [tool-system.md](./level-2-systems/tool-system.md) | Tool 레지스트리, Tool 로딩 메커니즘, 권한 검증, 실행 아키텍처 | 25분 |
| [command-system.md](./level-2-systems/command-system.md) | Commander.js 통합, 커맨드 파싱, 서브커맨드 체계, 헬프 생성 | 20분 |
| [permission-system.md](./level-2-systems/permission-system.md) | 권한 모델, 정책 평가, 감사 로깅, 사용자 제약 관리 | 25분 |
| [agent-coordinator.md](./level-2-systems/agent-coordinator.md) | 에이전트 오케스트레이션, 상태 관리, 태스크 스케줄링, 병렬 실행 | 25분 |
| [ui-ink-components.md](./level-2-systems/ui-ink-components.md) | React/Ink 통합, 컴포넌트 계층, 상태 렌더링, 터미널 UI 아키텍처 | 20분 |

### Level 3: 심화 분석 (고급 구현)

프로토콜 통합, 보안, 성능 최적화 등 고급 주제를 다룹니다.

| 문서 | 설명 | 읽기 시간 |
|------|------|---------|
| [mcp-lsp-integration.md](./level-3-internals/mcp-lsp-integration.md) | MCP 프로토콜 구현, LSP 서버 통합, 프로토콜 메시징, 핸들러 등록 | 30분 |
| [bridge-ide.md](./level-3-internals/bridge-ide.md) | IDE 통합 메커니즘, 언어 서버 브릿지, 에디터 협력 프로토콜 | 25분 |
| [plugin-skill-system.md](./level-3-internals/plugin-skill-system.md) | 플러그인 아키텍처, Skill 시스템, 동적 로딩, 버전 관리 | 30분 |
| [context-compression.md](./level-3-internals/context-compression.md) | 컨텍스트 크기 관리, 토큰 최적화, 압축 알고리즘, 메모리 효율화 | 25분 |
| [oauth-auth.md](./level-3-internals/oauth-auth.md) | OAuth 2.0 플로우, 토큰 관리, 갱신 메커니즘, 보안 모범 사례 | 30분 |
| [telemetry.md](./level-3-internals/telemetry.md) | OpenTelemetry 구현, 추적 설정, 메트릭 수집, 성능 모니터링 | 20분 |

### Appendix: 참고 자료

기술 용어, 파일 구조, 설계 패턴 참고 자료입니다.

| 문서 | 설명 |
|------|------|
| [glossary.md](./appendix/glossary.md) | 한영 기술 용어 대조표, 개념 정의, 약자 해설 |
| [file-map.md](./appendix/file-map.md) | 주요 파일 경로 인덱스, 모듈 위치, 디렉토리 구조 |
| [design-patterns.md](./appendix/design-patterns.md) | Claude Code에서 사용된 설계 패턴, 구현 사례, 적용 방법 |

---

## 기여 방법

이 분석 문서는 교육 및 연구 목적으로 작성된 것으로, 다음과 같은 방식으로 개선할 수 있습니다:

1. **오류 신고**: 잘못된 정보나 부정확한 설명을 발견하면 이슈로 등록해주세요.
2. **내용 추가**: 누락된 시스템이나 더 깊이 있는 분석이 필요한 부분을 제시해주세요.
3. **번역 검수**: 기술 용어의 적절한 표현이나 더 나은 번역 제안을 환영합니다.
4. **예제 개선**: 이해하기 쉬운 코드 예제나 다이어그램을 제시해주세요.

---

## 라이선스 및 면책

### 저작권 및 라이선스

이 문서는 **교육 목적의 기술 분석**으로 작성되었습니다. 원본 Claude Code 소스코드의 저작권은 **Anthropic Inc.**에 있습니다. 이 분석 문서는 공개된 npm 소스맵 정보를 바탕으로 한국 개발자 커뮤니티의 학습을 목적으로 작성되었습니다.

### 면책 사항

- **완전성 보장 안 함**: 이 문서는 Claude Code의 일부 측면을 분석한 것으로, 전체 기능을 포괄하지 않을 수 있습니다.
- **버전 의존성**: Claude Code는 지속적으로 업데이트되므로, 이 문서의 내용은 특정 버전을 기준으로 작성되었습니다.
- **상업적 용도 제한**: 이 문서는 개인 학습, 학술 연구, 비영리 교육 목적으로만 사용하세요.
- **보증 부재**: 이 문서의 정보 사용으로 인한 어떤 손해나 문제도 저자는 책임지지 않습니다.

### 원저작권자

Claude Code의 원본 소스코드는 [Anthropic Inc.](https://anthropic.com)의 저작물입니다.

---

## 시작하기

### 추천 학습 순서

1. 이 README를 완독하여 문서 구조를 파악합니다.
2. 본인의 목표에 맞춰 위의 "읽기 가이드"에서 경로를 선택합니다.
3. 선택한 경로의 문서를 순서대로 읽습니다.
4. 특정 시스템이 궁금하면 해당 Level 2 문서로 이동합니다.
5. 구현 세부사항이 필요하면 Level 3 문서를 참고합니다.
6. 용어나 개념이 불명확하면 Appendix의 용어사전과 설계 패턴을 확인합니다.

### 선수 지식

이 문서를 효과적으로 이해하기 위해서는 다음의 선수 지식이 도움됩니다:

- **TypeScript**: 기본 문법 및 타입 시스템 이해
- **Node.js/Bun**: JavaScript 런타임 개념
- **CLI 애플리케이션**: 커맨드 라인 도구의 기본 구조
- **React 기초**: 컴포넌트 기반 아키텍처 개념
- **네트워크 프로토콜**: HTTP, 소켓 통신의 기본 이해

선수 지식이 부족하더라도 Level 1부터 시작하면 필요한 개념을 학습하며 진행할 수 있습니다.

---

## 문서 유지 관리

### 최종 수정일

이 README는 **2026년 3월 31일** 기준으로 작성되었습니다. Claude Code의 버전 변화에 따라 내용이 변경될 수 있습니다.

### 피드백 및 연락

분석 내용에 대한 피드백, 오류 신고, 개선 제안은 프로젝트 이슈 트래커를 통해 제출해주세요.

---

**이 문서로 Claude Code의 내부 구조를 깊이 있게 이해하고, 더 나은 개발자가 되기를 바랍니다.**
