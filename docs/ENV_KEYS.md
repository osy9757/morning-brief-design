# MorningBrief — 환경변수 · API 키 설정 가이드

이 프로젝트를 돌리는 데 필요한 키·설정을 한 곳에 정리한 문서. 실제 값은 `.env`에 넣는다(`.env.example` 참고, `.env`는 git 커밋 금지).

> **핵심 원칙**: 시장 데이터·검색·국장 뉴스는 **키 없이도 동작**한다. 키가 필요한 건 **LLM 해설/이벤트 생성**과 **일부 데이터 소스(미국 뉴스·국내 공시)** 뿐이다. 키가 없으면 그 기능만 "degraded"로 비고, 서비스는 죽지 않는다.

---

## 1. 필수 — 서비스 실행 (외부 API 키 아님, 설정값)

| 항목 | 용도 | 비고 |
|---|---|---|
| `DATABASE_URL` | PostgreSQL 접속 | docker-compose 기본값 있음 |
| `JWT_SECRET` | 로그인 토큰(HS256) 서명 | 임의 64바이트 문자열 |
| `ADMIN_EMAIL` | admin 계정 이메일 | 이 계정만 설정·재수집·엔진 토글 가능 |
| `ADMIN_PASSWORD_HASH` | admin 비밀번호 (bcrypt) | `python scripts/hash_pw.py`로 생성 |

이 4개만 있으면 서비스가 뜨고, **지수·섹터·Fear&Greed·자금흐름·종목 시세·검색·국장 뉴스 수집**까지 무키로 동작한다.

---

## 2. LLM 엔진 — 분석·이벤트 생성 (최소 1개)

설정 페이지의 **전역 토글(Claude ↔ Codex)** 이 사용하는 키. 토글에서 선택한 엔진의 키만 있으면 된다(번갈아 쓰려면 둘 다).

| 키 | 엔진 | 모델(기본) | 발급처 |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | **Claude** | claude-sonnet-5 | console.anthropic.com |
| `OPENAI_API_KEY` | **Codex** | gpt-5-codex | platform.openai.com |
| `DEEPSEEK_API_KEY` | (선택, 토글 밖) | deepseek-chat | platform.deepseek.com |
| `GEMINI_API_KEY` | (선택, 토글 밖) | — | aistudio.google.com |

**켜지는 기능**: 오늘의 이벤트 카드·시장 총평, 섹터/종목 추천, 투자위원회(Bull/Bear/Risk/PM), 종목 탭 해설, 공시 요약(filing_digest), 포트폴리오 코칭.
키가 없으면 위 섹션만 비고, 숫자 기반 대시보드는 정상.

---

## 3. 데이터 소스 — 선택 (없으면 해당 기능만 degraded)

| 키 | 켜지는 것 | 발급처 |
|---|---|---|
| `FINNHUB_KEY` | 미국 종목 **뉴스** + **실적·컨센서스**(EPS 서프라이즈·투자의견 분포·목표주가) | finnhub.io |
| `DART_API_KEY` | 국내 **공시**(거버넌스 요약) — corp_code 매핑 필요(현재 불완전) | opendart.fss.or.kr |
| `TOSS_CLIENT_ID` / `TOSS_CLIENT_SECRET` | 국내 실시간 시세 (없으면 pykrx/FDR 폴백) | tossinvest 오픈API |

---

## 4. 무키 — 키 불필요, 자동 동작 ✅

| 소스 | 용도 |
|---|---|
| **yfinance** | 미국 시세·재무제표 |
| **FRED** | 매크로(수익률곡선·Sahm·신용스프레드 등) |
| **FinanceDataReader** | 한국 전 종목 리스트·시세 |
| **EDGAR** | 미국 공시 10-K/10-Q/8-K (MD&A·Risk Factors) |
| **Google News RSS** | 국장 뉴스(코스피·코스닥·국내증시) |

---

## 5. 알림 — 선택

| 키 | 용도 |
|---|---|
| `TELEGRAM_BOT_TOKEN` + `TELEGRAM_OWNER_CHAT_ID` | 06:30 브리핑·시그널 텔레그램 발송 |

---

## 6. 설정값 (키 아님, 동작 튜닝)

| 항목 | 기본 | 의미 |
|---|---|---|
| `TZ` | Asia/Seoul | 타임존 |
| `BATCH_CRON` | `30 6 * * *` | 배치 실행 시각(매일 06:30) |
| `RETENTION_ENABLED` | true | 오래된 데이터 자동 정리 on/off |
| `RETENTION_PRICES_DAYS` | 730 | 시세 보관 일수 |
| `RETENTION_NEWS_DAYS` | 60 | 뉴스 보관 일수 |
| `RETENTION_LLM_LOGS_DAYS` | 90 | LLM 호출 로그 보관 |
| `RETENTION_SNAPSHOT_DAYS` | 90 | 스냅샷 보관 |
| `RETENTION_BATCH_RUNS_DAYS` | 90 | 배치 실행 이력 보관 |
| `RETENTION_EVIDENCE_DAYS` | 60 | evidence 파일 보관 |
| `ANALYSIS_STALE_HOURS` | 6 | 온디맨드 분석 신선도(이 시간 내면 재수집 생략) |
| `ANALYSIS_TRIGGER_DEBOUNCE_MIN` | 5 | 재수집 버튼 디바운스(분) |
| `FRED_CACHE_TTL_SEC` | 86400 | FRED 캐시 |
| `YFINANCE_CACHE_TTL_SEC` | 300 | yfinance 캐시 |

> `ALPHAVANTAGE_KEY` 는 **미사용**(뉴스 감성은 LLM `news_classify`가 산출). 넣을 필요 없음.

---

## 7. 최소 추천 구성

```dotenv
# 실행
JWT_SECRET=<임의 64바이트>
ADMIN_EMAIL=you@example.com
ADMIN_PASSWORD_HASH=<scripts/hash_pw.py 결과>

# LLM 엔진 1개 (토글에서 쓰는 쪽)
ANTHROPIC_API_KEY=<...>      # 또는 OPENAI_API_KEY

# 미국 뉴스·컨센서스
FINNHUB_KEY=<...>
```

국내 공시까지 원하면 `DART_API_KEY` 추가.

---

## 8. 적용 순서

1. `.env.example` → `.env` 복사 후 위 값 채우기
2. admin 비밀번호 해시 생성: `python scripts/hash_pw.py`
3. 컨테이너 기동: `docker compose up -d`
4. 키를 나중에 넣었거나 바꿨으면 → 설정 페이지 **"데이터 새로고침"**(admin) 으로 재수집하면 목/degraded → 라이브로 전환

---

## 부록: 키별 "있으면 / 없으면"

| 키 | 있으면 | 없으면 |
|---|---|---|
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` | LLM 해설·이벤트·추천·위원회·공시요약 | 해당 섹션 비고(대시보드 숫자는 정상) |
| `FINNHUB_KEY` | 미국 뉴스 + 실적·컨센서스 탭 | 미국 뉴스·컨센서스 탭 빈값 |
| `DART_API_KEY` | 국내 공시 기반 거버넌스 | 국내 거버넌스는 뉴스/태그 기반으로만 |
| `TOSS_*` | 국내 실시간 시세 | pykrx/FDR로 대체(문제 없음) |
| `TELEGRAM_*` | 텔레그램 알림 | 알림만 미발송 |
