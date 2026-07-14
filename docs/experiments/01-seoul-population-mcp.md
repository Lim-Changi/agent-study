# 실험 01 — 서울 도시인구 MCP 분석 에이전트 (2026-07-14)

**질문**: 기존 로컬 MCP(서울 실시간 도시인구데이터)를 Hermes Agent에 붙여, 로컬 8B 모델이
MCP 도구를 호출해 다단계 데이터 분석을 해낼 수 있는가?

## 구성

```
Hermes Agent (hermes3:8b 로컬)
   │  MCP (stdio)
   ▼
firestore-seoul (node 서버) → Firestore (실시간_서울_도시인구데이터)
```

- 등록: `hermes mcp add firestore-seoul --command node --args .../server.js`
  - ⚠️ 비대화형 실행 시 "Enable all tools?" 프롬프트에서 멈춤 → `printf 'y\n' | hermes mcp add ...`로 해결
- 검증: `hermes mcp test firestore-seoul` → **✓ Connected (97ms), 6 tools** 정상
- 도구: `firestore_schema / latest / query / get_document / list_documents / list_collections`

## 데이터

- 컬렉션 `실시간_서울_도시인구데이터` — **장소 × 시각 시계열** (5분 간격 스냅샷)
- 문서 ID 예: `DDP-동대문디자인플라자-_20250625_1333`
- 인구현황 필드(`LIVE_PPLTN_STTS`): `AREA_CONGEST_LVL`(혼잡도 4단계 문자열),
  `AREA_PPLTN_MIN/MAX`(실시간 인구 추정), 성별·연령대 비율, 상주/비상주 비율, `PPLTN_TIME`

## 결과 — 8B의 위험한 실패

질문: "DDP의 6월 25일 시간대별 혼잡도·인구 추이를 분석하고 가장 붐빈 시간대를 알려줘"

`hermes3:8b` 응답: **`Messages: 2 (0 tool calls)`** — MCP 도구를 **한 번도 호출하지 않음.**
대신 스키마와 분석 결과를 **환각으로 지어냄.**

### 8B 환각 vs 실제 데이터 (같은 MCP로 직접 조회)

| 항목 | 🤖 8B가 지어낸 것 | ✅ 실제 데이터 |
|---|---|---|
| 혼잡도 필드 | `AREA_CONGEST_LVL` = **0.95** (숫자) | `AREA_CONGEST_LVL` = **"보통"** (4단계 문자열) |
| 인구 | **12,345 ~ 78,901명** | **5,500 ~ 6,000명** |
| 스키마 | `DDP_DAY`, `DDP_VISITOR` (존재 안 함) | `AREA_PPLTN_MIN/MAX`, `PPLTN_TIME`, 성별·연령·상주비율 |
| 실제 예시 | (없음) | 13:10 기준 남43.9%/여56.1%, 비상주 82.1%, "크게 붐비지 않음" |

8B는 **필드 이름도 값도 전부 환각**했다.

## 배운 것

1. **파이프라인(MCP↔Hermes)은 정상.** 같은 도구로 실제 데이터를 정확히 꺼낼 수 있었으니,
   연결·데이터엔 문제가 없다. 병목은 순전히 **모델 역량**.
2. **"도구 사용 가능 ≠ 도구 사용."** 도구가 연결돼 있어도 작은 모델은 호출하지 않을 수 있다.
3. **가장 위험한 실패 모드**: 도구를 안 부르고 **그럴듯한 가짜 데이터를 환각으로 생성**.
   `ls` 같은 단순 1스텝 도구는 잘했지만(실험 참고: [../principles/tool-calling.md]),
   MCP 다단계(스키마 확인 → 쿼리 → 집계 → 요약)는 8B의 역량 천장을 넘었다.
4. **결론**: 16GB 로컬 8B로는 MCP 다단계 분석을 신뢰성 있게 수행하지 못한다.
   실제 작동하는 분석 에이전트에는 더 큰(클라우드) 모델이 필요하다.

## 다음 (보류)

- [ ] **클라우드 트랙 비교**: 같은 MCP·같은 질문을 Hermes + 클라우드 모델(OpenRouter 또는
  claude-code OAuth)로 재실행 → "같은 하네스, 다른 두뇌" 성능 대조 완성.

## 재현 메모

```bash
# 로컬 스택 기동
OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0 OLLAMA_CONTEXT_LENGTH=65536 ollama serve   # (백그라운드)
# MCP 등록(최초 1회)
printf 'y\n' | hermes mcp add firestore-seoul --command node --args ~/.claude/mcp-servers/firestore-seoul/server.js
# 분석 실행
hermes chat -q "<분석 질문>"
```
