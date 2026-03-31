# 플러그인 & 스킬 확장 시스템 분석

## 1. 개요

Claude Code의 확장 시스템은 두 개의 독립적이지만 상호 연관된 레이어로 구성된다: **플러그인 시스템(Plugin System)**과 **스킬 시스템(Skill System)**. 두 시스템 모두 CLI 바이너리 외부에서 기능을 추가하거나 내장 기능을 사용자가 제어할 수 있도록 설계된 확장 메커니즘이다.

**시스템 간 주요 차이점:**

| 구분 | 플러그인 시스템 | 스킬 시스템 |
|------|---------------|------------|
| 사용자 제어 | `/plugin` UI에서 활성화/비활성화 가능 | 항상 활성화(번들) 또는 디렉토리 기반 로딩 |
| 진입점 | `builtinPlugins.ts` 레지스트리 | `bundledSkills.ts` 레지스트리 또는 `loadSkillsDir.ts` |
| 식별자 형식 | `{name}@builtin` (내장) / `{name}@{marketplace}` (마켓플레이스) | 스킬 이름 (예: `commit`, `review-pr`) |
| 구성 요소 | 스킬, 훅(hooks), MCP 서버를 복합 제공 가능 | 단일 스킬 단위 |
| 현재 상태 | 스캐폴딩(scaffolding) 단계 — 등록된 내장 플러그인 없음 | 10개 이상의 번들 스킬 제공 |

**파일 구조 요약:**

```
src/plugins/
  builtinPlugins.ts        — 내장 플러그인 레지스트리 및 쿼리 함수
  bundled/
    index.ts               — initBuiltinPlugins() 진입점 (현재 빈 스캐폴딩)

src/skills/
  bundledSkills.ts         — 번들 스킬 레지스트리, 파일 추출 로직
  loadSkillsDir.ts         — 디스크 기반 스킬 로딩, 프런트매터 파싱
  mcpSkillBuilders.ts      — MCP 스킬 빌더 싱글턴 레지스트리
  bundled/
    index.ts               — initBundledSkills() 진입점
    remember.ts, simplify.ts, verify.ts, ... — 개별 스킬 구현체

src/tools/SkillTool/
  SkillTool.ts             — 스킬 실행 도구 본체
  constants.ts             — SKILL_TOOL_NAME 상수
  prompt.ts                — 스킬 목록 프롬프트 생성 및 예산 관리
  UI.tsx                   — 렌더링 컴포넌트
```

---

## 2. 플러그인 시스템

### 2.1 플러그인 로딩 메커니즘

플러그인 시스템은 시작 시점에 `initBuiltinPlugins()`를 호출하여 초기화된다. 이 함수는 `src/plugins/bundled/index.ts`에 위치하며, 각 플러그인은 `registerBuiltinPlugin()`을 통해 중앙 `Map<string, BuiltinPluginDefinition>` 레지스트리에 등록된다.

```typescript
// builtinPlugins.ts
const BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition> = new Map()

export function registerBuiltinPlugin(
  definition: BuiltinPluginDefinition,
): void {
  BUILTIN_PLUGINS.set(definition.name, definition)
}
```

플러그인 활성화 여부는 다음 우선순위로 결정된다:

1. **사용자 설정** (`settings.enabledPlugins[pluginId]`): 명시적으로 `true`/`false` 지정 시 최우선 적용
2. **플러그인 기본값** (`definition.defaultEnabled`): 사용자 설정이 없을 때 플러그인이 선언한 기본값
3. **시스템 기본값** (`true`): 위 두 값이 모두 없을 때 활성화 상태가 기본

플러그인 가용성 필터링은 `isAvailable()` 콜백으로 처리된다. 이 콜백이 `false`를 반환하면 해당 플러그인은 활성화/비활성화 목록 모두에서 제외된다:

```typescript
for (const [name, definition] of BUILTIN_PLUGINS) {
  if (definition.isAvailable && !definition.isAvailable()) {
    continue  // 조건 미충족 시 목록에서 완전히 제외
  }
  // ...
}
```

### 2.2 번들 플러그인 구조

`src/plugins/bundled/index.ts`는 현재 빈 스캐폴딩 상태다. 주석에 따르면 이 구조는 기존 번들 스킬 중 사용자 토글이 필요한 것들을 마이그레이션하기 위한 준비 단계다:

```typescript
// bundled/index.ts
export function initBuiltinPlugins(): void {
  // No built-in plugins registered yet — this is the scaffolding for
  // migrating bundled skills that should be user-toggleable.
}
```

새 내장 플러그인 추가 절차는 주석에 명시되어 있다:
1. `builtinPlugins.ts`에서 `registerBuiltinPlugin` 임포트
2. `initBuiltinPlugins()` 내에서 `registerBuiltinPlugin()` 호출

### 2.3 플러그인 확장 포인트

`BuiltinPluginDefinition` 타입은 플러그인이 제공할 수 있는 세 가지 확장 포인트를 정의한다:

| 확장 포인트 | 타입 | 설명 |
|------------|------|------|
| `skills` | `BundledSkillDefinition[]` | 스킬 목록 (플러그인이 비활성화되면 스킬도 비활성화) |
| `hooks` | `HooksConfig` | 전처리/후처리 훅 설정 |
| `mcpServers` | `MCPServerConfig[]` | MCP 서버 자동 연결 설정 |

플러그인에서 제공된 스킬은 `skillDefinitionToCommand()` 함수를 통해 `Command` 객체로 변환된다. 이 변환 과정에서 `source: 'bundled'`로 설정되는 것이 중요한 설계 결정이다:

```typescript
// builtinPlugins.ts - skillDefinitionToCommand()
return {
  // ...
  // 'bundled' not 'builtin' — 'builtin' in Command.source means hardcoded
  // slash commands (/help, /clear). Using 'bundled' keeps these skills in
  // the Skill tool's listing, analytics name logging, and prompt-truncation
  // exemption. The user-toggleable aspect is tracked on LoadedPlugin.isBuiltin.
  source: 'bundled',
  loadedFrom: 'bundled',
}
```

이 설계를 통해 플러그인 스킬은 스킬 목록에서 번들 스킬과 동일하게 취급되며, 프롬프트 예산 우선 배정과 분석 이름 로깅 대상에 포함된다.

### 2.4 LoadedPlugin 구조체

`getBuiltinPlugins()`가 반환하는 `LoadedPlugin` 객체는 다음 주요 필드를 포함한다:

```typescript
const plugin: LoadedPlugin = {
  name,
  manifest: { name, description, version },
  path: BUILTIN_MARKETPLACE_NAME, // 'builtin' — 파일시스템 경로 없음
  source: pluginId,               // '{name}@builtin'
  repository: pluginId,
  enabled: isEnabled,
  isBuiltin: true,
  hooksConfig: definition.hooks,
  mcpServers: definition.mcpServers,
}
```

`path: 'builtin'`은 센티넬(sentinel) 값으로, 파일시스템 경로가 없는 내장 플러그인임을 나타낸다.

---

## 3. 스킬 시스템

### 3.1 스킬 로딩 메커니즘

스킬 로딩은 두 가지 경로로 진행된다:

**경로 1 — 번들 스킬 (programmatic registration):**
시작 시 `initBundledSkills()`가 호출되어 각 번들 스킬의 `register*()` 함수를 순차 호출한다. 각 `register*()` 함수는 `registerBundledSkill(definition)`을 호출하여 모듈 수준의 `bundledSkills: Command[]` 배열에 추가한다.

**경로 2 — 디스크 기반 스킬 (file-system loading):**
`loadSkillsDir.ts`의 `loadSkillsFromSkillsDir()`가 다음 경로들을 순회하여 스킬을 로드한다:

| 소스 | 경로 |
|------|------|
| `policySettings` | `{managedFilePath}/.claude/skills/` |
| `userSettings` | `{claudeConfigHomeDir}/skills/` |
| `projectSettings` | `.claude/skills/` |
| `plugin` | 플러그인 디렉토리 |

파일 기반 스킬은 `{skill-name}/SKILL.md` 형식의 디렉토리 구조를 따른다. 단일 `.md` 파일은 `/skills/` 디렉토리에서 지원되지 않는다.

`getSkillsPath()` 함수는 소스별 경로 해석을 담당한다:

```typescript
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string {
  switch (source) {
    case 'policySettings': return join(getManagedFilePath(), '.claude', dir)
    case 'userSettings':   return join(getClaudeConfigHomeDir(), dir)
    case 'projectSettings': return `.claude/${dir}`
    case 'plugin':          return 'plugin'
    default:                return ''
  }
}
```

### 3.2 번들 스킬 목록

`src/skills/bundled/index.ts`의 `initBundledSkills()`는 다음 스킬들을 등록한다:

| 스킬 | 파일 | 특이사항 |
|------|------|---------|
| `update-config` | `updateConfig.ts` | 설정 파일 업데이트 |
| `keybindings-help` | `keybindings.ts` | 키 바인딩 설명 |
| `verify` | `verify.ts` | 코드 검증 |
| `debug` | `debug.ts` | 디버깅 보조 |
| `lorem-ipsum` | `loremIpsum.ts` | 더미 텍스트 생성 |
| `skillify` | `skillify.ts` | 스킬 자동 생성 |
| `remember` | `remember.ts` | 메모리 레이어 검토 |
| `simplify` | `simplify.ts` | 코드 단순화 |
| `batch` | `batch.ts` | 배치 작업 |
| `stuck` | `stuck.ts` | 막힌 상황 탈출 보조 |

기능 플래그(feature flag)로 제어되는 조건부 스킬도 존재한다:

```typescript
if (feature('KAIROS') || feature('KAIROS_DREAM')) {
  const { registerDreamSkill } = require('./dream.js')
  registerDreamSkill()
}
if (feature('AGENT_TRIGGERS')) {
  const { registerLoopSkill } = require('./loop.js')
  registerLoopSkill()
}
if (feature('BUILDING_CLAUDE_APPS')) {
  const { registerClaudeApiSkill } = require('./claudeApi.js')
  registerClaudeApiSkill()
}
```

조건부 스킬에 `require()`를 사용하는 이유는 정적 임포트(static import)를 사용할 경우 트리 셰이킹(tree-shaking)을 통과하는 사이드 이펙트(side-effecting initializer)가 Bun 번들 바이너리에 포함되어 불필요한 모듈 초기화가 발생하기 때문이다.

### 3.3 번들 스킬의 파일 추출 메커니즘

`BundledSkillDefinition`의 `files` 필드를 통해 번들 스킬은 추가 참조 파일을 디스크에 추출할 수 있다. 이 메커니즘은 모델이 `Read`/`Grep` 도구로 스킬 파일을 접근할 수 있도록 허용한다:

```typescript
export type BundledSkillDefinition = {
  // ...
  files?: Record<string, string>  // 상대경로 → 파일 내용
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

`files`가 존재할 경우 `registerBundledSkill()`은 `getPromptForCommand`를 래핑하여 지연(lazy) 추출을 수행한다:

```typescript
// 프로세스당 한 번만 추출 — Promise 메모이제이션으로 경쟁 조건 방지
let extractionPromise: Promise<string | null> | undefined
getPromptForCommand = async (args, ctx) => {
  extractionPromise ??= extractBundledSkillFiles(definition.name, files)
  const extractedDir = await extractionPromise
  const blocks = await inner(args, ctx)
  if (extractedDir === null) return blocks
  return prependBaseDir(blocks, extractedDir)
}
```

**보안 설계:** 파일 추출 시 경로 순회 공격(path traversal)을 방지하기 위해 엄격한 검증을 적용한다:

```typescript
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (
    isAbsolute(normalized) ||
    normalized.split(pathSep).includes('..') ||
    normalized.split('/').includes('..')
  ) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

파일 쓰기는 `O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW` 플래그와 `0o600` 퍼미션으로 수행된다. `getBundledSkillsRoot()`의 프로세스별 난수(nonce)가 심볼릭 링크 선점 공격에 대한 1차 방어선이며, `O_NOFOLLOW | O_EXCL`는 추가 방어층이다.

### 3.4 스킬 프런트매터 파싱

디스크 기반 스킬의 `SKILL.md`는 YAML 프런트매터를 통해 메타데이터를 정의한다. `parseSkillFrontmatterFields()`가 처리하는 주요 필드:

| 프런트매터 키 | 타입 | 설명 |
|-------------|------|------|
| `description` | string | 스킬 설명 |
| `when_to_use` | string | 모델에게 제공하는 사용 시점 힌트 |
| `allowed-tools` | string[] | 허용된 도구 목록 |
| `argument-hint` | string | 인수 힌트 |
| `arguments` | string \| string[] | 인수 이름 목록 |
| `model` | string | 모델 오버라이드 (`inherit`이면 undefined) |
| `user-invocable` | boolean | 사용자 직접 호출 가능 여부 |
| `disable-model-invocation` | boolean | SkillTool 사용 비활성화 |
| `context` | `'fork'` | 실행 컨텍스트 (`fork` 시 서브에이전트 실행) |
| `agent` | string | 에이전트 타입 지정 |
| `effort` | string \| number | 에포트(effort) 레벨 |
| `paths` | string[] | 적용 경로 패턴 |
| `hooks` | HooksSettings | 스킬별 훅 설정 |

`paths` 필드는 `ignore` 라이브러리를 활용하여 glob 패턴 매칭을 수행한다. `/**` 접미사는 자동 제거되고 `**` 패턴만 있으면 `undefined`(전체 적용)로 처리된다.

### 3.5 MCP 기반 스킬 빌더

`mcpSkillBuilders.ts`는 모듈 간 순환 의존성(circular dependency) 문제를 해결하기 위한 쓰기 전용(write-once) 레지스트리 패턴을 구현한다.

**문제 배경:**
```
client.ts → mcpSkills.ts → loadSkillsDir.ts → ... → client.ts
```
이 의존성 그래프는 순환을 형성한다. `loadSkillsDir.ts`를 동적 임포트(`await import(variable)`)로 처리하면 Bun 번들 바이너리에서 `/$bunfs/root/…` 경로로 해석되어 런타임 오류가 발생한다.

**해결 방법 — 쓰기 전용 레지스트리:**
```typescript
// mcpSkillBuilders.ts — 의존성 그래프의 리프(leaf) 노드
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error(
      'MCP skill builders not registered — loadSkillsDir.ts has not been evaluated yet',
    )
  }
  return builders
}
```

`loadSkillsDir.ts`가 모듈 초기화 시점에 `registerMCPSkillBuilders()`를 호출하여 빌더 함수를 등록한다. `commands.ts`의 정적 임포트를 통해 시작 시점에 항상 평가되므로, MCP 서버가 연결되기 훨씬 전에 등록이 완료된다.

---

## 4. SkillTool 구현 분석

### 4.1 SkillTool 개요

`SkillTool`은 Claude Code의 스킬 실행 전용 도구(`name: 'Skill'`)다. 모델은 이 도구를 통해 슬래시 커맨드(slash command) 스킬을 호출한다.

**입력 스키마:**
```typescript
z.object({
  skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
  args: z.string().optional().describe('Optional arguments for the skill'),
})
```

**출력 스키마 (유니언 타입):**
```typescript
// 인라인 실행 결과
z.object({
  success: z.boolean(),
  commandName: z.string(),
  allowedTools: z.array(z.string()).optional(),
  model: z.string().optional(),
  status: z.literal('inline').optional(),
})

// 포크 실행 결과
z.object({
  success: z.boolean(),
  commandName: z.string(),
  status: z.literal('forked'),
  agentId: z.string(),
  result: z.string(),
})
```

### 4.2 커맨드 조회 및 MCP 스킬 통합

`getAllCommands()`는 로컬/번들 스킬과 MCP 스킬을 통합한다:

```typescript
async function getAllCommands(context: ToolUseContext): Promise<Command[]> {
  const mcpSkills = context
    .getAppState()
    .mcp.commands.filter(
      cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp',
    )
  if (mcpSkills.length === 0) return getCommands(getProjectRoot())
  const localCommands = await getCommands(getProjectRoot())
  return uniqBy([...localCommands, ...mcpSkills], 'name')
}
```

MCP 스킬(`loadedFrom === 'mcp'`)만 포함하고 일반 MCP 프롬프트는 제외한다. 이전에는 모델이 `mcp__server__prompt` 이름을 추측하여 일반 MCP 프롬프트를 SkillTool로 호출할 수 있었으나, 이 필터로 차단된다.

### 4.3 입력 검증 (validateInput)

검증은 다음 단계로 진행된다:

1. **빈 스킬 이름 검사** — `errorCode: 1`
2. **슬래시 접두사 정규화** — `/commit` → `commit` (이벤트 로깅 후 정규화)
3. **원격 스킬 처리** (`EXPERIMENTAL_SKILL_SEARCH`) — `_canonical_<slug>` 형식은 별도 경로 처리
4. **커맨드 존재 확인** — `findCommand()` 실패 시 `errorCode: 2`
5. **`disableModelInvocation` 확인** — 모델 호출 비활성화 스킬 거부 (`errorCode: 4`)
6. **프롬프트 타입 확인** — `type !== 'prompt'` 거부 (`errorCode: 5`)

### 4.4 실행 컨텍스트: 인라인 vs. 포크

스킬 실행은 `command.context` 필드에 따라 두 가지 모드로 분기된다:

**인라인 실행 (기본값):**
스킬 프롬프트를 현재 대화의 다음 응답으로 주입한다. 모델은 스킬 내용을 읽고 현재 컨텍스트에서 직접 응답한다. `SkillTool`이 `{ success: true, status: 'inline', allowedTools, model }` 결과를 반환하면 호출 루프가 허용된 도구와 모델을 적용하여 다음 턴을 실행한다.

**포크 실행 (`context: 'fork'`):**
`executeForkedSkill()`이 독립적인 서브에이전트를 생성하여 스킬을 실행한다:

```typescript
async function executeForkedSkill(
  command, commandName, args, context, canUseTool, parentMessage, onProgress
): Promise<ToolResult<Output>> {
  const agentId = createAgentId()
  // prepareForkedCommandContext()로 서브에이전트 환경 구성
  const { modifiedGetAppState, baseAgent, promptMessages, skillContent } =
    await prepareForkedCommandContext(command, args || '', context)

  // 스킬의 effort를 에이전트 정의에 병합
  const agentDefinition = command.effort !== undefined
    ? { ...baseAgent, effort: command.effort }
    : baseAgent

  for await (const message of runAgent({ agentDefinition, promptMessages, ... })) {
    agentMessages.push(message)
    // 도구 사용 진행 상황을 onProgress로 보고
  }

  return { data: { success: true, status: 'forked', agentId, result: resultText } }
}
```

포크 실행의 특성:
- 독립적인 토큰 예산을 가진 격리된 에이전트 실행
- 완료 후 결과 텍스트만 상위 컨텍스트에 반환
- 종료 시 `clearInvokedSkillsForAgent(agentId)`로 메모리 해제

### 4.5 스킬 목록 프롬프트 및 예산 관리

`prompt.ts`의 `formatCommandsWithinBudget()`은 스킬 목록을 컨텍스트 창 예산 내에서 포맷한다:

**예산 계산:**
```typescript
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 컨텍스트 창의 1%
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000           // 200k 컨텍스트 × 4 × 1%

export function getCharBudget(contextWindowTokens?: number): number {
  if (Number(process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET)) {
    return Number(process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET)
  }
  if (contextWindowTokens) {
    return Math.floor(contextWindowTokens * CHARS_PER_TOKEN * SKILL_BUDGET_CONTEXT_PERCENT)
  }
  return DEFAULT_CHAR_BUDGET
}
```

**예산 초과 시 우선순위 처리:**

번들 스킬(`source === 'bundled'`)은 항상 전체 설명이 보존되며, 나머지 스킬에 잔여 예산이 배분된다. 극단적 예산 부족 시에는 비번들 스킬을 이름만 표시하는 축약 모드로 전환한다:

```
// 정상: - commit: Commit staged changes with a conventional message
// 예산 부족: - commit
```

**항목당 최대 설명 길이:**
```typescript
export const MAX_LISTING_DESC_CHARS = 250
```
`whenToUse`와 `description`을 결합한 문자열이 250자를 초과하면 말줄임표(`…`)로 절단된다.

### 4.6 보안: MCP 스킬의 셸 실행 차단

`createSkillCommand()`의 `getPromptForCommand` 구현에서 MCP 스킬은 인라인 셸 실행(`!``...```)이 명시적으로 차단된다:

```typescript
// Security: MCP skills are remote and untrusted — never execute inline
// shell commands (!`…` / ```! … ```) from their markdown body.
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

로컬 스킬(번들, 사용자, 프로젝트, 플러그인)만 셸 실행(`executeShellCommandsInPrompt`)을 허용한다.

---

## 5. 설계 결정

### 5.1 플러그인과 번들 스킬의 이원화

현재 아키텍처는 사용자 토글 가능 여부에 따라 플러그인과 번들 스킬을 명확히 분리한다. 이 결정은 `src/plugins/bundled/index.ts`의 주석에 명시되어 있다:

> "Not all bundled features should be built-in plugins — use this for features that users should be able to explicitly enable/disable. For features with complex setup or automatic-enabling logic (e.g. claude-in-chrome), use src/skills/bundled/ instead."

`claude-in-chrome`처럼 환경 자동 감지(`shouldAutoEnableClaudeInChrome()`)로 활성화되는 스킬은 플러그인 시스템이 아닌 번들 스킬 시스템에 남아있다.

### 5.2 Command.source 필드의 의미론적 일관성

`source: 'bundled'` vs `source: 'builtin'`의 구분은 미묘하지만 중요하다:
- `'builtin'`: 하드코딩된 슬래시 커맨드(`/help`, `/clear` 등) — `Command` 시스템 외부에 위치
- `'bundled'`: 번들 스킬 — `Command` 객체로 등록되어 스킬 목록, 분석, 프롬프트 예산 로직에 포함

플러그인 스킬이 `source: 'bundled'`를 사용하는 이유는 이 값이 세 가지 시스템 동작을 결정하기 때문이다: (1) 스킬 목록 표시, (2) 분석 이벤트 이름 로깅, (3) 프롬프트 예산에서 설명 보존 우선순위.

### 5.3 싱글턴 도구 실행 정책

`SkillTool`은 `toAutoClassifierInput: ({ skill }) => skill ?? ''`을 정의하여 자동 분류기(auto-classifier)에 스킬 이름만 전달한다. 주석에 따르면 한 번에 하나의 스킬만 실행되어야 한다:

> "Only one skill/command should run at a time, since the tool expands the command into a full prompt that Claude must process before continuing."

이 제약은 스킬 확장 프롬프트가 다음 턴 처리 전에 완전히 평가되어야 한다는 구조적 요구에서 비롯된다.

### 5.4 지연 로딩과 메모이제이션 전략

번들 스킬의 파일 추출은 두 가지 레벨의 지연 로딩을 적용한다:

1. **첫 호출 시 추출** (`??=` 연산자로 Promise 메모이제이션): 동일 프로세스에서 동시 호출이 와도 단일 쓰기 작업만 수행
2. **실패 시 계속 실행** (`null` 반환): 파일 쓰기 실패가 스킬 실행을 중단하지 않음

스킬 목록 프롬프트는 `memoize(async (_cwd: string) => ...)` 패턴으로 캐시된다. `_cwd` 파라미터는 캐시 키로만 사용되며, 작업 디렉토리별로 별도 캐시가 유지된다.

### 5.5 로딩 출처(LoadedFrom) 추적

`LoadedFrom` 타입은 스킬의 출처를 세분화하여 추적한다:

```typescript
export type LoadedFrom =
  | 'commands_DEPRECATED'  // 레거시 /commands/ 디렉토리
  | 'skills'               // .claude/skills/ 디렉토리
  | 'plugin'               // 플러그인 제공
  | 'managed'              // 정책 관리자 배포
  | 'bundled'              // CLI 내장
  | 'mcp'                  // MCP 서버 제공
```

이 값은 보안 정책(셸 실행 허용 여부), 분석 이벤트 메타데이터, 스킬 중복 제거 로직에서 활용된다.

---

## 내비게이션

- 이전: [level-3-internals 개요](../README.md)
- 상위: [목차](../README.md)
