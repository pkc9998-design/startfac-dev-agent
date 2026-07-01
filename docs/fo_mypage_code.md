\# BagDrop FO 예약 확인·상세·이용 안내 구현 코드



> 작성: 개발 에이전트 가온

> 세션: Session 11

> 최종 업데이트: 2026-07-01

> 대상: 예약 조회, 예약 상세, StatusBadge, 이용 안내 (4개 파일)

> 기준: 모바일웹 max-width 430px / Tailwind CSS

> 상태: ✅ Session 11 확정



\---



\## 파일 목록



| 파일 경로 | 역할 |

|-----------|------|

| `src/components/fo/StatusBadge.tsx` | 예약 상태 배지 (공통) |

| `src/app/my-reservation/page.tsx` | 예약번호 조회 화면 |

| `src/app/my-reservation/\[reservationNo]/page.tsx` | 예약 상세 화면 |

| `src/app/guide/page.tsx` | 이용 안내 화면 |



\---



\## 1. `src/components/fo/StatusBadge.tsx`



```typescript

import type { ReservationStatus } from '@/lib/supabase'



interface StatusBadgeProps {

&#x20; status: ReservationStatus

&#x20; size?:  'sm' | 'md' | 'lg'

}



const STATUS\_CONFIG: Record<

&#x20; ReservationStatus,

&#x20; { label: string; bg: string; text: string; dot: string }

> = {

&#x20; RESERVED:  {

&#x20;   label: '예약완료',

&#x20;   bg:   'bg-blue-50',

&#x20;   text: 'text-blue-700',

&#x20;   dot:  'bg-blue-500',

&#x20; },

&#x20; STORED:    {

&#x20;   label: '보관중',

&#x20;   bg:   'bg-green-50',

&#x20;   text: 'text-green-700',

&#x20;   dot:  'bg-green-500',

&#x20; },

&#x20; RETURNED:  {

&#x20;   label: '반환완료',

&#x20;   bg:   'bg-gray-100',

&#x20;   text: 'text-gray-600',

&#x20;   dot:  'bg-gray-400',

&#x20; },

&#x20; CANCELLED: {

&#x20;   label: '취소완료',

&#x20;   bg:   'bg-red-50',

&#x20;   text: 'text-red-600',

&#x20;   dot:  'bg-red-400',

&#x20; },

}



const SIZE\_CONFIG = {

&#x20; sm: { badge: 'px-2 py-0.5 text-\[11px]', dot: 'h-1.5 w-1.5' },

&#x20; md: { badge: 'px-3 py-1 text-xs',       dot: 'h-2 w-2' },

&#x20; lg: { badge: 'px-4 py-1.5 text-sm',     dot: 'h-2 w-2' },

}



export default function StatusBadge({ status, size = 'md' }: StatusBadgeProps) {

&#x20; const cfg  = STATUS\_CONFIG\[status]

&#x20; const sz   = SIZE\_CONFIG\[size]



&#x20; return (

&#x20;   <span

&#x20;     className={`

&#x20;       inline-flex items-center gap-1.5 rounded-full font-semibold

&#x20;       ${cfg.bg} ${cfg.text} ${sz.badge}

&#x20;     `}

&#x20;   >

&#x20;     <span className={`rounded-full ${cfg.dot} ${sz.dot} shrink-0`} />

&#x20;     {cfg.label}

&#x20;   </span>

&#x20; )

}

```



\---



\## 2. `src/app/my-reservation/page.tsx`



```typescript

'use client'



import { useState, FormEvent, useRef, useEffect } from 'react'

import { useRouter, useSearchParams } from 'next/navigation'

import Link from 'next/link'



// 예약번호 형식 자동 포맷: 입력 중 대문자 변환 + 하이픈 보조

function formatReservationNo(raw: string): string {

&#x20; // 대문자 + 숫자 + 하이픈만 남김

&#x20; const clean = raw.toUpperCase().replace(/\[^A-Z0-9-]/g, '')

&#x20; return clean.slice(0, 20)   // BD-YYYYMMDD-NNNN = 최대 20자

}



// 예약번호 형식 유효성: BD-YYYYMMDD-NNNN

function isValidReservationNo(value: string): boolean {

&#x20; return /^BD-\\d{8}-\\d{4}$/.test(value)

}



export default function MyReservationPage() {

&#x20; const router       = useRouter()

&#x20; const searchParams = useSearchParams()



&#x20; const inputRef      = useRef<HTMLInputElement>(null)

&#x20; const \[value,       setValue]       = useState('')

&#x20; const \[inputError,  setInputError]  = useState('')

&#x20; const \[apiError,    setApiError]    = useState('')

&#x20; const \[isLoading,   setIsLoading]   = useState(false)



&#x20; // URL 파라미터로 에러 메시지 전달된 경우 (예: 이전 페이지에서 리디렉트)

&#x20; useEffect(() => {

&#x20;   const errParam = searchParams.get('error')

&#x20;   if (errParam === 'not\_found') {

&#x20;     setApiError('해당 예약번호를 찾을 수 없습니다. 다시 확인해 주세요.')

&#x20;   }

&#x20;   inputRef.current?.focus()

&#x20; }, \[searchParams])



&#x20; function handleChange(e: React.ChangeEvent<HTMLInputElement>) {

&#x20;   const formatted = formatReservationNo(e.target.value)

&#x20;   setValue(formatted)

&#x20;   // 입력 중에는 에러 초기화

&#x20;   if (inputError) setInputError('')

&#x20;   if (apiError)   setApiError('')

&#x20; }



&#x20; async function handleSubmit(e: FormEvent) {

&#x20;   e.preventDefault()

&#x20;   if (isLoading) return



&#x20;   // 형식 검증

&#x20;   if (!isValidReservationNo(value)) {

&#x20;     setInputError('올바른 예약번호 형식이 아닙니다. (예: BD-20260715-0001)')

&#x20;     inputRef.current?.focus()

&#x20;     return

&#x20;   }



&#x20;   setIsLoading(true)

&#x20;   setApiError('')



&#x20;   try {

&#x20;     const res  = await fetch(`/api/reservations/${value}`)

&#x20;     const json = await res.json()



&#x20;     if (res.status === 404 || (json.error?.code === 'RESERVATION\_NOT\_FOUND')) {

&#x20;       setApiError('해당 예약번호를 찾을 수 없습니다. 다시 확인해 주세요.')

&#x20;       return

&#x20;     }



&#x20;     if (!res.ok || !json.success) {

&#x20;       throw new Error(json.error?.message ?? '조회 중 오류가 발생했습니다.')

&#x20;     }



&#x20;     // 존재하면 상세 페이지로 이동

&#x20;     router.push(`/my-reservation/${value}`)

&#x20;   } catch (err) {

&#x20;     setApiError(

&#x20;       err instanceof Error ? err.message : '조회 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.'

&#x20;     )

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;   }

&#x20; }



&#x20; const canSubmit = isValidReservationNo(value) \&\& !isLoading



&#x20; return (

&#x20;   <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-white">



&#x20;     {/\* ── GNB ──────────────────────────────────────────── \*/}

&#x20;     <header className="flex h-14 items-center gap-3 border-b border-gray-100 px-4">

&#x20;       <Link

&#x20;         href="/"

&#x20;         aria-label="홈으로"

&#x20;         className="-ml-1 rounded-lg p-1.5 text-gray-500 hover:bg-gray-100"

&#x20;       >

&#x20;         <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;           <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;         </svg>

&#x20;       </Link>

&#x20;       <h1 className="text-base font-semibold text-gray-900">예약 확인</h1>

&#x20;     </header>



&#x20;     {/\* ── 조회 폼 ───────────────────────────────────────── \*/}

&#x20;     <form onSubmit={handleSubmit} className="flex flex-1 flex-col px-5 pt-10">

&#x20;       <div className="mb-8">

&#x20;         <p className="mb-6 text-lg font-bold text-gray-900">

&#x20;           예약번호를 입력해 주세요

&#x20;         </p>



&#x20;         {/\* 예약번호 입력 \*/}

&#x20;         <div className="mb-3">

&#x20;           <input

&#x20;             ref={inputRef}

&#x20;             type="text"

&#x20;             value={value}

&#x20;             onChange={handleChange}

&#x20;             placeholder="BD-YYYYMMDD-NNNN"

&#x20;             autoCapitalize="characters"

&#x20;             autoCorrect="off"

&#x20;             autoComplete="off"

&#x20;             spellCheck={false}

&#x20;             maxLength={20}

&#x20;             disabled={isLoading}

&#x20;             className={`

&#x20;               w-full rounded-xl border bg-white px-4 py-3.5

&#x20;               font-mono text-base tracking-wider text-gray-900

&#x20;               placeholder:font-sans placeholder:text-sm placeholder:tracking-normal placeholder:text-gray-400

&#x20;               focus:outline-none focus:ring-2

&#x20;               disabled:bg-gray-50 disabled:text-gray-400

&#x20;               ${inputError

&#x20;                 ? 'border-red-400 focus:border-red-400 focus:ring-red-400/20'

&#x20;                 : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500/20'

&#x20;               }

&#x20;             `}

&#x20;           />

&#x20;           {inputError \&\& (

&#x20;             <p className="mt-1.5 flex items-center gap-1 text-xs text-red-500">

&#x20;               <svg className="h-3.5 w-3.5 shrink-0" fill="currentColor" viewBox="0 0 20 20">

&#x20;                 <path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-8-5a.75.75 0 01.75.75v4.5a.75.75 0 01-1.5 0v-4.5A.75.75 0 0110 5zm0 10a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" />

&#x20;               </svg>

&#x20;               {inputError}

&#x20;             </p>

&#x20;           )}

&#x20;         </div>



&#x20;         {/\* API 오류 \*/}

&#x20;         {apiError \&\& (

&#x20;           <div className="rounded-xl bg-red-50 px-4 py-3 text-sm text-red-600">

&#x20;             {apiError}

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* 조회 버튼 \*/}

&#x20;         <button

&#x20;           type="submit"

&#x20;           disabled={!canSubmit}

&#x20;           className="

&#x20;             mt-4 flex w-full items-center justify-center gap-2

&#x20;             rounded-2xl py-4 text-base font-bold transition-colors

&#x20;             disabled:bg-gray-200 disabled:text-gray-400

&#x20;             enabled:bg-blue-600 enabled:text-white enabled:active:bg-blue-700

&#x20;           "

&#x20;         >

&#x20;           {isLoading ? (

&#x20;             <>

&#x20;               <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;               <span>조회 중…</span>

&#x20;             </>

&#x20;           ) : (

&#x20;             '예약 조회하기'

&#x20;           )}

&#x20;         </button>

&#x20;       </div>



&#x20;       {/\* 구분선 \*/}

&#x20;       <div className="mb-6 flex items-center gap-4">

&#x20;         <div className="h-px flex-1 bg-gray-100" />

&#x20;         <span className="text-xs text-gray-400">또는</span>

&#x20;         <div className="h-px flex-1 bg-gray-100" />

&#x20;       </div>



&#x20;       {/\* QR 안내 \*/}

&#x20;       <div className="rounded-2xl bg-gray-50 px-5 py-4 text-center">

&#x20;         <div className="mb-2 text-2xl">📱</div>

&#x20;         <p className="text-sm text-gray-600">

&#x20;           예약 완료 화면에서 보여드린<br />

&#x20;           QR 코드로도 확인 가능합니다

&#x20;         </p>

&#x20;       </div>

&#x20;     </form>

&#x20;   </div>

&#x20; )

}

```



\---



\## 3. `src/app/my-reservation/\[reservationNo]/page.tsx`



```typescript

'use client'



import { useState, useEffect, useCallback } from 'react'

import { useRouter } from 'next/navigation'

import Link from 'next/link'

import ReservationQR from '@/components/fo/ReservationQR'

import StatusBadge from '@/components/fo/StatusBadge'

import { TERMINAL\_LABEL, DIRECTION\_LABEL } from '@/constants'

import type { ReservationStatus } from '@/lib/supabase'

import { format, parseISO, isBefore, startOfDay } from 'date-fns'

import { ko } from 'date-fns/locale'



// ── 타입 ─────────────────────────────────────────────────────────



interface LogEntry {

&#x20; from\_status:  ReservationStatus | null

&#x20; to\_status:    ReservationStatus

&#x20; actor\_label:  string | null

&#x20; created\_at:   string

}



interface ReservationDetail {

&#x20; reservation\_no:        string

&#x20; terminal:              'T1' | 'T2'

&#x20; direction:             'DEPARTURE' | 'ARRIVAL'

&#x20; use\_date:              string

&#x20; bag\_count:             number

&#x20; customer\_name:         string

&#x20; customer\_phone\_masked: string

&#x20; amount:                number

&#x20; status:                ReservationStatus

&#x20; checked\_in\_at:         string | null

&#x20; checked\_out\_at:        string | null

&#x20; cancelled\_at:          string | null

&#x20; cancelled\_by:          'CUSTOMER' | 'ADMIN' | null

&#x20; is\_cancellable:        boolean

&#x20; payment: {

&#x20;   method:  string

&#x20;   paid\_at: string | null

&#x20; } | null

&#x20; logs: LogEntry\[]

}



// ── 서브 컴포넌트 ─────────────────────────────────────────────────



// 정보 행

function InfoRow({ label, value }: { label: string; value: string }) {

&#x20; return (

&#x20;   <div className="flex items-start justify-between gap-3 py-2.5">

&#x20;     <span className="shrink-0 text-sm text-gray-500">{label}</span>

&#x20;     <span className="text-right text-sm font-medium text-gray-900">{value}</span>

&#x20;   </div>

&#x20; )

}



// 복사 토스트

function CopyToast({ visible }: { visible: boolean }) {

&#x20; return (

&#x20;   <div

&#x20;     aria-live="polite"

&#x20;     className={`

&#x20;       pointer-events-none fixed left-1/2 top-20 z-50

&#x20;       -translate-x-1/2 rounded-full bg-gray-900 px-5 py-2.5

&#x20;       text-sm font-medium text-white shadow-lg

&#x20;       transition-all duration-300

&#x20;       ${visible ? 'opacity-100 translate-y-0' : 'opacity-0 -translate-y-2'}

&#x20;     `}

&#x20;   >

&#x20;     예약번호가 복사됐습니다

&#x20;   </div>

&#x20; )

}



// 이력 타임라인

function LogTimeline({ logs }: { logs: LogEntry\[] }) {

&#x20; if (!logs.length) return null



&#x20; return (

&#x20;   <div className="rounded-2xl bg-white px-5 py-4 shadow-sm">

&#x20;     <h3 className="mb-4 text-sm font-semibold text-gray-700">처리 이력</h3>

&#x20;     <ol className="relative space-y-4 border-l border-gray-200 pl-4">

&#x20;       {logs.map((log, idx) => {

&#x20;         const isLast = idx === logs.length - 1

&#x20;         const dt = parseISO(log.created\_at)

&#x20;         return (

&#x20;           <li key={idx} className="relative">

&#x20;             {/\* 타임라인 도트 \*/}

&#x20;             <span

&#x20;               className={`

&#x20;                 absolute -left-\[1.1rem] flex h-3.5 w-3.5 items-center justify-center

&#x20;                 rounded-full border-2 border-white

&#x20;                 ${isLast ? 'bg-blue-500' : 'bg-gray-300'}

&#x20;               `}

&#x20;             />

&#x20;             <div className="ml-1">

&#x20;               <p className="text-sm font-medium text-gray-900">

&#x20;                 {log.actor\_label ?? log.to\_status}

&#x20;               </p>

&#x20;               <time

&#x20;                 dateTime={log.created\_at}

&#x20;                 className="text-xs text-gray-400"

&#x20;               >

&#x20;                 {format(dt, 'yyyy.MM.dd HH:mm', { locale: ko })}

&#x20;               </time>

&#x20;             </div>

&#x20;           </li>

&#x20;         )

&#x20;       })}

&#x20;     </ol>

&#x20;   </div>

&#x20; )

}



// 취소 영역 — 케이스별 4가지 분기

function CancelSection({

&#x20; status,

&#x20; useDate,

&#x20; isCancellable,

&#x20; onCancelClick,

}: {

&#x20; status:        ReservationStatus

&#x20; useDate:       string

&#x20; isCancellable: boolean

&#x20; onCancelClick: () => void

}) {

&#x20; // Case 4: RETURNED / CANCELLED — 아무것도 표시하지 않음

&#x20; if (status === 'RETURNED' || status === 'CANCELLED') return null



&#x20; return (

&#x20;   <div className="rounded-2xl border border-gray-100 bg-gray-50 px-5 py-4">

&#x20;     {/\* Case 1: 취소 가능 (RESERVED + 이용일 전) \*/}

&#x20;     {status === 'RESERVED' \&\& isCancellable \&\& (

&#x20;       <div>

&#x20;         <p className="mb-1 text-sm font-medium text-gray-700">예약을 취소하시겠어요?</p>

&#x20;         <p className="mb-4 text-xs text-gray-500">이용일 전날 자정까지 100% 환불됩니다.</p>

&#x20;         <button

&#x20;           onClick={onCancelClick}

&#x20;           className="

&#x20;             w-full rounded-xl border-2 border-red-400 bg-white

&#x20;             py-3.5 text-sm font-semibold text-red-500

&#x20;             transition-colors active:bg-red-50

&#x20;           "

&#x20;         >

&#x20;           예약 취소하기

&#x20;         </button>

&#x20;       </div>

&#x20;     )}



&#x20;     {/\* Case 2: 취소 불가 — 당일 이후 \*/}

&#x20;     {status === 'RESERVED' \&\& !isCancellable \&\& (

&#x20;       <div className="flex items-start gap-2">

&#x20;         <svg className="mt-0.5 h-4 w-4 shrink-0 text-amber-500" fill="currentColor" viewBox="0 0 20 20">

&#x20;           <path fillRule="evenodd" d="M8.485 2.495c.673-1.167 2.357-1.167 3.03 0l6.28 10.875c.673 1.167-.17 2.625-1.516 2.625H3.72c-1.347 0-2.189-1.458-1.515-2.625L8.485 2.495zM10 5a.75.75 0 01.75.75v3.5a.75.75 0 01-1.5 0v-3.5A.75.75 0 0110 5zm0 9a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" />

&#x20;         </svg>

&#x20;         <p className="text-sm text-gray-700">

&#x20;           이용일 당일은 취소할 수 없습니다.

&#x20;         </p>

&#x20;       </div>

&#x20;     )}



&#x20;     {/\* Case 3: 취소 불가 — 이미 접수됨 \*/}

&#x20;     {status === 'STORED' \&\& (

&#x20;       <div className="flex items-start gap-2">

&#x20;         <svg className="mt-0.5 h-4 w-4 shrink-0 text-green-500" fill="currentColor" viewBox="0 0 20 20">

&#x20;           <path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-8-5a.75.75 0 01.75.75v4.5a.75.75 0 01-1.5 0v-4.5A.75.75 0 0110 5zm0 10a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" />

&#x20;         </svg>

&#x20;         <p className="text-sm text-gray-700">

&#x20;           짐이 이미 접수된 상태라 취소할 수 없습니다.

&#x20;         </p>

&#x20;       </div>

&#x20;     )}

&#x20;   </div>

&#x20; )

}



// 취소 확인 모달

function CancelModal({

&#x20; reservationNo,

&#x20; onConfirm,

&#x20; onClose,

&#x20; isLoading,

&#x20; errorMsg,

}: {

&#x20; reservationNo: string

&#x20; onConfirm:     () => void

&#x20; onClose:       () => void

&#x20; isLoading:     boolean

&#x20; errorMsg:      string | null

}) {

&#x20; return (

&#x20;   <div

&#x20;     className="fixed inset-0 z-50 flex items-end justify-center bg-black/50"

&#x20;     onClick={(e) => { if (e.target === e.currentTarget) onClose() }}

&#x20;   >

&#x20;     <div className="w-full max-w-\[430px] rounded-t-3xl bg-white px-5 pb-10 pt-6 shadow-2xl">

&#x20;       {/\* 드래그 핸들 \*/}

&#x20;       <div className="mx-auto mb-5 h-1 w-10 rounded-full bg-gray-200" />



&#x20;       <h2 className="mb-2 text-center text-lg font-bold text-gray-900">

&#x20;         예약을 취소할까요?

&#x20;       </h2>

&#x20;       <p className="mb-1 text-center text-sm text-gray-500">

&#x20;         취소 시 결제 금액 전액이 환불되며,

&#x20;       </p>

&#x20;       <p className="mb-6 text-center text-sm text-gray-500">복구할 수 없습니다.</p>



&#x20;       {/\* API 오류 \*/}

&#x20;       {errorMsg \&\& (

&#x20;         <div className="mb-4 rounded-xl bg-red-50 px-4 py-3 text-center text-sm text-red-600">

&#x20;           {errorMsg}

&#x20;         </div>

&#x20;       )}



&#x20;       <div className="flex gap-3">

&#x20;         <button

&#x20;           onClick={onClose}

&#x20;           disabled={isLoading}

&#x20;           className="flex-1 rounded-xl border border-gray-300 py-4 text-sm font-semibold text-gray-700 active:bg-gray-50 disabled:opacity-50"

&#x20;         >

&#x20;           돌아가기

&#x20;         </button>

&#x20;         <button

&#x20;           onClick={onConfirm}

&#x20;           disabled={isLoading}

&#x20;           className="flex-1 rounded-xl bg-red-500 py-4 text-sm font-semibold text-white active:bg-red-600 disabled:opacity-50"

&#x20;         >

&#x20;           {isLoading ? (

&#x20;             <span className="flex items-center justify-center gap-2">

&#x20;               <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;               처리 중…

&#x20;             </span>

&#x20;           ) : (

&#x20;             '취소하기'

&#x20;           )}

&#x20;         </button>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// 스켈레톤 UI (로딩 중)

function DetailSkeleton() {

&#x20; return (

&#x20;   <div className="space-y-4 px-4 pt-5">

&#x20;     {/\* 상태 배지 스켈레톤 \*/}

&#x20;     <div className="h-7 w-20 animate-pulse rounded-full bg-gray-200" />

&#x20;     {/\* QR 스켈레톤 \*/}

&#x20;     <div className="flex justify-center py-4">

&#x20;       <div className="h-64 w-64 animate-pulse rounded-2xl bg-gray-200" />

&#x20;     </div>

&#x20;     {/\* 정보 카드 스켈레톤 \*/}

&#x20;     {\[1, 2, 3].map((i) => (

&#x20;       <div key={i} className="rounded-2xl bg-white p-5 shadow-sm">

&#x20;         <div className="space-y-3">

&#x20;           {\[1, 2, 3].map((j) => (

&#x20;             <div key={j} className="flex justify-between">

&#x20;               <div className="h-3.5 w-16 animate-pulse rounded bg-gray-200" />

&#x20;               <div className="h-3.5 w-28 animate-pulse rounded bg-gray-200" />

&#x20;             </div>

&#x20;           ))}

&#x20;         </div>

&#x20;       </div>

&#x20;     ))}

&#x20;   </div>

&#x20; )

}



// ── 메인 페이지 ───────────────────────────────────────────────────



interface PageProps {

&#x20; params: { reservationNo: string }

}



export default function ReservationDetailPage({ params }: PageProps) {

&#x20; const { reservationNo } = params

&#x20; const router = useRouter()



&#x20; const \[data,        setData]        = useState<ReservationDetail | null>(null)

&#x20; const \[isLoading,   setIsLoading]   = useState(true)

&#x20; const \[fetchError,  setFetchError]  = useState<string | null>(null)

&#x20; const \[copyToast,   setCopyToast]   = useState(false)

&#x20; const \[showModal,   setShowModal]   = useState(false)

&#x20; const \[cancelLoading, setCancelLoading] = useState(false)

&#x20; const \[cancelError,   setCancelError]   = useState<string | null>(null)



&#x20; // 예약 상세 조회

&#x20; const fetchDetail = useCallback(async () => {

&#x20;   try {

&#x20;     const res  = await fetch(`/api/reservations/${reservationNo}`)

&#x20;     const json = await res.json()



&#x20;     if (res.status === 404) {

&#x20;       setFetchError('해당 예약번호를 찾을 수 없습니다.')

&#x20;       return

&#x20;     }

&#x20;     if (!res.ok || !json.success) {

&#x20;       throw new Error(json.error?.message ?? '예약 정보를 불러오지 못했습니다.')

&#x20;     }



&#x20;     setData(json.data as ReservationDetail)

&#x20;   } catch (err) {

&#x20;     setFetchError(err instanceof Error ? err.message : '오류가 발생했습니다.')

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;   }

&#x20; }, \[reservationNo])



&#x20; useEffect(() => {

&#x20;   fetchDetail()

&#x20; }, \[fetchDetail])



&#x20; // 예약번호 복사

&#x20; const handleCopy = useCallback(async () => {

&#x20;   try {

&#x20;     await navigator.clipboard.writeText(reservationNo)

&#x20;     setCopyToast(true)

&#x20;     setTimeout(() => setCopyToast(false), 2000)

&#x20;   } catch {}

&#x20; }, \[reservationNo])



&#x20; // 취소 실행

&#x20; async function handleCancelConfirm() {

&#x20;   setCancelLoading(true)

&#x20;   setCancelError(null)



&#x20;   try {

&#x20;     const res  = await fetch(`/api/reservations/${reservationNo}/cancel`, {

&#x20;       method: 'POST',

&#x20;     })

&#x20;     const json = await res.json()



&#x20;     if (!res.ok || !json.success) {

&#x20;       // 취소 가능 시간 경과 → 화면 새로고침

&#x20;       if (json.error?.code === 'CANCEL\_NOT\_ALLOWED') {

&#x20;         setCancelError('취소 가능 시간이 지났습니다. 이용일 당일은 취소할 수 없습니다.')

&#x20;         // 2초 후 데이터 재조회 (is\_cancellable 업데이트)

&#x20;         setTimeout(() => { setShowModal(false); fetchDetail() }, 2000)

&#x20;         return

&#x20;       }

&#x20;       throw new Error(json.error?.message ?? '취소 처리 중 오류가 발생했습니다.')

&#x20;     }



&#x20;     // 취소 성공: 모달 닫고 화면 갱신

&#x20;     setShowModal(false)

&#x20;     await fetchDetail()

&#x20;   } catch (err) {

&#x20;     setCancelError(

&#x20;       err instanceof Error

&#x20;         ? err.message

&#x20;         : '취소 처리 중 오류가 발생했습니다. 고객센터로 문의해 주세요.'

&#x20;     )

&#x20;   } finally {

&#x20;     setCancelLoading(false)

&#x20;   }

&#x20; }



&#x20; // 날짜 포맷

&#x20; const dateLabel = data?.use\_date

&#x20;   ? format(parseISO(data.use\_date), 'yyyy.MM.dd(eee)', { locale: ko })

&#x20;   : ''



&#x20; // 결제일시 포맷

&#x20; const paidAtLabel = data?.payment?.paid\_at

&#x20;   ? format(parseISO(data.payment.paid\_at), 'yyyy.MM.dd HH:mm', { locale: ko })

&#x20;   : '-'



&#x20; // 결제 수단 표시

&#x20; const methodLabel: Record<string, string> = {

&#x20;   TOSSPAY:  '토스페이',

&#x20;   KAKAOPAY: '카카오페이',

&#x20; }



&#x20; // ── 로딩 ──

&#x20; if (isLoading) {

&#x20;   return (

&#x20;     <div className="mx-auto min-h-svh max-w-\[430px] bg-gray-50">

&#x20;       <header className="flex h-14 items-center gap-3 border-b border-gray-100 bg-white px-4">

&#x20;         <div className="h-5 w-5 rounded bg-gray-200 animate-pulse" />

&#x20;         <div className="h-4 w-20 rounded bg-gray-200 animate-pulse" />

&#x20;       </header>

&#x20;       <DetailSkeleton />

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; // ── 오류 ──

&#x20; if (fetchError || !data) {

&#x20;   return (

&#x20;     <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-white">

&#x20;       <header className="flex h-14 items-center gap-3 border-b border-gray-100 px-4">

&#x20;         <Link href="/my-reservation" className="-ml-1 rounded-lg p-1.5 text-gray-500 hover:bg-gray-100">

&#x20;           <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;             <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;           </svg>

&#x20;         </Link>

&#x20;         <h1 className="text-base font-semibold text-gray-900">예약 상세</h1>

&#x20;       </header>

&#x20;       <div className="flex flex-1 flex-col items-center justify-center gap-4 px-5">

&#x20;         <div className="text-4xl">🔍</div>

&#x20;         <p className="text-center text-sm text-gray-600">

&#x20;           {fetchError ?? '예약 정보를 찾을 수 없습니다.'}

&#x20;         </p>

&#x20;         <Link

&#x20;           href="/my-reservation"

&#x20;           className="rounded-xl bg-blue-600 px-6 py-3 text-sm font-semibold text-white"

&#x20;         >

&#x20;           다시 조회하기

&#x20;         </Link>

&#x20;       </div>

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; // ── 정상 렌더 ──

&#x20; return (

&#x20;   <>

&#x20;     <div className="mx-auto min-h-svh max-w-\[430px] bg-gray-50">

&#x20;       {/\* GNB \*/}

&#x20;       <header className="sticky top-0 z-10 flex h-14 items-center gap-3 border-b border-gray-100 bg-white px-4">

&#x20;         <Link

&#x20;           href="/my-reservation"

&#x20;           aria-label="예약 확인으로"

&#x20;           className="-ml-1 rounded-lg p-1.5 text-gray-500 hover:bg-gray-100"

&#x20;         >

&#x20;           <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;             <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;           </svg>

&#x20;         </Link>

&#x20;         <h1 className="text-base font-semibold text-gray-900">예약 상세</h1>

&#x20;       </header>



&#x20;       <CopyToast visible={copyToast} />



&#x20;       <div className="space-y-4 px-4 pb-12 pt-5">



&#x20;         {/\* ── 상태 배지 ────────────────────────────────── \*/}

&#x20;         <StatusBadge status={data.status} size="lg" />



&#x20;         {/\* ── QR 코드 (RESERVED / STORED만 표시) ──────── \*/}

&#x20;         {(data.status === 'RESERVED' || data.status === 'STORED') \&\& (

&#x20;           <div className="flex flex-col items-center gap-3 rounded-2xl bg-white py-6 shadow-sm">

&#x20;             <ReservationQR value={data.reservation\_no} size={220} />

&#x20;             <p className="text-xs text-gray-400">부스에서 이 QR 코드를 보여주세요</p>

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* ── 예약번호 (탭하면 복사) ───────────────────── \*/}

&#x20;         <button

&#x20;           onClick={handleCopy}

&#x20;           className="flex w-full items-center justify-between rounded-2xl bg-white px-5 py-4 shadow-sm active:bg-gray-50"

&#x20;           aria-label="예약번호 복사"

&#x20;         >

&#x20;           <span className="text-sm text-gray-500">예약번호</span>

&#x20;           <div className="flex items-center gap-2">

&#x20;             <span className="font-mono text-sm font-bold tracking-wider text-blue-600">

&#x20;               {data.reservation\_no}

&#x20;             </span>

&#x20;             <svg className="h-4 w-4 text-gray-400" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;               <path strokeLinecap="round" strokeLinejoin="round" d="M15.666 3.888A2.25 2.25 0 0013.5 2.25h-3c-1.03 0-1.9.693-2.166 1.638m7.332 0c.055.194.084.4.084.612v0a.75.75 0 01-.75.75H9a.75.75 0 01-.75-.75v0c0-.212.03-.418.084-.612m7.332 0c.646.049 1.288.11 1.927.184 1.1.128 1.907 1.077 1.907 2.185V19.5a2.25 2.25 0 01-2.25 2.25H6.75A2.25 2.25 0 014.5 19.5V6.337c0-1.108.806-2.057 1.907-2.185a48.208 48.208 0 011.927-.184" />

&#x20;             </svg>

&#x20;           </div>

&#x20;         </button>



&#x20;         {/\* ── 예약 정보 ────────────────────────────────── \*/}

&#x20;         <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;           <div className="py-2">

&#x20;             <h3 className="py-1 text-xs font-semibold uppercase tracking-wide text-gray-400">예약 정보</h3>

&#x20;           </div>

&#x20;           <InfoRow label="터미널"   value={TERMINAL\_LABEL\[data.terminal]   ?? data.terminal} />

&#x20;           <InfoRow label="방향"     value={DIRECTION\_LABEL\[data.direction] ?? data.direction} />

&#x20;           <InfoRow label="날짜"     value={dateLabel} />

&#x20;           <InfoRow label="짐 개수"  value={`${data.bag\_count}개`} />

&#x20;           <InfoRow label="예약자"   value={data.customer\_name} />

&#x20;           <InfoRow label="연락처"   value={data.customer\_phone\_masked} />

&#x20;         </div>



&#x20;         {/\* ── 결제 정보 ────────────────────────────────── \*/}

&#x20;         {data.payment \&\& (

&#x20;           <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;             <div className="py-2">

&#x20;               <h3 className="py-1 text-xs font-semibold uppercase tracking-wide text-gray-400">결제 정보</h3>

&#x20;             </div>

&#x20;             <InfoRow label="결제 수단" value={methodLabel\[data.payment.method] ?? data.payment.method} />

&#x20;             <InfoRow label="결제 금액" value={`${data.amount.toLocaleString()}원`} />

&#x20;             <InfoRow label="결제 일시" value={paidAtLabel} />

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* ── 처리 이력 ────────────────────────────────── \*/}

&#x20;         <LogTimeline logs={data.logs} />



&#x20;         {/\* ── 취소 영역 (케이스별 4가지 분기) ─────────── \*/}

&#x20;         <CancelSection

&#x20;           status={data.status}

&#x20;           useDate={data.use\_date}

&#x20;           isCancellable={data.is\_cancellable}

&#x20;           onCancelClick={() => { setCancelError(null); setShowModal(true) }}

&#x20;         />



&#x20;       </div>

&#x20;     </div>



&#x20;     {/\* ── 취소 확인 모달 ─────────────────────────────── \*/}

&#x20;     {showModal \&\& (

&#x20;       <CancelModal

&#x20;         reservationNo={data.reservation\_no}

&#x20;         onConfirm={handleCancelConfirm}

&#x20;         onClose={() => { if (!cancelLoading) setShowModal(false) }}

&#x20;         isLoading={cancelLoading}

&#x20;         errorMsg={cancelError}

&#x20;       />

&#x20;     )}

&#x20;   </>

&#x20; )

}

```



\---



\## 4. `src/app/guide/page.tsx`



```typescript

import Link from 'next/link'



// ── 이용 순서 데이터 ──────────────────────────────────────────────



const STEPS = \[

&#x20; {

&#x20;   no:    '01',

&#x20;   emoji: '📱',

&#x20;   title: '온라인으로 예약',

&#x20;   desc:  '홈 화면에서 터미널, 방향, 날짜를 선택하고 토스페이로 결제하면 QR 코드가 발급됩니다.',

&#x20; },

&#x20; {

&#x20;   no:    '02',

&#x20;   emoji: '🧳',

&#x20;   title: '부스에서 짐 맡기기',

&#x20;   desc:  '터미널 입구 BagDrop 부스에서 QR 코드 또는 예약번호를 보여주고 짐을 맡기세요.',

&#x20; },

&#x20; {

&#x20;   no:    '03',

&#x20;   emoji: '🚗',

&#x20;   title: '가볍게 주차',

&#x20;   desc:  '짐 없이 주차장으로 이동하거나 차량을 가지러 가세요.',

&#x20; },

&#x20; {

&#x20;   no:    '04',

&#x20;   emoji: '✅',

&#x20;   title: '짐 찾기',

&#x20;   desc:  '돌아와서 다시 QR 코드 또는 예약번호를 보여주면 짐을 돌려드립니다.',

&#x20; },

]



// ── 부스 위치 데이터 ──────────────────────────────────────────────



const BOOTHS = \[

&#x20; {

&#x20;   terminal:  'T1',

&#x20;   label:     '제1여객터미널',

&#x20;   location:  '1층 출국장 입구 (3번 게이트 옆)',

&#x20;   hours:     '04:00 – 23:00',

&#x20;   emoji:     '✈️',

&#x20; },

&#x20; {

&#x20;   terminal:  'T2',

&#x20;   label:     '제2여객터미널',

&#x20;   location:  '1층 출국장 입구 (2번 게이트 옆)',

&#x20;   hours:     '05:00 – 23:00',

&#x20;   emoji:     '✈️',

&#x20; },

]



// ── FAQ 데이터 ───────────────────────────────────────────────────



const FAQS = \[

&#x20; {

&#x20;   q: '예약 없이 현장에서 바로 맡길 수 있나요?',

&#x20;   a: '아니요, 반드시 온라인 예약을 완료한 후 QR 코드 또는 예약번호로 접수해야 합니다. 현장 즉석 예약은 모바일웹으로 가능합니다.',

&#x20; },

&#x20; {

&#x20;   q: '짐은 몇 개까지 맡길 수 있나요?',

&#x20;   a: '예약 1건당 최대 2개까지 가능합니다. 3개 이상이라면 예약을 추가로 생성해 주세요.',

&#x20; },

&#x20; {

&#x20;   q: '요금은 얼마인가요?',

&#x20;   a: '예약 1건당 5,000원이며 짐 개수와 무관하게 동일합니다.',

&#x20; },

&#x20; {

&#x20;   q: '취소하면 환불이 되나요?',

&#x20;   a: '이용일 전날 자정(00:00)까지 취소하면 100% 환불됩니다. 이용일 당일에는 취소가 불가합니다.',

&#x20; },

&#x20; {

&#x20;   q: '얼마나 오래 맡길 수 있나요?',

&#x20;   a: '예약하신 당일 운영 시간 내에 짐을 찾아가셔야 합니다. 운영 시간이 종료되면 보관이 어렵습니다.',

&#x20; },

&#x20; {

&#x20;   q: '분실이나 파손은 어떻게 처리되나요?',

&#x20;   a: '귀중품, 파손되기 쉬운 물품은 보관을 삼가 주세요. 부스 직원과 협의 후 진행하시기 바랍니다.',

&#x20; },

]



// ── FAQ 아코디언 아이템 ───────────────────────────────────────────



'use client'



import { useState } from 'react'



function FaqItem({ q, a }: { q: string; a: string }) {

&#x20; const \[open, setOpen] = useState(false)



&#x20; return (

&#x20;   <div className="border-b border-gray-100 last:border-0">

&#x20;     <button

&#x20;       onClick={() => setOpen((v) => !v)}

&#x20;       className="flex w-full items-start justify-between gap-3 px-5 py-4 text-left"

&#x20;       aria-expanded={open}

&#x20;     >

&#x20;       <span className="text-sm font-medium text-gray-900">{q}</span>

&#x20;       <svg

&#x20;         className={`mt-0.5 h-4 w-4 shrink-0 text-gray-400 transition-transform duration-200 ${open ? 'rotate-180' : ''}`}

&#x20;         fill="none"

&#x20;         viewBox="0 0 24 24"

&#x20;         strokeWidth={2.5}

&#x20;         stroke="currentColor"

&#x20;       >

&#x20;         <path strokeLinecap="round" strokeLinejoin="round" d="M19.5 8.25l-7.5 7.5-7.5-7.5" />

&#x20;       </svg>

&#x20;     </button>

&#x20;     {open \&\& (

&#x20;       <div className="px-5 pb-4">

&#x20;         <p className="text-sm leading-relaxed text-gray-500">{a}</p>

&#x20;       </div>

&#x20;     )}

&#x20;   </div>

&#x20; )

}



// ── 메인 페이지 ──────────────────────────────────────────────────



export default function GuidePage() {

&#x20; return (

&#x20;   <div className="mx-auto min-h-svh max-w-\[430px] bg-gray-50">



&#x20;     {/\* GNB \*/}

&#x20;     <header className="flex h-14 items-center gap-3 border-b border-gray-100 bg-white px-4">

&#x20;       <Link

&#x20;         href="/"

&#x20;         aria-label="홈으로"

&#x20;         className="-ml-1 rounded-lg p-1.5 text-gray-500 hover:bg-gray-100"

&#x20;       >

&#x20;         <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;           <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;         </svg>

&#x20;       </Link>

&#x20;       <h1 className="text-base font-semibold text-gray-900">이용 안내</h1>

&#x20;     </header>



&#x20;     <div className="space-y-6 pb-12 pt-6">



&#x20;       {/\* ── 1. 서비스 이용 순서 ──────────────────────── \*/}

&#x20;       <section className="px-4">

&#x20;         <h2 className="mb-4 text-base font-bold text-gray-900">서비스 이용 방법</h2>

&#x20;         <div className="overflow-hidden rounded-2xl bg-white shadow-sm">

&#x20;           {STEPS.map((s, idx) => (

&#x20;             <div

&#x20;               key={s.no}

&#x20;               className={`

&#x20;                 flex items-start gap-4 px-5 py-4

&#x20;                 ${idx < STEPS.length - 1 ? 'border-b border-gray-50' : ''}

&#x20;               `}

&#x20;             >

&#x20;               {/\* 아이콘 \*/}

&#x20;               <div className="flex h-11 w-11 shrink-0 flex-col items-center justify-center rounded-2xl bg-blue-50">

&#x20;                 <span className="text-xl leading-none">{s.emoji}</span>

&#x20;               </div>

&#x20;               <div className="flex-1 pt-0.5">

&#x20;                 <div className="flex items-center gap-2">

&#x20;                   <span className="text-xs font-bold text-blue-500">{s.no}</span>

&#x20;                   <span className="text-sm font-semibold text-gray-900">{s.title}</span>

&#x20;                 </div>

&#x20;                 <p className="mt-1 text-xs leading-relaxed text-gray-500">{s.desc}</p>

&#x20;               </div>

&#x20;             </div>

&#x20;           ))}

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* ── 2. 부스 위치 안내 ────────────────────────── \*/}

&#x20;       <section className="px-4">

&#x20;         <h2 className="mb-4 text-base font-bold text-gray-900">부스 위치</h2>

&#x20;         <div className="space-y-3">

&#x20;           {BOOTHS.map((booth) => (

&#x20;             <div key={booth.terminal} className="overflow-hidden rounded-2xl bg-white shadow-sm">

&#x20;               {/\* 터미널 헤더 \*/}

&#x20;               <div className="flex items-center gap-3 bg-blue-600 px-5 py-3">

&#x20;                 <span className="text-xl">{booth.emoji}</span>

&#x20;                 <div>

&#x20;                   <span className="text-xs font-bold text-blue-200">{booth.terminal}</span>

&#x20;                   <p className="text-sm font-bold text-white">{booth.label}</p>

&#x20;                 </div>

&#x20;               </div>

&#x20;               {/\* 위치·운영 시간 \*/}

&#x20;               <div className="px-5 py-4">

&#x20;                 <div className="flex items-start gap-2 text-sm text-gray-700">

&#x20;                   <svg className="mt-0.5 h-4 w-4 shrink-0 text-gray-400" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;                     <path strokeLinecap="round" strokeLinejoin="round" d="M15 10.5a3 3 0 11-6 0 3 3 0 016 0z" />

&#x20;                     <path strokeLinecap="round" strokeLinejoin="round" d="M19.5 10.5c0 7.142-7.5 11.25-7.5 11.25S4.5 17.642 4.5 10.5a7.5 7.5 0 1115 0z" />

&#x20;                   </svg>

&#x20;                   <span>{booth.location}</span>

&#x20;                 </div>

&#x20;                 <div className="mt-2 flex items-center gap-2 text-sm text-gray-700">

&#x20;                   <svg className="h-4 w-4 shrink-0 text-gray-400" fill="none" viewBox="0 0 24 24" strokeWidth={1.8} stroke="currentColor">

&#x20;                     <path strokeLinecap="round" strokeLinejoin="round" d="M12 6v6h4.5m4.5 0a9 9 0 11-18 0 9 9 0 0118 0z" />

&#x20;                   </svg>

&#x20;                   <span>운영 시간 {booth.hours}</span>

&#x20;                 </div>

&#x20;                 {/\* 지도 이미지 placeholder \*/}

&#x20;                 <div className="mt-4 flex h-32 items-center justify-center rounded-xl bg-gray-100">

&#x20;                   <p className="text-xs text-gray-400">부스 위치 이미지</p>

&#x20;                 </div>

&#x20;               </div>

&#x20;             </div>

&#x20;           ))}

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* ── 3. FAQ ───────────────────────────────────── \*/}

&#x20;       <section className="px-4">

&#x20;         <h2 className="mb-4 text-base font-bold text-gray-900">자주 묻는 질문</h2>

&#x20;         <div className="overflow-hidden rounded-2xl bg-white shadow-sm">

&#x20;           {FAQS.map((faq, idx) => (

&#x20;             <FaqItem key={idx} q={faq.q} a={faq.a} />

&#x20;           ))}

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* ── 예약하기 CTA ─────────────────────────────── \*/}

&#x20;       <div className="px-4">

&#x20;         <Link

&#x20;           href="/reserve/step1"

&#x20;           className="

&#x20;             flex w-full items-center justify-center gap-2

&#x20;             rounded-2xl bg-blue-600 py-4

&#x20;             text-base font-bold text-white

&#x20;             shadow-lg shadow-blue-600/20 active:bg-blue-700

&#x20;           "

&#x20;         >

&#x20;           <span>🧳</span>

&#x20;           <span>지금 예약하기</span>

&#x20;         </Link>

&#x20;       </div>



&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 설계 노트



\### 취소 케이스 분기 — `CancelSection` 컴포넌트



서버에서 내려온 `is\_cancellable`을 그대로 사용한다. 클라이언트에서 날짜를 재계산하지 않는 이유:

\- 클라이언트 시계와 서버 KST 기준이 다를 수 있음

\- `is\_cancellable`은 API Route에서 `isCancellable(use\_date)` 헬퍼로 계산 후 반환됨



```

status === 'RESERVED' \&\& is\_cancellable === true  → Case 1: 취소 버튼 활성

status === 'RESERVED' \&\& is\_cancellable === false → Case 2: 당일 취소 불가 안내

status === 'STORED'                               → Case 3: 접수 후 취소 불가 안내

status === 'RETURNED' || 'CANCELLED'              → Case 4: 아무것도 없음

```



\### 취소 API 오류 처리 — 모달 내 인라인 오류



취소 실패 시 모달을 닫지 않고 \*\*모달 내부에 에러 메시지를 인라인으로 표시\*\*한다.

이유: 모달을 닫으면 사용자가 에러를 읽기 전에 사라지기 때문.



`CANCEL\_NOT\_ALLOWED` 오류는 예외 — 이용일이 지나 취소 창이 뜬 상태에서 서버 시간이 넘어간 케이스다. 이 경우 에러 메시지를 보여주고 2초 후 모달을 닫고 데이터를 재조회해 `is\_cancellable`이 `false`로 업데이트된 화면을 보여준다.



\### 이력 타임라인 — 마지막 항목 강조



```typescript

const isLast = idx === logs.length - 1

// → 마지막 도트를 blue-500으로, 나머지는 gray-300으로

```



가장 최근 상태가 시각적으로 눈에 띄어 현재 진행 단계를 빠르게 파악할 수 있다.



\### FAQ — 서버 컴포넌트 + 클라이언트 아코디언



`GuidePage`는 서버 컴포넌트로 두고, 아코디언 인터랙션이 필요한 `FaqItem`만 `'use client'`로 분리했다. Next.js 14 App Router에서 서버/클라이언트 컴포넌트를 혼합하는 표준 패턴이다.



\### QR 표시 조건 — RESERVED / STORED만



| 상태 | QR 표시 |

|------|--------|

| RESERVED | ✅ (접수 전 — 부스에서 스캔 필요) |

| STORED | ✅ (보관 중 — 반환 시 스캔 필요) |

| RETURNED | ❌ (완료 — 더 이상 필요 없음) |

| CANCELLED | ❌ (취소 — 무효) |



\---



\## Session 11 체크리스트



| 파일 | 구현 항목 | 상태 |

|------|----------|------|

| `StatusBadge.tsx` | 4가지 상태 색상 + 도트 + 크기 변형 | ✅ |

| `my-reservation/page.tsx` | 대문자 변환 + 형식 검증 + API 조회 + 오류 인라인 | ✅ |

| `\[reservationNo]/page.tsx` | 스켈레톤 / 상태 배지 / QR (RESERVED·STORED) / 복사 토스트 / 예약·결제 정보 / 타임라인 / 취소 케이스 4가지 / 취소 모달 + 인라인 오류 | ✅ |

| `guide/page.tsx` | 이용 순서 4단계 / 부스 위치 T1·T2 / FAQ 아코디언 / 예약 CTA | ✅ |



\---



\## 전체 FO 구현 완료 현황



| 세션 | 파일 | 상태 |

|------|------|------|

| Session 10 | 홈 + StepBar + STEP 1\~5 + QR (9개) | ✅ |

| Session 11 | 예약 조회 + 예약 상세 + StatusBadge + 이용 안내 (4개) | ✅ |



FO 전체 13개 파일 구현 완료. 이제 개발 착수 준비가 됐어요.



\---



\*다음 세션: Session 12 — BO 나머지 화면 구현 (접수/반환/보관현황/예약관리/매출)\*

