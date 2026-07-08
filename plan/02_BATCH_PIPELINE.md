# 02. 배치 파이프라인 — 매일 06:30 KST (요구 1)

트리거: APScheduler cron `30 6 * * *` (Asia/Seoul, env `BATCH_CRON`). 미국 정규장 마감(서머타임 05:00 KST / 표준시 06:00 KST) 이후이므로 항상 전일 마감 데이터가 확정된 상태다.
`brief_date` = 실행일(KST). 주말/미국 휴장일에도 실행하되 "마지막 거래일 기준" 데이터로 생성하고 `indices[].is_stale=true` 표기.

수동 트리거: `POST /api/v1/batch/run` (03 §7). 동일 brief_date 재실행 시 산출물 전부 UPSERT(멱등).

## 스테이지 DAG (순차 — #23)

```text
S1 수집
  └► S1.5 news_classify (LLM: 뉴스 60건 분류·감성 → news.sentiment 채움)
        └► S2 시장지표 계산 (S2.1~S2.6, LLM 미사용 결정적 계산; S2.6 risk_event가 sentiment 소비)
              │  ※ S2.6에서 분석 대상 유니버스 전체의 팩터 + 탭 metrics + 거버넌스 등급 + ma20/ATR14/진입존까지 선계산
              ├► S3 이벤트 다이제스트(LLM)   ← S2와 병렬 가능하나 S1.5(분류된 뉴스) 이후
              └► S4 섹터/종목 추천(LLM committee, 입력 지표는 S2.6 선계산분 사용)
                    └► S5 종목 상세(보유 ∪ S4 게시 종목: LLM 텍스트 + S4 미보유 종목 committee, evidence 동결)
                          └► S6 포트폴리오 시그널(규칙) ─(해설만 LLM) + 스냅샷 UPSERT
                                └► S7 publish (daily_briefs UPSERT + evidence 동결 + Telegram)
                                      └► S8 정리 (retention, best-effort)
```

순서 규칙(#23): committee(S4)·거버넌스 등급 등 "지표"는 전부 S2.6에서 엔진이 선계산하고, S4/S5의 LLM은 그 값을 인용만 한다(계산↔LLM 순환 제거). news.sentiment는 **S1.5(news_classify)에서 채워져 S2.6·S5가 소비**한다(S3에서 채우지 않음 — S2.6 시점 NULL 방지). S5는 S4 게시 확정 후, S6은 S5 tabs 생성 후 실행.

실패 정책(전 스테이지 공통): 스테이지 내 외부 호출은 재시도 2회(지수 백오프 1s/4s). 스테이지 실패 시 파이프라인은 계속 진행하고 `batch_runs.stage_status`에 사유 기록, 해당 섹션은 `daily_briefs.degraded_sections`에 추가. S1 전체 실패 시에만 run status=failed.
시간 예산: 전체 15분 이내 (S1 4분, S1.5 1분, S2 1분, S3 2분, S4 3분, S5 3분, S6+S7 1분).

---

## S1. 수집

| 대상 | 소스 | 범위 |
|---|---|---|
| 지수/ETF/종목/선물 일봉 | yfinance `download(tickers, period="2y", auto_adjust=False)` | **배치 핵심 유니버스**(`source='seed'` ∪ 보유 포지션 ∪ 최근 S4 게시 종목, 선물 ES=F NQ=F YM=F ZQ=F 포함) → prices_daily UPSERT. `source='krx'` 검색용 bulk 종목은 보유/추천 전까지 제외 |
| 보유 종목 시간외 최종가 | yfinance `Ticker.info`의 postMarketPrice/preMarketPrice(없으면 null) | 보유 종목만 → S6 after_hours_change_pct (#34) |
| 미국 펀더멘털 | yfinance `Ticker.info` + `quarterly_financials` | 배치 핵심 유니버스 중 미국 종목만 → financials |
| 미국 실적 캘린더 | yfinance `Ticker.get_earnings_dates(limit=8)` | 보유∪추천 유니버스, 향후 14일 → daily_briefs.earnings_calendar `[{ticker,date,when:BMO|AMC}]` (#12) |
| 경제지표 캘린더 | 로컬 `data/econ_calendar_2026.json`(FOMC·CPI·PCE·고용·GDP, 연 1회 수동 갱신) | 향후 7일 → daily_briefs.upcoming_events `[{date,name,importance}]` (#13) |
| 매크로 | FRED CSV `https://fred.stlouisfed.org/graph/fredgraph.csv?id={SID}` | T10Y3M, T10Y2Y, SAHMREALTIME, BAMLH0A0HYM2, NFCI, CFNAIMA3, PERMIT, DGS10 → macro_series (USSLIND 폐지 교체, #9/#2/#38) |
| 시장/종목 뉴스(US) | Finnhub `/news?category=general` + `/company-news`(보유·추천 후보, 최근 7일) | → news(market='US', url UNIQUE로 dedupe). **sentiment는 소스 미제공** → 수집 직후 **S1.5 news_classify**(04 §4.6)가 -1~1 산출해 news.sentiment 채움 (S2.6·S5 소비, #7) |
| 국내 증시 뉴스(KR) | Google News RSS(무키) `https://news.google.com/rss/search?q={query}&hl=ko&gl=KR&ceid=KR:ko`, query=`코스피`,`코스닥`,`국내증시` | → news(market='KR', url UNIQUE dedupe, 최대 약 40건). RSS `item`의 title/link/pubDate/source를 저장하고, 네트워크·파싱 실패는 빈 리스트로 graceful 처리 |
| 국내 가격 | FinanceDataReader(무키) 우선, yfinance `.KS`/`.KQ` 폴백 | 배치 핵심 유니버스 중 KR 종목만 → prices_daily. 검색용 `source='krx'` bulk 종목은 온디맨드 분석에서 즉석 수집 |
| 국내 공시 | OpenDART `list.json` (최근 7일, 보유 국내 종목) | → disclosures, risk_tags는 제목 정규식 매핑* |

*risk_tags 정규식(고정): `전환사채|CB`→CB, `신주인수권|BW`→BW, `유상증자`→유상증자, `최대주주.*변경`→최대주주변경, `소송|가처분`→소송, `관리종목|거래정지`→관리종목.

선물 처리(#1): ES=F/NQ=F/YM=F/ZQ=F는 24시간 근접 거래라 06:30 KST 시점 최신 체결이 현물 마감 이후의 시간외 방향을 반영한다. prices_daily에 일봉으로 저장하되, S2.2 스트립에서 "현물 마감 대비 선물 등락"으로 표기(선물의 change_pct_1d = 선물 당일 최신/전일 종가 − 1).

## S2. 시장 지표 계산 (LLM 미사용 — 전부 결정적 수식)

공통 유틸(전부 값 고정, #30): `ret(t, n)` = n거래일 수익률, `ma(t, n)` = n일 단순이동평균, `z()` = 유니버스 내 z-score(1~99% winsorize 후), `clip01(x, lo, hi)` = (x-lo)/(hi-lo)를 0~1로 클립, `percentile()` = 유니버스 내 백분위(0~1).
- `ATR14` = TR의 14일 단순평균, TR = max(High−Low, |High−Close₋₁|, |Low−Close₋₁|).
- `swing_low_60d` = 당일 제외 직전 60거래일(t−60~t−1) 저가(Low)의 최솟값.
- `전고점(60d)` = 당일 제외 직전 60거래일(t−60~t−1) 고가(High)의 최댓값. (당일 제외이므로 R4 신고가 돌파 판정이 성립)
- `ma20` = ma(t, 20). 변화율/등락 단위 규약: 엔진 내부 계산은 소수(fraction), DB 저장·API 직렬화는 ×100한 % 값 (#32).

### S2.1 매크로 레짐 (5-Factor 침체 컴포지트 → 4-Phase) — vibe-investing §3·§4 기반

Factor는 각각 서로 다른 유지되는 FRED 시리즈에 의존한다(USSLIND 이중계상·폐지 해소, #9). 변화율 단위는 전부 "차분(pp) 또는 비율"을 명시하고, "N개월 전"은 월별 시계열에서 N행 전을 뜻한다(#31).

| Factor | 가중 | 시리즈 | 산식 (0~1) |
|---|---|---|---|
| A 수익률곡선 | 30% | T10Y3M, T10Y2Y | 0.5×`clip01(-x1, -0.5, 1.5)` + 0.5×`clip01(-x2, -0.5, 1.5)` (x1=T10Y3M, x2=T10Y2Y 최신값 %; 10Y-2Y 성분 복원, #38) |
| B Sahm | 25% | SAHMREALTIME | 최신값 s: `clip01(s, 0.0, 0.8)` |
| C 경기활동 | 15% | CFNAIMA3 | Chicago Fed 3개월 이동평균 활동지수 최신값 a(음수=추세 이하): `clip01(-a, -0.2, 0.8)` (a<-0.7이 침체 신호) |
| D 선행지표 | 15% | PERMIT | 건축허가 6개월 변화율 l = value_t/value_{t-6} − 1 (비율): `clip01(-l, -0.15, 0.15)` |
| E 신용 | 15% | BAMLH0A0HYM2, NFCI | 0.6×`clip01(HY_OAS, 3.0, 8.0)` + 0.4×`clip01(NFCI, -0.5, 1.0)` |

`composite = Σ(w_i × f_i) × 100`. Phase 경계(반개구간, #31): `[0,25)` → 1(확장), `[25,50)` → 2(후기사이클), `[50,70]` → 3(침체경고), `(70,100]` → 4(침체). 시리즈 결손 시 해당 Factor를 빼고 나머지에 가중치 비례 재분배(C·D 동시 결손 케이스 포함).

**시장 내재 정책금리(#14)**: `implied_ffr = 100 − ZQ=F 당일 종가`(%), `implied_ffr_bp_1d = (implied_ffr − 전일값)×100`(bp). regime JSONB에 함께 저장(05 RegimeBanner·S3 입력에 사용).

**국내 레짐(#26, 국내 종목 macro_fit용)**: KODEX200(069500)로 KR 레짐 3단계 산출 — `close>ma(200)` AND `ret(t,5)≥0` → `RISK_ON`; `close<ma(200)` AND 20일 실현변동성 상위 → `RISK_OFF`; 그 외 `DEFENSIVE`. regime JSONB에 `kr_regime` 필드로 저장. 국내 종목 macro_fit(S2.6)은 이 kr_regime을 참조(RISK_ON→momentum 백분위, RISK_OFF/DEFENSIVE→low_vol 백분위).

### S2.2 지수 스트립 (요구 5)

대상(순서 고정, 총 12항목): S&P500(^GSPC), 나스닥(^IXIC), 나스닥100(^NDX), 다우(^DJI), **S&P500 선물(ES=F), 나스닥100 선물(NQ=F)**, VIX(^VIX), KOSPI(^KS11), KOSDAQ(^KQ11), USD/KRW(KRW=X), BTC(BTC-USD), **미국채 10Y(DGS10)**. (YM=F 다우선물은 시드·수집하되 스트립 미표시 — S&P/나스닥 선물로 시간외 방향은 충분.)
각 항목 공통: `{symbol, label, last, change_pct_1d, spark_30d:[값×30], is_stale}`.

**국장 홈 토글용 지수 스트립(additive)**: `daily_briefs.kr.indices`에는 KOSPI(`^KS11`), KOSDAQ(`^KQ11`), KOSPI200(`069500` KODEX 200), KOSDAQ150(`229200` KODEX 코스닥150) 4개를 같은 shape로 저장한다. US 최상위 `indices` 12개 계약은 유지한다.

- **선물 항목(ES=F/NQ=F, #1)**: change_pct_1d = 선물 최신/선물 전일종가 − 1(%). 05 S-A에서 label에 "선물" 표기해 현물과 구분. spark_30d = prices_daily close 최근 30.
- **미국채 10Y 항목(DGS10, #2)**: `symbol="DGS10"`(stocks 유니버스에 없어 05 S-A 클릭 네비게이션은 무동작), `label="미국채 10Y"`, `last`=macro_series DGS10 value 최신값(%), `spark_30d`=macro_series value 최근 30 관측치(가격 close가 아니라 value가 스파크라인 y). 등락은 % 착색이 무의미하므로 별도 필드 `change_bp_1d = (value − 전일 value)×100`(bp)을 두고 05 S-A에서 이 항목만 "+5bp/−3bp"로 착색(다른 항목의 change_pct_1d 규칙과 분리). `change_pct_1d`도 스키마 일관성상 `(value/전일 value − 1)×100`로 채우되 렌더에는 미사용.
- **is_stale(전 항목)**: 항목의 최신 관측일 < 스트립 공통 기준 거래일(주말·휴장 시 마지막 거래일) → true. FRED DGS10은 익영업일 갱신이라 06:30 배치 시 D-1이 최신일 수 있음.

### S2.3 Fear & Greed (0~100, 5-factor 자체 산식 — 고정)

| F | 이름 | 산식 (0~100) |
|---|---|---|
| F1 | 주가 모멘텀 | ^GSPC 종가/125일MA − 1 = m → `clip01(m, -0.08, 0.08)×100` |
| F2 | 변동성(역) | ^VIX/50일MA − 1 = v → `(1 − clip01(v, -0.4, 0.4))×100` |
| F3 | 신용 수요 | HYG/IEF 20일 상대수익 c → `clip01(c, -0.05, 0.05)×100` |
| F4 | 안전자산 회피 | SPY 20일 수익 − TLT 20일 수익 = s → `clip01(s, -0.10, 0.10)×100` |
| F5 | 시장 폭 | 섹터 ETF 11종 중 종가>50일MA 비율 × 100 |

`score = mean(F1..F5)`. 라벨: ≤25 Extreme Fear, ≤45 Fear, ≤55 Neutral, ≤75 Greed, >75 Extreme Greed. → fear_greed_history UPSERT.

**KR F&G(additive)**: `daily_briefs.kr.fear_greed`는 동일 factor 구조를 쓰되 입력을 KR로 치환한다. F1=KOSPI(`^KS11`) 모멘텀, F4=KODEX 200(`069500`) 20일 수익−KR 채권 ETF(`471230`) 20일 수익, F5=KR 섹터 ETF 중 종가>50일MA 비율. KR 전용 변동성/신용 프록시가 시드로 확정되지 않은 factor는 결손으로 보고 나머지 factor 평균으로 재분배한다. // TODO(claude): KR F2 변동성·F3 신용 프록시를 공식 시드 티커로 확정할지 결정.

### S2.4 섹터 추이 (요구 3 — 중급자용 지표 세트 고정)

대상: SPDR 11종 `XLK 기술, XLF 금융, XLE 에너지, XLV 헬스케어, XLY 임의소비재, XLP 필수소비재, XLI 산업재, XLU 유틸리티, XLB 소재, XLRE 리츠, XLC 커뮤니케이션`.
각 섹터 산출(전부 표기): `ret_1d, ret_1w, ret_1m, ret_3m, ret_ytd, rs_spy_1m`(= ret_1m − SPY ret_1m), `above_ma50, above_ma200`(bool), `vol_20d`(연율화), `rsi_14, volume_ratio`(5일 평균 거래대금/60일 평균), `flow_score`(S2.5), `trend`("UP" if close>ma50>ma200, "DOWN" if close<ma50<ma200, else "MIXED").
각 섹터는 `fear_greed: {score, label}`도 산출한다. `score = clamp(0..100, 0.30×rsi_14 + 0.25×(clip01(ret_1m,-0.10,0.10)×100) + 0.20×trend_score + 0.15×(clip01(flow_score,-3,3)×100) + 0.10×(clip01(rs_spy_1m,-5,5)×100))`, `trend_score=(above_ma50&above_ma200?100 : (above_ma50||above_ma200?50:0))`. 라벨은 S2.3 시장 F&G와 동일하게 ≤25 Extreme Fear, ≤45 Fear, ≤55 Neutral, ≤75 Greed, >75 Extreme Greed.
히트맵 노드: `{name, etf, size: 시가총액 가중치(고정 상수표*), value: ret_1d}` (색은 value 파생이므로 저장 필드 아님 — 05 S-D-1이 value로 착색. 03 §2·05와 필드 일치).
*시총 가중 고정 상수(연 1회 갱신): XLK 32, XLF 13, XLV 11, XLY 11, XLC 9, XLI 8, XLP 6, XLE 3, XLU 3, XLB 2, XLRE 2.

**KR 섹터(additive)**: `daily_briefs.kr.sectors/heatmap`은 같은 지표 shape를 유지하고, 대상은 `091160 반도체, 091170 은행, 091180 자동차, 305720 2차전지, 102970 증권, 117700 건설, 266420 헬스케어, 117680 철강, 117460 에너지화학, 266410 필수소비재`다. RS 기준은 SPY 대신 KOSPI(`^KS11`)이고, heatmap size는 MVP에서 균등 10으로 둔다.

### S2.5 자금 흐름 (요구 6)

**섹터 흐름**: 섹터 i의 `flow_score = clip(z(volume_ratio_i), 0, 3) × sign(rs_spy_5d_i)` → -3~+3 (#29 부호 오류 수정: 거래대금이 평소 이하일 때 z<0을 0으로 클립해 방향 반전을 막는다). 해석: 거래대금이 평소 대비 몰릴 때만(z>0) 방향을 부여 — 몰리며 시장을 이기면 유입(+), 몰리며 지면 매도세(−), 평소 이하 거래대금이면 0(중립).
**자산군 로테이션**: 9종 심볼→라벨 고정 매핑표(03·05 동일 문자열 사용): `SPY=미국 대형주, QQQ=성장주, IWM=소형주, TLT=장기채, IEF=중기채, GLD=금, UUP=달러, HYG=하이일드, BTC-USD=크립토`. 각 `ret_5d, ret_20d, volume_ratio` → `rotation_rank` = ret_20d 내림차순 순위. 산출: `{asset_flows: [{symbol, label, ret_5d, ret_20d, volume_ratio, rotation_rank}×9], inflow_top: 상위3 label, outflow_top: 하위3 label}`.

**KR 자산군 로테이션(additive)**: `daily_briefs.kr.money_flow.asset_flows`는 `069500=대형주, 229200=성장/중소형, 471230=채권, 132030=금, 261240=달러` 5종을 같은 산식으로 계산한다.

### S2.6 종목 팩터 (분석 대상 = source='seed' ∪ 보유 종목 ∪ 최근 S4 게시 종목)

**추천 후보 유니버스 정의(#6)**: (섹터 ETF 11종 ∪ 자산군 ETF ∪ M7/AI Infra 18종 ∪ S4 매핑표 전 종목). 01 시드의 부분집합이며 z-score도 이 집합에서 계산. 이 스테이지가 S4 committee·S5 탭·S6 시그널의 지표를 **모두 선계산**한다(#23).

**KRX bulk 스코프**: `scripts/load_kr_stocks.py`가 추가한 `source='krx'` 전종목은 검색/열람 발견성만 제공한다. 매일 06:30 배치의 prices_daily/financials 수집과 S2.6 팩터 계산은 배치 핵심 유니버스로 한정하며, bulk KR 종목은 보유 포지션에 편입되거나 최근 S4 게시 종목이 된 경우에만 포함한다.

기존 기획 §3 US Score 가중 그대로: `total = 0.30×momentum + 0.20×quality_growth + 0.15×value + 0.10×low_vol + 0.15×macro_fit + 0.10×risk_event` (각 0~100).
- momentum: 0.4×z(12-1M 수익) + 0.3×z(6-1M) + 0.3×z(3-1M) → 백분위×100. detail에 RSI14, ma50/200 괴리, 52주고점 대비 % 저장
- value: −z(PER)·−z(FwdPER)·−z(PBR) 평균 백분위 (결측 필드는 제외 평균)
- quality_growth: z(ROE)+z(영업이익률)+z(매출YoY)+z(EPS YoY) 평균 백분위
- low_vol: (1−percentile(60일 변동성))×100
- macro_fit: `phase_fit_table` 고정 — 미국 종목: P1/P2 → percentile(momentum)×100, P3/P4 → percentile(low_vol)×100. 국내 종목: S2.1 kr_regime 참조(RISK_ON→momentum, else low_vol 백분위).
- risk_event: 100 − (7일 내 risk_tags 1건당 25점 차감, 미국은 뉴스 sentiment<-0.5 1건당 10점 차감, 하한 0). **sentiment 원천(#7)**: news_classify(04 §4.6)가 채운 news.sentiment 사용. 특정 종목 뉴스가 0건이면 감점 0(중립 처리).
- **detail 필수 필드(entry_hint·타점 원천 + 하이브리드 범위 가드)**: 위 지표 + `ma20, ma50, ma200, ATR14, swing_low_60d, high_60d(전고점), rsi_14, from_52w_high_pct, low_252d(52주 저가), high_252d(52주 고가)` 및 **참고 진입존** `ref_buy_zone = [ma20 − 1×ATR14, ma20 + 0.5×ATR14]`을 전부 저장 → S4·S6의 LLM 진입 판단 입력 및 04 §5 범위 가드(`[low_252d×0.9, high_252d×1.1]`, last ±25%)에 사용.
signal: total≥70→BUY, ≥55→HOLD, ≥40→REDUCE, <40→EXIT. 단 total≥55이고 52주고점 대비 -15%~-30% & close>ma200이면 DIP_BUY로 승격.

**탭 metrics·거버넌스 등급 선계산(#22·#23)**: S5의 5개 탭(fundamental/governance/capability/technical/macro) metrics와 거버넌스 등급(S5 정의)도 이 단계에서 결정적으로 계산해 두고, S5 LLM은 텍스트만 생성한다.
국내 종목은 기존 기획 §2 Domestic Score 가중(30/20/20/10/10/10) 적용.

## S3. 오늘의 이벤트 다이제스트 (요구 4 전반, LLM task=event_digest)

0. 전처리 결과 소비: news_classify는 S1.5에서 이미 실행됨(여기서 재실행 아님) — NOISE·relevance 0 뉴스는 입력에서 제외.
1. 입력(US): 최근 24h news.market='US' 시장 뉴스(정제 후) + 매크로 지표 변화(레짐 factor·implied_ffr 전일 대비) + 선물(ES=F/NQ=F) 방향 + VIX/지수 급변(|1d|>1.5%) + 보유 종목 뉴스 + `upcoming_events`(경제 캘린더 향후 7일, #13) + `earnings_calendar`(#12).
2. 입력(KR additive): 최근 24h news.market='KR' Google News RSS 국내 증시 뉴스(정제 후) + `daily_briefs.kr.regime` + `daily_briefs.kr.indices` 급변 항목 + 공통 일정. 같은 `event_digest` task를 별도 1회 호출한다.
3. LLM 호출 결과로 **이벤트 카드 4~8장 + market_wrap** 생성. US 결과는 `daily_briefs.events`와 `daily_briefs.market_wrap`에 저장하고, KR 결과는 `daily_briefs.kr.events`와 `daily_briefs.kr.market_wrap`에 저장한다. 카드: `{headline(≤40자), body(2~3문장, 근거 문장 끝에 각주 [1][2], #35), category: MACRO|EARNINGS|GEOPOLITICS|SECTOR|CRYPTO|KR, importance: 1~3, source_urls[], related_tickers[]}`. `market_wrap`: 하루 총평 2~3문장(요구 '총정리', #15) — 입력 숫자만 인용.
4. 검증: source_urls는 해당 시장 입력 뉴스 url 집합의 부분집합, body의 각주 [n]은 source_urls 인덱스(1-based) 범위 내여야 함(위반 카드 폐기, #35). 카드 0장이면 해당 시장 events degraded(market_wrap은 별도 — 산출되면 유지). LLM 키가 없으면 US/KR 이벤트 카드는 빈 값으로 degraded 처리하되 KR RSS 수집 자체는 계속 동작한다.

## S4. 섹터/종목 추천 (요구 4 후반)

1. **섹터 랭킹(결정적)**: `sector_rank_score = 0.4×percentile(rs_spy_1m) + 0.3×percentile(flow_score) + 0.3×phase_fit`. `phase_fit ∈ {0,1}`(가중 전 원값): P1/P2에서 **경기민감 섹터표(고정, #10): {XLK, XLY, XLF, XLI, XLE}** 소속이면 +1, P3/P4에서 **방어 섹터표(고정): {XLP, XLV, XLU}** 소속이면 +1, 그 외 0. 상위 3개 섹터 선정.
2. **종목 후보(결정적)**: 섹터→대표종목 고정 매핑표(전 종목은 01 시드 유니버스의 부분집합, #6 불변조건)(XLK: NVDA MSFT AAPL AVGO AMD CRM, XLF: JPM BAC GS MS BRK-B, XLV: LLY UNH JNJ ABBV MRK, XLY: AMZN TSLA HD MCD NKE, XLC: GOOGL META NFLX DIS, XLE: XOM CVX SLB, XLI: CAT GE HON UNP, XLP: COST WMT PG KO, XLU: NEE DUK SO, XLB: LIN APD, XLRE: PLD AMT)에서 섹터당 factor total 상위 3종목.
3. **위원회 판정(LLM)**: 후보 종목별 task=committee 1회 호출(04 §4.4 — Bull/Bear/Risk/PM을 단일 호출 멀티롤로): conviction 0~10, verdict STRONG_BUY~AVOID. 입력 지표는 S2.6 선계산분 사용. **conviction≥6만 게시**, 섹터당 최대 2종목.
4. **섹터 해설(LLM)**: 상위 3섹터에 task=sector_outlook 1회 호출(04 §4.5) → 섹터별 `outlook_text`.
5. **entry_hint 하이브리드(사용자 결정)**: 게시 종목의 진입존은 committee LLM이 기술 데이터로 판단한다(04 §4.4 entry_hint + entry_rationale). 엔진은 참고구간 `ref_buy_zone = [ma20−1×ATR14, ma20+0.5×ATR14]`을 입력으로 주고, LLM 산출이 §5 범위 가드를 통과하면 그 값을, 위반/재시도실패/degraded면 ref_buy_zone(수식)으로 폴백해 recommendations.stocks[].entry_hint에 채운다. entry_rationale도 함께 저장.
6. 산출: `recommendations = {sectors:[{sector, etf, score, reason_metrics, outlook_text}×3], stocks:[{ticker, name, sector, conviction, verdict, one_liner, entry_hint:{zone_low, zone_high}}]}`.

## S5. 종목 상세 분석 (요구 7, 대상: 보유 종목 ∪ S4 게시 종목)

tabs metrics·거버넌스 등급은 S2.6에서 선계산됨(#22·#23). S5는 종목별 stock_analyses 1행 생성:
- LLM task=stock_report(04 §4.2)로 6개 탭 텍스트 생성(**overview 탭 없음** — UI '종합' 탭은 top-level score/committee로 조립, #22).
- **committee 채움(재감사 H2)**: S4에서 이미 committee를 받은 종목(S4 게시 후보)은 그 결과 재사용, **S4 committee가 없는 종목(보유-only·온디맨드 대상)은 S5에서 task=committee 1회 호출**해 stock_analyses.committee(NOT NULL)를 채운다.
- tabs.fundamental: value 지표 + financials 최근 8분기 시계열
- tabs.governance: 등급(HIGH/MID/LOW) + 항목. 국내=disclosures risk_tags 목록, 미국=회사 뉴스 중 sentiment<-0.3(news_classify 산출, #7) + 8-K성 키워드(regex: `SEC|lawsuit|investigation|offering|dilution|insider`) 필터 목록 → 등급: 건수 ≥3 HIGH, 1~2 MID, 0 LOW
- tabs.capability: quality_growth 지표(매출/EPS 성장, ROE, 마진 추이 4분기) + R&D 비중(있으면)
- tabs.consensus: Finnhub 실적 서프라이즈(`/stock/earnings`) + 투자의견 분포(`/stock/recommendation`) + 목표주가(`/stock/price-target`) + `upside_pct=(targetMean/last-1)*100`; FINNHUB_KEY 미설정 또는 KR 6자리 종목은 빈 배열/객체와 degraded 안내 텍스트 허용. 숫자는 엔진이 수집·계산하고 LLM은 해설만 작성한다.
- tabs.technical: technical detail 전부(ma20/50/200·ATR14·RSI·52주고점 대비 등) + 1년 가격/MA 시리즈
- tabs.macro: 현재 Phase(국내는 kr_regime), 종목 macro_fit, 소속 섹터 rs/flow
- evidence 동결: `data/evidence/{date}/{ticker}/` 에 `prices_1y.csv, factor_inputs.json, financials.json, news_used.json, disclosures.json, llm_io.json` 저장 + evidence_files(scope='stock') 등록 (요구 7 "근거 다운로드")

## S6. 포트폴리오 장전 시그널 (요구 8 — 전부 규칙, LLM은 해설만. 보유 종목 한정, 신규 매수는 S4 recommendations, #4)

포지션별 입력: `pnl_pct = last/avg_cost − 1`(엔진 소수, 저장 시 ×100, #32), factor_scores, Phase, kr_regime, 기술값(S2.6 detail의 ma20/50/200·ATR14·swing_low_60d·high_60d), `after_hours_change_pct`(#34), 보유일수(now − positions.created_at), target_weight, weight(=포지션 평가액/총 평가액).
규칙 평가 순서(위에서 첫 매치 확정, 룰 ID를 reasons에 기록):

| ID | 조건 | action |
|---|---|---|
| R0 | ticker ∈ {SH,PSQ,VIXY,DOG} AND Phase=1 (Tier4 하드룰, #27) | SELL |
| R0.5 | ticker=VIXY AND 보유일수 ≥ 30 (콘탱고 부식, #27) | SELL |
| R1 | last ≤ stop(= stop_loss_price, 없으면 avg_cost×(1−stop_loss_pct), 없으면 max(avg_cost×0.85, swing_low_60d − 1×ATR14)) | SELL |
| R2 | total < 40 AND pnl_pct < −0.15 | SELL |
| R3 | risk_event ≤ 50 AND governance 등급=HIGH | SELL |
| R4 | pnl_pct ≥ +0.10 AND momentum ≥ 70 AND last > high_60d(전고점, 당일 제외) | PYRAMID (불타기) |
| R5 | −0.25 ≤ pnl_pct ≤ −0.10 AND total ≥ 65 AND Phase ∈ {1,2} AND last > ma200×0.97 | AVERAGE_DOWN (물타기) |
| R6 | target_weight 설정 AND |weight − target_weight| ≥ 0.05 (리밸런싱, #8) | REBALANCE |
| R7 | 그 외 | WAIT |

`drift = weight − target_weight`: target_weight가 설정된 모든 포지션에서 계산·저장(drift 필드), REBALANCE 판정(R6)에만 사용. target_weight 미설정 시 drift=NULL. 정보성 태그(액션 불변, reasons에 부기): 실적 D-2 이내면 `INFO:EARNINGS_D-n`, |after_hours_change_pct|>3%면 `INFO:AFTERHOURS±x%`.
타점(entry_zone) 하이브리드(사용자 결정 — 액션은 규칙, 진입 가격대는 LLM 판단):
- 엔진이 액션별 **참고구간 ref_entry_zone**을 계산: PYRAMID `[max(ma20, high_60d−0.5×ATR14), high_60d+1×ATR14]`, AVERAGE_DOWN `[max(ma200×0.97, swing_low_60d), ma50]`, REBALANCE(drift<0) BUY존 `[ma20−1×ATR14, ma20+0.5×ATR14]`, SELL/WAIT/REBALANCE(drift>0) → null.
- 실행 대상 액션(PYRAMID·AVERAGE_DOWN·drift<0 REBALANCE)은 **portfolio_coach LLM이 기술 데이터로 entry_zone을 판단**(04 §4.3). §5 범위 가드 통과 시 그 값, 위반/degraded면 ref_entry_zone(수식) 폴백. entry_zone 산출값을 position_signals.entry_zone_low/high에 저장.
- SELL/WAIT: entry_zone NULL. stop_suggest는 항상 R1 stop 값(규칙, LLM 아님).
LLM task=portfolio_coach(04 §4.3)가 포지션별 3~4문장 해설 + entry_zone 판단 생성(손절·액션은 규칙 값 인용, 진입존만 판단, |after_hours|>3%면 해설에 반영).
evidence(scope='signal', ref_key '{date}/positions', #17): 포지션별 규칙 입력 원값을 `signal_inputs.json`으로 동결 + evidence_files 등록.
알림(action→alerts.type 매핑, #17): R1 트리거 SELL→`STOP_LOSS`, R3 트리거 SELL→`RISK_EVENT`, 그 외 SELL·PYRAMID·AVERAGE_DOWN·REBALANCE 또는 전일 대비 action 변경→`SIGNAL_CHANGE`. alerts 생성(+Telegram env 있으면 발송).
스냅샷(#37): S6 말미에 `portfolio_snapshots(portfolio_id, date, total_value_krw, total_pnl_pct)` 1행 UPSERT(자산곡선).

## S7. Publish

daily_briefs UPSERT(§S2~S4 산출 + market_wrap + earnings_calendar + upcoming_events 조립) → brief 레벨 evidence(scope='brief', `market_inputs.json`: 레짐/F&G/자금흐름 입력 원값) 동결 → batch_runs status 확정(success | degraded).
검증 게이트: indices·sectors·fear_greed가 전부 존재해야 publish. 하나라도 없으면 status=failed, 직전 brief 유지(웹은 항상 마지막 성공 brief 표시).
**Telegram 다이제스트(#36, best-effort — 실패해도 run status 무영향)**: env(TELEGRAM_BOT_TOKEN·CHAT_ID) 존재 시 발송 — `Phase {n} · F&G {score} {label} · 시그널: SELL {a}/불타기 {b}/물타기 {c}/리밸 {d} · 이벤트: {importance=3 headline 최대 2}` + 웹 링크.

## S8. 데이터 정리 (retention)

S7 publish 이후 매 배치(수동 재수집 포함)마다 실행한다. 멱등이며 best-effort다. 실패해도 `batch_runs.status`에는 영향을 주지 않고 `batch_runs.stage_status["S8"]`에 사유를 기록한다.

환경변수 기본값:

| env | 기본값 | 정리 대상 |
|---|---:|---|
| RETENTION_ENABLED | true | false면 no-op |
| RETENTION_PRICES_DAYS | 730 | `prices_daily.date < today-N` |
| RETENTION_NEWS_DAYS | 60 | `news.published_at < today-N` |
| RETENTION_LLM_LOGS_DAYS | 90 | `llm_call_logs.created_at < today-N` |
| RETENTION_SNAPSHOT_DAYS | 90 | `factor_scores.as_of`, `stock_analyses.as_of`, `position_signals.as_of < today-N` |
| RETENTION_BATCH_RUNS_DAYS | 90 | `batch_runs.started_at < today-N` |
| RETENTION_EVIDENCE_DAYS | 60 | `evidence_files.created_at` 또는 `data/evidence/{date}/...` 날짜 디렉터리 기준 |

영구 보존(삭제 금지): `daily_briefs`, `fear_greed_history`, `portfolio_snapshots`.

FK 안전 규칙: `daily_briefs.batch_run_id`가 참조하는 `batch_runs` 행은 삭제하지 않는다. `batch_runs` 정리는 `started_at < 컷오프 AND daily_briefs가 참조하지 않는` 행만 삭제한다. `position_signals`는 `positions` FK가 `ON DELETE CASCADE`이므로 오래된 `as_of` 행을 직접 삭제한다.

Evidence 정리: `data/evidence/` 하위 파일만 실제 unlink한다. `evidence_files.created_at`, `evidence_files.ref_key`의 날짜 접두어, 또는 evidence 루트 하위 날짜 디렉터리가 컷오프보다 오래되면 파일을 삭제하고 해당 `evidence_files` 행을 삭제한다. evidence 루트 밖 경로는 unlink하지 않는다.

`stage_status["S8"]`에는 삭제 건수를 요약한다. 예: `{"status":"ok","prices":123,"news":45,"llm_call_logs":20,"factor_scores":80,"stock_analyses":30,"position_signals":90,"batch_runs":5,"evidence_files":10,"evidence_disk_files":10}`.
