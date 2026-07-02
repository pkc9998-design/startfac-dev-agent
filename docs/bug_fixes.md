\# BagDrop 버그 수정 코드



> 작성: 개발 에이전트 가온

> 세션: Session 14

> 최종 업데이트: 2026-07-01

> 기반: QA 에이전트 바른 Session 13 리포트

> 상태: ✅ Session 14 확정



\---



\## 수정 목록



| # | 버그 | 위험도 | 파일 |

|---|------|--------|------|

| #1 | 전화번호 형식 정규화 | 🔴 | `src/lib/validate.ts` |

| #2 | 토스페이 승인 타임아웃 | 🔴 | `src/lib/toss.ts` |

| #3 | confirm use\_date 재검증 확인 | 🟠 | `src/lib/validate.ts` |

| #5 | 웹훅 환경변수 uncaught exception | 🔴 | `src/app/api/payments/webhook/route.ts` |

| #6 | BO 접수 이용일 검증 | 🟠 | `src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts` |

| #8 | SDK 로드 실패 UX | 🟠 | `src/app/reserve/step4/page.tsx` |

| #9 | 접수 완료 자동 복귀 | 🟡 | `src/app/admin/checkin/page.tsx` |

| #10 | 백그라운드 폴링 낭비 | 🟡 | `src/app/admin/dashboard/page.tsx`, `src/app/admin/storage/page.tsx` |

| #11 | 하단 탭 예약 상세 미활성 | 🟡 | `src/components/bo/BottomNav.tsx` |

| #12 | FO API Rate Limiting | 🟠 | `src/middleware.ts`, `package.json` |



\---



\## 버그 #1 + #3 — `src/lib/validate.ts`



\### 수정 전 (문제 부분)



```typescript

// 버그 #1: 숫자만 허용 → 하이픈 포함 값이 들어오면 검증 실패

export const phoneSchema = z

&#x20; .string()

&#x20; .regex(/^01\[0-9]\\d{7,8}$/, '올바른 휴대폰 번호 형식이 아닙니다.')

&#x20; .length(11, '휴대폰 번호는 11자리여야 합니다.')



// 버그 #3 확인: confirmReservationSchema에 use\_date: useDateSchema 포함됨 (이미 올바름)

// → 그러나 phoneSchema 자체가 버그여서 confirm도 영향받음

```



\### 수정 후 (전체 파일)



```typescript

// src/lib/validate.ts

import { z } from 'zod'

import { formatInTimeZone } from 'date-fns-tz'

import { addDays, parseISO, startOfDay } from 'date-fns'



const TZ = 'Asia/Seoul'



// ──────────────────────────────────────────────────────────────

// 공통 유효성 규칙

// ──────────────────────────────────────────────────────────────



/\*\* 예약번호 형식: BD-YYYYMMDD-NNNN \*/

export const reservationNoSchema = z

&#x20; .string()

&#x20; .regex(/^BD-\\d{8}-\\d{4}$/, '올바른 예약번호 형식이 아닙니다. (예: BD-20260715-0001)')



/\*\*

&#x20;\* 전화번호 스키마 — 버그 #1 수정

&#x20;\*

&#x20;\* \[수정 내용]

&#x20;\* 기존: .regex()로 숫자만 허용 → 하이픈 포함 입력("010-1234-5678") 시 검증 실패

&#x20;\* 변경: .transform()으로 수신 즉시 숫자만 추출 후 검증

&#x20;\*       → "010-1234-5678", "01012345678" 모두 수용하여 DB에는 항상 숫자만 저장

&#x20;\*

&#x20;\* 영향 범위: prepare / confirm 두 API 모두 자동 적용

&#x20;\*/

export const phoneSchema = z

&#x20; .string()

&#x20; .transform((v) => v.replace(/\\D/g, ''))          // ✅ 수신 즉시 숫자만 추출

&#x20; .pipe(

&#x20;   z

&#x20;     .string()

&#x20;     .regex(/^01\[0-9]\\d{8}$/, '올바른 휴대폰 번호 형식이 아닙니다. (예: 01012345678)')

&#x20;     .length(11, '휴대폰 번호는 11자리여야 합니다.')

&#x20; )



/\*\* 이용일: 오늘 \~ 오늘+90일 \*/

export const useDateSchema = z

&#x20; .string()

&#x20; .regex(/^\\d{4}-\\d{2}-\\d{2}$/, '날짜 형식이 올바르지 않습니다. (YYYY-MM-DD)')

&#x20; .refine((date) => {

&#x20;   const todayKst   = parseISO(formatInTimeZone(new Date(), TZ, 'yyyy-MM-dd'))

&#x20;   const maxDate    = addDays(todayKst, 90)

&#x20;   const parsedDate = parseISO(date)

&#x20;   return parsedDate >= todayKst \&\& parsedDate <= maxDate

&#x20; }, '이용일은 오늘부터 90일 이내여야 합니다.')



// ──────────────────────────────────────────────────────────────

// API 요청 스키마

// ──────────────────────────────────────────────────────────────



/\*\* POST /api/reservations/prepare \*/

export const preparReservationSchema = z.object({

&#x20; terminal:       z.enum(\['T1', 'T2']),

&#x20; direction:      z.enum(\['DEPARTURE', 'ARRIVAL']),

&#x20; use\_date:       useDateSchema,

&#x20; bag\_count:      z.number().int().min(1).max(2),

&#x20; customer\_name:  z.string().min(1, '이름을 입력해 주세요.').max(20).trim(),

&#x20; customer\_phone: phoneSchema,    // ✅ transform 적용으로 자동 정규화

})



/\*\*

&#x20;\* POST /api/reservations/confirm — 버그 #3 확인

&#x20;\*

&#x20;\* \[확인 결과]

&#x20;\* use\_date: useDateSchema → 이미 포함됨 (기존 구현 올바름)

&#x20;\* customer\_phone: phoneSchema → 버그 #1 수정으로 함께 해결됨

&#x20;\*/

export const confirmReservationSchema = z.object({

&#x20; payment\_key:    z.string().min(1),

&#x20; toss\_order\_id:  z.string().min(1),

&#x20; amount:         z.literal(5000),

&#x20; terminal:       z.enum(\['T1', 'T2']),

&#x20; direction:      z.enum(\['DEPARTURE', 'ARRIVAL']),

&#x20; use\_date:       useDateSchema,    // ✅ 이미 존재 — 버그 #3 확인 완료

&#x20; bag\_count:      z.number().int().min(1).max(2),

&#x20; customer\_name:  z.string().min(1).max(20).trim(),

&#x20; customer\_phone: phoneSchema,      // ✅ 버그 #1 수정으로 함께 해결

})



/\*\* GET /api/admin/reservations (Query Params) \*/

export const adminReservationListSchema = z.object({

&#x20; date:     z.string().regex(/^\\d{4}-\\d{2}-\\d{2}$/).optional(),

&#x20; terminal: z.enum(\['T1', 'T2']).optional(),

&#x20; status:   z.string().optional(),

&#x20; page:     z.coerce.number().int().min(1).default(1),

&#x20; limit:    z.coerce.number().int().min(1).max(100).default(20),

})



/\*\* GET /api/admin/sales (Query Params) \*/

export const adminSalesSchema = z.object({

&#x20; period:   z.enum(\['daily', 'weekly', 'monthly']).default('daily'),

&#x20; date:     z.string().regex(/^\\d{4}-\\d{2}-\\d{2}$/).optional(),

&#x20; terminal: z.enum(\['T1', 'T2']).optional(),

&#x20; page:     z.coerce.number().int().min(1).default(1),

&#x20; limit:    z.coerce.number().int().min(1).max(100).default(50),

})



// ──────────────────────────────────────────────────────────────

// 헬퍼 함수

// ──────────────────────────────────────────────────────────────



/\*\* 전화번호 마스킹: 01012345678 → 010-\*\*\*\*-5678 \*/

export function maskPhone(phone: string): string {

&#x20; // 숫자만 있는 형태(11자리) 기준

&#x20; const digits = phone.replace(/\\D/g, '')

&#x20; if (digits.length !== 11) return phone

&#x20; return `${digits.slice(0, 3)}-\*\*\*\*-${digits.slice(7)}`

}



/\*\* KST 기준 오늘 날짜 문자열 반환: YYYY-MM-DD \*/

export function getTodayKst(): string {

&#x20; return formatInTimeZone(new Date(), TZ, 'yyyy-MM-dd')

}



/\*\* KST 기준 날짜 문자열 반환: YYYYMMDD (채번용) \*/

export function getDateKst(date?: Date): string {

&#x20; return formatInTimeZone(date ?? new Date(), TZ, 'yyyyMMdd')

}



/\*\* 이용일 기준 취소 가능 여부 판단 \*/

export function isCancellable(useDate: string): boolean {

&#x20; const todayKst = getTodayKst()

&#x20; return useDate > todayKst

}



/\*\* 공통 API 에러 응답 생성 \*/

export function errorResponse(

&#x20; code: string,

&#x20; message: string,

&#x20; status: number = 400

): Response {

&#x20; return Response.json(

&#x20;   { success: false, error: { code, message } },

&#x20;   { status }

&#x20; )

}



/\*\* 공통 API 성공 응답 생성 \*/

export function successResponse<T>(data: T, status: number = 200): Response {

&#x20; return Response.json({ success: true, data }, { status })

}



/\*\* Zod 파싱 실패 시 첫 번째 오류 메시지 추출 \*/

export function getZodError(error: z.ZodError): string {

&#x20; return error.errors\[0]?.message ?? '요청 형식이 올바르지 않습니다.'

}

```



\### 수정 설명



\*\*버그 #1\*\*: `phoneSchema`에 `.transform(v => v.replace(/\\D/g, ''))`를 추가해 수신 즉시 하이픈·공백 등 비숫자 문자를 제거합니다. 이후 `.pipe()`로 이어지는 검증에서 순수 숫자 형식을 확인합니다. FO STEP 3에서 화면 표시용 하이픈 포맷(`010-1234-5678`)이 실수로 전달되어도 서버에서 자동 정규화됩니다.



\*\*버그 #3\*\*: `confirmReservationSchema`에 `use\_date: useDateSchema`가 이미 포함되어 있음을 확인했습니다. 버그 #1 수정으로 `phoneSchema`도 함께 보강되어 confirm 경로도 함께 해결됩니다.



\---



\## 버그 #2 — `src/lib/toss.ts`



\### 수정 전 (문제 부분)



```typescript

// fetch()에 타임아웃 없어 Vercel 함수(기본 10초) 초과 시 502 발생

const response = await fetch(`${TOSS\_API\_BASE}/v1/payments/confirm`, {

&#x20; method: 'POST',

&#x20; headers: { ... },

&#x20; body:    JSON.stringify({ ... }),

&#x20; // ❌ signal 없음 → 무한 대기 가능

})

```



\### 수정 후 (전체 파일)



```typescript

// src/lib/toss.ts

import crypto from 'crypto'



const TOSS\_API\_BASE       = 'https://api.tosspayments.com'

const CONFIRM\_TIMEOUT\_MS  = 8\_000   // ✅ 8초 타임아웃 (Vercel 10초 한계 여유)



// ──────────────────────────────────────────────────────────────

// 타입

// ──────────────────────────────────────────────────────────────



export interface TossConfirmRequest {

&#x20; paymentKey: string

&#x20; orderId:    string

&#x20; amount:     number

}



export interface TossPaymentResponse {

&#x20; paymentKey:  string

&#x20; orderId:     string

&#x20; status:      string

&#x20; totalAmount: number

&#x20; method:      string

&#x20; approvedAt:  string

&#x20; \[key: string]: unknown

}



export interface TossErrorResponse {

&#x20; code:    string

&#x20; message: string

}



// ──────────────────────────────────────────────────────────────

// 핵심 함수

// ──────────────────────────────────────────────────────────────



function getTossAuthHeader(): string {

&#x20; const secretKey = process.env.TOSS\_SECRET\_KEY

&#x20; if (!secretKey) throw new Error('TOSS\_SECRET\_KEY 환경변수가 설정되지 않았습니다.')

&#x20; const encoded = Buffer.from(`${secretKey}:`).toString('base64')

&#x20; return `Basic ${encoded}`

}



/\*\*

&#x20;\* 결제 승인 API 호출 — 버그 #2 수정

&#x20;\*

&#x20;\* \[수정 내용]

&#x20;\* AbortController로 8초 타임아웃을 설정.

&#x20;\* 토스페이 서버가 느리거나 네트워크 장애 시 Vercel 함수가 무한 대기하지 않도록 방지.

&#x20;\* 타임아웃 발생 시 ok: false + TIMEOUT 에러 코드 반환 → confirm API에서 422 응답.

&#x20;\* 결제창에서 이미 승인이 완료됐을 수 있으므로 웹훅(DONE 이벤트)이 나중에 도착하여

&#x20;\* webhook handler가 복구 처리함.

&#x20;\*/

export async function confirmTossPayment(

&#x20; params: TossConfirmRequest

): Promise<{ ok: true; data: TossPaymentResponse } | { ok: false; error: TossErrorResponse }> {

&#x20; // ✅ AbortController로 8초 타임아웃 설정

&#x20; const controller = new AbortController()

&#x20; const timeoutId  = setTimeout(() => controller.abort(), CONFIRM\_TIMEOUT\_MS)



&#x20; try {

&#x20;   const response = await fetch(`${TOSS\_API\_BASE}/v1/payments/confirm`, {

&#x20;     method:  'POST',

&#x20;     headers: {

&#x20;       Authorization:  getTossAuthHeader(),

&#x20;       'Content-Type': 'application/json',

&#x20;     },

&#x20;     body: JSON.stringify({

&#x20;       paymentKey: params.paymentKey,

&#x20;       orderId:    params.orderId,

&#x20;       amount:     params.amount,

&#x20;     }),

&#x20;     signal: controller.signal,   // ✅ 타임아웃 신호 연결

&#x20;   })



&#x20;   clearTimeout(timeoutId)   // ✅ 성공 시 타임아웃 해제



&#x20;   const body = await response.json()



&#x20;   if (!response.ok) {

&#x20;     return { ok: false, error: body as TossErrorResponse }

&#x20;   }



&#x20;   return { ok: true, data: body as TossPaymentResponse }



&#x20; } catch (err) {

&#x20;   clearTimeout(timeoutId)



&#x20;   // ✅ AbortError(타임아웃)와 일반 네트워크 오류 구분

&#x20;   if (err instanceof Error \&\& err.name === 'AbortError') {

&#x20;     console.error('\[toss] 결제 승인 타임아웃 (8초 초과):', params.orderId)

&#x20;     return {

&#x20;       ok:    false,

&#x20;       error: {

&#x20;         code:    'TIMEOUT',

&#x20;         message: '결제 서버 응답 시간이 초과됐습니다. 잠시 후 다시 시도해 주세요.',

&#x20;       },

&#x20;     }

&#x20;   }



&#x20;   console.error('\[toss] 결제 승인 네트워크 오류:', err)

&#x20;   return {

&#x20;     ok:    false,

&#x20;     error: { code: 'NETWORK\_ERROR', message: '토스페이 API 연결에 실패했습니다.' },

&#x20;   }

&#x20; }

}



/\*\*

&#x20;\* 토스페이 웹훅 서명 검증

&#x20;\* 환경변수 미설정 시 throw 대신 false 반환으로 변경 (버그 #5 대응)

&#x20;\*/

export function verifyTossWebhookSignature(

&#x20; payload:           string,

&#x20; receivedSignature: string

): boolean {

&#x20; const secret = process.env.TOSS\_WEBHOOK\_SECRET

&#x20; if (!secret) {

&#x20;   // ✅ throw 대신 false 반환으로 변경 (버그 #5 수정 참조)

&#x20;   console.error('\[toss] TOSS\_WEBHOOK\_SECRET 환경변수가 설정되지 않았습니다.')

&#x20;   return false

&#x20; }



&#x20; const expectedSignature = crypto

&#x20;   .createHmac('sha256', secret)

&#x20;   .update(payload, 'utf-8')

&#x20;   .digest('base64')



&#x20; try {

&#x20;   return crypto.timingSafeEqual(

&#x20;     Buffer.from(receivedSignature),

&#x20;     Buffer.from(expectedSignature)

&#x20;   )

&#x20; } catch {

&#x20;   return false

&#x20; }

}



/\*\*

&#x20;\* toss\_order\_id 생성

&#x20;\* 형식: bagdrop\_{YYYYMMDD}\_{8자리 랜덤 hex}

&#x20;\*/

export function generateTossOrderId(dateStr: string): string {

&#x20; const randomHex = crypto.randomBytes(4).toString('hex')

&#x20; return `bagdrop\_${dateStr}\_${randomHex}`

}

```



\### 수정 설명



`AbortController`를 사용해 8초 타임아웃을 설정합니다. Vercel 서버리스 함수의 기본 실행 제한(10초)보다 2초 여유를 두어, 타임아웃 시 graceful하게 `TIMEOUT` 에러 코드를 반환합니다. 이 시점에 토스페이 서버에서는 실제 승인이 완료됐을 수 있으므로, 토스페이가 재발송하는 DONE 웹훅으로 복구 처리됩니다. 또한 `verifyTossWebhookSignature`에서 환경변수 미설정 시 `throw` 대신 `false` 반환으로 변경하여 버그 #5를 함께 처리합니다.



\---



\## 버그 #5 — `src/app/api/payments/webhook/route.ts`



\### 수정 전 (문제 부분)



```typescript

// 서명 검증 시 throw가 catch되지 않아 unhandled exception 가능

const signature = request.headers.get('Toss-Signature') ?? ''



if (!verifyTossWebhookSignature(rawBody, signature)) {

&#x20; // ❌ verifyTossWebhookSignature 내부에서 throw Error()가 발생하면

&#x20; //    이 조건문에 도달하지 못하고 함수 전체가 crash

&#x20; return errorResponse('INVALID\_REQUEST', '웹훅 서명이 유효하지 않습니다.', 400)

}

```



\### 수정 후 (전체 파일)



```typescript

// src/app/api/payments/webhook/route.ts

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { verifyTossWebhookSignature } from '@/lib/toss'

import { errorResponse, successResponse } from '@/lib/validate'



interface TossWebhookPayload {

&#x20; eventType: string

&#x20; data: {

&#x20;   paymentKey:  string

&#x20;   orderId:     string

&#x20;   status:      'DONE' | 'CANCELED' | 'PARTIAL\_CANCELED' | 'ABORTED' | 'EXPIRED'

&#x20;   totalAmount: number

&#x20;   \[key: string]: unknown

&#x20; }

}



export async function POST(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 요청 본문 수신 (raw text — 서명 검증에 원본 필요) ──────

&#x20; let rawBody: string

&#x20; try {

&#x20;   rawBody = await request.text()

&#x20; } catch {

&#x20;   return errorResponse('INVALID\_REQUEST', '요청 본문을 읽을 수 없습니다.')

&#x20; }



&#x20; // ── 2. 토스페이 웹훅 서명 검증 — 버그 #5 수정 ────────────────

&#x20; //

&#x20; // \[수정 내용]

&#x20; // 기존: verifyTossWebhookSignature()가 환경변수 미설정 시 throw → unhandled exception

&#x20; // 변경 1: toss.ts에서 throw 대신 false 반환으로 수정 (버그 #2 수정 파일 참조)

&#x20; // 변경 2: 여기서도 try-catch로 추가 방어 — 예기치 못한 예외 대비 이중 보호

&#x20; const signature = request.headers.get('Toss-Signature') ?? ''

&#x20; let isValidSignature: boolean



&#x20; try {

&#x20;   isValidSignature = verifyTossWebhookSignature(rawBody, signature)

&#x20; } catch (err) {

&#x20;   // ✅ 이중 방어 — toss.ts에서 throw가 발생하더라도 여기서 캐치

&#x20;   console.error('\[webhook] 서명 검증 중 예외 발생:', err)

&#x20;   return errorResponse('INVALID\_REQUEST', '웹훅 서명 검증에 실패했습니다.', 400)

&#x20; }



&#x20; if (!isValidSignature) {

&#x20;   console.warn('\[webhook] 서명 검증 실패. 위조된 웹훅일 수 있음. signature:', signature.slice(0, 20))

&#x20;   return errorResponse('INVALID\_REQUEST', '웹훅 서명이 유효하지 않습니다.', 400)

&#x20; }



&#x20; // ── 3. JSON 파싱 ──────────────────────────────────────────────

&#x20; let payload: TossWebhookPayload

&#x20; try {

&#x20;   payload = JSON.parse(rawBody) as TossWebhookPayload

&#x20; } catch {

&#x20;   return errorResponse('INVALID\_REQUEST', '웹훅 페이로드가 올바른 JSON 형식이 아닙니다.')

&#x20; }



&#x20; const { eventType, data } = payload



&#x20; if (eventType !== 'PAYMENT\_STATUS\_CHANGED') {

&#x20;   return successResponse({ received: true })

&#x20; }



&#x20; const { paymentKey, orderId, status } = data

&#x20; const db = getSupabaseClient()



&#x20; // ── 4. payments 레코드 조회 ───────────────────────────────────

&#x20; const { data: payment, error: fetchError } = await db

&#x20;   .from('payments')

&#x20;   .select('id, status, reservation\_id')

&#x20;   .eq('toss\_order\_id', orderId)

&#x20;   .maybeSingle()



&#x20; if (fetchError) {

&#x20;   console.error('\[webhook] payments 조회 실패:', fetchError)

&#x20;   return Response.json({ success: false }, { status: 500 })

&#x20; }



&#x20; if (!payment) {

&#x20;   console.warn('\[webhook] 대응하는 payments 레코드 없음. orderId:', orderId)

&#x20;   return successResponse({ received: true })

&#x20; }



&#x20; // ── 5. 멱등 처리 ─────────────────────────────────────────────

&#x20; if (status === 'DONE' \&\& payment.status === 'PAID') {

&#x20;   return successResponse({ received: true, skipped: true })

&#x20; }



&#x20; const alreadyFinalized = \['PAID', 'FAILED', 'CANCELLED'].includes(payment.status)



&#x20; // ── 6. 상태별 분기 처리 ───────────────────────────────────────

&#x20; switch (status) {

&#x20;   case 'DONE': {

&#x20;     if (alreadyFinalized) {

&#x20;       console.warn('\[webhook] DONE 이벤트이나 payments 상태 불일치:', {

&#x20;         orderId, currentStatus: payment.status,

&#x20;       })

&#x20;       return successResponse({ received: true })

&#x20;     }



&#x20;     const { error: updateError } = await db

&#x20;       .from('payments')

&#x20;       .update({

&#x20;         toss\_payment\_key: paymentKey,

&#x20;         status:           'PAID',

&#x20;         toss\_response:    data as unknown as Record<string, unknown>,

&#x20;         paid\_at:          new Date().toISOString(),

&#x20;       })

&#x20;       .eq('id', payment.id)

&#x20;       .eq('status', 'PENDING')



&#x20;     if (updateError) {

&#x20;       console.error('\[webhook] DONE payments UPDATE 실패:', updateError)

&#x20;       return Response.json({ success: false }, { status: 500 })

&#x20;     }



&#x20;     console.info('\[webhook] DONE 복구 처리 완료:', orderId)

&#x20;     break

&#x20;   }



&#x20;   case 'CANCELED': {

&#x20;     if (alreadyFinalized \&\& payment.status !== 'PAID') {

&#x20;       return successResponse({ received: true, skipped: true })

&#x20;     }



&#x20;     const { error: cancelError } = await db

&#x20;       .from('payments')

&#x20;       .update({

&#x20;         status:           'CANCELLED',

&#x20;         cancelled\_amount: (data.totalAmount as number) ?? 0,

&#x20;         cancelled\_at:     new Date().toISOString(),

&#x20;       })

&#x20;       .eq('id', payment.id)



&#x20;     if (cancelError) {

&#x20;       console.error('\[webhook] CANCELED payments UPDATE 실패:', cancelError)

&#x20;       return Response.json({ success: false }, { status: 500 })

&#x20;     }



&#x20;     console.info('\[webhook] CANCELED 처리 완료:', orderId)

&#x20;     break

&#x20;   }



&#x20;   case 'ABORTED':

&#x20;   case 'EXPIRED': {

&#x20;     if (alreadyFinalized) {

&#x20;       return successResponse({ received: true, skipped: true })

&#x20;     }



&#x20;     const { error: failError } = await db

&#x20;       .from('payments')

&#x20;       .update({ status: 'FAILED' })

&#x20;       .eq('id', payment.id)

&#x20;       .eq('status', 'PENDING')



&#x20;     if (failError) {

&#x20;       console.error('\[webhook] ABORTED/EXPIRED payments UPDATE 실패:', failError)

&#x20;       return Response.json({ success: false }, { status: 500 })

&#x20;     }



&#x20;     console.info('\[webhook] ABORTED/EXPIRED 처리 완료:', orderId, status)

&#x20;     break

&#x20;   }



&#x20;   case 'PARTIAL\_CANCELED':

&#x20;     console.warn('\[webhook] 예상치 못한 PARTIAL\_CANCELED 이벤트:', orderId)

&#x20;     break



&#x20;   default:

&#x20;     console.warn('\[webhook] 처리되지 않은 status:', status, orderId)

&#x20;     break

&#x20; }



&#x20; return successResponse({ received: true })

}

```



\### 수정 설명



이중 방어 전략을 적용합니다. 첫째, `toss.ts`의 `verifyTossWebhookSignature`에서 환경변수 미설정 시 `throw` 대신 `false`를 반환하도록 변경했습니다(버그 #2 수정 파일 참조). 둘째, `webhook/route.ts`에서도 서명 검증 호출을 `try-catch`로 감싸 예기치 않은 예외를 추가로 방어합니다. 둘 중 하나만 적용해도 버그는 해결되지만, 방어 코드는 겹쳐도 해가 없습니다.



\---



\## 버그 #6 — `src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts`



\### 수정 전 (문제 부분)



```typescript

// 상태 검증만 있고 이용일 비교 없음

const { allowed, errorResponse: statusError } = guardStatus(reservation!.status, 'RESERVED')

if (!allowed) return statusError!



// ❌ 내일 이용일인 예약을 오늘 접수하거나, 어제 이용일 예약을 오늘 접수하는 것도 가능

```



\### 수정 후 (전체 파일)



```typescript

// src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession, fetchReservation, guardStatus } from '@/lib/admin'

import { sendSmsAsync } from '@/lib/sms'

import { successResponse, errorResponse, getTodayKst } from '@/lib/validate'



interface RouteContext {

&#x20; params: { reservationNo: string }

}



export async function PATCH(

&#x20; \_request: NextRequest,

&#x20; context: RouteContext

): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { session, authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. 예약 조회 ─────────────────────────────────────────────

&#x20; const { reservation, fetchError } = await fetchReservation(context.params.reservationNo)

&#x20; if (fetchError) return fetchError



&#x20; // ── 3. 상태 검증 (RESERVED여야 접수 가능) ─────────────────────

&#x20; const { allowed, errorResponse: statusError } = guardStatus(reservation!.status, 'RESERVED')

&#x20; if (!allowed) return statusError!



&#x20; // ── 4. 이용일 검증 — 버그 #6 수정 ────────────────────────────

&#x20; //

&#x20; // \[수정 내용]

&#x20; // 기존: 이용일 vs 오늘 비교 없음 → 내일/어제 예약을 오늘 접수 가능

&#x20; // 변경: 이용일이 오늘이어야만 접수 허용 (±1일 정책은 운영자 확인 후 변경 가능)

&#x20; //

&#x20; // \[정책 근거]

&#x20; // - 당일보다 이른 접수: 고객이 짐을 맡기러 오지 않았는데 접수 처리되는 운영 오류 방지

&#x20; // - 이용일 지난 예약 접수: 만료된 예약 처리 방지 (환불 정책과 충돌 가능)

&#x20; const todayKst = getTodayKst()   // KST 기준 오늘 날짜 (YYYY-MM-DD)



&#x20; if (reservation!.use\_date !== todayKst) {

&#x20;   const isPast   = reservation!.use\_date < todayKst

&#x20;   const isFuture = reservation!.use\_date > todayKst

&#x20;   return errorResponse(

&#x20;     'INVALID\_DATE',

&#x20;     isPast

&#x20;       ? `이용일이 지난 예약입니다. (이용일: ${reservation!.use\_date})`

&#x20;       : `오늘 이용일이 아닌 예약입니다. (이용일: ${reservation!.use\_date})`,

&#x20;     409

&#x20;   )

&#x20; }



&#x20; const db          = getSupabaseClient()

&#x20; const checkedInAt = new Date().toISOString()



&#x20; // ── 5. reservations UPDATE (RESERVED → STORED) ────────────────

&#x20; const { data: updated, error: updateError } = await db

&#x20;   .from('reservations')

&#x20;   .update({

&#x20;     status:        'STORED',

&#x20;     checked\_in\_at: checkedInAt,

&#x20;   })

&#x20;   .eq('id', reservation!.id)

&#x20;   .eq('status', 'RESERVED')   // 낙관적 잠금

&#x20;   .select('id')

&#x20;   .maybeSingle()



&#x20; if (updateError) {

&#x20;   console.error('\[checkin] UPDATE 실패:', updateError)

&#x20;   return errorResponse('INTERNAL\_ERROR', '접수 처리 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; if (!updated) {

&#x20;   return errorResponse('ALREADY\_STORED', '이미 접수 처리된 예약입니다.', 409)

&#x20; }



&#x20; // ── 6. reservation\_logs INSERT ───────────────────────────────

&#x20; const { error: logError } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .insert({

&#x20;     reservation\_id: reservation!.id,

&#x20;     from\_status:    'RESERVED',

&#x20;     to\_status:      'STORED',

&#x20;     actor\_type:     'ADMIN',

&#x20;     actor\_id:       session!.id,

&#x20;     actor\_label:    '짐 접수',

&#x20;   })



&#x20; if (logError) {

&#x20;   console.error('\[checkin] LOG INSERT 실패:', logError)

&#x20; }



&#x20; // ── 7. SMS 발송 (비동기) ──────────────────────────────────────

&#x20; sendSmsAsync({

&#x20;   reservationId: reservation!.id,

&#x20;   eventType:     'CHECKIN\_COMPLETE',

&#x20;   phone:         reservation!.customer\_phone,

&#x20;   data: {

&#x20;     reservationNo: reservation!.reservation\_no,

&#x20;     terminal:      reservation!.terminal,

&#x20;     direction:     reservation!.direction,

&#x20;     useDate:       reservation!.use\_date,

&#x20;     bagCount:      reservation!.bag\_count,

&#x20;     checkedInAt,

&#x20;   },

&#x20; })



&#x20; // ── 8. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   reservation\_no: reservation!.reservation\_no,

&#x20;   status:         'STORED',

&#x20;   checked\_in\_at:  checkedInAt,

&#x20; })

}

```



\### 수정 설명



`getTodayKst()`로 KST 기준 오늘 날짜를 가져와 예약의 `use\_date`와 비교합니다. 과거 이용일과 미래 이용일을 각각 다른 에러 메시지로 구분하여 운영자가 상황을 명확히 파악할 수 있습니다. 향후 운영 정책에 따라 당일±1일 허용이 필요하면 조건을 `Math.abs(diffDays) <= 1`로 완화할 수 있습니다.



\---



\## 버그 #8 — `src/app/reserve/step4/page.tsx` (수정 부분만)



\### 수정 전 (문제 부분)



```typescript

// SDK 로드 실패 시 sdkReady가 false로 유지 → "로딩 중" UI가 무한 지속

// 사용자는 왜 버튼이 비활성인지 알 수 없음

const \[sdkReady, setSdkReady] = useState(false)



<Script

&#x20; src="https://js.tosspayments.com/v1/payment"

&#x20; onLoad={() => setSdkReady(true)}

&#x20; // ❌ onError 없음

&#x20; strategy="afterInteractive"

/>



<button disabled={isLoading || !sdkReady} ...>

&#x20; 토스페이로 결제하기

</button>

```



\### 수정 후 (step4/page.tsx 전체)



```typescript

'use client'



import { useState, useEffect } from 'react'

import { useRouter } from 'next/navigation'

import Script from 'next/script'

import StepBar from '@/components/fo/StepBar'

import { reservationStore } from '@/lib/reservation-store'

import { TERMINAL\_LABEL, DIRECTION\_LABEL, RESERVATION\_AMOUNT } from '@/constants'

import { format, parseISO } from 'date-fns'

import { ko } from 'date-fns/locale'



declare global {

&#x20; interface Window {

&#x20;   TossPayments?: (clientKey: string) => {

&#x20;     requestPayment: (

&#x20;       method: string,

&#x20;       options: Record<string, unknown>

&#x20;     ) => Promise<{ paymentKey: string; orderId: string; amount: number }>

&#x20;   }

&#x20; }

}



function SummaryRow({ label, value }: { label: string; value: string }) {

&#x20; return (

&#x20;   <div className="flex items-start justify-between gap-4 py-2.5">

&#x20;     <span className="shrink-0 text-sm text-gray-500">{label}</span>

&#x20;     <span className="text-right text-sm font-medium text-gray-900">{value}</span>

&#x20;   </div>

&#x20; )

}



function PaymentErrorModal({

&#x20; message,

&#x20; onClose,

}: {

&#x20; message: string

&#x20; onClose: () => void

}) {

&#x20; return (

&#x20;   <div className="fixed inset-0 z-50 flex items-end justify-center bg-black/50 px-4 pb-8">

&#x20;     <div className="w-full max-w-\[430px] rounded-2xl bg-white p-6 shadow-xl">

&#x20;       <div className="mb-1 flex items-center gap-2 text-red-600">

&#x20;         <svg className="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">

&#x20;           <path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-8-5a.75.75 0 01.75.75v4.5a.75.75 0 01-1.5 0v-4.5A.75.75 0 0110 5zm0 10a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" />

&#x20;         </svg>

&#x20;         <span className="font-semibold">결제 실패</span>

&#x20;       </div>

&#x20;       <p className="mb-5 mt-2 text-sm text-gray-600">{message}</p>

&#x20;       <button

&#x20;         onClick={onClose}

&#x20;         className="w-full rounded-xl bg-gray-900 py-3.5 text-sm font-semibold text-white active:bg-gray-800"

&#x20;       >

&#x20;         다시 시도하기

&#x20;       </button>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



export default function Step4Page() {

&#x20; const router = useRouter()

&#x20; const draft  = reservationStore.get()



&#x20; useEffect(() => {

&#x20;   const { terminal, direction, use\_date, bag\_count, customer\_name, customer\_phone } = draft

&#x20;   if (!terminal || !direction || !use\_date || !bag\_count || !customer\_name || !customer\_phone) {

&#x20;     router.replace('/reserve/step1')

&#x20;   }

&#x20; }, \[])



&#x20; const \[isLoading, setIsLoading] = useState(false)

&#x20; const \[errorMsg,  setErrorMsg]  = useState<string | null>(null)

&#x20; const \[sdkReady,  setSdkReady]  = useState(false)

&#x20; // ✅ 버그 #8 수정: SDK 로드 실패 상태 추가

&#x20; const \[sdkError,  setSdkError]  = useState(false)



&#x20; const dateLabel = draft.use\_date

&#x20;   ? format(parseISO(draft.use\_date), 'yyyy.MM.dd(eee)', { locale: ko })

&#x20;   : ''



&#x20; const phoneDisplay = draft.customer\_phone

&#x20;   ? draft.customer\_phone.replace(/(\\d{3})(\\d{4})(\\d{4})/, '$1-$2-$3')

&#x20;   : ''



&#x20; async function handlePayment() {

&#x20;   if (isLoading || !sdkReady || sdkError) return

&#x20;   setIsLoading(true)

&#x20;   setErrorMsg(null)



&#x20;   try {

&#x20;     const prepareRes = await fetch('/api/reservations/prepare', {

&#x20;       method:  'POST',

&#x20;       headers: { 'Content-Type': 'application/json' },

&#x20;       body: JSON.stringify({

&#x20;         terminal:       draft.terminal,

&#x20;         direction:      draft.direction,

&#x20;         use\_date:       draft.use\_date,

&#x20;         bag\_count:      draft.bag\_count,

&#x20;         customer\_name:  draft.customer\_name,

&#x20;         customer\_phone: draft.customer\_phone,

&#x20;       }),

&#x20;     })



&#x20;     const prepareJson = await prepareRes.json()

&#x20;     if (!prepareRes.ok || !prepareJson.success) {

&#x20;       throw new Error(prepareJson.error?.message ?? '결제 준비에 실패했습니다.')

&#x20;     }



&#x20;     const { toss\_order\_id, amount, order\_name } = prepareJson.data

&#x20;     reservationStore.set({ toss\_order\_id })



&#x20;     if (!window.TossPayments) {

&#x20;       throw new Error('결제 모듈을 불러오지 못했습니다. 잠시 후 다시 시도해 주세요.')

&#x20;     }



&#x20;     const toss    = window.TossPayments(process.env.NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY!)

&#x20;     const payment = await toss.requestPayment('카드', {

&#x20;       amount,

&#x20;       orderId:      toss\_order\_id,

&#x20;       orderName:    order\_name,

&#x20;       customerName: draft.customer\_name,

&#x20;     })



&#x20;     const confirmRes = await fetch('/api/reservations/confirm', {

&#x20;       method:  'POST',

&#x20;       headers: { 'Content-Type': 'application/json' },

&#x20;       body: JSON.stringify({

&#x20;         payment\_key:    payment.paymentKey,

&#x20;         toss\_order\_id:  payment.orderId,

&#x20;         amount:         payment.amount,

&#x20;         terminal:       draft.terminal,

&#x20;         direction:      draft.direction,

&#x20;         use\_date:       draft.use\_date,

&#x20;         bag\_count:      draft.bag\_count,

&#x20;         customer\_name:  draft.customer\_name,

&#x20;         customer\_phone: draft.customer\_phone,

&#x20;       }),

&#x20;     })



&#x20;     const confirmJson = await confirmRes.json()

&#x20;     if (!confirmRes.ok || !confirmJson.success) {

&#x20;       throw new Error(confirmJson.error?.message ?? '예약 처리에 실패했습니다.')

&#x20;     }



&#x20;     const reservationNo = confirmJson.data.reservation\_no

&#x20;     reservationStore.clear()

&#x20;     router.replace(`/reserve/complete?id=${reservationNo}`)



&#x20;   } catch (err) {

&#x20;     if (err instanceof Error \&\& err.message.includes('PAY\_PROCESS\_CANCELED')) {

&#x20;       setIsLoading(false)

&#x20;       return

&#x20;     }

&#x20;     setErrorMsg(

&#x20;       err instanceof Error ? err.message : '결제 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.'

&#x20;     )

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;   }

&#x20; }



&#x20; return (

&#x20;   <>

&#x20;     {/\* ✅ 버그 #8 수정: onError 핸들러 추가 \*/}

&#x20;     <Script

&#x20;       src="https://js.tosspayments.com/v1/payment"

&#x20;       onLoad={() => {

&#x20;         setSdkReady(true)

&#x20;         setSdkError(false)

&#x20;       }}

&#x20;       onError={() => {

&#x20;         setSdkReady(false)

&#x20;         setSdkError(true)   // ✅ SDK 로드 실패 상태 업데이트

&#x20;         console.error('\[toss] 결제 SDK 로드 실패')

&#x20;       }}

&#x20;       strategy="afterInteractive"

&#x20;     />



&#x20;     <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-gray-50">

&#x20;       <StepBar current={4} onBack={() => router.push('/reserve/step3')} />



&#x20;       <div className="flex-1 space-y-5 px-4 pb-40 pt-4">

&#x20;         <h2 className="text-base font-semibold text-gray-900">예약 내용을 확인해 주세요</h2>



&#x20;         {/\* 예약 요약 카드 \*/}

&#x20;         <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;           <SummaryRow label="터미널"    value={draft.terminal  ? TERMINAL\_LABEL\[draft.terminal]   ?? draft.terminal  : '-'} />

&#x20;           <SummaryRow label="이용 방향" value={draft.direction ? DIRECTION\_LABEL\[draft.direction] ?? draft.direction : '-'} />

&#x20;           <SummaryRow label="이용 날짜" value={dateLabel} />

&#x20;           <SummaryRow label="짐 개수"   value={`${draft.bag\_count ?? '-'}개`} />

&#x20;           <SummaryRow label="예약자"    value={draft.customer\_name ?? '-'} />

&#x20;           <SummaryRow label="연락처"    value={phoneDisplay} />

&#x20;         </div>



&#x20;         {/\* 결제 금액 \*/}

&#x20;         <div className="rounded-2xl bg-white px-5 py-4 shadow-sm">

&#x20;           <div className="flex items-center justify-between py-1">

&#x20;             <span className="text-sm text-gray-500">서비스 이용료</span>

&#x20;             <span className="text-sm text-gray-700">{RESERVATION\_AMOUNT.toLocaleString()}원</span>

&#x20;           </div>

&#x20;           <div className="my-2 border-t border-gray-100" />

&#x20;           <div className="flex items-center justify-between py-1">

&#x20;             <span className="text-sm font-semibold text-gray-900">최종 결제 금액</span>

&#x20;             <span className="text-lg font-bold text-blue-600">{RESERVATION\_AMOUNT.toLocaleString()}원</span>

&#x20;           </div>

&#x20;         </div>



&#x20;         {/\* ✅ 버그 #8 수정: SDK 오류 메시지 영역 \*/}

&#x20;         {sdkError \&\& (

&#x20;           <div className="rounded-2xl border border-red-200 bg-red-50 px-5 py-4">

&#x20;             <p className="text-sm font-medium text-red-700">결제 모듈 로드 실패</p>

&#x20;             <p className="mt-1 text-xs text-red-500">

&#x20;               네트워크 연결을 확인하고 페이지를 새로고침해 주세요.

&#x20;             </p>

&#x20;             <button

&#x20;               onClick={() => window.location.reload()}

&#x20;               className="mt-3 rounded-xl bg-red-600 px-4 py-2 text-xs font-semibold text-white active:bg-red-700"

&#x20;             >

&#x20;               페이지 새로고침

&#x20;             </button>

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* 결제 수단 버튼 \*/}

&#x20;         {!sdkError \&\& (

&#x20;           <div>

&#x20;             <h3 className="mb-3 text-sm font-semibold text-gray-700">결제 수단 선택</h3>

&#x20;             <button

&#x20;               onClick={handlePayment}

&#x20;               disabled={isLoading || !sdkReady}

&#x20;               className="

&#x20;                 flex w-full items-center justify-center gap-3

&#x20;                 rounded-2xl bg-blue-600 py-4 text-base font-bold text-white

&#x20;                 shadow-sm active:bg-blue-700

&#x20;                 disabled:cursor-not-allowed disabled:bg-gray-300

&#x20;               "

&#x20;             >

&#x20;               {isLoading ? (

&#x20;                 <>

&#x20;                   <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;                   <span>결제 처리 중…</span>

&#x20;                 </>

&#x20;               ) : !sdkReady ? (

&#x20;                 <>

&#x20;                   <span className="h-4 w-4 animate-spin rounded-full border-2 border-gray-400/30 border-t-gray-400" />

&#x20;                   <span className="text-gray-400">결제 모듈 로딩 중…</span>

&#x20;                 </>

&#x20;               ) : (

&#x20;                 <>

&#x20;                   <span className="text-xl">💳</span>

&#x20;                   <span>토스페이로 결제하기</span>

&#x20;                 </>

&#x20;               )}

&#x20;             </button>

&#x20;           </div>

&#x20;         )}



&#x20;         <p className="text-center text-xs text-gray-400">

&#x20;           결제 시{' '}

&#x20;           <a href="/terms" className="underline">이용약관</a>

&#x20;           {' '}및{' '}

&#x20;           <a href="/privacy" className="underline">개인정보처리방침</a>

&#x20;           에 동의한 것으로 간주합니다.

&#x20;         </p>

&#x20;       </div>

&#x20;     </div>



&#x20;     {errorMsg \&\& (

&#x20;       <PaymentErrorModal message={errorMsg} onClose={() => setErrorMsg(null)} />

&#x20;     )}

&#x20;   </>

&#x20; )

}

```



\### 수정 설명



`sdkError` 상태를 추가하고 `<Script onError>` 핸들러에서 이를 업데이트합니다. SDK 로드 실패 시 결제 버튼 영역 대신 오류 안내 카드와 새로고침 버튼을 표시합니다. 정상 로딩 중(`sdkReady: false, sdkError: false`)에는 스피너와 "로딩 중" 텍스트를 보여 상태를 명확히 구분합니다.



\---



\## 버그 #9 — `src/app/admin/checkin/page.tsx` (DoneFeedback 수정)



\### 수정 전 (문제 부분)



```typescript

// 버튼 클릭으로만 복귀 — 2초 자동 복귀 없음

function DoneFeedback({ data, mode, onNext }: ...) {

&#x20; return (

&#x20;   <div ...>

&#x20;     <h2>접수가 완료됐습니다!</h2>

&#x20;     // ❌ useEffect 없음 → 자동 복귀 없음

&#x20;     <button onClick={onNext}>다음 고객 접수하기</button>

&#x20;   </div>

&#x20; )

}

```



\### 수정 후 (DoneFeedback 컴포넌트만)



```typescript

// ✅ 버그 #9 수정: 2초 자동 복귀 + 카운트다운 표시 추가

function DoneFeedback({

&#x20; data,

&#x20; mode,

&#x20; onNext,

}: {

&#x20; data:   ReservationPreview

&#x20; mode:   'checkin' | 'checkout'

&#x20; onNext: () => void

}) {

&#x20; const isCheckin = mode === 'checkin'



&#x20; // ✅ 2초 후 자동 복귀 (명세 §3-3 준수)

&#x20; const \[countdown, setCountdown] = useState(2)



&#x20; useEffect(() => {

&#x20;   // 카운트다운 타이머

&#x20;   const countTimer = setInterval(() => {

&#x20;     setCountdown((prev) => {

&#x20;       if (prev <= 1) {

&#x20;         clearInterval(countTimer)

&#x20;         return 0

&#x20;       }

&#x20;       return prev - 1

&#x20;     })

&#x20;   }, 1\_000)



&#x20;   // 2초 후 자동 복귀

&#x20;   const autoTimer = setTimeout(onNext, 2\_000)



&#x20;   return () => {

&#x20;     clearInterval(countTimer)

&#x20;     clearTimeout(autoTimer)

&#x20;   }

&#x20; }, \[onNext])



&#x20; return (

&#x20;   <div className="fixed inset-0 z-50 flex flex-col items-center justify-center bg-white px-6">

&#x20;     <div className="mb-6 flex h-20 w-20 items-center justify-center rounded-full bg-green-100">

&#x20;       <svg className="h-10 w-10 text-green-600" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;         <path strokeLinecap="round" strokeLinejoin="round" d="M4.5 12.75l6 6 9-13.5" />

&#x20;       </svg>

&#x20;     </div>

&#x20;     <h2 className="mb-2 text-xl font-bold text-gray-900">

&#x20;       {isCheckin ? '접수가 완료됐습니다!' : '반환이 완료됐습니다!'}

&#x20;     </h2>

&#x20;     <p className="mb-1 text-sm text-gray-500">{data.reservation\_no}</p>

&#x20;     <p className="mb-6 text-sm text-gray-500">

&#x20;       {data.customer\_name} · 짐 {data.bag\_count}개

&#x20;     </p>



&#x20;     {/\* ✅ 카운트다운 표시 \*/}

&#x20;     <p className="mb-4 text-xs text-gray-400">

&#x20;       {countdown > 0 ? `${countdown}초 후 자동으로 돌아갑니다` : '돌아가는 중…'}

&#x20;     </p>



&#x20;     <button

&#x20;       onClick={onNext}

&#x20;       className="w-full max-w-xs rounded-2xl bg-blue-600 py-4 text-base font-bold text-white active:bg-blue-700"

&#x20;     >

&#x20;       다음 고객 {isCheckin ? '접수' : '반환'}하기

&#x20;     </button>

&#x20;   </div>

&#x20; )

}

```



\### 수정 설명



`useEffect`에서 `setTimeout(onNext, 2000)`으로 자동 복귀를 구현하고, 1초 간격의 `setInterval`로 카운트다운을 시각화합니다. 컴포넌트 언마운트 시 두 타이머를 모두 정리합니다. 운영자가 버튼을 먼저 누르면 `onNext`가 즉시 호출되어 타이머는 cleanup에서 해제됩니다.



\---



\## 버그 #10 — 대시보드·보관현황 (폴링 훅 분리)



\### 공통 훅 생성: `src/hooks/usePolling.ts`



```typescript

// src/hooks/usePolling.ts

// 버그 #10 수정: Page Visibility API를 활용해 백그라운드 탭에서 폴링 중단

import { useEffect, useCallback, useRef } from 'react'



interface UsePollingOptions {

&#x20; /\*\* 폴링 콜백 함수 \*/

&#x20; onPoll:       () => void | Promise<void>

&#x20; /\*\* 폴링 간격 (ms) \*/

&#x20; intervalMs:   number

&#x20; /\*\* 탭 복귀 시 즉시 갱신 여부 (기본: true) \*/

&#x20; refreshOnFocus?: boolean

}



/\*\*

&#x20;\* Page Visibility API 기반 스마트 폴링 훅

&#x20;\*

&#x20;\* \[버그 #10 수정]

&#x20;\* 기존: setInterval로 탭 비활성 중에도 폴링 계속 실행

&#x20;\* 변경: 탭 활성일 때만 폴링, 탭 복귀 시 즉시 갱신

&#x20;\*/

export function usePolling({

&#x20; onPoll,

&#x20; intervalMs,

&#x20; refreshOnFocus = true,

}: UsePollingOptions): void {

&#x20; // onPoll이 자주 바뀌어도 인터벌 재설정 없이 최신 함수 참조 유지

&#x20; const onPollRef = useRef(onPoll)

&#x20; useEffect(() => { onPollRef.current = onPoll }, \[onPoll])



&#x20; useEffect(() => {

&#x20;   // ── 주기적 폴링 (탭 활성일 때만) ──────────────────────────

&#x20;   const timer = setInterval(() => {

&#x20;     if (document.visibilityState === 'visible') {

&#x20;       onPollRef.current()

&#x20;     }

&#x20;   }, intervalMs)



&#x20;   // ── 탭 복귀 시 즉시 갱신 ──────────────────────────────────

&#x20;   function handleVisibilityChange() {

&#x20;     if (document.visibilityState === 'visible' \&\& refreshOnFocus) {

&#x20;       onPollRef.current()

&#x20;     }

&#x20;   }



&#x20;   if (refreshOnFocus) {

&#x20;     document.addEventListener('visibilitychange', handleVisibilityChange)

&#x20;   }



&#x20;   return () => {

&#x20;     clearInterval(timer)

&#x20;     if (refreshOnFocus) {

&#x20;       document.removeEventListener('visibilitychange', handleVisibilityChange)

&#x20;     }

&#x20;   }

&#x20; }, \[intervalMs, refreshOnFocus])

&#x20; // ↑ onPoll은 ref로 관리하므로 deps에서 제외

}

```



\### `src/app/admin/dashboard/page.tsx` 수정 부분



```typescript

// 기존 폴링 코드 교체

// ❌ 수정 전:

useEffect(() => {

&#x20; const timer = setInterval(() => {

&#x20;   fetchDashboard(true)

&#x20; }, POLL\_INTERVAL\_MS)

&#x20; return () => clearInterval(timer)

}, \[fetchDashboard])



// ✅ 수정 후: usePolling 훅으로 교체

import { usePolling } from '@/hooks/usePolling'



// fetchDashboard를 useCallback으로 안정화 (기존 동일)

const fetchDashboard = useCallback(async (showRefreshing = false) => { ... }, \[terminal])



// 초기 로드

useEffect(() => {

&#x20; setIsLoading(true)

&#x20; fetchDashboard()

}, \[fetchDashboard])



// ✅ 스마트 폴링 적용 (백그라운드 탭에서 중단, 탭 복귀 시 즉시 갱신)

usePolling({

&#x20; onPoll:     () => fetchDashboard(true),

&#x20; intervalMs: POLL\_INTERVAL\_MS,

})

```



\### `src/app/admin/storage/page.tsx` 수정 부분



```typescript

// ❌ 수정 전:

useEffect(() => {

&#x20; const t = setInterval(() => fetchStorage(true), POLL\_INTERVAL\_MS)

&#x20; return () => clearInterval(t)

}, \[fetchStorage])



// ✅ 수정 후:

import { usePolling } from '@/hooks/usePolling'



usePolling({

&#x20; onPoll:     () => fetchStorage(true),

&#x20; intervalMs: POLL\_INTERVAL\_MS,

})

```



\### 수정 설명



`usePolling` 공통 훅을 만들어 두 페이지에서 재사용합니다. `setInterval` 콜백에 `document.visibilityState === 'visible'` 조건을 추가해 백그라운드에서는 실행하지 않습니다. `visibilitychange` 이벤트로 탭 복귀 시 즉시 최신 데이터를 가져옵니다. `onPollRef`를 사용해 `onPoll` 함수가 변경되어도 인터벌을 재설정하지 않아 불필요한 타이머 재생성을 방지합니다.



\---



\## 버그 #11 — `src/components/bo/BottomNav.tsx`



\### 수정 전 (문제 부분)



```typescript

// /admin/reservations 경로에서 어떤 탭도 활성화되지 않음

function isActive(href: string): boolean {

&#x20; if (href === '/admin/dashboard') return pathname === href

&#x20; return pathname.startsWith(href)

}

```



\### 수정 후 (전체 파일)



```typescript

// src/components/bo/BottomNav.tsx

'use client'



import Link from 'next/link'

import { usePathname } from 'next/navigation'



interface NavItem {

&#x20; href:  string

&#x20; label: string

&#x20; icon:  React.ReactNode

}



const HomeIcon = () => (

&#x20; <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;   <path strokeLinecap="round" strokeLinejoin="round" d="M2.25 12l8.954-8.955c.44-.439 1.152-.439 1.591 0L21.75 12M4.5 9.75v10.125c0 .621.504 1.125 1.125 1.125H9.75v-4.875c0-.621.504-1.125 1.125-1.125h2.25c.621 0 1.125.504 1.125 1.125V21h4.125c.621 0 1.125-.504 1.125-1.125V9.75M8.25 21h8.25" />

&#x20; </svg>

)

const CheckinIcon = () => (

&#x20; <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;   <path strokeLinecap="round" strokeLinejoin="round" d="M9 3.75H6.912a2.25 2.25 0 00-2.15 1.588L2.35 13.177a2.25 2.25 0 00-.1.661V18a2.25 2.25 0 002.25 2.25h15A2.25 2.25 0 0021.75 18v-4.162c0-.224-.034-.447-.1-.661L19.24 5.338a2.25 2.25 0 00-2.15-1.588H15M2.25 13.5h3.86a2.25 2.25 0 012.012 1.244l.256.512a2.25 2.25 0 002.013 1.244h3.218a2.25 2.25 0 002.013-1.244l.256-.512a2.25 2.25 0 012.013-1.244h3.859M12 3v8.25m0 0l-3-3m3 3l3-3" />

&#x20; </svg>

)

const CheckoutIcon = () => (

&#x20; <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;   <path strokeLinecap="round" strokeLinejoin="round" d="M9 3.75H6.912a2.25 2.25 0 00-2.15 1.588L2.35 13.177a2.25 2.25 0 00-.1.661V18a2.25 2.25 0 002.25 2.25h15A2.25 2.25 0 0021.75 18v-4.162c0-.224-.034-.447-.1-.661L19.24 5.338a2.25 2.25 0 00-2.15-1.588H15M2.25 13.5h3.86a2.25 2.25 0 012.012 1.244l.256.512a2.25 2.25 0 002.013 1.244h3.218a2.25 2.25 0 002.013-1.244l.256-.512a2.25 2.25 0 012.013-1.244h3.859M12 3v8.25m0 0l3 3m-3-3l-3 3" />

&#x20; </svg>

)

const StorageIcon = () => (

&#x20; <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;   <path strokeLinecap="round" strokeLinejoin="round" d="M8.25 6.75h12M8.25 12h12m-12 5.25h12M3.75 6.75h.007v.008H3.75V6.75zm.375 0a.375.375 0 11-.75 0 .375.375 0 01.75 0zM3.75 12h.007v.008H3.75V12zm.375 0a.375.375 0 11-.75 0 .375.375 0 01.75 0zm-.375 5.25h.007v.008H3.75v-.008zm.375 0a.375.375 0 11-.75 0 .375.375 0 01.75 0z" />

&#x20; </svg>

)

const SalesIcon = () => (

&#x20; <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;   <path strokeLinecap="round" strokeLinejoin="round" d="M2.25 18.75a60.07 60.07 0 0115.797 2.101c.727.198 1.453-.342 1.453-1.096V18.75M3.75 4.5v.75A.75.75 0 013 6h-.75m0 0v-.375c0-.621.504-1.125 1.125-1.125H20.25M2.25 6v9m18-10.5v.75c0 .414.336.75.75.75h.75m-1.5-1.5h.375c.621 0 1.125.504 1.125 1.125v9.75c0 .621-.504 1.125-1.125 1.125h-.375m1.5-1.5H21a.75.75 0 00-.75.75v.75m0 0H3.75m0 0h-.375a1.125 1.125 0 01-1.125-1.125V15m1.5 1.5v-.75A.75.75 0 003 15h-.75M15 10.5a3 3 0 11-6 0 3 3 0 016 0zm3 0h.008v.008H18V10.5zm-12 0h.008v.008H6V10.5z" />

&#x20; </svg>

)



const NAV\_ITEMS: NavItem\[] = \[

&#x20; { href: '/admin/dashboard', label: '홈',   icon: <HomeIcon /> },

&#x20; { href: '/admin/checkin',   label: '접수', icon: <CheckinIcon /> },

&#x20; { href: '/admin/checkout',  label: '반환', icon: <CheckoutIcon /> },

&#x20; { href: '/admin/storage',   label: '현황', icon: <StorageIcon /> },

&#x20; { href: '/admin/sales',     label: '매출', icon: <SalesIcon /> },

]



/\*\*

&#x20;\* 탭 활성 판단 — 버그 #11 수정

&#x20;\*

&#x20;\* \[수정 내용]

&#x20;\* 기존: /admin/dashboard는 정확 매칭, 나머지는 startsWith

&#x20;\*       → /admin/reservations 경로에서 어떤 탭도 활성화 안 됨

&#x20;\* 변경: /admin/reservations/\* 경로에서 홈(대시보드) 탭을 활성으로 표시

&#x20;\*       → 예약 관리는 대시보드 영역의 하위 화면으로 간주

&#x20;\*/

function isActive(href: string, pathname: string): boolean {

&#x20; if (href === '/admin/dashboard') {

&#x20;   // ✅ 홈 탭: dashboard 자체 OR 예약 관리(/admin/reservations/\*) 경로

&#x20;   return (

&#x20;     pathname === '/admin/dashboard' ||

&#x20;     pathname.startsWith('/admin/reservations')

&#x20;   )

&#x20; }

&#x20; return pathname.startsWith(href)

}



export default function BottomNav() {

&#x20; const pathname = usePathname()



&#x20; return (

&#x20;   <nav className="fixed bottom-0 left-0 right-0 z-50 border-t border-gray-200 bg-white pb-safe">

&#x20;     <div className="mx-auto flex max-w-\[430px] items-stretch">

&#x20;       {NAV\_ITEMS.map((item) => {

&#x20;         const active = isActive(item.href, pathname)

&#x20;         return (

&#x20;           <Link

&#x20;             key={item.href}

&#x20;             href={item.href}

&#x20;             className={`

&#x20;               relative flex flex-1 flex-col items-center gap-1 px-1 py-2.5

&#x20;               transition-colors

&#x20;               ${active ? 'text-blue-600' : 'text-gray-400 hover:text-gray-600'}

&#x20;             `}

&#x20;             aria-current={active ? 'page' : undefined}

&#x20;           >

&#x20;             {/\* 상단 활성 인디케이터 \*/}

&#x20;             {active \&\& (

&#x20;               <span className="absolute -top-px left-1/2 h-0.5 w-8 -translate-x-1/2 rounded-full bg-blue-600" />

&#x20;             )}

&#x20;             <span className={active ? 'text-blue-600' : 'text-gray-400'}>

&#x20;               {item.icon}

&#x20;             </span>

&#x20;             <span className={`text-\[11px] font-medium leading-none ${active ? 'text-blue-600' : 'text-gray-400'}`}>

&#x20;               {item.label}

&#x20;             </span>

&#x20;           </Link>

&#x20;         )

&#x20;       })}

&#x20;     </div>

&#x20;   </nav>

&#x20; )

}

```



\### 수정 설명



`isActive` 함수를 순수 함수로 분리하고 `/admin/reservations` 경로도 홈 탭 활성 조건에 포함했습니다. 예약 관리 화면은 대시보드에서 진입하는 하위 영역이므로 홈 탭을 활성화하는 것이 UX 일관성에 맞습니다.



\---



\## 버그 #12 — Rate Limiting



\### 패키지 추가



```bash

\# Upstash Redis 기반 Rate Limiting (Vercel KV와 호환)

npm install @upstash/ratelimit @upstash/redis

```



\### 환경변수 추가 (`.env.example`, `.env.local`)



```bash

\# ─── Upstash Redis (Rate Limiting) ──────────────────────────

UPSTASH\_REDIS\_REST\_URL=

UPSTASH\_REDIS\_REST\_TOKEN=

```



\### `src/middleware.ts` 전체 교체



```typescript

// src/middleware.ts

import { NextResponse } from 'next/server'

import type { NextRequest } from 'next/server'

import { getToken } from 'next-auth/jwt'



// ──────────────────────────────────────────────────────────────

// Rate Limiting 설정 — 버그 #12 수정

// ──────────────────────────────────────────────────────────────



// Upstash가 설정된 경우에만 Rate Limiting 활성화

// 미설정 시 기능은 정상 동작하되 Rate Limiting만 비활성

async function applyRateLimit(

&#x20; request: NextRequest

): Promise<Response | null> {

&#x20; const redisUrl   = process.env.UPSTASH\_REDIS\_REST\_URL

&#x20; const redisToken = process.env.UPSTASH\_REDIS\_REST\_TOKEN



&#x20; // 환경변수 미설정 시 Rate Limiting 스킵 (개발 환경 편의)

&#x20; if (!redisUrl || !redisToken) return null



&#x20; try {

&#x20;   const { Ratelimit } = await import('@upstash/ratelimit')

&#x20;   const { Redis }     = await import('@upstash/redis')



&#x20;   const redis = new Redis({ url: redisUrl, token: redisToken })



&#x20;   // FO API 경로별 Rate Limiting 정책

&#x20;   const { pathname } = request.nextUrl

&#x20;   let limiter: InstanceType<typeof Ratelimit> | null = null



&#x20;   if (pathname === '/api/reservations/prepare') {

&#x20;     // 결제 준비: IP당 분당 5회 (과도한 PENDING payments 방지)

&#x20;     limiter = new Ratelimit({

&#x20;       redis,

&#x20;       limiter:   Ratelimit.slidingWindow(5, '1 m'),

&#x20;       analytics: false,

&#x20;       prefix:    'rl:prepare',

&#x20;     })

&#x20;   } else if (pathname === '/api/reservations/confirm') {

&#x20;     // 결제 승인: IP당 분당 10회

&#x20;     limiter = new Ratelimit({

&#x20;       redis,

&#x20;       limiter:   Ratelimit.slidingWindow(10, '1 m'),

&#x20;       analytics: false,

&#x20;       prefix:    'rl:confirm',

&#x20;     })

&#x20;   } else if (pathname.startsWith('/api/reservations/') \&\& pathname.endsWith('/cancel')) {

&#x20;     // 취소: IP당 분당 10회

&#x20;     limiter = new Ratelimit({

&#x20;       redis,

&#x20;       limiter:   Ratelimit.slidingWindow(10, '1 m'),

&#x20;       analytics: false,

&#x20;       prefix:    'rl:cancel',

&#x20;     })

&#x20;   }



&#x20;   if (!limiter) return null



&#x20;   // IP 추출 (Vercel 환경에서는 x-forwarded-for 헤더 사용)

&#x20;   const ip = request.ip

&#x20;     ?? request.headers.get('x-forwarded-for')?.split(',')\[0]?.trim()

&#x20;     ?? '127.0.0.1'



&#x20;   const { success, limit, remaining, reset } = await limiter.limit(ip)



&#x20;   if (!success) {

&#x20;     const resetDate = new Date(reset)

&#x20;     const waitSec   = Math.ceil((resetDate.getTime() - Date.now()) / 1000)



&#x20;     console.warn('\[rate-limit] 차단:', { ip, pathname, limit, remaining })



&#x20;     return new Response(

&#x20;       JSON.stringify({

&#x20;         success: false,

&#x20;         error: {

&#x20;           code:    'TOO\_MANY\_REQUESTS',

&#x20;           message: `요청이 너무 많습니다. ${waitSec}초 후에 다시 시도해 주세요.`,

&#x20;         },

&#x20;       }),

&#x20;       {

&#x20;         status:  429,

&#x20;         headers: {

&#x20;           'Content-Type':     'application/json',

&#x20;           'X-RateLimit-Limit':     String(limit),

&#x20;           'X-RateLimit-Remaining': String(remaining),

&#x20;           'X-RateLimit-Reset':     String(reset),

&#x20;           'Retry-After':           String(waitSec),

&#x20;         },

&#x20;       }

&#x20;     )

&#x20;   }

&#x20; } catch (err) {

&#x20;   // Rate Limiting 자체 오류 시 서비스 중단 방지 — 통과시킴

&#x20;   console.error('\[rate-limit] 오류 (통과):', err)

&#x20; }



&#x20; return null   // null = 계속 진행

}



// ──────────────────────────────────────────────────────────────

// 미들웨어 메인

// ──────────────────────────────────────────────────────────────



export async function middleware(request: NextRequest): Promise<NextResponse | Response> {

&#x20; const { pathname } = request.nextUrl



&#x20; // ── 1. FO API Rate Limiting (버그 #12 수정) ────────────────

&#x20; if (

&#x20;   pathname === '/api/reservations/prepare' ||

&#x20;   pathname === '/api/reservations/confirm'  ||

&#x20;   (pathname.startsWith('/api/reservations/') \&\& pathname.endsWith('/cancel'))

&#x20; ) {

&#x20;   const rateLimitResponse = await applyRateLimit(request)

&#x20;   if (rateLimitResponse) return rateLimitResponse

&#x20; }



&#x20; // ── 2. BO 어드민 페이지 접근 제어 ─────────────────────────

&#x20; if (pathname.startsWith('/admin') \&\& pathname !== '/admin/login') {

&#x20;   const token = await getToken({

&#x20;     req:    request,

&#x20;     secret: process.env.NEXTAUTH\_SECRET,

&#x20;   })



&#x20;   if (!token) {

&#x20;     const loginUrl = new URL('/admin/login', request.url)

&#x20;     loginUrl.searchParams.set('callbackUrl', pathname)

&#x20;     return NextResponse.redirect(loginUrl)

&#x20;   }

&#x20; }



&#x20; // ── 3. 로그인 상태에서 /admin/login 접근 → 대시보드 리디렉트

&#x20; if (pathname === '/admin/login') {

&#x20;   const token = await getToken({

&#x20;     req:    request,

&#x20;     secret: process.env.NEXTAUTH\_SECRET,

&#x20;   })



&#x20;   if (token) {

&#x20;     return NextResponse.redirect(new URL('/admin/dashboard', request.url))

&#x20;   }

&#x20; }



&#x20; return NextResponse.next()

}



export const config = {

&#x20; matcher: \[

&#x20;   '/admin/:path\*',

&#x20;   '/api/reservations/prepare',

&#x20;   '/api/reservations/confirm',

&#x20;   '/api/reservations/:path\*/cancel',

&#x20; ],

}

```



\### 수정 설명



Upstash Redis 기반 `@upstash/ratelimit`을 사용합니다. 환경변수 미설정 시 조용히 스킵하므로 개발 환경에서는 별도 설정 없이 동작합니다. Rate Limiting 자체에 오류가 생겨도 `try-catch`로 감싸 서비스 중단을 방지합니다. 경로별로 정책을 다르게 적용하며 `prepare`(분당 5회)가 가장 엄격합니다. `Retry-After` 헤더를 응답에 포함해 클라이언트가 재시도 시점을 알 수 있도록 했습니다.



\---



\## 수정 사항 적용 순서



```bash

\# 1. 패키지 설치 (버그 #12)

npm install @upstash/ratelimit @upstash/redis



\# 2. 파일 수정 (아래 순서대로)

\# src/lib/validate.ts          ← 버그 #1, #3

\# src/lib/toss.ts              ← 버그 #2 (+ #5 일부)

\# src/app/api/payments/webhook/route.ts   ← 버그 #5

\# src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts  ← 버그 #6

\# src/app/reserve/step4/page.tsx          ← 버그 #8

\# src/app/admin/checkin/page.tsx          ← 버그 #9

\# src/hooks/usePolling.ts      ← 버그 #10 (신규 파일)

\# src/app/admin/dashboard/page.tsx        ← 버그 #10

\# src/app/admin/storage/page.tsx          ← 버그 #10

\# src/components/bo/BottomNav.tsx         ← 버그 #11

\# src/middleware.ts             ← 버그 #12



\# 3. 환경변수 추가 (.env.local)

\# UPSTASH\_REDIS\_REST\_URL=...

\# UPSTASH\_REDIS\_REST\_TOKEN=...



\# 4. 빌드 확인

npm run build



\# 5. 단위 테스트 (phoneSchema 정규화 확인)

\# npm test src/lib/validate.test.ts

```



\---



\## Session 14 체크리스트



| # | 버그 | 수정 파일 | 상태 |

|---|------|----------|------|

| #1 | 전화번호 형식 정규화 | `validate.ts` | ✅ |

| #2 | 토스페이 타임아웃 | `toss.ts` | ✅ |

| #3 | confirm use\_date 재검증 | `validate.ts` (기존 정상 확인) | ✅ |

| #5 | 웹훅 uncaught exception | `toss.ts` + `webhook/route.ts` | ✅ |

| #6 | BO 접수 이용일 검증 | `checkin/route.ts` | ✅ |

| #8 | SDK 로드 실패 UX | `step4/page.tsx` | ✅ |

| #9 | 접수 완료 자동 복귀 | `checkin/page.tsx` | ✅ |

| #10 | 백그라운드 폴링 | `usePolling.ts` (신규) + `dashboard` + `storage` | ✅ |

| #11 | 하단 탭 미활성 | `BottomNav.tsx` | ✅ |

| #12 | FO Rate Limiting | `middleware.ts` | ✅ |

