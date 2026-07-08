# MorningBrief — 개인 투자 분석 서비스 마스터 플랜

작성일: 2026-07-07 | 상태: 확정 (이 문서 세트의 모든 값은 결정 사항이며, 구현 시 임의 변경 금지)

> 이 문서 세트(00~05)만으로 어떤 구현자(LLM 포함)든 동일한 서비스를 만들 수 있도록 모든 선택지를 값으로 고정한다.
> 방법론 원전: `vibe-investing`. **이 서비스가 실제 채택한 것**: 멀티-LLM 위원회(04 §4.4), 5-Factor 침체 컴포지트(02 §S2.1), 4-Phase 레짐(02 §S2.1), 어닝 서프라이즈·캘린더 입력(02 §S1·§S2.6). **참고만 하고 미채택**: ARDS 5-Dimension 100점 채점, 어닝모멘텀 Top-30 전체 파이프라인 — 이 두 방법론의 종목 채점은 본 서비스의 US/Domestic Score(02 §S2.6)로 대체하며, 방법론 명칭을 근거로 별도 구현하지 않는다.
> 데이터/키 원전: `../vibe_investing_personal_analysis_site_plan.md` (이하 "기존 기획")

## 1. 서비스 정의

- **코드명**: `morning-brief`
- **한 줄 정의**: 매일 아침 6시 30분(KST) 배치가 미국 장 마감 데이터를 분석해 Bloomberg Terminal 밀도 + Perplexity 가독성의 데일리 브리핑을 생성하고, 개인 포트폴리오에 대해 장전 매매 시그널(대기/매도/불타기/물타기/리밸런싱 + 타점)을 제시하는 1인용 서비스. (신규 매수 후보는 별도로 메인 추천 카드에 표시 — 보유 종목 시그널과 분리, #4)
- **사용자**: 1명 (소유자). 중급 투자자 수준의 지표 밀도.
- **면책**: 개인 서비스이므로 추천 강도에 제한 없음. 단, LLM은 엔진이 계산한 숫자만 해설한다(숫자 생성 금지).

### 요구사항 → 문서 매핑

| # | 요구사항 | 구현 위치 |
|---|---|---|
> 문서 간 참조 규약: 05(프론트)의 위치는 섹션 ID(S-A~S-G, 종목/포트폴리오/설정 페이지)로만 가리킨다(§4.x 같은 소제목 번호는 05에 존재하지 않음).

| # | 요구사항 | 구현 위치 |
|---|---|---|
| 1 | 매일 06:30 KST 스케줄 분석 → 메인 데이터 생성 | 02_BATCH_PIPELINE.md |
| 2 | LLM을 변수로 교체 가능 | 04_LLM_SPEC.md (llm_profiles + task routing) |
| 3 | 메인: 섹터별 추이 (중급자용 지표) | 02 §S2.4, 05 S-F |
| 4 | 오늘의 이벤트(Perplexity/Bloomberg 느낌) + 유망 섹터/종목 추천 | 02 §S3·S4, 05 S-C·S-G |
| 5 | 메인 상단 floating 지수 스트립, 히트맵, Fear & Greed | 05 S-A·S-D, 02 §S2.2·S2.3 |
| 6 | 시장 자금 흐름 | 02 §S2.5, 05 S-E |
| 7 | 종목 상세: 펀더멘탈/거버넌스/케파빌리티 탭 + LLM 해석 + 근거 다운로드 | 02 §S5, 03 §4, 04 §4, 05 종목 상세 |
| 8 | 로그인 + 포트폴리오(평단/수량) + 장전 시그널/타점 | 01 (portfolios), 02 §S6, 03 §6, 05 포트폴리오 |

## 2. 확정 기술 스택

| 레이어 | 선택 | 버전 | 선정 이유 (변경 금지) |
|---|---|---|---|
| 백엔드 언어 | Python | 3.12 | yfinance/pandas 분석 생태계 |
| 백엔드 프레임워크 | FastAPI | 0.115.x | 비동기 + pydantic v2 스키마 계약 |
| ORM | SQLAlchemy | 2.0.x | |
| 마이그레이션 | Alembic | 1.13.x | |
| DB | PostgreSQL | 16 | JSONB로 brief/evidence 저장 |
| 스케줄러 | APScheduler | 3.10.x | AsyncIOScheduler, API 프로세스 내장 (별도 워커 없음 — 1인 서비스) |
| LLM 게이트웨이 | LiteLLM | 1.x | provider/model 문자열 교체만으로 벤더 스위칭 |
| 패키지 관리 | uv | latest | |
| 프론트 | Next.js (App Router) | 15.x | |
| 프론트 언어 | TypeScript | 5.x | strict |
| 스타일 | Tailwind CSS | 4.x | TOSS 토큰을 CSS 변수로 주입 (05 §2) |
| 서버상태 | TanStack Query | 5.x | |
| 클라 상태 | zustand | 4.x | |
| 차트 | Apache ECharts (echarts-for-react) | 5.x | treemap 히트맵/게이지/라인 전부 커버 |
| 인증 | 자체 JWT (HS256) | python-jose | 단일 사용자, bcrypt 해시 |
| 배포 | docker compose | v2 | db + api + web 3컨테이너 |
| 파이썬 데이터 | yfinance, pandas, numpy, pykrx, httpx | latest | |

미채택(이유 고정): Celery/Redis(1인 서비스 과설계), Supabase(자체 JWT로 충분), Recharts(treemap 미흡).

## 3. 아키텍처

```text
[APScheduler 06:30 KST]                [Next.js web :3000]
        │ (api 프로세스 내장)                  │ REST
        ▼                                     ▼
[FastAPI api :8000] ◄──────────────── /api/v1/*
   │  ├ collectors/   yfinance·FRED·Finnhub·Toss·DART·pykrx
   │  ├ engines/      factor·regime·fearGreed·moneyflow·sector·signal
   │  ├ llm/          LiteLLM 게이트웨이 + task routing + 프롬프트
   │  └ evidence/     근거 JSON/CSV 생성·저장 (로컬 볼륨 ./data/evidence)
   ▼
[PostgreSQL 16 :5432]   raw(가격·재무·공시·뉴스) + 산출(daily_brief·scores·signals)
```

원칙:
- **배치가 유일한 쓰기 경로**: 웹 요청은 사전 계산된 산출물만 읽는다(포트폴리오 CRUD 제외). 장중 실시간 갱신 없음.
- **LLM 역할 제한**: 입력 JSON의 숫자만 인용해 해설. 출력은 pydantic 스키마 검증, 실패 시 1회 재시도 후 degraded.
- **모든 산출물에 evidence**: 계산에 쓰인 원천 숫자를 JSON으로 동결·다운로드 가능.

## 4. 레포 구조 (전체 트리 — 파일 단위 고정)

```text
morning-brief/
├── docker-compose.yml
├── .env.example
├── plan/                          # 본 문서 세트
├── server/
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── alembic/
│   └── app/
│       ├── main.py                # FastAPI 앱 + APScheduler 기동
│       ├── config.py              # pydantic-settings, env 로드
│       ├── db.py                  # engine/session
│       ├── models/                # SQLAlchemy (01 문서의 테이블 1:1)
│       ├── schemas/               # pydantic 응답 (03 문서의 JSON 1:1)
│       ├── api/
│       │   ├── auth.py  brief.py  stocks.py  portfolio.py  llm_admin.py  batch.py
│       ├── collectors/
│       │   ├── yf.py  fred.py  finnhub.py  dart.py  toss.py  krx.py
│       ├── engines/
│       │   ├── factors.py         # momentum/value/quality_growth/low_vol (02 §S2.6)
│       │   ├── regime.py          # 5-factor composite + 4-phase (02 §S2.1)
│       │   ├── fear_greed.py      # (02 §S2.3)
│       │   ├── money_flow.py      # (02 §S2.5)
│       │   ├── sector.py          # (02 §S2.4)
│       │   ├── recommend.py       # 섹터/종목 추천 (02 §S4)
│       │   └── signals.py         # 포트폴리오 시그널·타점 (02 §S6)
│       ├── llm/
│       │   ├── gateway.py         # LiteLLM 래핑 + routing + 비용로그
│       │   └── prompts.py         # 04 문서의 프롬프트 원문 상수
│       ├── batch/
│       │   ├── pipeline.py        # S1~S7 오케스트레이션
│       │   └── stages/            # stage_collect.py ... stage_publish.py
│       └── evidence/store.py
├── web/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── app/
│       │   ├── page.tsx           # 메인 (05 S-A~S-G)
│       │   ├── stocks/[ticker]/page.tsx   # 종목 상세 (05 종목 상세)
│       │   ├── portfolio/page.tsx # (05 포트폴리오)
│       │   ├── settings/page.tsx  # LLM 프로파일 (05 설정)
│       │   └── login/page.tsx
│       ├── components/            # 05 컴포넌트 파일 매핑 1:1
│       ├── lib/api.ts             # fetch 래퍼 (Bearer)
│       └── styles/tokens.css      # TOSS 토큰 (05 §2)
└── data/
    ├── evidence/                  # {date}/{scope}/*.json|csv
    ├── econ_calendar_2026.json    # 경제지표 발표 일정 (연 1회 수동 갱신, 02 §S1·§S3)
    └── krx/                       # 유니버스 CSV (기존 기획 6.4)
```

## 5. 환경변수 (.env.example 전문)

```env
# ── Core
DATABASE_URL=postgresql+psycopg://mb:mb_dev@db:5432/morning_brief
JWT_SECRET=change-me-64-bytes
ADMIN_EMAIL=you@example.com
ADMIN_PASSWORD_HASH=            # bcrypt, scripts/hash_pw.py로 생성
TZ=Asia/Seoul
BATCH_CRON=30 6 * * *           # 요구 1. 변경은 env로만
RETENTION_ENABLED=true
RETENTION_PRICES_DAYS=730
RETENTION_NEWS_DAYS=60
RETENTION_LLM_LOGS_DAYS=90
RETENTION_SNAPSHOT_DAYS=90
RETENTION_BATCH_RUNS_DAYS=90
RETENTION_EVIDENCE_DAYS=60
ANALYSIS_STALE_HOURS=6
ANALYSIS_TRIGGER_DEBOUNCE_MIN=5

# ── LLM (04 문서: DB llm_profiles가 우선, env는 키 보관용)
ANTHROPIC_API_KEY=      # Claude 전역 엔진(claude-main)
OPENAI_API_KEY=         # Codex 전역 엔진(codex)
DEEPSEEK_API_KEY=
GEMINI_API_KEY=

# ── 데이터 소스 (기존 기획 §6 기반)
FINNHUB_KEY=              # 미국 종목 뉴스 수집 + 어닝 캘린더/서프라이즈(02 §S1)
# ALPHAVANTAGE_KEY=       # 미사용 — 뉴스 감성은 news_classify LLM task(04 §4.6)가 산출(#7 결정)
DART_API_KEY=
TOSS_CLIENT_ID=
TOSS_CLIENT_SECRET=
FRED_CACHE_TTL_SEC=86400
YFINANCE_CACHE_TTL_SEC=300

# ── 알림 (선택)
TELEGRAM_BOT_TOKEN=
TELEGRAM_OWNER_CHAT_ID=
```

## 6. docker-compose (확정 골격)

```yaml
services:
  db:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: mb, POSTGRES_PASSWORD: mb_dev, POSTGRES_DB: morning_brief }
    volumes: [ mb_pgdata:/var/lib/postgresql/data ]
    ports: [ "5434:5432" ]
  api:
    build: ./server
    env_file: .env
    environment: { TZ: Asia/Seoul }
    volumes: [ ./data:/app/data ]
    ports: [ "8010:8000" ]
    depends_on: [ db ]
  web:
    build: ./web
    ports: [ "3010:3000" ]
    depends_on: [ api ]
volumes: { mb_pgdata: {} }
```

포트 고정: db 5434 / api 8010 / web 3010 (기존 workCheck의 5433/8081과 충돌 방지).

## 7. 마일스톤 (구현 순서 고정)

| M | 범위 | 완료 기준 |
|---|---|---|
| M0 | 레포 골격 + compose + DB 마이그레이션 + 로그인 | /api/v1/auth/login 200, 웹 로그인 |
| M1 | 수집기(yfinance·FRED·Finnhub) + 지수/섹터/F&G/자금흐름 엔진 + 배치 S1~S2 | daily_brief 행 생성, 수동 트리거로 재현 |
| M2 | 메인 페이지 전체 (지수 스트립·이벤트·히트맵·F&G·자금흐름·섹터 테이블) | 05 S-A~S-G 컴포넌트 전부 실데이터 렌더 |
| M3 | 이벤트 다이제스트 + 섹터/종목 추천 (LLM task 4종: news_classify·event_digest·sector_outlook·committee) + LLM 설정 페이지 | S3·S4 산출 + 프로파일 교체 동작 |
| M4 | 종목 상세(탭 7종 + evidence 다운로드) | NVDA·005930 두 종목 검증 |
| M5 | 포트폴리오 CRUD + 장전 시그널/타점 + (선택) Telegram | S6 산출, 시그널 5종(WAIT·SELL·PYRAMID·AVERAGE_DOWN·REBALANCE) 규칙 테스트 통과 |

국내(Toss/DART) 수집기는 M4에서 합류 (미국 우선 — yfinance 무키로 즉시 가동 가능, 기존 기획 §7과 동일한 판단).

## 8. 문서 인덱스

| 문서 | 내용 |
|---|---|
| 01_DATABASE.md | 전체 DDL (테이블 21개) |
| 02_BATCH_PIPELINE.md | 06:30 파이프라인 S1~S7, 모든 산식·시그널 규칙 |
| 03_API_SPEC.md | REST 전 엔드포인트 요청/응답 JSON |
| 04_LLM_SPEC.md | LLM 변수화 구조 + task 6종 프롬프트 원문 + 출력 스키마 (라우팅 6종 + profile_test는 라우팅 밖) |
| 05_FRONTEND_SPEC.md | TOSS 디자인 토큰, 페이지/컴포넌트 명세 |
