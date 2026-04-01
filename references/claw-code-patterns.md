# Production Harness Patterns — 실전 아키텍처 레퍼런스

상용 에이전트 하네스에서 검증된 3가지 핵심 패턴. 각 패턴은 특정 Stage에서 설계 결정의 구체적 근거로 활용.

---

## 1. Token Scoring Router (Stage 2 연동)

프롬프트를 도구/커맨드 인벤토리에 매칭하는 경량 라우팅 알고리즘. Routing 패턴의 **구현 레퍼런스**.

### 동작 원리

```
사용자 프롬프트
  ↓
토큰화 (공백, '/', '-' 기준으로 분리)
  ↓
인벤토리 스캔 (커맨드 + 도구 목록 순회)
  ↓
매칭 스코어 계산:
  - name 필드에 토큰 포함? → +2점
  - source_hint에 포함? → +1점
  - responsibility에 포함? → +1점
  ↓
정렬: (-score, kind, name)
  ↓
종류별 상위 N개 수집 → 오버플로우 처리
```

### 의사코드

```python
def route_prompt(prompt: str, inventory: list[Module], limit: int) -> list[Match]:
    tokens = tokenize(prompt)  # split on whitespace, '/', '-'
    scored = []
    for module in inventory:
        score = 0
        for token in tokens:
            t = token.lower()
            if t in module.name.lower():
                score += 2
            if t in module.source_hint.lower():
                score += 1
            if t in module.responsibility.lower():
                score += 1
        if score > 0:
            scored.append(Match(module, score))

    scored.sort(key=lambda m: (-m.score, m.kind, m.name))
    return collect_top_per_kind(scored, limit)
```

### 설계 포인트

| 결정 | 이유 |
|------|------|
| LLM 없이 토큰 스코어링 | 지연 0ms, 비용 0, 결정론적 |
| 종류별 상위 N개 수집 | 커맨드와 도구 모두 고르게 매칭 보장 |
| source_hint 포함 | 모듈의 출처(plugin, skill, core)도 라우팅 힌트로 활용 |

### 적용 가이드

- **단순 라우팅**: 이 패턴으로 충분 (비용 0, 빠름)
- **의미 라우팅 필요 시**: 이 패턴을 1차 필터로, LLM 기반 분류를 2차로 결합 (Two-Level Dispatch)
- **하이브리드**: 토큰 스코어링으로 후보 5~10개 선별 → LLM이 최종 1~2개 선택

---

## 2. Execution Registry + Tool Specification (Stage 5 연동)

도구를 **선언(Specification) → 등록(Registry) → 실행(Execution)** 3단계로 분리하는 패턴. ACI 원칙의 **구체적 구현 구조**.

### 아키텍처

```
[도구 명세 (JSON/선언적)]
    ↓ 로드 (1회, 캐시)
[불변 레지스트리 (Immutable Registry)]
    ↓ 조회/필터링
[실행 래퍼 (Execution Wrapper)]
    ↓ 실행
[결과 반환]
```

### 3개 계층

#### 계층 1: Tool Specification (선언)

```python
# 도구를 JSON Schema로 선언
tool_spec = {
    "name": "read_file",
    "description": "Read contents of a file at the given path",
    "input_schema": {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "Absolute file path"},
            "limit": {"type": "integer", "description": "Max lines to read"}
        },
        "required": ["path"]
    }
}
```

**핵심**: 도구의 메타데이터와 실행 로직을 분리. 메타데이터만으로 LLM이 도구를 이해하고 선택할 수 있어야 함.

#### 계층 2: Immutable Registry (등록)

```python
@cache  # 한 번만 로드, 이후 캐시
def load_tool_registry() -> tuple[ToolSpec, ...]:
    snapshot = load_json("tools_snapshot.json")
    return tuple(ToolSpec.from_dict(entry) for entry in snapshot)

def get_tool(name: str) -> ToolSpec | None:
    return next((t for t in load_tool_registry() if t.name == name), None)

def get_tools(*, permission_ctx=None, simple_mode=False) -> tuple[ToolSpec, ...]:
    tools = load_tool_registry()
    if permission_ctx:
        tools = tuple(t for t in tools if not permission_ctx.blocks(t.name))
    if simple_mode:
        tools = tuple(t for t in tools if t.is_mvp)
    return tools
```

**핵심**: 불변(immutable) 튜플 + 캐시. 필터링은 합성 가능(composable) — 권한, 모드, MCP 포함 여부를 독립적으로 조합.

#### 계층 3: Execution Wrapper (실행)

```python
@dataclass(frozen=True)
class ExecutableModule:
    name: str
    source_hint: str

    def execute(self, payload: str) -> ExecutionResult:
        # 실제 실행 로직
        return ExecutionResult(handled=True, message=output)

@dataclass(frozen=True)
class ExecutionRegistry:
    commands: tuple[ExecutableModule, ...]
    tools: tuple[ExecutableModule, ...]

    def find(self, name: str) -> ExecutableModule | None:
        # 대소문자 무시 조회
        ...
```

**핵심**: 명세(spec)와 실행(execution)을 래퍼로 연결. 테스트 시 mock 주입 가능.

### 설계 포인트

| 결정 | 이유 |
|------|------|
| JSON Schema로 명세 | LLM이 학습한 형식, 자동 검증 가능 |
| 불변 튜플 + 캐시 | 동시 접근 안전, 상태 오염 방지 |
| 합성 가능한 필터 | 권한, 모드, MCP를 독립적으로 on/off |
| 래퍼로 실행 분리 | 테스트, mock, 로깅을 실행 계층에서만 처리 |

### ACI 원칙 대응

| ACI 원칙 | 이 패턴에서의 구현 |
|----------|-------------------|
| 명확성 | ToolSpec.name + description이 자기 설명적 |
| 문서화 | input_schema가 곧 문서 (JSON Schema) |
| 포카요케 | required 필드 강제, 타입 검증 자동화 |
| 자연스러운 포맷 | JSON Schema — LLM에게 자연스러운 형식 |

---

## 3. Two-Stage Permission Model (Stage 6 연동)

가드레일을 **정적 필터(빌드 타임) + 동적 승인(런타임)** 2단계로 분리하는 패턴.

### 아키텍처

```
도구 호출 요청
  ↓
[Stage 1: 정적 필터] ← 설정 파일/환경 변수에서 로드
  deny_names: {"delete_database", "rm_rf"}     (정확 매칭)
  deny_prefixes: {"mcp__dangerous_"}            (접두어 매칭)
  ↓
  차단됨? → 즉시 거부 (비용 0, 지연 0)
  통과? → Stage 2로
  ↓
[Stage 2: 동적 승인] ← 런타임 정책 엔진
  도구별 정책 조회:
    Allow → 즉시 실행
    Deny  → 사유와 함께 거부
    Prompt → 사용자에게 승인 요청
  ↓
  승인? → 실행
  거부? → 결과에 거부 사유 포함
```

### 의사코드

#### Stage 1: Static Filter (정적 필터)

```python
@dataclass(frozen=True)
class PermissionFilter:
    deny_names: frozenset[str]       # 정확 매칭 (대소문자 무시)
    deny_prefixes: tuple[str, ...]   # 접두어 매칭

    def blocks(self, tool_name: str) -> bool:
        lower = tool_name.lower()
        if lower in self.deny_names:
            return True
        return any(lower.startswith(p.lower()) for p in self.deny_prefixes)
```

**용도**: 환경별 도구 차단 (개발/스테이징/프로덕션), MCP 서버별 도구 필터링

#### Stage 2: Dynamic Authorization (동적 승인)

```python
class PermissionMode(Enum):
    ALLOW = "allow"      # 무조건 허용
    DENY = "deny"        # 무조건 거부
    PROMPT = "prompt"    # 사용자 확인 필요

@dataclass
class PermissionPolicy:
    default_mode: PermissionMode
    tool_overrides: dict[str, PermissionMode]  # 도구별 오버라이드

    def authorize(self, tool_name: str, input_data: dict,
                  prompter: PermissionPrompter | None) -> AuthResult:
        mode = self.tool_overrides.get(tool_name, self.default_mode)

        if mode == PermissionMode.ALLOW:
            return AuthResult.granted()
        if mode == PermissionMode.DENY:
            return AuthResult.denied(reason="Policy: tool denied")
        if mode == PermissionMode.PROMPT and prompter:
            decision = prompter.decide(PermissionRequest(tool_name, input_data))
            return AuthResult.from_decision(decision)
        return AuthResult.denied(reason="No prompter available")

# 환경별 Prompter 구현
class CLIPrompter(PermissionPrompter):
    def decide(self, req): ...  # stdin/stdout 대화

class WebPrompter(PermissionPrompter):
    def decide(self, req): ...  # WebSocket 이벤트

class AutoApprovePrompter(PermissionPrompter):
    def decide(self, req): return Decision.APPROVE  # CI/CD용
```

### 설계 포인트

| 결정 | 이유 |
|------|------|
| 2단계 분리 | 정적 필터는 비용 0으로 명확한 거부 처리, 동적 승인은 맥락 기반 판단 |
| frozenset + tuple | 불변 구조로 런타임 변조 방지 |
| Prompter 추상화 | CLI/Web/CI 등 환경에 무관한 승인 로직 |
| 도구별 오버라이드 | 기본 정책 + 예외 처리의 유연성 |

### 기존 가드레일 카탈로그와의 관계

```
가드레일 카탈로그 (6유형)
  └─ 도구 세이프가드 (유형 5)
      └─ Two-Stage Permission Model ← 이 패턴
          ├─ Stage 1: 규칙 기반 보호 (유형 6)와 결합
          └─ Stage 2: 도구 세이프가드의 런타임 구현
```

이 패턴은 카탈로그의 유형 5(도구 세이프가드)와 유형 6(규칙 기반 보호)을 **하나의 실행 파이프라인으로 통합**한 것.

---

## 패턴 간 관계

```
[Token Scoring Router]
    ↓ 매칭된 도구 목록
[Execution Registry]
    ↓ 도구 조회 + 필터링
[Two-Stage Permission]
    ↓ 승인된 도구만 실행
[실행 결과]
```

3개 패턴은 독립적으로 채택할 수 있지만, 결합하면 **"프롬프트 → 라우팅 → 도구 선택 → 권한 검증 → 실행"** 의 완전한 파이프라인을 형성한다.

---

## 채택 가이드

| 프로젝트 규모 | 권장 패턴 |
|-------------|----------|
| 단일 에이전트, 도구 5개 미만 | Two-Stage Permission만 |
| 중규모, 도구 10~20개 | Execution Registry + Two-Stage Permission |
| 대규모, 다중 에이전트 | 3개 패턴 모두 |

**주의**: 이 패턴들은 "필요할 때만 복잡성을 추가"하는 원칙에 따라 선택적으로 채택할 것. 단순한 에이전트에 3개 패턴을 모두 적용하면 과잉 엔지니어링.
