\# BagDrop API 엔드포인트 설계



> 작성: 기획 에이전트 여울

> 세션: Session 05

> 최종 업데이트: 2026-07-01

> 플랫폼: Next.js 14 App Router / API Routes

> 상태: ✅ Session 05 확정



\---



\## 1. 인증 처리 방식



\### FO (고객용) — 인증 없음

\- 모든 FO API는 인증 불필요 (공개 서비스)

\- 예약 조회는 `reservation\_no` 파라미터로 접근 제어

\- Rate Limiting으로 남용 방지 (Vercel Edge Config 또는 미들웨어)



\### BO (운영자용) — NextAuth.js 세션 인증

\- 모든 `/api/admin/\*` 경로는 NextAuth.js 세션 필수

\- 세션 없는 요청 → `401 Unauthorized` 반환

\- 세션 유효 시간: 12시간



```

\[BO API 인증 미들웨어 처리 흐름]



Request → /api/admin/\* 

&#x20; → getServerSession() 호출

&#x20; → 세션 없음 → 401 반환

&#x20; → 세션 있음 → 핸들러 실행

```



\### Supabase 클라이언트 분리 원칙

\- \*\*서버사이드 전용\*\*: `SUPABASE\_SERVICE\_ROLE\_KEY` — 모든 API Route에서만 사용

\- \*\*클라이언트 노출 금지\*\*: `service\_role` 키는 절대 브라우저에 노출하지 않음

\- FO/BO 클라이언트에서 Supabase 직접 호출 없음 → 반드시 Next.js API Route 경유



\---



\## 2. 공통 규칙



\### Base URL

```

Production:  https://bagdrop.kr/api

Development: http://localhost:3000/api

```



\### 공통 Request Headers



| 헤더 | 값 | 필수 여부 |

|------|----|----------|

| `Content-Type` | `application/json` | POST/PATCH 필수 |

| `Cookie` | NextAuth 세션 쿠키 | BO API 필수 |



\### 공통 Response 형식



```json

// 성공

{

&#x20; "success": true,

&#x20; "data": { ... }

}



// 실패

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "ERROR\_CODE",

&#x20;   "message": "사람이 읽을 수 있는 오류 메시지"

&#x20; }

}

```



\---



\## 3. 공통 에러 코드



| HTTP 상태 | 에러 코드 | 메시지 | 발생 상황 |

|-----------|-----------|--------|----------|

| 400 | `INVALID\_REQUEST` | 요청 형식이 올바르지 않습니다. | 필수 파라미터 누락, 형식 오류 |

| 400 | `INVALID\_RESERVATION\_NO` | 올바른 예약번호 형식이 아닙니다. | BD-YYYYMMDD-NNNN 형식 불일치 |

| 400 | `INVALID\_DATE` | 올바른 날짜 형식이 아닙니다. | 날짜 범위 또는 형식 오류 |

| 400 | `INVALID\_BAG\_COUNT` | 짐 개수는 1개 또는 2개만 선택 가능합니다. | bag\_count 범위 초과 |

| 401 | `UNAUTHORIZED` | 로그인이 필요합니다. | BO 세션 없음 |

| 404 | `RESERVATION\_NOT\_FOUND` | 예약을 찾을 수 없습니다. | 예약번호 없음 |

| 409 | `ALREADY\_STORED` | 이미 접수된 예약입니다. | STORED 상태 재접수 |

| 409 | `ALREADY\_RETURNED` | 이미 반환된 예약입니다. | RETURNED 상태 재처리 |

| 409 | `ALREADY\_CANCELLED` | 이미 취소된 예약입니다. | CANCELLED 상태 재처리 |

| 409 | `NOT\_STORED` | 아직 접수되지 않은 예약입니다. | RESERVED 상태에서 반환 시도 |

| 409 | `CANCEL\_NOT\_ALLOWED` | 이용일 당일은 취소할 수 없습니다. | 이용일 00시 이후 취소 시도 |

| 409 | `PAYMENT\_ALREADY\_EXISTS` | 이미 처리 중인 결제가 있습니다. | 중복 결제 시도 |

| 422 | `PAYMENT\_FAILED` | 결제 처리에 실패했습니다. | 토스페이 승인 실패 |

| 500 | `INTERNAL\_ERROR` | 서버 오류가 발생했습니다. 잠시 후 다시 시도해 주세요. | 예상치 못한 서버 오류 |

| 503 | `SMS\_FAILED` | SMS 발송에 실패했습니다. | 알리고 API 오류 (예약은 정상 처리됨) |



> \*\*SMS 실패 정책\*\*: SMS 발송 실패는 `503`을 반환하지 않고 `sms\_logs`에 FAILED로 기록 후 예약 처리는 성공 응답. 발송 실패를 이유로 예약 롤백하지 않음.



\---



\## 4. API 목록 전체



\### FO API (공개)



| Method | URL | 설명 |

|--------|-----|------|

| POST | `/api/reservations/prepare` | 결제 준비 (toss\_order\_id 발급) |

| POST | `/api/reservations/confirm` | 결제 승인 및 예약 확정 |

| GET | `/api/reservations/:reservationNo` | 예약 조회 (고객) |

| POST | `/api/reservations/:reservationNo/cancel` | 예약 취소 (고객) |

| POST | `/api/payments/webhook` | 토스페이 웹훅 수신 |



\### BO API (인증 필요)



| Method | URL | 설명 |

|--------|-----|------|

| POST | `/api/admin/auth/login` | 운영자 로그인 |

| POST | `/api/admin/auth/logout` | 운영자 로그아웃 |

| GET | `/api/admin/dashboard` | 대시보드 현황 카운트 |

| GET | `/api/admin/reservations` | 예약 목록 (필터) |

| GET | `/api/admin/reservations/:reservationNo` | 예약 상세 (운영자) |

| PATCH | `/api/admin/reservations/:reservationNo/checkin` | 짐 접수 처리 |

| PATCH | `/api/admin/reservations/:reservationNo/checkout` | 짐 반환 처리 |

| PATCH | `/api/admin/reservations/:reservationNo/cancel` | 예약 강제 취소 |

| GET | `/api/admin/storage` | 보관 현황 목록 |

| GET | `/api/admin/sales` | 매출 집계 |



\---



\## 5. FO API 상세



\---



\### POST `/api/reservations/prepare`



\*\*역할\*\*: 결제창 호출 전 `toss\_order\_id` 발급 및 payments 레코드 PENDING 생성



\#### Request



```json

// Body

{

&#x20; "terminal": "T1",                // "T1" | "T2"

&#x20; "direction": "DEPARTURE",       // "DEPARTURE" | "ARRIVAL"

&#x20; "use\_date": "2026-07-15",       // YYYY-MM-DD

&#x20; "bag\_count": 2,                  // 1 | 2

&#x20; "customer\_name": "홍길동",

&#x20; "customer\_phone": "01012345678" // 숫자만 11자리

}

```



\#### 유효성 검증



| 필드 | 검증 규칙 |

|------|----------|

| `terminal` | T1 또는 T2 |

| `direction` | DEPARTURE 또는 ARRIVAL |

| `use\_date` | 오늘 이상 \~ 오늘+90일 이하 |

| `bag\_count` | 1 또는 2 정수 |

| `customer\_name` | 1\~20자, 공백만 입력 불가 |

| `customer\_phone` | 01X 시작 11자리 숫자 |



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "toss\_order\_id": "bagdrop\_20260715\_a1b2c3d4",  // 결제창 호출 시 사용

&#x20;   "amount": 5000,

&#x20;   "order\_name": "BagDrop 짐 보관 서비스"           // 토스페이 결제창 표시명

&#x20; }

}



// 400 실패 예시

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "INVALID\_DATE",

&#x20;   "message": "이용일은 오늘부터 90일 이내여야 합니다."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 요청 유효성 검증

2\. toss\_order\_id 생성: "bagdrop\_{YYYYMMDD}\_{8자리 랜덤 hex}"

3\. payments INSERT (status: PENDING, toss\_order\_id, amount: 5000)

4\. 세션스토리지 저장용 임시 예약 데이터 반환 포함

&#x20;  (customer\_name, customer\_phone 등은 결제 완료 후 confirm API에서 사용)

```



> 이 단계에서 reservations는 생성하지 않음. 결제 완료 후 confirm API에서 생성.



\---



\### POST `/api/reservations/confirm`



\*\*역할\*\*: 토스페이 결제 승인 → 예약 확정 → SMS 발송



\#### Request



```json

// Body

{

&#x20; "payment\_key": "5zJ4xY7m0A....",         // 토스페이 paymentKey (결제 완료 후 콜백)

&#x20; "toss\_order\_id": "bagdrop\_20260715\_a1b2c3d4",

&#x20; "amount": 5000,                           // 클라이언트 검증용 (서버에서 재확인)



&#x20; // 예약 정보 (prepare 단계에서 세션스토리지에 보관 후 전달)

&#x20; "terminal": "T1",

&#x20; "direction": "DEPARTURE",

&#x20; "use\_date": "2026-07-15",

&#x20; "bag\_count": 2,

&#x20; "customer\_name": "홍길동",

&#x20; "customer\_phone": "01012345678"

}

```



\#### Response



```json

// 201 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "terminal": "T1",

&#x20;   "direction": "DEPARTURE",

&#x20;   "use\_date": "2026-07-15",

&#x20;   "bag\_count": 2,

&#x20;   "customer\_name": "홍길동",

&#x20;   "status": "RESERVED",

&#x20;   "created\_at": "2026-07-14T14:32:00+09:00"

&#x20; }

}



// 422 결제 승인 실패

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "PAYMENT\_FAILED",

&#x20;   "message": "결제 처리에 실패했습니다. 다시 시도해 주세요."

&#x20; }

}



// 409 중복 결제 시도

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "PAYMENT\_ALREADY\_EXISTS",

&#x20;   "message": "이미 처리된 결제입니다."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. toss\_order\_id로 payments 레코드 조회

&#x20;  → 없거나 PENDING이 아니면 409 반환

2\. 금액 재검증: amount == 5000 확인

3\. 토스페이 결제 승인 API 호출

&#x20;  POST https://api.tosspayments.com/v1/payments/confirm

&#x20;  { paymentKey, orderId, amount }

&#x20;  → 실패 시 payments.status = FAILED 업데이트 후 422 반환

4\. DB 트랜잭션 시작

&#x20;  a. get\_next\_reservation\_no(today\_kst) RPC 호출 → reservation\_no 발급

&#x20;  b. reservations INSERT (status: RESERVED)

&#x20;  c. payments UPDATE (status: PAID, toss\_payment\_key, toss\_response, paid\_at)

&#x20;  d. reservation\_logs INSERT (NULL → RESERVED, actor\_type: SYSTEM)

5\. DB 트랜잭션 커밋

6\. 알리고 SMS 발송 (예약 완료) — 비동기, 실패 시 sms\_logs에 FAILED 기록

7\. 201 응답 반환

```



\---



\### GET `/api/reservations/:reservationNo`



\*\*역할\*\*: 예약번호로 예약 정보 조회 (고객 예약 확인 화면)



\#### Request



```

// Path Params

reservationNo: "BD-20260715-0007"   // BD-YYYYMMDD-NNNN 형식

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "terminal": "T1",

&#x20;   "direction": "DEPARTURE",

&#x20;   "use\_date": "2026-07-15",

&#x20;   "bag\_count": 2,

&#x20;   "customer\_name": "홍길동",

&#x20;   "customer\_phone\_masked": "010-\*\*\*\*-5678",  // 마스킹 처리

&#x20;   "amount": 5000,

&#x20;   "status": "STORED",

&#x20;   "checked\_in\_at": "2026-07-15T09:11:00+09:00",

&#x20;   "checked\_out\_at": null,

&#x20;   "cancelled\_at": null,

&#x20;   "cancelled\_by": null,

&#x20;   "payment": {

&#x20;     "method": "TOSSPAY",

&#x20;     "paid\_at": "2026-07-14T14:32:00+09:00"

&#x20;   },

&#x20;   "logs": \[

&#x20;     { "from\_status": null,       "to\_status": "RESERVED", "created\_at": "2026-07-14T14:32:00+09:00", "actor\_label": "예약완료" },

&#x20;     { "from\_status": "RESERVED", "to\_status": "STORED",   "created\_at": "2026-07-15T09:11:00+09:00", "actor\_label": "짐 접수" }

&#x20;   ],

&#x20;   "is\_cancellable": false   // 취소 가능 여부 (서버에서 계산)

&#x20; }

}



// 404 실패

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "RESERVATION\_NOT\_FOUND",

&#x20;   "message": "해당 예약번호를 찾을 수 없습니다."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. reservation\_no 형식 검증 (BD-YYYYMMDD-NNNN)

2\. reservations + payments + reservation\_logs JOIN 조회

3\. customer\_phone 마스킹 처리 (서버에서 처리 후 반환)

4\. is\_cancellable 계산:

&#x20;  → status == 'RESERVED' AND use\_date > today\_kst → true

&#x20;  → 그 외 → false

5\. 200 응답

```



\---



\### POST `/api/reservations/:reservationNo/cancel`



\*\*역할\*\*: 고객 예약 취소 (이용일 00시 이전만 가능)



\#### Request



```

// Path Params

reservationNo: "BD-20260715-0007"

```



```json

// Body (없음 — Path Param만으로 처리)

{}

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "status": "CANCELLED",

&#x20;   "cancelled\_at": "2026-07-14T20:15:00+09:00",

&#x20;   "cancelled\_by": "CUSTOMER"

&#x20; }

}



// 409 취소 불가 — 이용일 당일

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "CANCEL\_NOT\_ALLOWED",

&#x20;   "message": "이용일 당일은 취소할 수 없습니다."

&#x20; }

}



// 409 취소 불가 — 이미 접수됨

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "ALREADY\_STORED",

&#x20;   "message": "이미 접수된 예약은 취소할 수 없습니다."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 예약 조회 (없으면 404)

2\. 취소 가능 여부 검증:

&#x20;  a. status == 'RESERVED' 인지 확인 (아니면 409)

&#x20;  b. use\_date > today\_kst 인지 확인 (당일 이후면 409)

3\. DB 트랜잭션 시작

&#x20;  a. reservations UPDATE (status: CANCELLED, cancelled\_by: CUSTOMER, cancelled\_at: now())

&#x20;  b. reservation\_logs INSERT (RESERVED → CANCELLED, actor\_type: CUSTOMER, actor\_label: '고객 직접 취소')

4\. DB 트랜잭션 커밋

5\. 알리고 SMS 발송 (취소 완료) — 비동기

&#x20;  ※ 실제 환불은 토스페이 어드민 콘솔에서 운영자가 수동 처리

6\. 200 응답

```



\---



\### POST `/api/payments/webhook`



\*\*역할\*\*: 토스페이 웹훅 수신 (결제 상태 변경 알림)



> 토스페이는 결제 완료/취소 이벤트를 웹훅으로 서버에 알림.

> confirm API와 이중 처리되지 않도록 payments.status 기반으로 멱등 처리.



\#### Request



```json

// Headers (토스페이 서명 검증용)

"Toss-Signature": "sha256=abcd1234..."



// Body (토스페이 웹훅 payload)

{

&#x20; "eventType": "PAYMENT\_STATUS\_CHANGED",

&#x20; "data": {

&#x20;   "paymentKey": "5zJ4xY7m0A....",

&#x20;   "orderId": "bagdrop\_20260715\_a1b2c3d4",

&#x20;   "status": "DONE",   // DONE | CANCELED | PARTIAL\_CANCELED | ABORTED | EXPIRED

&#x20;   "amount": 5000

&#x20; }

}

```



\#### Response



```json

// 200 (웹훅은 항상 200 반환, 처리 실패는 로그 기록)

{ "success": true }

```



\#### 비즈니스 로직



```

1\. 토스페이 웹훅 서명 검증 (HMAC-SHA256)

&#x20;  → 검증 실패 시 400 반환 (처리 안 함)

2\. paymentKey로 payments 레코드 조회

3\. 이미 처리된 건(PAID/FAILED) 이면 200 반환 (멱등 처리)

4\. status에 따라 분기:

&#x20;  DONE       → confirm API에서 이미 처리됨. 상태 불일치 시 로그만 기록

&#x20;  CANCELED   → payments.status = CANCELLED 업데이트 (환불 완료 확인용)

&#x20;  ABORTED    → payments.status = FAILED 업데이트

&#x20;  EXPIRED    → payments.status = FAILED 업데이트

5\. 200 응답

```



\---



\## 6. BO API 상세



\---



\### POST `/api/admin/auth/login`



\*\*역할\*\*: NextAuth.js Credentials Provider 로그인 (내부적으로 처리, 커스텀 엔드포인트 불필요)



> NextAuth.js의 `/api/auth/\[...nextauth]` 가 자동 처리.

> 아래는 NextAuth.js Credentials Provider 설정 명세.



```javascript

// pages/api/auth/\[...nextauth].js (또는 app/api/auth/\[...nextauth]/route.js)

CredentialsProvider({

&#x20; name: 'Credentials',

&#x20; credentials: {

&#x20;   username: { label: 'ID', type: 'text' },

&#x20;   password: { label: 'PW', type: 'password' }

&#x20; },

&#x20; async authorize(credentials) {

&#x20;   // 1. admin\_users 테이블에서 username으로 조회

&#x20;   // 2. is\_active == true 확인

&#x20;   // 3. locked\_until 확인 (잠금 중이면 null 반환)

&#x20;   // 4. bcrypt.compare(password, password\_hash)

&#x20;   // 5. 실패 시 login\_fail\_count +1, 5회 이상이면 locked\_until = now()+5min

&#x20;   // 6. 성공 시 login\_fail\_count = 0, last\_login\_at = now() 업데이트

&#x20;   // 7. { id, username, displayName } 반환

&#x20; }

})

```



\*\*로그인 실패 응답\*\*: NextAuth.js 표준 오류 처리 (`error=CredentialsSignin`)



\---



\### POST `/api/admin/auth/logout`



> NextAuth.js의 `signOut()` 클라이언트 함수로 처리. 별도 커스텀 엔드포인트 불필요.



\---



\### GET `/api/admin/dashboard`



\*\*역할\*\*: 대시보드 현황 카운트 (30초 폴링)



\#### Request



```

// Query Params (선택)

terminal: "T1" | "T2" | undefined   // 미지정 시 전체(T1+T2)

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "today\_total": 12,         // 오늘 총 예약 (RESERVED+STORED+RETURNED)

&#x20;   "current\_stored": 3,       // 현재 보관 중 (STORED)

&#x20;   "today\_returned": 7,       // 오늘 반환 완료 (RETURNED)

&#x20;   "recent\_pending": \[        // 최근 접수 대기 5건 (RESERVED)

&#x20;     {

&#x20;       "reservation\_no": "BD-20260715-0012",

&#x20;       "terminal": "T1",

&#x20;       "direction": "DEPARTURE",

&#x20;       "customer\_name": "홍길동",

&#x20;       "bag\_count": 2,

&#x20;       "created\_at": "2026-07-15T14:55:00+09:00",

&#x20;       "elapsed\_minutes": 10

&#x20;     }

&#x20;   ],

&#x20;   "as\_of": "2026-07-15T15:05:00+09:00"   // 조회 시각

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. terminal 파라미터에 따라 WHERE 조건 분기 (없으면 전체)

3\. KST 기준 today 날짜 계산

4\. 병렬 쿼리 실행:

&#x20;  a. today\_total: use\_date = today AND status IN (RESERVED, STORED, RETURNED)

&#x20;  b. current\_stored: status = STORED

&#x20;  c. today\_returned: use\_date = today AND status = RETURNED

&#x20;  d. recent\_pending: status = RESERVED ORDER BY created\_at DESC LIMIT 5

5\. elapsed\_minutes: (now - created\_at) / 60 계산

6\. 200 응답

```



\---



\### GET `/api/admin/reservations`



\*\*역할\*\*: 예약 목록 조회 (날짜·터미널·상태 필터)



\#### Request



```

// Query Params

date:     "2026-07-15"              // YYYY-MM-DD (기본: 오늘)

terminal: "T1" | "T2"              // 미지정 시 전체

status:   "RESERVED,STORED"        // 콤마 구분 다중 선택, 미지정 시 전체

page:     1                        // 페이지 (기본: 1)

limit:    20                       // 페이지당 건수 (기본: 20, 최대: 100)

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "items": \[

&#x20;     {

&#x20;       "reservation\_no": "BD-20260715-0012",

&#x20;       "terminal": "T1",

&#x20;       "direction": "DEPARTURE",

&#x20;       "use\_date": "2026-07-15",

&#x20;       "bag\_count": 2,

&#x20;       "customer\_name": "홍길동",

&#x20;       "status": "RESERVED",

&#x20;       "created\_at": "2026-07-15T14:55:00+09:00",

&#x20;       "checked\_in\_at": null,

&#x20;       "checked\_out\_at": null

&#x20;     }

&#x20;   ],

&#x20;   "total": 12,

&#x20;   "page": 1,

&#x20;   "limit": 20

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. 파라미터 파싱 및 유효성 검증

3\. status 파라미터 → 배열로 파싱: "RESERVED,STORED" → \['RESERVED', 'STORED']

4\. 동적 WHERE 조건 생성 후 SELECT + COUNT

5\. ORDER BY created\_at DESC

6\. 200 응답

```



\---



\### GET `/api/admin/reservations/:reservationNo`



\*\*역할\*\*: 예약 상세 조회 (운영자)



\#### Request



```

// Path Params

reservationNo: "BD-20260715-0007"

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "terminal": "T1",

&#x20;   "direction": "DEPARTURE",

&#x20;   "use\_date": "2026-07-15",

&#x20;   "bag\_count": 2,

&#x20;   "customer\_name": "홍길동",

&#x20;   "customer\_phone\_masked": "010-\*\*\*\*-5678",

&#x20;   "amount": 5000,

&#x20;   "status": "STORED",

&#x20;   "checked\_in\_at": "2026-07-15T09:11:00+09:00",

&#x20;   "checked\_out\_at": null,

&#x20;   "cancelled\_at": null,

&#x20;   "cancelled\_by": null,

&#x20;   "payment": {

&#x20;     "method": "TOSSPAY",

&#x20;     "toss\_payment\_key": "5zJ4xY7m0A....",

&#x20;     "paid\_at": "2026-07-14T14:32:00+09:00",

&#x20;     "status": "PAID"

&#x20;   },

&#x20;   "logs": \[

&#x20;     { "from\_status": null,       "to\_status": "RESERVED", "actor\_label": "예약완료",     "created\_at": "2026-07-14T14:32:00+09:00" },

&#x20;     { "from\_status": "RESERVED", "to\_status": "STORED",   "actor\_label": "짐 접수",      "created\_at": "2026-07-15T09:11:00+09:00" }

&#x20;   ]

&#x20; }

}

```



> FO 고객용 조회와 동일하나 `toss\_payment\_key` 포함 (운영자 환불 참조용).



\---



\### PATCH `/api/admin/reservations/:reservationNo/checkin`



\*\*역할\*\*: 짐 접수 처리 (RESERVED → STORED)



\#### Request



```

// Path Params

reservationNo: "BD-20260715-0007"



// Body (없음)

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "status": "STORED",

&#x20;   "checked\_in\_at": "2026-07-15T09:11:00+09:00"

&#x20; }

}



// 409 이미 접수됨

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "ALREADY\_STORED",

&#x20;   "message": "이미 접수 처리된 예약입니다."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. 예약 조회 (없으면 404)

3\. 상태 검증:

&#x20;  status != RESERVED → 적절한 409 반환

&#x20;    STORED    → ALREADY\_STORED

&#x20;    RETURNED  → ALREADY\_RETURNED

&#x20;    CANCELLED → ALREADY\_CANCELLED

4\. DB 트랜잭션 시작

&#x20;  a. reservations UPDATE (status: STORED, checked\_in\_at: now())

&#x20;  b. reservation\_logs INSERT (RESERVED → STORED, actor\_type: ADMIN, actor\_id: session.adminId, actor\_label: '짐 접수')

5\. DB 트랜잭션 커밋

6\. 알리고 SMS 발송 (짐 접수 완료) — 비동기

7\. 200 응답

```



\---



\### PATCH `/api/admin/reservations/:reservationNo/checkout`



\*\*역할\*\*: 짐 반환 처리 (STORED → RETURNED)



\#### Request



```

// Path Params

reservationNo: "BD-20260715-0007"



// Body (없음)

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "status": "RETURNED",

&#x20;   "checked\_out\_at": "2026-07-15T11:45:00+09:00"

&#x20; }

}



// 409 아직 접수 안 됨

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "NOT\_STORED",

&#x20;   "message": "아직 접수되지 않은 예약입니다. 먼저 짐 접수를 진행해 주세요."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. 예약 조회 (없으면 404)

3\. 상태 검증:

&#x20;  status != STORED → 적절한 409 반환

&#x20;    RESERVED  → NOT\_STORED

&#x20;    RETURNED  → ALREADY\_RETURNED

&#x20;    CANCELLED → ALREADY\_CANCELLED

4\. DB 트랜잭션 시작

&#x20;  a. reservations UPDATE (status: RETURNED, checked\_out\_at: now())

&#x20;  b. reservation\_logs INSERT (STORED → RETURNED, actor\_type: ADMIN, actor\_id: session.adminId, actor\_label: '짐 반환')

5\. DB 트랜잭션 커밋

6\. 알리고 SMS 발송 (짐 반환 완료) — 비동기

7\. 200 응답

```



\---



\### PATCH `/api/admin/reservations/:reservationNo/cancel`



\*\*역할\*\*: 운영자 강제 취소 (상태만 CANCELLED 전환, 환불은 토스페이 콘솔 수동)



\#### Request



```

// Path Params

reservationNo: "BD-20260715-0007"

```



```json

// Body

{

&#x20; "cancel\_reason": "고객 요청으로 인한 강제 취소"  // 선택

}

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "reservation\_no": "BD-20260715-0007",

&#x20;   "status": "CANCELLED",

&#x20;   "cancelled\_at": "2026-07-15T10:00:00+09:00",

&#x20;   "cancelled\_by": "ADMIN",

&#x20;   "toss\_payment\_key": "5zJ4xY7m0A....",   // 토스페이 콘솔 환불 시 사용

&#x20;   "refund\_guide": "토스페이 어드민 콘솔(https://dashboard.tosspayments.com)에서 위 paymentKey로 환불을 처리해 주세요."

&#x20; }

}



// 409 접수 이후 취소 불가

{

&#x20; "success": false,

&#x20; "error": {

&#x20;   "code": "ALREADY\_STORED",

&#x20;   "message": "접수된 예약은 강제 취소할 수 없습니다."

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. 예약 조회 (없으면 404)

3\. 상태 검증: status == 'RESERVED' 만 허용

&#x20;  STORED    → ALREADY\_STORED (409)

&#x20;  RETURNED  → ALREADY\_RETURNED (409)

&#x20;  CANCELLED → ALREADY\_CANCELLED (409)

4\. DB 트랜잭션 시작

&#x20;  a. reservations UPDATE (status: CANCELLED, cancelled\_by: ADMIN, cancelled\_at: now(), cancel\_reason)

&#x20;  b. reservation\_logs INSERT (RESERVED → CANCELLED, actor\_type: ADMIN, actor\_label: '운영자 강제 취소')

5\. DB 트랜잭션 커밋

6\. 알리고 SMS 발송 (취소 완료) — 비동기

&#x20;  ※ 토스페이 환불 API 호출 없음. 응답에 toss\_payment\_key 포함하여 운영자가 콘솔에서 직접 환불

7\. 200 응답

```



\---



\### GET `/api/admin/storage`



\*\*역할\*\*: 현재 보관 중 목록 (STORED 상태 전체, 30초 폴링용)



\#### Request



```

// Query Params (선택)

terminal: "T1" | "T2"   // 미지정 시 전체

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "items": \[

&#x20;     {

&#x20;       "reservation\_no": "BD-20260715-0008",

&#x20;       "terminal": "T1",

&#x20;       "direction": "DEPARTURE",

&#x20;       "customer\_name": "홍길동",

&#x20;       "customer\_phone\_masked": "010-\*\*\*\*-5678",

&#x20;       "bag\_count": 2,

&#x20;       "use\_date": "2026-07-15",

&#x20;       "checked\_in\_at": "2026-07-15T13:45:00+09:00",

&#x20;       "elapsed\_minutes": 80,

&#x20;       "elapsed\_warning": "ORANGE"   // null | "ORANGE" (2h+) | "RED" (4h+)

&#x20;     }

&#x20;   ],

&#x20;   "total": 3,

&#x20;   "as\_of": "2026-07-15T15:05:00+09:00"

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. status = STORED AND terminal = (파라미터 or 전체) 조회

3\. ORDER BY checked\_in\_at ASC (오래된 순)

4\. elapsed\_minutes 계산 및 elapsed\_warning 분류:

&#x20;  0\~119분  → null

&#x20;  120\~239분 → "ORANGE"

&#x20;  240분 이상 → "RED"

5\. 200 응답

```



\---



\### GET `/api/admin/sales`



\*\*역할\*\*: 매출 집계 (일간/주간/월간)



\#### Request



```

// Query Params

period:   "daily" | "weekly" | "monthly"  // 기본: daily

date:     "2026-07-15"                    // 기준 날짜 (daily: 해당일, weekly: 해당 주 포함, monthly: 해당 월)

terminal: "T1" | "T2"                    // 미지정 시 전체

page:     1

limit:    50

```



\#### Response



```json

// 200 성공

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "period": "daily",

&#x20;   "date\_range": {

&#x20;     "from": "2026-07-15",

&#x20;     "to": "2026-07-15"

&#x20;   },

&#x20;   "summary": {

&#x20;     "paid\_count": 12,

&#x20;     "cancelled\_count": 2,

&#x20;     "net\_revenue": 50000   // (12 - 2) \* 5000

&#x20;   },

&#x20;   "items": \[

&#x20;     {

&#x20;       "reservation\_no": "BD-20260715-0012",

&#x20;       "terminal": "T1",

&#x20;       "customer\_name": "홍길동",

&#x20;       "method": "TOSSPAY",

&#x20;       "amount": 5000,

&#x20;       "status": "RESERVED",

&#x20;       "is\_cancelled": false,

&#x20;       "paid\_at": "2026-07-15T14:55:00+09:00"

&#x20;     },

&#x20;     {

&#x20;       "reservation\_no": "BD-20260715-0004",

&#x20;       "terminal": "T2",

&#x20;       "customer\_name": "김영희",

&#x20;       "method": "TOSSPAY",

&#x20;       "amount": 5000,

&#x20;       "status": "CANCELLED",

&#x20;       "is\_cancelled": true,

&#x20;       "paid\_at": "2026-07-15T09:10:00+09:00"

&#x20;     }

&#x20;   ],

&#x20;   "total": 14,

&#x20;   "page": 1,

&#x20;   "limit": 50

&#x20; }

}

```



\#### 비즈니스 로직



```

1\. 세션 인증 확인

2\. period에 따라 날짜 범위 계산:

&#x20;  daily:   from = date, to = date

&#x20;  weekly:  from = 해당 주 월요일, to = 해당 주 일요일

&#x20;  monthly: from = 해당 월 1일, to = 해당 월 말일

3\. reservations JOIN payments WHERE paid\_at BETWEEN from AND to (KST 기준)

4\. terminal 필터 적용

5\. 집계 쿼리: paid\_count, cancelled\_count, net\_revenue

6\. 목록 쿼리: ORDER BY paid\_at DESC + 페이징

7\. 200 응답

```



\---



\## 7. 토스페이 연동 상세



\### 7-1. 결제 전체 흐름



```

\[클라이언트 FO]                    \[Next.js API]              \[토스페이]

&#x20;     │                                  │                        │

&#x20;     │ 1. POST /api/reservations/prepare│                        │

&#x20;     │─────────────────────────────────>│                        │

&#x20;     │ toss\_order\_id 반환               │                        │

&#x20;     │<─────────────────────────────────│                        │

&#x20;     │                                  │                        │

&#x20;     │ 2. 토스페이 JS SDK 결제창 호출  │                        │

&#x20;     │──────────────────────────────────────────────────────────>│

&#x20;     │              결제 완료 (paymentKey, orderId, amount 콜백) │

&#x20;     │<──────────────────────────────────────────────────────────│

&#x20;     │                                  │                        │

&#x20;     │ 3. POST /api/reservations/confirm│                        │

&#x20;     │─────────────────────────────────>│                        │

&#x20;     │                                  │ 4. POST /v1/payments/confirm

&#x20;     │                                  │───────────────────────>│

&#x20;     │                                  │ 승인 응답 (200)        │

&#x20;     │                                  │<───────────────────────│

&#x20;     │                                  │                        │

&#x20;     │                                  │ 5. DB 트랜잭션         │

&#x20;     │                                  │ (예약 생성 + 결제 확정)│

&#x20;     │                                  │                        │

&#x20;     │ 예약번호 반환                    │                        │

&#x20;     │<─────────────────────────────────│                        │

&#x20;     │                                  │                        │

&#x20;     │ 6. /reserve/complete 이동       │                        │

&#x20;     │                                  │                        │

&#x20;     │                          (별도) │ 7. 웹훅 POST /api/payments/webhook

&#x20;     │                                  │<───────────────────────│

&#x20;     │                                  │ 멱등 처리 후 200       │

&#x20;     │                                  │───────────────────────>│

```



\### 7-2. 토스페이 결제 승인 API



```

POST https://api.tosspayments.com/v1/payments/confirm

Authorization: Basic {Base64(시크릿키:)}

Content-Type: application/json



{

&#x20; "paymentKey": "5zJ4xY7m0A....",

&#x20; "orderId": "bagdrop\_20260715\_a1b2c3d4",

&#x20; "amount": 5000

}

```



\### 7-3. 환경변수



| 변수명 | 설명 |

|--------|------|

| `TOSS\_SECRET\_KEY` | 토스페이 시크릿 키 (서버사이드 전용) |

| `NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY` | 토스페이 클라이언트 키 (FE 노출 가능) |

| `TOSS\_WEBHOOK\_SECRET` | 웹훅 서명 검증용 시크릿 |



\### 7-4. toss\_order\_id 생성 규칙



```javascript

// 형식: bagdrop\_{YYYYMMDD}\_{8자리 랜덤 hex}

// 예: bagdrop\_20260715\_a1b2c3d4



const orderId = `bagdrop\_${format(new Date(), 'yyyyMMdd')}\_${crypto.randomBytes(4).toString('hex')}`;

```



> 토스페이 orderId 제약: 영문 대소문자, 숫자, 특수문자(`-`, `\_`) 허용 / 최대 64자



\---



\## 8. 알리고 SMS 연동 상세



\### 8-1. 발송 함수 (공통 유틸)



```javascript

// lib/sms.js

async function sendSms({ reservationId, eventType, phone, message }) {

&#x20; // 1. 알리고 API 호출

&#x20; const formData = new FormData();

&#x20; formData.append('key', process.env.ALIGO\_API\_KEY);

&#x20; formData.append('user\_id', process.env.ALIGO\_USER\_ID);

&#x20; formData.append('sender', process.env.ALIGO\_SENDER\_PHONE);

&#x20; formData.append('receiver', phone);

&#x20; formData.append('msg', message);

&#x20; formData.append('testmode\_yn', process.env.NODE\_ENV === 'production' ? 'N' : 'Y');



&#x20; const response = await fetch('https://apis.aligo.in/send/', {

&#x20;   method: 'POST',

&#x20;   body: formData

&#x20; });

&#x20; const result = await response.json();



&#x20; // 2. sms\_logs INSERT (성공/실패 모두 기록)

&#x20; await supabase.from('sms\_logs').insert({

&#x20;   reservation\_id: reservationId,

&#x20;   event\_type: eventType,

&#x20;   recipient\_phone: phone,

&#x20;   message\_content: message,

&#x20;   result: result.result\_code === '1' ? 'SUCCESS' : 'FAILED',

&#x20;   aligo\_msg\_id: result.msg\_id?.toString() || null,

&#x20;   error\_message: result.result\_code !== '1' ? result.message : null

&#x20; });

}

```



\### 8-2. 이벤트별 SMS 메시지 템플릿



```

\[RESERVATION\_COMPLETE] 예약 완료

────────────────────────────────

\[BagDrop] 예약이 완료됐습니다.



예약번호: {reservation\_no}

터미널: {terminal\_label}

이용일: {use\_date\_formatted} / {direction\_label}

짐 개수: {bag\_count}개



부스에서 QR 또는 예약번호를 제시해 주세요.

예약 확인: https://bagdrop.kr/my-reservation



────────────────────────────────

\[CHECKIN\_COMPLETE] 짐 접수 완료

────────────────────────────────

\[BagDrop] 짐이 접수됐습니다.



예약번호: {reservation\_no}

접수 시각: {checked\_in\_at\_formatted}



차량을 가져오신 후 부스에서 짐을 찾아가세요.



────────────────────────────────

\[CHECKOUT\_COMPLETE] 짐 반환 완료

────────────────────────────────

\[BagDrop] 짐이 반환됐습니다.



예약번호: {reservation\_no}

이용해 주셔서 감사합니다!



────────────────────────────────

\[CANCELLATION\_COMPLETE] 취소 완료

────────────────────────────────

\[BagDrop] 예약이 취소됐습니다.



예약번호: {reservation\_no}

결제 금액(5,000원)은 영업일 기준 3\~5일 내 환불됩니다.

```



\### 8-3. 환경변수



| 변수명 | 설명 |

|--------|------|

| `ALIGO\_API\_KEY` | 알리고 API 키 |

| `ALIGO\_USER\_ID` | 알리고 사용자 ID |

| `ALIGO\_SENDER\_PHONE` | 발신 번호 (사전 등록 필요) |



\---



\## 9. API 구현 파일 구조



```

app/

└── api/

&#x20;   ├── reservations/

&#x20;   │   ├── prepare/

&#x20;   │   │   └── route.ts               POST /api/reservations/prepare

&#x20;   │   ├── confirm/

&#x20;   │   │   └── route.ts               POST /api/reservations/confirm

&#x20;   │   └── \[reservationNo]/

&#x20;   │       ├── route.ts               GET  /api/reservations/:reservationNo

&#x20;   │       └── cancel/

&#x20;   │           └── route.ts           POST /api/reservations/:reservationNo/cancel

&#x20;   ├── payments/

&#x20;   │   └── webhook/

&#x20;   │       └── route.ts               POST /api/payments/webhook

&#x20;   └── admin/

&#x20;       ├── auth/

&#x20;       │   └── \[...nextauth]/

&#x20;       │       └── route.ts           NextAuth.js 핸들러

&#x20;       ├── dashboard/

&#x20;       │   └── route.ts               GET  /api/admin/dashboard

&#x20;       ├── reservations/

&#x20;       │   ├── route.ts               GET  /api/admin/reservations

&#x20;       │   └── \[reservationNo]/

&#x20;       │       ├── route.ts           GET  /api/admin/reservations/:reservationNo

&#x20;       │       ├── checkin/

&#x20;       │       │   └── route.ts       PATCH /api/admin/reservations/:reservationNo/checkin

&#x20;       │       ├── checkout/

&#x20;       │       │   └── route.ts       PATCH /api/admin/reservations/:reservationNo/checkout

&#x20;       │       └── cancel/

&#x20;       │           └── route.ts       PATCH /api/admin/reservations/:reservationNo/cancel

&#x20;       ├── storage/

&#x20;       │   └── route.ts               GET  /api/admin/storage

&#x20;       └── sales/

&#x20;           └── route.ts               GET  /api/admin/sales



lib/

├── supabase.ts       // Supabase 서비스 롤 클라이언트

├── sms.ts            // 알리고 SMS 발송 유틸

├── toss.ts           // 토스페이 API 호출 유틸

├── auth.ts           // NextAuth.js 설정

└── validate.ts       // 공통 유효성 검증 유틸

```



\---



\## 10. 미들웨어 설계



```typescript

// middleware.ts

export function middleware(request: NextRequest) {

&#x20; const { pathname } = request.nextUrl;



&#x20; // BO 어드민 페이지 접근 제어 (페이지 레벨)

&#x20; if (pathname.startsWith('/admin') \&\& pathname !== '/admin/login') {

&#x20;   const token = request.cookies.get('next-auth.session-token');

&#x20;   if (!token) {

&#x20;     return NextResponse.redirect(new URL('/admin/login', request.url));

&#x20;   }

&#x20; }



&#x20; return NextResponse.next();

}



export const config = {

&#x20; matcher: \['/admin/:path\*']

};

```



> API Route 레벨 인증은 각 핸들러에서 `getServerSession()` 으로 재검증. 미들웨어는 페이지 접근 제어 목적.



\---



\## Session 05 미결정 / 후속 확인 필요 사항



| # | 항목 | 영향 범위 | 중요도 |

|---|------|-----------|--------|

| 1 | Rate Limiting 구체적 임계값 (예: IP당 분당 10회) | 보안, Vercel 설정 | 🟡 |

| 2 | 토스페이 웹훅 실패 시 재시도 정책 (토스페이 측 자동 재시도 여부 확인 필요) | 결제 안정성 | 🟡 |

| 3 | `/api/admin/reservations` 목록 최대 조회 건수 (limit 상한값 확정) | 성능 | 🟢 |



\---



\*다음 세션: Session 06 — 디자인 에이전트 다올 착수 또는 가온(개발) 착수\*

