# 로컬 Ollama 셋업 — 16GB M1 Pro 맞춤

Hermes Agent를 **비용 0**으로 테스트하기 위한 로컬 Ollama provider 셋업.
이 노트는 **Apple M1 Pro / 16GB RAM** 기준으로 모델 크기를 맞췄다.

## 왜 로컬 Ollama인가

- 토큰 비용 0, 오프라인, 데이터 외부 유출 없음.
- provider 설정·에이전트 루프 **메커니즘 학습**엔 충분하다.
- 단, 로컬에 올릴 수 있는 작은 모델은 에이전트 능력이 약하다 → 성능 판단용은 아니다.

## 모델 크기 선택 (16GB 제약)

문서 예시인 `qwen2.5-coder:32b`는 16GB에 **안 들어간다**(Q4로도 ~20GB). 7B급이 현실적 상한.

| 모델 | 대략 메모리(Q4) | 16GB 적합성 |
|---|---|---|
| 32b (문서 예시) | ~20GB | ❌ 불가 |
| 14b | ~9GB + 컨텍스트 | △ 빡빡 |
| **7b (qwen2.5-coder)** | ~4.5GB + 컨텍스트 | ✅ 권장 |

## 셋업 레시피

```bash
# 1. Ollama 설치
brew install ollama            # 또는 https://ollama.com 에서 다운로드

# 2. tool-call 지원 코딩 모델 받기
ollama pull qwen2.5-coder:7b

# 3. 컨텍스트 늘려서 서버 실행 (기본 4K는 너무 작아 하네스가 대화를 잊음)
OLLAMA_CONTEXT_LENGTH=32768 ollama serve
```

Hermes 설정:
```yaml
# ~/.hermes/config.yaml
model:
  default: qwen2.5-coder:7b
  provider: custom
  base_url: http://localhost:11434/v1
  context_length: 32768
```

연동 확인:
```bash
hermes chat -q "안녕, 너 어떤 모델이야?"
ollama ps    # CONTEXT 컬럼으로 실제 컨텍스트 확인
```

## 기대치 & 주의 ⚠️

1. **컨텍스트 타협** — 공식 문서는 최소 64k를 권장하지만, 16GB에서 7b+64k는 KV캐시 때문에 빡빡하다.
   **32k로 시작**하고 안정적이면 올린다. (컨텍스트가 작으면 긴 에이전트 루프에서 앞부분을 잊는다.)
2. **작은 모델 = 에이전트 능력 약함** — 7b는 tool-call은 하지만 다단계 계획·도구 사용이 서툴다.
   → 메커니즘 학습용으론 충분, **성능 판단용으론 부적합**.
3. **tool-call 깨짐** — 도구 호출이 실행 안 되고 `{"name": ..., "arguments": {...}}` 텍스트로 찍히면
   로컬 모델의 tool-parsing 실패 신호. qwen2.5-coder는 지원하지만, 문제 시 서버 옵션 점검
   (llama.cpp `--jinja`, vLLM `--enable-auto-tool-choice --tool-call-parser hermes`).
4. **컨텍스트 부족 진단** — `hermes chat` 시작 배너의 "Context limit" 값 확인. 너무 작으면
   `config.yaml`의 `context_length`로 강제 설정.

## 참고: context_length ≠ max_tokens

- **context_length**: 총 컨텍스트 윈도우 (입력 + 출력)
- **max_tokens**: 단일 응답의 최대 길이

## 다음 단계

로컬 연동이 확인되면 `hermes model`로 provider를 OpenRouter/Anthropic으로 갈아끼워
"같은 과제, 다른 두뇌"를 비교한다. → [providers.md](providers.md) 참고.
