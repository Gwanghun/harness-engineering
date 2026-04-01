# Harness Engineering

AI 에이전트 하네스를 설계하고 실행 가능한 산출물을 생성하는 Claude Code 스킬.

## 개요

AI 에이전트 기반 기능을 개발할 때 필요한 하네스(가드레일, 오케스트레이션, 도구 설계, 평가 시스템, 인간 개입)를 **9단계 대화형 워크플로우**로 설계합니다. OpenAI와 Anthropic의 에이전트 구축 가이드에 기반합니다.

**"설계 문서"에서 끝나지 않고, 바로 실행 가능한 에이전트 정의 파일과 스킬 파일까지 산출합니다.**

## 워크플로우

| Stage | 단계 | 산출물 |
|:-----:|------|--------|
| 1 | **적합성 진단** | 적합성 결과 + 사용자 숙련도 판별 |
| 2 | **오케스트레이션 설계** | 실행 모드 + 패턴 + 데이터 전달 방식 + 아키텍처 다이어그램 |
| 3 | **에이전트 정의** | `.claude/agents/{name}.md` 파일 |
| 4 | **지침 및 스킬 설계** | 시스템 프롬프트 + `.claude/skills/{name}/skill.md` 파일 |
| 5 | **도구 설계** | 도구 명세서 + 코드 스캐폴딩 |
| 6 | **가드레일** | 가드레일 설계서 + 구현 코드 |
| 7 | **평가 시스템** | 평가 메트릭 + 테스트 케이스 |
| 8 | **인간 개입 + 배포** | 인간 개입 설계서 + 배포 로드맵 |
| 9 | **검증 및 테스트** | 검증 결과 보고서 (구조/트리거/실행/드라이런) |

## 설치 방법

`~/.claude/skills/` 디렉토리에 클론합니다:

```bash
cd ~/.claude/skills
git clone https://github.com/Gwanghun/harness-engineering.git
```

## 사용 방법

Claude Code에서 다음과 같이 호출합니다:

```
/harness-engineering
```

또는 자연어로 요청하면 자동 트리거됩니다:

- "AI 에이전트 설계해줘"
- "하네스 설계"
- "에이전트 팀 구성해줘"
- "가드레일 설계"
- "에이전트 오케스트레이션 패턴 선택"
- "human-in-the-loop 전략 수립"

## 주요 기능

- **사용자 숙련도 감지** -- 초급/중급/고급에 따라 진행 깊이 자동 조절
- **에이전트 팀 지원** -- 에이전트 팀 vs 서브 에이전트 실행 모드 선택
- **데이터 전달 프로토콜** -- 메시지/태스크/파일 기반 전략 정의
- **에이전트 정의 파일 생성** -- `.claude/agents/{name}.md` 직접 생성
- **스킬 파일 생성** -- Progressive Disclosure 기반 스킬 구조화
- **ACI 도구 설계** -- 명확성, 문서화, 포카요케, 자연스러운 포맷 + Execution Registry 패턴
- **계층적 가드레일** -- 6가지 유형 + 낙관적 실행 패턴 + Two-Stage Permission 모델
- **실전 아키텍처 패턴** -- 상용 하네스에서 검증된 Token Scoring Router, Execution Registry, Two-Stage Permission 의사코드 제공
- **검증 및 테스트** -- 구조/트리거/실행/드라이런 4단계 검증

## 디렉토리 구조

```
harness-engineering/
├── SKILL.md                                # 메인 스킬 (307줄)
├── README.md
└── references/
    ├── orchestration-patterns.md            # 7가지 오케스트레이션 패턴
    ├── instructions-design.md               # 지침 작성 4단계
    ├── tool-design-aci.md                   # ACI 설계 4원칙
    ├── guardrails-catalog.md                # 6가지 가드레일 유형
    ├── evaluation-system.md                 # 평가 메트릭 + 테스트 케이스
    ├── human-in-the-loop.md                 # 체크포인트 + 자율성 단계
    ├── verification-testing.md              # 검증 방법론 상세
    └── claw-code-patterns.md               # 실전 하네스 아키텍처 패턴 (NEW)
```

## 참조 문서

| 파일 | 내용 | 관련 Stage |
|------|------|:----------:|
| `orchestration-patterns.md` | Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer, Manager, Handoff 패턴의 결정 트리와 비교 매트릭스 | 2 |
| `instructions-design.md` | LLM 친화적 지침 작성 4단계, 엣지 케이스 체크리스트, 품질 검증 기준 | 4 |
| `tool-design-aci.md` | 도구 3유형(데이터/액션/오케스트레이션), ACI 4원칙, 명세 템플릿 | 5 |
| `guardrails-catalog.md` | 관련성 분류기, 안전성 분류기, PII 필터, 모더레이션, 도구 세이프가드, 규칙 기반 보호 | 6 |
| `evaluation-system.md` | 기능/성능/안전 메트릭, 테스트 케이스 4유형(Happy/Edge/Adversarial/Failure), 반복 개선 사이클 | 7 |
| `human-in-the-loop.md` | Approval Gate, Review Checkpoint, Confidence-Based Escalation, 점진적 자율성 확대 4단계 | 8 |
| `verification-testing.md` | 구조 검증, 트리거 검증(should/should-NOT trigger), 실행 테스트(with/without skill), 드라이런 | 9 |
| `claw-code-patterns.md` | Token Scoring Router, Execution Registry, Two-Stage Permission — 상용 하네스 아키텍처 패턴 의사코드 | 2, 5, 6 |

## 연동 스킬

이 스킬은 다음 Claude Code 스킬과 연동됩니다:

| 스킬 | 연동 시점 |
|------|----------|
| `/plan` | Stage 2 -- 구현 계획 수립 |
| `mcp-builder` | Stage 5 -- MCP 서버 구현 |
| `tdd` | Stage 7 -- 테스트 코드 구현 |
| `/plan-eng-review` | 완료 후 -- 엔지니어링 리뷰 |

## 라이선스

Apache-2.0
