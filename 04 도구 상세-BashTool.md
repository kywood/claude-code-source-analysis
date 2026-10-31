# 4장 도구 상세 설명 - BashTool

> **저자**: [Wang Yanshu](https://catyans.github.io)

> BashTool는 Claude Code의 가장 복잡한 도구 구현으로, 20개의 파일로 구성됩니다. 구문 보안 검사(23개 항목), 권한 규칙 일치, 읽기 전용 모드 적용 등 3계층 보안 아키텍처를 갖추고 있습니다. 이 장에서는 각 레이어의 디자인에 대한 심층 분석을 제공합니다.

## 1. 3계층 보안 아키텍처 개요

```
사용자 입력/모델 생성 명령
        │
        ▼
┌─────────────────────────────┐
│  레벨 1: bashSecurity.ts │ 문법 수준의 보안 점검(23개 항목)
│  (102KB, 위험모드 감지) │ → 주입, 난독화, 권한상승 거부
└──────────┬──────────────────┘
           │ 패스
           ▼
┌─────────────────────────────┐
│  레이어 2: bashPermissions.ts │ 권한 규칙 일치(98KB)
│  (명령어+경로+연산자 매칭) │ → 사용자/기업이 설정한 규칙 확인
└──────────┬──────────────────┘
           │ 패스
           ▼
┌─────────────────────────────┐
│  레이어 3: readOnlyValidation │ 읽기 전용 모드 강제(68KB)
│  (쓰기 동작 감지) │ → 읽기 전용 모드에서 쓰기 차단
└──────────┬──────────────────┘
           │ 패스
           ▼
        명령을 실행하다
```

세 계층은 독립적으로 작동하며 명확한 책임을 갖습니다.

- **계층 1**: 공격 방지(삽입, 난독화, 권한 상승) - 보안 문제
- **레이어 2**: 인증 확인(사용자가 이 명령 + 경로를 허용하는지) - 정책 문제
- **계층 3**: 강제 읽기 전용(제한된 모드에서 쓰기 방지) - 모드 문제

## 2. 23개 보안점검

### 2.1 ValidationContext

각 유효성 검사기는 통합된 컨텍스트 개체를 받습니다.

```typescript
interface ValidationContext {
  originalCommand: string;         // 원래 명령 문자열
  baseCommand: string;             // 구문 분석된 기본 명령(예: "git")
  unquotedContent: string;         // 따옴표 뒤의 내용을 제거하세요.
  fullyUnquotedContent: string;    // 모든 따옴표 레이어를 완전히 제거합니다.
  fullyUnquotedPreStrip: string;   // 따옴표를 제거하고 내용을 비워 두세요.
  unquotedKeepQuoteChars: string;  // 따옴표를 제거하되 따옴표 문자 자체는 유지하세요.
  treeSitter: TreeSitterAnalysis;  // Tree-sitter AST 분석 결과
}
```

`treeSitter`는 순수 정규식보다 더 정확한 명령에 대한 AST 수준 이해를 제공합니다. 예를 들어, 명령 인수의 `$()`와 실제 명령 대체를 구별할 수 있습니다.

### 2.2 전체 체크리스트

23가지 보안 검사가 모두 실행 순서대로 아래에 나열되어 있습니다.

#### 1. INCOMPLETE_COMMANDS(불완전한 명령)

```typescript
// 닫히지 않은 따옴표, 대괄호, 파이프 등을 감지합니다.
// 불완전한 구문을 통한 명령 삽입 방지
"echo 'hello"      // 거부: 닫히지 않은 작은따옴표
"cat file |"        // 거부 : 교수형 파이프
```

**위협**: 불완전한 명령은 shell에서 추가 입력을 기다리고 후속 명령 주입에 의해 악용될 수 있습니다.

#### 2. JQ_SYSTEM_FUNCTION (jq 시스템 기능)

```typescript
// jq에서 @base64d, @html, @json, env, $ENV 등을 감지합니다.
"jq 'env.HOME'"    // 거부됨: 환경 변수에 대한 액세스
```

**위협**: jq에는 환경 변수를 읽고 인코딩/디코딩을 수행하여 다른 검사를 우회할 수 있는 기능이 내장되어 있습니다.

#### 3. JQ_FILE_ARGUMENTS(jq 파일 매개변수)

```typescript
// jq의 --rawfile, --slurpfile 및 기타 파일 읽기 매개변수를 감지합니다.
"jq --rawfile x /etc/passwd"  // 거부됨: jq를 통해 민감한 파일 읽기
```

**위협**: jq는 매개변수를 통해 임의 파일의 내용을 읽을 수 있습니다.

#### 4. OBFUSCATED_FLAGS(혼란 플래그)

```typescript
// Unicode 또는 제어 문자로 위장한 명령줄 플래그 감지
"rm -\u200Brf /"   // 거부: 너비가 0인 공백이 플래그에 숨겨져 있습니다.
```

**위협**: 난독화된 플래그는 위험한 명령을 무해해 보이게 만들 수 있습니다.

#### 5. SHELL_METACHARACTERS(Shell 메타문자)

```typescript
// 위험한 shell 메타문자 조합 감지
"cat file; rm -rf /"    // 거부: 위험한 명령의 세미콜론 연결
"cat $(malicious)"      // 거부: 명령 대체
```

**위협**: 메타문자는 안전한 명령을 위험한 행동의 벡터로 바꿀 수 있습니다.

#### 6. DANGEROUS_VARIABLES(위험변수)

```typescript
// 키 shell 변수 수정 감지
"PATH=/tmp:$PATH cmd"   // 거부: PATH 변조
"LD_PRELOAD=evil.so"    // 거부됨: 라이브러리 삽입
```

**위협**: `PATH`, `LD_PRELOAD`, `LD_LIBRARY_PATH`를 수정하면 코드 삽입이 발생할 수 있습니다.

#### 7. NEWLINES(줄 바꿈)

```typescript
// 명령에서 줄바꿈 감지
"echo hello\nrm -rf /"  // 거부: 숨겨진 두 번째 명령
```

**위협**: 줄바꿈은 한 줄 명령처럼 보이는 추가 명령을 숨길 수 있습니다.

#### 8. COMMAND_SUBSTITUTION(명령 대체)

```typescript
// $() 및 `` 명령 대체 감지
"echo $(cat /etc/shadow)"  // 거부: 대체를 통한 정보 공개
```

**위협**: 명령 대체를 통해 매개변수 위치에서 임의 명령을 실행할 수 있습니다. 보안 예외가 있는 경우: git commit 메시지의 공통 패턴이 허용됩니다.

#### 9. INPUT_REDIRECTION(입력 리디렉션)

```typescript
// < 및 <<< 입력 리디렉션 감지
"mysql < drop_all.sql"  // 거부: 위험한 입력 리디렉션
```

**위협**: 입력 리디렉션은 프로그램에 임의의 콘텐츠를 삽입할 수 있습니다. 예외가 있습니다. heredoc(`<< 'EOF'`)는 특정 보안 시나리오에서 허용됩니다.

#### 10. OUTPUT_REDIRECTION(출력 리디렉션)

```typescript
// > 및 >> 출력 리디렉션 감지
"echo malicious > ~/.bashrc"  // 거부: 구성 파일 덮어쓰기
```

**위협**: 출력 리디렉션은 임의의 파일 콘텐츠를 덮어쓰거나 추가할 수 있습니다.

#### 11. IFS_INJECTION (IFS 주입)

```typescript
// IFS 변수 수정 감지
"IFS=/ cmd arg"  // 거부: 필드 구분 기호 변경
```

**위협**: `IFS`(Internal Field Separator)를 수정하면 shell가 매개변수를 구문 분석하는 방식이 변경되어 명령이 예기치 않게 분할될 수 있습니다.

#### 12. GIT_COMMIT_SUBSTITUTION(교체를 위해 제출된 Git)

```typescript
// git commit 메시지에서 명령 대체 감지
"git commit -m '$(whoami)'"  // REJECT: 커밋 메시지의 명령 대체
```

**위협**: 일부 git 후크는 커밋 메시지의 특수 문자를 처리할 수 있습니다.

#### 13. PROC_ENVIRON_ACCESS (/proc 환경 접근)

```typescript
// /proc/self/environ와 같은 경로에 대한 액세스를 감지합니다.
"cat /proc/self/environ"  // 거부: 환경 변수 유출(키 포함)
```

**위협**: `/proc/self/environ`에는 모든 환경 변수가 포함되어 있으며 API 키와 같은 민감한 정보가 포함될 수 있습니다.

#### 14. MALFORMED_TOKEN_INJECTION (잘못된 Token 주입)

```typescript
// token 경계에 삽입된 특수 문자 감지
"cmd arg\x00hidden"  // 거부됨: null 바이트 잘림
```

**위협**: null 바이트 및 기타 제어 문자는 명령 구문 분석을 자르거나 수정할 수 있습니다.

#### 15. BACKSLASH_ESCAPED_WHITESPACE(백슬래시로 이스케이프 처리된 공백)

```typescript
// 백슬래시로 이스케이프 처리된 공백 문자 감지
"rm\ -rf\ /"  // 거부: 이스케이프 처리하여 인수 분리 숨기기
```

**위협**: 공백을 이스케이프하면 매개변수 경계가 변경되어 위험한 매개변수가 안전한 매개변수로 병합될 수 있습니다.

#### 16. BRACE_EXPANSION(중괄호 확장)

```typescript
// 위험한 버팀대 확장 감지
"cat /etc/{passwd,shadow}"  // 거부: 확장을 통해 여러 중요한 파일에 대한 액세스
```

**위협**: 중괄호 확장은 단일 경로 권한 확인을 우회하여 단일 명령으로 여러 경로에 액세스할 수 있습니다.

#### 17. CONTROL_CHARACTERS(제어 문자)

```typescript
// ASCII 제어 문자 감지(\t\n\r 제외)
"cmd \x1b[2J"  // 거부됨: 터미널 이스케이프 시퀀스
```

**위협**: 제어 문자는 터미널 디스플레이를 조작하여 실행된 실제 명령을 숨길 수 있습니다.

#### 18. UNICODE_WHITESPACE (Unicode 공백)

```typescript
// ASCII가 아닌 공백 문자 감지
"cmd\u2003arg"  // 거부: em space 등 Unicode 공백
```

**위협**: Unicode 공백 문자는 터미널에 표시되지 않으며 명령 구문 분석을 변경할 수 있습니다.

#### 19. MID_WORD_HASH(파운드 기호 단어)

```typescript
// 단어 중간에 나타나는 # 문자 감지
"cmd#comment rm -rf"  // 거부: 주석 삽입 가능
```

**위협**: 일부 shell 구성에서는 `#`가 주석을 시작하여 이전 콘텐츠가 무시될 수 있습니다.

#### 20. ZSH_DANGEROUS_COMMANDS (zsh 위험한 명령)

```typescript
// zsh 고유의 위험한 내장 명령 감지
"zmodload zsh/system"  // 거부됨: 시스템 모듈 로드 중
"sched +5 malicious"   // 거부: 예약된 실행
```

**위협**: zsh에는 bash에는 없지만 시스템 수준 작업을 허용하는 몇 가지 내장 명령이 있습니다.

#### 21. BACKSLASH_ESCAPED_OPERATORS(백슬래시 이스케이프 연산자)

```typescript
// 백슬래시로 이스케이프 처리된 shell 연산자 감지
"cmd \&\& rm -rf"  // 거부: 숨겨진 명령 링크
```

**위협**: 특정 구문 분석 단계에서 이스케이프 연산자가 되돌려져 예상치 못한 명령 연결이 발생할 수 있습니다.

#### 22. COMMENT_QUOTE_DESYNC (댓글 인용문이 동기화되지 않았습니다)

```typescript
// 인용문과 댓글의 불일치 상태 감지
"echo 'hello' #' rm -rf /"  // 거부: 인용/댓글 혼동
```

**위협**: 다양한 파서(사람의 눈, shell, 보안 검사)는 인용/주석 경계를 다르게 해석할 수 있습니다.

#### 23. QUOTED_NEWLINE(따옴표 안의 줄바꿈)

```typescript
// 인용된 문자열 내에서 개행 문자 감지
"echo 'line1\nrm -rf /'"  // 상황을 확인하여 안전한지 결정하세요.
```

**위협**: 일부 시나리오에서는 따옴표 안의 줄 바꿈이 shell에 의해 실제 줄 바꿈으로 확장됩니다.

## 3. 검증인 체인 실행 순서

23개의 검증자는 **조기 허용(early-allow)** 메커니즘을 사용하여 엄격한 순서로 실행됩니다.

```typescript
function runValidators(ctx: ValidationContext): SecurityResult {
  // 안전 모드 패스트 패스
  if (isSafeGitCommit(ctx)) return { allowed: true };
  if (isSafeHeredoc(ctx)) return { allowed: true };
  if (isSafeKnownCommand(ctx)) return { allowed: true };

  // 23개의 검증인을 순차적으로 실행
  for (const validator of VALIDATORS) {
    const result = validator(ctx);
    if (result.denied) {
      return result; // 첫 번째 거부 시 중지
    }
  }

  return { allowed: true };
}
```

사전 권한 검사가 필요한 이유:

- **git commit**: 커밋 메시지에는 특수 문자가 포함되는 경우가 많습니다. 이러한 문자는 git의 컨텍스트에서 안전합니다.
- **heredoc**: heredoc 구문(`<< 'EOF'`)에는 줄 바꿈 및 특수 문자가 필요하며 따옴표로 보호됩니다.
- **알려진 안전 명령**: `ls`, `pwd`, `date` 등과 같은 순수 읽기 명령.

## 4. 권한 규칙 일치

### 4.1 bashToolHasPermission 프로세스

```
명령 문자열
    │
    ▼
분석(Parse)
    │ 기본 명령, 매개변수, 경로 추출
    ▼
의미론적 검사(Semantics Check)
    │ 명령 의도 이해
    ▼
보안검색(Security Checks)
    │ 23개 항목 유효성 검사기
    ▼
경로 제약 조건(Path Constraints)
    │ 명령과 관련된 파일 경로를 확인하십시오.
    ▼
명령 연산자 권한(Command Operator Permissions)
    │ 파이프, 리디렉션 등을 확인하세요.
    ▼
권한 모드(Permission Mode)
    │ ask / auto / bypass
    ▼
최종 결정
```

### 4.2 경로 제약 조건 확인

```typescript
function checkPathConstraints(
  command: ParsedCommand,
  rules: PermissionRule[]
): PermissionDecision {
  for (const path of command.involvedPaths) {
    const resolved = resolvePath(path, command.cwd);

    for (const rule of rules) {
      if (rule.pathPattern && matchPath(resolved, rule.pathPattern)) {
        if (rule.decision === 'deny') {
          return { decision: 'deny', reason: `경로 ${resolved} 규칙에 의해 금지됨` };
        }
      }
    }
  }

  return { decision: 'continue' }; // 확인하려면 다음 단계로 진행하세요.
}
```

### 4.3 명령 연산자 권한

파이프라인(`|`), 백그라운드(`&`) 및 명령 체인(`&&`, `||`)과 같은 연산자는 명령의 실행 의미를 변경합니다. 권한 시스템에는 다음 연산자에 대한 독립적인 검사가 있습니다.

```typescript
// 파이프라인의 각 명령은 개별적으로 확인됩니다.
"cat file | grep pattern"
→ checkPermission("cat file")    // 읽기 전용: 허용
→ checkPermission("grep pattern") // 읽기 전용: 허용
→ 전체: 허용

"cat file | rm -rf /"
→ checkPermission("cat file")    // 읽기 전용: 허용
→ checkPermission("rm -rf /")    // 위험: 거절
→ 전체: 거부
```

## 5. 읽기 전용 모드 적용

### 5.1 쓰기 동작 감지

```typescript
function isWriteOperation(command: ParsedCommand): boolean {
  // 알려진 파괴적인 명령 확인
  if (DESTRUCTIVE_COMMANDS.includes(command.baseCommand)) return true;

  // sed/awk의 쓰기 모드를 확인하세요.
  if (command.baseCommand === 'sed' && command.args.includes('-i')) return true;
  if (command.baseCommand === 'awk' && hasInplaceFlag(command)) return true;

  // 안전 명령 허용 목록 확인
  if (SAFE_COMMANDS.includes(command.baseCommand)) return false;

  // 기본 보수: 알 수 없는 명령은 쓰기 가능한 것으로 간주됩니다.
  return true;
}
```

### 5.2 파괴 명령 목록

```typescript
const DESTRUCTIVE_COMMANDS = [
  'rm', 'rmdir',           // 삭제
  'mv',                     // 이동/이름 바꾸기
  'cp',                     // 복사(덮어쓰기 가능)
  'chmod', 'chown', 'chgrp', // 권한 수정
  'touch',                  // 타임스탬프 생성/수정
  'mkdir',                  // 디렉터리 생성
  'ln',                     // 링크 만들기
  'dd',                     // 장치 작동 차단
  'truncate',               // 파일 자르기
  'shred',                  // 안전 삭제
  // ...자세히
];
```

### 5.3 보안 명령 화이트리스트

```typescript
const SAFE_COMMANDS = [
  'ls', 'dir', 'find',     // 목록/검색
  'cat', 'head', 'tail',   // 읽다
  'grep', 'rg', 'ag',      // 검색
  'wc', 'sort', 'uniq',    // 텍스트 처리(수정 없이 출력만 가능)
  'file', 'stat', 'du',    // 정보조회
  'git log', 'git status', // Git 읽기 전용 작업
  'git diff', 'git show',
  'pwd', 'which', 'type',  // 환경 쿼리
  'date', 'uname', 'id',   // 시스템 정보
  'echo', 'printf',        // 출력
  // ...자세히
];
```

## 6. 자동 분류기(Auto-Classifier)

### 6.1 추측 분류 확인

```typescript
async function speculativeClassifierCheck(
  command: string,
  context: ClassifierContext
): Promise<ClassificationResult> {
  // 빠른 패턴 매칭
  const quickResult = quickPatternMatch(command);
  if (quickResult !== 'unknown') return quickResult;

  // 분류를 위해 작은 모델 호출
  return await classifyWithModel(command, context);
}
```

### 6.2 보류 중인 분류 확인

```typescript
// 권한 확인 유예 기간 내에 실행
const pendingClassifierCheck = createPendingClassifier({
  gracePeridMs: 200,  // 200ms 유예 기간
  onResult: (result) => {
    if (result === 'allow') {
      resolvePermission({ decision: 'allow' });
    }
  },
});
```

투기적 분류자는 권한 대화 상자가 사용자에게 표시되기 전에 실행됩니다. 200ms 유예기간 내에 안전하다고 분류되면 사용자 방해 없이 자동 승인됩니다.

### 유예기간은 왜 필요한가요?

사용자가 빠르게 입력하면 사용자가 계속 입력하는 동안 도구 호출이 트리거될 수 있습니다. 200ms의 유예 기간은 자동 분류자가 판단을 내리고 사용자가 입력하는 동안 권한 대화 상자가 팝업되는 것을 방지할 수 있는 시간을 제공합니다.

## 7. 디자인 결정 요약

### 왜 하나가 아닌 세 개의 레이어가 필요한가요?

단일 계층 보안 검사의 문제점:

- 보안점검과 정책점검이 혼합되어 유지관리가 어려움
- 읽기 전용 모드의 로직과 보안 점검 로직이 얽혀있습니다.
- 다양한 배포 시나리오에는 다양한 정책 구성이 필요하지만 보안 검사는 통합되어야 합니다.

세 가지 분리 계층을 통해 다음이 가능합니다.

- 보안 검사(Tier 1)는 모든 배포에 균일하게 적용됩니다.
- 권한 규칙(계층 2)은 기업 관리자가 사용자 정의할 수 있습니다.
- 읽기 전용 모드(레이어 3)는 런타임 시 전환 가능

### 왜 23개를 확인해야 하나요?

각 검사는 특정 공격 벡터를 대상으로 합니다. 검사 결과를 다음 항목에 병합합니다.

- 개별 검사를 비활성화하는 데 어려움이 있음(디버깅 또는 특수 시나리오)
- 부정확한 오류 보고(사용자는 어떤 규칙이 트리거되었는지 알 수 없음)
- 초기에 논리 복잡성을 허용

### Tree-sitter의 역할

순수 정규 표현식의 한계:

- 따옴표 안팎의 특수문자를 구분할 수 없습니다.
- 중첩된 구조를 이해할 수 없음
- 구조적 우회로 쉽게 속임

Tree-sitter는 shell 스크립트의 AST를 제공하여 텍스트 수준이 아닌 구문 수준에서 보안 검사를 실행할 수 있도록 합니다. 예를 들어, `$(...)`가 큰따옴표 안에 있는지(안전함) 명령 위치에 있는지(위험함) 정확하게 식별합니다.

### 추측 분류기의 설계

투기적 분류자는 "낙관적 잠금"입니다. 이는 대부분의 명령이 안전하다고 가정하고 먼저 자동 승인을 시도하며 실패할 경우에만 대화형 승인으로 에스컬레이션합니다. 이렇게 하면 권한 대화 상자의 빈도가 줄어들고 보안을 희생하지 않고도 사용자 경험이 향상됩니다(일부 메커니즘은 항상 최종적으로 결정을 내립니다).

관련 장:

- [07 권한 시스템](07%20%EA%B6%8C%ED%95%9C%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): 권한 결정 파이프라인의 전체 프로세스
- [03 도구 시스템](03%20%EB%8F%84%EA%B5%AC%20%EC%8B%9C%EC%8A%A4%ED%85%9C.md): 도구 시스템에 BashTool 등록
- [02 쿼리 엔진](02%20%EC%BF%BC%EB%A6%AC%20%EC%97%94%EC%A7%84.md): BashTool 실행 결과 처리
- [15 Feature Flags와 컴파일 최적화](15%20Feature%20Flags%EC%99%80%20%EC%BB%B4%ED%8C%8C%EC%9D%BC%20%EC%B5%9C%EC%A0%81%ED%99%94.md): 일부 보안 검사는 feature flag에 의해 제어됩니다.
