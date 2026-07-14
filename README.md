# agent-study

Hermes Agent를 비롯한 AI 에이전트의 **동작 원리와 성능**을 직접 돌려보며 학습하는 스터디 레포.

## 스터디 목표

- AI 에이전트가 실제로 어떻게 도는지(agent loop, tool-calling, provider 라우팅)를 **손으로** 이해한다.
- **Hermes Agent**를 주 소재로, provider(두뇌)를 갈아끼우며 "같은 과제, 다른 모델"의 성능 차이를 관찰한다.
- 부딪히는 개념은 그때그때 `docs/`에 노트로 남긴다. → **구현 → 이해 → 기록** 사이클.

## 핵심 개념: 모델 vs 하네스

에이전트를 얘기할 때 두 개의 **레이어**가 섞이기 쉽다. 이 구분이 스터디의 출발점이다.

| 레이어 | 정체 | 예시 |
|---|---|---|
| **모델 (두뇌)** | 프롬프트를 받아 답을 뱉는 LLM 자체 | Claude, GPT, Gemini, Llama, Qwen |
| **하네스 (몸통)** | 모델을 감싸 tool-call·루프·파일접근을 붙인 에이전트 | **Hermes Agent**, Claude Code, OpenAI Codex CLI, Gemini CLI |

> **Hermes Agent는 "하네스"다** (모델이 아니다). `hermes` 명령으로 쓰는 CLI 에이전트 도구이며,
> 여러 **provider(모델 백엔드)**를 갈아끼울 수 있다. Claude Code와 같은 층에 있다.
>
> ⚠️ Nous Research의 Hermes *모델* 시리즈와 이름이 같지만 별개 개념이다. 이 레포에서 "Hermes"는 **에이전트(하네스)**를 가리킨다.

## 2-트랙 전략 (비용 최소화)

| 트랙 | 목적 | provider | 비용 |
|---|---|---|---|
| **로컬 트랙** | provider 설정·에이전트 루프 **메커니즘 학습** | Ollama 로컬 모델 (7B급) | 무료 |
| **클라우드 트랙** | 에이전트 **성능 기준선** 확인 | OpenRouter(소액) 또는 Anthropic OAuth | 종량제 |

> 메커니즘 학습은 로컬로 공짜, 성능 감별은 가끔 클라우드. 이렇게 나눠 비용을 최소화한다.

## 진행 인덱스

- [docs/hermes/providers.md](docs/hermes/providers.md) — Hermes Agent provider 설정 원리와 방법
- [docs/hermes/local-ollama-setup.md](docs/hermes/local-ollama-setup.md) — 16GB M1 Pro 맞춤 로컬 Ollama 셋업

### 다음 마일스톤 (예정)

- [x] Hermes Agent 설치 → 로컬 Ollama provider로 `hermes chat` 첫 연동 (2026-07-14)
- [x] tool-call 형식 불일치 관찰 & 해결 (qwen2.5-coder → hermes3:8b) → [docs/principles/tool-calling.md](docs/principles/tool-calling.md)
- [ ] `hermes model`로 provider 전환 실습 (Ollama ↔ OpenRouter/Anthropic)
- [ ] 같은 과제를 여러 provider로 돌려 tool-call·planning 비교 노트 작성
- [ ] 에이전트 원리 계속 정리 (ReAct, planning, memory) → `docs/principles/`

## 현재 로컬 셋업 (2026-07-14)

- 하네스: Hermes Agent v0.18.2 (`~/.hermes/`)
- 모델: `hermes3:8b` (Ollama, Hermes tool-call 포맷 네이티브, 128k 컨텍스트)
- provider: `custom` → `http://localhost:11434/v1`
- 환경: Apple M1 Pro / 16GB, 비용 0

## 참고

- Hermes Agent provider 공식 문서: https://hermes-agent.nousresearch.com/docs/integrations/providers
