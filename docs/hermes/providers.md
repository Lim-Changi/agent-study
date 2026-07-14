# Hermes Agent — Provider 설정

Hermes Agent(하네스)가 **어떤 LLM(두뇌)으로 요청을 보낼지** 정하는 것이 provider 설정이다.
공식 문서: https://hermes-agent.nousresearch.com/docs/integrations/providers

## 핵심 원리

> **요청이 어디로 갈지는 "모델명"이 아니라 "provider 설정"이 결정한다.**

모델명의 슬래시 접두사(`anthropic/`, `openai/`)는 단지 **모델 식별자**일 뿐 라우팅과 무관하다.
`anthropic/claude-sonnet-4-6`이라 적어도 provider가 OpenRouter면 요청은 **OpenRouter로** 간다.

## provider 결정 우선순위

위에서부터 이긴다:

| 순위 | 출처 |
|---|---|
| 1 | CLI `--provider` 플래그 |
| 2 | `config.yaml`의 `model.provider` |
| 3 | 환경변수 `HERMES_INFERENCE_PROVIDER` |
| 4 | 셋 다 없으면 `"auto"` (시작 시 사용 가능한 자격증명 자동 탐지) |

엔드포인트 URL의 **단일 출처는 `config.yaml`의 `model.base_url`** 이다.
(v0.7.0 이후 `.env`의 `OPENAI_BASE_URL`은 더 이상 참조하지 않음)

## 역할 분리 — 이걸 헷갈리면 함정에 빠진다

```
config.yaml  →  어디로 갈지 (provider, base_url, model, api_mode)   [라우팅]
   .env      →  무슨 키로 인증할지 (API 키만)                        [인증]
모델명 접두사 →  그냥 이름표 (라우팅과 무관)
```

### ⚠️ 가장 흔한 함정

> **`.env`에 API 키만 추가해도 provider는 안 바뀐다.**

예: config.yaml이 이미 `provider: custom` + `base_url: http://localhost:11434/v1`(로컬 Ollama)로
잡혀 있으면, `.env`에 `OPENROUTER_API_KEY=...`를 넣어도 요청은 **그대로 Ollama로** 간다.
`.env`의 키는 "이미 정해진 base_url에 *어떤 자격증명*을 붙일지"만 고를 뿐, provider를 바꾸지 않는다.

provider를 진짜 바꾸려면 셋 중 하나:

1. `hermes model` (인터랙티브 피커) — **가장 권장**
2. `config.yaml`의 `model.provider` / `base_url` / `default` 직접 수정
3. 일회성: `hermes chat --provider <slug> -m <model>`

## 주요 provider 설정

### Nous Portal (권장)
300+ 프론티어 모델 + Tool Gateway(웹검색·이미지·TTS·브라우저)를 하나의 구독으로.
```bash
hermes setup --portal   # OAuth + provider + 게이트웨이 한 번에
hermes portal info      # 라우팅 정보 확인
```

### OpenRouter (기본값)
provider를 **생략하면 무조건 OpenRouter 경유**다. 하나의 키로 Claude·GPT·Gemini·Llama 등 접근.
```yaml
# config.yaml — provider 생략!
model:
  default: "anthropic/claude-sonnet-4-6"
```
```bash
# .env
OPENROUTER_API_KEY=sk-or-...
```
- `provider_routing`으로 가격·속도 기준 정렬, 특정 provider only/ignore 가능.
- Anthropic·OpenAI에 **직접** 붙으려면 provider를 명시해야 한다. (이 비대칭이 핵심)

### Anthropic (Claude) — 별칭 `claude` / `claude-code`
인증 방식 3가지:

| 방식 | 필요한 것 | 비용 |
|---|---|---|
| **API 키** | `ANTHROPIC_API_KEY` | 종량제 (API 크레딧) |
| **OAuth** | Claude Max 구독 + **추가 크레딧 필수**. `hermes model` → Anthropic 선택 → 로그인. **Claude Code 인증정보 자동 사용** | 구독 + 크레딧 |
| 레거시 | `ANTHROPIC_TOKEN` | - |

```yaml
model:
  provider: "anthropic"        # 별칭: claude / claude-code
  default: "claude-sonnet-4-6"
  context_length: 200000
```

> **참고 (자주 하는 오해)**: OAuth 방식은 Hermes가 Claude Code를 *호출*하는 게 아니라,
> Claude Code가 저장해둔 OAuth 자격증명을 **빌려서 Anthropic API에 직접** 요청하는 것.
> "Claude Code 구독으로 완전 무료"는 아니다 — Claude Max + 추가 크레딧이 필요하다.

### 로컬 / 자체 호스팅 (무료)
`provider: custom` + OpenAI 호환 `base_url`. 상세는 [local-ollama-setup.md](local-ollama-setup.md).
```yaml
model:
  default: qwen2.5-coder:7b
  provider: custom
  base_url: http://localhost:11434/v1
  context_length: 32768
```
- **tool-call 플래그 주의**: llama.cpp `--jinja`, vLLM `--enable-auto-tool-choice --tool-call-parser hermes`.

### 기타 1급 provider
`--provider <slug>` + 환경변수: fireworks, novita, zai(GLM), kimi-coding, minimax, alibaba(Qwen),
xai(Grok), deepseek, huggingface, gemini, bedrock, vertex, copilot 등.

## provider 전환 — `hermes model`

설정 파일을 직접 안 건드리고 인터랙티브 피커로 전환. 선택하면 `config.yaml`의
`model.default` / `provider` / `base_url` / `api_mode`가 자동 갱신된다. 대화 중엔 `/model`.

## 부가 기능

- **fallback_providers**: 주 provider 실패 시 백업 체인으로 자동 재시도 (`hermes fallback add/list/...`).
- **보조 모델(auxiliary)**: vision·web요약·컨텍스트압축 등 가벼운 작업에 별도 저렴 모델. 기본 `auto` → OpenRouter의 `gemini-3-flash-preview` 폴백.
- **확인/진단**: `hermes chat -q "..."` / 문제 시 `hermes doctor` / 사용량 `hermes usage`.

## 이 스터디에서의 활용

1. 로컬 Ollama(`custom`)로 붙여 provider·루프 메커니즘을 무료로 실습.
2. `hermes model`로 provider만 갈아끼우며 "같은 과제, 다른 두뇌" 성능 비교.
3. Claude가 필요하면 `claude-code` OAuth 또는 OpenRouter 소액으로 기준선 확인.
