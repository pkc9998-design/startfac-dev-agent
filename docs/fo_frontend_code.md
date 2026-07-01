\# BagDrop FO 예약 플로우 프론트엔드 구현 코드



> 작성: 개발 에이전트 가온

> 세션: Session 10

> 최종 업데이트: 2026-07-01

> 대상: FO 홈 + 예약 플로우 STEP 1\~5 + QR 컴포넌트 (8개 파일)

> 기준: 모바일웹 max-width 430px / Tailwind CSS

> 상태: ✅ Session 10 확정



\---



\## 파일 목록



| 파일 경로 | 역할 |

|-----------|------|

| `src/lib/reservation-store.ts` | sessionStorage 상태 관리 유틸 |

| `src/app/page.tsx` | FO 홈 |

| `src/components/fo/StepBar.tsx` | 스텝 진행 바 |

| `src/app/reserve/step1/page.tsx` | STEP 1 — 터미널/방향/날짜 |

| `src/app/reserve/step2/page.tsx` | STEP 2 — 짐 개수/요금 |

| `src/app/reserve/step3/page.tsx` | STEP 3 — 고객 정보 |

| `src/app/reserve/step4/page.tsx` | STEP 4 — 결제 |

| `src/app/reserve/complete/page.tsx` | STEP 5 — 예약 완료 |

| `src/components/fo/ReservationQR.tsx` | QR 코드 컴포넌트 |



\---



\## 0. `src/lib/reservation-store.ts` (sessionStorage 상태 관리)



> STEP 간 이동 시 입력값 유지.

> sessionStorage를 직접 호출하는 코드를 각 페이지에 분산시키지 않고 한 곳에서 관리.



```typescript

// 예약 플로우 전체에서 공유하는 임시 상태

export interface ReservationDraft {

&#x20; // STEP 1

&#x20; terminal?:  'T1' | 'T2'

&#x20; direction?: 'DEPARTURE' | 'ARRIVAL'

&#x20; use\_date?:  string                    // YYYY-MM-DD



&#x20; // STEP 2

&#x20; bag\_count?: 1 | 2



&#x20; // STEP 3

&#x20; customer\_name?:  string

&#x20; customer\_phone?: string               // 숫자만 11자리



&#x20; // prepare API 응답 (STEP 4 결제 전 저장)

&#x20; toss\_order\_id?: string

}



const STORAGE\_KEY = 'bagdrop\_reservation\_draft'



export const reservationStore = {

&#x20; /\*\* 전체 draft 조회 \*/

&#x20; get(): ReservationDraft {

&#x20;   if (typeof window === 'undefined') return {}

&#x20;   try {

&#x20;     const raw = sessionStorage.getItem(STORAGE\_KEY)

&#x20;     return raw ? (JSON.parse(raw) as ReservationDraft) : {}

&#x20;   } catch {

&#x20;     return {}

&#x20;   }

&#x20; },



&#x20; /\*\* draft 부분 업데이트 (merge) \*/

&#x20; set(partial: Partial<ReservationDraft>): void {

&#x20;   if (typeof window === 'undefined') return

&#x20;   try {

&#x20;     const prev = reservationStore.get()

&#x20;     sessionStorage.setItem(STORAGE\_KEY, JSON.stringify({ ...prev, ...partial }))

&#x20;   } catch {

&#x20;     // sessionStorage 접근 실패는 무시

&#x20;   }

&#x20; },



&#x20; /\*\* draft 전체 초기화 \*/

&#x20; clear(): void {

&#x20;   if (typeof window === 'undefined') return

&#x20;   try {

&#x20;     sessionStorage.removeItem(STORAGE\_KEY)

&#x20;   } catch {}

&#x20; },

}

```



\---



\## 1. `src/app/page.tsx` (FO 홈)



```typescript

import Link from 'next/link'



// 이용 방법 단계 데이터

const HOW\_TO\_STEPS = \[

&#x20; {

&#x20;   step: '01',

&#x20;   icon: '🧳',

&#x20;   title: '짐 맡기기',

&#x20;   desc:  '터미널 입구 BagDrop 부스에 짐을 맡기세요',

&#x20; },

&#x20; {

&#x20;   step: '02',

&#x20;   icon: '🚗',

&#x20;   title: '주차하기',

&#x20;   desc:  '가볍게 주차장으로 이동하세요',

&#x20; },

&#x20; {

&#x20;   step: '03',

&#x20;   icon: '✅',

&#x20;   title: '짐 찾기',

&#x20;   desc:  '돌아와서 QR로 짐을 수령하세요',

&#x20; },

]



export default function HomePage() {

&#x20; return (

&#x20;   // min-h 대신 min-h-svh: 모바일 주소창 높이 변동 대응

&#x20;   <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-white">



&#x20;     {/\* ── GNB ─────────────────────────────────────────── \*/}

&#x20;     <header className="flex h-14 items-center justify-between px-5">

&#x20;       <span className="text-lg font-bold text-gray-900">BagDrop</span>

&#x20;       <Link

&#x20;         href="/guide"

&#x20;         className="text-sm font-medium text-gray-500 hover:text-gray-700"

&#x20;       >

&#x20;         이용안내

&#x20;       </Link>

&#x20;     </header>



&#x20;     {/\* ── 히어로 배너 ──────────────────────────────────── \*/}

&#x20;     <section className="bg-blue-600 px-5 py-10 text-white">

&#x20;       {/\* 서비스 일러스트 대체 이모지 (실제 서비스에서는 이미지로 교체) \*/}

&#x20;       <div className="mb-6 text-center text-6xl">🧳</div>

&#x20;       <h1 className="text-center text-2xl font-bold leading-snug">

&#x20;         짐은 여기 두고,<br />편하게 주차하세요

&#x20;       </h1>

&#x20;       <p className="mt-3 text-center text-sm text-blue-200">

&#x20;         인천공항 터미널 입구 짐 임시보관 서비스

&#x20;       </p>

&#x20;     </section>



&#x20;     {/\* ── 이용 방법 3단계 ──────────────────────────────── \*/}

&#x20;     <section className="px-5 py-8">

&#x20;       <h2 className="mb-5 text-center text-sm font-semibold text-gray-500 uppercase tracking-wide">

&#x20;         이용 방법

&#x20;       </h2>

&#x20;       <div className="space-y-4">

&#x20;         {HOW\_TO\_STEPS.map((s) => (

&#x20;           <div key={s.step} className="flex items-start gap-4">

&#x20;             {/\* 아이콘 원 \*/}

&#x20;             <div className="flex h-11 w-11 shrink-0 items-center justify-center rounded-full bg-blue-50 text-xl">

&#x20;               {s.icon}

&#x20;             </div>

&#x20;             <div className="pt-1">

&#x20;               <div className="flex items-center gap-2">

&#x20;                 <span className="text-xs font-bold text-blue-500">{s.step}</span>

&#x20;                 <span className="text-sm font-semibold text-gray-900">{s.title}</span>

&#x20;               </div>

&#x20;               <p className="mt-0.5 text-xs text-gray-500">{s.desc}</p>

&#x20;             </div>

&#x20;           </div>

&#x20;         ))}

&#x20;       </div>

&#x20;     </section>



&#x20;     {/\* ── 짐 초과 안내 배너 ────────────────────────────── \*/}

&#x20;     <div className="mx-5 mb-6 rounded-xl bg-amber-50 px-4 py-3">

&#x20;       <div className="flex items-start gap-2">

&#x20;         <span className="mt-0.5 text-base">⚠️</span>

&#x20;         <p className="text-xs leading-relaxed text-amber-800">

&#x20;           짐이 3개 이상이신 경우, 예약을 추가로 생성해 주세요.

&#x20;           <br />

&#x20;           <span className="font-medium">(예약 1건당 최대 2개)</span>

&#x20;         </p>

&#x20;       </div>

&#x20;     </div>



&#x20;     {/\* ── CTA 버튼 영역 ─────────────────────────────────── \*/}

&#x20;     <div className="px-5 pb-10">

&#x20;       <Link

&#x20;         href="/reserve/step1"

&#x20;         className="

&#x20;           flex w-full items-center justify-center gap-2

&#x20;           rounded-2xl bg-blue-600 py-4

&#x20;           text-base font-bold text-white

&#x20;           shadow-lg shadow-blue-600/30

&#x20;           active:bg-blue-700

&#x20;         "

&#x20;       >

&#x20;         <span>🧳</span>

&#x20;         <span>예약하기</span>

&#x20;       </Link>



&#x20;       <div className="mt-5 text-center">

&#x20;         <p className="text-sm text-gray-400">이미 예약하셨나요?</p>

&#x20;         <Link

&#x20;           href="/my-reservation"

&#x20;           className="mt-1 inline-block text-sm font-medium text-blue-600 hover:underline"

&#x20;         >

&#x20;           예약 확인하기 →

&#x20;         </Link>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 2. `src/components/fo/StepBar.tsx`



```typescript

interface StepBarProps {

&#x20; current: 1 | 2 | 3 | 4   // 현재 스텝 (총 4스텝)

&#x20; onBack:  () => void        // 뒤로가기 핸들러

}



export default function StepBar({ current, onBack }: StepBarProps) {

&#x20; const total    = 4

&#x20; const progress = (current / total) \* 100   // 퍼센트



&#x20; return (

&#x20;   <div className="flex items-center gap-3 px-4 py-3">

&#x20;     {/\* 뒤로가기 버튼 \*/}

&#x20;     <button

&#x20;       onClick={onBack}

&#x20;       aria-label="이전 단계로"

&#x20;       className="-ml-1 shrink-0 rounded-lg p-1.5 text-gray-500 hover:bg-gray-100 active:bg-gray-200"

&#x20;     >

&#x20;       <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;         <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;       </svg>

&#x20;     </button>



&#x20;     {/\* 진행 바 \*/}

&#x20;     <div className="flex flex-1 flex-col gap-1">

&#x20;       {/\* 스텝 텍스트 \*/}

&#x20;       <span className="text-right text-\[11px] font-medium text-gray-400">

&#x20;         {current}/{total}

&#x20;       </span>

&#x20;       {/\* 프로그레스 바 \*/}

&#x20;       <div className="h-1.5 w-full overflow-hidden rounded-full bg-gray-100">

&#x20;         <div

&#x20;           className="h-full rounded-full bg-blue-600 transition-all duration-300 ease-out"

&#x20;           style={{ width: `${progress}%` }}

&#x20;           role="progressbar"

&#x20;           aria-valuenow={current}

&#x20;           aria-valuemin={1}

&#x20;           aria-valuemax={total}

&#x20;         />

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 3. `src/app/reserve/step1/page.tsx`



```typescript

'use client'



import { useState, useEffect } from 'react'

import { useRouter } from 'next/navigation'

import StepBar from '@/components/fo/StepBar'

import { reservationStore } from '@/lib/reservation-store'

import { addDays, format, isBefore, startOfDay, parseISO } from 'date-fns'

import { ko } from 'date-fns/locale'



type Terminal  = 'T1' | 'T2'

type Direction = 'DEPARTURE' | 'ARRIVAL'



// ── 미니 달력 컴포넌트 ───────────────────────────────────────────

function MiniCalendar({

&#x20; value,

&#x20; onChange,

}: {

&#x20; value:    string | undefined

&#x20; onChange: (date: string) => void

}) {

&#x20; const today    = startOfDay(new Date())

&#x20; const maxDate  = addDays(today, 90)



&#x20; // 현재 달력에 표시할 기준 연월 (선택 날짜 또는 오늘 기준)

&#x20; const \[viewYear,  setViewYear]  = useState(() => today.getFullYear())

&#x20; const \[viewMonth, setViewMonth] = useState(() => today.getMonth())   // 0-indexed



&#x20; // 달력 구성

&#x20; const firstDay    = new Date(viewYear, viewMonth, 1)

&#x20; const lastDay     = new Date(viewYear, viewMonth + 1, 0)

&#x20; const startOffset = firstDay.getDay()   // 0=일요일



&#x20; const days: (Date | null)\[] = \[

&#x20;   ...Array(startOffset).fill(null),

&#x20;   ...Array.from({ length: lastDay.getDate() }, (\_, i) => new Date(viewYear, viewMonth, i + 1)),

&#x20; ]



&#x20; function isDisabled(d: Date): boolean {

&#x20;   return isBefore(startOfDay(d), today) || isBefore(maxDate, startOfDay(d))

&#x20; }



&#x20; function isSelected(d: Date): boolean {

&#x20;   if (!value) return false

&#x20;   return format(d, 'yyyy-MM-dd') === value

&#x20; }



&#x20; function isToday(d: Date): boolean {

&#x20;   return format(d, 'yyyy-MM-dd') === format(today, 'yyyy-MM-dd')

&#x20; }



&#x20; function prevMonth() {

&#x20;   if (viewMonth === 0) { setViewYear(y => y - 1); setViewMonth(11) }

&#x20;   else setViewMonth(m => m - 1)

&#x20; }

&#x20; function nextMonth() {

&#x20;   if (viewMonth === 11) { setViewYear(y => y + 1); setViewMonth(0) }

&#x20;   else setViewMonth(m => m + 1)

&#x20; }



&#x20; const DAY\_LABELS = \['일', '월', '화', '수', '목', '금', '토']



&#x20; return (

&#x20;   <div className="rounded-2xl bg-white p-4 shadow-sm">

&#x20;     {/\* 월 이동 헤더 \*/}

&#x20;     <div className="mb-3 flex items-center justify-between">

&#x20;       <button onClick={prevMonth} className="rounded-lg p-1.5 text-gray-500 hover:bg-gray-100">

&#x20;         <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;           <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;         </svg>

&#x20;       </button>

&#x20;       <span className="text-sm font-semibold text-gray-900">

&#x20;         {viewYear}년 {viewMonth + 1}월

&#x20;       </span>

&#x20;       <button onClick={nextMonth} className="rounded-lg p-1.5 text-gray-500 hover:bg-gray-100">

&#x20;         <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" strokeWidth={2.5} stroke="currentColor">

&#x20;           <path strokeLinecap="round" strokeLinejoin="round" d="M8.25 4.5l7.5 7.5-7.5 7.5" />

&#x20;         </svg>

&#x20;       </button>

&#x20;     </div>



&#x20;     {/\* 요일 헤더 \*/}

&#x20;     <div className="mb-1 grid grid-cols-7 text-center">

&#x20;       {DAY\_LABELS.map((d, i) => (

&#x20;         <span

&#x20;           key={d}

&#x20;           className={`text-\[11px] font-medium ${i === 0 ? 'text-red-400' : i === 6 ? 'text-blue-400' : 'text-gray-400'}`}

&#x20;         >

&#x20;           {d}

&#x20;         </span>

&#x20;       ))}

&#x20;     </div>



&#x20;     {/\* 날짜 그리드 \*/}

&#x20;     <div className="grid grid-cols-7 gap-y-1 text-center">

&#x20;       {days.map((d, idx) => {

&#x20;         if (!d) return <div key={`empty-${idx}`} />



&#x20;         const disabled = isDisabled(d)

&#x20;         const selected = isSelected(d)

&#x20;         const todayMark = isToday(d)

&#x20;         const isSun = d.getDay() === 0

&#x20;         const isSat = d.getDay() === 6



&#x20;         return (

&#x20;           <button

&#x20;             key={d.toISOString()}

&#x20;             disabled={disabled}

&#x20;             onClick={() => onChange(format(d, 'yyyy-MM-dd'))}

&#x20;             className={`

&#x20;               relative flex h-9 w-9 mx-auto items-center justify-center

&#x20;               rounded-full text-sm transition-colors

&#x20;               ${disabled

&#x20;                 ? 'text-gray-300 cursor-not-allowed'

&#x20;                 : selected

&#x20;                   ? 'bg-blue-600 font-semibold text-white'

&#x20;                   : todayMark

&#x20;                     ? 'font-semibold text-blue-600 hover:bg-blue-50'

&#x20;                     : isSun

&#x20;                       ? 'text-red-500 hover:bg-red-50'

&#x20;                       : isSat

&#x20;                         ? 'text-blue-500 hover:bg-blue-50'

&#x20;                         : 'text-gray-800 hover:bg-gray-100'

&#x20;               }

&#x20;             `}

&#x20;           >

&#x20;             {d.getDate()}

&#x20;             {/\* 오늘 표시 점 \*/}

&#x20;             {todayMark \&\& !selected \&\& (

&#x20;               <span className="absolute bottom-1 left-1/2 h-1 w-1 -translate-x-1/2 rounded-full bg-blue-500" />

&#x20;             )}

&#x20;           </button>

&#x20;         )

&#x20;       })}

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// ── 선택 카드 컴포넌트 ───────────────────────────────────────────

function SelectCard({

&#x20; selected,

&#x20; onClick,

&#x20; children,

}: {

&#x20; selected: boolean

&#x20; onClick:  () => void

&#x20; children: React.ReactNode

}) {

&#x20; return (

&#x20;   <button

&#x20;     onClick={onClick}

&#x20;     className={`

&#x20;       relative flex flex-1 flex-col items-center justify-center gap-1.5

&#x20;       rounded-2xl border-2 py-5 transition-all active:scale-\[0.98]

&#x20;       ${selected

&#x20;         ? 'border-blue-600 bg-blue-50 shadow-sm shadow-blue-100'

&#x20;         : 'border-gray-200 bg-white hover:border-gray-300'

&#x20;       }

&#x20;     `}

&#x20;   >

&#x20;     {/\* 선택 체크 배지 \*/}

&#x20;     {selected \&\& (

&#x20;       <span className="absolute right-2.5 top-2.5 flex h-5 w-5 items-center justify-center rounded-full bg-blue-600">

&#x20;         <svg className="h-3 w-3 text-white" fill="none" viewBox="0 0 24 24" strokeWidth={3} stroke="currentColor">

&#x20;           <path strokeLinecap="round" strokeLinejoin="round" d="M4.5 12.75l6 6 9-13.5" />

&#x20;         </svg>

&#x20;       </span>

&#x20;     )}

&#x20;     {children}

&#x20;   </button>

&#x20; )

}



// ── 메인 페이지 ──────────────────────────────────────────────────

export default function Step1Page() {

&#x20; const router = useRouter()

&#x20; const draft  = reservationStore.get()



&#x20; const \[terminal,  setTerminal]  = useState<Terminal | undefined>(draft.terminal)

&#x20; const \[direction, setDirection] = useState<Direction | undefined>(draft.direction)

&#x20; const \[useDate,   setUseDate]   = useState<string | undefined>(draft.use\_date)



&#x20; const canProceed = !!terminal \&\& !!direction \&\& !!useDate



&#x20; // 선택 날짜 포맷 표시용

&#x20; const dateLabel = useDate

&#x20;   ? format(parseISO(useDate), 'yyyy년 M월 d일 (eee)', { locale: ko })

&#x20;   : null



&#x20; function handleNext() {

&#x20;   if (!canProceed) return

&#x20;   reservationStore.set({ terminal, direction, use\_date: useDate })

&#x20;   router.push('/reserve/step2')

&#x20; }



&#x20; function handleBack() {

&#x20;   router.push('/')

&#x20; }



&#x20; return (

&#x20;   <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-gray-50">

&#x20;     {/\* 스텝 바 \*/}

&#x20;     <StepBar current={1} onBack={handleBack} />



&#x20;     <div className="flex-1 space-y-7 overflow-y-auto px-4 pb-32 pt-2">



&#x20;       {/\* 터미널 선택 \*/}

&#x20;       <section>

&#x20;         <h2 className="mb-3 text-base font-semibold text-gray-900">

&#x20;           터미널을 선택해 주세요

&#x20;         </h2>

&#x20;         <div className="flex gap-3">

&#x20;           <SelectCard selected={terminal === 'T1'} onClick={() => setTerminal('T1')}>

&#x20;             <span className="text-2xl">✈️</span>

&#x20;             <span className="text-sm font-bold text-gray-900">T1</span>

&#x20;             <span className="text-xs text-gray-500">제1여객터미널</span>

&#x20;           </SelectCard>

&#x20;           <SelectCard selected={terminal === 'T2'} onClick={() => setTerminal('T2')}>

&#x20;             <span className="text-2xl">✈️</span>

&#x20;             <span className="text-sm font-bold text-gray-900">T2</span>

&#x20;             <span className="text-xs text-gray-500">제2여객터미널</span>

&#x20;           </SelectCard>

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* 이용 방향 선택 \*/}

&#x20;       <section>

&#x20;         <h2 className="mb-3 text-base font-semibold text-gray-900">

&#x20;           이용 방향을 선택해 주세요

&#x20;         </h2>

&#x20;         <div className="flex gap-3">

&#x20;           <SelectCard selected={direction === 'DEPARTURE'} onClick={() => setDirection('DEPARTURE')}>

&#x20;             <span className="text-2xl">🛫</span>

&#x20;             <span className="text-sm font-bold text-gray-900">출국</span>

&#x20;             <span className="px-2 text-center text-xs text-gray-500">공항 떠나기 전 짐 보관</span>

&#x20;           </SelectCard>

&#x20;           <SelectCard selected={direction === 'ARRIVAL'} onClick={() => setDirection('ARRIVAL')}>

&#x20;             <span className="text-2xl">🛬</span>

&#x20;             <span className="text-sm font-bold text-gray-900">귀국</span>

&#x20;             <span className="px-2 text-center text-xs text-gray-500">차 가지러 가기 전 짐 보관</span>

&#x20;           </SelectCard>

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* 날짜 선택 \*/}

&#x20;       <section>

&#x20;         <h2 className="mb-3 text-base font-semibold text-gray-900">

&#x20;           이용 날짜를 선택해 주세요

&#x20;         </h2>

&#x20;         {dateLabel \&\& (

&#x20;           <p className="mb-2 text-center text-sm font-semibold text-blue-600">{dateLabel}</p>

&#x20;         )}

&#x20;         <MiniCalendar value={useDate} onChange={setUseDate} />

&#x20;       </section>

&#x20;     </div>



&#x20;     {/\* 다음 버튼 (화면 하단 고정) \*/}

&#x20;     <div className="fixed bottom-0 left-0 right-0 mx-auto max-w-\[430px] bg-white px-4 pb-safe pt-3 shadow-\[0\_-1px\_8px\_rgba(0,0,0,0.06)]">

&#x20;       <button

&#x20;         onClick={handleNext}

&#x20;         disabled={!canProceed}

&#x20;         className="

&#x20;           w-full rounded-2xl py-4 text-base font-bold

&#x20;           transition-colors

&#x20;           disabled:bg-gray-200 disabled:text-gray-400

&#x20;           enabled:bg-blue-600 enabled:text-white enabled:active:bg-blue-700

&#x20;         "

&#x20;       >

&#x20;         다음으로

&#x20;       </button>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 4. `src/app/reserve/step2/page.tsx`



```typescript

'use client'



import { useState, useEffect } from 'react'

import { useRouter } from 'next/navigation'

import StepBar from '@/components/fo/StepBar'

import { reservationStore } from '@/lib/reservation-store'

import { TERMINAL\_LABEL, DIRECTION\_LABEL, RESERVATION\_AMOUNT } from '@/constants'

import { format, parseISO } from 'date-fns'

import { ko } from 'date-fns/locale'



type BagCount = 1 | 2



function SummaryChip({ label }: { label: string }) {

&#x20; return (

&#x20;   <span className="rounded-full bg-white px-3 py-1 text-xs font-medium text-gray-600 shadow-sm">

&#x20;     {label}

&#x20;   </span>

&#x20; )

}



export default function Step2Page() {

&#x20; const router = useRouter()

&#x20; const draft  = reservationStore.get()



&#x20; // STEP 1 값 없으면 처음으로 리디렉트

&#x20; useEffect(() => {

&#x20;   if (!draft.terminal || !draft.direction || !draft.use\_date) {

&#x20;     router.replace('/reserve/step1')

&#x20;   }

&#x20; }, \[])



&#x20; const \[bagCount, setBagCount] = useState<BagCount | undefined>(

&#x20;   draft.bag\_count as BagCount | undefined

&#x20; )



&#x20; function handleNext() {

&#x20;   if (!bagCount) return

&#x20;   reservationStore.set({ bag\_count: bagCount })

&#x20;   router.push('/reserve/step3')

&#x20; }



&#x20; // 이전 스텝 요약 레이블

&#x20; const summaryChips = \[

&#x20;   draft.terminal  ? draft.terminal : null,

&#x20;   draft.direction ? DIRECTION\_LABEL\[draft.direction] : null,

&#x20;   draft.use\_date

&#x20;     ? format(parseISO(draft.use\_date), 'M/d(eee)', { locale: ko })

&#x20;     : null,

&#x20; ].filter(Boolean) as string\[]



&#x20; return (

&#x20;   <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-gray-50">

&#x20;     <StepBar current={2} onBack={() => router.push('/reserve/step1')} />



&#x20;     <div className="flex-1 space-y-6 px-4 pb-32 pt-3">



&#x20;       {/\* 이전 스텝 요약 칩 \*/}

&#x20;       {summaryChips.length > 0 \&\& (

&#x20;         <div className="flex flex-wrap gap-2 rounded-2xl bg-gray-100 px-4 py-3">

&#x20;           {summaryChips.map((chip) => (

&#x20;             <SummaryChip key={chip} label={chip} />

&#x20;           ))}

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* 짐 개수 선택 \*/}

&#x20;       <section>

&#x20;         <h2 className="mb-3 text-base font-semibold text-gray-900">

&#x20;           짐 개수를 선택해 주세요

&#x20;         </h2>

&#x20;         <div className="flex gap-3">

&#x20;           {(\[1, 2] as BagCount\[]).map((n) => (

&#x20;             <button

&#x20;               key={n}

&#x20;               onClick={() => setBagCount(n)}

&#x20;               className={`

&#x20;                 relative flex flex-1 flex-col items-center justify-center gap-2

&#x20;                 rounded-2xl border-2 py-8 transition-all active:scale-\[0.98]

&#x20;                 ${bagCount === n

&#x20;                   ? 'border-blue-600 bg-blue-50 shadow-sm'

&#x20;                   : 'border-gray-200 bg-white hover:border-gray-300'

&#x20;                 }

&#x20;               `}

&#x20;             >

&#x20;               {/\* 체크 배지 \*/}

&#x20;               {bagCount === n \&\& (

&#x20;                 <span className="absolute right-2.5 top-2.5 flex h-5 w-5 items-center justify-center rounded-full bg-blue-600">

&#x20;                   <svg className="h-3 w-3 text-white" fill="none" viewBox="0 0 24 24" strokeWidth={3} stroke="currentColor">

&#x20;                     <path strokeLinecap="round" strokeLinejoin="round" d="M4.5 12.75l6 6 9-13.5" />

&#x20;                   </svg>

&#x20;                 </span>

&#x20;               )}

&#x20;               {/\* 짐 이모지 \*/}

&#x20;               <div className="flex gap-1">

&#x20;                 {Array.from({ length: n }).map((\_, i) => (

&#x20;                   <span key={i} className="text-2xl">🧳</span>

&#x20;                 ))}

&#x20;               </div>

&#x20;               <span className={`text-sm font-bold ${bagCount === n ? 'text-blue-700' : 'text-gray-700'}`}>

&#x20;                 {n}개

&#x20;               </span>

&#x20;             </button>

&#x20;           ))}

&#x20;         </div>



&#x20;         {/\* 짐 초과 안내 \*/}

&#x20;         <div className="mt-3 flex items-start gap-2 rounded-xl bg-amber-50 px-4 py-3">

&#x20;           <span className="mt-0.5 text-sm">⚠️</span>

&#x20;           <p className="text-xs leading-relaxed text-amber-800">

&#x20;             짐이 3개 이상이신 경우, 예약을 추가로 생성해 주세요.

&#x20;             <span className="font-semibold"> (예약 1건당 최대 2개)</span>

&#x20;           </p>

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* 요금 안내 \*/}

&#x20;       <section>

&#x20;         <div className="rounded-2xl bg-white p-5 shadow-sm">

&#x20;           <div className="flex items-center justify-between">

&#x20;             <span className="text-sm text-gray-600">예약 1건</span>

&#x20;             <span className="text-lg font-bold text-gray-900">

&#x20;               {RESERVATION\_AMOUNT.toLocaleString()}원

&#x20;             </span>

&#x20;           </div>

&#x20;           <div className="my-3 border-t border-gray-100" />

&#x20;           <p className="text-center text-xs text-gray-400">

&#x20;             짐 개수와 무관하게 동일한 요금이 적용됩니다.

&#x20;           </p>

&#x20;         </div>

&#x20;       </section>

&#x20;     </div>



&#x20;     {/\* 다음 버튼 고정 \*/}

&#x20;     <div className="fixed bottom-0 left-0 right-0 mx-auto max-w-\[430px] bg-white px-4 pb-safe pt-3 shadow-\[0\_-1px\_8px\_rgba(0,0,0,0.06)]">

&#x20;       <button

&#x20;         onClick={handleNext}

&#x20;         disabled={!bagCount}

&#x20;         className="

&#x20;           w-full rounded-2xl py-4 text-base font-bold transition-colors

&#x20;           disabled:bg-gray-200 disabled:text-gray-400

&#x20;           enabled:bg-blue-600 enabled:text-white enabled:active:bg-blue-700

&#x20;         "

&#x20;       >

&#x20;         다음으로

&#x20;       </button>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 5. `src/app/reserve/step3/page.tsx`



```typescript

'use client'



import { useState, useEffect, useRef } from 'react'

import { useRouter } from 'next/navigation'

import StepBar from '@/components/fo/StepBar'

import { reservationStore } from '@/lib/reservation-store'



// 전화번호 자동 포맷: "01012345678" → "010-1234-5678"

function formatPhone(raw: string): string {

&#x20; const digits = raw.replace(/\\D/g, '').slice(0, 11)

&#x20; if (digits.length <= 3) return digits

&#x20; if (digits.length <= 7) return `${digits.slice(0, 3)}-${digits.slice(3)}`

&#x20; return `${digits.slice(0, 3)}-${digits.slice(3, 7)}-${digits.slice(7)}`

}



// 전화번호 유효성: 01X 시작 11자리

function isValidPhone(raw: string): boolean {

&#x20; const digits = raw.replace(/\\D/g, '')

&#x20; return /^01\[0-9]\\d{8}$/.test(digits)

}



export default function Step3Page() {

&#x20; const router    = useRouter()

&#x20; const draft     = reservationStore.get()

&#x20; const phoneRef  = useRef<HTMLInputElement>(null)



&#x20; useEffect(() => {

&#x20;   if (!draft.terminal || !draft.direction || !draft.use\_date || !draft.bag\_count) {

&#x20;     router.replace('/reserve/step1')

&#x20;   }

&#x20; }, \[])



&#x20; const \[name,         setName]         = useState(draft.customer\_name ?? '')

&#x20; const \[phoneDisplay, setPhoneDisplay] = useState(

&#x20;   draft.customer\_phone

&#x20;     ? formatPhone(draft.customer\_phone)

&#x20;     : ''

&#x20; )

&#x20; const \[nameError,   setNameError]   = useState('')

&#x20; const \[phoneError,  setPhoneError]  = useState('')

&#x20; const \[nameTouched, setNameTouched] = useState(false)

&#x20; const \[phoneTouched, setPhoneTouched] = useState(false)



&#x20; const phoneDigits  = phoneDisplay.replace(/\\D/g, '')

&#x20; const nameValid    = name.trim().length >= 1

&#x20; const phoneValid   = isValidPhone(phoneDigits)

&#x20; const canProceed   = nameValid \&\& phoneValid



&#x20; function handlePhoneChange(e: React.ChangeEvent<HTMLInputElement>) {

&#x20;   const formatted = formatPhone(e.target.value)

&#x20;   setPhoneDisplay(formatted)

&#x20; }



&#x20; function validateName() {

&#x20;   if (!name.trim()) setNameError('이름을 입력해 주세요.')

&#x20;   else setNameError('')

&#x20; }



&#x20; function validatePhone() {

&#x20;   if (!phoneDigits) setPhoneError('휴대폰 번호를 입력해 주세요.')

&#x20;   else if (!isValidPhone(phoneDigits)) setPhoneError('올바른 휴대폰 번호를 입력해 주세요.')

&#x20;   else setPhoneError('')

&#x20; }



&#x20; function handleNext() {

&#x20;   if (!canProceed) return

&#x20;   reservationStore.set({

&#x20;     customer\_name:  name.trim(),

&#x20;     customer\_phone: phoneDigits,

&#x20;   })

&#x20;   router.push('/reserve/step4')

&#x20; }



&#x20; return (

&#x20;   <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-gray-50">

&#x20;     <StepBar current={3} onBack={() => router.push('/reserve/step2')} />



&#x20;     <div className="flex-1 px-4 pb-32 pt-6">

&#x20;       <h2 className="mb-6 text-base font-semibold text-gray-900">

&#x20;         예약자 정보를 입력해 주세요

&#x20;       </h2>



&#x20;       <div className="space-y-5">

&#x20;         {/\* 이름 \*/}

&#x20;         <div>

&#x20;           <label htmlFor="name" className="mb-1.5 block text-sm font-medium text-gray-700">

&#x20;             이름

&#x20;           </label>

&#x20;           <input

&#x20;             id="name"

&#x20;             type="text"

&#x20;             value={name}

&#x20;             onChange={(e) => setName(e.target.value)}

&#x20;             onBlur={() => { setNameTouched(true); validateName() }}

&#x20;             placeholder="홍길동"

&#x20;             maxLength={20}

&#x20;             autoComplete="name"

&#x20;             returnKeyType="next"

&#x20;             onKeyDown={(e) => {

&#x20;               if (e.key === 'Enter') phoneRef.current?.focus()

&#x20;             }}

&#x20;             className={`

&#x20;               w-full rounded-xl border bg-white px-4 py-3.5

&#x20;               text-base text-gray-900 placeholder:text-gray-400

&#x20;               focus:outline-none focus:ring-2

&#x20;               ${nameTouched \&\& nameError

&#x20;                 ? 'border-red-400 focus:border-red-400 focus:ring-red-400/20'

&#x20;                 : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500/20'

&#x20;               }

&#x20;             `}

&#x20;           />

&#x20;           {nameTouched \&\& nameError \&\& (

&#x20;             <p className="mt-1.5 text-xs text-red-500">{nameError}</p>

&#x20;           )}

&#x20;         </div>



&#x20;         {/\* 휴대폰 번호 \*/}

&#x20;         <div>

&#x20;           <label htmlFor="phone" className="mb-1.5 block text-sm font-medium text-gray-700">

&#x20;             휴대폰 번호

&#x20;           </label>

&#x20;           <input

&#x20;             ref={phoneRef}

&#x20;             id="phone"

&#x20;             type="tel"

&#x20;             inputMode="numeric"

&#x20;             value={phoneDisplay}

&#x20;             onChange={handlePhoneChange}

&#x20;             onBlur={() => { setPhoneTouched(true); validatePhone() }}

&#x20;             placeholder="010-0000-0000"

&#x20;             autoComplete="tel"

&#x20;             className={`

&#x20;               w-full rounded-xl border bg-white px-4 py-3.5

&#x20;               text-base text-gray-900 placeholder:text-gray-400

&#x20;               focus:outline-none focus:ring-2

&#x20;               ${phoneTouched \&\& phoneError

&#x20;                 ? 'border-red-400 focus:border-red-400 focus:ring-red-400/20'

&#x20;                 : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500/20'

&#x20;               }

&#x20;             `}

&#x20;           />

&#x20;           {phoneTouched \&\& phoneError ? (

&#x20;             <p className="mt-1.5 text-xs text-red-500">{phoneError}</p>

&#x20;           ) : (

&#x20;             <p className="mt-1.5 text-xs text-gray-400">

&#x20;               입력하신 번호로 예약 확인 SMS가 발송됩니다.

&#x20;             </p>

&#x20;           )}

&#x20;         </div>

&#x20;       </div>

&#x20;     </div>



&#x20;     {/\* 다음 버튼 고정 \*/}

&#x20;     <div className="fixed bottom-0 left-0 right-0 mx-auto max-w-\[430px] bg-white px-4 pb-safe pt-3 shadow-\[0\_-1px\_8px\_rgba(0,0,0,0.06)]">

&#x20;       <button

&#x20;         onClick={handleNext}

&#x20;         disabled={!canProceed}

&#x20;         className="

&#x20;           w-full rounded-2xl py-4 text-base font-bold transition-colors

&#x20;           disabled:bg-gray-200 disabled:text-gray-400

&#x20;           enabled:bg-blue-600 enabled:text-white enabled:active:bg-blue-700

&#x20;         "

&#x20;       >

&#x20;         다음으로

&#x20;       </button>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 6. `src/app/reserve/step4/page.tsx`



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



// 토스페이먼츠 SDK 타입 (window 확장)

declare global {

&#x20; interface Window {

&#x20;   TossPayments?: (clientKey: string) => {

&#x20;     requestPayment: (

&#x20;       method: string,

&#x20;       options: Record<string, unknown>

&#x20;     ) => Promise<{

&#x20;       paymentKey: string

&#x20;       orderId:    string

&#x20;       amount:     number

&#x20;     }>

&#x20;   }

&#x20; }

}



// 예약 요약 행

function SummaryRow({ label, value }: { label: string; value: string }) {

&#x20; return (

&#x20;   <div className="flex items-start justify-between gap-4 py-2.5">

&#x20;     <span className="shrink-0 text-sm text-gray-500">{label}</span>

&#x20;     <span className="text-right text-sm font-medium text-gray-900">{value}</span>

&#x20;   </div>

&#x20; )

}



// 결제 실패 모달

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



&#x20; // 필수 정보 없으면 처음으로

&#x20; useEffect(() => {

&#x20;   const { terminal, direction, use\_date, bag\_count, customer\_name, customer\_phone } = draft

&#x20;   if (!terminal || !direction || !use\_date || !bag\_count || !customer\_name || !customer\_phone) {

&#x20;     router.replace('/reserve/step1')

&#x20;   }

&#x20; }, \[])



&#x20; const \[isLoading,   setIsLoading]   = useState(false)

&#x20; const \[errorMsg,    setErrorMsg]     = useState<string | null>(null)

&#x20; const \[sdkReady,    setSdkReady]    = useState(false)



&#x20; // 이용 날짜 포맷 (2026.07.15(수))

&#x20; const dateLabel = draft.use\_date

&#x20;   ? format(parseISO(draft.use\_date), 'yyyy.MM.dd(eee)', { locale: ko })

&#x20;   : ''



&#x20; // 전화번호 표시 포맷

&#x20; const phoneDisplay = draft.customer\_phone

&#x20;   ? draft.customer\_phone.replace(/(\\d{3})(\\d{4})(\\d{4})/, '$1-$2-$3')

&#x20;   : ''



&#x20; async function handlePayment() {

&#x20;   if (isLoading || !sdkReady) return

&#x20;   setIsLoading(true)

&#x20;   setErrorMsg(null)



&#x20;   try {

&#x20;     // ── 1. prepare API 호출 (toss\_order\_id 발급) ──────────

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



&#x20;     // ── 2. 토스페이먼츠 SDK 결제창 호출 ────────────────────

&#x20;     if (!window.TossPayments) {

&#x20;       throw new Error('결제 모듈을 불러오지 못했습니다. 잠시 후 다시 시도해 주세요.')

&#x20;     }



&#x20;     const toss    = window.TossPayments(process.env.NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY!)

&#x20;     const payment = await toss.requestPayment('카드', {

&#x20;       amount,

&#x20;       orderId:     toss\_order\_id,

&#x20;       orderName:   order\_name,

&#x20;       customerName: draft.customer\_name,

&#x20;       // 결제 완료 후 토스페이가 리디렉트할 성공/실패 URL

&#x20;       // (리디렉트 방식 대신 프로미스 방식 사용)

&#x20;     })



&#x20;     // ── 3. confirm API 호출 (결제 승인 + 예약 확정) ─────────

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



&#x20;     // ── 4. 완료 화면으로 이동 (sessionStorage 초기화) ───────

&#x20;     const reservationNo = confirmJson.data.reservation\_no

&#x20;     reservationStore.clear()



&#x20;     // replace 사용: 완료 화면에서 뒤로가기 시 홈으로 이동하도록

&#x20;     router.replace(`/reserve/complete?id=${reservationNo}`)



&#x20;   } catch (err) {

&#x20;     // 사용자가 결제창을 직접 닫은 경우 (토스페이 SDK에서 특정 에러 코드 발생)

&#x20;     if (err instanceof Error \&\& err.message.includes('PAY\_PROCESS\_CANCELED')) {

&#x20;       // 취소는 조용히 처리 (모달 미표시)

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

&#x20;     {/\* 토스페이먼츠 SDK 로드 \*/}

&#x20;     <Script

&#x20;       src="https://js.tosspayments.com/v1/payment"

&#x20;       onLoad={() => setSdkReady(true)}

&#x20;       strategy="afterInteractive"

&#x20;     />



&#x20;     <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col bg-gray-50">

&#x20;       <StepBar current={4} onBack={() => router.push('/reserve/step3')} />



&#x20;       <div className="flex-1 space-y-5 px-4 pb-40 pt-4">

&#x20;         <h2 className="text-base font-semibold text-gray-900">예약 내용을 확인해 주세요</h2>



&#x20;         {/\* 예약 요약 카드 \*/}

&#x20;         <div className="divide-y divide-gray-100 rounded-2xl bg-white px-5 shadow-sm">

&#x20;           <SummaryRow label="터미널"   value={draft.terminal ? TERMINAL\_LABEL\[draft.terminal] ?? draft.terminal : '-'} />

&#x20;           <SummaryRow label="이용 방향" value={draft.direction ? DIRECTION\_LABEL\[draft.direction] ?? draft.direction : '-'} />

&#x20;           <SummaryRow label="이용 날짜" value={dateLabel} />

&#x20;           <SummaryRow label="짐 개수"  value={`${draft.bag\_count ?? '-'}개`} />

&#x20;           <SummaryRow label="예약자"   value={draft.customer\_name ?? '-'} />

&#x20;           <SummaryRow label="연락처"   value={phoneDisplay} />

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



&#x20;         {/\* 결제 수단 \*/}

&#x20;         <div>

&#x20;           <h3 className="mb-3 text-sm font-semibold text-gray-700">결제 수단 선택</h3>

&#x20;           <button

&#x20;             onClick={handlePayment}

&#x20;             disabled={isLoading || !sdkReady}

&#x20;             className="

&#x20;               flex w-full items-center justify-center gap-3

&#x20;               rounded-2xl bg-blue-600 py-4 text-base font-bold text-white

&#x20;               shadow-sm active:bg-blue-700

&#x20;               disabled:cursor-not-allowed disabled:bg-gray-300

&#x20;             "

&#x20;           >

&#x20;             {isLoading ? (

&#x20;               <>

&#x20;                 <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;                 <span>결제 처리 중…</span>

&#x20;               </>

&#x20;             ) : (

&#x20;               <>

&#x20;                 <span className="text-xl">💳</span>

&#x20;                 <span>토스페이로 결제하기</span>

&#x20;               </>

&#x20;             )}

&#x20;           </button>

&#x20;         </div>



&#x20;         {/\* 약관 동의 안내 \*/}

&#x20;         <p className="text-center text-xs text-gray-400">

&#x20;           결제 시{' '}

&#x20;           <a href="/terms" className="underline">이용약관</a>

&#x20;           {' '}및{' '}

&#x20;           <a href="/privacy" className="underline">개인정보처리방침</a>

&#x20;           에 동의한 것으로 간주합니다.

&#x20;         </p>

&#x20;       </div>

&#x20;     </div>



&#x20;     {/\* 결제 실패 모달 \*/}

&#x20;     {errorMsg \&\& (

&#x20;       <PaymentErrorModal

&#x20;         message={errorMsg}

&#x20;         onClose={() => setErrorMsg(null)}

&#x20;       />

&#x20;     )}

&#x20;   </>

&#x20; )

}

```



\---



\## 7. `src/components/fo/ReservationQR.tsx`



```typescript

'use client'



import { QRCodeSVG } from 'qrcode.react'



interface ReservationQRProps {

&#x20; value: string    // QR에 인코딩할 데이터 (예약번호)

&#x20; size?: number    // QR 크기 (default: 220)

}



export default function ReservationQR({ value, size = 220 }: ReservationQRProps) {

&#x20; return (

&#x20;   <div

&#x20;     className="inline-flex items-center justify-center rounded-2xl bg-white p-5 shadow-sm"

&#x20;     style={{ minWidth: size + 40 }}

&#x20;   >

&#x20;     <QRCodeSVG

&#x20;       value={value}

&#x20;       size={size}

&#x20;       level="M"                 // 오류 정정 레벨 M (15%) - 스캔 안정성과 복잡도 균형

&#x20;       includeMargin={true}      // quiet zone 포함

&#x20;       bgColor="#FFFFFF"         // 배경: 흰색 (다크모드 강제 방지)

&#x20;       fgColor="#111111"         // QR 픽셀: 거의 검정 (스캔 인식률 최대화)

&#x20;     />

&#x20;   </div>

&#x20; )

}

```



\---



\## 8. `src/app/reserve/complete/page.tsx`



```typescript

'use client'



import { useEffect, useState, useCallback } from 'react'

import { useRouter, useSearchParams } from 'next/navigation'

import Link from 'next/link'

import ReservationQR from '@/components/fo/ReservationQR'

import { TERMINAL\_LABEL, DIRECTION\_LABEL } from '@/constants'

import { format, parseISO } from 'date-fns'

import { ko } from 'date-fns/locale'



interface ReservationData {

&#x20; reservation\_no: string

&#x20; terminal:       'T1' | 'T2'

&#x20; direction:      'DEPARTURE' | 'ARRIVAL'

&#x20; use\_date:       string

&#x20; bag\_count:      number

}



// 복사 완료 토스트

function CopyToast({ visible }: { visible: boolean }) {

&#x20; return (

&#x20;   <div

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



export default function ReserveCompletePage() {

&#x20; const router      = useRouter()

&#x20; const searchParams = useSearchParams()

&#x20; const reservationNo = searchParams.get('id')



&#x20; const \[data,       setData]       = useState<ReservationData | null>(null)

&#x20; const \[isLoading,  setIsLoading]  = useState(true)

&#x20; const \[error,      setError]      = useState<string | null>(null)

&#x20; const \[copyToast,  setCopyToast]  = useState(false)

&#x20; const \[canShare,   setCanShare]   = useState(false)



&#x20; // 브라우저 백버튼 → 홈으로 리디렉트 (히스토리 교체)

&#x20; useEffect(() => {

&#x20;   // popstate 이벤트로 뒤로가기 감지

&#x20;   function handlePopState() {

&#x20;     router.replace('/')

&#x20;   }

&#x20;   window.addEventListener('popstate', handlePopState)

&#x20;   return () => window.removeEventListener('popstate', handlePopState)

&#x20; }, \[router])



&#x20; // Web Share API 지원 여부 확인

&#x20; useEffect(() => {

&#x20;   setCanShare(typeof navigator !== 'undefined' \&\& !!navigator.share)

&#x20; }, \[])



&#x20; // 예약 정보 조회

&#x20; useEffect(() => {

&#x20;   if (!reservationNo) {

&#x20;     router.replace('/')

&#x20;     return

&#x20;   }



&#x20;   async function fetchReservation() {

&#x20;     try {

&#x20;       const res  = await fetch(`/api/reservations/${reservationNo}`)

&#x20;       const json = await res.json()

&#x20;       if (!res.ok || !json.success) {

&#x20;         throw new Error(json.error?.message ?? '예약 정보를 불러오지 못했습니다.')

&#x20;       }

&#x20;       setData(json.data as ReservationData)

&#x20;     } catch (err) {

&#x20;       setError(err instanceof Error ? err.message : '오류가 발생했습니다.')

&#x20;     } finally {

&#x20;       setIsLoading(false)

&#x20;     }

&#x20;   }



&#x20;   fetchReservation()

&#x20; }, \[reservationNo, router])



&#x20; // 예약번호 클립보드 복사

&#x20; const handleCopy = useCallback(async () => {

&#x20;   if (!reservationNo) return

&#x20;   try {

&#x20;     await navigator.clipboard.writeText(reservationNo)

&#x20;     setCopyToast(true)

&#x20;     setTimeout(() => setCopyToast(false), 2000)

&#x20;   } catch {

&#x20;     // 클립보드 API 미지원 브라우저 대응

&#x20;   }

&#x20; }, \[reservationNo])



&#x20; // 공유하기 (Web Share API)

&#x20; const handleShare = useCallback(async () => {

&#x20;   if (!reservationNo) return

&#x20;   try {

&#x20;     await navigator.share({

&#x20;       title: 'BagDrop 예약 확인',

&#x20;       text:  `예약번호: ${reservationNo}`,

&#x20;       url:   `${window.location.origin}/my-reservation/${reservationNo}`,

&#x20;     })

&#x20;   } catch {

&#x20;     // 사용자가 공유 취소한 경우 무시

&#x20;   }

&#x20; }, \[reservationNo])



&#x20; // 날짜 포맷

&#x20; const dateLabel = data?.use\_date

&#x20;   ? format(parseISO(data.use\_date), 'yyyy.MM.dd(eee)', { locale: ko })

&#x20;   : ''



&#x20; // ── 로딩 화면 ───────────────────────────────────────────────

&#x20; if (isLoading) {

&#x20;   return (

&#x20;     <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col items-center justify-center gap-4 bg-white">

&#x20;       <div className="h-10 w-10 animate-spin rounded-full border-2 border-gray-200 border-t-blue-600" />

&#x20;       <p className="text-sm text-gray-400">예약 확인 중…</p>

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; // ── 오류 화면 ───────────────────────────────────────────────

&#x20; if (error || !data) {

&#x20;   return (

&#x20;     <div className="mx-auto flex min-h-svh max-w-\[430px] flex-col items-center justify-center gap-4 bg-white px-5">

&#x20;       <div className="text-4xl">😢</div>

&#x20;       <p className="text-center text-sm text-gray-600">{error ?? '예약 정보를 찾을 수 없습니다.'}</p>

&#x20;       <Link href="/" className="mt-2 rounded-xl bg-blue-600 px-6 py-3 text-sm font-semibold text-white">

&#x20;         홈으로 돌아가기

&#x20;       </Link>

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; // ── 완료 화면 ───────────────────────────────────────────────

&#x20; return (

&#x20;   <div className="mx-auto min-h-svh max-w-\[430px] bg-white">



&#x20;     {/\* 복사 토스트 \*/}

&#x20;     <CopyToast visible={copyToast} />



&#x20;     {/\* GNB — 로고만 (뒤로가기 없음) \*/}

&#x20;     <header className="flex h-14 items-center justify-center border-b border-gray-100">

&#x20;       <span className="text-lg font-bold text-gray-900">BagDrop</span>

&#x20;     </header>



&#x20;     <div className="px-5 pb-16 pt-8">



&#x20;       {/\* 완료 메시지 \*/}

&#x20;       <div className="mb-8 text-center">

&#x20;         <div className="mb-3 text-5xl">✅</div>

&#x20;         <h1 className="text-xl font-bold text-gray-900">예약이 완료됐어요!</h1>

&#x20;         <p className="mt-2 text-sm text-gray-500">

&#x20;           부스에서 아래 QR 코드를 보여주세요

&#x20;         </p>

&#x20;       </div>



&#x20;       {/\* 예약번호 — 탭하면 복사 \*/}

&#x20;       <button

&#x20;         onClick={handleCopy}

&#x20;         className="mb-5 w-full rounded-2xl bg-gray-50 py-4 active:bg-gray-100"

&#x20;         aria-label="예약번호 복사"

&#x20;       >

&#x20;         <p className="text-xs text-gray-400">예약번호 (탭하면 복사)</p>

&#x20;         <p className="mt-1 text-xl font-bold tracking-widest text-blue-600">

&#x20;           {reservationNo}

&#x20;         </p>

&#x20;       </button>



&#x20;       {/\* QR 코드 — 중앙 정렬, 충분한 크기 \*/}

&#x20;       <div className="mb-5 flex justify-center">

&#x20;         <ReservationQR value={reservationNo ?? ''} size={240} />

&#x20;       </div>



&#x20;       {/\* 예약 정보 요약 \*/}

&#x20;       <div className="mb-6 rounded-2xl border border-gray-100 bg-gray-50 px-5 py-4">

&#x20;         <h2 className="mb-3 text-xs font-semibold uppercase tracking-wide text-gray-400">

&#x20;           예약 정보

&#x20;         </h2>

&#x20;         <div className="space-y-2">

&#x20;           {\[

&#x20;             { label: '터미널',   value: TERMINAL\_LABEL\[data.terminal]  ?? data.terminal },

&#x20;             { label: '방향',     value: DIRECTION\_LABEL\[data.direction] ?? data.direction },

&#x20;             { label: '날짜',     value: dateLabel },

&#x20;             { label: '짐 개수',  value: `${data.bag\_count}개` },

&#x20;           ].map(({ label, value }) => (

&#x20;             <div key={label} className="flex justify-between text-sm">

&#x20;               <span className="text-gray-500">{label}</span>

&#x20;               <span className="font-medium text-gray-900">{value}</span>

&#x20;             </div>

&#x20;           ))}

&#x20;         </div>

&#x20;       </div>



&#x20;       {/\* 액션 버튼 \*/}

&#x20;       <div className="space-y-3">

&#x20;         {/\* 공유 or 저장 안내 \*/}

&#x20;         {canShare ? (

&#x20;           <button

&#x20;             onClick={handleShare}

&#x20;             className="flex w-full items-center justify-center gap-2 rounded-2xl border border-blue-200 bg-blue-50 py-4 text-sm font-semibold text-blue-700 active:bg-blue-100"

&#x20;           >

&#x20;             <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;               <path strokeLinecap="round" strokeLinejoin="round" d="M7.217 10.907a2.25 2.25 0 100 2.186m0-2.186c.18.324.283.696.283 1.093s-.103.77-.283 1.093m0-2.186l9.566-5.314m-9.566 7.5l9.566 5.314m0 0a2.25 2.25 0 103.935 2.186 2.25 2.25 0 00-3.935-2.186zm0-12.814a2.25 2.25 0 103.933-2.185 2.25 2.25 0 00-3.933 2.185z" />

&#x20;             </svg>

&#x20;             예약 내역 공유하기

&#x20;           </button>

&#x20;         ) : (

&#x20;           <div className="rounded-2xl border border-gray-200 bg-gray-50 px-4 py-3 text-center text-xs text-gray-400">

&#x20;             화면을 캡처해 저장하거나, 예약 확인 페이지에서 다시 확인할 수 있습니다.

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* 홈으로 \*/}

&#x20;         <Link

&#x20;           href="/"

&#x20;           className="block text-center text-sm font-medium text-gray-400 underline-offset-2 hover:underline"

&#x20;         >

&#x20;           홈으로 돌아가기

&#x20;         </Link>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 설계 노트



\### sessionStorage 상태 관리 (`reservation-store.ts`)



STEP 간 URL 이동 시 React state는 초기화된다. `reservationStore`는 sessionStorage를 단일 키(`bagdrop\_reservation\_draft`)로 래핑해 전체 draft를 JSON으로 직렬화/역직렬화한다.



| 시점 | 동작 |

|------|------|

| STEP 진입 | `reservationStore.get()`으로 이전 값 복원 |

| STEP 이동 | `reservationStore.set(partial)`으로 부분 업데이트 |

| 결제 완료 | `reservationStore.clear()`로 전체 초기화 |

| 브라우저 새로고침 | STEP 1부터 재시작 (유효성 검증 누락 방지) |



각 STEP 페이지에서 필수 이전 값 부재 시 `/reserve/step1`로 `router.replace()` (히스토리 쌓지 않음).



\### 달력 — 외부 라이브러리 불사용



`react-datepicker` 같은 달력 라이브러리 대신 직접 구현했어요. 이유:

\- 번들 크기 절감 (react-datepicker: \~100KB gzip)

\- 모바일 터치 최적화 불필요 (직접 구현으로 간결하게)

\- 90일 범위 제한, 오늘 점 표시 등 커스텀 로직이 많아 라이브러리 오버라이드 비용이 큼



\### 토스페이 SDK — Promise 방식 사용



토스페이 SDK는 리디렉트 방식과 Promise 방식 두 가지를 지원한다.

Promise 방식(`await toss.requestPayment(...)`)을 선택한 이유:

\- 결제 완료 후 `paymentKey`, `orderId`를 즉시 받아 `confirm` API 호출 가능

\- 리디렉트 방식은 `successUrl`/`failUrl` 별도 페이지가 필요해 구현 복잡도 증가



\### 완료 화면 — 뒤로가기 차단



```typescript

useEffect(() => {

&#x20; function handlePopState() { router.replace('/') }

&#x20; window.addEventListener('popstate', handlePopState)

&#x20; return () => window.removeEventListener('popstate', handlePopState)

}, \[router])

```



브라우저 백버튼 누를 시 `popstate` 이벤트를 잡아 홈으로 replace한다.

Next.js `router.replace('/')`는 히스토리를 교체하므로 다시 뒤로가기해도 결제 화면으로 돌아가지 않는다.



\### QR 코드 — `level="M"` 선택 이유



| Level | 오류 정정률 | QR 복잡도 | 선택 |

|-------|------------|----------|------|

| L | 7% | 낮음 | 오류에 취약 |

| \*\*M\*\* | \*\*15%\*\* | \*\*중간\*\* | \*\*✅ 균형\*\* |

| Q | 25% | 높음 | 스캔 불리 |

| H | 30% | 매우 높음 | 불필요 |



예약번호(`BD-20260715-0001`) 정도의 짧은 데이터는 M 레벨로도 충분하고, 운영자 스캐너 스캔 시 인식률이 높다.



\---



\## Session 10 체크리스트



| 파일 | 구현 항목 | 상태 |

|------|----------|------|

| `reservation-store.ts` | get/set/clear + SSR 가드 | ✅ |

| `page.tsx` (홈) | 히어로/이용방법/초과안내/CTA | ✅ |

| `StepBar.tsx` | 진행률 바 + 뒤로가기 + aria | ✅ |

| `step1/page.tsx` | 터미널/방향 카드 선택 + 미니 달력 + 90일 제한 | ✅ |

| `step2/page.tsx` | 짐 개수 카드 + 초과 안내 + 요금 표시 | ✅ |

| `step3/page.tsx` | 이름/전화번호 + 실시간 포맷 + 블러 유효성 + 키보드 포커스 이동 | ✅ |

| `step4/page.tsx` | 예약 요약 + 토스페이 SDK + prepare→결제→confirm 흐름 + 실패 모달 | ✅ |

| `ReservationQR.tsx` | qrcode.react SVG + 흰 배경 강제 + quiet zone | ✅ |

| `complete/page.tsx` | QR 표시 + 예약번호 복사 + Web Share API + 뒤로가기 차단 + 예약 조회 | ✅ |



\---



\*다음 세션: Session 11 — FO 예약 확인/상세 페이지 + my-reservation 구현\*

