# 5장 도구 상세 설명 - AgentTool 및 다중 Agent

> **저자**: [Wang Yanshu](https://catyans.github.io)

> 이 장에서는 AgentTool(17개 파일), TeamCreateTool, SendMessageTool 및 Coordinator 패턴과 다중 Agent 메시지 라우팅을 분석합니다.

## 1. AgentTool 입력 Schema

```typescript
const AgentInputSchema = z.object({
  // 작업 설명(필수)
  description: z.string().describe("Agent 임무에 대한 간략한 설명"),

  // 하위 Agent에 전달할 전체 프롬프트
  prompt: z.string().describe("자세한 작업 지침"),

  // 하위 Agent 유형
  subagent_type: z.enum(['research', 'code', 'review', 'custom']).optional(),

  // 모델 선택
  model: z.string().optional().describe("기본 모델 재정의"),

  // 백그라운드에서 실행 중
  run_in_background: z.boolean().optional().describe("백그라운드에서 실행할지 여부"),

  // 이름
  name: z.string().optional().describe("Agent의 고유 이름"),

  // 팀 이름(Coordinator 패턴)
  team_name: z.string().optional(),

  // 작동 모드
  mode: z.enum(['sync', 'async']).optional(),

  // 격리 수준
  isolation: z.enum(['none', 'worktree', 'fork']).optional(),

  // 작업 디렉토리
  cwd: z.string().optional(),
});
```

### subagent_type이 필요한 이유

`subagent_type`는 단순한 라벨 그 이상이며 다음에 영향을 미칩니다.

- **research**: 읽기 전용 도구 세트(Grep, Glob, FileRead, WebSearch), 파일을 수정할 수 없습니다.
- **code**: 파일을 편집하고 명령을 실행할 수 있는 완전한 도구 세트
- **review**: 읽기 전용 + 코드 검토용 Diff 도구
- **custom**: `allowedTools` 매개변수로 지정됨

이 분류는 최소 권한 원칙에 따라 하위 Agent의 기능 범위를 제한합니다.

## 2. Agent 생성 과정

```
모델이 AgentTool 호출
    │
    ▼
정의 해석(Resolve Definition)
    │ agentDefinitions에서 찾거나 만들기
    ▼
MCP 요구 사항 확인
    │ 하위 Agent에 필요한 MCP 서버 확인
    ▼
도구 풀 구성
    │ subagent_type에 따라 도구 필터링
    ▼
생성(Spawn)
    ├─ sync: 현재 프로세스에서 실행
    ├─ async: 백그라운드 작업 생성
    └─ fork: spawn 하위 프로세스(FORK_AGENT feature)
    │
    ▼
자동 백그라운드 전환
    │ 120초 이상 완료되지 않음 → 백그라운드 작업으로 전환
    ▼
결과 수집
```

### 2.1 정의 분석

```typescript
function resolveAgentDefinition(
  input: AgentInput,
  existingDefs: AgentDefinition[]
): AgentDefinition {
  // 이름이 지정되면 기존 정의를 검색합니다.
  if (input.name) {
    const existing = existingDefs.find(d => d.name === input.name);
    if (existing) return existing;
  }

  // 새 정의 만들기
  return {
    name: input.name || generateUniqueName(),
    type: input.subagent_type || 'custom',
    model: input.model,
    allowedTools: getToolsForType(input.subagent_type),
    systemPrompt: buildAgentSystemPrompt(input),
  };
}
```

### 2.2 MCP 요청 확인

```typescript
async function checkMCPRequirements(
  def: AgentDefinition,
  mcpClients: MCPClient[]
): Promise<void> {
  if (def.mcpServerRequirements) {
    for (const serverName of def.mcpServerRequirements) {
      const client = mcpClients.find(c => c.serverName === serverName);
      if (!client) {
        throw new Error(`Agent requires MCP server '${serverName}' which is not available`);
      }
      if (!client.isConnected()) {
        await client.connect();
      }
    }
  }
}
```

## 3. runAgent 생성기

```typescript
async function* runAgent(
  def: AgentDefinition,
  input: AgentInput,
  context: ToolUseContext
): AsyncGenerator<AgentEvent, AgentResult> {
  // 1. MCP 서버 초기화
  const mcpServers = await initAgentMCPServers(def, context);

  // 2. 하위 Agent 컨텍스트 생성
  const subContext = createSubagentContext(def, context, mcpServers);

  // 3. 트리거 후크
  yield* executeHooks('subagent_start', { agent: def, input });

  // 4. 쿼리 루프
  const queryEngine = new QueryEngine(subContext);
  const result = yield* queryEngine.run(input.prompt);

  // 5. transcript 기록
  await recordTranscript(def.name, result.messages);

  // 6. 청소
  await cleanupMCPServers(mcpServers, def);

  return result;
}
```

## 4. Agent MCP 서버 초기화

MCP 서버에는 다양한 수명주기 관리에 해당하는 두 가지 참조 방법이 있습니다.

### 4.1 이름으로 참조(공유)

```typescript
// 기존 MCP 서버 참조
{
  mcpServers: [{ ref: "filesystem-server" }]
}
```

이름으로 참조되는 서버는 기본 프로세스와 동일한 MCP 클라이언트 연결을 공유합니다. 이 연결은 Agent가 종료될 때 **정리되지 않습니다**. 다른 Agent 또는 기본 프로세스가 여전히 이 연결을 사용 중일 수 있기 때문입니다.

### 4.2 인라인 정의(독점)

```typescript
// 새로운 MCP 서버 정의
{
  mcpServers: [{
    inline: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: "..." }
    }
  }]
}
```

인라인으로 정의된 서버는 Agent가 시작될 때 생성되고 종료 시 정리됩니다.

### 보안 조치

```typescript
// O_EXCL:는 TOCTOU 경쟁 조건을 방지합니다.
// O_NOFOLLOW: 심볼릭 링크 통과 공격 방지
const fd = fs.openSync(configPath, O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW, 0o600);
```

MCP 서버의 구성 파일은 원자성 쓰기(`O_EXCL`는 파일이 존재하지 않는 경우에만 생성되도록 보장) 및 심볼릭 링크 보호(`O_NOFOLLOW`는 심볼릭 링크 열기를 거부함)를 사용하여 공격자가 심볼릭 링크를 미리 배치하여 구성을 하이재킹하는 것을 방지합니다.

## 5. Fork 자식 프로세스 격리

`feature('FORK_AGENT')`가 활성화되면 하위 Agent는 별도의 하위 프로세스에서 실행됩니다.

```typescript
if (input.isolation === 'fork' || feature('FORK_AGENT')) {
  const child = fork(AGENT_WORKER_SCRIPT, {
    env: {
      ...process.env,
      AGENT_DEFINITION: JSON.stringify(def),
      AGENT_INPUT: JSON.stringify(input),
    },
    execArgv: ['--max-old-space-size=512'], // V8 힙 크기 제한
  });

  // IPC를 통한 통신
  child.on('message', handleAgentMessage);
}
```

### 프로세스 격리가 필요한 이유

- **메모리 격리**: 독립적인 V8 힙, Agent의 메모리 누수는 기본 프로세스에 영향을 미치지 않습니다.
- **GC 격리**: 하위 프로세스의 가비지 수집으로 인해 기본 프로세스가 정지되지 않습니다.
- **Crash Isolation**: 하위 프로세스의 충돌은 기본 프로세스 및 기타 Agent에 영향을 미치지 않습니다.
- **리소스 제한**: 메모리 및 CPU 제한을 독립적으로 설정할 수 있습니다.

## 6. 백엔드 탐지 순서

독립형 환경에서 Agent를 실행해야 하는 경우 시스템은 다음 순서로 사용 가능한 백엔드를 검색합니다.

```typescript
function detectAndGetBackend(): AgentBackend {
  // 1. In-Process 모드가 활성화되어 있는지 확인하세요.
  if (isInProcessEnabled()) {
    return new InProcessBackend();
  }

  // 2. iTerm2 감지
  if (isITerm2Available()) {
    return new ITerm2Backend(); // iTerm2 탭 사용
  }

  // 3. tmux 감지
  if (isTmuxAvailable()) {
    return new TmuxBackend(); // tmux를 사용하는 창
  }

  // 4. 다시 프로세스로 돌아가세요
  return new InProcessBackend();
}
```

### 여러 백엔드가 필요한 이유

- **iTerm2 탭**: macOS 사용자를 위한 최고의 경험, 각 Agent당 하나의 탭
- **tmux 창**: 크로스 플랫폼, SSH 원격 지원
- **진행 중**: 간편한 솔루션, 외부 도구 필요 없음

## 7. Coordinator 모드

### 7.1 패턴 감지

```typescript
function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') &&
    getAppState().permissionMode === 'coordinator';
}
```

### 7.2 Coordinator 특수 도구

Coordinator 모드에서 모델은 다음 내부 도구만 사용할 수 있습니다.

```typescript
const INTERNAL_WORKER_TOOLS = [
  'TeamCreate',       // Agent 팀 만들기
  'TeamDelete',       // 팀 Agent 삭제
  'SendMessage',      // Agent로 메시지 보내기
  'SyntheticOutput',  // 합성 출력
];
```

Coordinator **직접 실행할 수 없습니다** Bash, FileEdit 및 기타 조작 도구는 Worker를 생성하여 간접적으로만 작업을 완료할 수 있습니다.

### 7.3 시스템 프롬프트(라인 370)

Coordinator의 시스템 프롬프트는 네 가지 작업 단계를 정의합니다.

```
Phase 1: Research (연구)
  → research 유형의 Agent 생성
  → 프로젝트 정보, 코드 구조, 종속성 수집

Phase 2: Synthesis(종합)
  → 설문조사 결과 분석
  → 구현 계획 개발
  → 작업 할당

Phase 3: Implementation(구현)
  → code 유형의 Agent 생성
  → 특정 코딩 작업 할당
  → 진행상황 모니터링

Phase 4: Verification (검증)
  → review 유형의 Agent 생성
  → 코드 검토
  → 테스트 실행
```

### 4단계가 필수인 이유는 무엇인가요?

- **불완전한 정보 방지를 위한 조치**: Agent 가정에 기반한 코드 작성을 피하기 위해 조치를 취하기 전에 연구하십시오.
- **포괄적 우선순위**: Worker가 수집한 정보는 독립적으로 행동하기보다는 Coordinator의 포괄적인 판단이 필요합니다.
- **관심사 분리**: 연구, 코딩, 검토는 역할 혼란을 피하기 위해 서로 다른 Agent에서 수행됩니다.

## 8. 팀 파일 수명 주기

### 8.1 생성

```typescript
async function createTeam(name: string, leaderAgent: string): Promise<TeamFile> {
  // 고유한 이름 생성
  const teamName = generateUniqueTeamName(name);

  // leader 항목 만들기
  const leader: TeamMember = {
    name: leaderAgent,
    role: 'leader',
    status: 'active',
  };

  // 팀 파일에 쓰기
  const teamFile: TeamFile = {
    name: teamName,
    members: [leader],
    tasks: [],
    mailbox: [],
    createdAt: Date.now(),
  };

  await writeTeamFile(teamName, teamFile);

  // 세션 정리 등록
  registerSessionCleanup(() => deleteTeamFile(teamName));

  return teamFile;
}
```

### 8.2 작업 목록

```typescript
interface TeamTask {
  id: string;
  assignee: string;     // Agent 이름
  description: string;
  status: 'pending' | 'in_progress' | 'completed' | 'failed';
  result?: string;
}
```

## 9. Agent 간 메시지 라우팅

메시지 라우팅은 우선순위에 따라 여러 전송 방법을 시도합니다.

```
메시지 보내기 (from: A, to: B)
    │
    ├─ 1. 로컬 프로세스 내 → A와 B가 동일한 프로세스에 있음
    │     B의 메시지 처리 기능을 직접 호출합니다.
    │
    ├─ 2. UDS Socket → A 및 B는 동일한 기계에서 다른 프로세스에 있습니다.
    │     Unix Domain Socket를 통해 보내기
    │
    ├─ 3. Bridge 원격 → 원격 시스템의 B
    │     전달자: Bridge WebSocket
    │
    ├─ 4. Team 파일 메일함 → 비동기 통신
    │     팀 파일의 mailbox 필드에 쓰기
    │
    └─ 5. 브로드캐스트("*") → 모든 Agent에 전송됨
          모든 전송 방법을 탐색
```

### 9.1 로컬 프로세스 내 라우팅

```typescript
if (isInProcess(targetAgent)) {
  const handler = agentRegistry.get(targetAgent);
  await handler.onMessage(message);
  return;
}
```

### 9.2 UDS Socket 라우팅

```typescript
if (feature('UDS_INBOX')) {
  const socketPath = getAgentSocketPath(targetAgent);
  if (fs.existsSync(socketPath)) {
    const client = net.createConnection(socketPath);
    client.write(JSON.stringify(message));
    return;
  }
}
```

### 9.3 Team 파일 메일함

```typescript
// 최종 대체: 파일에 쓰기
async function sendViaMailbox(teamName: string, to: string, message: AgentMessage) {
  const teamFile = await readTeamFile(teamName);
  teamFile.mailbox.push({
    from: message.from,
    to,
    content: message.content,
    timestamp: Date.now(),
  });
  await writeTeamFile(teamName, teamFile);
}
```

### 다중 전송 방법을 사용하는 이유는 무엇입니까?

다양한 시나리오에는 다양한 통신 메커니즘이 필요합니다.

- **진행 중**: 가장 낮은 지연 시간, 동기 호출
- **UDS**: 크로스 프로세스이지만 동일한 머신, 짧은 대기 시간, 안정성
- **Bridge**: 교차 시스템, 네트워크에 따라 다름
- **파일 사서함**: 비동기식 공개로 두 당사자가 동시에 온라인 상태일 필요가 없습니다.

## 10. 구조화된 메시지

```typescript
type StructuredMessage =
  | { type: 'shutdown_request'; reason: string }
  | { type: 'shutdown_response'; acknowledged: boolean }
  | { type: 'plan_approval_response'; approved: boolean; feedback?: string }
  | { type: 'task_update'; taskId: string; status: string; result?: string }
  | { type: 'text'; content: string };
```

구조화된 메시지를 사용하면 시스템 수준 작업(예: 종료 요청)과 사용자 수준 작업(예: 문자 메시지)을 동일한 채널을 통해 전송할 수 있으며 메시지 유형에 따라 처리 방법이 구분됩니다.

## 11. Scratchpad 공유됨

```typescript
// tengu_scratch gate에 의해 제어되는 공유 노트
if (checkGate('tengu_scratch')) {
  const scratchPath = getScratchpadPath(teamName);

  // 모든 worker는 권한 프롬프트 없이 읽고 쓸 수 있습니다.
  const scratchContent = await readFile(scratchPath);
  await writeFile(scratchPath, updatedContent);
}
```

Scratchpad는 팀의 모든 Worker가 공유하는 임시 파일이며 다음 용도로 사용됩니다.

- 연구결과 공유
- 중간 상태 기록
- 업무 진행 조율

### 권한 프롬프트가 필요하지 않은 이유는 무엇입니까?

Scratchpad는 팀의 "화이트보드"입니다. 공유 공간에 대한 승인은 팀이 생성될 때 암시적으로 이루어집니다. 권한 부여 메시지가 자주 표시되면 다중 Agent 협업의 효율성이 심각하게 저하됩니다.

## 12. 자동 배경화

```typescript
const AUTO_BACKGROUND_TIMEOUT = 120_000; // 120초

async function* executeWithAutoBackground(
  agentGenerator: AsyncGenerator<AgentEvent, AgentResult>,
  taskManager: TaskManager
): AsyncGenerator<AgentEvent, AgentResult> {
  const timer = setTimeout(() => {
    // Agent를 백그라운드 작업으로 변환
    const task = taskManager.createTask({
      type: 'LocalAgent',
      name: agent.name,
      generator: agentGenerator,
    });
    // 알림UI
    yield { type: 'backgrounded', taskId: task.id };
  }, AUTO_BACKGROUND_TIMEOUT);

  try {
    return yield* agentGenerator;
  } finally {
    clearTimeout(timer);
  }
}
```

### 왜 120초인가요?

- 가장 간단한 작업을 120초 안에 완료(파일 검색, 사소한 수정)
- 120초 이상 걸리는 작업은 일반적으로 복잡한 작업(대규모 리팩토링, 다중 파일 수정)이므로 사용자는 기다리면 안 됩니다.
- 백그라운드 이후에도 사용자는 다른 작업에 REPL를 계속 사용할 수 있습니다.

## 13. 디자인 결정 요약

| 결정 | 이유 |
|------|------|
| Sub-Agent 유형 제한 도구 세트 | 최소 권한의 원칙 |
| MCP 공유 vs 독점 | 리소스 유출을 방지하려면 vs 반복 연결을 피하세요 |
| O_EXCL + O_NOFOLLOW | TOCTOU 및 심볼릭 링크 공격 방지 |
| Fork 프로세스 격리 | 메모리/GC/ 충돌 격리 |
| Coordinator 4단계 | 불완전한 정보 방지를 위한 조치 |
| 다중 전송 방법 | 다양한 배포 토폴로지 다루기 |
| 파일 메일함 보안 | 두 당사자가 동시에 온라인 상태일 필요가 없습니다 |
| 자동 배경 120s | 사용자가 긴 작업을 기다리지 않도록 방지 |

관련 장:

- [03 도구 시스템](03%20%EB%8F%84%EA%B5%AC%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): AgentTool 등록 및 실행
- [07 권한 시스템](07%20%EA%B6%8C%ED%95%9C%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md) : Coordinator/SwarmWorker의 권한 처리
- [09 서비스 계층](09%20%EC%84%9C%EB%B9%84%EC%8A%A4%20%EA%B3%84%EC%B8%B5.md): MCP 클라이언트 수명 주기
- [08 상태 관리](08%20%EC%83%81%ED%83%9C%20%EA%B4%80%EB%A6%AC.md): Agent 관련 AppState 필드
