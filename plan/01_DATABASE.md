# 01. 데이터베이스 스키마 (PostgreSQL 16, 확정 DDL)

규칙: 테이블/컬럼명 변경 금지. Alembic 초기 마이그레이션은 아래 DDL과 동일해야 한다.
기존 기획 §8 초안을 기반으로 서비스 요구(브리핑 산출물, LLM 라우팅, 시그널)를 추가 확장한 최종본.

```sql
-- ───────────────────────── 사용자/인증 (요구 8)
CREATE TABLE users (
  id            BIGSERIAL PRIMARY KEY,
  email         VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,          -- bcrypt
  role          VARCHAR(10)  NOT NULL DEFAULT 'user', -- admin|user
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- 시드: ADMIN_EMAIL/ADMIN_PASSWORD_HASH 1행(role='admin'). 회원가입 API 없음(1인 서비스).

-- ───────────────────────── LLM 변수화 (요구 2)
CREATE TABLE llm_profiles (
  id           BIGSERIAL PRIMARY KEY,
  name         VARCHAR(50)  NOT NULL UNIQUE,    -- 예: 'claude-main'
  provider     VARCHAR(30)  NOT NULL,           -- litellm provider: anthropic|openai|deepseek|gemini
  model        VARCHAR(100) NOT NULL,           -- 예: 'claude-sonnet-5'
  api_key_env  VARCHAR(50)  NOT NULL,           -- 키를 읽을 env 변수명 (DB에 키 원문 저장 금지)
  base_url     VARCHAR(255),
  temperature  NUMERIC(3,2) NOT NULL DEFAULT 0.3,
  max_tokens   INT          NOT NULL DEFAULT 4096,
  enabled      BOOLEAN      NOT NULL DEFAULT true,
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE llm_task_routing (
  task                VARCHAR(40) PRIMARY KEY,  -- 라우팅 대상 task 6종 (profile_test는 라우팅 밖 — 04 §2)
  profile_id          BIGINT NOT NULL REFERENCES llm_profiles(id),
  fallback_profile_id BIGINT REFERENCES llm_profiles(id),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_settings (
  key        VARCHAR(80) PRIMARY KEY,           -- 예: 'llm_engine'
  value      TEXT NOT NULL,                     -- llm_engine: claude|codex
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- 시드: ('llm_engine', 'claude') 1행. 전역 토글이 6개 LLM task 라우팅을 일괄 갱신한다.

CREATE TABLE llm_call_logs (                    -- 비용/감사 추적
  id          BIGSERIAL PRIMARY KEY,
  task        VARCHAR(40) NOT NULL,
  profile_id  BIGINT REFERENCES llm_profiles(id),
  batch_run_id BIGINT,
  input_tokens  INT, output_tokens INT,
  latency_ms    INT,
  status      VARCHAR(20) NOT NULL,             -- ok|schema_retry|failed
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ───────────────────────── 마스터/원천 (기존 기획 §8)
CREATE TABLE stocks (
  id        BIGSERIAL PRIMARY KEY,
  market    VARCHAR(10) NOT NULL,               -- US|KR
  ticker    VARCHAR(20) NOT NULL,               -- NVDA | 005930
  name      VARCHAR(100) NOT NULL,
  search_alias VARCHAR(120),                    -- 한글 별칭 검색어(예: 구글 알파벳)
  sector    VARCHAR(50),                        -- GICS 섹터 (02 §S2.4 매핑)
  industry  VARCHAR(80),                        -- KRX 로더는 전용 exchange 컬럼이 없어 Market(KOSPI/KOSDAQ)을 보관
  currency  VARCHAR(5) NOT NULL,
  is_etf    BOOLEAN NOT NULL DEFAULT false,
  source    VARCHAR(20) NOT NULL,
  UNIQUE (market, ticker)
);

CREATE TABLE prices_daily (
  stock_id BIGINT NOT NULL REFERENCES stocks(id),
  date     DATE   NOT NULL,
  open NUMERIC(18,4), high NUMERIC(18,4), low NUMERIC(18,4),
  close NUMERIC(18,4) NOT NULL, adj_close NUMERIC(18,4), volume BIGINT,
  source   VARCHAR(20) NOT NULL,
  PRIMARY KEY (stock_id, date)
);

CREATE TABLE financials (                        -- 분기/연간 (yfinance→미국, DART→국내)
  id BIGSERIAL PRIMARY KEY,
  stock_id BIGINT NOT NULL REFERENCES stocks(id),
  period   VARCHAR(10) NOT NULL,                -- 2026Q1|2025FY
  report_date DATE, effective_date DATE,
  revenue NUMERIC(20,2), operating_income NUMERIC(20,2), net_income NUMERIC(20,2),
  equity NUMERIC(20,2), total_debt NUMERIC(20,2), cash NUMERIC(20,2),
  eps NUMERIC(12,4), fcf NUMERIC(20,2),
  extras JSONB,                                  -- 소스별 추가 필드 (rnd, margin 등)
  source VARCHAR(20) NOT NULL,
  UNIQUE (stock_id, period, source)
);

CREATE TABLE disclosures (                       -- 국내 공시 (거버넌스 탭 원천)
  id BIGSERIAL PRIMARY KEY,
  stock_id BIGINT REFERENCES stocks(id),
  corp_code VARCHAR(10), rcept_no VARCHAR(20) UNIQUE,
  date DATE NOT NULL, title TEXT NOT NULL, category VARCHAR(50),
  url TEXT,
  risk_tags TEXT[],                              -- {CB,BW,유상증자,최대주주변경,소송,관리종목}
  source VARCHAR(20) NOT NULL DEFAULT 'dart'
);

CREATE TABLE news (
  id BIGSERIAL PRIMARY KEY,
  stock_id BIGINT REFERENCES stocks(id),         -- NULL = 시장 전반 뉴스
  market VARCHAR(4) NOT NULL DEFAULT 'US',        -- US|KR, event_digest 시장별 입력 분리
  published_at TIMESTAMPTZ NOT NULL,
  source VARCHAR(50), title TEXT NOT NULL, url TEXT, summary TEXT,
  sentiment NUMERIC(4,3),                        -- -1~1, 소스 제공 시
  UNIQUE (url)
);

CREATE TABLE macro_series (                      -- FRED 등 매크로 시계열 (전부 무키 CSV, 02 §S1)
  series_id VARCHAR(30) NOT NULL,                -- T10Y3M|T10Y2Y|SAHMREALTIME|BAMLH0A0HYM2|NFCI|CFNAIMA3|PERMIT|DGS10
  date DATE NOT NULL,                            -- (USSLIND 폐지→CFNAIMA3·PERMIT로 교체, 02 §S2.1 참조)
  value NUMERIC(14,6),
  PRIMARY KEY (series_id, date)
);

-- ───────────────────────── 배치 산출물 (요구 1,3,4,5,6)
CREATE TABLE batch_runs (
  id BIGSERIAL PRIMARY KEY,
  brief_date DATE NOT NULL,                      -- 브리핑 대상일 (KST)
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  finished_at TIMESTAMPTZ,
  status VARCHAR(20) NOT NULL DEFAULT 'running', -- running|success|degraded|failed
  stage_status JSONB NOT NULL DEFAULT '{}',      -- {"S1":"ok","S3":"skipped:finnhub_down",...}
  trigger VARCHAR(10) NOT NULL DEFAULT 'cron'    -- cron|manual
);

CREATE TABLE daily_briefs (                      -- 메인 페이지 = 이 한 행
  -- 저장(JSONB) vs 조회 시 조립 구분(#20): 아래 컬럼은 그대로 저장되고,
  --   03 §2 응답의 generated_at(←batch_runs.finished_at 조인)·fear_greed.history_30d(←fear_greed_history 최근 30행)는
  --   저장 컬럼이 아니라 조회 시 조립되는 파생 필드다. 그 두 필드를 제외하면 JSONB 컬럼 ↔ 03 §2 키가 1:1.
  brief_date DATE PRIMARY KEY,
  batch_run_id BIGINT REFERENCES batch_runs(id),
  regime JSONB NOT NULL,                         -- {composite, phase, phase_label, factors:{A..E 각 {value,score}}, implied_ffr, implied_ffr_bp_1d, kr_regime} (#14, kr_regime=국내레짐 02 §S2.1)
  indices JSONB NOT NULL,                        -- 지수 스트립 배열 (현물+선물+DGS10, 02 §S2.2 순서)
  fear_greed JSONB NOT NULL,                     -- {score, label, factors:{F1..F5}} (history_30d는 조회 시 조립)
  sectors JSONB NOT NULL,                        -- 섹터 11종 지표 배열
  heatmap JSONB NOT NULL,                        -- treemap 노드 배열
  money_flow JSONB NOT NULL,                     -- {sector_flows[], asset_flows[], inflow_top[], outflow_top[]}
  events JSONB NOT NULL,                         -- 오늘의 이벤트 카드 배열 (S3 산출)
  market_wrap TEXT NOT NULL DEFAULT '',          -- 하루 시장 총평 2~3문장 (요구 '총정리', event_digest 산출, #15)
  recommendations JSONB NOT NULL,                -- {sectors[{...,outlook_text}], stocks[{...,entry_hint}]} (S4 산출, #5)
  earnings_calendar JSONB NOT NULL DEFAULT '[]', -- [{ticker, date, when:BMO|AMC}] 보유∪추천 향후 14일 (#12)
  upcoming_events JSONB NOT NULL DEFAULT '[]',   -- [{date, name, importance}] 경제지표 향후 7일 (#13, econ_calendar_2026.json 소스)
  kr JSONB,                                      -- 국장 홈 토글 payload(nullable): {indices,sectors,heatmap,fear_greed,money_flow,regime,recommendations,events,market_wrap}
  degraded_sections TEXT[] NOT NULL DEFAULT '{}'
);

CREATE TABLE fear_greed_history (                -- 게이지 스파크라인용
  date DATE PRIMARY KEY,
  score NUMERIC(5,2) NOT NULL, label VARCHAR(20) NOT NULL,
  factors JSONB NOT NULL
);

CREATE TABLE factor_scores (                     -- 종목 팩터 (02 §S2.6)
  stock_id BIGINT NOT NULL REFERENCES stocks(id),
  as_of DATE NOT NULL,
  momentum NUMERIC(5,1), value NUMERIC(5,1), quality_growth NUMERIC(5,1),  -- 팩터명 통일 (#19: 02·03·05 동일)
  low_vol NUMERIC(5,1), macro_fit NUMERIC(5,1), risk_event NUMERIC(5,1),
  total NUMERIC(5,1) NOT NULL,
  signal VARCHAR(12) NOT NULL,                   -- BUY|HOLD|REDUCE|EXIT|DIP_BUY
  detail JSONB NOT NULL,                         -- 원시 지표 전부 (RSI, MA50/200 괴리, ma20, ATR14, 52주고점 대비, 성장률 …) (#3 entry_hint 원천)
  PRIMARY KEY (stock_id, as_of)
);

CREATE TABLE stock_analyses (                    -- 종목 상세 탭 산출 (요구 7)
  id BIGSERIAL PRIMARY KEY,
  stock_id BIGINT NOT NULL REFERENCES stocks(id),
  as_of DATE NOT NULL,
  tabs JSONB NOT NULL,                           -- 5키 {fundamental, governance, capability, technical, macro} 각 {metrics, llm_text} (#22: overview 탭은 없음 — UI '종합' 탭은 top-level score/committee로 조립)
  committee JSONB NOT NULL,                      -- {bull, bear, risk, pm:{verdict, conviction, summary}, one_liner}
  evidence_dir VARCHAR(255) NOT NULL,            -- data/evidence/{as_of}/{ticker}/
  updated_at TIMESTAMPTZ,                        -- 마지막 성공 갱신 시각(배치·온디맨드 공통 공유 결과)
  last_attempt_at TIMESTAMPTZ,                   -- 마지막 분석 시도 시각(성공·실패 무관, 버튼 디바운스)
  source VARCHAR(12) NOT NULL DEFAULT 'batch',   -- batch|on_demand
  UNIQUE (stock_id, as_of)
);

CREATE TABLE evidence_files (
  id BIGSERIAL PRIMARY KEY,
  scope VARCHAR(20) NOT NULL,                    -- brief|stock|signal
  ref_key VARCHAR(60) NOT NULL,                  -- '2026-07-07' | '2026-07-07/NVDA'
  filename VARCHAR(120) NOT NULL,                -- prices_1y.csv, factor_inputs.json …
  path VARCHAR(255) NOT NULL,
  bytes INT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (scope, ref_key, filename)
);

-- ───────────────────────── 포트폴리오 (요구 8, 기존 기획 §5)
CREATE TABLE portfolios (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  name VARCHAR(50) NOT NULL DEFAULT '기본',
  base_currency VARCHAR(5) NOT NULL DEFAULT 'KRW'
);

CREATE TABLE positions (
  id BIGSERIAL PRIMARY KEY,
  portfolio_id BIGINT NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  stock_id BIGINT NOT NULL REFERENCES stocks(id),
  quantity NUMERIC(18,6) NOT NULL,
  avg_cost NUMERIC(18,4) NOT NULL,               -- 평단 (요구 8)
  currency VARCHAR(5) NOT NULL,
  target_weight NUMERIC(5,4),                    -- 0~1, 선택 (설정 시 02 §S6 REBALANCE 판정에 사용, #8)
  stop_loss_price NUMERIC(18,4),                 -- 선택 (미입력 시 stop_loss_pct→기본식 순, 02 §S6 R1)
  stop_loss_pct NUMERIC(5,4),                    -- 선택, 평단 대비 손절률 (원전 §5, #24)
  take_profit_rule TEXT,                         -- 선택, 익절/트레일링 룰 서술 (원전 §5, #24 — MVP는 표시만)
  review_interval VARCHAR(20),                   -- 선택, 리뷰 주기 (원전 §5, #24 — MVP는 표시만)
  thesis TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), -- 보유일수 산출 기준 (02 §S6 R0.5 VIXY 30일, #27)
  UNIQUE (portfolio_id, stock_id)
);

CREATE TABLE position_signals (                  -- 장전 시그널 (S6 산출, 보유 종목 한정 — 신규 매수는 daily_briefs.recommendations, #4)
  position_id BIGINT NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
  as_of DATE NOT NULL,
  action VARCHAR(15) NOT NULL,                   -- WAIT|SELL|PYRAMID|AVERAGE_DOWN|REBALANCE (BUY 제거, #4/#8)
  entry_zone_low NUMERIC(18,4), entry_zone_high NUMERIC(18,4),  -- 타점 (요구 8)
  stop_suggest NUMERIC(18,4),
  drift NUMERIC(6,4),                            -- weight − target_weight (REBALANCE 사유, #8; target_weight 없으면 NULL)
  after_hours_change_pct NUMERIC(8,4),           -- 보유 종목 시간외 등락 %, 없으면 NULL (#34)
  pnl_pct NUMERIC(8,4) NOT NULL,                 -- 단위: % 값 (예 19.1). 엔진 규칙 평가는 소수(0.191), 저장·직렬화는 ×100 (#32)
  reasons JSONB NOT NULL,                        -- 규칙 트리거 목록 (02 §S6 룰 ID + INFO 태그)
  llm_text TEXT,                                 -- portfolio_coach 해설
  PRIMARY KEY (position_id, as_of)
);

CREATE TABLE portfolio_snapshots (               -- 자산곡선 (S6 말미 UPSERT, #37)
  portfolio_id BIGINT NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  total_value_krw NUMERIC(20,2) NOT NULL,
  total_pnl_pct NUMERIC(8,4) NOT NULL,           -- % 값 (#32)
  PRIMARY KEY (portfolio_id, date)
);

CREATE TABLE alerts (
  id BIGSERIAL PRIMARY KEY,
  position_id BIGINT REFERENCES positions(id) ON DELETE CASCADE,
  type VARCHAR(30) NOT NULL,                     -- STOP_LOSS|SIGNAL_CHANGE|RISK_EVENT (action→type 매핑은 02 §S6, #17)
  message TEXT NOT NULL,
  status VARCHAR(10) NOT NULL DEFAULT 'new',     -- new|sent|read
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ───────────────────────── 인덱스
CREATE INDEX idx_prices_date ON prices_daily(date);
CREATE INDEX idx_news_pub ON news(published_at DESC);
CREATE INDEX idx_disc_date ON disclosures(date DESC);
CREATE INDEX idx_analyses_asof ON stock_analyses(as_of DESC);
CREATE INDEX idx_stock_analyses_stock_asof ON stock_analyses(stock_id, as_of DESC);
```

## 시드 데이터 (마이그레이션에 포함)

문서 자기완결 원칙(00 §1): 아래 유니버스는 전 티커를 인라인으로 명기한다(외부 문서 참조로 대체 금지, #16).

1. `users`: env의 관리자 1행(role='admin').
2. `portfolios`: 관리자 user 소유 1행 (`name='기본'`, `base_currency='KRW'`) — 03 §6 단일 포트폴리오 전제가 첫 실행부터 성립 (#33).
3. `llm_profiles` 4행 + `llm_task_routing` 6행 — 04 §2 표 그대로 (profile_test는 라우팅 밖, #21).
4. `stocks`: 아래 고정 유니버스 (`source='seed'`).
   - **지수/프록시** (is_etf=false, sector NULL): `^GSPC ^IXIC ^NDX ^DJI ^VIX ^KS11 ^KQ11 KRW=X BTC-USD`
   - **지수 선물** (is_etf=false, sector NULL, market US): `ES=F NQ=F YM=F` — S&P500·나스닥100·다우 선물 (요구 4 '선물', #1)
   - **금리 선물** (is_etf=false, sector NULL): `ZQ=F` — 30일 연방기금 선물, 시장 내재 정책금리 계산 (#14)
   - **섹터 ETF 11종**: `XLK XLF XLE XLV XLY XLP XLI XLU XLB XLRE XLC`
   - **자산군 ETF**: `SPY QQQ IWM TLT IEF GLD UUP HYG LQD`
   - **M7 + AI Infra 18종** (유니크, NVDA 중복 제외, #39): `AAPL MSFT GOOGL AMZN META NVDA TSLA` + `AMD AVGO TSM MU ASML SMCI ANET DELL ORCL PLTR VRT`
   - **ARDS Tier1~4 25종** (스킬 §6 인라인): T1 `XLP XLV XLU VPU VHT` / T2 `TLT IEF SHV GLD IAU BIL` / T3 `BRK-B WMT COST JNJ KO PG PEP NEE VZ ABBV` / T4 `SH PSQ VIXY DOG` (Tier4 하드룰은 02 §S6 R0/R0.5, #27)
   - **S4 섹터→대표종목 매핑 종목** (02 §S4 매핑표 전 종목, #6): `CRM`(XLK) · `JPM BAC GS MS`(XLF) · `LLY UNH MRK`(XLV) · `HD MCD NKE`(XLY) · `NFLX DIS`(XLC) · `XOM CVX SLB`(XLE) · `CAT GE HON UNP`(XLI) · `DUK SO`(XLU) · `LIN APD`(XLB) · `PLD AMT`(XLRE)
     (그 외 매핑 종목 NVDA MSFT AAPL AVGO AMD/AMZN TSLA/GOOGL META/COST WMT PG KO/NEE 등은 위 M7·AI Infra·ETF 목록에 이미 포함. CRM만 위 목록에 없어 여기 명시적으로 추가 — 재감사 H3)
   - **국내 핵심/홈 유니버스**: `005930 000660 069500`(KODEX 200) `229200`(KODEX 코스닥150) · KR 섹터 ETF `091160`(KODEX 반도체) `091170`(KODEX 은행) `091180`(KODEX 자동차) `305720`(KODEX 2차전지산업) `102970`(KODEX 증권) `117700`(KODEX 건설) `266420`(KODEX 헬스케어) `117680`(KODEX 철강) `117460`(KODEX 에너지화학) `266410`(KODEX 필수소비재) · KR 자산군 ETF `471230`(KODEX 국고채10년액티브) `132030`(KODEX 골드선물(H)) `261240`(KODEX 미국달러선물)
   - `stocks.name`: 자동완성 품질을 위해 시드 시 실제 표시명을 upsert한다. 예: `NVDA=NVIDIA`, `AAPL=Apple`, `MSFT=Microsoft`, `GOOGL=Alphabet`, `AMZN=Amazon`, `META=Meta`, `TSLA=Tesla`, `AVGO=Broadcom`, `TSM=TSMC`, `JPM=JPMorgan`, `LLY=Eli Lilly`, `SPY=S&P500 ETF`, `QQQ=나스닥100 ETF`, `XLK=미국 기술 섹터`, `005930=삼성전자`, `000660=SK하이닉스`, `069500=KODEX 200`, `091160=KODEX 반도체`, `471230=KODEX 국고채10년액티브`. 매핑표에 없는 티커는 `name=ticker`로 폴백한다.
   - `stocks.search_alias`: 영문 표시명 종목의 한글 검색 품질을 위해 실제 통용 표기만 nullable 문자열로 upsert한다. 예: `AAPL=애플`, `MSFT=마이크로소프트`, `GOOGL=구글 알파벳`, `NVDA=엔비디아`, `TSLA=테슬라`, `SPY=S&P500 에스앤피`, `QQQ=나스닥100 나스닥`, `TLT=미국 장기채`, `GLD=금 골드`. 불확실한 별칭은 NULL로 둔다.
   - **불변조건**: 02 §S4 섹터→대표종목 매핑표의 모든 티커는 이 시드 유니버스의 부분집합이어야 한다. 02 §S2.6 "추천 후보 유니버스" = (섹터 ETF ∪ 자산군 ETF ∪ M7/AI Infra ∪ S4 매핑 종목). z-score 유니버스도 이 집합으로 계산. 포트폴리오에 유니버스 밖 종목 추가 시 03 §6 자동 등록(source='yfinance_auto', #11).
5. `scripts/load_kr_stocks.py`: 검색 전용 KRX 전종목 로더. `finance-datareader`의 `StockListing('KRX')`를 무계정 호출해 `stocks`에 upsert한다. 신규 행은 `ticker=Code(6자리)`, `name=Name`, `search_alias=영문 Name의 실제 통용 한글 별칭(있으면)`, `market='KR'`, `currency='KRW'`, `source='krx'`, `sector=Sector(있으면)`, `industry=Market(KOSPI/KOSDAQ)`, `is_etf=ETF/ETN/KODEX/TIGER 등 이름 휴리스틱`으로 저장한다. 기존 `source='seed'` 행(`005930`, `000660`, `069500` 등)은 매일 배치 핵심 유니버스 보존을 위해 `source`를 덮어쓰지 않는다. 이 로더는 seed와 분리된 수동/주기 실행 스크립트이며, 네트워크 실패는 명확히 보고하고 비치명 종료한다.

테이블 수: 21 (users, llm_profiles, llm_task_routing, llm_call_logs, stocks, prices_daily, financials, disclosures, news, macro_series, batch_runs, daily_briefs, fear_greed_history, factor_scores, stock_analyses, evidence_files, portfolios, positions, position_signals, portfolio_snapshots, alerts). `daily_briefs` JSONB 상세 구조는 03_API_SPEC.md §2 응답과 1:1(generated_at·fear_greed.history_30d 두 조립 필드 제외)이므로 03 §2를 단일 원본으로 삼는다. `daily_briefs.kr`은 국장 홈 토글 전용 additive payload이며 US 최상위 필드는 기존 계약을 유지한다.
