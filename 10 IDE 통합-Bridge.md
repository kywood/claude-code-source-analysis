# 10장 IDE 통합 - Bridge

> **저자**: [Wang Yanshu](https://catyans.github.io)

> 이 장에서는 v1/v2 이중 전송 경로, JWT 새로 고침 예약, 작업 폴링 프로토콜 및 다중 세션 관리를 다루는 bridge/ 디렉터리(31개 파일)를 분석합니다.

## 1. 이중 전송 경로

Bridge 시스템은 IDE(VSCode, JetBrains)와의 통신을 위한 전송 프로토콜의 v1 및 v2 버전을 모두 지원합니다.

### 1.1 V1: HybridTransport (WS 읽기 + POST 쓰기)

```typescript
class V1HybridTransport implements BridgeTransport {
  private ws: WebSocket;               // 채널 읽기
  private httpClient: AxiosInstance;    // 채널 쓰기

  async connect(environmentId: string): Promise<void> {
    // WebSocket는 실시간 이벤트를 수신하는 데 사용됩니다.
    this.ws = new WebSocket(`${WS_URL}/v1/environments/${environmentId}/events`);
    this.ws.on('message', this.handleEvent.bind(this));

    // HTTP POST는 응답 및 상태 업데이트를 보내는 데 사용됩니다.
    this.httpClient = axios.create({
      baseURL: `${HTTP_URL}/v1/environments/${environmentId}`,
    });
  }

  // 읽기: WebSocket를 통해
  onEvent(handler: (event: BridgeEvent) => void): void {
    this.eventHandlers.push(handler);
  }

  // 쓰기: HTTP를 통해 POST
  async send(data: BridgeMessage): Promise<void> {
    await this.httpClient.post('/messages', data);
  }
}
```

V1는 Environments API 모델을 사용합니다. 각 Claude Code 인스턴스는 "환경"으로 등록되고 IDE는 ID 환경을 통해 해당 인스턴스와 통신합니다.

### 1.2 V2: SSE 읽기 + CCR 쓰기

```typescript
class V2Transport implements BridgeTransport {
  private eventSource: EventSource;     // 읽기 채널(SSE)
  private ccrClient: CCRClient;         // 채널 쓰기

  async connect(sessionToken: string): Promise<void> {
    // SSE는 서버 푸시 이벤트를 수신하는 데 사용됩니다.
    this.eventSource = new EventSource(`${SSE_URL}/v2/sessions/events`, {
      headers: { Authorization: `Bearer ${sessionToken}` },
    });

    // 쓰기용 CCR(Claude Code Remote) 클라이언트
    this.ccrClient = new CCRClient({
      baseUrl: CCR_URL,
      sessionToken,
    });
  }

  onEvent(handler: (event: BridgeEvent) => void): void {
    this.eventSource.onmessage = (msg) => {
      handler(JSON.parse(msg.data));
    };
  }

  async send(data: BridgeMessage): Promise<void> {
    await this.ccrClient.send(data);
  }
}
```

V2는 Environment 개념을 제거하고 무상태 통신을 위해 OAuth session token를 직접 사용합니다.

### V1와 V2를 모두 지원하는 이유

- **점진적 마이그레이션**: claude.ai 애플리케이션의 session-list 쿼리가 점진적으로 v2로 전환됩니다.
- **역방향 호환성**: 이전 IDE 확장은 여전히 v1를 사용합니다.
- **대체 전략**: 연결이 실패하면 v2는 v1로 대체할 수 있습니다.

## 2. JWT 새로고침 일정

### 2.1 새로고침 타이밍 계산

```typescript
function scheduleJwtRefresh(token: string): void {
  // JWT payload 디코드(서명을 확인하지 않고 expiry만 읽음)
  const payload = decodeJwtPayload(token);
  const expiryMs = payload.exp * 1000;
  const now = Date.now();

  // 만료 5분 전에 새로 고침
  const refreshAt = expiryMs - 5 * 60 * 1000;
  const delay = Math.max(0, refreshAt - now);

  setTimeout(() => refreshToken(), delay);
}
```

### 2.2 Generation Counter (생성 카운터)

```typescript
class TokenRefreshScheduler {
  private generation = 0;  // 현재 세대

  async refreshToken(): Promise<void> {
    const myGeneration = ++this.generation;

    try {
      const newToken = await fetchNewToken();

      // 최신 새로 고침으로 대체되었는지 확인
      if (myGeneration !== this.generation) {
        // 또 다른 새로고침이 시작되었습니다. 이 결과를 삭제하세요.
        return;
      }

      applyToken(newToken);
      scheduleJwtRefresh(newToken);
    } catch (error) {
      if (myGeneration !== this.generation) return;

      this.failureCount++;
      if (this.failureCount < 3) {
        // 60초 후에 다시 시도하세요.
        setTimeout(() => this.refreshToken(), 60_000);
      } else {
        // 30분 후 최종 재시도
        setTimeout(() => this.refreshToken(), 30 * 60_000);
      }
    }
  }
}
```

### Generation Counter가 필요한 이유

비동기 환경에서는 새 새로 고침 요청이 시작될 때까지 이전 새로 고침 요청이 반환되지 않을 수 있습니다. 예를 들면:

```
T=0: refresh A 시작됨 (generation=1)
T=1: token가 만료되고, refresh B가 시작됩니다(generation=2).
T=2: refresh B는 새로운 token를 반환합니다.
T=3: refresh A가 반환됩니다(이전 token) ← generation를 선택하지 않으면 최신 token를 덮어쓰게 됩니다.
```

`myGeneration`를 `this.generation`와 비교하면 오래된 비동기 결과가 삭제됩니다.

### 2.3 실패 재시도 전략

|실패 횟수 | 행동 |
|---------|------|
| 1 | 60초 후에 다시 시도 |
| 2 | 60초 후에 다시 시도 |
| 3+ | 30분 후 최종 재시도 |

점진적 백오프는 서비스를 사용할 수 없을 때 빈번한 요청을 방지합니다. 일시적인 서비스 중단을 복원하려면 최종 재시도 30분이면 충분합니다.

## 3. 작업 폴링 프로토콜

### 3.1 폴링 요청

```typescript
async function pollForWork(environmentId: string): Promise<WorkItem | null> {
  const response = await httpClient.get(
    `/v1/environments/${environmentId}/work/poll`,
    { timeout: LONG_POLL_TIMEOUT }
  );

  if (response.status === 204) return null; // 직업이 없다
  return response.data as WorkItem;
}
```

### 3.2 용량에 따른 폴링 간격

```typescript
function getPollingInterval(capacity: CapacityStatus): number {
  switch (capacity) {
    case 'not-at-capacity':
      return 1000;   // 1초: 유휴 상태, 적극적으로 폴링
    case 'partial':
      return 5000;   // 5초: 부분적으로 통화 중
    case 'at-capacity':
      return 30000;  // 30초: 전체 로드, 폴링 감소
  }
}
```

### 용량이 폴링 빈도에 영향을 미치는 이유는 무엇입니까?

- **유휴 시 빠른 폴링**: 사용자는 응답을 기다리는 동안 최대한 빨리 작업을 수행하려고 합니다.
- **완전히 로드될 때 느린 폴링**: 모든 세션이 작동 중일 때 빈번한 폴링은 서버 리소스를 낭비합니다.
- **부분 사용 중 균형**: 응답성과 서버 로드 사이의 균형

### 3.3 빈 폴 카운트

```typescript
class WorkPoller {
  private emptyPollCount = 0;

  async poll(): Promise<void> {
    const work = await pollForWork(this.environmentId);

    if (!work) {
      this.emptyPollCount++;

      // 지속적인 빈 폴링 후 점차적으로 빈도를 줄입니다.
      if (this.emptyPollCount > 10) {
        this.interval = Math.min(this.interval * 1.5, MAX_POLL_INTERVAL);
      }
    } else {
      this.emptyPollCount = 0;
      this.interval = getPollingInterval(this.capacity);
      await this.processWork(work);
    }
  }
}
```

### 3.4 재활용 가능한 작품 (Reclaimable Work)

```typescript
async function checkReclaimableWork(): Promise<WorkItem[]> {
  // Worker 충돌로 인해 완료되지 않은 작업 찾기
  const reclaimable = await httpClient.get(
    `/v1/environments/${this.environmentId}/work/reclaimable`
  );
  return reclaimable.data;
}
```

Worker 프로세스가 충돌하더라도 처리 중이던 작업은 손실되지 않습니다. 다음 폴링된 Worker는 이 완료되지 않은 작업을 "재활용"할 수 있습니다.

## 4. 확인 및 하트비트

### 4.1 작업 확인(Acknowledgement)

```typescript
async function acknowledgeWork(workId: string, sessionToken: string): Promise<void> {
  await httpClient.post(
    `/v1/environments/${environmentId}/work/${workId}/ack`,
    { sessionToken }
  );
}
```

확인을 통해 서버는 "이 작업을 수신했으며 처리를 시작할 수 있습니다"라고 알립니다. 확인 전에 Worker가 충돌하면 해당 작업은 다른 Worker에 의해 재활용됩니다.

### 4.2 하트비트(Heartbeat)

```typescript
class WorkHeartbeat {
  private intervalId: NodeJS.Timer;

  start(workId: string): void {
    this.intervalId = setInterval(async () => {
      try {
        await httpClient.post(
          `/v1/environments/${environmentId}/work/${workId}/heartbeat`
        );
      } catch {
        // 하트비트 장애는 치명적이지 않으며 서버에는 유예 기간이 있습니다.
      }
    }, HEARTBEAT_INTERVAL);
  }

  stop(): void {
    clearInterval(this.intervalId);
  }
}
```

### 하트비트와 폴링이 분리되는 이유

- **폴링**은 새로운 일자리를 얻기 위한 것이며 복잡한 응답 구문 분석이 필요합니다.
- **하트비트**는 현재 직장의 '임대'를 갱신하고 생존을 확인하는 것입니다.
- 분리 후 하트비트가 더 자주 발생하고(임대 유지) 폴링이 더 스마트해질 수 있습니다(용량 기반).

## 5. 다중 세션 관리

### 5.1 maxSessions 용량 조절

```typescript
class SessionManager {
  private sessions = new Map<string, Session>();
  private readonly maxSessions: number;

  canAcceptWork(): boolean {
    return this.sessions.size < this.maxSessions;
  }

  getCapacityStatus(): CapacityStatus {
    if (this.sessions.size === 0) return 'not-at-capacity';
    if (this.sessions.size < this.maxSessions) return 'partial';
    return 'at-capacity';
  }
}
```

### 5.2 Worktree 격리

각 Bridge 세션은 독립적인 Git Worktree를 사용하여 동시 세션에 의한 파일 수정이 충돌하지 않도록 합니다.

```typescript
async function createSessionWorktree(sessionId: string): Promise<string> {
  const branchName = `claude-session-${sessionId}`;
  const worktreePath = path.join(GIT_WORKTREE_DIR, sessionId);

  await exec(`git worktree add -b ${branchName} ${worktreePath}`);

  return worktreePath;
}

async function cleanupSessionWorktree(sessionId: string): Promise<void> {
  const worktreePath = path.join(GIT_WORKTREE_DIR, sessionId);
  await exec(`git worktree remove ${worktreePath}`);
}
```

### 5.3 Work-Stealing(작업 훔치기)

```typescript
// Worker는 충돌한 다른 Worker의 작업을 "훔칠" 수 있습니다.
async function stealWork(reclaimableWorkId: string): Promise<WorkItem> {
  const response = await httpClient.post(
    `/v1/environments/${environmentId}/work/${reclaimableWorkId}/steal`
  );
  return response.data;
}
```

## 6. 세션 수명주기

```
1. 등록환경
   POST /v1/environments
   → environmentId

2. 여론조사 업무
   GET /v1/environments/{id}/work/poll
   → workItem

3. 작품 확인
   POST /v1/environments/{id}/work/{workId}/ack
   → sessionToken

4. 하트비트 주기
   POST /v1/environments/{id}/work/{workId}/heartbeat
   (N초마다)

5. 가공업무
   세션 생성 → 쿼리 실행 → 결과 반환

6. 완료/취소
   POST /v1/environments/{id}/work/{workId}/complete
   또는
   POST /v1/environments/{id}/work/{workId}/cancel

7. 환경에서 로그아웃
   DELETE /v1/environments/{id}
```

## 7. 권한 이전

### 7.1 Bridge 권한 콜백

```typescript
interface BridgePermissionCallbacks {
  // Claude Code에서 IDE로 권한 요청
  requestPermission(request: PermissionRequest): void;

  // IDE에서 Claude Code로 전송된 권한 응답
  onPermissionResponse(callback: (response: PermissionResponse) => void): void;
}
```

### 7.2 control_response 이벤트

```typescript
// IDE가 보낸 제어 응답
interface ControlResponseEvent {
  type: 'control_response';
  requestId: string;
  decision: 'allow' | 'deny';
  rememberForSession?: boolean;
}
```

Claude Code는 권한이 필요할 때 Bridge를 통해 IDE에 요청을 보내고, IDE는 사용자가 결정할 수 있도록 대화 상자를 표시한 후 `control_response` 이벤트를 통해 결과를 반환합니다.

## 8. Session Runner

### 8.1 createSessionSpawner

```typescript
function createSessionSpawner(config: SpawnerConfig) {
  return {
    async spawn(workItem: WorkItem): Promise<ChildProcess> {
      const worktreePath = await createSessionWorktree(workItem.sessionId);

      const child = fork(SESSION_WORKER_SCRIPT, {
        cwd: worktreePath,
        env: {
          ...process.env,
          CCR_V2: config.useV2 ? '1' : '0',
          WORKER_EPOCH: String(config.epoch),
          SESSION_ID: workItem.sessionId,
          BRIDGE_SESSION_TOKEN: workItem.sessionToken,
        },
      });

      // 청소 등록
      child.on('exit', async () => {
        await cleanupSessionWorktree(workItem.sessionId);
      });

      return child;
    }
  };
}
```

### 8.2 환경변수 전송

| 변수 | 설명 |
|------|------|
| `CCR_V2` | v2 프로토콜 사용 여부 |
| `WORKER_EPOCH` | 이전 프로세스를 식별하는 데 사용되는 Worker 세대 |
| `SESSION_ID` | 세션 ID |
| `BRIDGE_SESSION_TOKEN` | 세션 인증 token |

### 8.3 Session ID 매핑

```typescript
// session ID에서 Bridge work ID의 내부 매핑
const sessionWorkMap = new Map<string, string>();

function mapSession(sessionId: string, workId: string): void {
  sessionWorkMap.set(sessionId, workId);
}

function getWorkId(sessionId: string): string | undefined {
  return sessionWorkMap.get(sessionId);
}
```

## 9. 디자인 결정 요약

| 결정 | 이유 |
|------|------|
| V1/V2 공존 | 점진적 마이그레이션, 이전 버전과의 호환성 |
| JWT generation counter | 이전 비동기 새로 고침이 새 token를 덮어쓰지 않도록 방지 |
| 용량 의존적 폴링 | 유휴시 빠른 응답, 풀로드시 리소스 절약 |
| 하트비트와 폴링의 분리 | 다양한 빈도, 다양한 책임 |
| Worktree 격리 | 동시 세션에서 파일 충돌 방지 |
| Work-stealing | 작업 손실 없이 응급 복구 |
| 하위 프로세스 격리 | 세션 충돌이 기본 Bridge 프로세스에 영향을 주지 않습니다. |

관련 장:

- [01 시작 과정](01%20%EC%8B%9C%EC%9E%91%20%EA%B3%BC%EC%A0%95.md): Bridge 모드의 부팅 경로
- [07 권한 시스템](07%20%EA%B6%8C%ED%95%9C%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): 권한 파이프라인에서 Bridge 권한 콜백의 역할
- [09 서비스 계층](09%20%EC%84%9C%EB%B9%84%EC%8A%A4%20%EA%B3%84%EC%B8%B5.md): OAuth 인증은 Bridge JWT를 지원합니다.
- [05 도구 상세-AgentTool과 다중 에이전트](05%20%EB%8F%84%EA%B5%AC%20%EC%83%81%EC%84%B8-AgentTool%EA%B3%BC%20%EB%8B%A4%EC%A4%91%20%EC%97%90%EC%9D%B4%EC%A0%84%ED%8A%B8.md): Worktree를 사용하여 여러 Agent를 격리합니다.
