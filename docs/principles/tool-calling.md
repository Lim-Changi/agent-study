# 원리: Tool-Calling (도구 호출)

에이전트가 "말만 하는 챗봇"과 다른 핵심. 모델이 외부 도구(파일읽기·쉘·검색 등)를 **호출**하고
그 결과를 받아 다시 추론하는 루프가 에이전트를 에이전트로 만든다.

> 이 노트는 2026-07-14 로컬 셋업에서 **실제로 tool-call이 깨지는 걸 관찰**하며 정리했다.

## 에이전트 루프에서 tool-call의 자리

```
사용자 요청
   ↓
[모델] "이건 read_file 도구가 필요해" → tool_call 생성
   ↓
[하네스] tool_call을 파싱 → 실제 도구 실행 → 결과 획득
   ↓
[모델] 도구 결과를 보고 다음 판단 (또 다른 tool_call or 최종 답변)
   ↓  (반복)
최종 답변
```

여기서 **하네스와 모델은 "tool_call을 어떤 형식으로 주고받을지" 규약이 일치해야** 한다.
이 규약이 어긋나면 루프가 첫 스텝에서 끊긴다.

## tool-call "형식"은 하나가 아니다

| 형식 | 모양 | 쓰는 곳 |
|---|---|---|
| **OpenAI 구조화** | API 응답의 `tool_calls` 필드 (JSON) | OpenAI/호환 API의 `tools` 파라미터 |
| **Hermes(XML 태그)** | `<tool_call>{"name":...,"arguments":...}</tool_call>` | Hermes 모델·Hermes Agent |
| **맨 JSON** | `{"name": ..., "arguments": ...}` (래퍼 없음) | (형식 미지정 시 모델이 임의로 뱉는 것) |

## 실측: 형식 불일치로 깨진 사례

**구성**: Hermes Agent(하네스) + `qwen2.5-coder:7b`(Ollama 로컬 모델)

요청: "이 디렉토리에 어떤 파일이 있는지 도구로 확인해줘"

모델 출력:
```
{"name": "read_file", "arguments": {"path": ".", "offset": 1, "limit": "2000"}}
```
결과: `Messages: 2 (0 tool calls)` — **도구가 한 번도 실행되지 않음.**

### 무슨 일이 벌어졌나
- Hermes Agent는 **Hermes 포맷**(`<tool_call>` 태그)으로 tool-call이 오길 기대한다.
- qwen2.5-coder는 그 포맷을 모르고 **맨 JSON**을 뱉었다.
- 하네스 파서가 이를 tool_call로 인식하지 못하고 → 그냥 최종 텍스트로 출력 → 루프 종료.

### 배운 것 (핵심)
1. **capability ≠ compatibility.** `ollama show`에서 qwen2.5-coder는 `tools` capability가 *있다고*
   나왔지만, 그것이 "이 하네스가 기대하는 형식으로 뱉는다"는 보장은 아니다. 모델이 tool을 "할 줄 안다"와
   "하네스와 같은 언어로 말한다"는 다른 문제다.
2. **하네스는 특정 tool-call 방언을 전제한다.** Hermes Agent는 Nous가 만든 도구라 Hermes 방언을
   기대한다. 그래서 **Hermes 방언을 네이티브로 쓰는 모델**(= Nous Hermes 모델, 또는 vLLM의
   `--tool-call-parser hermes` 같은 어댑터)을 붙여야 파싱이 된다.
3. **해결**: 로컬 모델을 `hermes3`(Nous Hermes 3, Llama 3.1 기반)로 교체.
   Hermes 방언 네이티브 + 128k 컨텍스트로 64k 요구사항도 자연 해결. → "하네스와 모델의 방언을 맞춘다".

## 해결 확인 & 2차 발견: "형식"과 "역량"은 다른 문제

`hermes3:8b`로 교체 후 같은 요청을 다시 돌린 결과:
```
Messages: 4 (1 user, 2 tool calls)   ← 도구가 실제로 2회 실행됨 (qwen: 0회)
```
→ **형식 호환 문제는 해결.** Hermes 방언을 네이티브로 쓰니 하네스가 tool_call을 파싱·실행했다.

그런데 8B는 **엉뚱한 도구를 골랐다** — 파일 목록 도구 대신 `browser_snapshot`(브라우저)을 써서
"디렉토리"를 웹페이지로 오해하고 "파일이 없다"는 틀린 결론을 냈다 (`任何` 같은 깨진 토큰도 섞임).

### 여기서 tool-call 실패는 사실 두 층위다
| 층위 | 질문 | 이번 사례 |
|---|---|---|
| **형식 호환** | 모델이 하네스가 파싱 가능한 형식으로 뱉는가? | qwen ❌ → hermes3 ✅ |
| **모델 역량** | 올바른 도구를 골라 제대로 쓰는가? | hermes3:8b 여전히 약함 ⚠️ |

형식을 고쳐도(루프가 돌아도) 모델이 작으면 **도구 선택·추론**에서 실패한다.
→ 로컬 8B는 **"에이전트 메커니즘 학습"엔 충분**하지만 **"에이전트 성능"은 별개**다.
성능을 보려면 큰 모델(클라우드 트랙: OpenRouter/Anthropic)로 같은 과제를 돌려 비교해야 한다.

### 루프 자체는 건강하다 (대조 실험)
같은 모델에 **도구가 명백한 과제**를 주니 ("쉘에서 `ls -1` 실행, 브라우저 쓰지 마"):
```
  ┊ 💻 $  ls -1   1.3s
Messages: 4 (2 tool calls)   ·   Duration: 13s
```
올바른 터미널 도구를 골라 실제 쉘 명령을 실행하고 결과 기반으로 답했다 (앞선 브라우저 삽질 1m35s와 대조).
→ **앞선 실패는 루프 고장이 아니라 모델의 도구-선택 판단력 문제.** 과제가 모호할수록 작은 모델이 헤맨다.
프롬프트로 도구를 좁혀주면(브라우저 금지 등) 8B도 정상 동작한다.

## 일반화

에이전트를 새 모델로 돌릴 때 첫 번째로 확인할 것:
> **"이 모델이 이 하네스가 기대하는 tool-call 형식으로 뱉는가?"**

- 안 맞으면: (a) 하네스가 기대하는 형식을 쓰는 모델로 교체, (b) 추론 서버에 tool-call 파서 지정
  (vLLM `--tool-call-parser hermes`, llama.cpp `--jinja`), (c) 하네스가 OpenAI-native `tools`를
  쓰도록 설정(가능한 경우) 중 하나.

## 관련 노트

- [../hermes/providers.md](../hermes/providers.md) — Hermes 고유 tool-call 태그 규약
- [../hermes/local-ollama-setup.md](../hermes/local-ollama-setup.md) — tool-call 깨짐 트러블슈팅
