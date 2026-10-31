# 12장 Skill 및 플러그인 시스템

> **저자**: [Wang Yanshu](https://catyans.github.io)

> 이 장에서는 skills/(20개 파일) 및 plugins/ 디렉터리를 분석하여 번들 Skill 등록, 지연 파일 추출, 사용자 Skill 로딩 및 플러그인 수명주기를 다룹니다.

## 1. Skill 번들 등록

### 1.1 BundledSkillDefinition 유형

```typescript
interface BundledSkillDefinition {
  // 기본정보
  name: string;                    // Skill 이름(예: "commit")
  description: string;             // 기능 설명
  aliases: string[];               // 별칭 목록(예: ["cm", "git-commit"])

  // 트리거 조건
  whenToUse: string;               // 언제 사용해야 하는지에 대한 자연어 설명

  // 능력 제약
  allowedTools: string[];          // 허용된 도구 목록
  model?: string;                  // 모델 지정(기본값 재정의)

  // 수명주기
  hooks?: SkillHooks;              // 전면/후면 후크
  context?: SkillContext;          // 추가적인 맥락

  // Agent 구성
  agent?: AgentConfig;             // Skill에 하위 Agent가 필요한 경우

  // 파일 종속성
  files?: Record<string, string>;  // 포함된 파일 콘텐츠

  // 프롬프트 생성
  getPromptForCommand: (args: string) => string; // 실행 프롬프트 생성
}
```

### 1.2 registerBundledSkill

```typescript
const bundledSkills: BundledSkillDefinition[] = [];

function registerBundledSkill(skill: BundledSkillDefinition): void {
  // 고유성 확인
  if (bundledSkills.some(s => s.name === skill.name)) {
    throw new Error(`Duplicate skill name: ${skill.name}`);
  }

  bundledSkills.push(skill);
}
```

## 2. 지연 파일 추출

### 2.1 클로저 로컬 메모이제이션

```typescript
function createLazyExtractor(files: Record<string, string>) {
  let extracted = false;  // 클로저 지역 변수
  let extractedPath: string | null = null;

  return async function extract(): Promise<string> {
    if (extracted && extractedPath) {
      return extractedPath;
    }

    extractedPath = await extractBundledSkillFiles(files);
    extracted = true;
    return extractedPath;
  };
}
```

### 2.2 안전한 파일 추출

```typescript
async function extractBundledSkillFiles(
  files: Record<string, string>
): Promise<string> {
  // per-process 고유 경로 생성
  const nonce = crypto.randomBytes(8).toString('hex');
  const extractDir = path.join(SKILL_EXTRACT_DIR, `skill-${process.pid}-${nonce}`);

  // 디렉터리 생성
  await fs.mkdir(extractDir, { mode: 0o700, recursive: true });

  for (const [filename, content] of Object.entries(files)) {
    const filePath = path.join(extractDir, filename);

    // 원자적 쓰기: O_EXCL는 파일이 존재하지 않도록 보장합니다.
    // O_NOFOLLOW는 기호 링크 탐색을 방지합니다.
    const fd = await fs.open(filePath, O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW);
    await fd.writeFile(content);
    await fd.chmod(0o700);
    await fd.close();
  }

  return extractDir;
}
```

### 2.3 보안조치에 대한 자세한 설명

#### O_EXCL(독점 생성)

```
공격 시나리오(O_EXCL 제외):
1. Claude Code 파일 존재 여부 확인 → 존재하지 않음
2. 공격자는 동일한 이름 → /tmp/skill-123/file → /etc/passwd로 심볼릭 링크를 생성한다.
3. Claude Code가 파일에 기록됩니다. → /etc/passwd가 실제로 기록됩니다.

방어(O_EXCL 사용):
1. Claude Code O_EXCL를 사용하여 파일을 엽니다.
2. 파일이 이미 존재하는 경우(심볼릭 링크 포함) open()가 실패합니다.
3. 공격자는 파일을 미리 배치할 수 없습니다.
```

#### O_NOFOLLOW(심볼릭 링크 거부됨)

파일이 새로 생성되더라도 해당 경로의 디렉터리가 심볼릭 링크이면 `O_NOFOLLOW`는 열기를 거부합니다. 이중 보호.

#### Per-Process Nonce

```typescript
const nonce = crypto.randomBytes(8).toString('hex');
```

공격자가 프로세스 PID를 알고 있더라도 nonce 값을 예측할 수 없기 때문에 일치하는 디렉터리나 파일을 미리 생성할 수 없습니다.

### 지연 추출이 필요한 이유

대부분의 Skill는 세션 내에서 호출되지 않습니다. 모든 Skill 파일을 사전 추출합니다.

- 부팅 시간 증가(디스크 I/O)
- 임시 보관공간의 낭비
- 청소부담 증가

지연 추출을 사용하면 실제로 사용된 Skill만 압축이 풀립니다.

## 3. 사용자 Skill가 로드되었습니다.

### 3.1 디렉토리 검색

```typescript
async function loadUserSkills(): Promise<UserSkill[]> {
  const skillsDir = path.join(os.homedir(), '.claude', 'skills');

  if (!await fs.pathExists(skillsDir)) return [];

  const entries = await fs.readdir(skillsDir);
  const skills: UserSkill[] = [];

  for (const entry of entries) {
    if (!entry.endsWith('.md')) continue;

    const filePath = path.join(skillsDir, entry);
    const content = await fs.readFile(filePath, 'utf-8');
    const meta = parseFrontmatter(content);

    skills.push({
      name: meta.name || path.basename(entry, '.md'),
      description: meta.description || '',
      allowedTools: meta.allowedTools || [],
      model: meta.model,
      whenToUse: meta.when_to_use || '',
      promptContent: extractBody(content),
      source: 'user',
    });
  }

  return skills;
}
```

### 3.2 Frontmatter 형식

```markdown
---
name: "맞춤형 배포"
description: "내부 회사 도구를 사용하여 배포"
allowedTools:
  - Bash
  - FileRead
model: claude-sonnet-4-6
when_to_use: "사용자가 배포 또는 게시를 요청하는 경우"
---

# 배포 절차

1. 실행 `./scripts/pre-deploy.sh` 사전 조건 확인
2. 실행 `deploy-tool push --env production`
3. 배포 상태 확인 `deploy-tool status`
4. 실패하면 다음 명령 실행: `deploy-tool rollback`
```

### 3.3 포장은 Command입니다.

```typescript
function wrapSkillAsCommand(skill: UserSkill): Command {
  return {
    type: 'prompt',
    name: skill.name,
    description: skill.description,
    allowedTools: skill.allowedTools,
    progressMessage: `실행 ${skill.name}...`,
    isEnabled: () => true,
    availability: 'all',
    getPromptForCommand: (args: string) => {
      return `${skill.promptContent}\n\n사용자 입력: ${args}`;
    },
  };
}
```

## 4. 내장 Skill 디렉토리

| Skill | 기능 | 일반적인 트리거 |
|-------|------|---------|
| `remember` | 메시지를 메모리에 저장 | /remember |
| `updateConfig` | settings.json 수정 | /config |
| `keybindings` | 키 바인딩 구성 | /keybindings |
| `loop` | 타이밍 루프 실행 | /loop 5m /foo |
| `debug` | 디버깅 지원 | /debug |
| `stuck` | 막혔을 때의 조언 | /stuck |
| `scheduleRemoteAgents` | 원격 Agent 예약 | /schedule |
| `batch` | 일괄 작업 | /batch |
| `skillify` | 프로세스를 Skill로 변환 | /skillify |
| `simplify` | 코드 단순화 검토 | /simplify |
| `claudeApi` | Claude API 개발 지원 | /claude-api |
| `verify` | 코드 정확성 확인 | /verify |
| `claudeInChrome` | Chrome 브라우저 통합 | /chrome |
| `loremIpsum` | 자리 표시자 텍스트 생성 | /lorem |

## 5. 플러그인 시스템

### 5.1 Plugin 정의

```typescript
interface PluginDefinition {
  name: string;
  description: string;
  version: string;

  // 기본적으로 활성화됨
  defaultEnabled: boolean;

  // 가용성 확인
  isAvailable(): boolean;

  // 구성 요소
  skills: BundledSkillDefinition[];
  hooks: HookDefinition[];
  mcpServers: MCPServerConfig[];
}
```

### 5.2 로직 활성화/비활성화

```typescript
function isPluginEnabled(plugin: PluginDefinition): boolean {
  // 1. 사용자 설정 재정의
  const userSetting = getUserPluginSetting(plugin.name);
  if (userSetting !== undefined) return userSetting;

  // 2. 플러그인 기본값
  return plugin.defaultEnabled;
}

function isPluginAvailable(plugin: PluginDefinition): boolean {
  // isAvailable()가 false를 반환하면 플러그인이 완전히 표시되지 않습니다.
  return plugin.isAvailable();
}
```

### 기본적으로 활성화되어 있는 이유는 무엇인가요?

```typescript
defaultEnabled: true  // 대부분의 플러그인은 기본적으로 활성화되어 있습니다.
```

디자인 컨셉은 **제로 구성 환경**입니다. 사용자가 Claude Code를 설치한 후 모든 유용한 기능을 즉시 사용할 수 있습니다. 비활성화해야 하는 사용자가 주도적으로 끄고, 활성화해야 하는 사용자가 주도적으로 활성화합니다. 이는 기능 발견의 장벽을 낮춰줍니다.

### isAvailable vs isEnabled

- `isAvailable()`: 환경 제약(예: 특정 기능에는 특정 OS 또는 특정 도구 설치가 필요함). false를 반환할 때 설정에 플러그인이 **전혀 나타나지 않습니다**.
- `isEnabled`: 사용자 선택. 플러그인이 설정에 표시되지만 "비활성화"로 표시된 경우 false를 반환합니다.

두 가지를 구별하면 사용자 혼란을 피할 수 있습니다. 즉, 작동할 가능성이 없는 플러그인을 보고 이를 활성화하려고 시도하는 것입니다.

### 5.3 플러그인 Skill 추출

```typescript
function getBuiltinPluginSkillCommands(): Command[] {
  const commands: Command[] = [];

  for (const plugin of getAllPlugins()) {
    if (!isPluginAvailable(plugin)) continue;
    if (!isPluginEnabled(plugin)) continue;

    for (const skill of plugin.skills) {
      commands.push({
        ...wrapSkillAsCommand(skill),
        source: 'bundled',
        pluginName: plugin.name,
      });
    }
  }

  return commands;
}
```

### 5.4 플러그인 핫 로딩

```typescript
class PluginWatcher {
  private cache = new Map<string, { plugin: PluginDefinition; mtime: number }>();

  async checkForChanges(): Promise<boolean> {
    let changed = false;

    for (const [name, cached] of this.cache) {
      const currentMtime = await getPluginMtime(name);
      if (currentMtime > cached.mtime) {
        this.cache.delete(name);
        changed = true;
      }
    }

    return changed;
  }

  invalidateCache(pluginName: string): void {
    this.cache.delete(pluginName);
  }
}
```

### 5.5 needsRefresh 플래그

```typescript
// 변경 사항이 감지되면 플래그 설정
if (await pluginWatcher.checkForChanges()) {
  store.setState(s => ({ ...s, needsPluginRefresh: true }));
}

// 다음 도구 풀 새로 고침 시 적용
if (state.needsPluginRefresh) {
  refreshToolPool();
  store.setState(s => ({ ...s, needsPluginRefresh: false }));
}
```

## 6. Skill는 SkillTool를 통해 호출됩니다.

```typescript
const SkillTool = buildTool({
  name: 'SkillInvoke',
  description: '통화 등록된 Skill',

  async* call(input: { skillName: string; args: string }, context) {
    const skill = findSkill(input.skillName);
    if (!skill) throw new Error(`Skill not found: ${input.skillName}`);

    // 사용자 스크립트 유형 Skill의 경우 하위 프로세스에서 실행
    if (skill.source === 'user' && skill.type === 'script') {
      const child = spawn('node', [skill.scriptPath, input.args], {
        cwd: context.getAppState().cwd,
        timeout: 60_000,
      });

      // 스트리밍 출력
      for await (const chunk of child.stdout) {
        yield { type: 'progress', output: chunk.toString() };
      }

      const exitCode = await child.exitPromise;
      if (exitCode !== 0) {
        throw new Error(`Skill exited with code ${exitCode}`);
      }

      return { type: 'success' };
    }

    // prompt 유형 Skill의 경우
    return { type: 'prompt', content: skill.getPromptForCommand(input.args) };
  }
});
```

## 7. Skill 검색 및 중복 제거

### 7.1 로딩 우선순위

```typescript
async function loadAllSkills(): Promise<Skill[]> {
  // 3개의 소스를 병렬로 로드
  const [bundled, plugins, user] = await Promise.all([
    loadBundledSkills(),
    loadPluginSkills(),
    loadUserSkills(),
  ]);

  // 사용자 Skill는 우선순위가 가장 높습니다(내장 재정의 가능).
  return deduplicateSkills([...user, ...plugins, ...bundled]);
}

function deduplicateSkills(skills: Skill[]): Skill[] {
  const seen = new Map<string, Skill>();

  for (const skill of skills) {
    // 동일한 이름 Skill가 처음으로 나타나는 경우는 예약되어 있습니다(사용자 > 플러그인 > 내장).
    if (!seen.has(skill.name)) {
      seen.set(skill.name, skill);
    }
  }

  return Array.from(seen.values());
}
```

### 사용자 Skill가 내장된 기능을 재정의할 수 있는 이유

사용자는 내장된 Skill의 동작을 사용자 정의해야 할 수도 있습니다. 예를 들어 `/commit`의 기본값은 conventional commits 형식이지만 사용자 팀에서는 다른 형식을 사용할 수 있습니다. `~/.claude/skills/` 내에서 동일한 이름의 `commit.md`를 생성함으로써 사용자는 원활한 커버리지를 가질 수 있습니다.

### 7.2 Memoized by cwd

```typescript
const skillCache = new Map<string, { skills: Skill[]; mtime: number }>();

function getSkillsForCwd(cwd: string): Skill[] {
  const cached = skillCache.get(cwd);
  const currentMtime = getSkillsDirMtime(cwd);

  if (cached && cached.mtime >= currentMtime) {
    return cached.skills;
  }

  const skills = loadAllSkillsSync(cwd);
  skillCache.set(cwd, { skills, mtime: currentMtime });
  return skills;
}
```

cwd에 의한 캐싱은 모든 명령 호출에서 skills 디렉터리를 다시 검색하지 않도록 합니다.

## 8. 디자인 결정 요약

| 결정 | 이유 |
|------|------|
| 지연 파일 추출 | 대부분의 Skill는 사용되지 않으며 불필요한 I/O를 피하십시오 |
| O_EXCL + O_NOFOLLOW | TOCTOU 경쟁 조건 및 심볼릭 링크 공격 방지 |
| Per-process nonce | 사전 생성된 공격 디렉터리 방지 |
| 플러그인은 기본적으로 활성화됩니다 | 구성이 필요 없는 환경, 기능 검색 임계값 낮추기 |
| isAvailable vs isEnabled | 환경적 제약과 사용자 선택의 구별 |
| 사용자 Skill가 내장 기능을 대체합니다 | 사용자 정의 내장 동작 허용 |
| 세 가지 소스에서 병렬 로딩 | 시작 지연 최소화 |
| cwd의 캐시 | 반복적인 스캔 방지 |

관련 장:

- [14 명령 시스템](14%20%EB%AA%85%EB%A0%B9%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): Skill를 명령어로 등록하는 방법
- [03 도구 시스템](03%20%EB%8F%84%EA%B5%AC%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): 도구 시스템에서 SkillTool의 역할
- [11 메모리 시스템](11%20%EB%A9%94%EB%AA%A8%EB%A6%AC%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): remember Skill 메모리 시스템과의 상호 작용
- [15 Feature Flags와 컴파일 최적화](15%20Feature%20Flags%EC%99%80%20%EC%BB%B4%ED%8C%8C%EC%9D%BC%20%EC%B5%9C%EC%A0%81%ED%99%94.md): 빌드 시 DCE가 Skill에 미치는 영향
