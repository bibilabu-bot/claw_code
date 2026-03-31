# 한영 용어 대조표

> Claude Code 소스코드에서 사용되는 주요 기술 용어의 한국어 번역, 영문 원어, 정의, 소스코드 위치를 정리한 참조 테이블이다.

---

## 핵심 아키텍처

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 도구 | Tool | LLM이 호출할 수 있는 기능 단위. `name`, `description`, `inputSchema`, `call()` 메서드를 가진다 | `src/Tool.ts` |
| 도구 등록소 | Tool Registry | 활성화된 도구 집합을 중앙에서 관리하는 패턴. `getTools()`가 도구 배열을 반환 | `src/tools.ts` |
| 에이전트 | Agent | 독립적인 LLM 루프를 실행하는 서브프로세스. AgentTool을 통해 생성 | `src/tools/AgentTool/` |
| 서브에이전트 | Sub-agent | 코디네이터 또는 부모 에이전트로부터 위임받은 작업을 독립 실행하는 에이전트 | `src/tools/AgentTool/AgentTool.ts` |
| 질의 엔진 | QueryEngine | LLM API 호출, 도구 실행, 대화 루프를 조율하는 중앙 엔진 | `src/QueryEngine.ts` |
| 쿼리 | Query | 단일 LLM API 호출 + 응답 처리 사이클 | `src/query.ts` |
| 에이전트 루프 | Agent Loop | LLM 응답 → 도구 실행 → 결과 피드백 → LLM 응답을 반복하는 제어 흐름 | `src/QueryEngine.ts` |
| 코디네이터 모드 | Coordinator Mode | 여러 서브에이전트를 조율하는 상위 에이전트 실행 모드 | `src/coordinator/coordinatorMode.ts` |
| 팀메이트 | Teammate | 스웜(swarm) 아키텍처에서 협력하는 에이전트 인스턴스 | `src/utils/teammate.ts` |

---

## 권한 시스템

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 권한 모드 | Permission Mode | 도구 실행 시 사용자 승인 요구 수준. `default`, `acceptEdits`, `bypassPermissions`, `plan`, `dontAsk`, `auto` | `src/types/permissions.ts` |
| 권한 결과 | PermissionResult | 도구 실행 전 권한 검사 결과. `allow`, `deny`, `ask` | `src/types/permissions.ts` |
| 권한 인터셉터 | Permission Interceptor | 도구 실행 전 승인 여부를 결정하는 게이트 로직 | `src/hooks/useCanUseTool.ts` |
| 바이패스 권한 | Bypass Permissions | 모든 도구 실행을 자동 승인하는 최고 신뢰 모드 | `src/types/permissions.ts` |
| 계획 모드 | Plan Mode | 파일 수정 없이 계획만 수립하는 읽기 전용 권한 모드 | `src/tools/EnterPlanModeTool/` |
| 정책 제한 | Policy Limits | 관리자 또는 MDM을 통해 부과되는 외부 제약 | `src/services/policyLimits/` |

---

## UI/렌더링

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 인크 컴포넌트 | Ink Component | React 기반 터미널 UI 컴포넌트. Ink 라이브러리의 커스텀 구현 | `src/ink/` |
| 스트리밍 | Streaming | LLM 응답을 토큰 단위로 실시간 수신·렌더링하는 기법 (SSE 기반) | `src/services/api/claude.ts` |
| SSE | Server-Sent Events | LLM API가 스트리밍 응답을 전송하는 HTTP 프로토콜 | `src/services/api/` |
| REPL | Read-Eval-Print Loop | 대화형 입력-처리-출력 반복 루프. 인터랙티브 세션 진입점 | `src/replLauncher.tsx` |
| 오버레이 | Overlay | 메인 UI 위에 표시되는 모달 또는 프롬프트 레이어 | `src/context/overlayContext.tsx` |

---

## 명령어 시스템

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 슬래시 커맨드 | Slash Command | `/` 접두사로 시작하는 사용자 입력 명령어 | `src/commands.ts` |
| 스킬 | Skill | 슬래시 커맨드 형태로 등록되는 재사용 가능한 작업 템플릿 | `src/skills/` |
| 스킬 도구 | Skill Tool | 스킬을 LLM 도구로 노출하는 래퍼 | `src/tools/SkillTool/SkillTool.ts` |
| 빌트인 플러그인 | Bundled Plugin | 기본 내장된 플러그인. 동적 로딩 없이 번들에 포함 | `src/plugins/bundled/` |
| 플러그인 | Plugin | 기능을 확장하는 모듈 단위. 도구, 커맨드, 스킬을 제공 | `src/types/plugin.ts` |

---

## 컨텍스트 & 메모리

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 컨텍스트 윈도우 | Context Window | LLM이 한 번에 처리할 수 있는 최대 토큰 수 | `src/utils/context.ts` |
| 토큰 예산 | Token Budget | 단일 쿼리 또는 세션에서 사용 가능한 최대 토큰 할당량 | `src/utils/thinking.ts` |
| 컴팩트 | Compact | 긴 대화 히스토리를 요약하여 컨텍스트 윈도우를 확보하는 작업 | `src/QueryEngine.ts` |
| 메모리 프롬프트 | Memory Prompt | `CLAUDE.md` 등 파일에서 로드되는 지속적 지시사항 | `src/memdir/memdir.ts` |
| 세션 | Session | UUID로 식별되는 단일 대화 세션. 트랜스크립트 저장 단위 | `src/bootstrap/state.ts` |
| 트랜스크립트 | Transcript | 세션 내 모든 메시지와 도구 결과의 기록 | `src/utils/sessionStorage.ts` |

---

## 로딩 & 최적화

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 지연 로딩 | Lazy Loading | 모듈을 즉시 임포트하지 않고 필요 시점에 `require()`로 동적 로드하는 패턴 | `src/main.tsx:69-82` |
| 병렬 프리페치 | Parallel Prefetch | 독립적인 초기화 작업을 동시에 실행하는 스타트업 최적화 기법 | `src/main.tsx:16-20` |
| 데드 코드 제거 | Dead Code Elimination | `feature()` 플래그 기반으로 번들 타임에 미사용 코드를 제거하는 최적화 | `src/tools.ts:14-52` |
| 메모이제이션 | Memoization | 동일 인자에 대한 함수 결과를 캐시하여 재계산을 방지하는 기법 | `src/services/analytics/datadog.ts` |

---

## 외부 서비스 & 프로토콜

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| MCP | Model Context Protocol | 외부 서버가 도구와 리소스를 Claude에 제공하는 표준 프로토콜 | `src/services/mcp/` |
| LSP | Language Server Protocol | 코드 지능(정의 이동, 참조 검색 등)을 제공하는 표준 프로토콜 | `src/tools/LSPTool/` |
| OAuth | OAuth 2.0 | Claude.ai 계정 인증에 사용하는 표준 인증 위임 프로토콜 | `src/services/oauth/` |
| JWT | JSON Web Token | OAuth 인증 흐름에서 사용하는 토큰 포맷 | `src/utils/auth.ts` |
| gRPC | gRPC | 1P 이벤트 로깅 익스포터가 사용하는 원격 프로시저 호출 프로토콜 (HTTP/2 기반) | `src/services/analytics/firstPartyEventLoggingExporter.ts` |
| OpenTelemetry | OpenTelemetry | 1P 이벤트 로깅에 사용하는 벤더 중립적 관측성 프레임워크 | `src/services/analytics/firstPartyEventLogger.ts` |

---

## 보안 & 인증

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 키체인 | Keychain | macOS 보안 자격증명 저장소. OAuth 토큰과 API 키를 보관 | `src/utils/secureStorage/` |
| 키체인 프리페치 | Keychain Prefetch | 스타트업 레이턴시 감소를 위해 키체인 읽기를 병렬로 미리 실행 | `src/utils/secureStorage/keychainPrefetch.ts` |
| MDM | Mobile Device Management | 기업 환경에서 Claude Code 설정을 중앙 관리하는 시스템 | `src/utils/settings/mdm/` |
| 신뢰 다이얼로그 | Trust Dialog | 새 프로젝트 디렉토리에 대한 사용자 신뢰 확인 UI | `src/bootstrap/state.ts` |

---

## 피처 플래그 & A/B 테스트

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| GrowthBook | GrowthBook | 피처 플래그와 A/B 테스트를 관리하는 외부 서비스 | `src/services/analytics/growthbook.ts` |
| 피처 플래그 | Feature Flag | 런타임에서 기능을 켜고 끄는 제어 메커니즘 | `src/services/analytics/growthbook.ts` |
| 피처 게이트 | Feature Gate | GrowthBook의 불리언 피처 플래그. `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | `src/services/analytics/growthbook.ts` |
| 동적 설정 | Dynamic Config | GrowthBook에서 런타임에 제공되는 구조화된 설정값 | `src/services/analytics/growthbook.ts` |
| 번들 피처 | Bundle Feature | 번들 타임에 평가되는 `feature()` 플래그 (Bun 번들러 전용) | `src/types/permissions.ts` |

---

## 런타임 & 빌드

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| Bun | Bun | Claude Code의 JavaScript 런타임 및 번들러 | `bun.lock`, `package.json` |
| `bun:bundle` | bun:bundle | Bun 번들러가 제공하는 특수 모듈. `feature()` 함수로 번들 타임 조건 분기 | `src/main.tsx:21` |
| Commander.js | Commander.js | CLI 인자 파싱에 사용하는 라이브러리 (`@commander-js/extra-typings`) | `src/main.tsx:22` |
| Zod | Zod | 스키마 검증 라이브러리. 도구 입력 스키마와 설정 파일 검증에 사용 | `src/Tool.ts` |

---

## 데이터 & 타입

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| DeepImmutable | DeepImmutable | 타입 레벨에서 중첩 객체까지 불변성을 강제하는 유틸리티 타입 | `src/utils/` |
| 마커 타입 | Marker Type | 실제 값을 담지 않고 개발자 의도를 타입 시스템에 문서화하는 `never` 기반 타입 | `src/services/analytics/index.ts:19` |
| PII 태깅 | PII Tagging | 개인 식별 정보를 `_PROTO_*` 키로 표시하여 특권 스토리지로만 라우팅하는 패턴 | `src/services/analytics/index.ts:27` |

---

## 브릿지 & 통합

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 브릿지 | Bridge | Claude Code와 외부 환경(Chrome, 원격 세션)을 연결하는 통신 레이어 | `src/bridge/` |
| 훅 | Hook | 특정 이벤트 시점에 실행되는 사용자 정의 스크립트 (Pre/PostToolUse 등) | `src/types/hooks.ts` |
| 워크트리 | Worktree | Git worktree를 활용한 병렬 개발 환경 관리 기능 | `src/tools/EnterWorktreeTool/` |
| 업스트림 프록시 | Upstream Proxy | 기업 네트워크에서 Claude API 요청을 중계하는 프록시 설정 | `src/upstreamproxy/` |

---

## 분석 & 모니터링

| 한국어 | English | 정의 | 소스코드 위치 |
|--------|---------|------|--------------|
| 분석 싱크 | Analytics Sink | 이벤트를 실제 백엔드로 전달하는 라우팅 레이어 | `src/services/analytics/sink.ts` |
| 이벤트 샘플링 | Event Sampling | 고빈도 이벤트의 일부만 기록하여 스토리지 비용을 절감하는 기법 | `src/services/analytics/firstPartyEventLogger.ts` |
| 사용자 버킷 | User Bucket | 사용자 ID 해시 기반 0~29 정수. 고유 사용자 수를 PII 없이 근사하는 데 사용 | `src/services/analytics/datadog.ts` |
| 킬스위치 | Killswitch | 런타임에서 특정 분석 싱크를 즉시 비활성화하는 메커니즘 | `src/services/analytics/sinkKillswitch.ts` |
| 텔레메트리 | Telemetry | 앱 동작에 관한 사용 데이터를 자동 수집·전송하는 시스템 전체 | `src/services/analytics/` |
