# 04. LLM 명세 — 변수화 구조 + task 프롬프트 (요구 2·4·7·8)

task 총 7종 중 **라우팅 대상은 6종**(event_digest, sector_outlook, committee, stock_report, portfolio_coach, news_classify). profile_test는 라우팅 테이블 밖 — 설정 페이지에서 선택한 프로파일 id를 직접 호출한다(#21).

## 1. 원칙 (vibe-investing 철학 + 진입 타점 예외)

1. **LLM은 원칙적으로 숫자를 만들지 않는다** — 해설·판정 수치는 엔진 산출 JSON에서 인용만 한다(출력에 입력에 없는 수치가 있으면 검증 실패). **예외(사용자 결정): 진입 타점(entry_hint/entry_zone)은 LLM이 기술 데이터로 판단해 산출한다.** 단 할루시네이션을 막기 위해 (a) 엔진이 계산한 기술 지표 전체 + 참고구간 `ref_buy_zone`을 입력으로 주고, (b) 산출 가격이 §5의 범위 가드를 통과해야 하며, (c) 실패/키없음 시 엔진 `ref_buy_zone`(수식)으로 폴백한다. **매수/대기/매도/불타기/물타기/리밸런싱 액션 판정(R0~R7)은 LLM이 아니라 규칙**으로 유지(리스크 안전).
2. **프로바이더는 변수다.** 코드에는 모델명 하드코딩 금지 — 호출 직전 `llm_task_routing` 조회.
3. **모든 출력은 JSON 스키마 검증** (pydantic). 실패 시 에러 메시지를 붙여 1회 재시도 → 재실패 시 fallback 프로파일로 1회 → 그래도 실패면 해당 섹션 degraded.
4. 모든 호출은 llm_call_logs에 기록. 종목 분석 호출의 입출력 원문은 evidence `llm_io.json`으로 동결.

## 2. 게이트웨이 구조

```python
# llm/gateway.py 핵심 시그니처 (구현 고정)
async def call(task: str, payload: dict, out_schema: type[BaseModel]) -> BaseModel:
    route = await get_routing(task)              # llm_task_routing
    for profile in [route.profile, route.fallback]:
        rsp = await litellm.acompletion(
            model=f"{profile.provider}/{profile.model}",
            api_key=os.environ[profile.api_key_env],
            base_url=profile.base_url,
            temperature=profile.temperature, max_tokens=profile.max_tokens,
            messages=[{"role": "system", "content": SYSTEM[task]},
                      {"role": "user", "content": render(task, payload)}],
            response_format={"type": "json_object"})
        # → out_schema.model_validate_json, 실패 시 오류 첨부 재시도 1회
```

### 시드 프로파일 4종 + 전역 엔진 라우팅 (초기값 — 설정 페이지에서 자유 변경)

| name | provider/model | 용도 |
|---|---|---|
| claude-main | anthropic/claude-sonnet-5 | 전역 엔진 Claude |
| claude-deep | anthropic/claude-opus-4-8 | 위원회(주 1회 심층용 수동 전환 대비) |
| deepseek-cheap | deepseek/deepseek-chat | 대량 텍스트 요약 |
| codex | openai/gpt-5-codex | 전역 엔진 Codex. 설정 화면에서 model 값을 자유 입력으로 수정 가능 |

전역 엔진 상태는 `app_settings`의 `key='llm_engine'`, `value='claude'|'codex'` 1행에 저장한다. 기본값은 `claude`다.
`PUT /admin/llm-engine`은 아래 6개 task 전체를 선택 엔진 프로파일로, fallback을 상대 엔진 프로파일로 일괄 UPDATE한다.

| task | 초기 profile | fallback | 호출 시점 |
|---|---|---|---|
| event_digest | claude-main | codex | S3, 1회/일 |
| sector_outlook | claude-main | codex | S4, 1회/일 |
| committee | claude-main | codex | S4·S5, 종목당 1회 |
| stock_report | claude-main | codex | S5, 종목당 1회 |
| portfolio_coach | claude-main | codex | S6, 포지션당 1회 |
| news_classify | claude-main | codex | S1.5 전처리(S2·S3 이전, 60건 1배치) |

라우팅 테이블(llm_task_routing)에는 위 6종만 시드된다. profile_test는 라우팅을 거치지 않고 설정 페이지가 지정한 프로파일 id로 직접 호출(#21).

## 3. 공통 시스템 프롬프트 (전 task 앞부분에 고정 삽입)

```text
너는 개인 투자자 1인을 위한 애널리스트다. 규칙:
1. 제공된 JSON의 숫자만 인용한다. 새로운 수치·가격·날짜를 만들어내지 않는다.
2. 결론을 먼저, 근거를 뒤에. 한국어로 쓴다. 존댓말 대신 간결한 평서문.
3. 불확실하면 불확실하다고 쓴다. 확신 없는 단정 금지.
4. 출력은 지정된 JSON 스키마만. 다른 텍스트 금지.
```

## 4. Task별 사용자 프롬프트 원문 + 출력 스키마

### 4.1 event_digest (S3)

```text
아래는 오늘 브리핑 입력이다.
[매크로 스냅샷] {regime JSON: composite, phase, factors 전일 대비 변화}
[지수 변동] {indices 중 |1d|>1.0% 항목}
[뉴스 목록] {[{id, title, summary, source, url, published_at}] 최대 60건}
[보유 종목] {티커 목록}

[예정 일정] {upcoming_events 향후 7일, earnings_calendar 향후 14일}

임무: 오늘 아침 투자자가 반드시 알아야 할 이벤트 4~8개를 골라 카드로 만들고, 하루 총평(market_wrap)을 작성하라.
- 같은 사건의 중복 기사는 1장으로 묶는다.
- headline은 40자 이내, body는 2~3문장 (숫자는 입력에서만 인용).
- **body의 근거 문장 끝에는 source_urls 배열 순서 기반 각주 [1][2]를 붙여라(Perplexity식, #35). [n]은 그 카드 source_urls의 1-based 인덱스 범위 내여야 한다.**
- importance: 3=시장 전체 방향성, 2=섹터/자산군 영향, 1=참고.
- source_urls에는 근거 기사 url을 그대로 복사한다 (목록에 없는 url 금지).
- 보유 종목 관련 이벤트가 있으면 우선 포함하고 related_tickers에 표기.
- market_wrap: 오늘 시장을 2~3문장으로 총정리(요구 '총정리', #15). 지수·레짐·자금흐름 입력 숫자만 인용.
```

출력: `{"market_wrap": str, "cards": [{"headline": str, "body": str, "category": "MACRO|EARNINGS|GEOPOLITICS|SECTOR|CRYPTO|KR", "importance": 1|2|3, "source_urls": [str], "related_tickers": [str]}]}`

### 4.2 stock_report (S5 — 탭 6개 해설 일괄 생성)

```text
종목 {ticker} {name}의 오늘자 분석 데이터다.
[점수] {score.breakdown + total + signal}
[펀더멘탈] {fundamental.metrics + quarters 8분기}
[거버넌스] {governance.items + grade}
[케파빌리티] {capability.metrics}
[테크니컬] {technical.metrics}
[매크로] {macro: phase, macro_fit, 섹터 rs/flow}
[실적·컨센서스] {consensus.earnings_surprises + recommendation + price_target}

임무: 각 탭에 들어갈 해설을 작성하라. 탭당 3~5문장.
- fundamental: 밸류에이션이 비싼가 싼가, 어느 분기 숫자가 그 판단의 근거인가.
- governance: 리스크 항목이 있으면 각각 무엇이 문제인지, 없으면 "특이 리스크 없음"과 그 판단 범위(조회 기간)를 명시.
- capability: 이 회사가 돈을 더 잘 벌게 되는 중인가 — 성장률·마진 추이로 설명.
- technical: 지금 자리(이평·고점 대비·RSI)가 진입/관망 어느 쪽에 유리한가.
- macro: 현재 국면이 이 종목에 순풍인가 역풍인가.
- consensus: 입력 숫자만 인용해 실적 서프라이즈, 투자의견 분포, 목표가 상승여력을 1문장으로 설명한다. 데이터가 비어 있으면 빈 이유를 간단히 쓴다.
```

출력: `{"fundamental": str, "governance": str, "capability": str, "technical": str, "macro": str, "consensus": str}`

### 4.3 portfolio_coach (S6)

```text
보유 포지션 장전 브리핑이다.
[포지션] {ticker, quantity, avg_cost, last, pnl_pct, weight, target_weight, after_hours_change_pct}
[확정 시그널] {action, ref_entry_zone(엔진 참고구간), stop_suggest, drift, reasons(룰 ID와 트리거 값)}
[기술 데이터] {last, ma20/50/200, ATR14, high_60d, swing_low_60d, rsi_14, low_252d, high_252d}
[종목 점수] {score.breakdown}
[시장] {phase, fear_greed.score}

임무: 이 포지션에 대한 3~4문장 코칭 + 진입 타점 판단을 작성하라.
- 첫 문장은 액션 요약 ("오늘은 {action}" 형태로 시작). 액션(매수/대기/매도/불타기/물타기/리밸런싱)은 규칙이 정한 것이므로 바꾸지 마라.
- **진입 타점(entry_zone) 판단**: action이 실행 대상(PYRAMID·AVERAGE_DOWN·BUY성 REBALANCE)이면, ref_entry_zone을 참고하되 기술 데이터로 실제 진입 구간을 판단해 {low, high}로 제시(지지/저항·이평·ATR 근거). action이 SELL·WAIT면 entry_zone=null. 제시 값은 last 대비 ±25% 및 [low_252d, high_252d] 내.
- 시그널의 손절 숫자는 그대로 인용, reasons 트리거를 자연어로. |after_hours_change_pct|>3%면 시간외 급변을 한 문장(#34).
- 시그널의 약점(예: 물타기 후 추가 하락 리스크)을 한 문장 언급.
```

출력: `{"text": str, "entry_zone": [number, number] | null, "entry_rationale": str | null}`

### 4.4 committee (S4·S5 — 단일 호출 멀티롤 토론, vibe-investing §1 축약판)

```text
종목 {ticker}에 대해 4명의 위원이 되어 순서대로 의견을 내라.
[데이터] {score.breakdown, technical.metrics(last·ma20/50/200·ATR14·high_60d·swing_low_60d·rsi_14·from_52w_high_pct·low_252d·high_252d), ref_buy_zone(엔진 참고구간), fundamental.metrics, capability.metrics, governance.grade+items, macro fit, 최근 뉴스 제목 10개}

1) BULL: 매수 논거 3개 (각각 근거 숫자 인용).
2) BEAR: 반대 논거 3개.
3) RISK: 최악 시나리오 1개와 그 트리거 조건.
4) PM: 위 토론을 심판해 최종 판정.
   - verdict: STRONG_BUY|BUY|HOLD|SELL|AVOID
   - conviction: 0~10 (Bull·Bear 논거의 질로 판단, 데이터 결측이 많으면 감점)
   - summary: 2문장.
one_liner: 추천 카드용 한 줄 (25자 이내).
5) ENTRY(진입 타점 판단): 위 기술 데이터를 근거로 실제 진입할 가격 구간을 판단하라.
   - entry_hint: {zone_low, zone_high} — ref_buy_zone을 참고하되 지지/저항·이평·ATR·최근 흐름을 종합해 조정 가능. 두 값은 last 대비 ±25% 및 [low_252d, high_252d] 범위 내여야 함.
   - entry_rationale: 왜 그 구간인지 1~2문장(어느 지표를 근거로).
```

출력: `{"bull": str, "bear": str, "risk": str, "pm": {"verdict": str, "conviction": int, "summary": str}, "one_liner": str, "entry_hint": {"zone_low": number, "zone_high": number}, "entry_rationale": str}`

※ 진입 타점 하이브리드(사용자 결정): entry_hint는 이제 LLM이 데이터로 판단한다. §5 범위 가드 위반 또는 degraded 시 엔진 ref_buy_zone(수식)으로 폴백. 액션 verdict는 그대로 LLM/규칙 판정이되 매수 실행존만 이 방식.

### 4.5 sector_outlook (S4)

```text
오늘의 섹터 랭킹 결과다.
[상위 3 섹터] {sector, rank_score, rs_spy_1m, flow_score, phase_fit, trend}
[시장 국면] {phase, composite, fear_greed}

임무: 각 섹터가 왜 유망한지 2문장씩. rank_score 구성요소 숫자를 인용해 설명하라.
```

출력: `{"outlooks": [{"sector": str, "outlook_text": str}]}` → 파이프라인이 recommendations.sectors[].outlook_text에 매핑(#5, 키 이름 일치).

### 4.6 news_classify (S1.5 전처리 — S2.6·S5·S3가 소비)

```text
뉴스 60건의 제목·요약이다. 각 건을 분류하고 감성을 매겨라.
{[{id, title, summary, stock_ticker(있으면)}]}
category: MACRO|EARNINGS|GEOPOLITICS|SECTOR|CRYPTO|KR|NOISE
NOISE = 시황과 무관(가십·광고성). relevance: 0~2.
sentiment: -1(매우 부정)~+1(매우 긍정), 제목·요약 톤 기준.
```

출력: `{"items": [{"id": int, "category": str, "relevance": int, "sentiment": number}]}` → sentiment는 news.sentiment에 저장(#7, S2.6 risk_event·S5 거버넌스 입력원). relevance 0 및 NOISE는 event_digest 입력에서 제외.

### 4.7 profile_test

`"1+1을 계산하고 {\"ok\": true, \"answer\": 2} 형태로만 답하라."` — 설정 페이지 연결 테스트용.

## 5. 검증·비용 가드

- 숫자 검증(해설·판정 텍스트): 출력 텍스트에서 정규식 `[0-9]+(\.[0-9]+)?`로 수치 추출 → 입력 payload 직렬화 문자열에 존재하지 않는 3자리 이상 수치가 있으면 재시도 사유에 명시. (반올림 허용: 소수 1자리 반올림 값도 허용 집합에 포함. event_digest body의 각주 `[n]`은 수치 검증에서 제외하되, n이 해당 카드 source_urls 인덱스 범위를 벗어나면 카드 폐기, #35)
- **진입 타점 범위 가드(하이브리드)**: committee.entry_hint·portfolio_coach.entry_zone의 zone_low·zone_high는 (a) `zone_low ≤ zone_high`, (b) 둘 다 양수, (c) `[low_252d×0.9, high_252d×1.1]` 범위 내, (d) `last`의 ±25% 내를 만족해야 한다. 위반 시 1회 재시도 → 그래도 위반이거나 degraded면 **엔진 ref_buy_zone(수식)으로 폴백**하고 entry_rationale에 "수식 폴백"을 표기. (진입 타점은 숫자 생성이 허용된 예외이므로 위 '숫자 검증'의 3자리 규칙 대상에서 zone 값은 제외.)
- 하루 호출 상한: task별 상한 상수 — event_digest 2, sector_outlook 2, committee 30(S4 게시 후보 + S5 보유-only + 온디맨드 합. S5는 S4 committee를 재사용하므로 중복 계산 없음), stock_report 30, portfolio_coach 30, news_classify 4. 상한 초과 시: committee/stock_report는 stock_analyses NOT NULL 충족 위해 skip 대신 마지막 정상 프로파일로 축약 호출(빈 값 금지), 그 외 task는 skip + degraded.
- 타임아웃: 호출당 90s.
