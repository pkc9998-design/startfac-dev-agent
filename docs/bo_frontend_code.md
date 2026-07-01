\# BagDrop BO 프론트엔드 구현 코드



> 작성: 개발 에이전트 가온

> 세션: Session 09

> 최종 업데이트: 2026-07-01

> 대상: NextAuth 핸들러, 로그인, 공통 레이아웃, 대시보드 (5개 파일)

> 기준: 모바일웹 max-width 430px / Tailwind CSS

> 상태: ✅ Session 09 확정



\---



\## 파일 목록



| 파일 경로 | 역할 |

|-----------|------|

| `src/app/api/admin/auth/\[...nextauth]/route.ts` | NextAuth App Router 핸들러 |

| `src/app/admin/login/page.tsx` | 로그인 페이지 |

| `src/components/bo/BottomNav.tsx` | 하단 탭 네비게이션 |

| `src/components/bo/AdminLayout.tsx` | BO 공통 레이아웃 (GNB + BottomNav) |

| `src/app/admin/dashboard/page.tsx` | 대시보드 페이지 |



\---



\## 1. `src/app/api/admin/auth/\[...nextauth]/route.ts`



```typescript

// Next.js 14 App Router에서 NextAuth.js를 사용하기 위한 핸들러

// GET: 세션 조회, 로그인 페이지 렌더 등

// POST: 로그인(credentials), 로그아웃 처리



import NextAuth from 'next-auth'

import { authOptions } from '@/lib/auth'



const handler = NextAuth(authOptions)



export { handler as GET, handler as POST }

```



\---



\## 2. `src/app/admin/login/page.tsx`



```typescript

'use client'



import { useState, useEffect, FormEvent } from 'react'

import { signIn, useSession } from 'next-auth/react'

import { useRouter, useSearchParams } from 'next/navigation'



// 에러 코드 → 사용자 메시지 매핑

const ERROR\_MESSAGES: Record<string, string> = {

&#x20; CredentialsSignin: '아이디 또는 비밀번호가 올바르지 않습니다.',

&#x20; SessionRequired:   '로그인이 필요합니다.',

&#x20; Default:           '로그인 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.',

}



export default function AdminLoginPage() {

&#x20; const router        = useRouter()

&#x20; const searchParams  = useSearchParams()

&#x20; const { status }    = useSession()



&#x20; const \[username,      setUsername]      = useState('')

&#x20; const \[password,      setPassword]      = useState('')

&#x20; const \[showPassword,  setShowPassword]  = useState(false)

&#x20; const \[isLoading,     setIsLoading]     = useState(false)

&#x20; const \[errorMsg,      setErrorMsg]      = useState<string | null>(null)



&#x20; // URL의 error 파라미터 처리 (NextAuth 리디렉트 시 붙는 ?error=...)

&#x20; useEffect(() => {

&#x20;   const errorCode = searchParams.get('error')

&#x20;   if (errorCode) {

&#x20;     setErrorMsg(ERROR\_MESSAGES\[errorCode] ?? ERROR\_MESSAGES.Default)

&#x20;   }

&#x20; }, \[searchParams])



&#x20; // 이미 로그인된 경우 대시보드로 이동

&#x20; useEffect(() => {

&#x20;   if (status === 'authenticated') {

&#x20;     const callbackUrl = searchParams.get('callbackUrl') ?? '/admin/dashboard'

&#x20;     router.replace(callbackUrl)

&#x20;   }

&#x20; }, \[status, router, searchParams])



&#x20; async function handleSubmit(e: FormEvent<HTMLFormElement>) {

&#x20;   e.preventDefault()

&#x20;   if (isLoading) return



&#x20;   setIsLoading(true)

&#x20;   setErrorMsg(null)



&#x20;   const result = await signIn('credentials', {

&#x20;     username: username.trim(),

&#x20;     password,

&#x20;     redirect: false,   // 수동으로 리디렉트 처리

&#x20;   })



&#x20;   setIsLoading(false)



&#x20;   if (result?.error) {

&#x20;     setErrorMsg(ERROR\_MESSAGES\[result.error] ?? ERROR\_MESSAGES.Default)

&#x20;     // 보안: 비밀번호 필드 초기화

&#x20;     setPassword('')

&#x20;     return

&#x20;   }



&#x20;   if (result?.ok) {

&#x20;     const callbackUrl = searchParams.get('callbackUrl') ?? '/admin/dashboard'

&#x20;     router.replace(callbackUrl)

&#x20;   }

&#x20; }



&#x20; // 세션 로딩 중에는 빈 화면 (깜빡임 방지)

&#x20; if (status === 'loading') {

&#x20;   return (

&#x20;     <div className="flex min-h-screen items-center justify-center bg-gray-50">

&#x20;       <div className="h-8 w-8 animate-spin rounded-full border-2 border-gray-300 border-t-blue-600" />

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; return (

&#x20;   <div className="flex min-h-screen flex-col items-center justify-center bg-gray-50 px-5">

&#x20;     <div className="w-full max-w-\[430px]">



&#x20;       {/\* ── 헤더 ─────────────────────────────────────── \*/}

&#x20;       <div className="mb-10 text-center">

&#x20;         <h1 className="text-2xl font-bold text-gray-900">BagDrop</h1>

&#x20;         <p className="mt-1 text-sm text-gray-500">어드민 로그인</p>

&#x20;       </div>



&#x20;       {/\* ── 로그인 폼 ─────────────────────────────────── \*/}

&#x20;       <form onSubmit={handleSubmit} noValidate className="space-y-4">



&#x20;         {/\* 아이디 \*/}

&#x20;         <div>

&#x20;           <label htmlFor="username" className="mb-1.5 block text-sm font-medium text-gray-700">

&#x20;             아이디

&#x20;           </label>

&#x20;           <input

&#x20;             id="username"

&#x20;             type="text"

&#x20;             value={username}

&#x20;             onChange={(e) => setUsername(e.target.value)}

&#x20;             placeholder="아이디를 입력해 주세요"

&#x20;             autoComplete="username"

&#x20;             autoCapitalize="none"

&#x20;             autoCorrect="off"

&#x20;             spellCheck={false}

&#x20;             required

&#x20;             disabled={isLoading}

&#x20;             className="

&#x20;               w-full rounded-xl border border-gray-300 bg-white px-4 py-3.5

&#x20;               text-base text-gray-900 placeholder:text-gray-400

&#x20;               focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-500/20

&#x20;               disabled:bg-gray-100 disabled:text-gray-400

&#x20;             "

&#x20;           />

&#x20;         </div>



&#x20;         {/\* 비밀번호 \*/}

&#x20;         <div>

&#x20;           <label htmlFor="password" className="mb-1.5 block text-sm font-medium text-gray-700">

&#x20;             비밀번호

&#x20;           </label>

&#x20;           <div className="relative">

&#x20;             <input

&#x20;               id="password"

&#x20;               type={showPassword ? 'text' : 'password'}

&#x20;               value={password}

&#x20;               onChange={(e) => setPassword(e.target.value)}

&#x20;               placeholder="비밀번호를 입력해 주세요"

&#x20;               autoComplete="current-password"

&#x20;               required

&#x20;               disabled={isLoading}

&#x20;               className="

&#x20;                 w-full rounded-xl border border-gray-300 bg-white px-4 py-3.5 pr-12

&#x20;                 text-base text-gray-900 placeholder:text-gray-400

&#x20;                 focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-500/20

&#x20;                 disabled:bg-gray-100 disabled:text-gray-400

&#x20;               "

&#x20;             />

&#x20;             {/\* 눈 아이콘 - 비밀번호 표시 토글 \*/}

&#x20;             <button

&#x20;               type="button"

&#x20;               onClick={() => setShowPassword((v) => !v)}

&#x20;               aria-label={showPassword ? '비밀번호 숨기기' : '비밀번호 보기'}

&#x20;               className="absolute right-3 top-1/2 -translate-y-1/2 p-1 text-gray-400 hover:text-gray-600"

&#x20;             >

&#x20;               {showPassword ? (

&#x20;                 // 눈 감은 아이콘

&#x20;                 <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={1.8}>

&#x20;                   <path strokeLinecap="round" strokeLinejoin="round" d="M3.98 8.223A10.477 10.477 0 001.934 12C3.226 16.338 7.244 19.5 12 19.5c.993 0 1.953-.138 2.863-.395M6.228 6.228A10.45 10.45 0 0112 4.5c4.756 0 8.773 3.162 10.065 7.498a10.523 10.523 0 01-4.293 5.774M6.228 6.228L3 3m3.228 3.228l3.65 3.65m7.894 7.894L21 21m-3.228-3.228l-3.65-3.65m0 0a3 3 0 10-4.243-4.243m4.242 4.242L9.88 9.88" />

&#x20;                 </svg>

&#x20;               ) : (

&#x20;                 // 눈 뜬 아이콘

&#x20;                 <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={1.8}>

&#x20;                   <path strokeLinecap="round" strokeLinejoin="round" d="M2.036 12.322a1.012 1.012 0 010-.639C3.423 7.51 7.36 4.5 12 4.5c4.638 0 8.573 3.007 9.963 7.178.07.207.07.431 0 .639C20.577 16.49 16.64 19.5 12 19.5c-4.638 0-8.573-3.007-9.963-7.178z" />

&#x20;                   <path strokeLinecap="round" strokeLinejoin="round" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />

&#x20;                 </svg>

&#x20;               )}

&#x20;             </button>

&#x20;           </div>

&#x20;         </div>



&#x20;         {/\* 오류 메시지 \*/}

&#x20;         {errorMsg \&\& (

&#x20;           <div

&#x20;             role="alert"

&#x20;             className="flex items-start gap-2 rounded-xl bg-red-50 px-4 py-3 text-sm text-red-700"

&#x20;           >

&#x20;             <svg className="mt-0.5 h-4 w-4 shrink-0" fill="currentColor" viewBox="0 0 20 20">

&#x20;               <path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-8-5a.75.75 0 01.75.75v4.5a.75.75 0 01-1.5 0v-4.5A.75.75 0 0110 5zm0 10a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" />

&#x20;             </svg>

&#x20;             <span>{errorMsg}</span>

&#x20;           </div>

&#x20;         )}



&#x20;         {/\* 로그인 버튼 \*/}

&#x20;         <button

&#x20;           type="submit"

&#x20;           disabled={isLoading || !username.trim() || !password}

&#x20;           className="

&#x20;             mt-2 flex w-full items-center justify-center gap-2

&#x20;             rounded-xl bg-blue-600 px-4 py-4

&#x20;             text-base font-semibold text-white

&#x20;             transition-colors hover:bg-blue-700

&#x20;             disabled:cursor-not-allowed disabled:bg-gray-300 disabled:text-gray-500

&#x20;           "

&#x20;         >

&#x20;           {isLoading ? (

&#x20;             <>

&#x20;               <span className="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white" />

&#x20;               <span>로그인 중…</span>

&#x20;             </>

&#x20;           ) : (

&#x20;             '로그인'

&#x20;           )}

&#x20;         </button>

&#x20;       </form>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}

```



\---



\## 3. `src/components/bo/BottomNav.tsx`



```typescript

'use client'



import Link from 'next/link'

import { usePathname } from 'next/navigation'



interface NavItem {

&#x20; href:  string

&#x20; label: string

&#x20; icon:  React.ReactNode

}



// SVG 아이콘 모음 (Heroicons Outline 기반)

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

&#x20; { href: '/admin/dashboard',    label: '홈',   icon: <HomeIcon /> },

&#x20; { href: '/admin/checkin',      label: '접수', icon: <CheckinIcon /> },

&#x20; { href: '/admin/checkout',     label: '반환', icon: <CheckoutIcon /> },

&#x20; { href: '/admin/storage',      label: '현황', icon: <StorageIcon /> },

&#x20; { href: '/admin/sales',        label: '매출', icon: <SalesIcon /> },

]



export default function BottomNav() {

&#x20; const pathname = usePathname()



&#x20; // 현재 경로와 nav item href 비교 (정확 매칭 또는 하위 경로 포함)

&#x20; function isActive(href: string): boolean {

&#x20;   if (href === '/admin/dashboard') return pathname === href

&#x20;   return pathname.startsWith(href)

&#x20; }



&#x20; return (

&#x20;   // safe-area-inset-bottom: 아이폰 홈 인디케이터 영역 대응

&#x20;   <nav className="fixed bottom-0 left-0 right-0 z-50 border-t border-gray-200 bg-white pb-safe">

&#x20;     <div className="mx-auto flex max-w-\[430px] items-stretch">

&#x20;       {NAV\_ITEMS.map((item) => {

&#x20;         const active = isActive(item.href)

&#x20;         return (

&#x20;           <Link

&#x20;             key={item.href}

&#x20;             href={item.href}

&#x20;             className={`

&#x20;               flex flex-1 flex-col items-center gap-1 px-1 py-2.5

&#x20;               transition-colors

&#x20;               ${active

&#x20;                 ? 'text-blue-600'

&#x20;                 : 'text-gray-400 hover:text-gray-600'

&#x20;               }

&#x20;             `}

&#x20;             aria-current={active ? 'page' : undefined}

&#x20;           >

&#x20;             {/\* 아이콘 \*/}

&#x20;             <span className={`${active ? 'text-blue-600' : 'text-gray-400'}`}>

&#x20;               {item.icon}

&#x20;             </span>

&#x20;             {/\* 레이블 \*/}

&#x20;             <span className={`text-\[11px] font-medium leading-none ${active ? 'text-blue-600' : 'text-gray-400'}`}>

&#x20;               {item.label}

&#x20;             </span>

&#x20;             {/\* 활성 인디케이터 도트 \*/}

&#x20;             {active \&\& (

&#x20;               <span className="absolute -top-px left-1/2 h-0.5 w-8 -translate-x-1/2 rounded-full bg-blue-600" />

&#x20;             )}

&#x20;           </Link>

&#x20;         )

&#x20;       })}

&#x20;     </div>

&#x20;   </nav>

&#x20; )

}

```



\---



\## 4. `src/components/bo/AdminLayout.tsx`



```typescript

'use client'



import { useSession, signOut } from 'next-auth/react'

import { useRouter } from 'next/navigation'

import { useEffect, useState } from 'react'

import BottomNav from '@/components/bo/BottomNav'



interface AdminLayoutProps {

&#x20; children:    React.ReactNode

&#x20; title?:      string      // 화면별 타이틀 (GNB에 표시)

&#x20; showBack?:   boolean     // 뒤로가기 버튼 표시 여부

&#x20; onBack?:     () => void  // 커스텀 뒤로가기 동작

}



// 로그아웃 확인 모달

function LogoutModal({

&#x20; onConfirm,

&#x20; onCancel,

}: {

&#x20; onConfirm: () => void

&#x20; onCancel:  () => void

}) {

&#x20; return (

&#x20;   <div className="fixed inset-0 z-\[100] flex items-center justify-center bg-black/50 px-5">

&#x20;     <div className="w-full max-w-\[320px] rounded-2xl bg-white p-6 shadow-xl">

&#x20;       <h3 className="text-center text-base font-semibold text-gray-900">로그아웃</h3>

&#x20;       <p className="mt-2 text-center text-sm text-gray-500">로그아웃 하시겠습니까?</p>

&#x20;       <div className="mt-5 flex gap-3">

&#x20;         <button

&#x20;           onClick={onCancel}

&#x20;           className="flex-1 rounded-xl border border-gray-300 py-3 text-sm font-medium text-gray-700 hover:bg-gray-50"

&#x20;         >

&#x20;           취소

&#x20;         </button>

&#x20;         <button

&#x20;           onClick={onConfirm}

&#x20;           className="flex-1 rounded-xl bg-red-500 py-3 text-sm font-medium text-white hover:bg-red-600"

&#x20;         >

&#x20;           로그아웃

&#x20;         </button>

&#x20;       </div>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// 세션 만료 모달

function SessionExpiredModal({ onConfirm }: { onConfirm: () => void }) {

&#x20; return (

&#x20;   <div className="fixed inset-0 z-\[100] flex items-center justify-center bg-black/50 px-5">

&#x20;     <div className="w-full max-w-\[320px] rounded-2xl bg-white p-6 shadow-xl">

&#x20;       <h3 className="text-center text-base font-semibold text-gray-900">로그인 만료</h3>

&#x20;       <p className="mt-2 text-center text-sm text-gray-500">

&#x20;         로그인이 만료됐습니다. 다시 로그인해 주세요.

&#x20;       </p>

&#x20;       <button

&#x20;         onClick={onConfirm}

&#x20;         className="mt-5 w-full rounded-xl bg-blue-600 py-3 text-sm font-semibold text-white hover:bg-blue-700"

&#x20;       >

&#x20;         확인

&#x20;       </button>

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



export default function AdminLayout({

&#x20; children,

&#x20; title,

&#x20; showBack = false,

&#x20; onBack,

}: AdminLayoutProps) {

&#x20; const { data: session, status } = useSession()

&#x20; const router = useRouter()



&#x20; const \[showLogoutModal,  setShowLogoutModal]  = useState(false)

&#x20; const \[showExpiredModal, setShowExpiredModal]  = useState(false)



&#x20; // 세션 만료 감지

&#x20; useEffect(() => {

&#x20;   if (status === 'unauthenticated') {

&#x20;     setShowExpiredModal(true)

&#x20;   }

&#x20; }, \[status])



&#x20; async function handleLogout() {

&#x20;   await signOut({ redirect: false })

&#x20;   router.replace('/admin/login')

&#x20; }



&#x20; function handleExpiredConfirm() {

&#x20;   setShowExpiredModal(false)

&#x20;   router.replace('/admin/login')

&#x20; }



&#x20; function handleBack() {

&#x20;   if (onBack) {

&#x20;     onBack()

&#x20;   } else {

&#x20;     router.back()

&#x20;   }

&#x20; }



&#x20; // 세션 로딩 중

&#x20; if (status === 'loading') {

&#x20;   return (

&#x20;     <div className="flex min-h-screen items-center justify-center bg-gray-50">

&#x20;       <div className="h-8 w-8 animate-spin rounded-full border-2 border-gray-300 border-t-blue-600" />

&#x20;     </div>

&#x20;   )

&#x20; }



&#x20; return (

&#x20;   <div className="relative mx-auto flex min-h-screen max-w-\[430px] flex-col bg-gray-50">



&#x20;     {/\* ── GNB ──────────────────────────────────────────── \*/}

&#x20;     <header className="sticky top-0 z-40 border-b border-gray-200 bg-white">

&#x20;       <div className="flex h-14 items-center px-4">



&#x20;         {/\* 왼쪽: 뒤로가기 OR 서비스명 \*/}

&#x20;         <div className="flex min-w-0 flex-1 items-center gap-2">

&#x20;           {showBack ? (

&#x20;             <button

&#x20;               onClick={handleBack}

&#x20;               aria-label="뒤로 가기"

&#x20;               className="-ml-1 rounded-lg p-1.5 text-gray-600 hover:bg-gray-100"

&#x20;             >

&#x20;               <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;                 <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 19.5L8.25 12l7.5-7.5" />

&#x20;               </svg>

&#x20;             </button>

&#x20;           ) : null}

&#x20;           <div className="min-w-0">

&#x20;             <span className="block truncate text-sm font-semibold text-gray-900">

&#x20;               {title ?? 'BagDrop 어드민'}

&#x20;             </span>

&#x20;           </div>

&#x20;         </div>



&#x20;         {/\* 오른쪽: 관리자명 + 로그아웃 \*/}

&#x20;         <div className="ml-2 flex items-center gap-1">

&#x20;           <span className="text-xs text-gray-500">

&#x20;             {session?.user?.displayName ?? '관리자'}

&#x20;           </span>

&#x20;           <button

&#x20;             onClick={() => setShowLogoutModal(true)}

&#x20;             aria-label="로그아웃"

&#x20;             className="rounded-lg p-1.5 text-gray-400 hover:bg-gray-100 hover:text-gray-600"

&#x20;           >

&#x20;             <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;               <path strokeLinecap="round" strokeLinejoin="round" d="M15.75 9V5.25A2.25 2.25 0 0013.5 3h-6a2.25 2.25 0 00-2.25 2.25v13.5A2.25 2.25 0 007.5 21h6a2.25 2.25 0 002.25-2.25V15m3 0l3-3m0 0l-3-3m3 3H9" />

&#x20;             </svg>

&#x20;           </button>

&#x20;         </div>

&#x20;       </div>

&#x20;     </header>



&#x20;     {/\* ── 메인 콘텐츠 ──────────────────────────────────── \*/}

&#x20;     {/\* pb-20: 하단 탭 높이(64px) + 여유 공간 \*/}

&#x20;     <main className="flex-1 overflow-y-auto pb-20">

&#x20;       {children}

&#x20;     </main>



&#x20;     {/\* ── 하단 탭 네비게이션 ──────────────────────────── \*/}

&#x20;     <BottomNav />



&#x20;     {/\* ── 모달 ─────────────────────────────────────────── \*/}

&#x20;     {showLogoutModal \&\& (

&#x20;       <LogoutModal

&#x20;         onConfirm={handleLogout}

&#x20;         onCancel={() => setShowLogoutModal(false)}

&#x20;       />

&#x20;     )}

&#x20;     {showExpiredModal \&\& (

&#x20;       <SessionExpiredModal onConfirm={handleExpiredConfirm} />

&#x20;     )}

&#x20;   </div>

&#x20; )

}

```



\---



\## 5. `src/app/admin/dashboard/page.tsx`



```typescript

'use client'



import { useState, useEffect, useCallback } from 'react'

import { useRouter } from 'next/navigation'

import AdminLayout from '@/components/bo/AdminLayout'

import { TERMINAL\_LABEL, DIRECTION\_LABEL } from '@/constants'



// ── 타입 ────────────────────────────────────────────────────────



type TerminalFilter = 'T1' | 'T2' | undefined



interface RecentItem {

&#x20; reservation\_no:  string

&#x20; terminal:        'T1' | 'T2'

&#x20; direction:       'DEPARTURE' | 'ARRIVAL'

&#x20; customer\_name:   string

&#x20; bag\_count:       number

&#x20; created\_at:      string

&#x20; elapsed\_minutes: number

}



interface DashboardData {

&#x20; today\_total:    number

&#x20; current\_stored: number

&#x20; today\_returned: number

&#x20; recent\_pending: RecentItem\[]

&#x20; as\_of:          string

}



// ── 서브 컴포넌트 ────────────────────────────────────────────────



// 현황 카드 스켈레톤

function StatCardSkeleton() {

&#x20; return (

&#x20;   <div className="flex-1 animate-pulse rounded-2xl bg-white p-4 shadow-sm">

&#x20;     <div className="mb-2 h-3 w-12 rounded bg-gray-200" />

&#x20;     <div className="h-8 w-10 rounded bg-gray-200" />

&#x20;   </div>

&#x20; )

}



// 현황 카드

function StatCard({

&#x20; label,

&#x20; value,

&#x20; color,

&#x20; onClick,

}: {

&#x20; label:   string

&#x20; value:   number

&#x20; color:   'blue' | 'green' | 'gray'

&#x20; onClick: () => void

}) {

&#x20; const colorMap = {

&#x20;   blue:  { bg: 'bg-blue-50',  text: 'text-blue-700',  num: 'text-blue-600' },

&#x20;   green: { bg: 'bg-green-50', text: 'text-green-700', num: 'text-green-600' },

&#x20;   gray:  { bg: 'bg-gray-50',  text: 'text-gray-600',  num: 'text-gray-700' },

&#x20; }

&#x20; const c = colorMap\[color]



&#x20; return (

&#x20;   <button

&#x20;     onClick={onClick}

&#x20;     className={`flex flex-1 flex-col items-start rounded-2xl ${c.bg} p-4 shadow-sm active:opacity-80`}

&#x20;   >

&#x20;     <span className={`text-xs font-medium ${c.text}`}>{label}</span>

&#x20;     <span className={`mt-1.5 text-3xl font-bold ${c.num}`}>{value}</span>

&#x20;     <span className={`mt-0.5 text-xs ${c.text}`}>건</span>

&#x20;   </button>

&#x20; )

}



// 경과 시간 포맷 (분 → "N분 전" or "N시간 M분")

function formatElapsed(minutes: number): string {

&#x20; if (minutes < 60) return `${minutes}분 전`

&#x20; const h = Math.floor(minutes / 60)

&#x20; const m = minutes % 60

&#x20; return m > 0 ? `${h}시간 ${m}분 전` : `${h}시간 전`

}



// 대기 예약 아이템 스켈레톤

function PendingItemSkeleton() {

&#x20; return (

&#x20;   <div className="animate-pulse px-4 py-3.5">

&#x20;     <div className="flex items-center justify-between">

&#x20;       <div className="space-y-1.5">

&#x20;         <div className="h-3 w-36 rounded bg-gray-200" />

&#x20;         <div className="h-3 w-28 rounded bg-gray-200" />

&#x20;       </div>

&#x20;       <div className="h-3 w-12 rounded bg-gray-200" />

&#x20;     </div>

&#x20;   </div>

&#x20; )

}



// ── 메인 페이지 ──────────────────────────────────────────────────



const POLL\_INTERVAL\_MS = 30\_000  // 30초 폴링



export default function DashboardPage() {

&#x20; const router = useRouter()



&#x20; const \[terminal,   setTerminal]   = useState<TerminalFilter>(undefined)

&#x20; const \[data,       setData]       = useState<DashboardData | null>(null)

&#x20; const \[isLoading,  setIsLoading]  = useState(true)

&#x20; const \[isRefreshing, setIsRefreshing] = useState(false)

&#x20; const \[error,      setError]      = useState<string | null>(null)

&#x20; const \[lastUpdated, setLastUpdated] = useState<Date | null>(null)



&#x20; // ── 데이터 패치 ────────────────────────────────────────────────

&#x20; const fetchDashboard = useCallback(async (showRefreshing = false) => {

&#x20;   if (showRefreshing) setIsRefreshing(true)



&#x20;   try {

&#x20;     const params = new URLSearchParams()

&#x20;     if (terminal) params.set('terminal', terminal)



&#x20;     const res  = await fetch(`/api/admin/dashboard?${params}`, {

&#x20;       // 30초 폴링이므로 캐시 없이 항상 최신 데이터

&#x20;       cache: 'no-store',

&#x20;     })

&#x20;     const json = await res.json()



&#x20;     if (!res.ok || !json.success) {

&#x20;       throw new Error(json.error?.message ?? '데이터를 불러오지 못했습니다.')

&#x20;     }



&#x20;     setData(json.data as DashboardData)

&#x20;     setError(null)

&#x20;     setLastUpdated(new Date())

&#x20;   } catch (err) {

&#x20;     setError(err instanceof Error ? err.message : '알 수 없는 오류가 발생했습니다.')

&#x20;   } finally {

&#x20;     setIsLoading(false)

&#x20;     setIsRefreshing(false)

&#x20;   }

&#x20; }, \[terminal])



&#x20; // 초기 로드 + 터미널 필터 변경 시

&#x20; useEffect(() => {

&#x20;   setIsLoading(true)

&#x20;   fetchDashboard()

&#x20; }, \[fetchDashboard])



&#x20; // 30초 폴링

&#x20; useEffect(() => {

&#x20;   const timer = setInterval(() => {

&#x20;     fetchDashboard(true)

&#x20;   }, POLL\_INTERVAL\_MS)

&#x20;   return () => clearInterval(timer)

&#x20; }, \[fetchDashboard])



&#x20; // ── 탭 이동 핸들러 ─────────────────────────────────────────────

&#x20; function goToReservations(status?: string) {

&#x20;   const params = new URLSearchParams()

&#x20;   if (status)   params.set('status', status)

&#x20;   if (terminal) params.set('terminal', terminal)

&#x20;   router.push(`/admin/reservations?${params}`)

&#x20; }



&#x20; // ── 마지막 업데이트 시각 포맷 ─────────────────────────────────

&#x20; function formatLastUpdated(): string {

&#x20;   if (!lastUpdated) return ''

&#x20;   return lastUpdated.toLocaleTimeString('ko-KR', {

&#x20;     hour:   '2-digit',

&#x20;     minute: '2-digit',

&#x20;     second: '2-digit',

&#x20;   })

&#x20; }



&#x20; // ── 오늘 날짜 ────────────────────────────────────────────────

&#x20; const todayStr = new Date().toLocaleDateString('ko-KR', {

&#x20;   year:    'numeric',

&#x20;   month:   'long',

&#x20;   day:     'numeric',

&#x20;   weekday: 'short',

&#x20; })



&#x20; return (

&#x20;   <AdminLayout title="BagDrop 어드민">

&#x20;     <div className="space-y-5 px-4 pt-5">



&#x20;       {/\* ── 날짜 + 마지막 갱신 시각 ─────────────────────── \*/}

&#x20;       <div className="flex items-end justify-between">

&#x20;         <div>

&#x20;           <p className="text-xs text-gray-400">{todayStr}</p>

&#x20;         </div>

&#x20;         <div className="flex items-center gap-1.5">

&#x20;           {isRefreshing \&\& (

&#x20;             <span className="h-3 w-3 animate-spin rounded-full border-2 border-gray-300 border-t-blue-500" />

&#x20;           )}

&#x20;           {lastUpdated \&\& (

&#x20;             <span className="text-\[11px] text-gray-400">{formatLastUpdated()} 기준</span>

&#x20;           )}

&#x20;         </div>

&#x20;       </div>



&#x20;       {/\* ── 터미널 필터 ──────────────────────────────────── \*/}

&#x20;       <div className="flex gap-2">

&#x20;         {(\['전체', 'T1', 'T2'] as const).map((t) => {

&#x20;           const value = t === '전체' ? undefined : t

&#x20;           const active = terminal === value

&#x20;           return (

&#x20;             <button

&#x20;               key={t}

&#x20;               onClick={() => setTerminal(value)}

&#x20;               className={`

&#x20;                 rounded-full px-4 py-1.5 text-sm font-medium transition-colors

&#x20;                 ${active

&#x20;                   ? 'bg-blue-600 text-white'

&#x20;                   : 'bg-white text-gray-600 shadow-sm hover:bg-gray-50'

&#x20;                 }

&#x20;               `}

&#x20;             >

&#x20;               {t === '전체' ? '전체' : TERMINAL\_LABEL\[t]}

&#x20;             </button>

&#x20;           )

&#x20;         })}

&#x20;       </div>



&#x20;       {/\* ── 오류 메시지 ──────────────────────────────────── \*/}

&#x20;       {error \&\& (

&#x20;         <div className="rounded-xl bg-red-50 px-4 py-3 text-sm text-red-700">

&#x20;           {error}

&#x20;         </div>

&#x20;       )}



&#x20;       {/\* ── 오늘 현황 카드 ───────────────────────────────── \*/}

&#x20;       <section>

&#x20;         <h2 className="mb-3 text-sm font-semibold text-gray-700">오늘 현황</h2>

&#x20;         <div className="flex gap-3">

&#x20;           {isLoading ? (

&#x20;             <>

&#x20;               <StatCardSkeleton />

&#x20;               <StatCardSkeleton />

&#x20;               <StatCardSkeleton />

&#x20;             </>

&#x20;           ) : (

&#x20;             <>

&#x20;               <StatCard

&#x20;                 label="예약"

&#x20;                 value={data?.today\_total ?? 0}

&#x20;                 color="blue"

&#x20;                 onClick={() => goToReservations()}

&#x20;               />

&#x20;               <StatCard

&#x20;                 label="보관중"

&#x20;                 value={data?.current\_stored ?? 0}

&#x20;                 color="green"

&#x20;                 onClick={() => goToReservations('STORED')}

&#x20;               />

&#x20;               <StatCard

&#x20;                 label="완료"

&#x20;                 value={data?.today\_returned ?? 0}

&#x20;                 color="gray"

&#x20;                 onClick={() => goToReservations('RETURNED')}

&#x20;               />

&#x20;             </>

&#x20;           )}

&#x20;         </div>

&#x20;       </section>



&#x20;       {/\* ── 빠른 액션 ────────────────────────────────────── \*/}

&#x20;       <section className="flex gap-3">

&#x20;         <button

&#x20;           onClick={() => router.push('/admin/checkin')}

&#x20;           className="flex flex-1 items-center justify-center gap-2 rounded-2xl bg-blue-600 py-4 text-sm font-semibold text-white shadow-sm active:bg-blue-700"

&#x20;         >

&#x20;           <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;             <path strokeLinecap="round" strokeLinejoin="round" d="M9 3.75H6.912a2.25 2.25 0 00-2.15 1.588L2.35 13.177a2.25 2.25 0 00-.1.661V18a2.25 2.25 0 002.25 2.25h15A2.25 2.25 0 0021.75 18v-4.162c0-.224-.034-.447-.1-.661L19.24 5.338a2.25 2.25 0 00-2.15-1.588H15M2.25 13.5h3.86a2.25 2.25 0 012.012 1.244l.256.512a2.25 2.25 0 002.013 1.244h3.218a2.25 2.25 0 002.013-1.244l.256-.512a2.25 2.25 0 012.013-1.244h3.859M12 3v8.25m0 0l-3-3m3 3l3-3" />

&#x20;           </svg>

&#x20;           짐 접수하기

&#x20;         </button>

&#x20;         <button

&#x20;           onClick={() => router.push('/admin/checkout')}

&#x20;           className="flex flex-1 items-center justify-center gap-2 rounded-2xl border border-blue-200 bg-blue-50 py-4 text-sm font-semibold text-blue-700 shadow-sm active:bg-blue-100"

&#x20;         >

&#x20;           <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" strokeWidth={2} stroke="currentColor">

&#x20;             <path strokeLinecap="round" strokeLinejoin="round" d="M9 3.75H6.912a2.25 2.25 0 00-2.15 1.588L2.35 13.177a2.25 2.25 0 00-.1.661V18a2.25 2.25 0 002.25 2.25h15A2.25 2.25 0 0021.75 18v-4.162c0-.224-.034-.447-.1-.661L19.24 5.338a2.25 2.25 0 00-2.15-1.588H15M2.25 13.5h3.86a2.25 2.25 0 012.012 1.244l.256.512a2.25 2.25 0 002.013 1.244h3.218a2.25 2.25 0 002.013-1.244l.256-.512a2.25 2.25 0 012.013-1.244h3.859M12 3v8.25m0 0l3 3m-3-3l-3 3" />

&#x20;           </svg>

&#x20;           짐 반환하기

&#x20;         </button>

&#x20;       </section>



&#x20;       {/\* ── 최근 접수 대기 목록 ──────────────────────────── \*/}

&#x20;       <section>

&#x20;         <div className="mb-3 flex items-center justify-between">

&#x20;           <h2 className="text-sm font-semibold text-gray-700">최근 접수 대기</h2>

&#x20;           <button

&#x20;             onClick={() => goToReservations('RESERVED')}

&#x20;             className="text-xs text-blue-600 hover:underline"

&#x20;           >

&#x20;             전체 보기 →

&#x20;           </button>

&#x20;         </div>



&#x20;         <div className="overflow-hidden rounded-2xl bg-white shadow-sm">

&#x20;           {isLoading ? (

&#x20;             <div className="divide-y divide-gray-100">

&#x20;               <PendingItemSkeleton />

&#x20;               <PendingItemSkeleton />

&#x20;               <PendingItemSkeleton />

&#x20;             </div>

&#x20;           ) : !data?.recent\_pending?.length ? (

&#x20;             // 빈 상태

&#x20;             <div className="flex flex-col items-center justify-center py-10 text-center">

&#x20;               <svg className="h-10 w-10 text-gray-300" fill="none" viewBox="0 0 24 24" stroke="currentColor">

&#x20;                 <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={1.5} d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2" />

&#x20;               </svg>

&#x20;               <p className="mt-3 text-sm text-gray-400">현재 접수 대기 중인 예약이 없습니다.</p>

&#x20;             </div>

&#x20;           ) : (

&#x20;             <ul className="divide-y divide-gray-100">

&#x20;               {data.recent\_pending.map((item) => (

&#x20;                 <li key={item.reservation\_no}>

&#x20;                   <button

&#x20;                     onClick={() => router.push(`/admin/reservations/${item.reservation\_no}`)}

&#x20;                     className="flex w-full items-center justify-between px-4 py-3.5 text-left active:bg-gray-50"

&#x20;                   >

&#x20;                     <div className="min-w-0 flex-1">

&#x20;                       {/\* 예약번호 + 터미널 배지 \*/}

&#x20;                       <div className="flex items-center gap-2">

&#x20;                         <span className="text-sm font-semibold text-gray-900">

&#x20;                           {item.reservation\_no}

&#x20;                         </span>

&#x20;                         <span className="rounded-full bg-gray-100 px-2 py-0.5 text-\[10px] font-medium text-gray-600">

&#x20;                           {item.terminal}

&#x20;                         </span>

&#x20;                       </div>

&#x20;                       {/\* 예약자 + 짐 수 + 방향 \*/}

&#x20;                       <p className="mt-0.5 text-xs text-gray-500">

&#x20;                         {item.customer\_name} · 짐 {item.bag\_count}개 ·{' '}

&#x20;                         {DIRECTION\_LABEL\[item.direction] ?? item.direction}

&#x20;                       </p>

&#x20;                     </div>

&#x20;                     {/\* 경과 시간 \*/}

&#x20;                     <span className="ml-3 shrink-0 text-xs text-gray-400">

&#x20;                       {formatElapsed(item.elapsed\_minutes)}

&#x20;                     </span>

&#x20;                   </button>

&#x20;                 </li>

&#x20;               ))}

&#x20;             </ul>

&#x20;           )}

&#x20;         </div>

&#x20;       </section>



&#x20;     </div>

&#x20;   </AdminLayout>

&#x20; )

}

```



\---



\## 설계 노트



\### 폴링 구현 (`useEffect` + `setInterval`)



```

fetchDashboard 의존성: \[terminal]

&#x20; → terminal 변경 시 즉시 재패치 (isLoading = true 리셋)



setInterval(fetchDashboard, 30\_000)

&#x20; → 30초마다 isRefreshing = true 로 표시 (스피너 미니)

&#x20; → cleanup: clearInterval 반드시 실행 (컴포넌트 언마운트 시)

```



페이지를 떠났다가 돌아오면 useEffect 재실행으로 즉시 신선한 데이터를 패치한다.



\### Tailwind `pb-safe` 클래스



iOS Safari 홈 인디케이터 영역(safe area inset bottom)을 처리하기 위해 `pb-safe`를 사용한다.

`tailwind.config.ts`에 아래 설정 추가 필요:



```typescript

// tailwind.config.ts

import type { Config } from 'tailwindcss'



const config: Config = {

&#x20; content: \['./src/\*\*/\*.{js,ts,jsx,tsx,mdx}'],

&#x20; theme: {

&#x20;   extend: {

&#x20;     padding: {

&#x20;       safe: 'env(safe-area-inset-bottom)',

&#x20;     },

&#x20;   },

&#x20; },

&#x20; plugins: \[],

}



export default config

```



\### `AdminLayout` — `'use client'` 사용 이유



NextAuth의 `useSession()`이 클라이언트 훅이므로 `'use client'` 필수.

세션 상태(로딩/인증/만료)에 따라 UI를 동적으로 분기하므로 서버 컴포넌트로 분리할 수 없다.



대시보드 페이지 자체도 `fetch` + `setInterval` 폴링 때문에 `'use client'` 적용.



\### `BottomNav` — `usePathname` 활성 탭 판단



`/admin/checkin`에서 `/admin/checkin/confirm`으로 이동해도 `startsWith` 덕분에 접수 탭이 활성 상태를 유지한다.

단, 대시보드는 `/admin/dashboard`로 정확 매칭을 사용해 `/admin/checkin` 접근 시 홈 탭이 활성화되지 않도록 한다.



\---



\## Session 09 체크리스트



| 파일 | 구현 항목 | 상태 |

|------|----------|------|

| `\[...nextauth]/route.ts` | GET/POST 핸들러 export | ✅ |

| `login/page.tsx` | 폼 / 눈 아이콘 / 로딩 / 에러 메시지 / callbackUrl 복귀 | ✅ |

| `BottomNav.tsx` | 5개 탭 / SVG 아이콘 / startsWith 활성 판단 / safe-area | ✅ |

| `AdminLayout.tsx` | GNB / 로그아웃 모달 / 세션 만료 모달 / showBack | ✅ |

| `dashboard/page.tsx` | 터미널 필터 / 현황 카드 3개 / 빠른 액션 / 대기 목록 / 30초 폴링 / 스켈레톤 | ✅ |



\---



\*다음 세션: Session 10 — FO 예약 플로우 프론트엔드 구현 (홈 → STEP 1\~5 → 예약 완료)\*

