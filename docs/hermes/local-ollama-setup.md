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

# 3. 64k 컨텍스트 + 메모리 절약 옵션으로 서버 실행
#    (Hermes는 최소 64k를 "강제"한다. 아래는 실측으로 확정된 조합)
OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0 OLLAMA_CONTEXT_LENGTH=65536 ollama serve
```

Hermes 설정:
```yaml
# ~/.hermes/config.yaml
model:
  default: qwen2.5-coder:7b
  provider: custom
  base_url: http://localhost:11434/v1
  context_length: 65536
  ollama_num_ctx: 65536      # ★ 필수. 이게 없으면 Ollama가 모델 native(32k)로 로드해 거부당함
```

연동 확인:
```bash
hermes chat -q "안녕, 너 어떤 모델이야?"
ollama ps    # CONTEXT 컬럼으로 실제 컨텍스트 확인 (65536 이어야 함)
```

## 실제 셋업에서 겪은 이슈 (2026-07-14, 실측)

이 3개는 문서만 봐선 모르고 직접 붙이며 알게 된 것들 — 스터디의 핵심 기록.

1. **Hermes는 64k를 "권장"이 아니라 "강제"한다.**
   `context_length: 32768`이면 초기화 자체가 거부된다:
   > `Model ... has a context window of 32,768 tokens, which is below the minimum 64,000 required`
   → `context_length: 65536`으로 올려야 통과.

2. **config만 올려도 안 된다 — Ollama 런타임 컨텍스트도 실제로 64k여야 한다.**
   config를 65536으로 해도 Ollama가 모델을 native 32k로 로드하면 또 거부된다:
   > `Ollama loaded qwen2.5-coder:7b with only 32,768 tokens of runtime context`
   qwen2.5-coder:7b의 native 컨텍스트가 32k라서 `OLLAMA_CONTEXT_LENGTH` 환경변수만으론 안 올라감.
   → **`ollama_num_ctx: 65536`을 config에 추가**해 Hermes가 로드 시 num_ctx를 명시하게 하면 해결.
   (모델을 이미 로드했다면 `ollama stop qwen2.5-coder:7b`로 언로드 후 재시도)

3. **`hermes` 명령이 `~/.local/bin`에 안 만들어질 수 있다 (권한).**
   설치 마지막에 `Permission denied`로 런처 등록 실패 → 원인은 `~/.local/bin`이 **root 소유**인 경우
   (과거 다른 CLI가 sudo로 심볼릭링크를 만들며 디렉토리를 root가 차지). 코드 자체
   (`~/.hermes/hermes-agent/venv/bin/hermes`)는 정상 설치돼 있으니, PATH에 있고 사용자가 쓸 수 있는
   디렉토리에 런처 shim을 만들면 된다:
   ```bash
   cat > /opt/homebrew/bin/hermes <<'EOF'
   #!/usr/bin/env bash
   unset PYTHONPATH; unset PYTHONHOME
   exec "$HOME/.hermes/hermes-agent/venv/bin/hermes" "$@"
   EOF
   chmod +x /opt/homebrew/bin/hermes
   ```
   (근본 해결은 `sudo chown -R $(whoami) ~/.local/bin`)

## 기대치 & 주의 ⚠️

1. **메모리는 생각보다 여유 있다** — 7b(Q4 ~4.7GB) + 64k KV캐시(q8_0 ~1.8GB) + 오버헤드 ≈ **총 8GB 안팎**.
   16GB M1 Pro에서 충분. flash attention + `OLLAMA_KV_CACHE_TYPE=q8_0`가 KV 메모리를 절반으로 줄여줌.
2. **속도는 느리다** — 7b가 64k 컨텍스트로 첫 로드될 때 응답에 1~2분 걸릴 수 있음 (M1 Pro 기준).
3. **작은 모델 = 에이전트 능력 약함** — 7b는 tool-call은 하지만 다단계 계획·도구 사용이 서툴다.
   → 메커니즘 학습용으론 충분, **성능 판단용으론 부적합**.
4. **tool-call 깨짐** — 도구 호출이 실행 안 되고 `{"name": ..., "arguments": {...}}` 텍스트로 찍히면
   로컬 모델의 tool-parsing 실패 신호. qwen2.5-coder는 지원하지만, 문제 시 서버 옵션 점검
   (llama.cpp `--jinja`, vLLM `--enable-auto-tool-choice --tool-call-parser hermes`).

## 참고: context_length ≠ max_tokens

- **context_length**: 총 컨텍스트 윈도우 (입력 + 출력)
- **max_tokens**: 단일 응답의 최대 길이

## 다음 단계

로컬 연동이 확인되면 `hermes model`로 provider를 OpenRouter/Anthropic으로 갈아끼워
"같은 과제, 다른 두뇌"를 비교한다. → [providers.md](providers.md) 참고.
