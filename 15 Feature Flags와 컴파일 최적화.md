# 15장 Feature Flags 및 컴파일 최적화

> **저자**: [Wang Yanshu](https://catyans.github.io)

> 이 장에서는 빌드타임 DCE(Dead Code Elimination), 런타임 Feature Gate 및 Claude Code의 구성 마이그레이션 시스템을 분석합니다.

## 1. DCE (Dead Code Elimination) 빌드 시

### 1.1 feature() 매크로

```typescript
// bun:bundle에서 제공하는 컴파일 타임 기능
import { feature } from 'bun:bundle';

// 컴파일 시간 평가
if (feature('COORDINATOR_MODE')) {
  const { CoordinatorTools } = require('./coordinator/tools');
  registerTools(CoordinatorTools);
}
```

`feature()`는 Bun의 bundler 단계 중에 `true` 또는 `false` 리터럴로 대체됩니다. `false`로 교체하면 후속 tree-shaking 단계에서 전체 `if` 블록이 제거되고 `require()`를 포함한 모듈은 포장되지 않습니다.

### 1.2 삼항 require 모드

```typescript
// 컴파일 시간 조건 require
const CoordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinator')
  : null;

// 런타임 시 사용하기에 안전함
function maybeCreateCoordinator() {
  if (CoordinatorModule) {
    return new CoordinatorModule.Coordinator();
  }
  return null;
}
```

이 패턴은 `if` 블록보다 모듈 수준 변수에 더 적합합니다. 삼항 표현식은 `const`에 할당될 수 있는 반면, `if` 블록의 `require`에는 `let` 또는 추가 함수 래퍼가 필요합니다.

### 1.3 DCE 이후의 코드

```typescript
// 소스 코드
if (feature('KAIROS')) {
  const { KairosAssistant } = require('./kairos');
  registerAssistant(KairosAssistant);
}

// 외부 빌드 후 (KAIROS=false)
if (false) {
  // 전체 블록이 제거됩니다.
}

// 최종 제품: 코드가 전혀 없음
```

## 2. 알려진 빌드 타임 플래그

| 플래그 | 설명 | 외부 빌드 값 |
|------|------|-----------|
| `PROACTIVE` | 적극적 제안제도 | false |
| `KAIROS` | Anthropic 내부 보조 모드 | false |
| `BRIDGE_MODE` | Bridge IDE 통합 | true |
| `DAEMON` | 데몬 모드 | false |
| `VOICE_MODE` | 음성 입력 | false |
| `AGENT_TRIGGERS` | Agent 플립플롭 | false |
| `MONITOR_TOOL` | 시스템 모니터링 도구 | false |
| `COORDINATOR_MODE` | 여러 Agent 코디네이터 | false |
| `WEB_BROWSER_TOOL` | 브라우저 자동화 도구 | false |
| `TEAMMEM` | 팀 추억 공유 | false |
| `DIRECT_CONNECT` | DirectConnect 서버 | false |
| `SSH_REMOTE` | SSH 원격 모드 | false |
| `CHICAGO_MCP` | 시카고 MCP 확장 | false |
| `TEMPLATES` | 템플릿 시스템 | false |
| `UPLOAD_USER_SETTINGS` | 사용자 설정 원격 측정 업로드 | true |
| `FORK_AGENT` | Fork 프로세스 격리 Agent | false |
| `UDS_INBOX` | Unix Domain Socket 메시지 상자 | false |

### 외부 빌드에서 대부분의 플래그가 false인 이유는 무엇입니까?

이러한 기능은 다음 중 하나입니다.

- **Anthropic 내부 함수**(KAIROS, PROACTIVE): 공개 빌드에 표시되어서는 안 됩니다.
- **실험적 기능** (FORK_AGENT, UDS_INBOX): 아직 안정적이지 않음
- **플랫폼별 기능**(VOICE_MODE): 추가 권한 및 서비스 지원이 필요합니다.
- **엔터프라이즈 기능**(COORDINATOR_MODE, TEAMMEM): 특정 배포에서만 활성화됩니다.

## 3. 런타임 Feature Gate (GrowthBook)

### 3.1 Gate 목록

| Gate 이름 | 설명 |
|----------|------|
| `tengu_ccr_bridge` | CCR Bridge v2 활성화 |
| `tengu_bridge_repl_v2` | Bridge REPL v2 |
| `tengu_ccr_bridge_multi_session` | Bridge 다중 세션 |
| `tengu_scratch` | Scratchpad 공유 |
| `tengu_coral_fern` | 알 수 없는 기능(내부 코드명) |
| `tengu_amber_quartz_disabled` | 음성 입력 비활성화 스위치 |
| `tengu_otk_slot_v1` | OTK Slot v1 |
| `tengu_satin_quoll` | 알 수 없는 기능(내부 코드명) |
| `tengu_disable_streaming_to_non_streaming_fallback` | 비스트리밍 폴백으로 스트리밍 비활성화 |

### 3.2 두 가지 검사 모드

#### getFeatureValue_CACHED_MAY_BE_STALE(비차단)

```typescript
// 로컬 캐시에서 읽습니다. 이전 값일 수 있음
const value = getFeatureValue_CACHED_MAY_BE_STALE('tengu_scratch', false);
```

**사용 시나리오**: UI 렌더링, 중요하지 않은 경로. 렌더링을 차단하지 않으려면 캐시된 값을 사용하세요. 값이 만료되면 다음에 새로 고칠 때 업데이트됩니다.

#### checkGate_CACHED_OR_BLOCKING(차단 가능)

```typescript
// 캐시 사용이 우선시됩니다. 캐시가 만료되면 차단되고 새로 고침을 기다립니다.
const isEnabled = await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge');
```

**사용 시나리오**: 보안에 중요한 경로입니다. gate가 보안 기능(예: 권한 우회)을 제어하는 ​​경우 최신 값을 사용해야 합니다.

### 모드가 두 가지인 이유는 무엇인가요?

- **CACHED_MAY_BE_STALE**: UI 기능(예: 패널 표시 여부)의 경우 몇 초 전의 값을 사용하는 것은 완벽하게 허용되지만 UI 렌더링을 차단하는 것은 허용되지 않습니다.
- **CACHED_OR_BLOCKING**: 보안 기능(특정 작업 허용 여부 등)의 경우 만료된 값을 사용하면 보안 허점이 발생할 수 있으므로 몇 백 밀리초 정도 기다리는 것이 좋습니다.

## 4. 구성 마이그레이션

### 4.1 마이그레이션 파일

```typescript
// migrations/001_fennec_to_opus.ts
export const migration = {
  id: '001_fennec_to_opus',
  description: '모델명 Fennec를 Opus로 업데이트',

  shouldRun(settings: Settings): boolean {
    return settings.model?.includes('fennec') ?? false;
  },

  run(settings: Settings): Settings {
    return {
      ...settings,
      model: settings.model!.replace('fennec', 'opus'),
    };
  }
};
```

### 4.2 12개 마이그레이션 추적 모델의 진화

| ID 마이그레이션 | 설명 |
|---------|------|
| 001 | Fennec → Opus |
| 002 | Sonnet 4.5 → Sonnet 4.6 |
| 003 | Opus → Opus 1M |
| 004 | 이전 권한 형식 → 새 형식 |
| 005 | MCP 구성 형식 업데이트 |
| 006 | 테마 구성 마이그레이션 |
| 007 | 단축키 구성 마이그레이션 |
| 008 | 모델 별칭 업데이트 |
| 009 | API 엔드포인트 마이그레이션 |
| 010 | 플러그인 구성 형식 업데이트 |
| 011 | 메모리 디렉터리 마이그레이션 |
| 012 | 세션 형식 버전 업데이트 |

### 4.3 마이그레이션 실행

```typescript
async function runMigrations(): Promise<void> {
  const settings = await loadSettings();
  const appliedMigrations = await getAppliedMigrations();

  for (const migration of ALL_MIGRATIONS) {
    // 건너뛰기가 적용됨
    if (appliedMigrations.has(migration.id)) continue;

    // 실행해야 하는지 확인하세요.
    if (!migration.shouldRun(settings)) {
      await markMigrationApplied(migration.id);
      continue;
    }

    // 마이그레이션 실행
    const updatedSettings = migration.run(settings);
    await saveSettings(updatedSettings);
    await markMigrationApplied(migration.id);

    logger.info(`Applied migration: ${migration.id}`);
  }
}
```

### 시스템을 마이그레이션해야 하는 이유는 무엇인가요?

Claude Code는 모델명을 자주 업데이트합니다(Anthropic의 모델 출시 주기에 따라). 마이그레이션 시스템이 없는 경우:

- 사용자의 settings.json에는 오래된 모델 이름이 남아 있습니다.
- 수동 업데이트는 오류가 발생하기 쉽고 잊어버리기 쉽습니다.
- 엔터프라이즈 배포에는 균일하게 업데이트해야 하는 구성이 많을 수 있습니다.

## 5. 변형 배포

### 5.1 네 가지 변형

| 변형 | 대상 사용자 | 빌드 기능 |
|------|---------|---------|
| External | 일반 사용자 | 대부분의 feature=false, 최소 크기 |
| Ant-internal | Anthropic 직원 | 모두 feature=true |
| Assistant | KAIROS 보조 | KAIROS=true, 기타 옵션 |
| Bridge | IDE 통합 | BRIDGE_MODE=true |

### 5.2 빌드 프로세스

```typescript
// build.ts
function buildVariant(variant: BuildVariant) {
  const features = getFeatureFlags(variant);

  return Bun.build({
    entrypoints: ['./src/entrypoints/main.tsx'],
    define: {
      // feature flags를 컴파일 타임 상수로 삽입합니다.
      ...Object.fromEntries(
        Object.entries(features).map(([k, v]) => [
          `feature('${k}')`,
          String(v),
        ])
      ),
    },
    minify: variant === 'external',
    sourcemap: variant === 'external' ? 'external' : 'inline',
  });
}
```

### 5.3 여러 웨어하우스 대신 단일 코드 기반을 사용하는 이유는 무엇입니까?

| 솔루션 | 장점 | 단점 |
|------|------|------|
| 여러 창고 | 완전한 격리 | 코드 중복 및 동기화 어려움 |
| 단일 창고 + 지점 | 독립적인 진화 | 병합 충돌, 느린 기능 전파 |
| 단일 창고 + DCE | 공유 코어, 정밀 제어 | 건설의 복잡성 |

Claude Code는 다음과 같은 이유로 단일 창고 + DCE를 선택했습니다.

- 핵심 로직(QueryEngine, 도구 시스템, 권한 시스템)은 모든 변형에서 공유됩니다.
- 쉽게 검토할 수 있도록 Feature flags 변형 차이점이 명확하게 표시됨
- DCE는 외부 빌드에 내부 코드가 포함되지 않음을 보장합니다(보안).
- 모든 변형에 대한 bug 자동 전파 문제를 수정했습니다.

## 6. Feature Flag의 수명주기

```
1. 개발자가 새로운 기능을 추가합니다.
   ├─ feature('NEW_FEATURE') 가드 추가
   └─ 기본 false(기존 빌드에는 영향을 주지 않음)

2. 내부 테스트
   └─ Ant-internal는 공사중입니다. NEW_FEATURE=true

3. A/B 테스트
   └─ 일부 사용자에게 활성화된 GrowthBook gate 추가

4. 정식 출시
   └─ External는 공사중입니다. NEW_FEATURE=true

5. 청소
   └─ feature() 가드를 제거하고 코드를 기본 경로로 만듭니다.
```

### 정리 단계가 필요한 이유는 무엇인가요?

feature flags를 장기간 보존하면 다음과 같은 결과가 발생합니다.

- 코드 이해가 어려움(어떤 빌드에서 어떤 flag가 활성화되었는지 추적해야 함)
- 테스트 매트릭스 폭발(flag의 2^N 조합)
- 데드 코드 축적(절대 활성화되지 않는 이전 flag)

## 7. 디자인 결정 요약

| 결정 | 이유 |
|------|------|
| 구축 시 DCE | 외부 빌드에 내부 코드가 포함되어 있지 않습니다. |
| 런타임 GrowthBook | A/B 테스트 및 kill-switch |
| 두 가지 gate 검사 모드 | 일관성과 성능의 균형 |
| 12개 구성 마이그레이션 | 모델 이름 변경 추적 |
| 단일 코드 베이스 + DCE | 공유 코어, 변형 차이의 정밀한 제어 |
| Feature Flag 수명주기 | 실험부터 전체 볼륨, 정리까지 전체 프로세스 |

관련 장:

- [01 시작 과정](01%20%EC%8B%9C%EC%9E%91%20%EA%B3%BC%EC%A0%95.md): feature flags는 시작 경로에 영향을 줍니다.
- [03 도구 시스템](03%20%EB%8F%84%EA%B5%AC%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): 조건부 도구 등록
- [09 서비스 계층](09%20%EC%84%9C%EB%B9%84%EC%8A%A4%20%EA%B3%84%EC%B8%B5.md) : GrowthBook 서비스 초기화
- [05 도구 상세-AgentTool과 다중 에이전트](05%20%EB%8F%84%EA%B5%AC%20%EC%83%81%EC%84%B8-AgentTool%EA%B3%BC%20%EB%8B%A4%EC%A4%91%20%EC%97%90%EC%9D%B4%EC%A0%84%ED%8A%B8.md) : Coordinator gate의 feature
