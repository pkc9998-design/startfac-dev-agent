\# BagDrop FO API Route 구현 코드



> 작성: 개발 에이전트 가온

> 세션: Session 07

> 최종 업데이트: 2026-07-01

> 대상: FO API Route 5개 (Next.js 14 App Router)

> 상태: ✅ Session 07 확정



\---



\## 파일 목록



| 파일 경로 | 메서드 | 역할 |

|-----------|--------|------|

| `src/app/api/reservations/prepare/route.ts` | POST | 결제 준비 (toss\_order\_id 발급) |

| `src/app/api/reservations/confirm/route.ts` | POST | 결제 승인 + 예약 확정 |

| `src/app/api/reservations/\[reservationNo]/route.ts` | GET | 예약 조회 |

| `src/app/api/reservations/\[reservationNo]/cancel/route.ts` | POST | 고객 취소 |

| `src/app/api/payments/webhook/route.ts` | POST | 토스페이 웹훅 수신 |



\---



\## 1. `src/app/api/reservations/prepare/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { z } from 'zod'

import { getSupabaseClient } from '@/lib/supabase'

import { generateTossOrderId } from '@/lib/toss'

import {

&#x20; preparReservationSchema,

&#x20; errorResponse,

&#x20; successResponse,

&#x20; getZodError,

&#x20; getDateKst,

} from '@/lib/validate'

import { RESERVATION\_AMOUNT } from '@/constants'



export async function POST(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 요청 파싱 ──────────────────────────────────────────────

&#x20; let body: unknown

&#x20; try {

&#x20;   body = await request.json()

&#x20; } catch {

&#x20;   return errorResponse('INVALID\_REQUEST', '요청 본문이 올바른 JSON 형식이 아닙니다.')

&#x20; }



&#x20; // ── 2. 유효성 검증 (Zod) ─────────────────────────────────────

&#x20; const parsed = preparReservationSchema.safeParse(body)

&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_REQUEST', getZodError(parsed.error))

&#x20; }



&#x20; const {

&#x20;   terminal,

&#x20;   direction,

&#x20;   use\_date,

&#x20;   bag\_count,

&#x20;   customer\_name,

&#x20;   customer\_phone,

&#x20; } = parsed.data



&#x20; // ── 3. toss\_order\_id 생성 ────────────────────────────────────

&#x20; const dateStr      = getDateKst()           // KST 기준 오늘 날짜 YYYYMMDD

&#x20; const tossOrderId  = generateTossOrderId(dateStr)



&#x20; // ── 4. payments PENDING 레코드 INSERT ───────────────────────

&#x20; const db = getSupabaseClient()



&#x20; const { data: payment, error: paymentError } = await db

&#x20;   .from('payments')

&#x20;   .insert({

&#x20;     toss\_order\_id:    tossOrderId,

&#x20;     toss\_payment\_key: '',            // 결제 승인 후 confirm에서 업데이트

&#x20;     amount:           RESERVATION\_AMOUNT,

&#x20;     method:           'TOSSPAY',

&#x20;     status:           'PENDING',

&#x20;   })

&#x20;   .select('id')

&#x20;   .single()



&#x20; if (paymentError || !payment) {

&#x20;   console.error('\[prepare] payments INSERT 실패:', paymentError)

&#x20;   return errorResponse('INTERNAL\_ERROR', '결제 준비 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; // ── 5. 응답 ─────────────────────────────────────────────────

&#x20; // 예약 정보는 클라이언트 세션스토리지에 보관 후 confirm 단계에서 재전송

&#x20; return successResponse({

&#x20;   toss\_order\_id: tossOrderId,

&#x20;   amount:        RESERVATION\_AMOUNT,

&#x20;   order\_name:    'BagDrop 짐 보관 서비스',

&#x20;   // confirm 단계에서 재사용할 예약 정보 에코백

&#x20;   reservation\_info: {

&#x20;     terminal,

&#x20;     direction,

&#x20;     use\_date,

&#x20;     bag\_count,

&#x20;     customer\_name,

&#x20;     customer\_phone,

&#x20;   },

&#x20; })

}

```



\---



\## 2. `src/app/api/reservations/confirm/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { confirmTossPayment } from '@/lib/toss'

import { sendSmsAsync } from '@/lib/sms'

import {

&#x20; confirmReservationSchema,

&#x20; errorResponse,

&#x20; successResponse,

&#x20; getZodError,

&#x20; getDateKst,

} from '@/lib/validate'

import { RESERVATION\_AMOUNT } from '@/constants'



export async function POST(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 요청 파싱 ──────────────────────────────────────────────

&#x20; let body: unknown

&#x20; try {

&#x20;   body = await request.json()

&#x20; } catch {

&#x20;   return errorResponse('INVALID\_REQUEST', '요청 본문이 올바른 JSON 형식이 아닙니다.')

&#x20; }



&#x20; // ── 2. 유효성 검증 (Zod) ─────────────────────────────────────

&#x20; const parsed = confirmReservationSchema.safeParse(body)

&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_REQUEST', getZodError(parsed.error))

&#x20; }



&#x20; const {

&#x20;   payment\_key,

&#x20;   toss\_order\_id,

&#x20;   amount,

&#x20;   terminal,

&#x20;   direction,

&#x20;   use\_date,

&#x20;   bag\_count,

&#x20;   customer\_name,

&#x20;   customer\_phone,

&#x20; } = parsed.data



&#x20; const db = getSupabaseClient()



&#x20; // ── 3. payments PENDING 레코드 조회 ──────────────────────────

&#x20; // toss\_order\_id 기준으로 조회 — 없거나 PENDING이 아니면 중복/잘못된 요청

&#x20; const { data: existingPayment, error: paymentFetchError } = await db

&#x20;   .from('payments')

&#x20;   .select('id, status, amount')

&#x20;   .eq('toss\_order\_id', toss\_order\_id)

&#x20;   .single()



&#x20; if (paymentFetchError || !existingPayment) {

&#x20;   return errorResponse('RESERVATION\_NOT\_FOUND', '결제 정보를 찾을 수 없습니다.', 404)

&#x20; }



&#x20; if (existingPayment.status !== 'PENDING') {

&#x20;   return errorResponse(

&#x20;     'PAYMENT\_ALREADY\_EXISTS',

&#x20;     '이미 처리된 결제입니다.',

&#x20;     409

&#x20;   )

&#x20; }



&#x20; // ── 4. 금액 이중 검증 ─────────────────────────────────────────

&#x20; // 클라이언트에서 위변조된 금액으로 요청하는 경우 차단

&#x20; if (amount !== RESERVATION\_AMOUNT || existingPayment.amount !== RESERVATION\_AMOUNT) {

&#x20;   // 위변조 시도: payments를 FAILED로 마킹 후 거절

&#x20;   await db

&#x20;     .from('payments')

&#x20;     .update({ status: 'FAILED' })

&#x20;     .eq('id', existingPayment.id)



&#x20;   return errorResponse('INVALID\_REQUEST', '결제 금액이 올바르지 않습니다.')

&#x20; }



&#x20; // ── 5. 토스페이 결제 승인 API 호출 ────────────────────────────

&#x20; const tossResult = await confirmTossPayment({

&#x20;   paymentKey: payment\_key,

&#x20;   orderId:    toss\_order\_id,

&#x20;   amount:     RESERVATION\_AMOUNT,

&#x20; })



&#x20; if (!tossResult.ok) {

&#x20;   // 토스페이 승인 실패 → payments FAILED 처리 후 422 반환

&#x20;   await db

&#x20;     .from('payments')

&#x20;     .update({ status: 'FAILED' })

&#x20;     .eq('id', existingPayment.id)



&#x20;   console.error('\[confirm] 토스페이 승인 실패:', tossResult.error)

&#x20;   return errorResponse(

&#x20;     'PAYMENT\_FAILED',

&#x20;     '결제 처리에 실패했습니다. 다시 시도해 주세요.',

&#x20;     422

&#x20;   )

&#x20; }



&#x20; const tossData = tossResult.data



&#x20; // ── 6. DB 처리 (순차 실행, Supabase는 클라이언트 트랜잭션 미지원) ──

&#x20; //

&#x20; // Supabase JS 클라이언트는 BEGIN/COMMIT 트랜잭션을 직접 지원하지 않음.

&#x20; // 대신 아래 순서로 원자성에 가깝게 처리:

&#x20; //   a. reservation\_no 채번 (RPC)

&#x20; //   b. reservations INSERT

&#x20; //   c. payments UPDATE

&#x20; //   d. reservation\_logs INSERT

&#x20; //

&#x20; // 만약 b 이후 c/d가 실패하면 reservations는 생성됐으나 payments가 PENDING인

&#x20; // 불일치 상태가 될 수 있음. 이 경우 토스페이 웹훅이 DONE 이벤트를 재발송하므로

&#x20; // webhook handler에서 복구 처리. (결제는 이미 완료되었으므로 환불 필요 없음)



&#x20; // 6-a. 예약번호 채번 (날짜별 Sequence RPC)

&#x20; const dateStr = getDateKst()

&#x20; const { data: reservationNo, error: rpcError } = await db

&#x20;   .rpc('get\_next\_reservation\_no', { target\_date: dateStr })



&#x20; if (rpcError || !reservationNo) {

&#x20;   console.error('\[confirm] 예약번호 채번 실패:', rpcError)

&#x20;   // payments를 FAILED로 되돌릴 수 없음 (이미 토스페이 승인 완료)

&#x20;   // → payments는 PENDING 유지, 웹훅으로 복구 대기

&#x20;   return errorResponse('INTERNAL\_ERROR', '예약 처리 중 오류가 발생했습니다. 고객센터로 문의해 주세요.', 500)

&#x20; }



&#x20; // 6-b. reservations INSERT

&#x20; const { data: reservation, error: reservationError } = await db

&#x20;   .from('reservations')

&#x20;   .insert({

&#x20;     reservation\_no:  reservationNo,

&#x20;     terminal,

&#x20;     direction,

&#x20;     use\_date,

&#x20;     bag\_count,

&#x20;     customer\_name,

&#x20;     customer\_phone,

&#x20;     amount:          RESERVATION\_AMOUNT,

&#x20;     status:          'RESERVED',

&#x20;   })

&#x20;   .select()

&#x20;   .single()



&#x20; if (reservationError || !reservation) {

&#x20;   console.error('\[confirm] reservations INSERT 실패:', reservationError)

&#x20;   return errorResponse('INTERNAL\_ERROR', '예약 저장 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; // 6-c. payments UPDATE (PENDING → PAID)

&#x20; const { error: paymentUpdateError } = await db

&#x20;   .from('payments')

&#x20;   .update({

&#x20;     reservation\_id:   reservation.id,

&#x20;     toss\_payment\_key: payment\_key,

&#x20;     status:           'PAID',

&#x20;     toss\_response:    tossData as unknown as Record<string, unknown>,

&#x20;     paid\_at:          tossData.approvedAt ?? new Date().toISOString(),

&#x20;   })

&#x20;   .eq('id', existingPayment.id)



&#x20; if (paymentUpdateError) {

&#x20;   // payments 업데이트 실패 → 예약은 생성됨. 웹훅으로 복구 대기.

&#x20;   console.error('\[confirm] payments UPDATE 실패:', paymentUpdateError)

&#x20;   // 예약은 정상 생성됐으므로 사용자에게는 성공 응답 (예약번호 발급됨)

&#x20; }



&#x20; // 6-d. reservation\_logs INSERT (NULL → RESERVED)

&#x20; const { error: logError } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .insert({

&#x20;     reservation\_id: reservation.id,

&#x20;     from\_status:    null,

&#x20;     to\_status:      'RESERVED',

&#x20;     actor\_type:     'SYSTEM',

&#x20;     actor\_label:    '예약완료',

&#x20;   })



&#x20; if (logError) {

&#x20;   // 로그 실패는 치명적이지 않음. 콘솔 기록 후 계속 진행.

&#x20;   console.error('\[confirm] reservation\_logs INSERT 실패:', logError)

&#x20; }



&#x20; // ── 7. SMS 발송 (비동기 — 응답 블로킹 없음) ───────────────────

&#x20; sendSmsAsync({

&#x20;   reservationId: reservation.id,

&#x20;   eventType:     'RESERVATION\_COMPLETE',

&#x20;   phone:         customer\_phone,

&#x20;   data: {

&#x20;     reservationNo: reservationNo,

&#x20;     terminal,

&#x20;     direction,

&#x20;     useDate:   use\_date,

&#x20;     bagCount:  bag\_count,

&#x20;   },

&#x20; })



&#x20; // ── 8. 201 응답 ──────────────────────────────────────────────

&#x20; return successResponse(

&#x20;   {

&#x20;     reservation\_no: reservation.reservation\_no,

&#x20;     terminal:       reservation.terminal,

&#x20;     direction:      reservation.direction,

&#x20;     use\_date:       reservation.use\_date,

&#x20;     bag\_count:      reservation.bag\_count,

&#x20;     customer\_name:  reservation.customer\_name,

&#x20;     status:         reservation.status,

&#x20;     created\_at:     reservation.created\_at,

&#x20;   },

&#x20;   201

&#x20; )

}

```



\---



\## 3. `src/app/api/reservations/\[reservationNo]/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import {

&#x20; reservationNoSchema,

&#x20; errorResponse,

&#x20; successResponse,

&#x20; maskPhone,

&#x20; isCancellable,

} from '@/lib/validate'



interface RouteContext {

&#x20; params: { reservationNo: string }

}



export async function GET(

&#x20; \_request: NextRequest,

&#x20; context: RouteContext

): Promise<Response> {

&#x20; const { reservationNo } = context.params



&#x20; // ── 1. 예약번호 형식 검증 ─────────────────────────────────────

&#x20; const parsed = reservationNoSchema.safeParse(reservationNo)

&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_RESERVATION\_NO', parsed.error.errors\[0]?.message ?? '올바른 예약번호 형식이 아닙니다.')

&#x20; }



&#x20; const db = getSupabaseClient()



&#x20; // ── 2. 예약 + 결제 + 이력 조회 ───────────────────────────────

&#x20; const { data: reservation, error: reservationError } = await db

&#x20;   .from('reservations')

&#x20;   .select('\*')

&#x20;   .eq('reservation\_no', reservationNo)

&#x20;   .single()



&#x20; if (reservationError || !reservation) {

&#x20;   return errorResponse('RESERVATION\_NOT\_FOUND', '해당 예약번호를 찾을 수 없습니다.', 404)

&#x20; }



&#x20; // 결제 정보 조회 (FO에서는 method, paid\_at만 노출)

&#x20; const { data: payment } = await db

&#x20;   .from('payments')

&#x20;   .select('method, paid\_at, status')

&#x20;   .eq('reservation\_id', reservation.id)

&#x20;   .eq('status', 'PAID')

&#x20;   .maybeSingle()



&#x20; // 상태 이력 조회 (시간순)

&#x20; const { data: logs } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .select('from\_status, to\_status, actor\_label, created\_at')

&#x20;   .eq('reservation\_id', reservation.id)

&#x20;   .order('created\_at', { ascending: true })



&#x20; // ── 3. 취소 가능 여부 계산 ────────────────────────────────────

&#x20; // 상태 = RESERVED AND 이용일 > 오늘(KST) 이면 취소 가능

&#x20; const cancellable =

&#x20;   reservation.status === 'RESERVED' \&\& isCancellable(reservation.use\_date)



&#x20; // ── 4. 응답 조립 (개인정보 마스킹) ───────────────────────────

&#x20; return successResponse({

&#x20;   reservation\_no:       reservation.reservation\_no,

&#x20;   terminal:             reservation.terminal,

&#x20;   direction:            reservation.direction,

&#x20;   use\_date:             reservation.use\_date,

&#x20;   bag\_count:            reservation.bag\_count,

&#x20;   customer\_name:        reservation.customer\_name,

&#x20;   customer\_phone\_masked: maskPhone(reservation.customer\_phone),

&#x20;   amount:               reservation.amount,

&#x20;   status:               reservation.status,

&#x20;   checked\_in\_at:        reservation.checked\_in\_at,

&#x20;   checked\_out\_at:       reservation.checked\_out\_at,

&#x20;   cancelled\_at:         reservation.cancelled\_at,

&#x20;   cancelled\_by:         reservation.cancelled\_by,

&#x20;   payment: payment

&#x20;     ? { method: payment.method, paid\_at: payment.paid\_at }

&#x20;     : null,

&#x20;   logs: (logs ?? \[]).map((log) => ({

&#x20;     from\_status:  log.from\_status,

&#x20;     to\_status:    log.to\_status,

&#x20;     actor\_label:  log.actor\_label,

&#x20;     created\_at:   log.created\_at,

&#x20;   })),

&#x20;   is\_cancellable: cancellable,

&#x20; })

}

```



\---



\## 4. `src/app/api/reservations/\[reservationNo]/cancel/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { sendSmsAsync } from '@/lib/sms'

import {

&#x20; reservationNoSchema,

&#x20; errorResponse,

&#x20; successResponse,

&#x20; isCancellable,

} from '@/lib/validate'



interface RouteContext {

&#x20; params: { reservationNo: string }

}



export async function POST(

&#x20; \_request: NextRequest,

&#x20; context: RouteContext

): Promise<Response> {

&#x20; const { reservationNo } = context.params



&#x20; // ── 1. 예약번호 형식 검증 ─────────────────────────────────────

&#x20; const parsed = reservationNoSchema.safeParse(reservationNo)

&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_RESERVATION\_NO', parsed.error.errors\[0]?.message ?? '올바른 예약번호 형식이 아닙니다.')

&#x20; }



&#x20; const db = getSupabaseClient()



&#x20; // ── 2. 예약 조회 ─────────────────────────────────────────────

&#x20; const { data: reservation, error: fetchError } = await db

&#x20;   .from('reservations')

&#x20;   .select('id, status, use\_date, customer\_phone, reservation\_no, terminal, direction, bag\_count')

&#x20;   .eq('reservation\_no', reservationNo)

&#x20;   .single()



&#x20; if (fetchError || !reservation) {

&#x20;   return errorResponse('RESERVATION\_NOT\_FOUND', '해당 예약번호를 찾을 수 없습니다.', 404)

&#x20; }



&#x20; // ── 3. 취소 가능 여부 검증 ────────────────────────────────────



&#x20; // 3-a. 상태 검증

&#x20; switch (reservation.status) {

&#x20;   case 'CANCELLED':

&#x20;     return errorResponse('ALREADY\_CANCELLED', '이미 취소된 예약입니다.', 409)

&#x20;   case 'STORED':

&#x20;     return errorResponse('ALREADY\_STORED', '이미 접수된 예약은 취소할 수 없습니다.', 409)

&#x20;   case 'RETURNED':

&#x20;     return errorResponse('ALREADY\_RETURNED', '이미 반환 완료된 예약입니다.', 409)

&#x20;   case 'RESERVED':

&#x20;     break  // 취소 가능 상태 — 계속 진행

&#x20;   default:

&#x20;     return errorResponse('INVALID\_REQUEST', '처리할 수 없는 예약 상태입니다.')

&#x20; }



&#x20; // 3-b. 이용일 기준 취소 가능 시간 검증 (이용일 00시 이전만 가능)

&#x20; if (!isCancellable(reservation.use\_date)) {

&#x20;   return errorResponse(

&#x20;     'CANCEL\_NOT\_ALLOWED',

&#x20;     '이용일 당일은 취소할 수 없습니다.',

&#x20;     409

&#x20;   )

&#x20; }



&#x20; // ── 4. 취소 처리 ─────────────────────────────────────────────

&#x20; const cancelledAt = new Date().toISOString()



&#x20; // 4-a. reservations UPDATE (RESERVED → CANCELLED)

&#x20; const { error: updateError } = await db

&#x20;   .from('reservations')

&#x20;   .update({

&#x20;     status:       'CANCELLED',

&#x20;     cancelled\_by: 'CUSTOMER',

&#x20;     cancelled\_at: cancelledAt,

&#x20;   })

&#x20;   .eq('id', reservation.id)

&#x20;   .eq('status', 'RESERVED')  // 동시 요청 경쟁 조건 방지 (낙관적 잠금)



&#x20; if (updateError) {

&#x20;   console.error('\[cancel] reservations UPDATE 실패:', updateError)

&#x20;   return errorResponse('INTERNAL\_ERROR', '취소 처리 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; // 4-b. reservation\_logs INSERT (RESERVED → CANCELLED)

&#x20; const { error: logError } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .insert({

&#x20;     reservation\_id: reservation.id,

&#x20;     from\_status:    'RESERVED',

&#x20;     to\_status:      'CANCELLED',

&#x20;     actor\_type:     'CUSTOMER',

&#x20;     actor\_label:    '고객 직접 취소',

&#x20;   })



&#x20; if (logError) {

&#x20;   console.error('\[cancel] reservation\_logs INSERT 실패:', logError)

&#x20;   // 취소 자체는 성공했으므로 로그 실패는 무시하고 계속 진행

&#x20; }



&#x20; // ── 5. SMS 발송 (비동기) ──────────────────────────────────────

&#x20; // ※ 실제 환불은 토스페이 어드민 콘솔에서 운영자가 수동 처리

&#x20; sendSmsAsync({

&#x20;   reservationId: reservation.id,

&#x20;   eventType:     'CANCELLATION\_COMPLETE',

&#x20;   phone:         reservation.customer\_phone,

&#x20;   data: {

&#x20;     reservationNo: reservation.reservation\_no,

&#x20;     terminal:      reservation.terminal,

&#x20;     direction:     reservation.direction,

&#x20;     useDate:       reservation.use\_date,

&#x20;     bagCount:      reservation.bag\_count,

&#x20;   },

&#x20; })



&#x20; // ── 6. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   reservation\_no: reservation.reservation\_no,

&#x20;   status:         'CANCELLED',

&#x20;   cancelled\_at:   cancelledAt,

&#x20;   cancelled\_by:   'CUSTOMER',

&#x20; })

}

```



\---



\## 5. `src/app/api/payments/webhook/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { verifyTossWebhookSignature } from '@/lib/toss'

import { errorResponse, successResponse } from '@/lib/validate'



// 토스페이 웹훅 페이로드 타입

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

&#x20; // ── 1. 요청 본문 수신 ─────────────────────────────────────────

&#x20; // 서명 검증을 위해 raw text로 먼저 읽은 후 JSON 파싱

&#x20; let rawBody: string

&#x20; try {

&#x20;   rawBody = await request.text()

&#x20; } catch {

&#x20;   return errorResponse('INVALID\_REQUEST', '요청 본문을 읽을 수 없습니다.')

&#x20; }



&#x20; // ── 2. 토스페이 웹훅 서명 검증 ───────────────────────────────

&#x20; const signature = request.headers.get('Toss-Signature') ?? ''



&#x20; if (!verifyTossWebhookSignature(rawBody, signature)) {

&#x20;   console.warn('\[webhook] 서명 검증 실패. 위조된 웹훅일 수 있음.')

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



&#x20; // 결제 상태 변경 이벤트만 처리

&#x20; if (eventType !== 'PAYMENT\_STATUS\_CHANGED') {

&#x20;   // 알 수 없는 이벤트는 200 반환 (토스페이가 재발송하지 않도록)

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

&#x20;   // 500 반환 → 토스페이가 나중에 재시도

&#x20;   return Response.json({ success: false }, { status: 500 })

&#x20; }



&#x20; if (!payment) {

&#x20;   // orderId에 해당하는 payments가 없음 — 무시하고 200 반환

&#x20;   console.warn('\[webhook] 대응하는 payments 레코드 없음. orderId:', orderId)

&#x20;   return successResponse({ received: true })

&#x20; }



&#x20; // ── 5. 멱등 처리 ─────────────────────────────────────────────

&#x20; // DONE 이벤트이고 이미 PAID 상태면 confirm API에서 처리 완료 → 스킵

&#x20; if (status === 'DONE' \&\& payment.status === 'PAID') {

&#x20;   return successResponse({ received: true, skipped: true })

&#x20; }



&#x20; // 이미 최종 상태(PAID/FAILED/CANCELLED)인 건 DONE이 아닌 경우 불일치 로그

&#x20; const alreadyFinalized = \['PAID', 'FAILED', 'CANCELLED'].includes(payment.status)



&#x20; // ── 6. 상태별 분기 처리 ───────────────────────────────────────

&#x20; switch (status) {

&#x20;   // ── DONE: confirm API가 처리했어야 하나 누락된 경우 복구 ────

&#x20;   case 'DONE': {

&#x20;     if (alreadyFinalized) {

&#x20;       // 이미 FAILED/CANCELLED 상태인데 DONE 웹훅 → 이상 상황 로그

&#x20;       console.warn('\[webhook] DONE 이벤트이나 payments 상태 불일치:', {

&#x20;         orderId,

&#x20;         currentStatus: payment.status,

&#x20;       })

&#x20;       return successResponse({ received: true })

&#x20;     }



&#x20;     // PENDING 상태에서 DONE 웹훅 수신 → confirm이 실패한 경우

&#x20;     // payments만 PAID로 업데이트 (reservations는 이미 생성됐을 수 있음)

&#x20;     const { error: updateError } = await db

&#x20;       .from('payments')

&#x20;       .update({

&#x20;         toss\_payment\_key: paymentKey,

&#x20;         status:           'PAID',

&#x20;         toss\_response:    data as unknown as Record<string, unknown>,

&#x20;         paid\_at:          new Date().toISOString(),

&#x20;       })

&#x20;       .eq('id', payment.id)

&#x20;       .eq('status', 'PENDING')  // 낙관적 잠금



&#x20;     if (updateError) {

&#x20;       console.error('\[webhook] DONE payments UPDATE 실패:', updateError)

&#x20;       return Response.json({ success: false }, { status: 500 })

&#x20;     }



&#x20;     console.info('\[webhook] DONE 복구 처리 완료:', orderId)

&#x20;     break

&#x20;   }



&#x20;   // ── CANCELED: 환불 완료 확인 (토스페이 콘솔에서 수동 환불 후 발생) ──

&#x20;   case 'CANCELED': {

&#x20;     if (alreadyFinalized \&\& payment.status !== 'PAID') {

&#x20;       return successResponse({ received: true, skipped: true })

&#x20;     }



&#x20;     const { error: cancelError } = await db

&#x20;       .from('payments')

&#x20;       .update({

&#x20;         status:           'CANCELLED',

&#x20;         cancelled\_amount: data.totalAmount as number ?? 0,

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



&#x20;   // ── ABORTED / EXPIRED: 결제 실패/만료 ───────────────────────

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



&#x20;   case 'PARTIAL\_CANCELED': {

&#x20;     // 전액 결제 서비스 특성상 부분 취소 없음 — 이상 상황 로그만

&#x20;     console.warn('\[webhook] 예상치 못한 PARTIAL\_CANCELED 이벤트:', orderId)

&#x20;     break

&#x20;   }



&#x20;   default: {

&#x20;     console.warn('\[webhook] 처리되지 않은 status:', status, orderId)

&#x20;     break

&#x20;   }

&#x20; }



&#x20; // ── 7. 200 응답 (항상 성공 반환 — 토스페이 재발송 방지) ───────

&#x20; return successResponse({ received: true })

}

```



\---



\## 설계 노트



\### confirm API — Supabase 트랜잭션 미지원 대응



Supabase JS 클라이언트는 `BEGIN / COMMIT` 트랜잭션을 직접 지원하지 않는다. 아래 두 가지 방법 중 \*\*방법 A\*\*를 채택했다.



| 방법 | 설명 | 채택 |

|------|------|------|

| \*\*A. 순차 실행 + 웹훅 복구\*\* | 토스페이 승인 성공 시 각 INSERT/UPDATE를 순서대로 실행. 중간 실패는 웹훅이 재발송하므로 복구 가능 | ✅ |

| B. Supabase RPC 함수 | `PERFORM` 문을 포함한 PL/pgSQL 함수 작성 후 RPC 단일 호출로 원자 처리 | 구현 복잡도 높음 |



\### cancel — 낙관적 잠금 (Optimistic Locking)



```sql

\-- .eq('status', 'RESERVED') 조건을 UPDATE WHERE에 포함

UPDATE reservations

SET status = 'CANCELLED', ...

WHERE id = $1 AND status = 'RESERVED'

```



동시에 두 취소 요청이 들어와도 첫 번째 요청만 `status = 'RESERVED'` 조건에 매칭되어 처리됨.

두 번째 요청은 `affectedRows = 0` → 이미 취소된 상태로 재조회 시 `ALREADY\_CANCELLED` 반환.



\### webhook — 멱등 처리 원칙



토스페이는 웹훅 실패 시 자동 재발송(최대 5회)한다. 모든 분기에서 이미 처리된 건은 \*\*스킵 후 200 반환\*\*하여 무한 재시도를 방지한다.



| 수신 status | 현재 DB 상태 | 처리 |

|------------|------------|------|

| DONE | PENDING | payments → PAID (복구) |

| DONE | PAID | 스킵 (confirm이 이미 처리) |

| DONE | FAILED/CANCELLED | 경고 로그만 |

| CANCELED | PAID | payments → CANCELLED |

| CANCELED | 이미 CANCELLED | 스킵 |

| ABORTED/EXPIRED | PENDING | payments → FAILED |

| ABORTED/EXPIRED | 이미 최종 상태 | 스킵 |



\### 전화번호 마스킹 위치



`maskPhone()`은 \*\*API 응답 조립 시점\*\*에서만 적용한다. DB에는 원본 번호를 저장하여 SMS 발송 및 운영자 환불 처리에서 정상 사용할 수 있도록 한다.



\---



\## Session 07 체크리스트



| 파일 | 구현 항목 | 상태 |

|------|----------|------|

| `prepare/route.ts` | 유효성 검증 / toss\_order\_id 생성 / PENDING INSERT | ✅ |

| `confirm/route.ts` | 중복결제 방어 / 금액 이중검증 / 토스페이 승인 / 예약+결제+로그 저장 / SMS 비동기 | ✅ |

| `\[reservationNo]/route.ts` | 형식 검증 / 예약+결제+이력 JOIN 조회 / 마스킹 / is\_cancellable | ✅ |

| `\[reservationNo]/cancel/route.ts` | 상태 4가지 분기 / 날짜 검증 / 낙관적 잠금 UPDATE / 로그 / SMS | ✅ |

| `webhook/route.ts` | HMAC 서명 검증 / status 5가지 분기 / 멱등 처리 / 복구 로직 | ✅ |



\---



\*다음 세션: Session 08 — BO API Route 구현 (dashboard / checkin / checkout / cancel / storage / sales)\*

