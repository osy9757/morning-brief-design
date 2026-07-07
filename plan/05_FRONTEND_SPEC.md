# 05. 프론트엔드 명세 — TOSS 디자인 시스템 (Next.js 15)

## 1. 디자인 방향

TOSS(토스증권) 스타일: 어두운 배경 위 고밀도 숫자, 큰 타이포 위계, 카드 단위 정보 블록, 상승=빨강/하락=파랑(한국 관례). 다크 모드 단일 테마(라이트 모드 미구현 — 결정 사항).

## 2. 디자인 토큰 (`web/src/styles/tokens.css` — 값 고정)

```css
:root {
  /* 색상 — TOSS 팔레트 준용 */
  --bg-base: #101013;          /* 페이지 배경 */
  --bg-card: #1B1C20;          /* 카드 */
  --bg-elevated: #26272C;      /* 호버/중첩 카드 */
  --border: #2E2F36;
  --text-primary: #E4E6EB;
  --text-secondary: #9DA1AB;
  --text-tertiary: #6B6F7A;
  --accent: #3182F6;           /* Toss Blue — CTA/링크/활성탭 */
  --up: #F04452;               /* 상승/수익 (빨강) */
  --down: #3485FA;             /* 하락/손실 (파랑) */
  --flat: #9DA1AB;
  --warn: #FFB331;             /* degraded/리스크 MID */
  --danger: #F04452;           /* 리스크 HIGH/SELL */
  --success: #1CB868;          /* 데이터 정상/BUY 계열 */

  /* 타이포 — Pretendard (TOSS 계열 대체 폰트, self-host woff2) */
  --font-family: 'Pretendard Variable', -apple-system, sans-serif;
  --font-num: 'Pretendard Variable', 'SF Mono', monospace; /* 숫자는 tabular-nums */
  --fs-display: 28px; --fs-title: 20px; --fs-body: 15px; --fs-caption: 13px; --fs-micro: 11px;
  --fw-bold: 700; --fw-semi: 600; --fw-reg: 400;

  /* 간격/형태 */
  --radius-card: 20px; --radius-chip: 10px;
  --sp-1: 4px; --sp-2: 8px; --sp-3: 12px; --sp-4: 16px; --sp-5: 24px; --sp-6: 32px;
  --shadow-float: 0 4px 20px rgba(0,0,0,.45);
  --maxw: 1200px;              /* 데스크톱 콘텐츠 폭, 모바일 100% 반응형 */
}
```

공통 규칙: 모든 수치는 `font-variant-numeric: tabular-nums`. 등락 표기는 `+1.23%`/`-0.87%`에 --up/--down 착색. 카드 = `bg-card + radius-card + padding sp-5`. 스켈레톤 로딩 필수(TanStack Query isLoading).

## 3. 공통 레이아웃

- `<AppShell>`: 상단 고정 헤더(로고 "MorningBrief", 종목 검색 인풋 `SearchBox`, 탭 네비: 홈·포트폴리오·설정) + 콘텐츠. 헤더 아래 **IndexTicker가 sticky로 붙는다**.
- 인증 가드: 홈(`/`)·종목(`/stocks/*`)은 공개 렌더, `/portfolio`는 로그인 user 이상만 접근(JWT 없으면 /login), `/settings`는 admin만 접근(JWT 없으면 /login, role!='admin'이면 홈). 헤더는 비로그인 상태에서 "로그인" 버튼을 표시하고, 설정 링크는 role=='admin'일 때만 노출한다. `lib/api.ts`는 보호 경로에서 401 수신 시 토큰 폐기 후 /login.
- 데이터: 메인은 `GET /brief/today` 1회 → 각 섹션에 props 분배. `degraded_sections`에 포함된 섹션은 카드 우상단에 `--warn` 칩 "데이터 일부 누락".

## 4. 메인 페이지 `/` — 섹션 순서 고정 (요구 5의 배치 그대로)

```text
[S-A IndexTicker]  ← sticky floating, 스크롤해도 상단 고정
[S-B RegimeBanner]
[S-C TodayEvents]        (요구 4 전반)
[S-D 2단 그리드: SectorHeatmap(2/3) + FearGreedGauge(1/3)]
[S-E MoneyFlowBoard]     (요구 6)
[S-F SectorTable]        (요구 3)
[S-G Recommendations]    (요구 4 후반)
```

### S-A `IndexTicker` (floating 지수 스트립)

- 가로 스크롤 없는 wrap 스트립. 12개 항목은 반응형 `flex-wrap`/그리드로 여러 줄 표시한다. `position: sticky; top: 헤더높이; backdrop-filter: blur(12px); background: rgba(16,16,19,.82); box-shadow: --shadow-float`.
- 항목: brief.indices 순서 그대로 12개(현물 4 + 선물 2(ES=F·NQ=F) + VIX + 국내 2 + 환율 + BTC + 미국채10Y). YM=F(다우선물)는 수집만·미표시. 셀 = `label · last(fw-semi) · change_pct(착색) · 30일 스파크라인(ECharts mini-line 60×24px, 색은 등락 방향)`. is_stale이면 label 옆 `--text-tertiary` "휴장" 칩.
- **선물 항목(ES=F/NQ=F, #1)**: label에 "선물" 배지로 현물과 구분하고, label+"선물" 배지는 한 줄로 표시한다. 값은 현물 마감 대비 등락으로 읽힌다.
- **미국채 10Y 항목(symbol=DGS10, #2)**: change_pct 대신 `change_bp_1d`를 "+5bp/−3bp"로 착색(금리는 % 착색이 무의미). 클릭 네비게이션 없음(stocks 유니버스 밖).
- 클릭 시 해당 심볼이 stocks에 있으면 /stocks/{ticker} 이동(선물·DGS10은 무동작).

### S-B `RegimeBanner`

한 줄 배너: `Phase {n} · {phase_label}` 칩(Phase1 --success, 2 --warn, 3 #FF7A00, 4 --danger) + composite 게이지 바(0~100) + "침체 컴포지트 {composite}%" + "시장 내재 정책금리 {implied_ffr}% ({±implied_ffr_bp_1d}bp)" 텍스트(#14) + factors A~E 미니 도트(score>0.5 붉은 도트). 클릭 → 툴팁으로 5-factor 상세. 국내 종목 보유 시 `kr_regime`(RISK_ON/OFF/DEFENSIVE) 배지 병기.

### S-C `TodayEvents` (Perplexity/Bloomberg 무드)

- **상단 총평 블록(#15)**: `market_wrap` 2~3문장을 카드 최상단에 fs-body로 표시(요구 '총정리').
- **일정 스트립(#12·#13)**: 총평 아래 한 줄 — `upcoming_events`(이번 주 경제지표, importance=3은 강조) + `earnings_calendar`(이번 주 실적, "NVDA D-2" 형태) 칩 나열.
- 세로 피드형 카드 리스트. importance=3은 대형 카드(headline fs-title), 1~2는 컴팩트 행.
- 카드 구성: 카테고리 칩(MACRO 파랑/EARNINGS 초록/GEOPOLITICS 빨강/SECTOR 보라 #8B5CF6/CRYPTO 주황/KR 회색) + headline + body(**본문 각주 [n]을 source_urls[n-1]로 링크되는 상단첨자로 치환, Perplexity식, #35**) + 출처 링크(파비콘+도메인, 새 탭) + related_tickers 칩(클릭→종목 페이지).
- 우상단 "생성 {generated_at HH:mm}" 표시 — 신뢰성 각인.

### S-D-1 `SectorHeatmap`

ECharts treemap. 노드 = brief.heatmap (size=고정 시총 가중, value=ret_1d). 색 스케일 고정: -3% `#2E5BFF` ~ 0% `#3A3B42` ~ +3% `#F04452` (연속 보간, 범위 밖 클립). 노드 라벨: `{name}\n{ret_1d}%`. 클릭 → S-F 해당 행으로 스크롤.

### S-D-2 `FearGreedGauge`

ECharts gauge(반원). 기본은 시장 전체 `brief.fear_greed`, 섹터 히트맵 노드 또는 섹터 테이블 행 클릭 시 해당 섹터의 `fear_greed.score/label`로 전환한다. "전체" 버튼 또는 같은 섹터 재클릭으로 시장 전체로 리셋한다. 바늘 = score. 구간 배경 5색: 0-25 `#2E5BFF`, 25-45 `#5B7BD5`, 45-55 `#6B6F7A`, 55-75 `#D5705B`, 75-100 `#F04452`. 하단: label 한글(극단적 공포/공포/중립/탐욕/극단적 탐욕). F1~F5 미니 바와 30일 히스토리 스파크라인은 시장 전체일 때만 표시한다.

### S-E `MoneyFlowBoard` (요구 6)

좌: **자산군 로테이션 바 차트** — asset_flows 9종을 ret_20d 내림차순 가로 바(양수 --up, 음수 --down), 각 바 끝에 ret_5d 보조 표기. 상단 요약문(정적 조립, LLM 아님): `"자금 유입: {inflow_top.join(' · ')} / 유출: {outflow_top.join(' · ')}"`.
우: **섹터 플로우 히트 스트립** — 11 섹터를 flow_score 순 정렬, 칩 색 = flow_score(-3 파랑 ~ +3 빨강), 칩 내용 `{name} {flow_score}`. 툴팁: volume_ratio, rs_spy_5d.

### S-F `SectorTable` (요구 3 — 중급자 밀도)

11행 고정 테이블, 컬럼(순서 고정): `섹터 | 1D | 1W | 1M | 3M | YTD | RS(1M) | 추세(UP/MIXED/DOWN 칩) | MA50 | MA200 | RSI | 거래대금비 | 자금흐름`. 수익률 셀 전부 등락 착색, MA는 ✓/✗, 기본 정렬 = 1D 내림차순(헤더 클릭 재정렬). 모바일: 가로 스크롤 컨테이너.

### S-G `Recommendations`

상단: 유망 섹터 3카드 — `{sector} · score` + `outlook_text`(sector_outlook 산출, #5) + reason_metrics 3개 칩.
하단: 추천 종목 카드 그리드(= brief.recommendations.stocks, 신규 매수 후보 진입점 #4) — `ticker/name`, verdict 칩(STRONG_BUY `--success` 채움/BUY 테두리), `확신도 {conviction}/10` 도트 게이지, one_liner, `참고 진입 {entry_hint.zone_low}~{entry_hint.zone_high}`(엔진 산출 #3). 카드 클릭 → /stocks/{ticker}. 카드 하단 마이크로 카피: "개인 분석용 · 투자판단 본인 책임" (fs-micro, text-tertiary).

## 5. 종목 상세 `/stocks/[ticker]` (요구 7)

헤더: name/ticker · last · 1d등락 · signal 칩 · total 점수 링(0~100 원형 게이지) + 레이더 차트(breakdown 6축, ECharts radar).
**위원회 블록**: 4열(모바일 아코디언) — 🐂 Bull / 🐻 Bear / ⚠ Risk / ⚖ PM(verdict+conviction 강조). PM 카드만 --accent 테두리.
**탭 바** (순서 고정): `종합 | 펀더멘탈 | 거버넌스 | 케파빌리티 | 테크니컬 | 매크로 | 근거`. 활성 탭 하단 2px --accent.

탭은 stock_analyses.tabs의 5키(펀더멘탈·거버넌스·케파빌리티·테크니컬·매크로)와 UI 조립 탭 2개(종합·근거)로 구성 — '종합'은 별도 저장 키가 아니라 top-level score/committee로 조립(#22).

| 탭 | 구성 |
|---|---|
| 종합 | 점수 링 + 레이더 + PM summary + 5개 탭 llm_text 첫 문장 모음(각 탭 링크) — tabs.overview 키 없음(#22) |
| 펀더멘탈 | metrics 4칩(PER·FwdPER·PBR·배당) + 8분기 매출/순이익 이중 바 차트 + EPS 라인 + llm_text |
| 거버넌스 | grade 칩(HIGH --danger/MID --warn/LOW --success) + items 타임라인(date·tag 칩·title·원문 링크) + llm_text |
| 케파빌리티 | 성장 지표 칩(매출YoY·EPS YoY·ROE·R&D비중) + 마진 추이 4분기 라인 + llm_text |
| 테크니컬 | 1년 캔들 대신 **종가+MA50+MA200 라인 차트**(ECharts, 고정 결정) + metrics 칩(RSI·MA괴리·52주고점比·변동성) + llm_text |
| 매크로 | Phase 칩 + macro_fit 게이지 + 소속 섹터 RS/flow 재표기 + llm_text |
| 근거 | evidence 파일 리스트(파일명·크기·다운로드 버튼) + "이 분석의 모든 숫자는 좌측 파일에서 재계산 가능합니다" 카피 |

미분석 종목 진입 시: "아직 분석되지 않은 종목" 상태 화면 + [지금 분석하기] 버튼(POST analyze → 폴링 스피너 → 완료 시 리로드).

## 6. 포트폴리오 `/portfolio` (요구 8)

- 상단 요약 카드: 총 평가액(KRW, fs-display) + **30일 자산곡선 스파크라인**(value_history_30d, common/Sparkline 재사용, #37) · 총 손익 % · 오늘 시그널 요약 칩 카운트(`SELL 1 · 물타기 2 · 리밸 1 · 대기 4`).
- 포지션 테이블: `종목 | 수량 | 평단 | 현재가 | 손익%(착색) | 비중(목표比) | 오늘 시그널`. 현재가 옆 `시간외 {±x}%` 미니 칩(after_hours_change_pct 있을 때, #34). 종목명 옆 실적 임박 시 "D-2" 칩(#12). 비중 셀은 `weight`(target_weight 있으면 "/목표 {t}%" 병기). 시그널 칩 색(보유 종목 한정, BUY 없음 #4): SELL --danger, PYRAMID(불타기) --up, AVERAGE_DOWN(물타기) --accent, REBALANCE(리밸런싱) --warn, WAIT --text-tertiary.
- 행 확장(아코디언): `entry_zone [{low} ~ {high}]`(REBALANCE 초과보유 시 매도 안내로 zone 없음) + `손절 제안 {stop_suggest}` + drift(REBALANCE 시) + reasons 룰 칩(INFO 태그 포함) + llm_text 코칭 + [상세 분석 →](/stocks/{ticker}).
- 신규 매수 후보는 여기 아닌 메인 S-G Recommendations에 표시(#4).
- 포지션 추가/수정 모달: ticker 검색(SearchBox 재사용, 유니버스 밖 종목은 자동 등록 #11) → quantity, avg_cost, currency, target_weight, stop_loss_price·stop_loss_pct(선택), take_profit_rule·review_interval(선택), thesis (#24). TOSS식 풀스크린 바텀시트(모바일)/중앙 모달(데스크톱).
- 알림 영역: alerts status=new 목록 배너.

## 7. 설정 `/settings` (요구 2)

- **LLM 프로파일 카드 리스트**: provider·model·temperature 표시, [테스트] 버튼(→ /test, 결과 latency 토스트), 추가/수정 폼(provider 셀렉트 4종 + model 자유입력 + api_key_env 셀렉트).
- **Task 라우팅 테이블**: 6 task 행(profile_test 제외, #21) × (기본 프로파일 셀렉트, fallback 셀렉트). 변경 즉시 PUT. profile_test는 프로파일 카드의 [테스트] 버튼이 선택 프로파일로 직접 호출.
- **사용량**: 최근 30일 task별 토큰/호출수 바 차트.
- **배치**: 마지막 실행 상태(stage_status 칩 8개 S1·S1.5·S2·S3·S4·S5·S6·S7), [데이터 새로고침] 버튼(확인 다이얼로그 → POST /batch/run → 진행 폴링, admin 전용 재수집). 당일 brief evidence(scope='brief') 다운로드 링크 목록 — market_inputs.json 등(#17).

## 8. 컴포넌트 파일 매핑 (web/src/components/)

```text
layout/AppShell.tsx  layout/SearchBox.tsx
brief/IndexTicker.tsx  brief/RegimeBanner.tsx  brief/TodayEvents.tsx
brief/SectorHeatmap.tsx  brief/FearGreedGauge.tsx  brief/MoneyFlowBoard.tsx
brief/SectorTable.tsx  brief/Recommendations.tsx
stock/StockHeader.tsx  stock/CommitteePanel.tsx  stock/AnalysisTabs.tsx  stock/EvidenceList.tsx
portfolio/SummaryCard.tsx  portfolio/PositionTable.tsx  portfolio/PositionModal.tsx  portfolio/SignalDetail.tsx
settings/LlmProfiles.tsx  settings/TaskRouting.tsx  settings/UsageChart.tsx  settings/BatchPanel.tsx
common/Card.tsx  common/Chip.tsx  common/DeltaText.tsx  common/Sparkline.tsx  common/ScoreRing.tsx  common/Skeleton.tsx
```

수용 기준(전 페이지): ① 모바일 375px~데스크톱 1200px 반응형 ② 모든 등락 수치 착색+tabular-nums ③ 로딩 스켈레톤 ④ degraded 섹션 경고 칩 ⑤ 빈 상태(첫 배치 전) 안내 화면.
