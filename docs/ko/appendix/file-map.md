# 주요 파일 경로 인덱스

> Claude Code `src/` 디렉토리의 핵심 파일 및 서브디렉토리에 대한 한국어 설명과 문서 역참조 테이블이다.

---

## src/ 디렉토리 트리

```
src/
├── main.tsx                         # CLI 진입점. 스타트업 시퀀스, Commander.js 명령어 등록
├── Tool.ts                          # Tool 인터페이스 타입 정의
├── tools.ts                         # 도구 레지스트리. getTools()로 활성 도구 배열 반환
├── commands.ts                      # 슬래시 커맨드 등록 및 라우팅
├── QueryEngine.ts                   # 에이전트 루프 핵심 엔진. LLM → Tool → LLM 반복
├── query.ts                         # 단일 LLM API 쿼리 실행 함수
├── context.ts                       # 시스템/사용자 컨텍스트 구성
├── history.ts                       # 대화 히스토리 관리
├── cost-tracker.ts                  # API 호출 비용 추적
├── costHook.ts                      # 비용 훅 바인딩
├── setup.ts                         # 초기 설정 마법사
├── ink.ts                           # Ink 렌더러 공개 API 재내보내기
├── tasks.ts                         # 태스크 시스템 공개 API
├── Task.ts                          # Task 타입 정의
├── replLauncher.tsx                 # 인터랙티브 REPL 진입점
├── interactiveHelpers.tsx           # 대화 UI 헬퍼 함수
├── dialogLaunchers.tsx              # 모달 다이얼로그 실행기
├── projectOnboardingState.ts        # 프로젝트 온보딩 상태 관리
│
├── assistant/                       # Kairos(어시스턴트) 모드 구현
│   ├── index.ts                     # 어시스턴트 모드 진입점
│   ├── gate.ts                      # Kairos 피처 게이트 체크
│   └── sessionHistory.ts           # 어시스턴트 세션 히스토리
│
├── bootstrap/
│   └── state.ts                     # 앱 전역 상태 (세션 ID, 권한 수락 여부 등)
│
├── bridge/                          # 외부 환경 브릿지 (Chrome, 원격 세션)
│
├── buddy/                           # 버디(짝 프로그래밍) 기능
│
├── cli/                             # CLI 서브커맨드 구현
│
├── commands.ts                      # 슬래시 커맨드 정의
│
├── components/                      # Ink React 컴포넌트
│   └── Spinner.ts                   # 로딩 스피너 컴포넌트
│
├── constants/                       # 상수 정의
│   ├── keys.ts                      # GrowthBook 클라이언트 키 등
│   ├── oauth.ts                     # OAuth 엔드포인트 설정
│   ├── product.ts                   # 제품명, 원격 세션 URL
│   ├── querySource.ts               # 쿼리 소스 타입
│   ├── tools.ts                     # 도구명 상수
│   └── xml.ts                       # XML 태그 상수
│
├── context/                         # React Context 프로바이더
│   ├── fpsMetrics.tsx               # FPS 성능 지표 컨텍스트
│   ├── mailbox.tsx                  # 메시지 메일박스 컨텍스트
│   ├── modalContext.tsx             # 모달 상태 컨텍스트
│   ├── notifications.tsx            # 알림 컨텍스트
│   ├── overlayContext.tsx           # UI 오버레이 컨텍스트
│   ├── promptOverlayContext.tsx     # 프롬프트 오버레이 컨텍스트
│   ├── QueuedMessageContext.tsx     # 대기 메시지 큐 컨텍스트
│   ├── stats.tsx                    # 통계 컨텍스트 (토큰, 비용)
│   └── voice.tsx                    # 음성 입력 컨텍스트
│
├── coordinator/
│   └── coordinatorMode.ts           # 코디네이터 모드 활성화/도구 필터링 로직
│
├── entrypoints/                     # 진입점별 초기화 로직
│   ├── agentSdkTypes.ts             # Agent SDK 타입 정의
│   └── init.ts                      # 공통 초기화 (텔레메트리, 신뢰 다이얼로그)
│
├── hooks/                           # React 커스텀 훅
│   └── useCanUseTool.ts             # 도구 실행 권한 검사 훅
│
├── ink/                             # Ink 터미널 UI 엔진 (커스텀 구현)
│   ├── ink.tsx                      # Ink 코어 렌더러
│   ├── reconciler.ts                # React Reconciler 구현
│   ├── renderer.ts                  # 터미널 렌더링 파이프라인
│   ├── dom.ts                       # 가상 DOM 노드 타입
│   ├── output.ts                    # 출력 버퍼 관리
│   ├── root.ts                      # 루트 컴포넌트
│   ├── screen.ts                    # 터미널 화면 관리
│   ├── terminal.ts                  # 터미널 인터페이스
│   ├── termio.ts                    # 터미널 I/O 제어
│   ├── parse-keypress.ts            # 키 입력 파싱
│   ├── styles.ts                    # 레이아웃 스타일 시스템
│   ├── wrap-text.ts                 # 텍스트 줄바꿈
│   ├── colorize.ts                  # ANSI 색상 처리
│   ├── optimizer.ts                 # 렌더링 최적화
│   ├── selection.ts                 # 텍스트 선택
│   ├── searchHighlight.ts           # 검색어 하이라이트
│   └── vim/                         # Vim 키바인딩 지원
│
├── keybindings/                     # 키 바인딩 시스템
│   ├── schema.ts                    # 키 바인딩 스키마
│   ├── match.ts                     # 키 입력 매칭 로직
│   ├── resolver.ts                  # 바인딩 우선순위 해석
│   └── loadUserBindings.ts          # 사용자 정의 바인딩 로드
│
├── memdir/                          # CLAUDE.md 메모리 디렉토리 시스템
│   ├── memdir.ts                    # 메모리 프롬프트 로드
│   └── paths.ts                     # 메모리 파일 경로 관리
│
├── migrations/                      # 설정 마이그레이션 스크립트
│   ├── migrateLegacyOpusToCurrent.ts
│   ├── migrateSonnet45ToSonnet46.ts
│   └── ...                          # 버전별 마이그레이션
│
├── native-ts/                       # Node.js 네이티브 바인딩
│
├── plugins/                         # 플러그인 시스템
│   └── bundled/                     # 기본 내장 플러그인
│       └── index.ts                 # 빌트인 플러그인 초기화
│
├── query/                           # 쿼리 서브시스템
│
├── remote/                          # 원격 세션 지원
│
├── schemas/                         # Zod 스키마 정의
│
├── screens/                         # 전체 화면 UI 컴포넌트
│
├── server/                          # 로컬 서버 (SDK 모드)
│
├── services/                        # 외부 서비스 통합
│   ├── analytics/                   # 텔레메트리 시스템
│   │   ├── index.ts                 # 공개 API (logEvent, attachAnalyticsSink)
│   │   ├── config.ts                # 분석 비활성화 조건
│   │   ├── sink.ts                  # 이벤트 라우팅 레이어
│   │   ├── datadog.ts               # Datadog HTTP Intake 백엔드
│   │   ├── growthbook.ts            # GrowthBook 피처 플래그
│   │   ├── firstPartyEventLogger.ts # OpenTelemetry 1P 로거
│   │   ├── firstPartyEventLoggingExporter.ts  # 1P HTTP 익스포터
│   │   ├── metadata.ts              # 이벤트 메타데이터 풍부화
│   │   └── sinkKillswitch.ts        # 싱크 킬스위치
│   │
│   ├── api/                         # Anthropic API 클라이언트
│   │   ├── claude.ts                # Claude API 호출
│   │   ├── bootstrap.ts             # 부트스트랩 데이터 페치
│   │   ├── errors.ts                # API 에러 분류
│   │   ├── filesApi.ts              # 파일 API
│   │   └── logging.ts               # API 호출 로깅
│   │
│   ├── mcp/                         # MCP 클라이언트
│   │   ├── client.ts                # MCP 서버 연결 관리
│   │   ├── types.ts                 # MCP 타입 정의
│   │   └── officialRegistry.ts      # 공식 MCP 서버 레지스트리
│   │
│   ├── oauth/                       # OAuth 클라이언트
│   │   └── client.ts                # OAuth 토큰 관리
│   │
│   ├── policyLimits/                # 정책 제한 서비스
│   └── remoteManagedSettings/       # 원격 관리 설정
│
├── skills/                          # 스킬 시스템
│   └── bundled/                     # 기본 내장 스킬
│       └── index.ts                 # 빌트인 스킬 초기화
│
├── state/                           # 앱 상태 관리
│   └── AppState.ts                  # 전역 앱 상태 타입
│
├── tasks/                           # 태스크 시스템
│   ├── types.ts                     # 태스크 타입 정의
│   ├── LocalMainSessionTask.ts      # 로컬 세션 태스크
│   └── stopTask.ts                  # 태스크 중지 로직
│
├── tools/                           # 도구 구현체
│   ├── AgentTool/                   # 서브에이전트 실행 도구
│   ├── BashTool/                    # 배시 명령어 실행
│   ├── FileEditTool/                # 파일 편집 (Edit)
│   ├── FileReadTool/                # 파일 읽기 (Read)
│   ├── FileWriteTool/               # 파일 쓰기 (Write)
│   ├── GlobTool/                    # 파일 패턴 검색
│   ├── GrepTool/                    # 내용 검색
│   ├── LSPTool/                     # LSP 코드 지능
│   ├── MCPTool/                     # MCP 프록시 도구
│   ├── NotebookEditTool/            # Jupyter 노트북 편집
│   ├── SkillTool/                   # 스킬 실행 래퍼
│   ├── TodoWriteTool/               # 할 일 목록 관리
│   ├── WebFetchTool/                # URL 페치
│   ├── WebSearchTool/               # 웹 검색
│   ├── EnterWorktreeTool/           # Git Worktree 진입
│   ├── ExitWorktreeTool/            # Git Worktree 종료
│   ├── SyntheticOutputTool/         # 합성 출력 도구 (SDK)
│   └── TaskStopTool/                # 태스크 종료 도구
│
├── types/                           # 공유 타입 정의
│   ├── command.ts                   # 커맨드 타입
│   ├── hooks.ts                     # 훅 타입
│   ├── ids.ts                       # UUID 타입 별칭
│   ├── logs.ts                      # 로그 옵션 타입
│   ├── permissions.ts               # 권한 모드/결과 타입
│   ├── plugin.ts                    # 플러그인 타입
│   ├── textInputTypes.ts            # 텍스트 입력 타입
│   └── generated/                   # protobuf 생성 타입
│
├── upstreamproxy/                   # 업스트림 프록시 지원
│   ├── upstreamproxy.ts             # 프록시 설정 파싱
│   └── relay.ts                     # 프록시 릴레이
│
├── utils/                           # 유틸리티 함수
│   ├── auth.ts                      # 인증 (OAuth, 구독 타입)
│   ├── config.ts                    # 전역 설정 읽기/쓰기
│   ├── context.ts                   # 컨텍스트 윈도우 계산
│   ├── cwd.ts                       # 현재 작업 디렉토리
│   ├── debug.ts                     # 디버그 로깅
│   ├── env.ts                       # 환경 변수 헬퍼
│   ├── envUtils.ts                  # 환경 유틸리티 (isEnvTruthy 등)
│   ├── errors.ts                    # 에러 처리 유틸리티
│   ├── git.ts                       # Git 유틸리티
│   ├── log.ts                       # 로그 함수
│   ├── messages.ts                  # 메시지 생성 유틸리티
│   ├── model/                       # 모델 관련 유틸리티
│   │   ├── model.ts                 # 모델 이름 처리
│   │   ├── providers.ts             # API 제공자 감지
│   │   └── deprecation.ts           # 모델 지원 종료 경고
│   ├── permissions/                 # 권한 검사 로직
│   ├── platform.ts                  # 플랫폼 감지 (WSL, distro)
│   ├── secureStorage/               # 키체인/보안 스토리지
│   │   └── keychainPrefetch.ts      # 스타트업 키체인 프리페치
│   ├── settings/                    # 설정 시스템
│   │   ├── mdm/                     # MDM 설정 로드
│   │   │   └── rawRead.ts           # MDM 원시 읽기 (병렬 실행)
│   │   └── changeDetector.ts        # 설정 변경 감지
│   ├── signal.ts                    # 신호 유틸리티
│   ├── slowOperations.ts            # JSON 파싱/직렬화 (성능 주의 표시)
│   ├── startupProfiler.ts           # 스타트업 성능 프로파일러
│   ├── swarm/                       # 에이전트 스웜 유틸리티
│   ├── teammate.ts                  # 팀메이트 에이전트 유틸리티
│   └── thinking.ts                  # 확장 사고(extended thinking) 설정
│
├── vim/                             # Vim 키바인딩 엔진
│   ├── motions.ts                   # 이동 명령
│   ├── operators.ts                 # 편집 연산자
│   ├── textObjects.ts               # 텍스트 객체
│   ├── transitions.ts               # 모드 전환
│   └── types.ts                     # Vim 타입 정의
│
└── voice/                           # 음성 입력 지원
```

---

## 파일 → 문서 역참조 테이블

| 소스파일 | 관련 문서 |
|---------|----------|
| `src/services/analytics/index.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §2 |
| `src/services/analytics/config.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §3 |
| `src/services/analytics/datadog.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §4 |
| `src/services/analytics/growthbook.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §6, [용어집](./glossary.md#피처-플래그--ab-테스트) |
| `src/services/analytics/sink.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §7 |
| `src/services/analytics/firstPartyEventLogger.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §5 |
| `src/services/analytics/firstPartyEventLoggingExporter.ts` | [텔레메트리 시스템](../level-3-internals/telemetry.md) §5.3 |
| `src/main.tsx` | [설계 패턴](./design-patterns.md) §1, §2, §3 |
| `src/tools.ts` | [설계 패턴](./design-patterns.md) §4, [용어집](./glossary.md#핵심-아키텍처) |
| `src/Tool.ts` | [용어집](./glossary.md#핵심-아키텍처) |
| `src/QueryEngine.ts` | [설계 패턴](./design-patterns.md) §5, [용어집](./glossary.md#핵심-아키텍처) |
| `src/types/permissions.ts` | [용어집](./glossary.md#권한-시스템), [설계 패턴](./design-patterns.md) §7 |
| `src/coordinator/coordinatorMode.ts` | [용어집](./glossary.md#핵심-아키텍처) |
| `src/utils/secureStorage/keychainPrefetch.ts` | [설계 패턴](./design-patterns.md) §1 |
| `src/utils/settings/mdm/rawRead.ts` | [설계 패턴](./design-patterns.md) §1 |
| `src/ink/` | [용어집](./glossary.md#uirendering), [설계 패턴](./design-patterns.md) §6 |
| `src/services/mcp/` | [용어집](./glossary.md#외부-서비스--프로토콜) |
| `src/utils/teammate.ts` | [용어집](./glossary.md#핵심-아키텍처) |
| `src/memdir/memdir.ts` | [용어집](./glossary.md#컨텍스트--메모리) |
| `src/services/analytics/sinkKillswitch.ts` | [용어집](./glossary.md#분석--모니터링) |

---

## 주요 진입점 요약

| 진입점 | 파일 | 설명 |
|--------|------|------|
| CLI 메인 | `src/main.tsx` | Commander.js 기반 CLI 명령어 파싱 및 스타트업 |
| REPL | `src/replLauncher.tsx` | 인터랙티브 대화 세션 진입 |
| SDK 서버 | `src/server/` | Agent SDK 연동 서버 모드 |
| 어시스턴트 모드 | `src/assistant/index.ts` | Kairos 어시스턴트 모드 (피처 플래그 제어) |
| 초기화 | `src/entrypoints/init.ts` | 텔레메트리, 신뢰 다이얼로그, 공통 초기화 |
