# 티켓 발권 백엔드 TRD

## 1. 시스템 개요
- 대상: 발권(티켓 생성) API와 환불 API의 기술 설계
- 저장소: 로컬 SQLite `data/app.db`
- 프레임워크: FastAPI, Python 3.12

## 2. 아키텍처 개요
- 단일 프로세스 Web API
- SQLite 기반 영속화 계층(SQLAlchemy). 트랜잭션으로 정합성 보장.
- 레이어
  - Route Layer: 라우팅/엔드포인트(`route/`)
  - Service Layer: 비즈니스 규칙(발권/환불, 멱등성)(`service/`)
  - Repository/DB Layer: SQLAlchemy 세션/ORM, CRUD(`db/`, `entity/`)
  - Scheme Layer: 요청/응답 Pydantic 스키마(`scheme/`)
  - Util Layer: 공통 유틸(멱등성 캐시 등)(`util/`)

## 3. 주요 컴포넌트
| 파일 | 설명 |
| --- | --- |
| `db/session.py` | SQLAlchemy 엔진/세션 생성, Base 및 `create_all` 제공 |
| `db/repository.py` | 티켓 테이블 CRUD 및 트랜잭션 헬퍼 |
| `entity/ticket.py` | Ticket ORM 엔티티 정의 |
| `scheme/ticket.py` | 발권/환불/조회 요청·응답 Pydantic 스키마 |
| `service/ticket_service.py` | 비즈니스 로직: 발권, 환불, 조회, 멱등성 처리 |
| `route/ticket_route.py` | REST 라우트: `/tickets/issue`, `/tickets/refund`, `/tickets/{ticket_id}`, `/tickets` |
| `util/idempotency.py` | Idempotency-Key 캐시/검증 유틸리티 |
| `app.py` | FastAPI 앱 팩토리 `create_app()`과 라우터 마운트, startup 훅. |
| `__init__.py` | `main()`에서 uvicorn을 factory 모드로 실행(포트 8080 고정). |

## 4. 데이터 모델
- Ticket (ORM/SQL)
  - id: TEXT ({uuid}`)
  - theater_name: TEXT
  - user_id: TEXT
  - movie_title: TEXT
  - price_krw: INTEGER
  - status: TEXT ("issued" | "canceled")
  - issued_at: DATETIME(UTC)
  - canceled_at: DATETIME(UTC) | NULL
  - memo: TEXT | NULL

## 5. API 설계
### 5.1 발권 API
   - Method/Path: POST `/tickets/issue`
   - Headers:
     - Content-Type: application/json
     - Idempotency-Key: string (optional)
   - Request JSON:
     - theater_name: string
     - movie_title: string
     - price_krw: int
     - quantity: int (optional, default 1)
     - memo: string (optional)
   - Response:
     - 201 Created
     - Body:
       - ticket_ids: string[]
       - count: int
       - summary: { theater_name, movie_title, price_krw }
   - Errors:
     - 400: 입력 유효성 실패
     - 409: 멱등성 충돌
     - 500: DB 쓰기 실패

### 5.2 환불 API
   - Method/Path: POST `/tickets/refund`
   - Headers:
     - Content-Type: application/json
   - Request JSON:
     - ticket_ids: string[]
     - reason: string (optional)
   - 처리:
     - 존재하는 티켓 중 status=issued인 항목만 canceled로 전환
     - 이미 canceled는 `already_canceled`에 분류
     - 없는 ID는 `not_found`에 분류
   - Response:
     - 200 OK
     - Body:
       - refunded: string[]
       - already_canceled: string[]
       - not_found: string[]
   - Errors:
     - 400: 입력 유효성 실패
     - 500: DB 쓰기 실패

### 5.3 티켓 조회 API
#### 5.3.1 단일 티켓 조회
   - Method/Path: GET `/tickets/{ticket_id}`
   - Path Parameters:
     - ticket_id: string (티켓 ID)
   - Response:
     - 200 OK
     - Body:
       - id: string
       - theater_name: string
       - user_id : string
       - movie_title: string
       - price_krw: int
       - status: string ("issued" | "canceled")
       - issued_at: string (ISO 8601)
       - canceled_at: string | null (ISO 8601)
       - memo: string | null
   - Errors:
     - 404: 티켓이 존재하지 않음
     - 500: DB 읽기 실패

#### 5.3.2 티켓 목록 조회
   - Method/Path: GET `/tickets`
   - Query Parameters (모두 선택적):
     - theater_name: string (극장명 필터)
     - user_id : string
     - movie_title: string (영화명 필터)
     - status: string ("issued" | "canceled", 상태 필터)
     - limit: int (최대 조회 개수, 기본값: 100, 최대: 1000)
     - offset: int (페이징 오프셋, 기본값: 0)
   - Response:
     - 200 OK
     - Body:
       - tickets: array (티켓 객체 배열, 각 객체는 5.3.1과 동일한 구조)
       - total: int (필터 조건에 맞는 전체 티켓 개수)
       - limit: int (적용된 limit)
       - offset: int (적용된 offset)
   - Errors:
     - 400: 쿼리 파라미터 유효성 실패
     - 500: DB 읽기 실패

### 5.4 상태 전이
- issued → canceled (환불)
- canceled 상태는 재환불 요청 시 변경 없음(무해 처리)

## 6. 예외 및 검증 정책
- theater_name: 1~100자
- movie_title: 1~200자
- price_krw: 정수, 0 < price_krw ≤ 1,000,000
- quantity: 정수, 1 ≤ quantity ≤ 10
- ticket_ids: 비어있지 않은 배열, 각 원소는 문자열

## 7. 동시성 및 저장 전략
- 경로: `data/app.db`
- 초기화: 앱 시작 시 `Base.metadata.create_all(bind=engine)`로 테이블 생성
- 쓰기 전략: 세션 단위 트랜잭션, 실패 시 롤백

### 7.1 멱등성 전략
- 요청 헤더 `Idempotency-Key`(선택)
- Service에서 `idem:{key}` → 최근 동일 요청 해시와 응답 캐시를 메모리로 유지(util/idempotency.py)
- 동일 키에 다른 요청 해시가 오면 409 Conflict
- 파일 기반만 요구될 경우, 간단 구현: 생략 가능(비권장)

## 8. 배포/실행
- 로컬 실행: `python app.py`
- 호스트/포트: `0.0.0.0:9000`

## 8.1 API 문서(FASTAPI)
- Swagger UI: `http://localhost:9000/docs`
- OpenAPI JSON: `http://localhost:9000/openapi.json`

## 9. 로깅/모니터링
- 응답 코드, 소요 시간(ms), 생성/환불 개수
- DB 예외 시 에러 로그 및 롤백

## 10. 테스트 전략
- 발권: 단건/복수 발권 성공, 필드 누락/유효성 실패
- 환불: 존재/미존재/이미 취소 혼합 케이스
- 조회: 단일 티켓 조회 성공/실패(404), 목록 조회 필터링/페이징 테스트
- 멱등성: 동일 키 재시도 시 응답 재사용, 상이한 본문 충돌
- 동시성(선택): 단일 프로세스 내 동시 요청 시 데이터 정합성 확인

## 11. 폴더/모듈 구조
```
src/movie_ticket_backend/
  __init__.py
  app.py
  db/
    session.py           # 엔진/세션/Base
    repository.py        # 티켓 리포지토리
  entity/
    ticket.py            # SQLAlchemy ORM 엔티티
  scheme/
    ticket.py            # Pydantic 요청/응답 스키마
  service/
    ticket_service.py    # 비즈니스 로직
  route/
    __init__.py          # FastAPI 앱/라우터 등록
    ticket_route.py      # 엔드포인트 구현
  util/
    idempotency.py       # 멱등성 캐시 유틸
data/
  app.db                 # SQLite DB 파일
```