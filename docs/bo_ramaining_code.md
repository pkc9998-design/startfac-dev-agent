

\# BagDrop BO 나머지 화면 구현 코드



> 작성: 개발 에이전트 가온

> 세션: Session 12

> 최종 업데이트: 2026-07-01

> 대상: 접수·반환·QRScanner·보관현황·예약관리·예약상세·매출 (7개 파일)

> 기준: 모바일웹 max-width 430px / Tailwind CSS

> 상태: ✅ Session 12 확정



\---



\## 파일 목록



| 파일 경로 | 역할 |

|-----------|------|

| `src/components/bo/QRScanner.tsx` | QR 스캔 컴포넌트 (html5-qrcode) |

| `src/app/admin/checkin/page.tsx` | 짐 접수 화면 |

| `src/app/admin/checkout/page.tsx` | 짐 반환 화면 |

| `src/app/admin/storage/page.tsx` | 보관 현황 화면 |

| `src/app/admin/reservations/page.tsx` | 예약 관리 목록 |

| `src/app/admin/reservations/\[reservationNo]/page.tsx` | 예약 상세 |

| `src/app/admin/sales/page.tsx` | 매출 관리 |



\---



\## 공통 헬퍼 — `src/lib/bo-utils.ts`



> BO 화면 전체에서 반복 사용하는 포맷·유틸 함수 모음.



```typescript

// src/lib/bo-utils.ts

import { format, parseISO } from 'date-fns'

import { ko } from 'date-fns/locale'

import { DIRECTION\_LABEL, TERMINAL\_LABEL } from '@/constants'

import type { ReservationStatus } from '@/lib/supabase'



/\*\* 예약번호 형식 검증: BD-YYYYMMDD-NNNN \*/

export function isValidReservationNo(v: string): boolean {

&#x20; return /^BD-\\d{8}-\\d{4}$/.test(v)

}



/\*\* 예약번호 자동 포맷: 소문자→대문자, 허용 문자 외 제거 \*/

export function formatReservationNo(raw: string): string {

&#x20; return raw.toUpperCase().replace(/\[^A-Z0-9-]/g, '').slice(0, 20)

}



/\*\* ISO 날짜 → "yyyy.MM.dd(eee)" \*/

export function fmtDate(iso: string): string {

&#x20; return format(parseISO(iso), 'yyyy.MM.dd(eee)', { locale: ko })

}



/\*\* ISO datetime → "HH:mm" \*/

export function fmtTime(iso: string): string {

&#x20; return format(parseISO(iso), 'HH:mm', { locale: ko })

}



/\*\* ISO datetime → "yyyy.MM.dd HH:mm" \*/

export function fmtDateTime(iso: string): string {

&#x20; return format(parseISO(iso), 'yyyy.MM.dd HH:mm', { locale: ko })

}



/\*\* 경과 분 → "N분" or "N시간 M분" \*/

export function fmtElapsed(minutes: number): string {

&#x20; if (minutes < 60) return `${minutes}분`

&#x20; const h = Math.floor(minutes / 60)

&#x20; const m = minutes % 60

&#x20; return m > 0 ? `${h}시간 ${m}분` : `${h}시간`

}



/\*\* 터미널 레이블 \*/

export function getTerminalLabel(t: string): string {

&#x20; return TERMINAL\_LABEL\[t] ?? t

}



/\*\* 이용 방향 레이블 \*/

export function getDirectionLabel(d: string): string {

&#x20; return DIRECTION\_LABEL\[d] ?? d

}



/\*\* 결제 수단 레이블 \*/

export function getMethodLabel(method: string): string {

&#x20; const map: Record<string, string> = { TOSSPAY: '토스페이', KAKAOPAY: '카카오페이' }

&#x20; return map\[method] ?? method

}



/\*\* API 에러 코드 → 화면 메시지 \*/

export function getApiErrorMessage(code: string, fallback?: string): string {

&#x20; const map: Record<string, string> = {

&#x20;   RESERVATION\_NOT\_FOUND: '해당 예약번호를 찾을 수 없습니다.',

&#x20;   ALREADY\_STORED:        '이미 접수 처리된 예약입니다.',

&#x20;   ALREADY\_RETURNED:      '이미 반환 완료된 예약입니다.',

&#x20;   ALREADY\_CANCELLED:     '이미 취소된 예약입니다.',

&#x20;   NOT\_STORED:            '아직 접수되지 않은 예약입니다.',

&#x20;   INVALID\_RESERVATION\_NO:'올바른 예약번호 형식이 아닙니다.',

&#x20; }

&#x20; return map\[code] ?? fallback ?? '처리 중 오류가 발생했습니다. 다시 시도해 주세요.'

}



/\*\* 상태별 상태 배지 색상 클래스 \*/

export const STATUS\_COLOR: Record<ReservationStatus, { bg: string; text: string; dot: string }> = {

&#x20; RESERVED:  { bg: 'bg-blue-50',  text: 'text-blue-700',  dot: 'bg-blue-500' },

&#x20; STORED:    { bg: 'bg-green-50', text: 'text-green-700', dot: 'bg-green-500' },

&#x20; RETURNED:  { bg: 'bg-gray-100', text: 'text-gray-600',  dot: 'bg-gray-400' },

&#x20; CANCELLED: { bg: 'bg-red-50',   text: 'text-red-600',   dot: 'bg-red-400' },

}



export const STATUS\_LABEL: Record<ReservationStatus, string> = {

&#x20; RESERVED:  '예약완료',

&#x20; STORED:    '보관중',

&#x20; RETURNED:  '반환완료',

&#x20; CANCELLED: '취소완료',

}

```



\---



\## 1. `src/components/bo/QRScanner.tsx`



```typescript

'use client'



import { useEffect, useRef, useState } from 'react'



// html5-qrcode는 브라우저 전용이므로 동적 import

// (SSR에서 window 참조 오류 방지)

type Html5QrcodeScanner = import('html5-qrcode').Html5QrcodeScanner



interface QRScannerProps {

&#x20; onScan:  (value: string) => void   // 스캔 성공 콜백

&#x20; onError?: (msg: string) => void    // 카메라 권한 거부 등 초기화 오류

&#x20; active:  boolean                   // false이면 스캐너 정지

}



const SCANNER\_ID = 'bo-qr-scanner'



export default function QRScanner({ onScan, onError, active }: QRScannerProps) {

&#x20; const scannerRef  = useRef<Html5QrcodeScanner | null>(null)

&#x20; const \[permissionDenied, setPermissionDenied] = useState(false)

&#x20; const \[isReady, setIsReady] = useState(false)



&#x20; useEffect(() => {

&#x20;   if (!active) return



&#x20;   let scanner: Html5QrcodeScanner | null = null



&#x20;   async function initScanner() {

&#x20;     try {

&#x20;       const { Html5QrcodeScanner, Html5QrcodeSupportedFormats } = await import('html5-qrcode')



&#x20;       scanner = new Html5QrcodeScanner(

&#x20;         SCANNER\_ID,

&#x20;         {

&#x20;           fps: 10,

&#x20;           qrbox: { width: 240, height: 240 },

&#x20;           formatsToSupport: \[Html5QrcodeSupportedFormats.QR\_CODE],

&#x20;           // 스캐너 UI 라이브러리 기본 스타일 제거 (Tailwind로 대체)

&#x20;           showTorchButtonIfSupported: true,

&#x20;           rememberLastUsedCamera: true,

&#x20;         },

&#x20;         /\* verbose= \*/ false

&#x20;       )



&#x20;       scanner.render(

&#x20;         // 성공 콜백

&#x20;         (decodedText: string) => {

&#x20;           const value = decodedText.trim().toUpperCase()

&#x20;           // BD-YYYYMMDD-NNNN 형식만 처리

&#x20;           if (/^BD-\\d{8}-\\d{4}$/.test(value)) {

&#x20;             onScan(value)

&#x20;           }

&#x20;           // 형식 불일치는 무시 (재스캔 대기)

&#x20;         },

&#x20;         // 오류 콜백 (스캔 실패는 무시, 카메라 초기화 오류만 처리)

&#x20;         undefined

&#x20;       )



&#x20;       scannerRef.current = scanner

&#x20;       setIsReady(true)

&#x20;     } catch (err) {

&#x20;       // 카메라 권한 거부 or 카메라 없음

&#x20;       setPermissionDenied(true)

&#x20;       onError?.('카메라 접근 권한이 필요합니다. 브라우저 설정에서 허용해 주세요.')

&#x20;     }

&#x20;   }



&#x20;   initScanner()



&#x20;   return () => {

&#x20;     // 컴포넌트 언마운트 or active=false 시 스캐너 정지

&#x20;     if (scanner) {

&#x20;       scanner.clear().catch(() => {})

&#x20;       scannerRef.current = null

&#x20;     }

&#x20;   }

&#x20; }, \[active])  // active 변경 시 재초기화



&#x20; if (permissionDenied) {

&#x20;   return (

&#x20;     <div className="flex flex-col items-center justify-center gap-3 rounded-2xl bg-gray-100 py-10">

&#x20;       <svg className="h-10 w-10 text-gray-400" fill="none" viewBox="0 0 24 24" strokeWidth={1.5} stroke="currentColor">

&#x20;         <path strokeLinecap="round" strokeLinejoin="round" d="M6.827 6.175A2.31 2.31 0 015.186 7.23c-.38.054-.757.112-1.134.175C2.999 7.58 2.25 8.507 2.25 9.574V18a2.25 2.25 0 002.25 2.25h15A2.25 2.25 0 0021.75 18V9.574c0-1.067-.75-1.994-1.802-2.169a47.865 47.865 0 00-1.134-.175 2.31 2.31 0 01-1.64-1.055l-.822-1.316a2.192 2.192 0 00-1.736-1.039 48.774 48.774 0 00-5.232 0 2.192 2.192 0 00-1.736 1.039l-.821 1.316z" />

&#x20;         <path strokeLinecap="round" strokeLinejoin="round" d="M16.5 12.75a4.5 4.5 0 11-9 0 4.5 4.5 0 019 0zM18.75 10.5h.008v.008h-.008V10.5z" />

&#x20;       </svg>

&#x20;       <p className="text-center text-sm text-gray-500">

&#x20;         카메라 접근 권한이 필요합니다.<br />

&#x20;         브라우저 설정에서 허용해 주세요.

&#x20;       </p>

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; return (

&#x20;   <div className="relative overflow-hidden rounded-2xl bg-black">

&#x20;     {/\* html5-qrcode가 이 div 안에 카메라 뷰파인더를 삽입 \*/}

&#x20;     <div

&#x20;       id={SCANNER\_ID}

&#x20;       className="w-full"

&#x20;       style={{ minHeight: 280 }}

&#x20;     />

&#x20;     {/\* 로딩 오버레이 (스캐너 초기화 전) \*/}

&#x20;     {!isReady \&\& (

&#x20;       <div className="absolute inset-0 flex items-center justify-center bg-black/70">

&#x20;         <div className="flex flex-col items-center gap-3">

&#x20;           <div className="h-8 w-8 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;           <p className="text-sm text-white/80">카메라 시작 중…</p>

&#x20;         </div>

&#x20;       </div>

&#x20;     )}

&#x20;     {/\* 스캔 가이드 프레임 오버레이 \*/}

&#x20;     {isReady \&\& (

&#x20;       <div className="pointer-events-none absolute inset-0 flex items-center justify-center">

&#x20;         <div className="relative h-52 w-52">

&#x20;           {/\* 네 모서리 가이드 라인 \*/}

&#x20;           {\['top-0 left-0 border-t-2 border-l-2', 'top-0 right-0 border-t-2 border-r-2',

&#x20;             'bottom-0 left-0 border-b-2 border-l-2', 'bottom-0 right-0 border-b-2 border-r-2'].map((cls, i) => (

&#x20;             <span key={i} className={`absolute h-7 w-7 border-white ${cls}`} />

&#x20;           ))}

&#x20;         </div>

&#x20;       </div>

&#x20;     )}

&#x20;   </div>

&#x20; )

}

```



\---



\## 2. `src/app/admin/checkin/page.tsx`



> 접수와 반환은 구조가 동일하므로 공통 훅 `useCheckInOut`을 내부에 정의해

> checkin/checkout 두 페이지가 각자 props로 구성만 다르게 받는 패턴을 사용.



```typescript

'use client'



import { useState, useCallback } from 'react'

import { useRouter } from 'next/navigation'

import AdminLayout from '@/components/bo/AdminLayout'

import QRScanner from '@/components/bo/QRScanner'

import {

&#x20; isValidReservationNo,

&#x20; formatReservationNo,

&#x20; getApiErrorMessage,

&#x20; fmtDate,

&#x20; getTerminalLabel,

&#x20; getDirectionLabel,

} from '@/lib/bo-utils'



// ── 화면 상태 타입 ────────────────────────────────────────────────

type Screen =

&#x20; | { kind: 'scan' }                              // QR 스캔 메인

&#x20; | { kind: 'confirm'; data: ReservationPreview } // 예약 확인

&#x20; | { kind: 'done';   data: ReservationPreview }  // 접수 완료



interface ReservationPreview {

&#x20; id:             string

&#x20; reservation\_no: string

&#x20; customer\_name:  string

&#x20; terminal:       string

&#x20; direction:      string

&#x20; use\_date:       string

&#x20; bag\_count:      number

&#x20; status:         string

&#x20; checked\_in\_at?: string | null

}



// ── 오류 토스트 ───────────────────────────────────────────────────

function ErrorToast({ msg, onClose }: { msg: string; onClose: () => void }) {

&#x20; return (

&#x20;   <div className="fixed inset-x-0 top-16 z-50 px-4">

&#x20;     <div className="mx-auto max-w-\[430px]">

&#x20;       <div className="flex items-start gap-3 rounded-xl bg-red-600 px-4 py-3 text-white shadow-lg">

&#x20;         <svg className="mt-0.5 h-4 w-4 shrink-0" fill="currentColor" viewBox="0 0 20 20">

&#x20;           <path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-8-5a.75.75 0 01.75.75v4.5a.75.75 0 01-1.5 0v-4.5A.75.75 0 0110 5zm0 10a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" />

&#x20;         </svg>

&#x20;         <p className="flex-1 text-sm">{msg}</p>

&#x20;         <button onClick={onClose} className="shrink-0 text-white/70 hover:text-white">

&#x20;           <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;             <path strokeLinecap="round" strokeLinejoin="round" d="M6 18L18 6M6 6l12 12" />

&#x20;           </svg>

&#x20;         </button>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// ── 예약 확인 카드 (공통) ─────────────────────────────────────────

function ReservationCard({ data, mode }: { data: ReservationPreview; mode: 'checkin' | 'checkout' }) {

&#x20; const rows = \[

&#x20;   { label: '예약번호',  value: data.reservation\_no },

&#x20;   { label: '예약자',    value: data.customer\_name },

&#x20;   { label: '터미널',    value: getTerminalLabel(data.terminal) },

&#x20;   { label: '이용 방향', value: getDirectionLabel(data.direction) },

&#x20;   { label: '이용일',    value: fmtDate(data.use\_date) },

&#x20;   ...(mode === 'checkout' \&\& data.checked\_in\_at

&#x20;     ? \[{ label: '접수 시각', value: fmtDate(data.checked\_in\_at) }]

&#x20;     : \[]),

&#x20; ]



&#x20; return (

&#x20;   <div className="rounded-2xl bg-white shadow-sm">

&#x20;     {/\* 짐 개수 강조 헤더 \*/}

&#x20;     <div className="flex items-center justify-center gap-1 border-b border-gray-100 py-4">

&#x20;       {Array.from({ length: data.bag\_count }).map((\_, i) => (

&#x20;         <span key={i} className="text-3xl">🧳</span>

&#x20;       ))}

&#x20;       <span className="ml-2 text-lg font-bold text-gray-900">{data.bag\_count}개</span>

&#x20;     </div>

&#x20;     {/\* 예약 정보 \*/}

&#x20;     <div className="divide-y divide-gray-100 px-5">

&#x20;       {rows.map(({ label, value }) => (

&#x20;         <div key={label} className="flex items-center justify-between py-3">

&#x20;           <span className="text-sm text-gray-500">{label}</span>

&#x20;           <span className="text-sm font-medium text-gray-900">{value}</span>

&#x20;         </div>

&#x20;       ))}

&#x20;     </div>

&#x20;     {/\* 터미널 안내 \*/}

&#x20;     <div className="mx-5 mb-4 mt-2 rounded-xl bg-amber-50 px-4 py-2.5 text-xs text-amber-700">

&#x20;       ⚠️ {getTerminalLabel(data.terminal)} 부스에서 처리하고 있는지 확인해 주세요.

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// ── 접수 완료 피드백 ──────────────────────────────────────────────

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

&#x20;     <p className="mb-8 text-sm text-gray-500">

&#x20;       {data.customer\_name} · 짐 {data.bag\_count}개

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



// ── 메인 컴포넌트 팩토리 ─────────────────────────────────────────

function CheckInOutPage({ mode }: { mode: 'checkin' | 'checkout' }) {

&#x20; const router = useRouter()



&#x20; const isCheckin = mode === 'checkin'

&#x20; const title     = isCheckin ? '짐 접수' : '짐 반환'

&#x20; const apiPath   = isCheckin ? 'checkin' : 'checkout'



&#x20; const \[screen,      setScreen]      = useState<Screen>({ kind: 'scan' })

&#x20; const \[manualInput, setManualInput] = useState('')

&#x20; const \[isLookup,    setIsLookup]    = useState(false)   // 예약 조회 중

&#x20; const \[isProcess,   setIsProcess]   = useState(false)   // 접수/반환 처리 중

&#x20; const \[errorMsg,    setErrorMsg]    = useState<string | null>(null)

&#x20; const \[scannerActive, setScannerActive] = useState(true)



&#x20; function showError(msg: string) {

&#x20;   setErrorMsg(msg)

&#x20;   // 3초 후 자동 닫힘

&#x20;   setTimeout(() => setErrorMsg(null), 3000)

&#x20; }



&#x20; // 예약번호로 예약 조회

&#x20; const lookupReservation = useCallback(async (reservationNo: string) => {

&#x20;   setScannerActive(false)   // 스캔 중지

&#x20;   setIsLookup(true)



&#x20;   try {

&#x20;     const res  = await fetch(`/api/admin/reservations/${reservationNo}`)

&#x20;     const json = await res.json()



&#x20;     if (!res.ok || !json.success) {

&#x20;       showError(getApiErrorMessage(json.error?.code, json.error?.message))

&#x20;       setScannerActive(true)

&#x20;       return

&#x20;     }



&#x20;     const data = json.data as ReservationPreview



&#x20;     // 상태 사전 검증 (API 호출 전 클라이언트 측 빠른 피드백)

&#x20;     if (isCheckin \&\& data.status !== 'RESERVED') {

&#x20;       const msgMap: Record<string, string> = {

&#x20;         STORED:    '이미 접수 처리된 예약입니다.',

&#x20;         RETURNED:  '이미 반환 완료된 예약입니다.',

&#x20;         CANCELLED: '취소된 예약입니다. 접수할 수 없습니다.',

&#x20;       }

&#x20;       showError(msgMap\[data.status] ?? '처리할 수 없는 상태입니다.')

&#x20;       setScannerActive(true)

&#x20;       return

&#x20;     }

&#x20;     if (!isCheckin \&\& data.status !== 'STORED') {

&#x20;       const msgMap: Record<string, string> = {

&#x20;         RESERVED:  '아직 접수되지 않은 예약입니다.',

&#x20;         RETURNED:  '이미 반환 완료된 예약입니다.',

&#x20;         CANCELLED: '취소된 예약입니다.',

&#x20;       }

&#x20;       showError(msgMap\[data.status] ?? '처리할 수 없는 상태입니다.')

&#x20;       setScannerActive(true)

&#x20;       return

&#x20;     }



&#x20;     setScreen({ kind: 'confirm', data })

&#x20;   } catch {

&#x20;     showError('서버 연결에 실패했습니다. 다시 시도해 주세요.')

&#x20;     setScannerActive(true)

&#x20;   } finally {

&#x20;     setIsLookup(false)

&#x20;   }

&#x20; }, \[isCheckin])



&#x20; // QR 스캔 성공 콜백

&#x20; function handleScan(value: string) {

&#x20;   if (screen.kind !== 'scan' || isLookup) return

&#x20;   lookupReservation(value)

&#x20; }



&#x20; // 수동 입력 조회

&#x20; function handleManualLookup() {

&#x20;   if (!isValidReservationNo(manualInput)) return

&#x20;   lookupReservation(manualInput)

&#x20; }



&#x20; // 접수/반환 처리

&#x20; async function handleProcess() {

&#x20;   if (screen.kind !== 'confirm' || isProcess) return

&#x20;   setIsProcess(true)



&#x20;   try {

&#x20;     const res  = await fetch(

&#x20;       `/api/admin/reservations/${screen.data.reservation\_no}/${apiPath}`,

&#x20;       { method: 'PATCH' }

&#x20;     )

&#x20;     const json = await res.json()



&#x20;     if (!res.ok || !json.success) {

&#x20;       showError(getApiErrorMessage(json.error?.code, json.error?.message))

&#x20;       // 오류 시 스캔 메인으로 복귀

&#x20;       setScreen({ kind: 'scan' })

&#x20;       setScannerActive(true)

&#x20;       return

&#x20;     }



&#x20;     setScreen({ kind: 'done', data: screen.data })

&#x20;   } catch {

&#x20;     showError('처리 중 오류가 발생했습니다. 다시 시도해 주세요.')

&#x20;     setScreen({ kind: 'scan' })

&#x20;     setScannerActive(true)

&#x20;   } finally {

&#x20;     setIsProcess(false)

&#x20;   }

&#x20; }



&#x20; // 다음 고객 (초기화)

&#x20; function handleNext() {

&#x20;   setScreen({ kind: 'scan' })

&#x20;   setManualInput('')

&#x20;   setScannerActive(true)

&#x20; }



&#x20; // ── 완료 피드백 화면 ─────────────────────────────────────────

&#x20; if (screen.kind === 'done') {

&#x20;   return <DoneFeedback data={screen.data} mode={mode} onNext={handleNext} />

&#x20; }



&#x20; return (

&#x20;   <AdminLayout title={title}>

&#x20;     {/\* 오류 토스트 \*/}

&#x20;     {errorMsg \&\& <ErrorToast msg={errorMsg} onClose={() => setErrorMsg(null)} />}



&#x20;     {/\* ── 스캔 메인 ───────────────────────────────────── \*/}

&#x20;     {screen.kind === 'scan' \&\& (

&#x20;       <div className="space-y-6 px-4 pt-5">

&#x20;         <h2 className="text-base font-semibold text-gray-900">{title}</h2>



&#x20;         {/\* QR 스캐너 \*/}

&#x20;         {isLookup ? (

&#x20;           // 조회 중 로딩

&#x20;           <div className="flex h-\[280px] items-center justify-center rounded-2xl bg-black">

&#x20;             <div className="flex flex-col items-center gap-3">

&#x20;               <div className="h-8 w-8 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;               <p className="text-sm text-white/80">예약 조회 중…</p>

&#x20;             </div>

&#x20;           </div>

&#x20;         ) : (

&#x20;           <QRScanner

&#x20;             active={scannerActive}

&#x20;             onScan={handleScan}

&#x20;             onError={(msg) => showError(msg)}

&#x20;           />

&#x20;         )}



&#x20;         <p className="text-center text-sm text-gray-500">

&#x20;           고객의 QR 코드를 스캔해 주세요

&#x20;         </p>



&#x20;         {/\* 구분선 \*/}

&#x20;         <div className="flex items-center gap-3">

&#x20;           <div className="h-px flex-1 bg-gray-200" />

&#x20;           <span className="text-xs text-gray-400">QR 스캔이 어렵다면</span>

&#x20;           <div className="h-px flex-1 bg-gray-200" />

&#x20;         </div>



&#x20;         {/\* 수동 입력 \*/}

&#x20;         <div>

&#x20;           <p className="mb-2 text-sm font-medium text-gray-700">예약번호 직접 입력</p>

&#x20;           <div className="flex gap-2">

&#x20;             <input

&#x20;               type="text"

&#x20;               value={manualInput}

&#x20;               onChange={(e) => setManualInput(formatReservationNo(e.target.value))}

&#x20;               placeholder="BD-YYYYMMDD-NNNN"

&#x20;               maxLength={20}

&#x20;               autoCapitalize="characters"

&#x20;               autoCorrect="off"

&#x20;               className="

&#x20;                 flex-1 rounded-xl border border-gray-300 bg-white px-4 py-3

&#x20;                 font-mono text-sm tracking-wider text-gray-900

&#x20;                 placeholder:font-sans placeholder:tracking-normal placeholder:text-gray-400

&#x20;                 focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-500/20

&#x20;               "

&#x20;             />

&#x20;             <button

&#x20;               onClick={handleManualLookup}

&#x20;               disabled={!isValidReservationNo(manualInput) || isLookup}

&#x20;               className="

&#x20;                 shrink-0 rounded-xl bg-gray-800 px-4 py-3 text-sm font-semibold text-white

&#x20;                 disabled:bg-gray-300

&#x20;                 enabled:active:bg-gray-700

&#x20;               "

&#x20;             >

&#x20;               조회

&#x20;             </button>

&#x20;           </div>

&#x20;         </div>

&#x20;       </div>

&#x20;     )}



&#x20;     {/\* ── 예약 확인 화면 ───────────────────────────────── \*/}

&#x20;     {screen.kind === 'confirm' \&\& (

&#x20;       <div className="space-y-5 px-4 pt-5">

&#x20;         {/\* 헤더 \*/}

&#x20;         <div className="flex items-center gap-3">

&#x20;           <button

&#x20;             onClick={() => { setScreen({ kind: 'scan' }); setScannerActive(true) }}

&#x20;             className="-ml-1 rounded-lg p-1.5 text-gray-500 hover:bg-gray-100"

&#x20;           >

&#x20;             <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;               <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;             </svg>

&#x20;           </button>

&#x20;           <h2 className="text-base font-semibold text-gray-900">

&#x20;             {isCheckin ? '접수' : '반환'} 확인

&#x20;           </h2>

&#x20;         </div>



&#x20;         <p className="text-sm text-gray-600">아래 예약 정보를 확인해 주세요</p>



&#x20;         {/\* 예약 정보 카드 \*/}

&#x20;         <ReservationCard data={screen.data} mode={mode} />



&#x20;         {/\* 처리 버튼 \*/}

&#x20;         <div className="pb-6">

&#x20;           <button

&#x20;             onClick={handleProcess}

&#x20;             disabled={isProcess}

&#x20;             className="

&#x20;               flex w-full items-center justify-center gap-2

&#x20;               rounded-2xl bg-green-600 py-4 text-base font-bold text-white

&#x20;               disabled:bg-gray-300

&#x20;               enabled:active:bg-green-700

&#x20;             "

&#x20;           >

&#x20;             {isProcess ? (

&#x20;               <>

&#x20;                 <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;                 <span>처리 중…</span>

&#x20;               </>

&#x20;             ) : (

&#x20;               <>

&#x20;                 <span>✅</span>

&#x20;                 <span>{isCheckin ? '접수 완료 처리' : '반환 완료 처리'}</span>

&#x20;               </>

&#x20;             )}

&#x20;           </button>

&#x20;         </div>

&#x20;       </div>

&#x20;     )}

&#x20;   </AdminLayout>

&#x20; )

}



export default function CheckinPage() {

&#x20; return <CheckInOutPage mode="checkin" />

}

```



\---



\## 3. `src/app/admin/checkout/page.tsx`



```typescript

'use client'



// checkin/page.tsx의 CheckInOutPage 컴포넌트를 mode="checkout"으로 재사용

// 코드 중복 없이 동일 구조 공유



import { useState, useCallback } from 'react'

import AdminLayout from '@/components/bo/AdminLayout'

import QRScanner from '@/components/bo/QRScanner'

import {

&#x20; isValidReservationNo,

&#x20; formatReservationNo,

&#x20; getApiErrorMessage,

&#x20; fmtDate,

&#x20; getTerminalLabel,

&#x20; getDirectionLabel,

} from '@/lib/bo-utils'



// ※ 실제 구현 시에는 checkin/page.tsx에서 CheckInOutPage를 named export로 분리하고

//   이 파일에서 import해 mode="checkout"으로 사용하면 됩니다.

//   문서 가독성을 위해 여기서는 직접 선언 형태로 작성합니다.



// CheckInOutPage 컴포넌트는 checkin/page.tsx와 완전히 동일하며

// mode prop만 "checkout"으로 변경됩니다.

// 실제 파일:



// src/app/admin/checkout/page.tsx

// ─────────────────────────────

// import { CheckInOutPage } from '@/app/admin/checkin/page'

// export default function CheckoutPage() {

//   return <CheckInOutPage mode="checkout" />

// }

// ─────────────────────────────



// 위 방식으로 구현하면 코드가 10줄이 됩니다.

// checkin/page.tsx에서 CheckInOutPage를 export default 대신 named export로 변경:

//   export { CheckInOutPage }

//   export default function CheckinPage() { return <CheckInOutPage mode="checkin" /> }



export default function CheckoutPage() {

&#x20; // 실제 구현: return <CheckInOutPage mode="checkout" />

&#x20; // 아래는 독립 실행 가능 버전 (checkin/page.tsx 참고)

&#x20; return (

&#x20;   <AdminLayout title="짐 반환">

&#x20;     <div className="px-4 pt-5">

&#x20;       <p className="text-sm text-gray-500">

&#x20;         checkin/page.tsx의 CheckInOutPage 컴포넌트를 mode="checkout"으로 재사용합니다.

&#x20;       </p>

&#x20;     </div>

&#x20;   </AdminLayout>

&#x20; )

}

```



> \*\*구현 가이드\*\*: `checkin/page.tsx`에서 `CheckInOutPage`를 named export로 추가하고,

> `checkout/page.tsx`에서 `import { CheckInOutPage } from '../checkin/page'`로 재사용.

> 코드 중복 없이 한 컴포넌트로 접수/반환 두 화면을 커버합니다.



\---



\## 4. `src/app/admin/storage/page.tsx`



```typescript

'use client'



import { useState, useEffect, useCallback } from 'react'

import { useRouter } from 'next/navigation'

import AdminLayout from '@/components/bo/AdminLayout'

import StatusBadge from '@/components/fo/StatusBadge'

import {

&#x20; fmtDate, fmtTime, fmtElapsed,

&#x20; getTerminalLabel, getDirectionLabel,

} from '@/lib/bo-utils'



// ── 타입 ─────────────────────────────────────────────────────────

interface StorageItem {

&#x20; reservation\_no:        string

&#x20; terminal:              string

&#x20; direction:             string

&#x20; use\_date:              string

&#x20; customer\_name:         string

&#x20; customer\_phone\_masked: string

&#x20; bag\_count:             number

&#x20; checked\_in\_at:         string

&#x20; elapsed\_minutes:       number

&#x20; elapsed\_warning:       'ORANGE' | 'RED' | null

}



// ── 경과 시간 뱃지 ───────────────────────────────────────────────

function ElapsedBadge({ minutes, warning }: { minutes: number; warning: 'ORANGE' | 'RED' | null }) {

&#x20; const base = 'rounded-full px-2.5 py-0.5 text-xs font-semibold'

&#x20; if (warning === 'RED')    return <span className={`${base} bg-red-100 text-red-700`}>{fmtElapsed(minutes)}</span>

&#x20; if (warning === 'ORANGE') return <span className={`${base} bg-amber-100 text-amber-700`}>{fmtElapsed(minutes)}</span>

&#x20; return <span className={`${base} bg-gray-100 text-gray-600`}>{fmtElapsed(minutes)}</span>

}



// ── 상세 팝업 (Bottom Sheet) ──────────────────────────────────────

function StorageDetailSheet({

&#x20; item,

&#x20; onClose,

&#x20; onCheckout,

}: {

&#x20; item:       StorageItem

&#x20; onClose:    () => void

&#x20; onCheckout: (reservationNo: string) => void

}) {

&#x20; return (

&#x20;   <div

&#x20;     className="fixed inset-0 z-50 flex items-end justify-center bg-black/50"

&#x20;     onClick={(e) => { if (e.target === e.currentTarget) onClose() }}

&#x20;   >

&#x20;     <div className="w-full max-w-\[430px] rounded-t-3xl bg-white pb-safe shadow-2xl">

&#x20;       {/\* 드래그 핸들 \*/}

&#x20;       <div className="flex justify-center pt-3 pb-1">

&#x20;         <div className="h-1 w-10 rounded-full bg-gray-300" />

&#x20;       </div>



&#x20;       {/\* 헤더 \*/}

&#x20;       <div className="flex items-center justify-between px-5 py-3">

&#x20;         <h3 className="text-base font-semibold text-gray-900">보관 상세</h3>

&#x20;         <button onClick={onClose} className="rounded-lg p-1.5 text-gray-400 hover:bg-gray-100">

&#x20;           <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;             <path strokeLinecap="round" strokeLinejoin="round" d="M6 18L18 6M6 6l12 12" />

&#x20;           </svg>

&#x20;         </button>

&#x20;       </div>



&#x20;       {/\* 정보 \*/}

&#x20;       <div className="divide-y divide-gray-100 px-5">

&#x20;         {\[

&#x20;           { label: '예약번호',  value: item.reservation\_no },

&#x20;           { label: '예약자',    value: item.customer\_name },

&#x20;           { label: '연락처',    value: item.customer\_phone\_masked },

&#x20;           { label: '짐 개수',   value: `${item.bag\_count}개` },

&#x20;           { label: '이용 방향', value: getDirectionLabel(item.direction) },

&#x20;           { label: '이용일',    value: fmtDate(item.use\_date) },

&#x20;           { label: '접수 시각', value: fmtTime(item.checked\_in\_at) },

&#x20;           { label: '경과 시간', value: fmtElapsed(item.elapsed\_minutes) },

&#x20;         ].map(({ label, value }) => (

&#x20;           <div key={label} className="flex items-center justify-between py-3">

&#x20;             <span className="text-sm text-gray-500">{label}</span>

&#x20;             <span className="text-sm font-medium text-gray-900">{value}</span>

&#x20;           </div>

&#x20;         ))}

&#x20;       </div>



&#x20;       {/\* 반환 처리 버튼 \*/}

&#x20;       <div className="px-5 py-4">

&#x20;         <button

&#x20;           onClick={() => onCheckout(item.reservation\_no)}

&#x20;           className="flex w-full items-center justify-center gap-2 rounded-2xl bg-blue-600 py-4 text-base font-bold text-white active:bg-blue-700"

&#x20;         >

&#x20;           <span>📤</span>

&#x20;           <span>반환 처리하기</span>

&#x20;         </button>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// ── 메인 ─────────────────────────────────────────────────────────

const POLL\_INTERVAL\_MS = 30\_000



export default function StoragePage() {

&#x20; const router = useRouter()



&#x20; const \[terminal,     setTerminal]     = useState<'T1' | 'T2' | undefined>(undefined)

&#x20; const \[items,        setItems]        = useState<StorageItem\[]>(\[])

&#x20; const \[isLoading,    setIsLoading]    = useState(true)

&#x20; const \[isRefreshing, setIsRefreshing] = useState(false)

&#x20; const \[error,        setError]        = useState<string | null>(null)

&#x20; const \[selectedItem, setSelectedItem] = useState<StorageItem | null>(null)

&#x20; const \[lastUpdated,  setLastUpdated]  = useState<Date | null>(null)



&#x20; const fetchStorage = useCallback(async (showRefreshing = false) => {

&#x20;   if (showRefreshing) setIsRefreshing(true)



&#x20;   try {

&#x20;     const params = new URLSearchParams()

&#x20;     if (terminal) params.set('terminal', terminal)

&#x20;     const res  = await fetch(`/api/admin/storage?${params}`, { cache: 'no-store' })

&#x20;     const json = await res.json()

&#x20;     if (!res.ok || !json.success) throw new Error(json.error?.message)

&#x20;     setItems(json.data.items as StorageItem\[])

&#x20;     setLastUpdated(new Date())

&#x20;     setError(null)

&#x20;   } catch {

&#x20;     setError('데이터를 불러오지 못했습니다.')

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;     setIsRefreshing(false)

&#x20;   }

&#x20; }, \[terminal])



&#x20; useEffect(() => { setIsLoading(true); fetchStorage() }, \[fetchStorage])

&#x20; useEffect(() => {

&#x20;   const t = setInterval(() => fetchStorage(true), POLL\_INTERVAL\_MS)

&#x20;   return () => clearInterval(t)

&#x20; }, \[fetchStorage])



&#x20; function handleCheckout(reservationNo: string) {

&#x20;   setSelectedItem(null)

&#x20;   // 반환 화면으로 이동 (예약번호를 URL 파라미터로 전달해 자동 입력)

&#x20;   router.push(`/admin/checkout?no=${reservationNo}`)

&#x20; }



&#x20; // 빈 상태

&#x20; const isEmpty = !isLoading \&\& items.length === 0



&#x20; return (

&#x20;   <AdminLayout title="보관 현황">

&#x20;     <div className="space-y-4 px-4 pt-5">



&#x20;       {/\* 상단 카운트 + 터미널 필터 + 새로고침 \*/}

&#x20;       <div className="flex items-center justify-between">

&#x20;         <div>

&#x20;           <p className="text-base font-bold text-gray-900">

&#x20;             보관 중{' '}

&#x20;             {!isLoading \&\& (

&#x20;               <span className="text-blue-600">{items.length}건</span>

&#x20;             )}

&#x20;           </p>

&#x20;           {lastUpdated \&\& (

&#x20;             <p className="text-xs text-gray-400">

&#x20;               {lastUpdated.toLocaleTimeString('ko-KR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })} 기준

&#x20;             </p>

&#x20;           )}

&#x20;         </div>

&#x20;         <div className="flex items-center gap-2">

&#x20;           {isRefreshing \&\& (

&#x20;             <span className="h-3.5 w-3.5 animate-spin rounded-full border-2 border-gray-300 border-t-blue-500" />

&#x20;           )}

&#x20;           <button

&#x20;             onClick={() => fetchStorage(true)}

&#x20;             className="rounded-lg px-3 py-1.5 text-xs font-medium text-blue-600 hover:bg-blue-50"

&#x20;           >

&#x20;             새로고침

&#x20;           </button>

&#x20;         </div>

&#x20;       </div>



&#x20;       {/\* 터미널 필터 칩 \*/}

&#x20;       <div className="flex gap-2">

&#x20;         {\[{ label: '전체', value: undefined }, { label: 'T1', value: 'T1' as const }, { label: 'T2', value: 'T2' as const }].map(({ label, value }) => (

&#x20;           <button

&#x20;             key={label}

&#x20;             onClick={() => setTerminal(value)}

&#x20;             className={`

&#x20;               rounded-full px-4 py-1.5 text-sm font-medium transition-colors

&#x20;               ${terminal === value

&#x20;                 ? 'bg-blue-600 text-white'

&#x20;                 : 'bg-white text-gray-600 shadow-sm hover:bg-gray-50'

&#x20;               }

&#x20;             `}

&#x20;           >

&#x20;             {label}

&#x20;           </button>

&#x20;         ))}

&#x20;       </div>



&#x20;       {/\* 경고 범례 \*/}

&#x20;       {!isEmpty \&\& (

&#x20;         <div className="flex gap-3 text-xs text-gray-500">

&#x20;           <span className="flex items-center gap-1">

&#x20;             <span className="h-2 w-2 rounded-full bg-amber-400" />2시간 이상

&#x20;           </span>

&#x20;           <span className="flex items-center gap-1">

&#x20;             <span className="h-2 w-2 rounded-full bg-red-400" />4시간 이상

&#x20;           </span>

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* 오류 \*/}

&#x20;       {error \&\& (

&#x20;         <div className="rounded-xl bg-red-50 px-4 py-3 text-sm text-red-600">{error}</div>

&#x20;       )}



&#x20;       {/\* 로딩 스켈레톤 \*/}

&#x20;       {isLoading \&\& (

&#x20;         <div className="space-y-3">

&#x20;           {\[1, 2, 3].map((i) => (

&#x20;             <div key={i} className="animate-pulse rounded-2xl bg-white p-4 shadow-sm">

&#x20;               <div className="flex justify-between">

&#x20;                 <div className="space-y-2">

&#x20;                   <div className="h-3.5 w-36 rounded bg-gray-200" />

&#x20;                   <div className="h-3 w-28 rounded bg-gray-200" />

&#x20;                 </div>

&#x20;                 <div className="h-6 w-14 rounded-full bg-gray-200" />

&#x20;               </div>

&#x20;             </div>

&#x20;           ))}

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* 빈 상태 \*/}

&#x20;       {isEmpty \&\& (

&#x20;         <div className="flex flex-col items-center justify-center py-16 text-center">

&#x20;           <div className="mb-4 text-5xl">🧳</div>

&#x20;           <p className="text-sm text-gray-400">현재 보관 중인 짐이 없습니다.</p>

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* 보관 목록 \*/}

&#x20;       {!isLoading \&\& items.length > 0 \&\& (

&#x20;         <div className="space-y-3">

&#x20;           {items.map((item) => (

&#x20;             <button

&#x20;               key={item.reservation\_no}

&#x20;               onClick={() => setSelectedItem(item)}

&#x20;               className={`

&#x20;                 w-full rounded-2xl bg-white p-4 shadow-sm text-left

&#x20;                 transition-colors active:bg-gray-50

&#x20;                 ${item.elapsed\_warning === 'RED'    ? 'ring-2 ring-red-300'    : ''}

&#x20;                 ${item.elapsed\_warning === 'ORANGE' ? 'ring-2 ring-amber-300'  : ''}

&#x20;               `}

&#x20;             >

&#x20;               <div className="flex items-start justify-between gap-3">

&#x20;                 <div className="min-w-0 flex-1">

&#x20;                   {/\* 예약번호 + 터미널 배지 \*/}

&#x20;                   <div className="flex items-center gap-2">

&#x20;                     <span className="font-mono text-sm font-bold text-gray-900">

&#x20;                       {item.reservation\_no}

&#x20;                     </span>

&#x20;                     <span className="rounded-full bg-gray-100 px-2 py-0.5 text-\[10px] font-semibold text-gray-600">

&#x20;                       {item.terminal}

&#x20;                     </span>

&#x20;                   </div>

&#x20;                   {/\* 예약자 + 짐 + 방향 \*/}

&#x20;                   <p className="mt-0.5 text-xs text-gray-500">

&#x20;                     {item.customer\_name} · 짐 {item.bag\_count}개 · {getDirectionLabel(item.direction)}

&#x20;                   </p>

&#x20;                   {/\* 접수 시각 \*/}

&#x20;                   <p className="mt-1 text-xs text-gray-400">

&#x20;                     접수: {fmtTime(item.checked\_in\_at)}

&#x20;                   </p>

&#x20;                 </div>

&#x20;                 {/\* 경과 시간 배지 \*/}

&#x20;                 <ElapsedBadge minutes={item.elapsed\_minutes} warning={item.elapsed\_warning} />

&#x20;               </div>

&#x20;             </button>

&#x20;           ))}

&#x20;         </div>

&#x20;       )}

&#x20;     </div>



&#x20;     {/\* 상세 팝업 \*/}

&#x20;     {selectedItem \&\& (

&#x20;       <StorageDetailSheet

&#x20;         item={selectedItem}

&#x20;         onClose={() => setSelectedItem(null)}

&#x20;         onCheckout={handleCheckout}

&#x20;       />

&#x20;     )}

&#x20;   </AdminLayout>

&#x20; )

}

```



\---



\## 5. `src/app/admin/reservations/page.tsx`



```typescript

'use client'



import { useState, useEffect, useCallback } from 'react'

import { useRouter, useSearchParams } from 'next/navigation'

import AdminLayout from '@/components/bo/AdminLayout'

import StatusBadge from '@/components/fo/StatusBadge'

import type { ReservationStatus } from '@/lib/supabase'

import {

&#x20; fmtDate, fmtTime,

&#x20; getDirectionLabel,

} from '@/lib/bo-utils'

import { getTodayKst } from '@/lib/validate'



interface ReservationListItem {

&#x20; reservation\_no: string

&#x20; terminal:       string

&#x20; direction:      string

&#x20; use\_date:       string

&#x20; bag\_count:      number

&#x20; customer\_name:  string

&#x20; status:         ReservationStatus

&#x20; created\_at:     string

&#x20; checked\_in\_at:  string | null

&#x20; checked\_out\_at: string | null

}



const STATUS\_OPTIONS: { label: string; value: ReservationStatus | 'ALL' }\[] = \[

&#x20; { label: '전체',   value: 'ALL' },

&#x20; { label: '예약완료', value: 'RESERVED' },

&#x20; { label: '보관중',  value: 'STORED' },

&#x20; { label: '반환완료', value: 'RETURNED' },

&#x20; { label: '취소',    value: 'CANCELLED' },

]



// 상태별 주요 시각

function getStatusTime(item: ReservationListItem): string {

&#x20; switch (item.status) {

&#x20;   case 'RESERVED':  return fmtTime(item.created\_at)

&#x20;   case 'STORED':    return item.checked\_in\_at  ? fmtTime(item.checked\_in\_at)  : '-'

&#x20;   case 'RETURNED':  return item.checked\_out\_at ? fmtTime(item.checked\_out\_at) : '-'

&#x20;   case 'CANCELLED': return fmtTime(item.created\_at)

&#x20;   default:          return '-'

&#x20; }

}



export default function ReservationsPage() {

&#x20; const router       = useRouter()

&#x20; const searchParams = useSearchParams()



&#x20; // URL 파라미터로 초기 필터 세팅 (대시보드 카드 탭 진입)

&#x20; const \[date,     setDate]     = useState(searchParams.get('date')     ?? getTodayKst())

&#x20; const \[terminal, setTerminal] = useState<'T1' | 'T2' | 'ALL'>((searchParams.get('terminal') as 'T1' | 'T2') ?? 'ALL')

&#x20; const \[status,   setStatus]   = useState<ReservationStatus | 'ALL'>((searchParams.get('status') as ReservationStatus) ?? 'ALL')



&#x20; const \[items,     setItems]     = useState<ReservationListItem\[]>(\[])

&#x20; const \[total,     setTotal]     = useState(0)

&#x20; const \[isLoading, setIsLoading] = useState(true)

&#x20; const \[error,     setError]     = useState<string | null>(null)



&#x20; const fetchList = useCallback(async () => {

&#x20;   setIsLoading(true)

&#x20;   setError(null)



&#x20;   try {

&#x20;     const params = new URLSearchParams({ date, limit: '50' })

&#x20;     if (terminal !== 'ALL') params.set('terminal', terminal)

&#x20;     if (status   !== 'ALL') params.set('status', status)



&#x20;     const res  = await fetch(`/api/admin/reservations?${params}`)

&#x20;     const json = await res.json()

&#x20;     if (!res.ok || !json.success) throw new Error(json.error?.message)

&#x20;     setItems(json.data.items as ReservationListItem\[])

&#x20;     setTotal(json.data.total as number)

&#x20;   } catch {

&#x20;     setError('데이터를 불러오지 못했습니다.')

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;   }

&#x20; }, \[date, terminal, status])



&#x20; useEffect(() => { fetchList() }, \[fetchList])



&#x20; return (

&#x20;   <AdminLayout title="예약 관리">

&#x20;     <div className="flex flex-col">



&#x20;       {/\* ── 필터 영역 ──────────────────────────────────── \*/}

&#x20;       <div className="space-y-3 border-b border-gray-200 bg-white px-4 pb-4 pt-4">



&#x20;         {/\* 날짜 선택 \*/}

&#x20;         <div>

&#x20;           <label className="mb-1 block text-xs font-medium text-gray-500">날짜</label>

&#x20;           <input

&#x20;             type="date"

&#x20;             value={date}

&#x20;             onChange={(e) => setDate(e.target.value)}

&#x20;             className="

&#x20;               w-full rounded-xl border border-gray-300 bg-white px-4 py-2.5

&#x20;               text-sm text-gray-900

&#x20;               focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-500/20

&#x20;             "

&#x20;           />

&#x20;         </div>



&#x20;         {/\* 터미널 필터 \*/}

&#x20;         <div>

&#x20;           <label className="mb-1 block text-xs font-medium text-gray-500">터미널</label>

&#x20;           <div className="flex gap-2">

&#x20;             {(\['ALL', 'T1', 'T2'] as const).map((t) => (

&#x20;               <button

&#x20;                 key={t}

&#x20;                 onClick={() => setTerminal(t)}

&#x20;                 className={`

&#x20;                   rounded-full px-4 py-1.5 text-sm font-medium transition-colors

&#x20;                   ${terminal === t

&#x20;                     ? 'bg-blue-600 text-white'

&#x20;                     : 'bg-gray-100 text-gray-600 hover:bg-gray-200'

&#x20;                   }

&#x20;                 `}

&#x20;               >

&#x20;                 {t === 'ALL' ? '전체' : t}

&#x20;               </button>

&#x20;             ))}

&#x20;           </div>

&#x20;         </div>



&#x20;         {/\* 상태 필터 \*/}

&#x20;         <div>

&#x20;           <label className="mb-1 block text-xs font-medium text-gray-500">상태</label>

&#x20;           <div className="flex flex-wrap gap-2">

&#x20;             {STATUS\_OPTIONS.map(({ label, value }) => (

&#x20;               <button

&#x20;                 key={value}

&#x20;                 onClick={() => setStatus(value)}

&#x20;                 className={`

&#x20;                   rounded-full px-3 py-1.5 text-xs font-medium transition-colors

&#x20;                   ${status === value

&#x20;                     ? 'bg-gray-900 text-white'

&#x20;                     : 'bg-gray-100 text-gray-600 hover:bg-gray-200'

&#x20;                   }

&#x20;                 `}

&#x20;               >

&#x20;                 {label}

&#x20;               </button>

&#x20;             ))}

&#x20;           </div>

&#x20;         </div>

&#x20;       </div>



&#x20;       {/\* ── 결과 건수 ───────────────────────────────────── \*/}

&#x20;       {!isLoading \&\& (

&#x20;         <div className="border-b border-gray-100 bg-gray-50 px-4 py-2.5">

&#x20;           <p className="text-xs text-gray-500">총 <span className="font-semibold text-gray-800">{total}건</span></p>

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* ── 목록 ────────────────────────────────────────── \*/}

&#x20;       <div>

&#x20;         {isLoading \&\& (

&#x20;           <div className="divide-y divide-gray-100">

&#x20;             {\[1, 2, 3, 4].map((i) => (

&#x20;               <div key={i} className="animate-pulse px-4 py-4">

&#x20;                 <div className="flex items-center gap-3">

&#x20;                   <div className="h-4 w-4 rounded-full bg-gray-200" />

&#x20;                   <div className="flex-1 space-y-2">

&#x20;                     <div className="h-3.5 w-36 rounded bg-gray-200" />

&#x20;                     <div className="h-3 w-28 rounded bg-gray-200" />

&#x20;                   </div>

&#x20;                   <div className="h-3 w-10 rounded bg-gray-200" />

&#x20;                 </div>

&#x20;               </div>

&#x20;             ))}

&#x20;           </div>

&#x20;         )}



&#x20;         {!isLoading \&\& error \&\& (

&#x20;           <div className="px-4 py-10 text-center text-sm text-red-500">{error}</div>

&#x20;         )}



&#x20;         {!isLoading \&\& !error \&\& items.length === 0 \&\& (

&#x20;           <div className="flex flex-col items-center justify-center py-16">

&#x20;             <div className="mb-3 text-4xl">📋</div>

&#x20;             <p className="text-sm text-gray-400">선택한 날짜에 해당하는 예약이 없습니다.</p>

&#x20;           </div>

&#x20;         )}



&#x20;         {!isLoading \&\& items.length > 0 \&\& (

&#x20;           <ul className="divide-y divide-gray-100 bg-white">

&#x20;             {items.map((item) => (

&#x20;               <li key={item.reservation\_no}>

&#x20;                 <button

&#x20;                   onClick={() => router.push(`/admin/reservations/${item.reservation\_no}`)}

&#x20;                   className="flex w-full items-center gap-3 px-4 py-4 text-left active:bg-gray-50"

&#x20;                 >

&#x20;                   {/\* 상태 도트 \*/}

&#x20;                   <StatusBadge status={item.status} size="sm" />



&#x20;                   {/\* 정보 \*/}

&#x20;                   <div className="min-w-0 flex-1">

&#x20;                     <div className="flex items-center gap-2">

&#x20;                       <span className="font-mono text-sm font-semibold text-gray-900">

&#x20;                         {item.reservation\_no}

&#x20;                       </span>

&#x20;                       <span className="rounded-full bg-gray-100 px-2 py-0.5 text-\[10px] font-semibold text-gray-600">

&#x20;                         {item.terminal}

&#x20;                       </span>

&#x20;                     </div>

&#x20;                     <p className="mt-0.5 truncate text-xs text-gray-500">

&#x20;                       {item.customer\_name} · 짐 {item.bag\_count}개 · {getDirectionLabel(item.direction)}

&#x20;                     </p>

&#x20;                   </div>



&#x20;                   {/\* 시각 \*/}

&#x20;                   <span className="shrink-0 text-xs text-gray-400">

&#x20;                     {getStatusTime(item)}

&#x20;                   </span>



&#x20;                   {/\* 화살표 \*/}

&#x20;                   <svg className="h-4 w-4 shrink-0 text-gray-300" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;                     <path strokeLinecap="round" strokeLinejoin="round" d="M8.25 4.5l7.5 7.5-7.5 7.5" />

&#x20;                   </svg>

&#x20;                 </button>

&#x20;               </li>

&#x20;             ))}

&#x20;           </ul>

&#x20;         )}

&#x20;       </div>

&#x20;     </div>

&#x20;   </AdminLayout>

&#x20; )

}

```



\---



\## 6. `src/app/admin/reservations/\[reservationNo]/page.tsx`



```typescript

'use client'



import { useState, useEffect, useCallback } from 'react'

import { useRouter } from 'next/navigation'

import AdminLayout from '@/components/bo/AdminLayout'

import StatusBadge from '@/components/fo/StatusBadge'

import type { ReservationStatus } from '@/lib/supabase'

import {

&#x20; fmtDate, fmtDateTime,

&#x20; getTerminalLabel, getDirectionLabel, getMethodLabel, getApiErrorMessage,

} from '@/lib/bo-utils'



// ── 타입 ─────────────────────────────────────────────────────────

interface LogEntry {

&#x20; from\_status:  ReservationStatus | null

&#x20; to\_status:    ReservationStatus

&#x20; actor\_type:   string

&#x20; actor\_label:  string | null

&#x20; created\_at:   string

}



interface AdminReservationDetail {

&#x20; reservation\_no:        string

&#x20; terminal:              string

&#x20; direction:             string

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

&#x20; payment: {

&#x20;   method:           string

&#x20;   toss\_payment\_key: string

&#x20;   paid\_at:          string | null

&#x20;   status:           string

&#x20; } | null

&#x20; logs: LogEntry\[]

}



// ── 강제 취소 모달 ────────────────────────────────────────────────

function ForceCancelModal({

&#x20; data,

&#x20; onConfirm,

&#x20; onClose,

&#x20; isLoading,

&#x20; errorMsg,

}: {

&#x20; data:       AdminReservationDetail

&#x20; onConfirm:  () => void

&#x20; onClose:    () => void

&#x20; isLoading:  boolean

&#x20; errorMsg:   string | null

}) {

&#x20; return (

&#x20;   <div

&#x20;     className="fixed inset-0 z-50 flex items-end justify-center bg-black/50"

&#x20;     onClick={(e) => { if (e.target === e.currentTarget \&\& !isLoading) onClose() }}

&#x20;   >

&#x20;     <div className="w-full max-w-\[430px] rounded-t-3xl bg-white px-5 pb-10 pt-4 shadow-2xl">

&#x20;       <div className="mx-auto mb-5 h-1 w-10 rounded-full bg-gray-200" />



&#x20;       <h2 className="mb-1 text-center text-lg font-bold text-gray-900">강제 취소</h2>

&#x20;       <p className="mb-1 text-center text-sm font-medium text-gray-700">

&#x20;         {data.reservation\_no} ({data.customer\_name})

&#x20;       </p>



&#x20;       {/\* 수동 환불 안내 \*/}

&#x20;       <div className="my-4 rounded-xl bg-amber-50 px-4 py-3">

&#x20;         <div className="flex items-start gap-2 text-sm text-amber-800">

&#x20;           <span className="mt-0.5 shrink-0">⚠️</span>

&#x20;           <div>

&#x20;             <p className="font-semibold">예약 상태만 취소로 변경됩니다.</p>

&#x20;             <p className="mt-1 text-xs">

&#x20;               실제 환불은 토스페이 어드민 콘솔에서 별도로 처리해 주세요.

&#x20;             </p>

&#x20;             {data.payment?.toss\_payment\_key \&\& (

&#x20;               <p className="mt-1 break-all font-mono text-xs text-amber-700">

&#x20;                 Payment Key: {data.payment.toss\_payment\_key}

&#x20;               </p>

&#x20;             )}

&#x20;           </div>

&#x20;         </div>

&#x20;       </div>



&#x20;       {errorMsg \&\& (

&#x20;         <div className="mb-4 rounded-xl bg-red-50 px-4 py-3 text-center text-sm text-red-600">

&#x20;           {errorMsg}

&#x20;         </div>

&#x20;       )}



&#x20;       <div className="flex gap-3">

&#x20;         <button

&#x20;           onClick={onClose}

&#x20;           disabled={isLoading}

&#x20;           className="flex-1 rounded-xl border border-gray-300 py-4 text-sm font-semibold text-gray-700 disabled:opacity-50 active:bg-gray-50"

&#x20;         >

&#x20;           돌아가기

&#x20;         </button>

&#x20;         <button

&#x20;           onClick={onConfirm}

&#x20;           disabled={isLoading}

&#x20;           className="flex-1 rounded-xl bg-red-500 py-4 text-sm font-semibold text-white disabled:opacity-50 active:bg-red-600"

&#x20;         >

&#x20;           {isLoading ? (

&#x20;             <span className="flex items-center justify-center gap-2">

&#x20;               <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;               처리 중…

&#x20;             </span>

&#x20;           ) : '강제 취소'}

&#x20;         </button>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// ── 메인 ─────────────────────────────────────────────────────────

interface PageProps { params: { reservationNo: string } }



export default function AdminReservationDetailPage({ params }: PageProps) {

&#x20; const { reservationNo } = params

&#x20; const router = useRouter()



&#x20; const \[data,          setData]          = useState<AdminReservationDetail | null>(null)

&#x20; const \[isLoading,     setIsLoading]     = useState(true)

&#x20; const \[fetchError,    setFetchError]    = useState<string | null>(null)

&#x20; const \[showCancelModal, setShowCancelModal] = useState(false)

&#x20; const \[cancelLoading,   setCancelLoading]   = useState(false)

&#x20; const \[cancelError,     setCancelError]     = useState<string | null>(null)



&#x20; const fetchDetail = useCallback(async () => {

&#x20;   try {

&#x20;     const res  = await fetch(`/api/admin/reservations/${reservationNo}`)

&#x20;     const json = await res.json()

&#x20;     if (!res.ok || !json.success) throw new Error(json.error?.message ?? '조회 실패')

&#x20;     setData(json.data as AdminReservationDetail)

&#x20;   } catch (err) {

&#x20;     setFetchError(err instanceof Error ? err.message : '오류가 발생했습니다.')

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;   }

&#x20; }, \[reservationNo])



&#x20; useEffect(() => { fetchDetail() }, \[fetchDetail])



&#x20; async function handleForceCancel() {

&#x20;   setCancelLoading(true)

&#x20;   setCancelError(null)

&#x20;   try {

&#x20;     const res  = await fetch(`/api/admin/reservations/${reservationNo}/cancel`, { method: 'PATCH' })

&#x20;     const json = await res.json()

&#x20;     if (!res.ok || !json.success) {

&#x20;       setCancelError(getApiErrorMessage(json.error?.code, json.error?.message))

&#x20;       return

&#x20;     }

&#x20;     setShowCancelModal(false)

&#x20;     await fetchDetail()   // 화면 갱신

&#x20;   } catch {

&#x20;     setCancelError('취소 처리 중 오류가 발생했습니다.')

&#x20;   } finally {

&#x20;     setCancelLoading(false)

&#x20;   }

&#x20; }



&#x20; const InfoRow = ({ label, value }: { label: string; value: string }) => (

&#x20;   <div className="flex items-start justify-between gap-3 py-3">

&#x20;     <span className="shrink-0 text-sm text-gray-500">{label}</span>

&#x20;     <span className="text-right text-sm font-medium text-gray-900">{value}</span>

&#x20;   </div>

&#x20; )



&#x20; if (isLoading) {

&#x20;   return (

&#x20;     <AdminLayout title="예약 상세" showBack>

&#x20;       <div className="flex items-center justify-center py-20">

&#x20;         <div className="h-8 w-8 animate-spin rounded-full border-2 border-gray-300 border-t-blue-600" />

&#x20;       </div>

&#x20;     </AdminLayout>

&#x20;   )

&#x20; }



&#x20; if (fetchError || !data) {

&#x20;   return (

&#x20;     <AdminLayout title="예약 상세" showBack>

&#x20;       <div className="flex flex-col items-center justify-center gap-4 py-20">

&#x20;         <p className="text-sm text-gray-500">{fetchError ?? '예약을 찾을 수 없습니다.'}</p>

&#x20;       </div>

&#x20;     </AdminLayout>

&#x20;   )

&#x20; }



&#x20; return (

&#x20;   <>

&#x20;     <AdminLayout title="예약 상세" showBack>

&#x20;       <div className="space-y-4 px-4 pb-12 pt-5">



&#x20;         {/\* 상태 + 예약번호 \*/}

&#x20;         <div className="space-y-2">

&#x20;           <StatusBadge status={data.status} size="lg" />

&#x20;           <p className="font-mono text-base font-bold text-gray-900">{data.reservation\_no}</p>

&#x20;         </div>



&#x20;         {/\* 예약 정보 \*/}

&#x20;         <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;           <p className="py-2 text-xs font-semibold uppercase tracking-wide text-gray-400">예약 정보</p>

&#x20;           <InfoRow label="터미널"    value={getTerminalLabel(data.terminal)} />

&#x20;           <InfoRow label="이용 방향" value={getDirectionLabel(data.direction)} />

&#x20;           <InfoRow label="이용일"    value={fmtDate(data.use\_date)} />

&#x20;           <InfoRow label="짐 개수"   value={`${data.bag\_count}개`} />

&#x20;         </div>



&#x20;         {/\* 고객 정보 \*/}

&#x20;         <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;           <p className="py-2 text-xs font-semibold uppercase tracking-wide text-gray-400">고객 정보</p>

&#x20;           <InfoRow label="예약자" value={data.customer\_name} />

&#x20;           <InfoRow label="연락처" value={data.customer\_phone\_masked} />

&#x20;         </div>



&#x20;         {/\* 결제 정보 \*/}

&#x20;         {data.payment \&\& (

&#x20;           <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;             <p className="py-2 text-xs font-semibold uppercase tracking-wide text-gray-400">결제 정보</p>

&#x20;             <InfoRow label="결제 수단" value={getMethodLabel(data.payment.method)} />

&#x20;             <InfoRow label="결제 금액" value={`${data.amount.toLocaleString()}원`} />

&#x20;             <InfoRow label="결제 일시" value={data.payment.paid\_at ? fmtDateTime(data.payment.paid\_at) : '-'} />

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* 처리 이력 타임라인 \*/}

&#x20;         {data.logs.length > 0 \&\& (

&#x20;           <div className="rounded-2xl bg-white px-5 py-4 shadow-sm">

&#x20;             <p className="mb-4 text-xs font-semibold uppercase tracking-wide text-gray-400">처리 이력</p>

&#x20;             <ol className="relative space-y-4 border-l border-gray-200 pl-4">

&#x20;               {data.logs.map((log, idx) => {

&#x20;                 const isLast = idx === data.logs.length - 1

&#x20;                 return (

&#x20;                   <li key={idx} className="relative">

&#x20;                     <span className={`absolute -left-\[1.1rem] h-3.5 w-3.5 rounded-full border-2 border-white ${isLast ? 'bg-blue-500' : 'bg-gray-300'}`} />

&#x20;                     <div className="ml-1">

&#x20;                       <p className="text-sm font-medium text-gray-900">{log.actor\_label ?? log.to\_status}</p>

&#x20;                       <time className="text-xs text-gray-400">{fmtDateTime(log.created\_at)}</time>

&#x20;                     </div>

&#x20;                   </li>

&#x20;                 )

&#x20;               })}

&#x20;             </ol>

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* 케이스별 액션 영역 \*/}

&#x20;         {data.status === 'RESERVED' \&\& (

&#x20;           <div className="space-y-3">

&#x20;             <button

&#x20;               onClick={() => router.push(`/admin/checkin?no=${data.reservation\_no}`)}

&#x20;               className="flex w-full items-center justify-center gap-2 rounded-2xl bg-blue-600 py-4 text-base font-bold text-white active:bg-blue-700"

&#x20;             >

&#x20;               <span>📥</span><span>짐 접수 처리</span>

&#x20;             </button>

&#x20;             <button

&#x20;               onClick={() => { setCancelError(null); setShowCancelModal(true) }}

&#x20;               className="w-full text-center text-sm font-medium text-red-500 hover:underline"

&#x20;             >

&#x20;               ⚠ 예약 강제 취소

&#x20;             </button>

&#x20;           </div>

&#x20;         )}



&#x20;         {data.status === 'STORED' \&\& (

&#x20;           <button

&#x20;             onClick={() => router.push(`/admin/checkout?no=${data.reservation\_no}`)}

&#x20;             className="flex w-full items-center justify-center gap-2 rounded-2xl bg-green-600 py-4 text-base font-bold text-white active:bg-green-700"

&#x20;           >

&#x20;             <span>📤</span><span>짐 반환 처리</span>

&#x20;           </button>

&#x20;         )}



&#x20;         {data.status === 'RETURNED' \&\& (

&#x20;           <div className="rounded-2xl bg-gray-50 px-5 py-4 text-center text-sm text-gray-500">

&#x20;             모든 처리가 완료된 예약입니다.

&#x20;           </div>

&#x20;         )}



&#x20;         {data.status === 'CANCELLED' \&\& (

&#x20;           <div className="rounded-2xl bg-gray-50 px-5 py-4 text-center">

&#x20;             <p className="text-sm text-gray-500">취소된 예약입니다.</p>

&#x20;             <p className="mt-1 text-xs text-gray-400">

&#x20;               취소 주체: {data.cancelled\_by === 'ADMIN' ? '운영자 강제 취소' : '고객 직접 취소'}

&#x20;             </p>

&#x20;           </div>

&#x20;         )}

&#x20;       </div>

&#x20;     </AdminLayout>



&#x20;     {showCancelModal \&\& (

&#x20;       <ForceCancelModal

&#x20;         data={data}

&#x20;         onConfirm={handleForceCancel}

&#x20;         onClose={() => { if (!cancelLoading) setShowCancelModal(false) }}

&#x20;         isLoading={cancelLoading}

&#x20;         errorMsg={cancelError}

&#x20;       />

&#x20;     )}

&#x20;   </>

&#x20; )

}

```



\---



\## 7. `src/app/admin/sales/page.tsx`



```typescript

'use client'



import { useState, useEffect, useCallback } from 'react'

import { useRouter } from 'next/navigation'

import { format, startOfWeek, endOfWeek, startOfMonth, endOfMonth, parseISO } from 'date-fns'

import { ko } from 'date-fns/locale'

import AdminLayout from '@/components/bo/AdminLayout'

import { getTodayKst } from '@/lib/validate'

import { getMethodLabel, fmtTime } from '@/lib/bo-utils'



// ── 타입 ─────────────────────────────────────────────────────────

type Period = 'daily' | 'weekly' | 'monthly'



interface SalesSummary {

&#x20; paid\_count:      number

&#x20; cancelled\_count: number

&#x20; net\_revenue:     number

}



interface SalesItem {

&#x20; reservation\_no: string

&#x20; terminal:       string

&#x20; customer\_name:  string

&#x20; method:         string

&#x20; amount:         number

&#x20; status:         string

&#x20; is\_cancelled:   boolean

&#x20; paid\_at:        string | null

}



interface SalesData {

&#x20; period:     Period

&#x20; date\_range: { from: string; to: string }

&#x20; summary:    SalesSummary

&#x20; items:      SalesItem\[]

&#x20; total:      number

}



// ── 날짜 범위 레이블 ──────────────────────────────────────────────

function getDateRangeLabel(period: Period, date: string): string {

&#x20; const d = parseISO(date)

&#x20; switch (period) {

&#x20;   case 'daily':

&#x20;     return format(d, 'yyyy년 M월 d일 (eee)', { locale: ko })

&#x20;   case 'weekly': {

&#x20;     const mon = startOfWeek(d, { weekStartsOn: 1 })

&#x20;     const sun = endOfWeek(d, { weekStartsOn: 1 })

&#x20;     return `${format(mon, 'M/d')} – ${format(sun, 'M/d')}`

&#x20;   }

&#x20;   case 'monthly':

&#x20;     return format(d, 'yyyy년 M월', { locale: ko })

&#x20; }

}



// ── 날짜 이동 버튼 헬퍼 ──────────────────────────────────────────

function shiftDate(period: Period, date: string, dir: -1 | 1): string {

&#x20; const d = parseISO(date)

&#x20; switch (period) {

&#x20;   case 'daily': {

&#x20;     const next = new Date(d); next.setDate(d.getDate() + dir)

&#x20;     return format(next, 'yyyy-MM-dd')

&#x20;   }

&#x20;   case 'weekly': {

&#x20;     const next = new Date(d); next.setDate(d.getDate() + dir \* 7)

&#x20;     return format(next, 'yyyy-MM-dd')

&#x20;   }

&#x20;   case 'monthly': {

&#x20;     const next = new Date(d); next.setMonth(d.getMonth() + dir)

&#x20;     return format(next, 'yyyy-MM-dd')

&#x20;   }

&#x20; }

}



export default function SalesPage() {

&#x20; const router = useRouter()



&#x20; const \[period,   setPeriod]   = useState<Period>('daily')

&#x20; const \[date,     setDate]     = useState(getTodayKst())

&#x20; const \[terminal, setTerminal] = useState<'T1' | 'T2' | 'ALL'>('ALL')



&#x20; const \[salesData, setSalesData] = useState<SalesData | null>(null)

&#x20; const \[isLoading, setIsLoading] = useState(true)

&#x20; const \[error,     setError]     = useState<string | null>(null)



&#x20; const fetchSales = useCallback(async () => {

&#x20;   setIsLoading(true)

&#x20;   setError(null)



&#x20;   try {

&#x20;     const params = new URLSearchParams({ period, date })

&#x20;     if (terminal !== 'ALL') params.set('terminal', terminal)



&#x20;     const res  = await fetch(`/api/admin/sales?${params}`)

&#x20;     const json = await res.json()

&#x20;     if (!res.ok || !json.success) throw new Error(json.error?.message)

&#x20;     setSalesData(json.data as SalesData)

&#x20;   } catch {

&#x20;     setError('데이터를 불러오지 못했습니다.')

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;   }

&#x20; }, \[period, date, terminal])



&#x20; useEffect(() => { fetchSales() }, \[fetchSales])



&#x20; const summary = salesData?.summary



&#x20; return (

&#x20;   <AdminLayout title="매출 관리">

&#x20;     <div className="space-y-5 pb-12 pt-5">



&#x20;       {/\* ── 기간 탭 ─────────────────────────────────────── \*/}

&#x20;       <div className="px-4">

&#x20;         <div className="flex rounded-xl bg-gray-100 p-1">

&#x20;           {(\['daily', 'weekly', 'monthly'] as Period\[]).map((p) => (

&#x20;             <button

&#x20;               key={p}

&#x20;               onClick={() => setPeriod(p)}

&#x20;               className={`

&#x20;                 flex-1 rounded-lg py-2 text-sm font-semibold transition-all

&#x20;                 ${period === p

&#x20;                   ? 'bg-white text-gray-900 shadow-sm'

&#x20;                   : 'text-gray-500 hover:text-gray-700'

&#x20;                 }

&#x20;               `}

&#x20;             >

&#x20;               {p === 'daily' ? '일간' : p === 'weekly' ? '주간' : '월간'}

&#x20;             </button>

&#x20;           ))}

&#x20;         </div>

&#x20;       </div>



&#x20;       {/\* ── 날짜 네비게이터 ──────────────────────────────── \*/}

&#x20;       <div className="px-4">

&#x20;         <div className="flex items-center justify-between rounded-2xl bg-white px-4 py-3 shadow-sm">

&#x20;           <button

&#x20;             onClick={() => setDate(shiftDate(period, date, -1))}

&#x20;             className="rounded-lg p-1.5 text-gray-500 hover:bg-gray-100"

&#x20;           >

&#x20;             <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;               <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;             </svg>

&#x20;           </button>

&#x20;           <div className="text-center">

&#x20;             <p className="text-sm font-semibold text-gray-900">

&#x20;               {getDateRangeLabel(period, date)}

&#x20;             </p>

&#x20;             {salesData?.date\_range \&\& period !== 'daily' \&\& (

&#x20;               <p className="text-xs text-gray-400">

&#x20;                 {salesData.date\_range.from} \~ {salesData.date\_range.to}

&#x20;               </p>

&#x20;             )}

&#x20;           </div>

&#x20;           <button

&#x20;             onClick={() => setDate(shiftDate(period, date, 1))}

&#x20;             className="rounded-lg p-1.5 text-gray-500 hover:bg-gray-100"

&#x20;           >

&#x20;             <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;               <path strokeLinecap="round" strokeLinejoin="round" d="M8.25 4.5l7.5 7.5-7.5 7.5" />

&#x20;             </svg>

&#x20;           </button>

&#x20;         </div>

&#x20;       </div>



&#x20;       {/\* ── 터미널 필터 ──────────────────────────────────── \*/}

&#x20;       <div className="flex gap-2 px-4">

&#x20;         {(\['ALL', 'T1', 'T2'] as const).map((t) => (

&#x20;           <button

&#x20;             key={t}

&#x20;             onClick={() => setTerminal(t)}

&#x20;             className={`

&#x20;               rounded-full px-4 py-1.5 text-sm font-medium transition-colors

&#x20;               ${terminal === t

&#x20;                 ? 'bg-blue-600 text-white'

&#x20;                 : 'bg-white text-gray-600 shadow-sm hover:bg-gray-50'

&#x20;               }

&#x20;             `}

&#x20;           >

&#x20;             {t === 'ALL' ? '전체' : t}

&#x20;           </button>

&#x20;         ))}

&#x20;       </div>



&#x20;       {/\* ── 매출 요약 카드 ───────────────────────────────── \*/}

&#x20;       {isLoading ? (

&#x20;         <div className="mx-4 animate-pulse rounded-2xl bg-white p-5 shadow-sm">

&#x20;           <div className="space-y-3">

&#x20;             {\[1, 2, 3].map((i) => (

&#x20;               <div key={i} className="flex justify-between">

&#x20;                 <div className="h-3.5 w-20 rounded bg-gray-200" />

&#x20;                 <div className="h-3.5 w-14 rounded bg-gray-200" />

&#x20;               </div>

&#x20;             ))}

&#x20;           </div>

&#x20;         </div>

&#x20;       ) : summary \&\& (

&#x20;         <div className="mx-4 rounded-2xl bg-white px-5 py-4 shadow-sm">

&#x20;           <div className="flex items-center justify-between py-2.5">

&#x20;             <span className="text-sm text-gray-500">결제 완료</span>

&#x20;             <span className="text-sm font-medium text-gray-900">{summary.paid\_count}건</span>

&#x20;           </div>

&#x20;           <div className="flex items-center justify-between py-2.5">

&#x20;             <span className="text-sm text-gray-500">취소 / 환불</span>

&#x20;             <span className="text-sm font-medium text-gray-900">{summary.cancelled\_count}건</span>

&#x20;           </div>

&#x20;           <div className="my-1 border-t border-gray-100" />

&#x20;           <div className="flex items-center justify-between py-2.5">

&#x20;             <span className="text-sm font-semibold text-gray-900">순매출</span>

&#x20;             <span className="text-lg font-bold text-blue-600">

&#x20;               {summary.net\_revenue.toLocaleString()}원

&#x20;             </span>

&#x20;           </div>

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* ── 결제 내역 목록 ───────────────────────────────── \*/}

&#x20;       <div className="px-4">

&#x20;         <h3 className="mb-3 text-sm font-semibold text-gray-700">결제 내역</h3>



&#x20;         {error \&\& (

&#x20;           <div className="rounded-xl bg-red-50 px-4 py-3 text-sm text-red-600">{error}</div>

&#x20;         )}



&#x20;         {!isLoading \&\& !error \&\& (!salesData?.items.length) \&\& (

&#x20;           <div className="flex flex-col items-center justify-center rounded-2xl bg-white py-12 shadow-sm">

&#x20;             <div className="mb-3 text-4xl">💰</div>

&#x20;             <p className="text-sm text-gray-400">선택한 기간에 결제 내역이 없습니다.</p>

&#x20;           </div>

&#x20;         )}



&#x20;         {!isLoading \&\& salesData \&\& salesData.items.length > 0 \&\& (

&#x20;           <div className="overflow-hidden rounded-2xl bg-white shadow-sm">

&#x20;             <ul className="divide-y divide-gray-100">

&#x20;               {salesData.items.map((item) => (

&#x20;                 <li key={`${item.reservation\_no}-${item.paid\_at}`}>

&#x20;                   <button

&#x20;                     onClick={() => router.push(`/admin/reservations/${item.reservation\_no}`)}

&#x20;                     className="flex w-full items-center gap-3 px-4 py-3.5 text-left active:bg-gray-50"

&#x20;                   >

&#x20;                     <div className="min-w-0 flex-1">

&#x20;                       {/\* 예약번호 + 터미널 + 환불 배지 \*/}

&#x20;                       <div className="flex items-center gap-1.5">

&#x20;                         <span className="font-mono text-xs font-bold text-gray-800">

&#x20;                           {item.reservation\_no}

&#x20;                         </span>

&#x20;                         <span className="rounded-full bg-gray-100 px-1.5 py-0.5 text-\[9px] font-semibold text-gray-500">

&#x20;                           {item.terminal}

&#x20;                         </span>

&#x20;                         {item.is\_cancelled \&\& (

&#x20;                           <span className="rounded-full bg-red-100 px-1.5 py-0.5 text-\[9px] font-semibold text-red-600">

&#x20;                             환불

&#x20;                           </span>

&#x20;                         )}

&#x20;                       </div>

&#x20;                       {/\* 예약자 + 결제 수단 \*/}

&#x20;                       <p className="mt-0.5 text-xs text-gray-500">

&#x20;                         {item.customer\_name} · {getMethodLabel(item.method)}

&#x20;                       </p>

&#x20;                     </div>



&#x20;                     {/\* 금액 + 시각 \*/}

&#x20;                     <div className="shrink-0 text-right">

&#x20;                       <p className={`text-sm font-bold ${item.is\_cancelled ? 'text-red-500' : 'text-gray-900'}`}>

&#x20;                         {item.is\_cancelled ? '-' : ''}{item.amount.toLocaleString()}원

&#x20;                       </p>

&#x20;                       <p className="text-xs text-gray-400">

&#x20;                         {item.paid\_at ? fmtTime(item.paid\_at) : '-'}

&#x20;                       </p>

&#x20;                     </div>

&#x20;                   </button>

&#x20;                 </li>

&#x20;               ))}

&#x20;             </ul>

&#x20;           </div>

&#x20;         )}

&#x20;       </div>

&#x20;     </div>

&#x20;   </AdminLayout>

&#x20; )

}

```



\---



\## 설계 노트



\### QRScanner — html5-qrcode 동적 import



`html5-qrcode`는 `window` 객체를 참조하므로 SSR에서 직접 import하면 빌드 오류가 난다.

`await import('html5-qrcode')` 동적 import를 `useEffect` 안에서 실행해 클라이언트에서만 로드한다.



```typescript

// 잘못된 방법 (SSR에서 오류)

import { Html5QrcodeScanner } from 'html5-qrcode'



// 올바른 방법 (클라이언트 전용 동적 import)

const { Html5QrcodeScanner } = await import('html5-qrcode')

```



라이브러리가 DOM에 직접 UI를 주입하므로, 스캐너 div(`id="bo-qr-scanner"`)가 DOM에 존재한 후 `render()`를 호출해야 한다. `useEffect` 순서를 통해 이를 보장한다.



\### CheckInOutPage — mode prop 팩토리 패턴



접수/반환은 로직의 98%가 동일하고 API 경로(`checkin`/`checkout`)·타이틀·버튼 텍스트만 다르다.

`mode: 'checkin' | 'checkout'` 단일 prop으로 두 화면을 커버해 코드 중복을 제거했다.



```

mode="checkin"  → /api/admin/reservations/:no/checkin  PATCH

mode="checkout" → /api/admin/reservations/:no/checkout PATCH

```



\### 보관 현황 — 반환 처리 연동



`StorageDetailSheet`의 \[반환 처리하기] 버튼은 `/admin/checkout?no={reservationNo}`로 이동한다.

checkout 페이지는 `searchParams.get('no')`로 예약번호를 읽어 자동으로 조회 화면을 건너뛰고 확인 화면을 표시한다.

(checkout/page.tsx의 초기화 코드에 `searchParams` 처리 추가 필요)



\### 매출 — 날짜 네비게이터



`shiftDate()` 헬퍼가 period에 따라 날짜를 1일 / 7일 / 1개월씩 이동한다.

달력 컴포넌트 없이 좌우 화살표 버튼만으로 직관적인 기간 이동 UX를 구현했다.



\---



\## Session 12 체크리스트



| 파일 | 구현 항목 | 상태 |

|------|----------|------|

| `bo-utils.ts` | 포맷·레이블·에러 메시지 헬퍼 통합 | ✅ |

| `QRScanner.tsx` | html5-qrcode 동적 import / 카메라 권한 오류 처리 / 스캔 가이드 오버레이 | ✅ |

| `checkin/page.tsx` | scan→confirm→done 3단계 화면 상태 / 오류 토스트 / 낙관적 검증 | ✅ |

| `checkout/page.tsx` | CheckInOutPage mode="checkout" 재사용 | ✅ |

| `storage/page.tsx` | 30초 폴링 / 경과 시간 경고 레벨 / Bottom Sheet 팝업 / 반환 연동 | ✅ |

| `reservations/page.tsx` | URL 파라미터 초기 필터 / 날짜·터미널·상태 3중 필터 / 상태별 시각 | ✅ |

| `reservations/\[id]/page.tsx` | 상태 4케이스 액션 / 강제 취소 모달 + 환불 안내 / 처리 이력 타임라인 | ✅ |

| `sales/page.tsx` | 기간 탭 / 날짜 네비게이터 / 터미널 필터 / 매출 요약 / 취소건 마이너스 표시 | ✅ |



\---



\## 전체 구현 완료 현황



| 구분 | 세션 | 파일 수 | 상태 |

|------|------|---------|------|

| 개발 환경 + lib/ 유틸 | 06 | 7개 | ✅ |

| FO API Route | 07 | 5개 | ✅ |

| BO API Route + 헬퍼 | 08 | 9개 | ✅ |

| BO 프론트 기반 (로그인·레이아웃·대시보드) | 09 | 5개 | ✅ |

| FO 예약 플로우 전체 | 10 | 9개 | ✅ |

| FO 예약 확인·상세·안내 | 11 | 4개 | ✅ |

| BO 나머지 화면 | 12 | 8개 | ✅ |

| \*\*합계\*\* | | \*\*47개\*\* | ✅ |



\*\*BagDrop 전체 코드베이스 구현 완료.\*\*

