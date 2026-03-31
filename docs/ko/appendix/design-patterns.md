# Claude Code 설계 패턴 모음

> Claude Code 소스코드에서 발견된 핵심 설계 패턴을 정리한다. 각 패턴에 대해 목적, 코드 예시, 장점과 트레이드오프를 기술한다.

---

## 1. 병렬 Prefetch 패턴 (Parallel Prefetch Pattern)

### 설명

`main.tsx` 진입 직후, 독립적인 초기화 작업(MDM 읽기, 키체인 읽기, 환경 설정 적용)을 **모듈 임포트 단계에서 즉시 비동기적으로 시작**한다. 이후 실제로 해당 값이 필요한 시점까지 I/O가 백그라운드에서 완료되므로 스타트업 레이턴시가 크게 감소한다.

### 코드 예시

```typescript
// src/main.tsx - 최상단 사이드이펙트 구간

import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');   // 1. 스타트업 타임스탬프 기록

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                      // 2. MDM 서브프로세스 즉시 실행 (plutil/reg query)

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();               // 3. macOS 키체인 읽기 병렬 시작
                                       //    (OAuth + 레거시 API 키 동시 조회)

// 이후 ~135ms의 모듈 임포트가 진행되는 동안
// 위 세 작업이 백그라운드에서 완료됨
```

```typescript
// 나중에 실제로 필요할 때 완료 대기
await ensureKeychainPrefetchCompleted()
```

### 장점

- MDM 읽기와 키체인 읽기를 직렬로 수행할 경우 약 65ms의 동기 블로킹이 발생하는데, 병렬화로 이를 제거
- 스타트업 크리티컬 패스에서 I/O 대기 시간을 숨김 (latency hiding)
- 각 프리페치 모듈이 독립적이어서 실패가 격리됨

### 트레이드오프

- 사이드이펙트가 임포트 단계에서 발생하므로 `eslint-disable custom-rules/no-top-level-side-effects` 주석이 필요
- 프리페치 완료 전에 해당 값에 접근하면 블로킹이 발생하므로 호출 순서를 신중히 관리해야 함
- 테스트에서 모킹이 까다로움

---

## 2. Dead Code Elimination 패턴 (Bundle Feature Flag Pattern)

### 설명

Bun 번들러의 `feature()` 함수를 사용해 컴파일 타임에 피처 플래그 값을 평가하고, `false`인 브랜치의 코드를 번들에서 완전히 제거한다. 결과적으로 프로덕션 번들에 불필요한 코드가 포함되지 않으며, 피처 플래그별로 서로 다른 번들을 생성할 수 있다.

### 코드 예시

```typescript
// src/main.tsx

import { feature } from 'bun:bundle'   // Bun 번들러 전용 모듈

// 번들 타임에 COORDINATOR_MODE가 false이면
// 아래 require()와 모든 참조가 번들에서 제거됨
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') as typeof import('./coordinator/coordinatorMode.js')
  : null

// Kairos(어시스턴트) 모드도 동일 패턴
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') as typeof import('./assistant/index.js')
  : null
```

```typescript
// src/tools.ts - 도구 레지스트리에서도 동일 패턴
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      // ...
    ]
  : []
```

```typescript
// src/types/permissions.ts - 타입 레벨에서도 사용
export const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? (['auto'] as const) : ([] as const)),
] as const satisfies readonly PermissionMode[]
```

### 장점

- 피처별로 최소 크기의 번들 생성 가능 (번들 크기 감소)
- 사용하지 않는 기능의 코드가 런타임 메모리를 차지하지 않음
- 피처 플래그가 코드 분기가 아닌 번들 분기이므로 런타임 오버헤드 없음

### 트레이드오프

- Bun 번들러에 특화된 API이므로 다른 런타임에서는 동작하지 않음
- `require()` 방식의 동적 임포트로 인해 TypeScript 타입 추론이 복잡해짐 (`as typeof import(...)` 캐스팅 필요)
- 번들 타임 플래그와 런타임 GrowthBook 플래그를 혼동하지 않도록 주의 필요

---

## 3. 지연 로딩 패턴 (Lazy Loading for Circular Dependency Resolution)

### 설명

순환 의존성(circular dependency)이 발생하는 모듈은 `require()`를 함수로 감싸 **실제 사용 시점까지 임포트를 지연**한다. 모듈 평가 순서 문제를 회피하면서도 타입 안전성을 유지한다.

### 코드 예시

```typescript
// src/main.tsx

// teammate.ts → AppState.tsx → ... → main.tsx 순환 의존성 방지
const getTeammateUtils = () =>
  require('./utils/teammate.js') as typeof import('./utils/teammate.js')

const getTeammatePromptAddendum = () =>
  require('./utils/swarm/teammatePromptAddendum.js') as typeof import('./utils/swarm/teammatePromptAddendum.js')

const getTeammateModeSnapshot = () =>
  require('./utils/swarm/backends/teammateModeSnapshot.js') as typeof import('./utils/swarm/backends/teammateModeSnapshot.js')

// 사용 시점에 호출
const teammateUtils = getTeammateUtils()
await teammateUtils.someFunction()
```

### 장점

- 순환 의존성으로 인한 `undefined` 참조 오류를 런타임에서 방지
- `as typeof import(...)` 패턴으로 완전한 타입 추론 유지
- 해당 기능이 실제로 사용되지 않으면 모듈 로딩 비용 없음

### 트레이드오프

- 함수 호출마다 `require()` 캐시 조회가 발생 (Node.js는 캐시하므로 실제 비용은 미미)
- ESLint `no-require-imports` 규칙 비활성화 주석이 필요
- IDE의 "Go to Definition" 기능이 `typeof import()` 패턴에서 완벽히 동작하지 않을 수 있음

---

## 4. 레지스트리 패턴 (Registry Pattern)

### 설명

도구(Tool)와 커맨드(Command)를 중앙 레지스트리 함수(`getTools()`, `getCommands()`)에서 배열로 관리한다. 런타임 설정, 피처 플래그, 사용자 권한에 따라 활성화할 항목을 동적으로 결정한다.

### 코드 예시

```typescript
// src/tools.ts - 도구 레지스트리

import { AgentTool } from './tools/AgentTool/AgentTool.js'
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'
// ... 모든 도구 임포트

export function getTools(options: GetToolsOptions): Tools {
  const tools: Tool[] = [
    new BashTool(),
    new FileEditTool(),
    new FileReadTool(),
    new FileWriteTool(),
    new GlobTool(),
    new GrepTool(),
    // ...
  ]

  // 조건부 도구 추가 (피처 플래그, 사용자 타입 등)
  if (REPLTool) tools.push(new REPLTool())
  if (SleepTool) tools.push(new SleepTool())
  if (cronTools.length) tools.push(...cronTools.map(T => new T()))

  return tools
}
```

```typescript
// src/Tool.ts - 도구 인터페이스
export interface Tool {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  call(input: unknown, context: ToolUseContext): Promise<ToolResult>
}
```

### 장점

- 새 도구 추가 시 레지스트리에 한 줄만 추가하면 됨 (확장 용이)
- 도구 목록을 런타임에 동적으로 필터링 가능
- LLM에 전달되는 도구 스키마와 실제 구현이 동일 객체에서 관리됨

### 트레이드오프

- 모든 도구 클래스가 `tools.ts`에 임포트되어 번들 크기가 증가 (Dead Code Elimination으로 완화)
- 도구 이름 충돌 시 런타임에서야 감지됨 (정적 분석 불가)

---

## 5. 에이전트 루프 패턴 (Agent Loop Pattern)

### 설명

`QueryEngine.ts`가 구현하는 핵심 제어 흐름으로, LLM 응답을 받아 도구 호출 요청을 추출하고, 도구를 실행하고, 결과를 다시 LLM에 피드백하는 루프를 반복한다. 루프는 도구 호출이 없는 최종 응답이 오거나 중단 신호를 받을 때 종료된다.

### 코드 예시

```
QueryEngine 실행 흐름:

1. 사용자 입력 수신
   │
   ▼
2. 시스템 프롬프트 + 대화 히스토리 + 사용 가능 도구 목록 구성
   │
   ▼
3. LLM API 호출 (스트리밍)
   │
   ├─ 텍스트 응답 → UI에 스트리밍 출력
   │
   └─ 도구 호출 요청 (ToolUseBlock)
       │
       ▼
4. CanUseTool 권한 검사
   ├─ deny → 거부 메시지 생성
   ├─ ask  → 사용자 승인 UI 표시
   └─ allow → 도구 실행
       │
       ▼
5. 도구 결과를 ToolResultBlock으로 변환
   │
   ▼
6. 대화 히스토리에 추가 → 3번으로 돌아감
   │
   └─ 도구 호출 없음 → 루프 종료
```

```typescript
// src/QueryEngine.ts - 루프 핵심 구조 (개념적 표현)
while (true) {
  const response = await query(messages, tools, options)

  if (response.stop_reason === 'end_turn') break
  if (response.stop_reason !== 'tool_use') break

  const toolResults = await Promise.all(
    response.content
      .filter(block => block.type === 'tool_use')
      .map(block => executeTool(block, canUseTool))
  )

  messages.push(
    { role: 'assistant', content: response.content },
    { role: 'user', content: toolResults },
  )
}
```

### 장점

- 임의 깊이의 도구 체인 실행 가능 (도구가 다른 도구를 호출하는 AgentTool 포함)
- 루프 각 단계가 독립적으로 취소 가능 (AbortController)
- 스트리밍과 루프 제어 로직이 분리되어 있어 테스트 용이

### 트레이드오프

- 루프 깊이에 따라 컨텍스트 윈도우가 소모됨 (컴팩트 메커니즘으로 완화)
- 무한 루프 방지를 위한 최대 반복 횟수 제한이 필요
- 병렬 도구 실행 시 결과 순서 보장 로직이 복잡해짐

---

## 6. 스트리밍 파이프라인 패턴 (Streaming Pipeline Pattern)

### 설명

Claude API의 SSE(Server-Sent Events) 스트림을 구독하여 토큰 단위로 응답을 수신하고, 즉시 Ink 컴포넌트를 통해 터미널에 렌더링한다. 파싱, 축적, 렌더링이 파이프라인으로 연결되어 첫 토큰부터 사용자에게 표시된다.

### 코드 예시

```
SSE 스트림
    │
    ▼
event: message_delta          ← HTTP 청크 수신
    │
    ▼
파싱: content_block_delta     ← text / tool_use 블록 분류
    │
    ├─ text_delta
    │       │
    │       ▼
    │   React state 업데이트
    │       │
    │       ▼
    │   Ink Reconciler
    │       │
    │       ▼
    │   터미널 출력 (log-update 방식)
    │
    └─ tool_use_delta
            │
            ▼
        JSON 입력 축적
            │
            ▼
        도구 실행 단계로 전달
```

```typescript
// src/services/api/claude.ts (개념적 표현)
for await (const event of stream) {
  if (event.type === 'content_block_delta') {
    if (event.delta.type === 'text_delta') {
      onText(event.delta.text)           // UI 즉시 업데이트
    } else if (event.delta.type === 'input_json_delta') {
      accumulateToolInput(event.delta.partial_json)
    }
  }
  if (event.type === 'message_stop') {
    break
  }
}
```

### 장점

- 첫 번째 토큰 표시 시간(TTFT, Time to First Token)이 최소화됨
- 긴 응답에서도 사용자가 즉각적인 피드백을 받음
- 스트림 중단(Ctrl+C) 시 즉시 취소 가능

### 트레이드오프

- 스트리밍 도중 도구 입력 JSON이 불완전한 상태로 축적되므로 파싱 전략이 필요
- 렌더링 빈도를 조절하지 않으면 터미널 깜박임(flicker) 발생 (`tengu_flicker` 이벤트로 추적)
- 네트워크 중단 시 부분 응답 처리 로직이 필요

---

## 7. 권한 인터셉터 패턴 (Permission Interceptor Pattern)

### 설명

모든 도구 실행 전에 `CanUseTool` 함수가 권한을 검사하는 게이트 역할을 한다. 권한 모드(Permission Mode), 사용자 설정, 정책 제한을 조합하여 `allow`, `deny`, `ask` 중 하나를 반환한다. `ask`의 경우 UI를 통해 사용자에게 실시간으로 승인을 요청한다.

### 코드 예시

```typescript
// src/hooks/useCanUseTool.ts (개념적 표현)
export type CanUseToolFn = (
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
) => Promise<PermissionResult>

// 권한 결정 흐름
async function canUseTool(tool, input, context): Promise<PermissionResult> {
  // 1. 정책 제한 확인 (관리자 강제 규칙)
  if (isPolicyDenied(tool.name)) return { behavior: 'deny', reason: 'policy' }

  // 2. bypassPermissions 모드: 모두 허용
  if (permissionMode === 'bypassPermissions') return { behavior: 'allow' }

  // 3. 사용자 설정에서 규칙 조회
  const rule = findMatchingRule(tool.name, input, userSettings)
  if (rule) return { behavior: rule.behavior }

  // 4. 기본값: 사용자에게 확인 요청
  return { behavior: 'ask' }
}
```

```typescript
// 에이전트 루프에서의 사용
const permission = await canUseTool(tool, input, context)

switch (permission.behavior) {
  case 'allow':
    return await tool.call(input, context)
  case 'deny':
    return createDenialResult(permission.reason)
  case 'ask':
    const userDecision = await showPermissionPrompt(tool, input)
    if (userDecision === 'allow') return await tool.call(input, context)
    return createDenialResult('user_rejected')
}
```

### 장점

- 권한 로직이 도구 구현과 완전히 분리됨 (SRP)
- 새로운 권한 모드 추가 시 도구 코드를 수정할 필요 없음
- 영구/임시 승인 선택을 통해 사용자 설정에 자동 저장 가능

### 트레이드오프

- `ask` 모드에서 사용자 응답 대기로 인해 에이전트 루프가 블로킹됨
- 권한 결정이 비동기이므로 병렬 도구 실행 시 여러 프롬프트가 동시에 나타날 수 있음
- 권한 규칙 매칭 로직이 복잡해질수록 성능 영향 가능성

---

## 8. DeepImmutable 패턴 (Deep Immutable Type Pattern)

### 설명

타입 레벨에서 중첩 객체까지 완전한 불변성을 강제하는 유틸리티 타입 패턴이다. `Readonly<T>`는 최상위 속성만 보호하지만, `DeepImmutable<T>`는 모든 중첩 객체와 배열까지 `readonly`로 만들어 의도치 않은 변경을 컴파일 타임에 차단한다.

### 코드 예시

```typescript
// src/utils/ (개념적 구현)
type DeepImmutable<T> =
  T extends (infer U)[]
    ? ReadonlyArray<DeepImmutable<U>>
    : T extends object
      ? { readonly [K in keyof T]: DeepImmutable<T[K]> }
      : T

// 사용 예시: AppState를 불변으로 전달
function renderUI(state: DeepImmutable<AppState>): JSX.Element {
  // state.messages.push(...)  → 컴파일 에러 (readonly array)
  // state.config.model = '...' → 컴파일 에러 (readonly property)
  return <View messages={state.messages} />
}
```

```typescript
// src/services/analytics/index.ts 의 마커 타입도 유사 원리
// never 타입을 사용해 "이 타입은 직접 값을 가질 수 없다"는 불변 제약 표현
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```

### 장점

- 전역 상태가 컴포넌트에서 변경되는 것을 컴파일 타임에 방지
- `Object.freeze()`와 달리 런타임 오버헤드 없음 (타입 전용)
- 함수형 프로그래밍 스타일(불변 업데이트)을 타입 시스템이 강제

### 트레이드오프

- 복잡한 재귀 타입으로 인해 TypeScript 컴파일 시간이 증가할 수 있음
- 의도적으로 변경이 필요한 경우 `as Mutable<T>` 캐스팅이 필요하여 가독성이 저하됨
- 깊은 중첩 타입에서 타입 에러 메시지가 읽기 어려워짐

---

## 패턴 간 상호작용

```
병렬 Prefetch (패턴 1)
    └─ 스타트업 완료
           │
           ▼
Dead Code Elimination (패턴 2)
    └─ 피처별 도구 포함/제외
           │
           ▼
레지스트리 패턴 (패턴 4)
    └─ 활성 도구 목록 구성
           │
           ▼
에이전트 루프 (패턴 5)
    └─ LLM ↔ 도구 반복
           │
    ┌──────┴──────┐
    │             │
스트리밍 파이프라인 (패턴 6)  권한 인터셉터 (패턴 7)
    └─ UI 실시간 업데이트      └─ 도구 실행 전 승인 게이트
```

각 패턴은 독립적으로 동작하지만, Claude Code의 전체 실행 흐름에서 위와 같이 순차적으로 조합된다. 지연 로딩(패턴 3)과 DeepImmutable(패턴 8)은 횡단 관심사(cross-cutting concern)로서 여러 패턴에 걸쳐 적용된다.
