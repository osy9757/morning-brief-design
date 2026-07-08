# 03. API 명세 (REST, `/api/v1`)

공통: 인증 필요 엔드포인트는 `Authorization: Bearer <JWT>`. 공개 엔드포인트는 `GET /brief/*`, `GET /stocks/*`(search·analysis·evidence 다운로드), `GET /fear-greed/*`이다. `POST /stocks/{ticker}/analyze`는 로그인 user 이상만 호출한다(미인증 401). `/portfolio/*`는 로그인 user 이상, `/llm/*`와 `/batch/*`는 admin 전용이다(role!=admin이면 403 `FORBIDDEN`). 에러 응답 통일 `{"error": {"code": "STRING_CODE", "message": "한국어 메시지"}}`. 날짜는 `YYYY-MM-DD`(KST), 시각은 ISO8601. 숫자는 JSON number(퍼센트는 소수 아닌 % 값: 1.23 = +1.23%).

## 1. 인증

`POST /auth/login` — body `{"email": "...", "password": "..."}` → 200 `{"access_token": "...", "expires_in": 604800, "role": "admin"}` (7일, JWT에도 `role` 클레임 포함). 401 `INVALID_CREDENTIALS`. 회원가입/리프레시 없음(1인 서비스, 만료 시 재로그인).

## 2. 데일리 브리핑 (메인 페이지 — 단일 호출)

`GET /brief/today` (= 최신 발행 brief: status ∈ {success, degraded}; degraded는 시장 데이터 완전·LLM 섹션만 빈 상태로 degraded_sections에 표기, failed는 제외), `GET /brief/{date}` → 200:

```json
{
  "brief_date": "2026-07-07",
  "generated_at": "2026-07-07T06:41:12+09:00",
  "degraded_sections": [],
  "regime": {
    "composite": 28.2, "phase": 2, "phase_label": "후기 사이클", "kr_regime": "RISK_ON",
    "implied_ffr": 4.25, "implied_ffr_bp_1d": -3.0,
    "factors": {"A": {"value": -0.12, "score": 0.31}, "B": {"value": 0.10, "score": 0.13},
                 "C": {"value": -0.20, "score": 0.40}, "D": {"value": 0.02, "score": 0.43}, "E": {"value": 4.1, "score": 0.21}}
  },
  "//regime_note": "composite = Σ(가중×score)×100 = 0.30·0.31+0.25·0.13+0.15·0.40+0.15·0.43+0.15·0.21 = 28.2 (산식 재계산 가능해야 함, #28). A=T10Y3M/T10Y2Y 블렌드, C=CFNAIMA3, D=PERMIT 6M, E=HY_OAS/NFCI (02 §S2.1)",
  "indices": [
    {"symbol": "^GSPC", "label": "S&P 500", "last": 6321.44, "change_pct_1d": 0.82,
     "spark_30d": [6100.2, 6113.9], "is_stale": false},
    {"symbol": "ES=F", "label": "S&P500 선물", "last": 6330.0, "change_pct_1d": 0.14,
     "spark_30d": [6108.0, 6120.5], "is_stale": false},
    {"symbol": "DGS10", "label": "미국채 10Y", "last": 4.28, "change_pct_1d": -0.70,
     "change_bp_1d": -3.0, "spark_30d": [4.31, 4.28], "is_stale": false}
  ],
  "fear_greed": {"score": 63.4, "label": "Greed",
    "factors": {"F1": 71.2, "F2": 58.0, "F3": 66.1, "F4": 60.3, "F5": 61.5},
    "history_30d": [{"date": "2026-06-08", "score": 48.1}]},
  "sectors": [
    {"etf": "XLK", "name": "기술", "ret_1d": 1.4, "ret_1w": 2.1, "ret_1m": 5.3, "ret_3m": 9.8, "ret_ytd": 14.2,
     "rs_spy_1m": 1.9, "above_ma50": true, "above_ma200": true, "vol_20d": 18.3,
     "rsi_14": 62.1, "volume_ratio": 1.31, "flow_score": 1.8,
     "fear_greed": {"score": 76.66, "label": "Extreme Greed"}, "trend": "UP"}
  ],
  "heatmap": [{"name": "기술", "etf": "XLK", "size": 32, "value": 1.4}],
  "money_flow": {
    "sector_flows": [{"etf": "XLK", "name": "기술", "flow_score": 1.8, "rs_spy_5d": 1.1, "volume_ratio": 1.31}],
    "asset_flows": [{"symbol": "SPY", "label": "미국 대형주", "ret_5d": 1.2, "ret_20d": 3.4, "volume_ratio": 1.05, "rotation_rank": 2}],
    "inflow_top": ["성장주", "크립토", "금"], "outflow_top": ["장기채", "달러", "소형주"]
  },
  "market_wrap": "…하루 총평 2~3문장 (요구 '총정리', #15)…",
  "events": [
    {"headline": "6월 CPI 둔화, 9월 인하 확률 상승", "body": "…근거 문장 끝에 각주[1][2]…",
     "category": "MACRO", "importance": 3,
     "source_urls": ["https://...", "https://..."], "related_tickers": ["TLT", "^GSPC"]}
  ],
  "earnings_calendar": [{"ticker": "NVDA", "date": "2026-07-09", "when": "AMC"}],
  "upcoming_events": [{"date": "2026-07-08", "name": "6월 CPI", "importance": 3}],
  "recommendations": {
    "sectors": [{"sector": "기술", "etf": "XLK", "score": 0.87,
                  "reason_metrics": {"rs_spy_1m": 1.9, "flow_score": 1.8, "phase_fit": 1},
                  "outlook_text": "…sector_outlook 2문장 (#5)…"}],
    "stocks": [{"ticker": "NVDA", "name": "엔비디아", "sector": "기술",
                 "conviction": 8, "verdict": "BUY", "one_liner": "…",
                 "entry_hint": {"zone_low": 178.2, "zone_high": 184.5},
                 "entry_rationale": "…LLM이 지지·이평·ATR로 판단한 근거 1~2문장(수식 폴백 시 그 표기)…"}]
  },
  "kr": {
    "indices": [{"symbol": "^KS11", "label": "KOSPI", "last": 3294.12, "change_pct_1d": 0.43, "spark_30d": [3200.0, 3294.12], "is_stale": false}],
    "regime": {"kr_regime": "RISK_ON"},
    "fear_greed": {"score": 58.4, "label": "Greed", "factors": {"F1": 62.0, "F4": 55.2, "F5": 58.0}},
    "sectors": [{"etf": "091160", "name": "반도체", "ret_1d": 1.9, "ret_1w": 2.1, "ret_1m": 5.3, "ret_3m": 9.8, "ret_ytd": 14.2,
      "rs_spy_1m": 1.9, "above_ma50": true, "above_ma200": true, "vol_20d": 18.3, "rsi_14": 62.1, "volume_ratio": 1.31,
      "flow_score": 1.8, "fear_greed": {"score": 76.66, "label": "Extreme Greed"}, "trend": "UP"}],
    "heatmap": [{"name": "반도체", "etf": "091160", "size": 10, "value": 1.9}],
    "money_flow": {"sector_flows": [{"etf": "091160", "name": "반도체", "flow_score": 1.8, "rs_spy_5d": 1.1, "volume_ratio": 1.31}],
      "asset_flows": [{"symbol": "069500", "label": "대형주", "ret_5d": 1.2, "ret_20d": 3.4, "volume_ratio": 1.05, "rotation_rank": 1}],
      "inflow_top": ["대형주", "성장/중소형", "금"], "outflow_top": ["채권", "달러", "금"]},
    "recommendations": {"sectors": [], "stocks": []},
    "events": [],
    "market_wrap": ""
  }
}
```

주: `generated_at`은 batch_runs.finished_at 조인, `fear_greed.history_30d`는 fear_greed_history 조립 필드(daily_briefs 저장 컬럼 아님, 01 #20). US 홈 데이터는 기존처럼 최상위 필드에 유지하고, 국장 홈 토글용 데이터는 nullable `kr` 객체로 추가한다. `//`로 시작하는 키는 문서용 주석이며 실제 응답에 없음.
404 `BRIEF_NOT_FOUND` (아직 첫 배치 전).

## 3. 브리핑 보조

- `GET /brief/dates?limit=30` → `{"dates": ["2026-07-07", "..."]}` (과거 브리핑 네비게이션)
- `GET /fear-greed/history?days=90` → `[{"date", "score", "label"}]`

## 4. 종목

- `GET /stocks/search?q=NVID` → `[{"ticker": "NVDA", "name": "NVIDIA", "market": "US", "sector": "기술"}]` (ticker/name/search_alias ILIKE, 최대 10, 비로그인 공개)
- `GET /stocks/search?q=엔비디아` → `[{"ticker": "NVDA", "name": "NVIDIA", "market": "US", "sector": "기술"}]` (영문 표시명 종목의 한글 별칭 검색)
- `GET /stocks/search?q=삼성` → `[{"ticker": "005930", "name": "삼성전자", "market": "KR", "sector": "기술"}]`
- `GET /stocks/search?q=카카오` → `[{"ticker": "035720", "name": "카카오", "market": "KR", "sector": "커뮤니케이션"}]` (`scripts/load_kr_stocks.py`로 적재한 KRX 전종목도 동일 검색)
- `GET /stocks/{ticker}/analysis?date=` (date 생략 = 최신) → 200:

```json
{
  "ticker": "NVDA", "name": "NVIDIA", "as_of": "2026-07-07", "source": "batch",
  "price": {"last": 181.4, "change_pct_1d": 2.1},
  "score": {"total": 74.5, "signal": "BUY",
            "breakdown": {"momentum": 82, "value": 41, "quality_growth": 88, "low_vol": 45, "macro_fit": 80, "risk_event": 90}},
  "committee": {
    "bull": "…논거 3개 문단…", "bear": "…", "risk": "…하락 시나리오…",
    "pm": {"verdict": "BUY", "conviction": 8, "summary": "…"}
  },
  "tabs": {
    "fundamental": {"metrics": {"per": 42.1, "fwd_per": 33.0, "pbr": 21.2, "dividend_yield": 0.03},
                     "quarters": [{"period": "2026Q1", "revenue": 44100000000, "net_income": 22100000000, "eps": 0.89}],
                     "llm_text": "…해설…"},
    "governance": {"grade": "LOW", "items": [{"date": "2026-07-01", "title": "…", "tag": "소송", "url": "…"}], "llm_text": "…"},
    "capability": {"metrics": {"revenue_yoy": 62.1, "eps_yoy": 71.3, "roe": 88.2, "op_margin_trend": [61.1, 62.4, 64.0, 64.8], "rnd_ratio": 9.1}, "llm_text": "…"},
    "consensus": {"earnings_surprises": [{"period": "2026-04-30", "actual": 0.89, "estimate": 0.82, "surprise_pct": 8.54}],
                  "recommendation": {"period": "2026-07-01", "strong_buy": 5, "buy": 4, "hold": 3, "sell": 0, "strong_sell": 0, "buy_total": 9, "analyst_total": 12},
                  "price_target": {"mean": 210.0, "high": 250.0, "low": 170.0, "upside_pct": 15.77},
                  "llm_text": "…"},
    "technical": {"metrics": {"rsi_14": 63.2, "ma50_gap_pct": 4.1, "ma200_gap_pct": 18.9, "from_52w_high_pct": -3.2, "vol_60d": 34.1},
                   "series_1y": [{"date": "2025-07-07", "close": 120.1, "ma50": 118.2, "ma200": 109.4}], "llm_text": "…"},
    "macro": {"phase": 2, "macro_fit": 80, "sector_rs_1m": 1.9, "sector_flow": 1.8, "llm_text": "…"}
  },
  "evidence": [{"filename": "factor_inputs.json", "bytes": 18211,
                 "download_url": "/api/v1/evidence/stock/2026-07-07%2FNVDA/factor_inputs.json"}]
}
```

GET은 저장된 최신 성공 분석을 그대로 반환하며 재수집·재계산을 수행하지 않는다. 응답의 `source`는 `batch|on_demand`이다.

404 `ANALYSIS_NOT_FOUND` — 분석 대상이 아니었던 종목이면 로그인 후 `POST /stocks/{ticker}/analyze` → 200(성공 시 위 분석 payload + `refresh`) 후 GET 재조회. **온디맨드 경로(재감사 M4)**: 대상 티커의 가격을 즉석 수집(S1) → S2.6 팩터·탭 metrics·거버넌스 등급 즉석 계산(단일 종목이라 z-score는 최신 유니버스 분포 기준) → committee 1회 + stock_report → stock_analyses 저장. 미국/기타 종목은 yfinance 조회 실패 시, KR 6자리 종목은 FinanceDataReader(`DataReader(ticker)`) 또는 yfinance `.KS`/`.KQ` 폴백 조회 실패 시 404 `STOCK_NOT_FOUND`. `source='krx'` KR 종목은 재무·뉴스·공시가 없을 수 있으므로 해당 탭은 빈 배열/null metric을 허용하고 분석은 저장한다.

`POST /stocks/{ticker}/analyze` 재수집 실행 조건은 (1) 마지막 성공 `updated_at`이 `ANALYSIS_STALE_HOURS`(기본 6)보다 오래됨, (2) 마지막 `last_attempt_at`이 `ANALYSIS_TRIGGER_DEBOUNCE_MIN`(기본 5)보다 오래됨을 모두 만족해야 한다. 데이터가 신선하면 재계산 없이 기존 공유 결과와 `refresh.status="fresh"`를 반환한다. 버튼 디바운스 안이면 429 `ANALYSIS_TRIGGER_DEBOUNCED`. 재계산은 먼저 `last_attempt_at=now`를 기록한 뒤 실행하고, 성공 시 `updated_at`과 `source='on_demand'`를 갱신한다. 실패 시 `updated_at`은 유지하지/올리지 않고 `last_attempt_at`만 남긴다. 같은 종목 동시 트리거는 행 잠금으로 한 번만 실행한다.

## 5. Evidence (요구 7)

`GET /evidence/{scope}/{ref_key}/{filename}` → 파일 스트림 (Content-Disposition: attachment). scope ∈ brief|stock|signal. 404 `EVIDENCE_NOT_FOUND`.

## 6. 포트폴리오 (요구 8)

- `GET /portfolio` → `{"id": 1, "name": "기본", "base_currency": "KRW", "total_value_krw": 84210000, "total_pnl_pct": 12.4, "value_history_30d": [{"date": "2026-06-08", "total_value_krw": 79800000}], "positions": [아래 항목]}` (value_history_30d는 portfolio_snapshots 최근 30행, #37)
- `POST /portfolio/positions` — body `{"ticker": "NVDA", "quantity": 30, "avg_cost": 152.3, "currency": "USD", "target_weight": 0.2, "stop_loss_price": null, "stop_loss_pct": 0.15, "take_profit_rule": null, "review_interval": null, "thesis": "AI 캐펙스"}` → 201. 중복 ticker 409 `POSITION_EXISTS`. **유니버스 밖 ticker(#11)**: stocks에 없으면 yfinance `Ticker.info`로 market/name/sector/currency를 채워 자동 등록(source='yfinance_auto') 후 포지션 생성. yfinance 조회 실패 시 404 `STOCK_NOT_FOUND`. 자동 등록 종목은 다음 배치부터 S1 수집·S2.6 팩터 대상.
- `PUT /portfolio/positions/{id}` (동일 body 부분 갱신) / `DELETE /portfolio/positions/{id}` → 204
- `GET /portfolio/signals?date=` → 포지션별 장전 시그널 (보유 종목 한정, 신규 매수는 brief.recommendations, #4):

```json
[{
  "position_id": 3, "ticker": "NVDA", "quantity": 30, "avg_cost": 152.3,
  "last": 181.4, "pnl_pct": 19.1, "weight": 0.24, "target_weight": 0.20,
  "after_hours_change_pct": 1.2,
  "signal": {"action": "PYRAMID", "entry_zone": [178.2, 184.5], "entry_rationale": "…LLM 진입 판단 근거(또는 수식 폴백)…", "stop_suggest": 161.0,
              "drift": 0.04,
              "reasons": ["R4:pnl+19%>=10%", "R4:momentum82>=70", "R4:신고가돌파", "INFO:EARNINGS_D-2"],
              "llm_text": "…3~4문장 해설…"},
  "analysis_link": "/stocks/NVDA"
}]
```

action ∈ WAIT|SELL|PYRAMID|AVERAGE_DOWN|REBALANCE. REBALANCE 시 drift(=weight−target_weight)가 사유. pnl_pct·after_hours_change_pct는 % 값(엔진 소수 ×100, #32).
- `GET /alerts?status=new` / `PUT /alerts/{id}/read`

환율: 자산가치 합산 시 KRW=X 최신 종가로 USD→KRW 환산(brief 데이터 재사용).

## 7. LLM 관리 (요구 2)

전 엔드포인트 admin 전용. 무인증 401, role!='admin' 403 `FORBIDDEN`.

- `GET /llm/profiles` → `[{"id", "name", "provider", "model", "api_key_env", "temperature", "max_tokens", "enabled"}]`
- `POST /llm/profiles` / `PUT /llm/profiles/{id}` / `DELETE /llm/profiles/{id}` (라우팅에서 참조 중이면 409 `PROFILE_IN_USE`)
- `GET /llm/routing` → `[{"task": "event_digest", "profile": "claude-main", "fallback_profile": "deepseek-cheap"}]` (6 task — profile_test 제외, #21)
- `PUT /llm/routing/{task}` — body `{"profile_id": 2, "fallback_profile_id": 1}`
- `POST /llm/profiles/{id}/test` → 소형 프롬프트 1회 호출 `{"ok": true, "latency_ms": 812, "sample": "…"}`
- `GET /llm/usage?days=30` → task별 호출수/토큰 합계 (llm_call_logs 집계)

## 8. 배치

전 엔드포인트 admin 전용. 무인증 401, role!='admin' 403 `FORBIDDEN`.

- `POST /batch/run` — body `{"brief_date": null}` (null=오늘) → 202 `{"run_id": 41}`. 실행 중이면 409 `BATCH_RUNNING`.
- `GET /batch/runs/{id}` → `{"status": "running", "stage_status": {"S1": "ok", "S2": "running"}}`
- `GET /batch/runs?limit=14` → 최근 실행 이력

## 프론트 페이지 ↔ API 매핑

| 페이지 | 호출 |
|---|---|
| 메인 `/` | GET /brief/today (1회로 전부) |
| 종목 `/stocks/[ticker]` | GET /stocks/{t}/analysis + evidence 링크 |
| 포트폴리오 `/portfolio` | GET /portfolio + GET /portfolio/signals |
| 설정 `/settings` | /llm/* + /batch/* |
