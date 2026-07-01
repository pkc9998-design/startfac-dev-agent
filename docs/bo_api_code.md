\# BagDrop BO API Route 구현 코드



> 작성: 개발 에이전트 가온

> 세션: Session 08

> 최종 업데이트: 2026-07-01

> 대상: BO API Route 8개 (Next.js 14 App Router)

> 상태: ✅ Session 08 확정



\---



\## 파일 목록



| 파일 경로 | 메서드 | 역할 |

|-----------|--------|------|

| `src/app/api/admin/dashboard/route.ts` | GET | 대시보드 현황 |

| `src/app/api/admin/reservations/route.ts` | GET | 예약 목록 |

| `src/app/api/admin/reservations/\[reservationNo]/route.ts` | GET | 예약 상세 |

| `src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts` | PATCH | 짐 접수 |

| `src/app/api/admin/reservations/\[reservationNo]/checkout/route.ts` | PATCH | 짐 반환 |

| `src/app/api/admin/reservations/\[reservationNo]/cancel/route.ts` | PATCH | 강제 취소 |

| `src/app/api/admin/storage/route.ts` | GET | 보관 현황 |

| `src/app/api/admin/sales/route.ts` | GET | 매출 집계 |



\---



\## 공통 헬퍼 — `src/lib/admin.ts`



> BO API 전체에서 반복 사용하는 인증 확인 + 예약 조회 패턴을 분리.

> 각 route.ts 코드량을 줄이고 일관성을 유지한다.



```typescript

// src/lib/admin.ts

import { getServerSession } from 'next-auth'

import { authOptions } from '@/lib/auth'

import { getSupabaseClient, type Reservation, type ReservationStatus } from '@/lib/supabase'

import { reservationNoSchema, errorResponse } from '@/lib/validate'



// ──────────────────────────────────────────────────────────────

// 세션 인증

// ──────────────────────────────────────────────────────────────



export interface AdminSession {

&#x20; id:          string

&#x20; username:    string

&#x20; displayName: string

}



/\*\*

&#x20;\* BO API 인증 확인 헬퍼

&#x20;\* 세션 없으면 { session: null, response: 401 Response } 반환

&#x20;\*/

export async function getAdminSession(): Promise<

&#x20; | { session: AdminSession; authError: null }

&#x20; | { session: null; authError: Response }

> {

&#x20; const session = await getServerSession(authOptions)

&#x20; if (!session?.user?.id) {

&#x20;   return {

&#x20;     session:   null,

&#x20;     authError: errorResponse('UNAUTHORIZED', '로그인이 필요합니다.', 401),

&#x20;   }

&#x20; }

&#x20; return { session: session.user as AdminSession, authError: null }

}



// ──────────────────────────────────────────────────────────────

// 예약 조회 (BO 공통)

// ──────────────────────────────────────────────────────────────



export interface FetchReservationResult {

&#x20; reservation: Reservation | null

&#x20; fetchError:  Response | null

}



/\*\*

&#x20;\* 예약번호로 예약 단건 조회 + 형식 검증

&#x20;\* 없으면 404 Response 반환

&#x20;\*/

export async function fetchReservation(

&#x20; reservationNo: string

): Promise<FetchReservationResult> {

&#x20; // 형식 검증

&#x20; const parsed = reservationNoSchema.safeParse(reservationNo)

&#x20; if (!parsed.success) {

&#x20;   return {

&#x20;     reservation: null,

&#x20;     fetchError:  errorResponse('INVALID\_RESERVATION\_NO', parsed.error.errors\[0]?.message ?? '올바른 예약번호 형식이 아닙니다.'),

&#x20;   }

&#x20; }



&#x20; const db = getSupabaseClient()

&#x20; const { data, error } = await db

&#x20;   .from('reservations')

&#x20;   .select('\*')

&#x20;   .eq('reservation\_no', reservationNo)

&#x20;   .single()



&#x20; if (error || !data) {

&#x20;   return {

&#x20;     reservation: null,

&#x20;     fetchError:  errorResponse('RESERVATION\_NOT\_FOUND', '해당 예약번호를 찾을 수 없습니다.', 404),

&#x20;   }

&#x20; }



&#x20; return { reservation: data, fetchError: null }

}



// ──────────────────────────────────────────────────────────────

// 경과 시간 계산

// ──────────────────────────────────────────────────────────────



/\*\* 접수 시각 기준 경과 분 계산 \*/

export function calcElapsedMinutes(checkedInAt: string): number {

&#x20; return Math.floor((Date.now() - new Date(checkedInAt).getTime()) / (1000 \* 60))

}



/\*\* 경과 분 → 경고 레벨 \*/

export function calcWarningLevel(minutes: number): 'ORANGE' | 'RED' | null {

&#x20; if (minutes >= 240) return 'RED'

&#x20; if (minutes >= 120) return 'ORANGE'

&#x20; return null

}



// ──────────────────────────────────────────────────────────────

// 상태 전환 공통 검증

// ──────────────────────────────────────────────────────────────



interface StatusGuardResult {

&#x20; allowed:       boolean

&#x20; errorResponse: Response | null

}



/\*\*

&#x20;\* 상태 전환 허용 여부 검증

&#x20;\* allowed = true 이면 전환 진행, false 이면 errorResponse를 반환

&#x20;\*/

export function guardStatus(

&#x20; currentStatus: ReservationStatus,

&#x20; requiredStatus: ReservationStatus

): StatusGuardResult {

&#x20; if (currentStatus === requiredStatus) {

&#x20;   return { allowed: true, errorResponse: null }

&#x20; }



&#x20; const statusErrorMap: Record<ReservationStatus, { code: string; message: string }> = {

&#x20;   CANCELLED: { code: 'ALREADY\_CANCELLED', message: '이미 취소된 예약입니다.' },

&#x20;   STORED:    { code: 'ALREADY\_STORED',    message: '이미 접수 처리된 예약입니다.' },

&#x20;   RETURNED:  { code: 'ALREADY\_RETURNED',  message: '이미 반환 완료된 예약입니다.' },

&#x20;   RESERVED:  { code: 'NOT\_STORED',        message: '아직 접수되지 않은 예약입니다. 먼저 짐 접수를 진행해 주세요.' },

&#x20; }



&#x20; const err = statusErrorMap\[currentStatus]

&#x20; return {

&#x20;   allowed:       false,

&#x20;   errorResponse: errorResponse(err.code, err.message, 409),

&#x20; }

}

```



\---



\## 1. `src/app/api/admin/dashboard/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { z } from 'zod'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession } from '@/lib/admin'

import { errorResponse, successResponse, getTodayKst } from '@/lib/validate'



const querySchema = z.object({

&#x20; terminal: z.enum(\['T1', 'T2']).optional(),

})



export async function GET(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. Query Params 파싱 ─────────────────────────────────────

&#x20; const { searchParams } = request.nextUrl

&#x20; const parsed = querySchema.safeParse({

&#x20;   terminal: searchParams.get('terminal') ?? undefined,

&#x20; })

&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_REQUEST', '잘못된 터미널 값입니다.')

&#x20; }

&#x20; const { terminal } = parsed.data

&#x20; const today        = getTodayKst()   // KST 기준 오늘 날짜 YYYY-MM-DD

&#x20; const db           = getSupabaseClient()



&#x20; // ── 3. 병렬 쿼리 실행 ────────────────────────────────────────

&#x20; // Supabase는 Promise.all로 병렬 처리 가능



&#x20; // 3-a. 오늘 총 예약 (RESERVED + STORED + RETURNED)

&#x20; const todayTotalQuery = db

&#x20;   .from('reservations')

&#x20;   .select('id', { count: 'exact', head: true })

&#x20;   .eq('use\_date', today)

&#x20;   .in('status', \['RESERVED', 'STORED', 'RETURNED'])



&#x20; // 3-b. 현재 보관 중 (STORED)

&#x20; const storedQuery = db

&#x20;   .from('reservations')

&#x20;   .select('id', { count: 'exact', head: true })

&#x20;   .eq('status', 'STORED')



&#x20; // 3-c. 오늘 반환 완료 (RETURNED)

&#x20; const returnedQuery = db

&#x20;   .from('reservations')

&#x20;   .select('id', { count: 'exact', head: true })

&#x20;   .eq('use\_date', today)

&#x20;   .eq('status', 'RETURNED')



&#x20; // 3-d. 최근 접수 대기 5건 (RESERVED, created\_at 역순)

&#x20; const pendingQuery = db

&#x20;   .from('reservations')

&#x20;   .select('reservation\_no, terminal, direction, customer\_name, bag\_count, created\_at')

&#x20;   .eq('status', 'RESERVED')

&#x20;   .order('created\_at', { ascending: false })

&#x20;   .limit(5)



&#x20; // 터미널 필터 적용

&#x20; if (terminal) {

&#x20;   todayTotalQuery.eq('terminal', terminal)

&#x20;   storedQuery.eq('terminal', terminal)

&#x20;   returnedQuery.eq('terminal', terminal)

&#x20;   pendingQuery.eq('terminal', terminal)

&#x20; }



&#x20; const \[

&#x20;   { count: todayTotal,    error: e1 },

&#x20;   { count: currentStored, error: e2 },

&#x20;   { count: todayReturned, error: e3 },

&#x20;   { data:  recentPending, error: e4 },

&#x20; ] = await Promise.all(\[

&#x20;   todayTotalQuery,

&#x20;   storedQuery,

&#x20;   returnedQuery,

&#x20;   pendingQuery,

&#x20; ])



&#x20; if (e1 || e2 || e3 || e4) {

&#x20;   console.error('\[dashboard] 쿼리 실패:', { e1, e2, e3, e4 })

&#x20;   return errorResponse('INTERNAL\_ERROR', '데이터 조회 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; const now = new Date()



&#x20; // ── 4. elapsed\_minutes 계산 ───────────────────────────────────

&#x20; const pendingWithElapsed = (recentPending ?? \[]).map((r) => ({

&#x20;   ...r,

&#x20;   elapsed\_minutes: Math.floor(

&#x20;     (now.getTime() - new Date(r.created\_at).getTime()) / (1000 \* 60)

&#x20;   ),

&#x20; }))



&#x20; // ── 5. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   today\_total:    todayTotal    ?? 0,

&#x20;   current\_stored: currentStored ?? 0,

&#x20;   today\_returned: todayReturned ?? 0,

&#x20;   recent\_pending: pendingWithElapsed,

&#x20;   as\_of:          now.toISOString(),

&#x20; })

}

```



\---



\## 2. `src/app/api/admin/reservations/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient, type ReservationStatus } from '@/lib/supabase'

import { getAdminSession } from '@/lib/admin'

import {

&#x20; adminReservationListSchema,

&#x20; errorResponse,

&#x20; successResponse,

&#x20; getZodError,

&#x20; getTodayKst,

} from '@/lib/validate'



// 허용 상태값 목록

const VALID\_STATUSES: ReservationStatus\[] = \['RESERVED', 'STORED', 'RETURNED', 'CANCELLED']



export async function GET(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. Query Params 파싱 + 유효성 검증 ───────────────────────

&#x20; const { searchParams } = request.nextUrl

&#x20; const parsed = adminReservationListSchema.safeParse({

&#x20;   date:     searchParams.get('date')     ?? undefined,

&#x20;   terminal: searchParams.get('terminal') ?? undefined,

&#x20;   status:   searchParams.get('status')   ?? undefined,

&#x20;   page:     searchParams.get('page')     ?? '1',

&#x20;   limit:    searchParams.get('limit')    ?? '20',

&#x20; })



&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_REQUEST', getZodError(parsed.error))

&#x20; }



&#x20; const { date, terminal, status: statusParam, page, limit } = parsed.data

&#x20; const targetDate = date ?? getTodayKst()

&#x20; const offset     = (page - 1) \* limit



&#x20; // status 파라미터 → 배열로 파싱 ("RESERVED,STORED" → \['RESERVED', 'STORED'])

&#x20; let statusFilter: ReservationStatus\[] | null = null

&#x20; if (statusParam) {

&#x20;   const raw = statusParam.split(',').map((s) => s.trim().toUpperCase())

&#x20;   const valid = raw.filter((s): s is ReservationStatus =>

&#x20;     VALID\_STATUSES.includes(s as ReservationStatus)

&#x20;   )

&#x20;   if (valid.length > 0) statusFilter = valid

&#x20; }



&#x20; const db = getSupabaseClient()



&#x20; // ── 3. 쿼리 빌드 ─────────────────────────────────────────────

&#x20; // 목록 조회

&#x20; let listQuery = db

&#x20;   .from('reservations')

&#x20;   .select(

&#x20;     'reservation\_no, terminal, direction, use\_date, bag\_count, customer\_name, status, created\_at, checked\_in\_at, checked\_out\_at'

&#x20;   )

&#x20;   .eq('use\_date', targetDate)

&#x20;   .order('created\_at', { ascending: false })

&#x20;   .range(offset, offset + limit - 1)



&#x20; // 건수 조회 (페이징 total 계산용)

&#x20; let countQuery = db

&#x20;   .from('reservations')

&#x20;   .select('id', { count: 'exact', head: true })

&#x20;   .eq('use\_date', targetDate)



&#x20; // 동적 필터 적용

&#x20; if (terminal) {

&#x20;   listQuery  = listQuery.eq('terminal', terminal)

&#x20;   countQuery = countQuery.eq('terminal', terminal)

&#x20; }

&#x20; if (statusFilter \&\& statusFilter.length > 0) {

&#x20;   listQuery  = listQuery.in('status', statusFilter)

&#x20;   countQuery = countQuery.in('status', statusFilter)

&#x20; }



&#x20; // ── 4. 병렬 실행 ─────────────────────────────────────────────

&#x20; const \[

&#x20;   { data: items, error: listError },

&#x20;   { count: total, error: countError },

&#x20; ] = await Promise.all(\[listQuery, countQuery])



&#x20; if (listError || countError) {

&#x20;   console.error('\[reservations] 쿼리 실패:', { listError, countError })

&#x20;   return errorResponse('INTERNAL\_ERROR', '데이터 조회 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; // ── 5. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   items: items ?? \[],

&#x20;   total: total ?? 0,

&#x20;   page,

&#x20;   limit,

&#x20; })

}

```



\---



\## 3. `src/app/api/admin/reservations/\[reservationNo]/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession, fetchReservation } from '@/lib/admin'

import { successResponse, maskPhone } from '@/lib/validate'



interface RouteContext {

&#x20; params: { reservationNo: string }

}



export async function GET(

&#x20; \_request: NextRequest,

&#x20; context: RouteContext

): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. 예약 조회 ─────────────────────────────────────────────

&#x20; const { reservation, fetchError } = await fetchReservation(context.params.reservationNo)

&#x20; if (fetchError) return fetchError



&#x20; const db = getSupabaseClient()



&#x20; // ── 3. 결제 정보 조회 (운영자용: toss\_payment\_key 포함) ───────

&#x20; const { data: payment } = await db

&#x20;   .from('payments')

&#x20;   .select('method, toss\_payment\_key, paid\_at, status, cancelled\_amount, cancelled\_at')

&#x20;   .eq('reservation\_id', reservation!.id)

&#x20;   .in('status', \['PAID', 'CANCELLED'])  // PENDING/FAILED 제외

&#x20;   .order('created\_at', { ascending: false })

&#x20;   .limit(1)

&#x20;   .maybeSingle()



&#x20; // ── 4. 상태 이력 조회 ─────────────────────────────────────────

&#x20; const { data: logs } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .select('from\_status, to\_status, actor\_type, actor\_label, created\_at')

&#x20;   .eq('reservation\_id', reservation!.id)

&#x20;   .order('created\_at', { ascending: true })



&#x20; // ── 5. 응답 조립 ─────────────────────────────────────────────

&#x20; // FO 조회와 달리 toss\_payment\_key 노출 (운영자 환불 참조용)

&#x20; return successResponse({

&#x20;   reservation\_no:        reservation!.reservation\_no,

&#x20;   terminal:              reservation!.terminal,

&#x20;   direction:             reservation!.direction,

&#x20;   use\_date:              reservation!.use\_date,

&#x20;   bag\_count:             reservation!.bag\_count,

&#x20;   customer\_name:         reservation!.customer\_name,

&#x20;   customer\_phone\_masked: maskPhone(reservation!.customer\_phone),

&#x20;   amount:                reservation!.amount,

&#x20;   status:                reservation!.status,

&#x20;   checked\_in\_at:         reservation!.checked\_in\_at,

&#x20;   checked\_out\_at:        reservation!.checked\_out\_at,

&#x20;   cancelled\_at:          reservation!.cancelled\_at,

&#x20;   cancelled\_by:          reservation!.cancelled\_by,

&#x20;   payment: payment

&#x20;     ? {

&#x20;         method:           payment.method,

&#x20;         toss\_payment\_key: payment.toss\_payment\_key,

&#x20;         paid\_at:          payment.paid\_at,

&#x20;         status:           payment.status,

&#x20;         cancelled\_amount: payment.cancelled\_amount,

&#x20;         cancelled\_at:     payment.cancelled\_at,

&#x20;       }

&#x20;     : null,

&#x20;   logs: (logs ?? \[]).map((log) => ({

&#x20;     from\_status:  log.from\_status,

&#x20;     to\_status:    log.to\_status,

&#x20;     actor\_type:   log.actor\_type,

&#x20;     actor\_label:  log.actor\_label,

&#x20;     created\_at:   log.created\_at,

&#x20;   })),

&#x20; })

}

```



\---



\## 4. `src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession, fetchReservation, guardStatus } from '@/lib/admin'

import { sendSmsAsync } from '@/lib/sms'

import { successResponse, errorResponse } from '@/lib/validate'



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



&#x20; // ── 3. 상태 검증 (RESERVED 여야 접수 가능) ────────────────────

&#x20; const { allowed, errorResponse: statusError } = guardStatus(reservation!.status, 'RESERVED')

&#x20; if (!allowed) return statusError!



&#x20; const db           = getSupabaseClient()

&#x20; const checkedInAt  = new Date().toISOString()



&#x20; // ── 4. reservations UPDATE (RESERVED → STORED) ────────────────

&#x20; // eq('status', 'RESERVED') — 낙관적 잠금: 동시 요청 중 첫 번째만 성공

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



&#x20; // 낙관적 잠금 실패 — 동시 요청에 의해 이미 상태가 변경된 경우

&#x20; if (!updated) {

&#x20;   return errorResponse('ALREADY\_STORED', '이미 접수 처리된 예약입니다.', 409)

&#x20; }



&#x20; // ── 5. reservation\_logs INSERT (RESERVED → STORED) ────────────

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

&#x20;   // 접수 자체는 성공했으므로 로그 실패는 무시하고 응답

&#x20; }



&#x20; // ── 6. SMS 발송 (비동기) ──────────────────────────────────────

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



&#x20; // ── 7. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   reservation\_no: reservation!.reservation\_no,

&#x20;   status:         'STORED',

&#x20;   checked\_in\_at:  checkedInAt,

&#x20; })

}

```



\---



\## 5. `src/app/api/admin/reservations/\[reservationNo]/checkout/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession, fetchReservation, guardStatus } from '@/lib/admin'

import { sendSmsAsync } from '@/lib/sms'

import { successResponse, errorResponse } from '@/lib/validate'



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



&#x20; // ── 3. 상태 검증 (STORED 여야 반환 가능) ─────────────────────

&#x20; const { allowed, errorResponse: statusError } = guardStatus(reservation!.status, 'STORED')

&#x20; if (!allowed) return statusError!



&#x20; const db            = getSupabaseClient()

&#x20; const checkedOutAt  = new Date().toISOString()



&#x20; // ── 4. reservations UPDATE (STORED → RETURNED) ────────────────

&#x20; const { data: updated, error: updateError } = await db

&#x20;   .from('reservations')

&#x20;   .update({

&#x20;     status:         'RETURNED',

&#x20;     checked\_out\_at: checkedOutAt,

&#x20;   })

&#x20;   .eq('id', reservation!.id)

&#x20;   .eq('status', 'STORED')   // 낙관적 잠금

&#x20;   .select('id')

&#x20;   .maybeSingle()



&#x20; if (updateError) {

&#x20;   console.error('\[checkout] UPDATE 실패:', updateError)

&#x20;   return errorResponse('INTERNAL\_ERROR', '반환 처리 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; if (!updated) {

&#x20;   return errorResponse('ALREADY\_RETURNED', '이미 반환 완료된 예약입니다.', 409)

&#x20; }



&#x20; // ── 5. reservation\_logs INSERT (STORED → RETURNED) ────────────

&#x20; const { error: logError } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .insert({

&#x20;     reservation\_id: reservation!.id,

&#x20;     from\_status:    'STORED',

&#x20;     to\_status:      'RETURNED',

&#x20;     actor\_type:     'ADMIN',

&#x20;     actor\_id:       session!.id,

&#x20;     actor\_label:    '짐 반환',

&#x20;   })



&#x20; if (logError) {

&#x20;   console.error('\[checkout] LOG INSERT 실패:', logError)

&#x20; }



&#x20; // ── 6. SMS 발송 (비동기) ──────────────────────────────────────

&#x20; sendSmsAsync({

&#x20;   reservationId: reservation!.id,

&#x20;   eventType:     'CHECKOUT\_COMPLETE',

&#x20;   phone:         reservation!.customer\_phone,

&#x20;   data: {

&#x20;     reservationNo: reservation!.reservation\_no,

&#x20;     terminal:      reservation!.terminal,

&#x20;     direction:     reservation!.direction,

&#x20;     useDate:       reservation!.use\_date,

&#x20;     bagCount:      reservation!.bag\_count,

&#x20;   },

&#x20; })



&#x20; // ── 7. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   reservation\_no:  reservation!.reservation\_no,

&#x20;   status:          'RETURNED',

&#x20;   checked\_out\_at:  checkedOutAt,

&#x20; })

}

```



\---



\## 6. `src/app/api/admin/reservations/\[reservationNo]/cancel/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { z } from 'zod'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession, fetchReservation } from '@/lib/admin'

import { sendSmsAsync } from '@/lib/sms'

import { successResponse, errorResponse, getZodError } from '@/lib/validate'



interface RouteContext {

&#x20; params: { reservationNo: string }

}



const bodySchema = z.object({

&#x20; cancel\_reason: z.string().max(200).optional(),

})



export async function PATCH(

&#x20; request: NextRequest,

&#x20; context: RouteContext

): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { session, authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. Body 파싱 (cancel\_reason 선택) ────────────────────────

&#x20; let cancelReason: string | undefined

&#x20; try {

&#x20;   const raw   = await request.json().catch(() => ({}))

&#x20;   const parsed = bodySchema.safeParse(raw)

&#x20;   if (!parsed.success) {

&#x20;     return errorResponse('INVALID\_REQUEST', getZodError(parsed.error))

&#x20;   }

&#x20;   cancelReason = parsed.data.cancel\_reason

&#x20; } catch {

&#x20;   // Body 없어도 허용

&#x20; }



&#x20; // ── 3. 예약 조회 ─────────────────────────────────────────────

&#x20; const { reservation, fetchError } = await fetchReservation(context.params.reservationNo)

&#x20; if (fetchError) return fetchError



&#x20; // ── 4. 상태 검증 (RESERVED만 강제 취소 가능) ──────────────────

&#x20; // STORED/RETURNED 상태는 강제 취소 불가 (물리적으로 짐이 이미 부스에 있음)

&#x20; switch (reservation!.status) {

&#x20;   case 'CANCELLED':

&#x20;     return errorResponse('ALREADY\_CANCELLED', '이미 취소된 예약입니다.', 409)

&#x20;   case 'STORED':

&#x20;     return errorResponse('ALREADY\_STORED', '접수된 예약은 강제 취소할 수 없습니다.', 409)

&#x20;   case 'RETURNED':

&#x20;     return errorResponse('ALREADY\_RETURNED', '이미 반환 완료된 예약입니다.', 409)

&#x20;   case 'RESERVED':

&#x20;     break   // 진행

&#x20; }



&#x20; const db          = getSupabaseClient()

&#x20; const cancelledAt = new Date().toISOString()



&#x20; // ── 5. reservations UPDATE (RESERVED → CANCELLED) ─────────────

&#x20; const { data: updated, error: updateError } = await db

&#x20;   .from('reservations')

&#x20;   .update({

&#x20;     status:        'CANCELLED',

&#x20;     cancelled\_by:  'ADMIN',

&#x20;     cancelled\_at:  cancelledAt,

&#x20;     cancel\_reason: cancelReason ?? null,

&#x20;   })

&#x20;   .eq('id', reservation!.id)

&#x20;   .eq('status', 'RESERVED')   // 낙관적 잠금

&#x20;   .select('id')

&#x20;   .maybeSingle()



&#x20; if (updateError) {

&#x20;   console.error('\[admin/cancel] UPDATE 실패:', updateError)

&#x20;   return errorResponse('INTERNAL\_ERROR', '취소 처리 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; if (!updated) {

&#x20;   // 낙관적 잠금 실패 → 상태 재조회 후 적합한 에러 반환

&#x20;   return errorResponse('ALREADY\_STORED', '처리 중 상태가 변경됐습니다. 예약을 다시 확인해 주세요.', 409)

&#x20; }



&#x20; // ── 6. reservation\_logs INSERT (RESERVED → CANCELLED) ─────────

&#x20; const { error: logError } = await db

&#x20;   .from('reservation\_logs')

&#x20;   .insert({

&#x20;     reservation\_id: reservation!.id,

&#x20;     from\_status:    'RESERVED',

&#x20;     to\_status:      'CANCELLED',

&#x20;     actor\_type:     'ADMIN',

&#x20;     actor\_id:       session!.id,

&#x20;     actor\_label:    '운영자 강제 취소',

&#x20;     note:           cancelReason ?? null,

&#x20;   })



&#x20; if (logError) {

&#x20;   console.error('\[admin/cancel] LOG INSERT 실패:', logError)

&#x20; }



&#x20; // ── 7. SMS 발송 (비동기) ──────────────────────────────────────

&#x20; sendSmsAsync({

&#x20;   reservationId: reservation!.id,

&#x20;   eventType:     'CANCELLATION\_COMPLETE',

&#x20;   phone:         reservation!.customer\_phone,

&#x20;   data: {

&#x20;     reservationNo: reservation!.reservation\_no,

&#x20;     terminal:      reservation!.terminal,

&#x20;     direction:     reservation!.direction,

&#x20;     useDate:       reservation!.use\_date,

&#x20;     bagCount:      reservation!.bag\_count,

&#x20;   },

&#x20; })



&#x20; // ── 8. 결제 키 조회 (운영자 환불 참조용) ──────────────────────

&#x20; const { data: payment } = await db

&#x20;   .from('payments')

&#x20;   .select('toss\_payment\_key')

&#x20;   .eq('reservation\_id', reservation!.id)

&#x20;   .eq('status', 'PAID')

&#x20;   .maybeSingle()



&#x20; // ── 9. 응답 (toss\_payment\_key + 환불 안내 포함) ───────────────

&#x20; return successResponse({

&#x20;   reservation\_no:   reservation!.reservation\_no,

&#x20;   status:           'CANCELLED',

&#x20;   cancelled\_at:     cancelledAt,

&#x20;   cancelled\_by:     'ADMIN',

&#x20;   toss\_payment\_key: payment?.toss\_payment\_key ?? null,

&#x20;   refund\_guide:     payment?.toss\_payment\_key

&#x20;     ? '토스페이 어드민 콘솔(https://dashboard.tosspayments.com)에서 위 paymentKey로 환불을 처리해 주세요.'

&#x20;     : null,

&#x20; })

}

```



\---



\## 7. `src/app/api/admin/storage/route.ts`



```typescript

import { NextRequest } from 'next/server'

import { z } from 'zod'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession, calcElapsedMinutes, calcWarningLevel } from '@/lib/admin'

import { errorResponse, successResponse } from '@/lib/validate'



const querySchema = z.object({

&#x20; terminal: z.enum(\['T1', 'T2']).optional(),

})



export async function GET(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. Query Params 파싱 ─────────────────────────────────────

&#x20; const { searchParams } = request.nextUrl

&#x20; const parsed = querySchema.safeParse({

&#x20;   terminal: searchParams.get('terminal') ?? undefined,

&#x20; })

&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_REQUEST', '잘못된 터미널 값입니다.')

&#x20; }

&#x20; const { terminal } = parsed.data



&#x20; const db = getSupabaseClient()



&#x20; // ── 3. STORED 상태 예약 조회 ──────────────────────────────────

&#x20; // 접수 시각 오름차순 — 오래된 짐이 위

&#x20; let query = db

&#x20;   .from('reservations')

&#x20;   .select(

&#x20;     'reservation\_no, terminal, direction, use\_date, customer\_name, customer\_phone, bag\_count, checked\_in\_at'

&#x20;   )

&#x20;   .eq('status', 'STORED')

&#x20;   .order('checked\_in\_at', { ascending: true })



&#x20; if (terminal) {

&#x20;   query = query.eq('terminal', terminal)

&#x20; }



&#x20; const { data: items, error } = await query



&#x20; if (error) {

&#x20;   console.error('\[storage] 쿼리 실패:', error)

&#x20;   return errorResponse('INTERNAL\_ERROR', '데이터 조회 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; // ── 4. 경과 시간 + 경고 레벨 계산 ───────────────────────────

&#x20; const enriched = (items ?? \[]).map((item) => {

&#x20;   const elapsed = item.checked\_in\_at

&#x20;     ? calcElapsedMinutes(item.checked\_in\_at)

&#x20;     : 0

&#x20;   return {

&#x20;     reservation\_no:        item.reservation\_no,

&#x20;     terminal:              item.terminal,

&#x20;     direction:             item.direction,

&#x20;     use\_date:              item.use\_date,

&#x20;     customer\_name:         item.customer\_name,

&#x20;     customer\_phone\_masked: maskPhoneInline(item.customer\_phone),

&#x20;     bag\_count:             item.bag\_count,

&#x20;     checked\_in\_at:         item.checked\_in\_at,

&#x20;     elapsed\_minutes:       elapsed,

&#x20;     elapsed\_warning:       calcWarningLevel(elapsed),

&#x20;   }

&#x20; })



&#x20; // ── 5. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   items:  enriched,

&#x20;   total:  enriched.length,

&#x20;   as\_of:  new Date().toISOString(),

&#x20; })

}



// 전화번호 마스킹 인라인 (lib/validate의 maskPhone 재사용 가능하나 import 순환 방지)

function maskPhoneInline(phone: string): string {

&#x20; if (!phone || phone.length !== 11) return phone

&#x20; return `${phone.slice(0, 3)}-\*\*\*\*-${phone.slice(7)}`

}

```



\---



\## 8. `src/app/api/admin/sales/route.ts`



```typescript

import { NextRequest } from 'next/server'

import {

&#x20; startOfWeek, endOfWeek,

&#x20; startOfMonth, endOfMonth,

&#x20; parseISO, format,

} from 'date-fns'

import { toZonedTime } from 'date-fns-tz'

import { getSupabaseClient } from '@/lib/supabase'

import { getAdminSession } from '@/lib/admin'

import {

&#x20; adminSalesSchema,

&#x20; errorResponse,

&#x20; successResponse,

&#x20; getZodError,

&#x20; getTodayKst,

} from '@/lib/validate'

import { RESERVATION\_AMOUNT } from '@/constants'



const TZ = 'Asia/Seoul'



// ── 날짜 범위 계산 헬퍼 ──────────────────────────────────────────

function calcDateRange(

&#x20; period: 'daily' | 'weekly' | 'monthly',

&#x20; dateStr: string

): { from: string; to: string } {

&#x20; // dateStr을 KST 기준 Date로 파싱

&#x20; const kstDate = toZonedTime(parseISO(dateStr), TZ)



&#x20; switch (period) {

&#x20;   case 'daily':

&#x20;     return { from: dateStr, to: dateStr }



&#x20;   case 'weekly': {

&#x20;     // 한국 기준 주 시작: 월요일 (weekStartsOn: 1)

&#x20;     const mon = startOfWeek(kstDate, { weekStartsOn: 1 })

&#x20;     const sun = endOfWeek(kstDate, { weekStartsOn: 1 })

&#x20;     return {

&#x20;       from: format(mon, 'yyyy-MM-dd'),

&#x20;       to:   format(sun, 'yyyy-MM-dd'),

&#x20;     }

&#x20;   }



&#x20;   case 'monthly': {

&#x20;     const first = startOfMonth(kstDate)

&#x20;     const last  = endOfMonth(kstDate)

&#x20;     return {

&#x20;       from: format(first, 'yyyy-MM-dd'),

&#x20;       to:   format(last,  'yyyy-MM-dd'),

&#x20;     }

&#x20;   }

&#x20; }

}



export async function GET(request: NextRequest): Promise<Response> {

&#x20; // ── 1. 인증 확인 ─────────────────────────────────────────────

&#x20; const { authError } = await getAdminSession()

&#x20; if (authError) return authError



&#x20; // ── 2. Query Params 파싱 + 유효성 검증 ───────────────────────

&#x20; const { searchParams } = request.nextUrl

&#x20; const parsed = adminSalesSchema.safeParse({

&#x20;   period:   searchParams.get('period')   ?? 'daily',

&#x20;   date:     searchParams.get('date')     ?? undefined,

&#x20;   terminal: searchParams.get('terminal') ?? undefined,

&#x20;   page:     searchParams.get('page')     ?? '1',

&#x20;   limit:    searchParams.get('limit')    ?? '50',

&#x20; })



&#x20; if (!parsed.success) {

&#x20;   return errorResponse('INVALID\_REQUEST', getZodError(parsed.error))

&#x20; }



&#x20; const { period, date, terminal, page, limit } = parsed.data

&#x20; const targetDate = date ?? getTodayKst()

&#x20; const { from, to } = calcDateRange(period, targetDate)

&#x20; const offset = (page - 1) \* limit



&#x20; const db = getSupabaseClient()



&#x20; // ── 3. 매출 집계 쿼리 ────────────────────────────────────────

&#x20; // payments.paid\_at 기준으로 날짜 범위 필터

&#x20; // KST 00:00 \~ 23:59:59 범위로 변환

&#x20; const fromTs = `${from}T00:00:00+09:00`

&#x20; const toTs   = `${to}T23:59:59+09:00`



&#x20; // 집계: 결제 완료 건수, 취소 건수 (payments JOIN reservations)

&#x20; // Supabase는 복잡한 집계 쿼리를 직접 지원하지 않으므로

&#x20; // 목록을 가져온 뒤 애플리케이션 레이어에서 집계



&#x20; // 집계용 — 전체 (페이징 없이)

&#x20; let summaryQuery = db

&#x20;   .from('payments')

&#x20;   .select(`

&#x20;     id,

&#x20;     status,

&#x20;     amount,

&#x20;     reservations!inner (

&#x20;       id,

&#x20;       terminal,

&#x20;       status

&#x20;     )

&#x20;   `)

&#x20;   .eq('status', 'PAID')   // 결제 완료된 건만

&#x20;   .gte('paid\_at', fromTs)

&#x20;   .lte('paid\_at', toTs)



&#x20; // 목록용 — 페이징 적용

&#x20; let listQuery = db

&#x20;   .from('payments')

&#x20;   .select(`

&#x20;     id,

&#x20;     method,

&#x20;     amount,

&#x20;     paid\_at,

&#x20;     status,

&#x20;     reservations!inner (

&#x20;       reservation\_no,

&#x20;       terminal,

&#x20;       customer\_name,

&#x20;       status

&#x20;     )

&#x20;   `)

&#x20;   .eq('status', 'PAID')

&#x20;   .gte('paid\_at', fromTs)

&#x20;   .lte('paid\_at', toTs)

&#x20;   .order('paid\_at', { ascending: false })

&#x20;   .range(offset, offset + limit - 1)



&#x20; // 취소 건 집계용 (reservations 기준 — cancelled\_at 기간 내)

&#x20; const cancelledFromTs = `${from}T00:00:00+09:00`

&#x20; const cancelledToTs   = `${to}T23:59:59+09:00`



&#x20; let cancelQuery = db

&#x20;   .from('reservations')

&#x20;   .select('id', { count: 'exact', head: true })

&#x20;   .eq('status', 'CANCELLED')

&#x20;   .gte('cancelled\_at', cancelledFromTs)

&#x20;   .lte('cancelled\_at', cancelledToTs)



&#x20; // 터미널 필터 적용

&#x20; if (terminal) {

&#x20;   // Supabase foreign table 필터 — reservations 컬럼으로 필터

&#x20;   summaryQuery = summaryQuery.eq('reservations.terminal', terminal)

&#x20;   listQuery    = listQuery.eq('reservations.terminal', terminal)

&#x20;   cancelQuery  = cancelQuery.eq('terminal', terminal)

&#x20; }



&#x20; // ── 4. 병렬 실행 ─────────────────────────────────────────────

&#x20; const \[

&#x20;   { data: summaryItems, error: summaryError },

&#x20;   { data: listItems,    error: listError    },

&#x20;   { count: cancelCount, error: cancelError  },

&#x20; ] = await Promise.all(\[summaryQuery, listQuery, cancelQuery])



&#x20; if (summaryError || listError || cancelError) {

&#x20;   console.error('\[sales] 쿼리 실패:', { summaryError, listError, cancelError })

&#x20;   return errorResponse('INTERNAL\_ERROR', '데이터 조회 중 오류가 발생했습니다.', 500)

&#x20; }



&#x20; // ── 5. 집계 계산 ─────────────────────────────────────────────

&#x20; const paidCount      = summaryItems?.length ?? 0

&#x20; const cancelledCount = cancelCount ?? 0

&#x20; const netRevenue     = (paidCount - cancelledCount) \* RESERVATION\_AMOUNT



&#x20; // ── 6. 목록 조립 ─────────────────────────────────────────────

&#x20; // Supabase join 결과 타입 단언 (reservations는 단일 객체로 반환)

&#x20; type ListItem = {

&#x20;   id:       string

&#x20;   method:   string

&#x20;   amount:   number

&#x20;   paid\_at:  string | null

&#x20;   status:   string

&#x20;   reservations: {

&#x20;     reservation\_no: string

&#x20;     terminal:       string

&#x20;     customer\_name:  string

&#x20;     status:         string

&#x20;   }

&#x20; }



&#x20; const items = (listItems as unknown as ListItem\[] ?? \[]).map((item) => ({

&#x20;   reservation\_no: item.reservations.reservation\_no,

&#x20;   terminal:       item.reservations.terminal,

&#x20;   customer\_name:  item.reservations.customer\_name,

&#x20;   method:         item.method,

&#x20;   amount:         item.amount,

&#x20;   status:         item.reservations.status,

&#x20;   is\_cancelled:   item.reservations.status === 'CANCELLED',

&#x20;   paid\_at:        item.paid\_at,

&#x20; }))



&#x20; // ── 7. 응답 ──────────────────────────────────────────────────

&#x20; return successResponse({

&#x20;   period,

&#x20;   date\_range: { from, to },

&#x20;   summary: {

&#x20;     paid\_count:      paidCount,

&#x20;     cancelled\_count: cancelledCount,

&#x20;     net\_revenue:     Math.max(0, netRevenue),  // 음수 방지 (이론상 발생 안 함)

&#x20;   },

&#x20;   items,

&#x20;   total: paidCount,   // 페이징 total = 결제 완료 총 건수

&#x20;   page,

&#x20;   limit,

&#x20; })

}

```



\---



\## 설계 노트



\### 공통 헬퍼 `lib/admin.ts` 분리 이유



BO API 8개가 모두 ① 인증 확인 → ② 예약 조회 → ③ 상태 검증 패턴을 공유한다.

`getAdminSession()`, `fetchReservation()`, `guardStatus()`를 별도 파일로 분리하면:

\- route.ts별 코드 약 40줄 절감

\- 인증·조회·검증 로직 변경 시 한 곳만 수정

\- 테스트 시 각 헬퍼 단위 테스트 가능



\### guardStatus — 단방향 테이블 설계



```

접수(checkin):  RESERVED만 허용  → 나머지 3가지 상태는 각각 다른 에러 코드

반환(checkout): STORED만 허용    → 나머지 3가지 상태는 각각 다른 에러 코드

```



`switch/case`가 아닌 `statusErrorMap` 객체 룩업으로 구현해 향후 상태값 추가 시 맵에 한 줄만 추가하면 된다.



\### sales API — JOIN 쿼리 전략



Supabase JS의 `!inner` JOIN을 활용해 payments ↔ reservations를 단일 쿼리로 처리한다.

집계(paid\_count, net\_revenue)는 summary용 쿼리와 list용 쿼리를 분리해 `Promise.all` 병렬 실행한다.



취소 건수는 `payments.status = CANCELLED`가 아닌 `reservations.status = CANCELLED` + `cancelled\_at` 기간 필터로 계산한다.

이유: 토스페이 환불이 수동이므로 `payments.status`가 즉시 업데이트되지 않을 수 있기 때문이다.



\### 낙관적 잠금 패턴 (checkin / checkout / cancel)



```typescript

// UPDATE WHERE id = $id AND status = $expectedStatus

const { data: updated } = await db

&#x20; .from('reservations')

&#x20; .update({ status: 'STORED', ... })

&#x20; .eq('id', reservation.id)

&#x20; .eq('status', 'RESERVED')   // ← 이 조건이 낙관적 잠금

&#x20; .select('id')

&#x20; .maybeSingle()



if (!updated) {

&#x20; // 동시 요청이 먼저 처리됨 → 상태 불일치 에러 반환

}

```



SELECT 후 UPDATE 사이에 다른 요청이 상태를 변경해도 `WHERE status = 'RESERVED'` 조건이 실패하여 `updated = null`이 되므로 중복 처리가 원천 차단된다.



\---



\## Session 08 체크리스트



| 파일 | 구현 항목 | 상태 |

|------|----------|------|

| `lib/admin.ts` (공통 헬퍼) | getAdminSession / fetchReservation / guardStatus / calcElapsed / calcWarning | ✅ |

| `dashboard/route.ts` | 병렬 쿼리 4개 / 터미널 필터 / elapsed\_minutes | ✅ |

| `reservations/route.ts` | 날짜·터미널·상태 동적 필터 / 페이징 / 건수 병렬 조회 | ✅ |

| `reservations/\[id]/route.ts` | toss\_payment\_key 포함 / 이력 조회 | ✅ |

| `checkin/route.ts` | 낙관적 잠금 / 로그 / SMS 비동기 | ✅ |

| `checkout/route.ts` | 낙관적 잠금 / 로그 / SMS 비동기 | ✅ |

| `cancel/route.ts` | RESERVED만 허용 / 로그 / toss\_payment\_key + 환불 안내 응답 | ✅ |

| `storage/route.ts` | checked\_in\_at 오름차순 / 경과 시간 + 경고 레벨 | ✅ |

| `sales/route.ts` | 일/주/월 날짜 범위 계산 / JOIN 쿼리 / 집계 + 목록 분리 | ✅ |



\---



\*다음 세션: Session 09 — NextAuth.js 설정 완성 및 BO 로그인 페이지 구현\*

