\# BagDrop DB 스키마 설계



> 작성: 기획 에이전트 여울

> 세션: Session 04

> 최종 업데이트: 2026-07-01

> DB: Supabase (PostgreSQL 15)

> 상태: ✅ Session 04 확정



\---



\## 1. 테이블 목록



| 테이블명 | 역할 | 비고 |

|----------|------|------|

| `reservations` | 고객 예약 정보 전체 (핵심 테이블) | FO 예약 플로우 전체 |

| `payments` | 결제 정보 (토스페이 연동) | 예약 1건 = 결제 1건 |

| `reservation\_logs` | 예약 상태 변경 이력 | 접수/반환/취소 시각 추적 |

| `sms\_logs` | SMS 발송 이력 | 알리고 API 발송 기록 |

| `admin\_users` | BO 운영자 계정 | NextAuth.js 연동 |



\---



\## 2. Enum 타입 정의



```sql

\-- 예약 상태

CREATE TYPE reservation\_status AS ENUM (

&#x20; 'RESERVED',   -- 예약완료 (결제 완료, 미접수)

&#x20; 'STORED',     -- 보관중 (운영자 접수 완료)

&#x20; 'RETURNED',   -- 반환완료 (운영자 반환 완료)

&#x20; 'CANCELLED'   -- 취소완료 (고객 취소 또는 운영자 강제 취소)

);



\-- 터미널 구분

CREATE TYPE terminal\_type AS ENUM (

&#x20; 'T1',  -- 제1여객터미널

&#x20; 'T2'   -- 제2여객터미널

);



\-- 이용 방향

CREATE TYPE travel\_direction AS ENUM (

&#x20; 'DEPARTURE',  -- 출국

&#x20; 'ARRIVAL'     -- 귀국

);



\-- 취소 주체

CREATE TYPE canceller\_type AS ENUM (

&#x20; 'CUSTOMER',   -- 고객 직접 취소

&#x20; 'ADMIN'       -- 운영자 강제 취소

);



\-- 결제 상태

CREATE TYPE payment\_status AS ENUM (

&#x20; 'PENDING',    -- 결제 진행 중

&#x20; 'PAID',       -- 결제 완료

&#x20; 'CANCELLED',  -- 결제 취소 (환불 포함)

&#x20; 'FAILED'      -- 결제 실패

);



\-- SMS 발송 이벤트 유형

CREATE TYPE sms\_event\_type AS ENUM (

&#x20; 'RESERVATION\_COMPLETE',  -- 예약 완료

&#x20; 'CHECKIN\_COMPLETE',      -- 짐 접수 완료

&#x20; 'CHECKOUT\_COMPLETE',     -- 짐 반환 완료

&#x20; 'CANCELLATION\_COMPLETE'  -- 취소 완료

);



\-- SMS 발송 결과

CREATE TYPE sms\_result AS ENUM (

&#x20; 'SUCCESS',  -- 발송 성공

&#x20; 'FAILED'    -- 발송 실패

);

```



\---



\## 3. 테이블 상세 정의



\### 3-1. `reservations` — 예약



```sql

CREATE TABLE reservations (

&#x20; -- 식별자

&#x20; id              UUID          PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_no  VARCHAR(20)   NOT NULL UNIQUE,  -- BD-YYYYMMDD-NNNN 형식



&#x20; -- 예약 정보

&#x20; terminal        terminal\_type     NOT NULL,  -- T1 / T2

&#x20; direction       travel\_direction  NOT NULL,  -- DEPARTURE / ARRIVAL

&#x20; use\_date        DATE              NOT NULL,  -- 이용일 (YYYY-MM-DD)

&#x20; bag\_count       SMALLINT          NOT NULL CHECK (bag\_count BETWEEN 1 AND 2),



&#x20; -- 고객 정보

&#x20; customer\_name   VARCHAR(20)   NOT NULL,

&#x20; customer\_phone  VARCHAR(11)   NOT NULL,  -- 숫자만 저장 (01012345678)



&#x20; -- 요금

&#x20; amount          INTEGER       NOT NULL DEFAULT 5000 CHECK (amount = 5000),



&#x20; -- 상태

&#x20; status          reservation\_status  NOT NULL DEFAULT 'RESERVED',

&#x20; cancelled\_by    canceller\_type      NULL,     -- 취소 주체 (취소 시에만)

&#x20; cancel\_reason   TEXT                NULL,     -- 취소 사유 (선택)



&#x20; -- 운영 처리 시각

&#x20; checked\_in\_at   TIMESTAMPTZ   NULL,  -- 짐 접수 완료 시각 (STORED 전환 시)

&#x20; checked\_out\_at  TIMESTAMPTZ   NULL,  -- 짐 반환 완료 시각 (RETURNED 전환 시)

&#x20; cancelled\_at    TIMESTAMPTZ   NULL,  -- 취소 처리 시각



&#x20; -- 메타

&#x20; created\_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),

&#x20; updated\_at      TIMESTAMPTZ   NOT NULL DEFAULT now()

);

```



\#### 컬럼 상세



| 컬럼명 | 타입 | 제약조건 | 설명 |

|--------|------|----------|------|

| `id` | UUID | PK, DEFAULT gen\_random\_uuid() | 내부 식별자 |

| `reservation\_no` | VARCHAR(20) | UNIQUE, NOT NULL | 고객 노출 예약번호. BD-YYYYMMDD-NNNN |

| `terminal` | terminal\_type | NOT NULL | T1 / T2 |

| `direction` | travel\_direction | NOT NULL | 출국 / 귀국 |

| `use\_date` | DATE | NOT NULL | 이용일. 취소 가능 여부 판단 기준 |

| `bag\_count` | SMALLINT | NOT NULL, CHECK 1\~2 | 짐 개수 (최대 2개 정책) |

| `customer\_name` | VARCHAR(20) | NOT NULL | 예약자 이름 |

| `customer\_phone` | VARCHAR(11) | NOT NULL | 숫자만 저장. 표시 시 포맷 변환 |

| `amount` | INTEGER | NOT NULL, CHECK = 5000 | 결제 금액. 현재 5,000원 고정 |

| `status` | reservation\_status | NOT NULL, DEFAULT 'RESERVED' | 현재 예약 상태 |

| `cancelled\_by` | canceller\_type | NULL | 취소 시 주체 기록 |

| `cancel\_reason` | TEXT | NULL | 취소 사유 (선택 항목) |

| `checked\_in\_at` | TIMESTAMPTZ | NULL | 접수 완료 시각. STORED 전환 시 기록 |

| `checked\_out\_at` | TIMESTAMPTZ | NULL | 반환 완료 시각. RETURNED 전환 시 기록 |

| `cancelled\_at` | TIMESTAMPTZ | NULL | 취소 처리 시각 |

| `created\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 예약 생성 시각 (결제 완료 시각) |

| `updated\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 최종 수정 시각 |



\#### 예약번호 채번 규칙



```

BD-YYYYMMDD-NNNN

&#x20; └─ BD      : 서비스 접두어 고정

&#x20; └─ YYYYMMDD: 예약 생성일 (Asia/Seoul 기준 KST 날짜)

&#x20; └─ NNNN    : 날짜별 독립 Sequence에서 발급한 일련번호 (0001부터 시작, 4자리 zero-padding)



예시: BD-20260715-0001

```



> \*\*채번 방식: 날짜별 PostgreSQL Sequence\*\*

>

> PostgreSQL 네이티브 Sequence를 사용한다. Sequence는 트랜잭션 롤백과 무관하게 원자적으로 증가하므로 동시 요청이 발생해도 중복 없이 채번된다. SELECT MAX+1 방식은 동시 트랜잭션에서 같은 값을 읽을 수 있어(Race Condition) 사용하지 않는다.

>

> \*\*구현 방식 — 날짜별 동적 Sequence 생성\*\*

>

> ```sql

> -- 날짜별 Sequence 이름 규칙: reservation\_seq\_YYYYMMDD

> -- 예: reservation\_seq\_20260715

>

> -- Sequence가 없으면 생성, 있으면 skip (멱등 처리)

> CREATE SEQUENCE IF NOT EXISTS reservation\_seq\_20260715

>   START WITH 1

>   INCREMENT BY 1

>   NO CYCLE;

>

> -- 다음 번호 발급 (원자적, 잠금 없음)

> SELECT nextval('reservation\_seq\_20260715');

> -- → 1, 2, 3, ... 순서 보장

> ```

>

> \*\*애플리케이션 레이어 처리 흐름 (Next.js API Route)\*\*

>

> ```

> 1. KST 기준 오늘 날짜 문자열 생성 → '20260715'

> 2. Supabase RPC 호출: get\_next\_reservation\_no('20260715')

>    ├─ Sequence 미존재 시 CREATE SEQUENCE IF NOT EXISTS

>    └─ nextval() 실행 → 정수 반환

> 3. 반환값을 4자리 zero-padding → '0003'

> 4. 조합: 'BD-20260715-0003'

> 5. reservations INSERT reservation\_no = 'BD-20260715-0003'

> ```

>

> \*\*Supabase RPC 함수 (DB 레이어 캡슐화)\*\*

>

> ```sql

> CREATE OR REPLACE FUNCTION get\_next\_reservation\_no(target\_date TEXT)

> RETURNS TEXT

> LANGUAGE plpgsql

> AS $$

> DECLARE

>   seq\_name TEXT;

>   next\_val BIGINT;

>   date\_part TEXT;

> BEGIN

>   -- 날짜 형식 검증 (YYYYMMDD 8자리)

>   IF target\_date !\~ '^\[0-9]{8}$' THEN

>     RAISE EXCEPTION 'Invalid date format: %', target\_date;

>   END IF;

>

>   seq\_name := 'reservation\_seq\_' || target\_date;

>

>   -- Sequence 없으면 생성 (멱등)

>   EXECUTE format(

>     'CREATE SEQUENCE IF NOT EXISTS %I START WITH 1 INCREMENT BY 1 NO CYCLE',

>     seq\_name

>   );

>

>   -- 원자적으로 다음 번호 발급

>   EXECUTE format('SELECT nextval(%L)', seq\_name) INTO next\_val;

>

>   -- BD-YYYYMMDD-NNNN 형식으로 조합

>   RETURN 'BD-' || target\_date || '-' || LPAD(next\_val::TEXT, 4, '0');

> END;

> $$;

> ```

>

> \*\*사용 예시\*\*

>

> ```sql

> SELECT get\_next\_reservation\_no('20260715');

> -- 결과: 'BD-20260715-0001'

>

> SELECT get\_next\_reservation\_no('20260715');

> -- 결과: 'BD-20260715-0002'  (동시 호출도 안전)

> ```

>

> \*\*Sequence 설계 선택 이유\*\*

>

> | 방식 | 동시성 안전 | 번호 공백 | 구현 복잡도 | 채택 |

> |------|-----------|---------|------------|------|

> | SELECT MAX+1 (애플리케이션) | ❌ Race Condition | 없음 | 낮음 | ✗ |

> | SELECT FOR UPDATE (잠금) | ✅ | 없음 | 중간 | ✗ |

> | \*\*PostgreSQL Sequence\*\* | \*\*✅ 원자적\*\* | \*\*있음 (롤백 시)\*\* | \*\*낮음\*\* | \*\*✅\*\* |

>

> 번호 공백(예: 0003 다음 0005)은 결제 실패/롤백 시 발생할 수 있으나 서비스 운영상 무해하다. 예약번호는 순번 연속성보다 유일성이 핵심이므로 Sequence가 최적.



\---



\### 3-2. `payments` — 결제



```sql

CREATE TABLE payments (

&#x20; -- 식별자

&#x20; id                  UUID          PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_id      UUID          NOT NULL REFERENCES reservations(id) ON DELETE RESTRICT,



&#x20; -- 토스페이 연동 키

&#x20; toss\_payment\_key    VARCHAR(200)  NOT NULL UNIQUE,  -- 토스페이 paymentKey

&#x20; toss\_order\_id       VARCHAR(64)   NOT NULL UNIQUE,  -- 토스페이 orderId (결제 요청 시 생성)



&#x20; -- 결제 정보

&#x20; amount              INTEGER       NOT NULL CHECK (amount = 5000),

&#x20; method              VARCHAR(50)   NOT NULL,   -- 'TOSSPAY' (단일화)

&#x20; status              payment\_status  NOT NULL DEFAULT 'PENDING',



&#x20; -- 토스페이 응답 원본

&#x20; toss\_response       JSONB         NULL,       -- 결제 승인 응답 전체 저장



&#x20; -- 취소 정보

&#x20; cancelled\_amount    INTEGER       NULL,       -- 취소된 금액 (환불 완료 시)

&#x20; cancelled\_at        TIMESTAMPTZ   NULL,       -- 취소 처리 시각



&#x20; -- 메타

&#x20; paid\_at             TIMESTAMPTZ   NULL,       -- 결제 완료 시각

&#x20; created\_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),

&#x20; updated\_at          TIMESTAMPTZ   NOT NULL DEFAULT now()

);

```



\#### 컬럼 상세



| 컬럼명 | 타입 | 제약조건 | 설명 |

|--------|------|----------|------|

| `id` | UUID | PK | 내부 결제 식별자 |

| `reservation\_id` | UUID | FK → reservations(id), NOT NULL | 연결된 예약 |

| `toss\_payment\_key` | VARCHAR(200) | UNIQUE, NOT NULL | 토스페이 결제 고유키. 환불 API 호출 시 사용 |

| `toss\_order\_id` | VARCHAR(64) | UNIQUE, NOT NULL | 토스페이 주문번호. 결제창 호출 시 생성 |

| `amount` | INTEGER | NOT NULL, CHECK = 5000 | 결제 금액 |

| `method` | VARCHAR(50) | NOT NULL | 결제 수단. 'TOSSPAY' 고정 |

| `status` | payment\_status | NOT NULL, DEFAULT 'PENDING' | 결제 상태 |

| `toss\_response` | JSONB | NULL | 토스페이 결제 승인 응답 원본. 분쟁 대비 보존 |

| `cancelled\_amount` | INTEGER | NULL | 환불 금액. 토스페이 콘솔 수동 처리 후 업데이트 |

| `cancelled\_at` | TIMESTAMPTZ | NULL | 환불 처리 시각 |

| `paid\_at` | TIMESTAMPTZ | NULL | 결제 완료 시각 |

| `created\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 결제 레코드 생성 시각 |

| `updated\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 최종 수정 시각 |



> \*\*결제-예약 생성 순서\*\*

> 1. FO에서 결제창 호출 전 `toss\_order\_id` 생성 → `payments` 레코드 INSERT (status: PENDING)

> 2. 토스페이 결제 승인 콜백 → `payments` 업데이트 (status: PAID, toss\_response 저장)

> 3. `reservations` INSERT (status: RESERVED) → SMS 발송

> 4. 결제 실패/취소 → `payments` 업데이트 (status: FAILED/CANCELLED), `reservations` 미생성



\---



\### 3-3. `reservation\_logs` — 상태 변경 이력



```sql

CREATE TABLE reservation\_logs (

&#x20; id              UUID                  PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_id  UUID                  NOT NULL REFERENCES reservations(id) ON DELETE CASCADE,



&#x20; -- 상태 변경 내용

&#x20; from\_status     reservation\_status    NULL,   -- 변경 전 상태 (최초 생성 시 NULL)

&#x20; to\_status       reservation\_status    NOT NULL,  -- 변경 후 상태



&#x20; -- 처리 주체

&#x20; actor\_type      VARCHAR(20)           NOT NULL,  -- 'CUSTOMER' / 'ADMIN' / 'SYSTEM'

&#x20; actor\_id        UUID                  NULL,      -- admin\_users.id (운영자 처리 시)

&#x20; actor\_label     VARCHAR(50)           NULL,      -- 표시용 레이블 ('고객 직접 취소', '운영자 강제 취소' 등)



&#x20; -- 메모

&#x20; note            TEXT                  NULL,



&#x20; created\_at      TIMESTAMPTZ           NOT NULL DEFAULT now()

);

```



\#### 컬럼 상세



| 컬럼명 | 타입 | 제약조건 | 설명 |

|--------|------|----------|------|

| `id` | UUID | PK | 이력 식별자 |

| `reservation\_id` | UUID | FK → reservations(id), CASCADE | 대상 예약 |

| `from\_status` | reservation\_status | NULL | 변경 전 상태. 최초 RESERVED 생성 시 NULL |

| `to\_status` | reservation\_status | NOT NULL | 변경 후 상태 |

| `actor\_type` | VARCHAR(20) | NOT NULL | 처리 주체 유형 |

| `actor\_id` | UUID | NULL | 운영자 처리 시 admin\_users.id 기록 |

| `actor\_label` | VARCHAR(50) | NULL | 화면 표시용 레이블 |

| `note` | TEXT | NULL | 비고 메모 |

| `created\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 이력 기록 시각 |



\#### 이력 적재 시점



| 이벤트 | from\_status | to\_status | actor\_type |

|--------|-------------|-----------|------------|

| 예약 생성 (결제 완료) | NULL | RESERVED | SYSTEM |

| 짐 접수 | RESERVED | STORED | ADMIN |

| 짐 반환 | STORED | RETURNED | ADMIN |

| 고객 취소 | RESERVED | CANCELLED | CUSTOMER |

| 운영자 강제 취소 | RESERVED | CANCELLED | ADMIN |



\---



\### 3-4. `sms\_logs` — SMS 발송 이력



```sql

CREATE TABLE sms\_logs (

&#x20; id              UUID              PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_id  UUID              NOT NULL REFERENCES reservations(id) ON DELETE CASCADE,



&#x20; -- 발송 정보

&#x20; event\_type      sms\_event\_type    NOT NULL,  -- 발송 이벤트 유형

&#x20; recipient\_phone VARCHAR(11)       NOT NULL,  -- 수신 번호 (숫자만)

&#x20; message\_content TEXT              NOT NULL,  -- 실제 발송된 메시지 내용



&#x20; -- 발송 결과

&#x20; result          sms\_result        NOT NULL DEFAULT 'SUCCESS',

&#x20; aligo\_msg\_id    VARCHAR(50)       NULL,      -- 알리고 API 응답 메시지 ID

&#x20; error\_message   TEXT              NULL,      -- 실패 시 오류 내용



&#x20; created\_at      TIMESTAMPTZ       NOT NULL DEFAULT now()

);

```



\#### 컬럼 상세



| 컬럼명 | 타입 | 제약조건 | 설명 |

|--------|------|----------|------|

| `id` | UUID | PK | 발송 이력 식별자 |

| `reservation\_id` | UUID | FK → reservations(id), CASCADE | 대상 예약 |

| `event\_type` | sms\_event\_type | NOT NULL | 발송 이벤트 유형 |

| `recipient\_phone` | VARCHAR(11) | NOT NULL | 수신 번호 |

| `message\_content` | TEXT | NOT NULL | 실제 발송 메시지 내용 (디버깅 및 분쟁 대비) |

| `result` | sms\_result | NOT NULL, DEFAULT 'SUCCESS' | 발송 결과 |

| `aligo\_msg\_id` | VARCHAR(50) | NULL | 알리고 메시지 ID (성공 시) |

| `error\_message` | TEXT | NULL | 실패 사유 (실패 시) |

| `created\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 발송 시각 |



\---



\### 3-5. `admin\_users` — 운영자 계정



```sql

CREATE TABLE admin\_users (

&#x20; id              UUID          PRIMARY KEY DEFAULT gen\_random\_uuid(),



&#x20; -- 로그인 정보

&#x20; username        VARCHAR(50)   NOT NULL UNIQUE,

&#x20; password\_hash   VARCHAR(255)  NOT NULL,  -- bcrypt hash



&#x20; -- 표시 정보

&#x20; display\_name    VARCHAR(50)   NOT NULL,



&#x20; -- 계정 상태

&#x20; is\_active       BOOLEAN       NOT NULL DEFAULT true,

&#x20; last\_login\_at   TIMESTAMPTZ   NULL,

&#x20; login\_fail\_count  SMALLINT    NOT NULL DEFAULT 0,

&#x20; locked\_until    TIMESTAMPTZ   NULL,  -- 5회 실패 시 잠금 해제 시각



&#x20; -- 메타

&#x20; created\_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),

&#x20; updated\_at      TIMESTAMPTZ   NOT NULL DEFAULT now()

);

```



\#### 컬럼 상세



| 컬럼명 | 타입 | 제약조건 | 설명 |

|--------|------|----------|------|

| `id` | UUID | PK | 운영자 식별자 |

| `username` | VARCHAR(50) | UNIQUE, NOT NULL | 로그인 아이디 |

| `password\_hash` | VARCHAR(255) | NOT NULL | bcrypt 해시 (NextAuth.js Credentials Provider) |

| `display\_name` | VARCHAR(50) | NOT NULL | GNB 표시용 이름 |

| `is\_active` | BOOLEAN | NOT NULL, DEFAULT true | 계정 활성 여부 |

| `last\_login\_at` | TIMESTAMPTZ | NULL | 마지막 로그인 시각 |

| `login\_fail\_count` | SMALLINT | NOT NULL, DEFAULT 0 | 연속 로그인 실패 횟수 |

| `locked\_until` | TIMESTAMPTZ | NULL | 계정 잠금 해제 시각. NULL이면 잠금 없음 |

| `created\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 계정 생성 시각 |

| `updated\_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 최종 수정 시각 |



\---



\## 4. 테이블 관계 (ERD 텍스트)



```

┌─────────────────────────────────────────────────────────────────┐

│                         reservations                            │

│                                                                 │

│  id (PK)                                                        │

│  reservation\_no (UNIQUE)                                        │

│  terminal / direction / use\_date / bag\_count                    │

│  customer\_name / customer\_phone                                 │

│  amount / status                                                │

│  checked\_in\_at / checked\_out\_at / cancelled\_at                 │

│  created\_at / updated\_at                                        │

└──────────────────┬──────────────────────────────────────────────┘

&#x20;                  │ 1

&#x20;                  │

&#x20;      ┌───────────┼────────────────────┐

&#x20;      │           │                    │

&#x20;      │ N         │ N                  │ N

&#x20;      ▼           ▼                    ▼

┌──────────────┐  ┌────────────────┐  ┌─────────────┐

│   payments   │  │reservation\_logs│  │  sms\_logs   │

│              │  │                │  │             │

│ id (PK)      │  │ id (PK)        │  │ id (PK)     │

│ reservation\_ │  │ reservation\_id │  │ reservation │

│   id (FK)    │  │   (FK)         │  │   \_id (FK)  │

│ toss\_payment │  │ from\_status    │  │ event\_type  │

│   \_key       │  │ to\_status      │  │ recipient\_  │

│ toss\_order   │  │ actor\_type     │  │   phone     │

│   \_id        │  │ actor\_id (FK)──┼──┤ result      │

│ amount       │  │ actor\_label    │  │ created\_at  │

│ status       │  │ created\_at     │  └─────────────┘

│ toss\_response│  └────────────────┘

│ paid\_at      │          │ actor\_id (FK, NULL)

└──────────────┘          │

&#x20;                         │ N

&#x20;                         ▼

&#x20;                ┌─────────────────┐

&#x20;                │   admin\_users   │

&#x20;                │                 │

&#x20;                │ id (PK)         │

&#x20;                │ username        │

&#x20;                │ password\_hash   │

&#x20;                │ display\_name    │

&#x20;                │ is\_active       │

&#x20;                │ login\_fail\_count│

&#x20;                │ locked\_until    │

&#x20;                └─────────────────┘



관계 요약:

&#x20; reservations 1 ─── N payments          (예약 1건 : 결제 이력 N건, 실질적 1:1)

&#x20; reservations 1 ─── N reservation\_logs  (예약 1건 : 상태 이력 N건)

&#x20; reservations 1 ─── N sms\_logs          (예약 1건 : SMS 발송 이력 N건)

&#x20; admin\_users  1 ─── N reservation\_logs  (운영자 1명 : 처리 이력 N건, nullable)

```



\---



\## 5. 인덱스 설계



```sql

\-- reservations

CREATE UNIQUE INDEX idx\_reservations\_reservation\_no

&#x20; ON reservations (reservation\_no);

\-- 역할: QR 스캔/예약번호 조회 핵심 경로. 유일성 보장.



CREATE INDEX idx\_reservations\_use\_date\_status

&#x20; ON reservations (use\_date, status);

\-- 역할: 대시보드 "오늘 현황" 카운트 조회 (use\_date = today AND status = ...)



CREATE INDEX idx\_reservations\_status

&#x20; ON reservations (status);

\-- 역할: 보관현황 목록 (status = 'STORED') 조회



CREATE INDEX idx\_reservations\_terminal\_use\_date

&#x20; ON reservations (terminal, use\_date);

\-- 역할: 터미널 필터 + 날짜 필터 조합 조회 (예약 관리, 매출 관리)



CREATE INDEX idx\_reservations\_created\_at

&#x20; ON reservations (created\_at DESC);

\-- 역할: 대시보드 최근 접수 대기 목록 정렬



\-- payments

CREATE UNIQUE INDEX idx\_payments\_toss\_payment\_key

&#x20; ON payments (toss\_payment\_key);

\-- 역할: 토스페이 웹훅 수신 시 paymentKey로 조회



CREATE UNIQUE INDEX idx\_payments\_toss\_order\_id

&#x20; ON payments (toss\_order\_id);

\-- 역할: 결제 승인 콜백에서 orderId로 조회



CREATE INDEX idx\_payments\_reservation\_id

&#x20; ON payments (reservation\_id);

\-- 역할: 예약 상세에서 결제 정보 조회



\-- reservation\_logs

CREATE INDEX idx\_reservation\_logs\_reservation\_id

&#x20; ON reservation\_logs (reservation\_id, created\_at ASC);

\-- 역할: 예약 상세 처리 이력 시간순 조회



\-- sms\_logs

CREATE INDEX idx\_sms\_logs\_reservation\_id

&#x20; ON sms\_logs (reservation\_id);

\-- 역할: 예약별 SMS 발송 이력 조회



\-- admin\_users

CREATE UNIQUE INDEX idx\_admin\_users\_username

&#x20; ON admin\_users (username);

\-- 역할: 로그인 시 username 조회

```



\---



\## 6. updated\_at 자동 갱신 트리거



```sql

\-- 공통 트리거 함수

CREATE OR REPLACE FUNCTION update\_updated\_at\_column()

RETURNS TRIGGER AS $$

BEGIN

&#x20; NEW.updated\_at = now();

&#x20; RETURN NEW;

END;

$$ LANGUAGE plpgsql;



\-- reservations

CREATE TRIGGER trg\_reservations\_updated\_at

&#x20; BEFORE UPDATE ON reservations

&#x20; FOR EACH ROW EXECUTE FUNCTION update\_updated\_at\_column();



\-- payments

CREATE TRIGGER trg\_payments\_updated\_at

&#x20; BEFORE UPDATE ON payments

&#x20; FOR EACH ROW EXECUTE FUNCTION update\_updated\_at\_column();



\-- admin\_users

CREATE TRIGGER trg\_admin\_users\_updated\_at

&#x20; BEFORE UPDATE ON admin\_users

&#x20; FOR EACH ROW EXECUTE FUNCTION update\_updated\_at\_column();

```



\---



\## 7. 주요 비즈니스 로직 제약



```sql

\-- 예약번호 형식 검증

ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_reservation\_no\_format

&#x20; CHECK (reservation\_no \~ '^BD-\[0-9]{8}-\[0-9]{4}$');



\-- 이용일은 과거 날짜 불가 (생성 시점 기준, 당일 이후만 허용)

\-- ※ 애플리케이션 레이어에서 검증 (DB 레벨은 미적용 - 배치/시드 데이터 예외 허용)



\-- 접수 시각은 STORED 상태일 때만 존재

ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_checked\_in\_at

&#x20; CHECK (

&#x20;   (status = 'STORED' AND checked\_in\_at IS NOT NULL) OR

&#x20;   (status != 'STORED')

&#x20; );



\-- 반환 시각은 RETURNED 상태일 때만 존재

ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_checked\_out\_at

&#x20; CHECK (

&#x20;   (status = 'RETURNED' AND checked\_out\_at IS NOT NULL) OR

&#x20;   (status != 'RETURNED')

&#x20; );



\-- 취소 주체는 CANCELLED 상태일 때만 존재

ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_cancelled\_by

&#x20; CHECK (

&#x20;   (status = 'CANCELLED' AND cancelled\_by IS NOT NULL) OR

&#x20;   (status != 'CANCELLED')

&#x20; );

```



\---



\## 8. Row Level Security (RLS) — Supabase 설정



```sql

\-- reservations: 공개 조회 허용 (예약번호 기반 고객 조회)

ALTER TABLE reservations ENABLE ROW LEVEL SECURITY;



CREATE POLICY "고객: 예약번호로 본인 예약 조회"

&#x20; ON reservations FOR SELECT

&#x20; USING (true);

\-- ※ 실제 접근 제어는 API 레이어(Next.js)에서 reservation\_no 파라미터로 처리.

\--   DB 레벨은 anon key 노출 대비 최소 제한.



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON reservations FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- payments: 고객 직접 접근 불가 (서비스 롤만 접근)

ALTER TABLE payments ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON payments FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- reservation\_logs: 서비스 롤만 접근

ALTER TABLE reservation\_logs ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON reservation\_logs FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- sms\_logs: 서비스 롤만 접근

ALTER TABLE sms\_logs ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON sms\_logs FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- admin\_users: 서비스 롤만 접근

ALTER TABLE admin\_users ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON admin\_users FOR ALL

&#x20; USING (auth.role() = 'service\_role');

```



> \*\*설계 원칙\*\*: 모든 DB 접근은 Next.js API Routes를 통해 `service\_role` 키로만 처리.

> FO/BO 클라이언트에는 Supabase `anon` 키를 노출하지 않음.



\---



\## 9. 초기 데이터 (Seed)



\### 9-1. 운영자 계정



```sql

\-- 비밀번호: bagdrop2026! (배포 전 반드시 변경)

\-- bcrypt 해시 (rounds=12): $2b$12$...

INSERT INTO admin\_users (

&#x20; id,

&#x20; username,

&#x20; password\_hash,

&#x20; display\_name,

&#x20; is\_active

) VALUES (

&#x20; gen\_random\_uuid(),

&#x20; 'admin',

&#x20; '$2b$12$여기에\_실제\_bcrypt\_해시\_입력',  -- 배포 시 교체 필수

&#x20; '관리자',

&#x20; true

);

```



> ⚠️ \*\*보안 주의\*\*: `password\_hash` 값은 배포 전 반드시 실제 해시값으로 교체.

> 초기 비밀번호는 환경변수 `ADMIN\_INITIAL\_PASSWORD`로 관리하고 시드 스크립트 실행 시 동적 해싱 권장.



\### 9-2. 시드 스크립트 (Node.js)



```javascript

// scripts/seed.js

const bcrypt = require('bcrypt');

const { createClient } = require('@supabase/supabase-js');



const supabase = createClient(

&#x20; process.env.NEXT\_PUBLIC\_SUPABASE\_URL,

&#x20; process.env.SUPABASE\_SERVICE\_ROLE\_KEY

);



async function seed() {

&#x20; const password = process.env.ADMIN\_INITIAL\_PASSWORD || 'bagdrop2026!';

&#x20; const passwordHash = await bcrypt.hash(password, 12);



&#x20; const { error } = await supabase

&#x20;   .from('admin\_users')

&#x20;   .upsert({

&#x20;     username: 'admin',

&#x20;     password\_hash: passwordHash,

&#x20;     display\_name: '관리자',

&#x20;     is\_active: true,

&#x20;   }, { onConflict: 'username' });



&#x20; if (error) {

&#x20;   console.error('Seed 실패:', error);

&#x20;   process.exit(1);

&#x20; }



&#x20; console.log('✅ 운영자 계정 시드 완료');

}



seed();

```



\---



\## 10. 주요 쿼리 예시



\### 10-1. 오늘 현황 카운트 (대시보드)



```sql

\-- 오늘 예약 건수 (터미널 필터 적용)

SELECT COUNT(\*) AS today\_count

FROM reservations

WHERE use\_date = CURRENT\_DATE

&#x20; AND status IN ('RESERVED', 'STORED', 'RETURNED')

&#x20; AND terminal = 'T1';  -- 전체 조회 시 이 조건 제거



\-- 현재 보관 중 건수

SELECT COUNT(\*) AS stored\_count

FROM reservations

WHERE status = 'STORED'

&#x20; AND terminal = 'T1';



\-- 오늘 반환 완료 건수

SELECT COUNT(\*) AS returned\_count

FROM reservations

WHERE use\_date = CURRENT\_DATE

&#x20; AND status = 'RETURNED'

&#x20; AND terminal = 'T1';

```



\### 10-2. 예약번호로 예약 조회 (접수/반환/고객 확인)



```sql

SELECT

&#x20; r.\*,

&#x20; p.toss\_payment\_key,

&#x20; p.method,

&#x20; p.paid\_at

FROM reservations r

LEFT JOIN payments p ON p.reservation\_id = r.id AND p.status = 'PAID'

WHERE r.reservation\_no = 'BD-20260715-0001';

```



\### 10-3. 보관 현황 목록 (오래된 순)



```sql

SELECT

&#x20; reservation\_no,

&#x20; terminal,

&#x20; direction,

&#x20; customer\_name,

&#x20; bag\_count,

&#x20; checked\_in\_at,

&#x20; EXTRACT(EPOCH FROM (now() - checked\_in\_at)) / 60 AS elapsed\_minutes

FROM reservations

WHERE status = 'STORED'

&#x20; AND terminal = 'T1'  -- 필터 적용 시

ORDER BY checked\_in\_at ASC;

```



\### 10-4. 예약 상태 이력 조회



```sql

SELECT

&#x20; rl.from\_status,

&#x20; rl.to\_status,

&#x20; rl.actor\_type,

&#x20; rl.actor\_label,

&#x20; rl.created\_at

FROM reservation\_logs rl

WHERE rl.reservation\_id = '예약-UUID'

ORDER BY rl.created\_at ASC;

```



\### 10-5. 일별 매출 집계



```sql

SELECT

&#x20; COUNT(\*) FILTER (WHERE status != 'CANCELLED') AS paid\_count,

&#x20; COUNT(\*) FILTER (WHERE status = 'CANCELLED')  AS cancelled\_count,

&#x20; SUM(amount) FILTER (WHERE status != 'CANCELLED')

&#x20;   - COALESCE(SUM(amount) FILTER (WHERE status = 'CANCELLED'), 0)

&#x20;   AS net\_revenue

FROM reservations

WHERE DATE(created\_at AT TIME ZONE 'Asia/Seoul') = '2026-07-15'

&#x20; AND terminal = 'T1';  -- 전체 시 제거

```



\### 10-6. 예약번호 채번 (Sequence RPC 호출)



```sql

\-- Sequence 방식: 원자적, 잠금 없음, 동시 호출 안전

\-- 애플리케이션에서 아래 RPC 한 번 호출로 예약번호 발급 완료



SELECT get\_next\_reservation\_no(

&#x20; TO\_CHAR(NOW() AT TIME ZONE 'Asia/Seoul', 'YYYYMMDD')

);

\-- 결과 예시: 'BD-20260715-0007'



\-- Next.js API Route에서의 호출 방식

\-- const { data } = await supabase.rpc('get\_next\_reservation\_no', {

\--   target\_date: format(new Date(), 'yyyyMMdd', { timeZone: 'Asia/Seoul' })

\-- });

\-- const reservationNo = data; // 'BD-20260715-0007'

```



> \*\*번호 공백 허용\*\*: 결제 실패 또는 트랜잭션 롤백 시 Sequence는 이미 증가했으므로 해당 번호는 영구 결번이 된다. (예: 0003 발급 후 결제 실패 → 0003 결번, 다음 예약은 0004) 이는 Sequence의 정상 동작이며 운영상 무해하다.



\---



\## 11. 마이그레이션 실행 순서



```

1\. CREATE TYPE (Enum 6종)

2\. CREATE TABLE admin\_users

3\. CREATE TABLE reservations

4\. CREATE TABLE payments

5\. CREATE TABLE reservation\_logs

6\. CREATE TABLE sms\_logs

7\. CREATE INDEX (인덱스 전체)

8\. CREATE TRIGGER (updated\_at 트리거)

9\. ALTER TABLE (비즈니스 로직 CHECK 제약)

10\. CREATE FUNCTION get\_next\_reservation\_no (채번 RPC 함수)

11\. ALTER TABLE ENABLE ROW LEVEL SECURITY + CREATE POLICY

12\. 시드 스크립트 실행 (npm run seed)

```



\---



\## 12. 환경변수 목록 (DB 관련)



| 변수명 | 설명 | 예시 |

|--------|------|------|

| `NEXT\_PUBLIC\_SUPABASE\_URL` | Supabase 프로젝트 URL | https://xxxx.supabase.co |

| `NEXT\_PUBLIC\_SUPABASE\_ANON\_KEY` | Supabase anon 키 (FE 노출용, 실질적 미사용) | eyJ... |

| `SUPABASE\_SERVICE\_ROLE\_KEY` | Supabase 서비스 롤 키 (서버사이드 전용, 절대 FE 노출 금지) | eyJ... |

| `ADMIN\_INITIAL\_PASSWORD` | 초기 어드민 계정 비밀번호 (시드 전용) | bagdrop2026! |

| `NEXTAUTH\_SECRET` | NextAuth.js 세션 암호화 키 | 32자 이상 랜덤 문자열 |



\---



\## Session 04 결정 사항 및 미결정 항목



\### ✅ 이번 세션에서 확정된 사항



| # | 항목 | 결정 내용 |

|---|------|----------|

| 1 | 예약번호 채번 동시성 처리 | PostgreSQL 날짜별 Sequence + `get\_next\_reservation\_no()` RPC 함수 |



\### 🟡 후속 확인 필요 사항



| # | 항목 | 영향 범위 | 중요도 |

|---|------|-----------|--------|

| 1 | 결제 실패 시 PENDING 상태 payments 레코드 보존 기간 정책 | DB 용량, 배치 삭제 여부 | 🟡 |

| 2 | toss\_response JSONB 컬럼 보존 기간 (개인정보 포함 가능성) | 개인정보 처리방침 | 🟡 |



\---



\*다음 세션: Session 05 — API 엔드포인트 설계 또는 디자인 에이전트 다올 착수\*

