# 텔레메트리 시스템

> **대상 독자**: Claude Code 내부 구현을 심층적으로 이해하려는 개발자
> **소스 경로**: `src/services/analytics/`

---

## 1. 아키텍처 개요

Claude Code의 텔레메트리 시스템은 세 개의 독립적인 백엔드로 구성된다.

```
이벤트 발생
    │
    ▼
logEvent() / logEventAsync()          ← src/services/analytics/index.ts
    │
    ├─ 싱크 미부착 시 → 이벤트 큐 (메모리)
    │
    ▼
AnalyticsSink (attachAnalyticsSink 이후)
    │
    ├─ shouldSampleEvent() → 샘플링 결정
    │
    ├─ Datadog HTTP Intake  ← src/services/analytics/datadog.ts
    │
    └─ 1P OpenTelemetry     ← src/services/analytics/firstPartyEventLogger.ts
           │
           └─ FirstPartyEventLoggingExporter (gRPC-over-HTTP)
```

이 설계의 핵심 원칙은 **싱크와 발행자의 완전한 분리**다. `index.ts`는 어떤 외부 의존성도 갖지 않으며, 앱 초기화 전에 발생한 이벤트를 메모리 큐에 보관한 후 싱크 부착 시 `queueMicrotask`를 통해 비동기적으로 소화한다.

---

## 2. 이벤트 발행 레이어 (`index.ts`)

### 2.1 공개 API

```typescript
// src/services/analytics/index.ts

export function logEvent(eventName: string, metadata: LogEventMetadata): void
export async function logEventAsync(eventName: string, metadata: LogEventMetadata): Promise<void>
export function attachAnalyticsSink(newSink: AnalyticsSink): void
```

`LogEventMetadata`는 `{ [key: string]: boolean | number | undefined }` 타입으로 제한된다. **문자열은 의도적으로 제외**되어 있으며, 코드 스니펫이나 파일 경로의 우발적 기록을 타입 시스템 수준에서 방지한다.

### 2.2 마커 타입 (Marker Types)

```typescript
// 개발자가 "이 값이 코드/파일경로가 아님을 확인했다"고 명시할 때 사용
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

// PII 태깅된 proto 컬럼으로 라우팅되는 값 (BQ 특권 접근 컬럼)
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
```

두 타입 모두 `never`로 선언되어 실제 값을 담을 수 없다. 타입 캐스팅(`as`)의 형태로만 사용되며, 코드 리뷰 시 감사 추적 역할을 한다.

### 2.3 `_PROTO_*` 키 처리

```typescript
export function stripProtoFields<V>(metadata: Record<string, V>): Record<string, V>
```

`_PROTO_` 접두사를 가진 키는 PII 태깅된 값으로서 1P 익스포터만 접근할 수 있다. Datadog 전송 전 반드시 `stripProtoFields()`를 호출해야 하며, `sink.ts`가 이 역할을 중앙에서 수행한다.

### 2.4 싱크 큐 드레인 흐름

```typescript
export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // 멱등성 보장
  sink = newSink

  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0

    queueMicrotask(() => {         // 스타트업 레이턴시 차단 방지
      for (const event of queuedEvents) {
        if (event.async) void sink!.logEventAsync(event.eventName, event.metadata)
        else sink!.logEvent(event.eventName, event.metadata)
      }
    })
  }
}
```

`queueMicrotask`를 사용해 큐 드레인이 현재 실행 컨텍스트를 차단하지 않도록 한다.

---

## 3. 분석 비활성화 조건 (`config.ts`)

```typescript
// src/services/analytics/config.ts

export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test' ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isTelemetryDisabled()
  )
}
```

서드파티 클라우드 제공자(Bedrock, Vertex, Foundry)를 사용하는 경우 분석이 자동으로 비활성화된다. 피드백 설문(`isFeedbackSurveyDisabled`)은 로컬 UI 프롬프트로서 서드파티 차단을 적용하지 않는다.

---

## 4. Datadog 백엔드 (`datadog.ts`)

### 4.1 허용 이벤트 화이트리스트

Datadog로 전송되는 이벤트는 `DATADOG_ALLOWED_EVENTS` Set에 명시적으로 등록된 것만 허용된다.

```typescript
const DATADOG_ALLOWED_EVENTS = new Set([
  'tengu_init',
  'tengu_api_success',
  'tengu_api_error',
  'tengu_tool_use_success',
  'tengu_tool_use_error',
  'chrome_bridge_connection_succeeded',
  // ... (총 약 40개 이벤트)
])
```

### 4.2 배치 전송 메커니즘

```
이벤트 → logBatch[] 추가
    │
    ├─ 배치 크기 >= 100 → 즉시 플러시
    └─ 그 외 → 15초 타이머 (scheduleFlush)
```

- `MAX_BATCH_SIZE`: 100
- `DEFAULT_FLUSH_INTERVAL_MS`: 15,000ms
- `NETWORK_TIMEOUT_MS`: 5,000ms
- 엔드포인트: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`

### 4.3 카디널리티 감소 전처리

Datadog 전송 전 다음 변환이 적용된다.

| 필드 | 처리 |
|------|------|
| MCP 도구 이름 (`mcp__*`) | `"mcp"`로 정규화 |
| 모델 이름 (외부 사용자) | 단축 이름으로 정규화, 미등록 모델은 `"other"` |
| 개발 버전 문자열 | 타임스탬프/SHA 제거 (예: `2.0.53-dev.20251124`) |
| HTTP 상태 코드 | `http_status` + `http_status_range` (1xx~5xx)로 분리 |

### 4.4 사용자 버킷 (`getUserBucket`)

사용자 ID를 SHA-256 해시하여 0~29 사이의 버킷 번호에 할당한다. 사용자 ID 직접 노출 없이 영향받은 고유 사용자 수를 근사 추정하는 데 사용된다.

```typescript
const NUM_USER_BUCKETS = 30
const getUserBucket = memoize((): number => {
  const hash = createHash('sha256').update(getOrCreateUserID()).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

### 4.5 태그 필드 목록

다음 필드는 Datadog `ddtags`로 인덱싱된다.

```
arch, clientType, errorType, http_status_range, http_status,
kairosActive, model, platform, provider, skillMode,
subscriptionType, toolName, userBucket, userType, version, versionBase
```

---

## 5. OpenTelemetry + 1P 이벤트 로깅 (`firstPartyEventLogger.ts`, `firstPartyEventLoggingExporter.ts`)

### 5.1 OpenTelemetry 구성

1P 이벤트 로깅은 OpenTelemetry SDK를 사용한다.

```typescript
// src/services/analytics/firstPartyEventLogger.ts
import {
  BatchLogRecordProcessor,
  LoggerProvider,
} from '@opentelemetry/sdk-logs'
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions'
```

`LoggerProvider`에 `FirstPartyEventLoggingExporter`를 `BatchLogRecordProcessor`와 함께 등록한다. 서비스 리소스 속성은 `resourceFromAttributes()`로 정의된다.

### 5.2 이벤트 샘플링

```typescript
// tengu_event_sampling_config GrowthBook 동적 설정에서 로드
export type EventSamplingConfig = {
  [eventName: string]: { sample_rate: number }
}

export function shouldSampleEvent(eventName: string): number | null {
  // null → 100% 로깅 (설정 없음)
  // 0    → 전량 드롭
  // 양수  → 해당 비율로 샘플링, sample_rate를 메타데이터에 추가
}
```

### 5.3 `FirstPartyEventLoggingExporter`

Anthropic의 내부 이벤트 로깅 API(`/api/event_logging/batch`)로 이벤트를 전송하는 커스텀 OpenTelemetry `LogRecordExporter`다.

```typescript
type FirstPartyEventLoggingEvent = {
  event_type: 'ClaudeCodeInternalEvent' | 'GrowthbookExperimentEvent'
  event_data: unknown
}
```

전송 실패 시 이벤트는 `~/.claude/telemetry/` 디렉토리에 JSONL 파일로 저장되어 다음 세션에서 재전송을 시도한다. 파일명에는 `BATCH_UUID`가 포함되어 세션 간 격리를 보장한다.

### 5.4 `_PROTO_*` 키 라우팅

1P 익스포터는 `_PROTO_*` 키를 proto 필드로 호이스팅한 후 `stripProtoFields()`로 나머지 메타데이터에서 제거한다. 이 방식으로 PII 태깅된 값이 일반 접근 스토리지(Datadog, BQ JSON blob)로 유출되는 것을 방지한다.

---

## 6. GrowthBook 피처 플래그 시스템 (`growthbook.ts`)

### 6.1 개요

GrowthBook은 Claude Code의 피처 플래그와 A/B 테스트를 관리하는 외부 서비스다. SDK는 `@growthbook/growthbook` 패키지를 사용한다.

### 6.2 사용자 속성

```typescript
export type GrowthBookUserAttributes = {
  id: string                    // 사용자 UUID
  sessionId: string             // 세션 ID
  deviceID: string              // 기기 ID
  platform: 'win32' | 'darwin' | 'linux'
  organizationUUID?: string
  accountUUID?: string
  userType?: string             // 'ant' | 'external'
  subscriptionType?: string
  rateLimitTier?: string
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

### 6.3 캐시된 조회 패턴

GrowthBook API는 네트워크 레이턴시가 있으므로, 대부분의 조회 함수는 `_CACHED_MAY_BE_STALE` 접미사를 명시적으로 사용한다.

```typescript
checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gateName: string): boolean
getDynamicConfig_CACHED_MAY_BE_STALE<T>(configName: string, defaultValue: T): T
getFeatureValue_CACHED_MAY_BE_STALE(featureName: string): unknown
```

이 명명 규칙은 "이 값은 이전 세션의 캐시일 수 있다"는 사실을 호출자에게 명확히 전달한다. 스타트업 초기에는 이전 세션에서 저장된 캐시를 사용하고, 네트워크 응답 후 갱신된다.

### 6.4 remoteEval 캐시 우회 문제

SDK의 `setForcedFeatures`가 `remoteEval` 응답과 함께 신뢰할 수 없이 동작하는 알려진 문제가 있어, 원격 평가 피처 값에 대한 별도 캐시(`remoteEvalFeatureCache`)를 유지한다.

### 6.5 인증 변경 후 재초기화

인증 상태가 변경될 때(로그인/로그아웃) `refreshGrowthBookAfterAuthChange()`를 호출하여 사용자 속성을 업데이트하고 피처 플래그를 재평가한다. 클라이언트는 인증 여부(`clientCreatedWithAuth`)를 추적하여 필요 시 재생성한다.

---

## 7. 싱크 라우팅 레이어 (`sink.ts`)

```typescript
// src/services/analytics/sink.ts

function logEventImpl(eventName: string, metadata: LogEventMetadata): void {
  const sampleResult = shouldSampleEvent(eventName)
  if (sampleResult === 0) return  // 드롭

  const metadataWithSampleRate = sampleResult !== null
    ? { ...metadata, sample_rate: sampleResult }
    : metadata

  if (shouldTrackDatadog()) {
    // Datadog: _PROTO_* 키 제거 후 전송
    void trackDatadogEvent(eventName, stripProtoFields(metadataWithSampleRate))
  }

  // 1P: _PROTO_* 키 포함 전체 페이로드 전송
  logEventTo1P(eventName, metadataWithSampleRate)
}
```

### 7.1 Datadog 게이트

`tengu_log_datadog_events` GrowthBook 피처 게이트로 Datadog 전송을 제어한다. 게이트 값은 스타트업 시 `initializeAnalyticsGates()`에서 캐시되며, 초기화 전에는 이전 세션 캐시를 폴백으로 사용한다.

### 7.2 킬스위치 (`sinkKillswitch.ts`)

런타임에서 특정 싱크를 비활성화할 수 있는 메커니즘이다. `isSinkKilled('datadog')`가 `true`이면 GrowthBook 게이트와 무관하게 Datadog 전송이 차단된다.

---

## 8. 스타트업 통합 (`main.tsx`)

```typescript
// src/main.tsx - 스타트업 순서

// 1단계: 싱크 초기화 (이벤트 큐 드레인 시작)
initializeAnalyticsSink()

// 2단계: GrowthBook 초기화 (피처 플래그 로드)
await initializeGrowthBook(userAttributes)

// 3단계: Datadog 게이트 확인
initializeAnalyticsGates()
```

GrowthBook 초기화(`initializeGrowthBook`) 전에 발생한 이벤트는 `sink.ts`의 `shouldTrackDatadog()` 함수가 캐시된 이전 세션 값으로 라우팅 결정을 내린다.

---

## 9. 메타데이터 풍부화 (`metadata.ts`)

`getEventMetadata()`는 모든 이벤트에 공통으로 첨부되는 환경 정보를 수집한다.

- 플랫폼 정보 (`platform`, `arch`)
- 앱 버전, 모델명
- 세션 ID, 부모 세션 ID
- 구독 유형, API 제공자
- WSL 버전, Linux 배포판 정보
- Git 저장소 원격 해시 (식별 가능 정보 없이)
- 에이전트 컨텍스트 (팀메이트 여부, 에이전트 ID)

---

## 10. 개인정보 보호 설계 원칙

| 원칙 | 구현 |
|------|------|
| 코드/경로 기록 방지 | `LogEventMetadata`에서 문자열 타입 제외 |
| MCP 서버 설정 보호 | MCP 도구명 `mcp_tool`로 마스킹 |
| PII 격리 | `_PROTO_*` 키 분리 및 특권 컬럼 라우팅 |
| 사용자 ID 직접 노출 방지 | 버킷 해시(`userBucket`) 사용 |
| 서드파티 클라우드 격리 | Bedrock/Vertex/Foundry 시 분석 비활성화 |
| 옵트아웃 지원 | `isTelemetryDisabled()` + 킬스위치 |
