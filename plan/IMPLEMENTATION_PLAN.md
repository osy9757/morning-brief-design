# 구현 계획 — 마일스톤 브리프 + QA 게이트

Claude(아키텍처·QA)가 정의하고, Codex(구현)가 마일스톤 단위로 수행한다. 각 마일스톤은 QA 게이트를 통과해야 다음으로 진행한다. 스펙 원본은 `plan/00~05`.

## 진행 상태 (TDD — 각 단계는 테스트 green + 해당 Playwright E2E 통과로만 완료)

| M | 범위 | 상태 |
|---|---|---|
| M0 | 레포 골격 + compose + DB 마이그레이션 + 로그인 | ✅ 완료·검증(Run 1: pytest 6, playwright 1 green) |
| M1 | 수집기(yfinance·FRED·Finnhub) + 지수/섹터/F&G/자금흐름 엔진 + 배치 S1~S2 | ✅ 완료·검증(pytest 18 green, 배치 daily_briefs 생성) |
| M2 | 메인 페이지 전체 (S-A~S-G) | ✅ 완료·검증·커밋(pytest 22, playwright 2 green) |
| M3 | 이벤트 다이제스트 + 섹터/종목 추천 + LLM 설정 | ✅ 완료·검증·커밋(pytest 33×2, playwright 3×2) |
| M4 | 종목 상세(탭 7종 + evidence) | ✅ 완료·검증·커밋(pytest 37×2, playwright 4×2) |
| M5 | 포트폴리오 CRUD + 장전 시그널/타점 + Telegram | ⏳ 진행 |

## Run 1 — 기반 실동작 검증 + 테스트 하네스 + M0 로그인 TDD

**목표:** M0 골격이 실제로 돈다는 것을 테스트로 증명하고, 이후 전 마일스톤이 쓸 테스트 하네스를 세운다.
1. `.env`(gitignore) 생성 — 테스트 관리자(알려진 비번 해시), DB 접속값.
2. `uv sync` → `docker compose up -d db` → `alembic upgrade head` → `python scripts/seed.py` 실제 실행.
3. **pytest 하네스**: `server/tests/` + conftest(테스트 DB·httpx ASGI 클라이언트·시드 fixture). 통합 테스트: 로그인 성공(200+JWT), 실패(401 INVALID_CREDENTIALS), 인증 필요 라우트 미인증 401.
4. **Playwright 하네스**: `web/e2e/` + playwright.config. E2E: `/login` 렌더 → 관리자 자격 입력·제출 → 홈 리다이렉트 + 토큰 저장 확인. api+web 개발서버 기동해 실제 클릭스루.
5. 시드 정합성 단위 테스트: stocks 유니버스에 CRM·선물·ZQ=F 존재, 라우팅 6행, 테이블 21개.
6. green 안 나오면 M0 코드를 수정해 통과시킨다.

**DoD:** `pytest` 전부 green + `npx playwright test` 로그인 E2E green. 실행 명령 출력으로 증명.
**QA 게이트(Claude):** 테스트가 실제로 실행됐고 green인지 로그 확인, 로그인 계약(03 §1) 준수, 시드 정합성, 시크릿 미커밋.

---

## M0 — 레포 골격 + DB + 로그인

**범위 (이것만):**
1. `plan/00 §4` 레포 트리대로 `server/`, `web/`, `data/`, `docker-compose.yml`, `.env.example`, `.gitignore` 생성.
2. **server**: FastAPI 앱(`app/main.py`) + `app/config.py`(pydantic-settings) + `app/db.py`(SQLAlchemy 2.0 async 엔진/세션) + `pyproject.toml`(uv). APScheduler는 기동 훅만 배선(잡 등록은 M1). 
3. **DB**: `plan/01`의 DDL 21개 테이블을 SQLAlchemy 모델(`app/models/`)로 1:1 정의 + Alembic 초기 마이그레이션 1개. 시드(`plan/01` 시드절): 관리자 user 1행, portfolios 기본 1행, llm_profiles 4행 + llm_task_routing 6행, stocks 전 유니버스(선물·ZQ=F·CRM 포함). 시드는 Alembic 또는 `scripts/seed.py`로.
4. **auth**: `POST /api/v1/auth/login`(bcrypt 검증 → JWT HS256, 7일). `app/api/auth.py`. `scripts/hash_pw.py`(관리자 비번 해시 생성). 인증 의존성(`get_current_user`).
5. **web**: Next.js 15 앱 골격 + `plan/05 §2` TOSS 토큰(`src/styles/tokens.css`) + `/login` 페이지(이메일/비번 → 토큰 저장) + `lib/api.ts`(Bearer + 401 처리) + `<AppShell>` 스켈레톤.
6. **docker-compose.yml**: `plan/00 §6` 골격 그대로(db 5434 / api 8010 / web 3010).

**완료 정의(DoD):** `docker compose up -d db` 후 alembic 마이그레이션 성공, `uvicorn` 기동, seed된 관리자 계정으로 `POST /api/v1/auth/login` → 200 + JWT. 웹 `/login` 렌더 + 로그인 성공 시 홈 리다이렉트.

**범위 밖:** 배치 실제 로직, 수집기, 메인 페이지 데이터, 포트폴리오. (골격/배선만)

**QA 게이트(Claude):** ① 22개 모델 ↔ plan/01 DDL 컬럼·타입·제약 대조 ② 시드 티커/개수 대조(테이블 22·라우팅 6·CRM 존재) ③ 로그인 실제 호출 200 ④ 포트/버전 스펙 일치 ⑤ 시크릿 미커밋(.gitignore) ⑥ TODO(claude) 목록 검토.

---

## M1 — 수집기 + 엔진(S2) + 배치 S1~S2 (TDD)

스펙: `plan/02 §S1·§S2` 전체. 이번 단계는 **LLM 없음**(S3~S4는 M3). 결정적 산식만.

**범위:**
1. **수집기**(`server/app/collectors/`): yfinance(가격·펀더멘털·시간외·실적일), FRED CSV(T10Y3M·T10Y2Y·SAHMREALTIME·BAMLH0A0HYM2·NFCI·CFNAIMA3·PERMIT·DGS10), Finnhub(뉴스). **테스트 가능 구조**: HTTP 클라이언트/응답을 주입 가능하게 해 목으로 대체. yfinance/FRED는 무키.
2. **엔진**(`server/app/engines/`): regime(5-factor→phase, implied_ffr, kr_regime), fear_greed(F1~F5), sector(S2.4 지표+heatmap 노드), money_flow(S2.5, flow_score=clip(z,0,3)×sign), factors(S2.6 total·signal·detail: ma20/ATR14/ref_buy_zone 포함). **전부 plan/02 산식·계수 그대로.**
3. **배치**(`server/app/batch/`): S1(수집→prices_daily/financials/news/macro_series/earnings/econ) → S1.5(news_classify는 목 or 스킵, M3에서 실LLM) → S2(엔진 계산) → daily_briefs의 regime·indices·fear_greed·sectors·heatmap·money_flow 부분 UPSERT. 수동 트리거 `POST /api/v1/batch/run`(03 §8) 배선 + APScheduler cron(BATCH_CRON) 등록.

**TDD 요구:**
- 엔진 단위테스트: plan 예시값으로 검증 — regime composite=28.2(03 §2 입력 재계산), phase 경계 반개구간, fear_greed F1~F5, flow_score 부호(z<0→0), factors signal 임계(70/55/40)·DIP_BUY 승격. 고정 fixture 입력.
- 배치 통합테스트: prices_daily/macro_series에 fixture 시드 → S2 실행 → daily_briefs 행의 regime/fear_greed/sectors/money_flow shape·값 단언.
- 수집기 테스트: 목 HTTP 응답 → 파싱·저장 검증(라이브 API 미의존).

**범위 밖:** LLM(event_digest·committee·추천·stock_report), 종목 상세, 포트폴리오, 프론트 렌더(M2). daily_briefs의 events·recommendations·market_wrap은 이 단계에서 빈 값/플레이스홀더.

**DoD:** `uv run pytest` 전부 green(신규 엔진·배치·수집기 테스트 포함). 배치 수동 트리거로 daily_briefs 행 생성 재현.
**QA 게이트(Claude):** ① pytest 실제 green 재실행 ② regime/fear_greed/flow_score/factors 산식이 plan/02와 일치(코드 대조) ③ 배치가 daily_briefs를 스펙 shape로 씀 ④ 무키로 동작(키 없이 크래시 없음).

## M2 — 메인 페이지 S-A~S-G + brief API (TDD + E2E)

스펙: `plan/03 §2·§3`(brief API), `plan/05 §3·§4`(레이아웃·S-A~S-G). 차트는 ECharts(echarts-for-react).

**범위:**
1. **백엔드**: `GET /brief/today`(최신 성공 brief), `/brief/{date}`, `/brief/dates`, `/fear-greed/history`. daily_briefs 읽기 + 조립 필드(generated_at←batch_runs.finished_at, fear_greed.history_30d←fear_greed_history). 404 BRIEF_NOT_FOUND.
2. **프론트**: `plan/05 §3` AppShell(sticky IndexTicker) + §4 S-A~S-G 컴포넌트 전부 — IndexTicker(12항목, DGS10 bp·선물 배지), RegimeBanner(phase·composite·implied_ffr), TodayEvents(빈 상태 허용), SectorHeatmap(treemap), FearGreedGauge(반원 게이지), MoneyFlowBoard(자산군 바+섹터 플로우), SectorTable(11행), Recommendations(빈 상태 허용). TOSS 토큰·tabular-nums·등락 착색·스켈레톤·degraded 칩. TanStack Query로 GET /brief/today 1회.
3. **결정성**: E2E 전 고정 daily_briefs fixture를 시드(라이브 배치 아님)해 화면 값이 예측 가능하게.

**TDD:**
- 백엔드 pytest: /brief/today 200 shape(regime·indices 12·fear_greed·sectors 11·money_flow), 없을 때 404, /fear-greed/history 조립.
- Playwright E2E: 로그인 → 메인 → **실제로** 지수 스트립 12개·섹터 테이블 11행·F&G 게이지·히트맵·RegimeBanner phase 텍스트가 보이는지 클릭/단언. 섹터 히트맵 클릭→테이블 스크롤 같은 인터랙션 1개 이상.

**범위 밖:** events·recommendations 내용 생성(M3 LLM), 종목 상세(M4), 포트폴리오(M5) — 해당 섹션은 빈 상태 렌더.

**DoD:** `uv run pytest` green + `npx playwright test` 메인 페이지 E2E green(실제 렌더 데이터 클릭·단언).
**QA 게이트(Claude):** ① pytest·playwright 실제 green 재실행 ② 05 §4 컴포넌트·필드 대조(지수 12·섹터 11·DGS10 bp) ③ brief API 조립 필드 정합 ④ 빈 상태·degraded 처리.

## M3 — LLM 게이트웨이 + 이벤트/추천 + 설정 (TDD, LLM 전부 목)

스펙: `plan/04`(게이트웨이·task 6종 프롬프트·검증), `plan/02 §S1.5·§S3·§S4`, `plan/03 §7`(LLM admin), `plan/05 §7`(설정).

**범위:**
1. **게이트웨이**(`server/app/llm/gateway.py`): LiteLLM 래핑. `call(task, payload, out_schema)` — llm_task_routing 조회 → profile.api_key_env를 os.environ에서 → litellm 호출 → pydantic 검증(실패 1회 재시도 → fallback → degraded). llm_call_logs 기록. **테스트에서 litellm 호출을 목으로 주입 가능**하게(의존성 주입/monkeypatch). 프롬프트 원문은 `prompts.py`에 plan/04 그대로.
2. **배치 S1.5/S3/S4**: news_classify(sentiment 채움) · event_digest(카드+market_wrap, 각주검증) · sector_outlook · committee · recommendations(섹터랭킹 결정적 + 종목후보 + committee + **entry_hint는 엔진이 ref_buy_zone 주입**). daily_briefs의 events·recommendations·market_wrap·earnings_calendar·upcoming_events 채움.
3. **LLM admin API**(03 §7): GET/POST/PUT/DELETE /llm/profiles, GET/PUT /llm/routing(6 task), POST /llm/profiles/{id}/test, GET /llm/usage. 배치: POST /batch/run·GET /batch/runs(03 §8).
4. **설정 페이지**(05 §7): LLM 프로파일 카드+폼, Task 라우팅 6행, 사용량 차트, 배치 패널(stage 8칩·지금실행).

**TDD (키 없이 전부 green):**
- 게이트웨이: 목 응답으로 스키마 검증·재시도·fallback·degraded 경로. 숫자검증(입력에 없는 3자리+ 수치 거부)·event_digest 각주 범위 검증.
- 추천 엔진: 섹터랭킹 점수(0.4·rs + 0.3·flow + 0.3·phase_fit)·경기민감 섹터표·entry_hint=ref_buy_zone 결정적 단위테스트.
- 배치 S3/S4 통합: 목 LLM로 daily_briefs.events/recommendations 채워지는지.
- E2E: 설정 페이지 렌더(프로파일·라우팅 6행)·라우팅 변경 PUT, 메인 페이지에 (목 배치로 채운) 이벤트 카드·추천 카드 렌더.

**범위 밖:** 종목 상세(M4), 포트폴리오(M5). 실제 LLM 키(사용자가 나중에).

**DoD:** `uv run pytest` green(LLM 목) + `npx playwright test` 설정·메인 green. 키 없이 동작.
**QA 게이트(Claude):** ① pytest·playwright 실제 green ② 프롬프트 원문 plan/04 대조 ③ entry_hint 엔진주입·각주검증·숫자검증 동작 ④ 키 미설정 시 graceful degraded(크래시 없음).

## M4 — 종목 상세(탭 7종 + evidence 다운로드) (TDD + E2E)

스펙: `plan/02 §S5`, `plan/03 §4·§5`, `plan/05 종목 상세`, `plan/04 §4.2`(stock_report)·§4.4(committee).

**범위:**
1. **백엔드**: `GET /stocks/{ticker}/analysis?date=`(03 §4 계약: price·score.breakdown·committee·tabs 5키·evidence), `GET /stocks/search`, `POST /stocks/{ticker}/analyze`(온디맨드: 대상 티커 S1수집+S2.6+S5 즉석), `GET /evidence/{scope}/{ref_key}/{filename}`(첨부 다운로드).
2. **배치 S5**: stock_analyses 생성 — tabs metrics(S2.6 선계산 조립: fundamental·governance 등급·capability·technical·macro), stock_report(LLM 5탭 텍스트, 목), committee(S4 미보유 종목은 여기서 호출), evidence 동결(`data/evidence/{date}/{ticker}/`에 prices_1y.csv·factor_inputs.json·financials.json·news_used.json·disclosures.json·llm_io.json + evidence_files 등록).
3. **프론트**(`/stocks/[ticker]`): 헤더(score 링·레이더 6축)·위원회 4열(Bull/Bear/Risk/PM)·탭 바(종합|펀더멘탈|거버넌스|케파빌리티|테크니컬|매크로|근거)·탭별 차트(8분기 바·마진 라인·종가+MA 라인)·근거 파일 다운로드 리스트·미분석 종목 "지금 분석하기".

**TDD:**
- 백엔드: analysis 200 shape(tabs 5·committee), 미분석 404→analyze 202, evidence 다운로드(Content-Disposition), governance 등급 산출(뉴스 건수 규칙), 온디맨드 즉석 계산.
- E2E: 로그인→종목(예 NVDA) 상세→탭 전환(펀더멘탈·거버넌스·테크니컬)→근거 탭 다운로드 클릭. 결정적 fixture stock_analyses 시드.

**범위 밖:** 포트폴리오(M5). 실제 LLM 키(목).

**DoD:** `uv run pytest` green(2회) + `npx playwright test` 종목 상세 E2E green(2회).
**QA 게이트(Claude):** ① pytest·playwright 직접 2회 green ② tabs 5키·committee·evidence 파일 실제 생성 ③ 03 §4 계약·05 탭 대조 ④ 온디맨드·다운로드 동작.

## M5 — 포트폴리오 + 장전 시그널 + 타점 + Telegram (TDD + E2E)

스펙: `plan/02 §S6·§S7`(규칙·타점·알림·스냅샷·Telegram), `plan/03 §6`(포트폴리오 API), `plan/05 포트폴리오`, `plan/04 §4.3`(portfolio_coach).

**범위:**
1. **백엔드 API**(03 §6): GET /portfolio(total_value_krw·total_pnl_pct·value_history_30d·positions), POST/PUT/DELETE /portfolio/positions(**유니버스 밖 ticker는 yfinance 자동등록 #11**, 중복 409), GET /portfolio/signals, GET /alerts·PUT /alerts/{id}/read. 환율 KRW=X.
2. **배치 S6**: 포지션별 규칙 R0~R7 평가(첫 매치 확정) — R0(Tier4 Phase1 SELL)·R0.5(VIXY 30일)·R1(stop)·R2(total<40&손실)·R3(거버넌스 HIGH)·R4(불타기)·R5(물타기)·R6(REBALANCE drift≥0.05)·R7(WAIT). 타점 산식(PYRAMID·AVERAGE_DOWN·REBALANCE·BUY존), drift, after_hours, portfolio_coach(목), signal evidence(scope=signal), alerts(action→type: R1→STOP_LOSS·R3→RISK_EVENT·그외→SIGNAL_CHANGE), portfolio_snapshots UPSERT.
3. **S7 Telegram**(§S7): env 있으면 다이제스트 발송(best-effort, 실패 무영향).
4. **프론트**(/portfolio): 요약 카드(총평가액·30일 자산곡선 스파크·시그널 카운트), 포지션 테이블(시그널 칩 SELL/PYRAMID/AVERAGE_DOWN/REBALANCE/WAIT·시간외 칩·실적 D-n), 행 확장(entry_zone·stop·drift·reasons·coach), 추가/수정 모달(전 필드), 알림 배너.

**TDD:**
- 규칙 단위테스트: R0~R7 각 트리거 조건·우선순위(첫 매치)·타점 산식(plan/02 §S6 값)·drift·alerts 매핑. pnl_pct 단위(엔진 소수/저장 %).
- API: 포지션 CRUD·자동등록·중복 409·signals shape·snapshots.
- E2E: 로그인→포트폴리오→포지션 추가(모달)→시그널 표시→행 확장. 결정적 fixture.

**DoD:** `uv run pytest` green(2회) + `npx playwright test` 포트폴리오 E2E green(2회). 기존 M0~M4 전부 green 유지.
**QA 게이트(Claude):** ① pytest·playwright 2회 green ② R0~R7 규칙·타점 산식 plan/02 §S6 코드 대조 ③ 자동등록·스냅샷·alerts 매핑 ④ Telegram env 없을 때 무영향.

## 최종 — 전체 통합 E2E

M5 후: 배치 전체(S1~S7) 1회 실행 → 메인·종목·포트폴리오 전 화면이 실데이터로 렌더되는 통합 Playwright 스모크 + 전체 pytest·playwright green. `.env`에 실제 키 투입 시 라이브 전환 절차를 README에 기록.

각 마일스톤 착수 직전 Claude가 위 M0 형식(범위/DoD/범위밖/QA게이트)으로 브리프를 확정해 이 파일에 추가한다. 원칙:
- M1: `plan/02 §S1~S2` 결정적 산식 우선(LLM 없음). 수집기는 무키 소스(yfinance·FRED)부터.
- M2: `plan/05 §4`(S-A~S-G) + `plan/03 §2` GET /brief/today 1회 호출.
- M3: `plan/04` LLM task 6종 + 게이트웨이(LiteLLM) + 라우팅 CRUD.
- M4: `plan/02 §S5` + `plan/03 §4` + evidence 다운로드.
- M5: `plan/02 §S6` 규칙 R0~R7 + `plan/03 §6`.
