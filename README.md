# Toss Auto Trader

토스증권 API 조회 데이터를 활용한 국내 주식 분석·모의매매 프로젝트입니다. 현재 저장소에는 **비동기 조회 클라이언트와 기능별 테스트 코드**가 포함되어 있습니다. 테스트는 종목 추천, 모의 자동매매, 포트폴리오 관리, 백테스트, 일별 성과 기록을 다룹니다.

> **현재 상태:** 설정 및 매매·분석 모듈 일부가 저장소에 없어 애플리케이션 실행과 전체 테스트 통과에 필요한 구현이 아직 갖춰지지 않았습니다. 아래 문서는 실제 포함된 코드와 테스트가 요구하는 기능을 구분해 설명합니다.

## 구현된 조회 클라이언트

[`TossInvestClient`](toss-auto-trader/app/toss/client.py)는 `httpx.AsyncClient` 기반의 조회 전용 클라이언트입니다. 실제 매수·매도 주문 메서드는 포함되어 있지 않습니다.

| 메서드 | 기능 | 코드에 정의된 제약·동작 |
| --- | --- | --- |
| `accounts()` | 계좌 목록 조회 | `/api/v1/accounts` 호출 |
| `resolve_account_seq()` | 조회할 계좌 선택 | 설정된 계좌를 사용하며, 미설정 시 첫 번째 계좌 선택 |
| `holdings(symbol=None)` | 보유 종목 조회 | 계좌 헤더 전달, 종목별 조회 가능 |
| `prices(symbols)` | 여러 종목의 가격 조회 | 공백 제거·대문자 변환·중복 제거 후 1~200개 허용 |
| `candles(symbol, count=30)` | 일봉 조회 | 20~200개 허용, 리스트 및 `candles`·`records`·`items` 응답 처리 |
| `list_stocks(market)` | 국내 시장 종목 목록 조회 | `KOSPI`·`KOSDAQ` 지원, 활성 보통주 필터 전달 |
| `stock_infos(symbols)` | 종목 상세정보 조회 | 중복 제거 후 1~200개 허용 |
| `warnings(symbol)` | 종목 경고 정보 조회 | 종목별 경고 조회 경로 호출 |
| `rankings(count=100)` | 국내 거래대금 순위 조회 | 조회 수를 1~100개로 제한 |
| `market_calendar_kr(date=None)` | 국내 시장 운영정보 조회 | 날짜 미지정 조회 결과를 5분간 캐시 |
| `clear_session()` / `reconnect()` | 연결 상태 초기화·복구 | 토큰과 시장 캐시 삭제, 재연결 시 새 토큰으로 계좌 조회 |

### 인증 및 오류 처리

- OAuth 2.0 `client_credentials` 방식으로 토큰을 발급받고, 만료 60초 전까지 재사용합니다.
- 토큰 발급에 비동기 잠금을 사용해 동시 요청의 중복 발급을 줄입니다.
- API 조회 요청의 타임아웃은 10초입니다.
- 조회 요청이 HTTP `429`를 반환하면 `Retry-After`를 기준으로 최대 5초 대기한 뒤 한 번 재시도합니다.
- 조회 요청이 HTTP `401` 또는 `403`을 반환하면 저장된 토큰과 시장 캐시를 삭제합니다.
- `TossApiError`에 상태 코드, 요청 ID, 오류 코드를 담습니다. `IP address not allowed` 오류에는 WTS 허용 IP 확인 안내를 붙입니다.

위 내용은 저장소의 클라이언트 구현을 설명하며, 실제 외부 API 연결 검증 결과를 의미하지 않습니다.

## 테스트가 요구하는 기능

다음 기능은 테스트에 기대 동작이 정의되어 있으나, 대응하는 구현 모듈은 현재 저장소에 없습니다.

| 영역 | 테스트에 정의된 기대 동작 |
| --- | --- |
| 종목 전략 | 예산·경고 조건 필터링, 일봉 기반 MA5·MA20·RSI14·거래량 비율 계산 |
| 실시간 추천 | 시장 종목 목록·거래대금 순위·상세정보·경고·일봉 결합, 경고 종목 제외, 종목명 캐시 활용 |
| 모의 자동매매 | 손절·익절·트레일링 스톱·최대 보유 기간, 정규장 판별, 주문 수량 제한, 시장 상태 필터 |
| 거래 및 설정 저장 | 모의 매수와 자동매매 신호 기록, 중복 신호 차단, 실행 중 설정 저장 |
| 포트폴리오 | 거래 내역으로 보유 수량·평균 매입가 계산, 전량 매도 시 보유 종목 제거 |
| 백테스트 | 최소 일봉 수 검증, 익절 거래, 최종 자산·총손익·수익률·최대 낙폭 계산 |
| 일별 성과 | 거래 원장 기반 현금 복원, 전일 성과 보완, 당일 성과 갱신 |

테스트는 SQLite 방식의 연결과 SQL 테이블을 사용하는 저장 계층을 전제로 합니다. 데이터베이스 초기화·스키마 구현은 포함되어 있지 않습니다.

## 프로젝트 구조

```text
ai-vibecoding-2026-main/
├── README.md
├── requirements.txt                # 의존성 목록
├── .env.example                    # 환경 변수 예시
├── .gitignore
└── toss-auto-trader/
    ├── app/
    │   ├── __init__.py
    │   └── toss/
    │       ├── __init__.py          # TossApiError, TossInvestClient 공개
    │       └── client.py           # 비동기 조회 클라이언트
    └── tests/
        ├── test_toss_client.py
        ├── test_strategy.py
        ├── test_recommendation.py
        ├── test_auto_trading.py
        ├── test_portfolio.py
        ├── test_backtest.py
        └── test_performance_history.py
```

루트의 `assets/`, `auto_trader/`, `tests/` 디렉터리는 현재 로컬 작업 폴더에서 비어 있습니다. 실제 소스와 테스트는 `toss-auto-trader/`에 있습니다.

## 개발 환경 준비

소스는 `str | None` 등 Python 3.10 이상 문법을 사용합니다. Python과 `pip`가 설치된 환경에서 저장소 루트를 기준으로 진행합니다.

### Windows PowerShell

```powershell
py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

### macOS / Linux

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
cp .env.example .env
```

`.env`가 이미 있다면 복사 단계를 생략하고 기존 파일을 편집하세요. 위 명령은 개발 환경을 준비하는 단계이며, 누락된 소스 모듈을 생성하지는 않습니다.

### 의존성

| 패키지 | 버전 범위 | 용도 |
| --- | --- | --- |
| `fastapi` | `>=0.115,<1` | 웹 API 구성용 의존성; 현재 서버 구현 없음 |
| `uvicorn[standard]` | `>=0.30,<1` | ASGI 서버 실행용 의존성; 현재 실행 진입점 없음 |
| `pydantic-settings` | `>=2.6,<3` | 설정 관리용 의존성; 현재 설정 모듈 없음 |
| `httpx` | `>=0.27,<1` | 비동기 HTTP 요청 및 테스트용 모의 전송 |
| `pytest` | `>=8,<9` | 테스트 실행 |

## 환경 변수

[`.env.example`](.env.example)에 아래 값이 정의되어 있습니다.

| 변수 | 예시 값 | 의미 |
| --- | --- | --- |
| `TRADING_MODE` | `PAPER` | 모의매매 모드 설정값 |
| `PAPER_INITIAL_CASH` | `10000000` | 모의매매 초기 현금 설정값 |
| `RECOMMENDED_TRADE_RATIO` | `0.10` | 추천 거래 비율 설정값; 적용 로직은 현재 없음 |
| `TOSS_CLIENT_ID` | 빈 값 | 토큰 발급 요청에 사용할 클라이언트 ID |
| `TOSS_CLIENT_SECRET` | 빈 값 | 토큰 발급 요청에 사용할 클라이언트 시크릿 |
| `TOSS_ACCOUNT_SEQ` | 빈 값 | 보유 종목 조회에 사용할 계좌 식별값; 미설정 시 첫 번째 계좌 선택 |

`.env`는 `.gitignore`에 포함되어 있습니다. 실제 인증 정보는 `.env.example`에 넣지 마세요.

현재 `app/config.py`가 없어 `.env` 로딩, 값 검증, 설정 기본값은 확인할 수 없습니다. 클라이언트는 설정 객체에 다음 속성이 있다고 가정합니다.

```text
toss_configured
toss_api_base_url
toss_client_id
toss_client_secret
toss_account_seq
```

테스트에서는 추가로 `database_path`, `paper_initial_cash`, `paper_max_positions`, `paper_max_order_amount` 설정을 사용합니다.

## 테스트 실행

테스트는 Python 표준 라이브러리의 `unittest` 기반이며 `pytest`로도 실행할 수 있습니다. `app` 패키지를 찾도록 **`toss-auto-trader/` 디렉터리에서 실행**합니다.

Windows PowerShell:

```powershell
Set-Location toss-auto-trader
..\.venv\Scripts\python.exe -m unittest discover -s tests -v
# 또는
..\.venv\Scripts\python.exe -m pytest tests -v
```

macOS / Linux:

```bash
cd toss-auto-trader
../.venv/bin/python -m unittest discover -s tests -v
# 또는
../.venv/bin/python -m pytest tests -v
```

조회 클라이언트 테스트는 `httpx.MockTransport`로 응답을 모의하고, 추천·성과 테스트는 가짜 클라이언트를 사용합니다. 해당 테스트의 외부 API 호출에는 실제 인증 정보가 필요하지 않습니다.

### 현재 실행을 막는 누락 파일

아래 모듈을 테스트의 기대 동작에 맞게 구현하거나 복원해야 합니다.

```text
toss-auto-trader/app/config.py
toss-auto-trader/app/database.py
toss-auto-trader/app/strategy.py
toss-auto-trader/app/recommendation.py
toss-auto-trader/app/auto_trading.py
toss-auto-trader/app/portfolio.py
toss-auto-trader/app/backtest.py
toss-auto-trader/app/performance_history.py
```

의존성을 설치해도 누락된 모듈 때문에 테스트 수집·가져오기 단계에서 `ModuleNotFoundError`가 발생할 수 있습니다. 조회 클라이언트도 `app.config`를 가져오기 때문에 해당 설정 모듈이 먼저 필요합니다.

현재 FastAPI 애플리케이션, API 라우트, 서버 실행 진입점, 프런트엔드는 포함되어 있지 않아 서버 실행 명령을 제공할 수 없습니다.

## 개발 시 참고

- 기능을 추가할 때는 `toss-auto-trader/tests/`의 입력 데이터와 기대 결과를 기준으로 구현합니다.
- API 응답 오류는 `TossApiError`의 `status_code`, `request_id`, `error_code`, `is_ip_not_allowed`로 구분할 수 있습니다.
- `reconnect()`는 캐시를 비운 뒤 계좌 조회까지 성공해야 완료됩니다. 단순 토큰 초기화만 필요하면 `clear_session()`을 사용합니다.
- 실주문 기능은 현재 구현 범위에 포함되어 있지 않습니다. 테스트의 자동매매 관련 동작은 모의 거래 저장을 대상으로 합니다.
