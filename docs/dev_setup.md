\# BagDrop 개발 환경 구성 및 초기 셋업



> 작성: 개발 에이전트 가온

> 세션: Session 06

> 최종 업데이트: 2026-07-01

> 런타임: Node.js 20 LTS / Next.js 14 App Router

> 상태: ✅ Session 06 확정



\---



\## 1. Next.js 프로젝트 생성



```bash

\# Node.js 20 LTS 확인

node -v   # v20.x.x 이상



\# 프로젝트 생성 (대화형 옵션은 아래 선택값 기준)

npx create-next-app@14 bagdrop \\

&#x20; --typescript \\

&#x20; --tailwind \\

&#x20; --eslint \\

&#x20; --app \\

&#x20; --src-dir \\

&#x20; --import-alias "@/\*"



cd bagdrop

```



\### create-next-app 선택 옵션



| 질문 | 선택 |

|------|------|

| TypeScript? | Yes |

| ESLint? | Yes |

| Tailwind CSS? | Yes |

| `src/` directory? | Yes |

| App Router? | Yes |

| Import alias? | Yes (`@/\*`) |



\### tsconfig.json 추가 설정



```json

{

&#x20; "compilerOptions": {

&#x20;   "target": "ES2017",

&#x20;   "lib": \["dom", "dom.iterable", "esnext"],

&#x20;   "allowJs": true,

&#x20;   "skipLibCheck": true,

&#x20;   "strict": true,

&#x20;   "noEmit": true,

&#x20;   "esModuleInterop": true,

&#x20;   "module": "esnext",

&#x20;   "moduleResolution": "bundler",

&#x20;   "resolveJsonModule": true,

&#x20;   "isolatedModules": true,

&#x20;   "jsx": "preserve",

&#x20;   "incremental": true,

&#x20;   "plugins": \[{ "name": "next" }],

&#x20;   "paths": {

&#x20;     "@/\*": \["./src/\*"]

&#x20;   }

&#x20; },

&#x20; "include": \["next-env.d.ts", "\*\*/\*.ts", "\*\*/\*.tsx", ".next/types/\*\*/\*.ts"],

&#x20; "exclude": \["node\_modules"]

}

```



\---



\## 2. 패키지 설치



```bash

\# 런타임 패키지

npm install \\

&#x20; @supabase/supabase-js \\

&#x20; next-auth \\

&#x20; bcryptjs \\

&#x20; qrcode.react \\

&#x20; html5-qrcode \\

&#x20; date-fns \\

&#x20; date-fns-tz \\

&#x20; zod



\# 개발 전용 패키지

npm install -D \\

&#x20; @types/bcryptjs \\

&#x20; @types/node \\

&#x20; @types/react \\

&#x20; @types/react-dom

```



\### 패키지별 역할



| 패키지 | 버전 | 역할 |

|--------|------|------|

| `@supabase/supabase-js` | ^2.x | Supabase DB 클라이언트 |

| `next-auth` | ^4.x | BO 운영자 인증 (Credentials Provider) |

| `bcryptjs` | ^2.x | 비밀번호 해시 (순수 JS, native 빌드 불필요) |

| `qrcode.react` | ^3.x | FO 예약 완료 화면 QR 코드 생성 |

| `html5-qrcode` | ^2.x | BO 짐 접수/반환 QR 스캔 |

| `date-fns` | ^3.x | 날짜 포맷/계산 유틸 |

| `date-fns-tz` | ^3.x | KST(Asia/Seoul) 시간대 처리 |

| `zod` | ^3.x | API 요청 유효성 검증 스키마 |



\---



\## 3. 전체 폴더 구조



```

bagdrop/

├── src/

│   ├── app/

│   │   ├── layout.tsx                    # 루트 레이아웃

│   │   ├── page.tsx                      # FO 홈 (/)

│   │   │

│   │   ├── reserve/                      # FO 예약 플로우

│   │   │   ├── step1/page.tsx            # 기본 정보 선택

│   │   │   ├── step2/page.tsx            # 짐 정보 입력

│   │   │   ├── step3/page.tsx            # 고객 정보 입력

│   │   │   ├── step4/page.tsx            # 결제

│   │   │   └── complete/page.tsx         # 예약 완료

│   │   │

│   │   ├── my-reservation/               # FO 예약 확인

│   │   │   ├── page.tsx                  # 예약번호 조회

│   │   │   └── \[reservationNo]/page.tsx  # 예약 상세

│   │   │

│   │   ├── guide/page.tsx                # FO 이용 안내

│   │   │

│   │   ├── admin/                        # BO 어드민

│   │   │   ├── login/page.tsx            # 로그인

│   │   │   ├── dashboard/page.tsx        # 대시보드

│   │   │   ├── checkin/page.tsx          # 짐 접수

│   │   │   ├── checkout/page.tsx         # 짐 반환

│   │   │   ├── storage/page.tsx          # 보관 현황

│   │   │   ├── reservations/

│   │   │   │   ├── page.tsx              # 예약 관리 목록

│   │   │   │   └── \[reservationNo]/page.tsx  # 예약 상세

│   │   │   └── sales/page.tsx            # 매출 관리

│   │   │

│   │   └── api/

│   │       ├── reservations/

│   │       │   ├── prepare/route.ts

│   │       │   ├── confirm/route.ts

│   │       │   └── \[reservationNo]/

│   │       │       ├── route.ts

│   │       │       └── cancel/route.ts

│   │       ├── payments/

│   │       │   └── webhook/route.ts

│   │       └── admin/

│   │           ├── auth/

│   │           │   └── \[...nextauth]/route.ts

│   │           ├── dashboard/route.ts

│   │           ├── reservations/

│   │           │   ├── route.ts

│   │           │   └── \[reservationNo]/

│   │           │       ├── route.ts

│   │           │       ├── checkin/route.ts

│   │           │       ├── checkout/route.ts

│   │           │       └── cancel/route.ts

│   │           ├── storage/route.ts

│   │           └── sales/route.ts

│   │

│   ├── components/

│   │   ├── fo/                           # FO 전용 컴포넌트

│   │   │   ├── ReservationQR.tsx         # QR 코드 표시

│   │   │   ├── StatusBadge.tsx           # 예약 상태 배지

│   │   │   └── StepBar.tsx               # 예약 단계 진행 바

│   │   ├── bo/                           # BO 전용 컴포넌트

│   │   │   ├── QRScanner.tsx             # QR 스캔

│   │   │   ├── StatusBadge.tsx           # 상태 배지 (BO용 색상)

│   │   │   └── BottomNav.tsx             # 하단 탭 네비게이션

│   │   └── common/                       # 공통 컴포넌트

│   │       ├── Modal.tsx

│   │       ├── Toast.tsx

│   │       └── LoadingSpinner.tsx

│   │

│   ├── lib/

│   │   ├── supabase.ts                   # Supabase 클라이언트

│   │   ├── auth.ts                       # NextAuth.js 설정

│   │   ├── validate.ts                   # Zod 유효성 검증 스키마

│   │   ├── toss.ts                       # 토스페이 API 유틸

│   │   └── sms.ts                        # 알리고 SMS 유틸

│   │

│   ├── types/

│   │   └── index.ts                      # 공통 TypeScript 타입 정의

│   │

│   ├── constants/

│   │   └── index.ts                      # 서비스 상수 (요금, 상태값 등)

│   │

│   └── middleware.ts                     # 인증 미들웨어

│

├── scripts/

│   └── seed.ts                           # 초기 데이터 시드

│

├── .env.local                            # 환경변수 (Git 제외)

├── .env.example                          # 환경변수 템플릿 (Git 포함)

├── .gitignore

├── next.config.js

├── tailwind.config.ts

├── tsconfig.json

└── package.json

```



\---



\## 4. 환경변수 파일



\### `.env.example` (Git에 포함 — 값 없이 키만)



```bash

\# ─── Supabase ───────────────────────────────────────────

NEXT\_PUBLIC\_SUPABASE\_URL=

NEXT\_PUBLIC\_SUPABASE\_ANON\_KEY=

SUPABASE\_SERVICE\_ROLE\_KEY=



\# ─── NextAuth.js ────────────────────────────────────────

NEXTAUTH\_URL=

NEXTAUTH\_SECRET=



\# ─── 토스페이 ───────────────────────────────────────────

NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY=

TOSS\_SECRET\_KEY=

TOSS\_WEBHOOK\_SECRET=



\# ─── 알리고 SMS ─────────────────────────────────────────

ALIGO\_API\_KEY=

ALIGO\_USER\_ID=

ALIGO\_SENDER\_PHONE=



\# ─── 어드민 초기 계정 (시드 전용) ───────────────────────

ADMIN\_INITIAL\_PASSWORD=



\# ─── 앱 설정 ────────────────────────────────────────────

NEXT\_PUBLIC\_APP\_URL=

NODE\_ENV=development

```



\### `.env.local` (로컬 개발용 — Git 제외)



```bash

\# ─── Supabase ───────────────────────────────────────────

NEXT\_PUBLIC\_SUPABASE\_URL=https://xxxxxxxxxxxx.supabase.co

NEXT\_PUBLIC\_SUPABASE\_ANON\_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

SUPABASE\_SERVICE\_ROLE\_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...



\# ─── NextAuth.js ────────────────────────────────────────

NEXTAUTH\_URL=http://localhost:3000

\# openssl rand -base64 32 로 생성

NEXTAUTH\_SECRET=your-32-char-random-secret-here



\# ─── 토스페이 ───────────────────────────────────────────

\# 테스트 키 (https://developers.tosspayments.com)

NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY=test\_ck\_...

TOSS\_SECRET\_KEY=test\_sk\_...

TOSS\_WEBHOOK\_SECRET=your-webhook-secret



\# ─── 알리고 SMS ─────────────────────────────────────────

ALIGO\_API\_KEY=your-aligo-api-key

ALIGO\_USER\_ID=your-aligo-user-id

ALIGO\_SENDER\_PHONE=01000000000



\# ─── 어드민 초기 계정 (시드 전용) ───────────────────────

ADMIN\_INITIAL\_PASSWORD=bagdrop2026!



\# ─── 앱 설정 ────────────────────────────────────────────

NEXT\_PUBLIC\_APP\_URL=http://localhost:3000

NODE\_ENV=development

```



\---



\## 5. Supabase 클라이언트 (`src/lib/supabase.ts`)



```typescript

import { createClient, SupabaseClient } from '@supabase/supabase-js'



// ──────────────────────────────────────────────────────────────

// 타입 정의

// ──────────────────────────────────────────────────────────────



export type ReservationStatus = 'RESERVED' | 'STORED' | 'RETURNED' | 'CANCELLED'

export type TerminalType      = 'T1' | 'T2'

export type TravelDirection   = 'DEPARTURE' | 'ARRIVAL'

export type CancellerType     = 'CUSTOMER' | 'ADMIN'

export type PaymentStatus     = 'PENDING' | 'PAID' | 'CANCELLED' | 'FAILED'

export type SmsEventType      = 'RESERVATION\_COMPLETE' | 'CHECKIN\_COMPLETE' | 'CHECKOUT\_COMPLETE' | 'CANCELLATION\_COMPLETE'

export type SmsResult         = 'SUCCESS' | 'FAILED'



export interface Reservation {

&#x20; id:              string

&#x20; reservation\_no:  string

&#x20; terminal:        TerminalType

&#x20; direction:       TravelDirection

&#x20; use\_date:        string          // YYYY-MM-DD

&#x20; bag\_count:       number

&#x20; customer\_name:   string

&#x20; customer\_phone:  string

&#x20; amount:          number

&#x20; status:          ReservationStatus

&#x20; cancelled\_by:    CancellerType | null

&#x20; cancel\_reason:   string | null

&#x20; checked\_in\_at:   string | null

&#x20; checked\_out\_at:  string | null

&#x20; cancelled\_at:    string | null

&#x20; created\_at:      string

&#x20; updated\_at:      string

}



export interface Payment {

&#x20; id:               string

&#x20; reservation\_id:   string

&#x20; toss\_payment\_key: string

&#x20; toss\_order\_id:    string

&#x20; amount:           number

&#x20; method:           string

&#x20; status:           PaymentStatus

&#x20; toss\_response:    Record<string, unknown> | null

&#x20; cancelled\_amount: number | null

&#x20; cancelled\_at:     string | null

&#x20; paid\_at:          string | null

&#x20; created\_at:       string

&#x20; updated\_at:       string

}



export interface ReservationLog {

&#x20; id:             string

&#x20; reservation\_id: string

&#x20; from\_status:    ReservationStatus | null

&#x20; to\_status:      ReservationStatus

&#x20; actor\_type:     string

&#x20; actor\_id:       string | null

&#x20; actor\_label:    string | null

&#x20; note:           string | null

&#x20; created\_at:     string

}



export interface SmsLog {

&#x20; id:              string

&#x20; reservation\_id:  string

&#x20; event\_type:      SmsEventType

&#x20; recipient\_phone: string

&#x20; message\_content: string

&#x20; result:          SmsResult

&#x20; aligo\_msg\_id:    string | null

&#x20; error\_message:   string | null

&#x20; created\_at:      string

}



export interface AdminUser {

&#x20; id:               string

&#x20; username:         string

&#x20; password\_hash:    string

&#x20; display\_name:     string

&#x20; is\_active:        boolean

&#x20; last\_login\_at:    string | null

&#x20; login\_fail\_count: number

&#x20; locked\_until:     string | null

&#x20; created\_at:       string

&#x20; updated\_at:       string

}



// ──────────────────────────────────────────────────────────────

// Database 타입 (Supabase 자동생성 타입 대체용 수동 정의)

// ──────────────────────────────────────────────────────────────



export interface Database {

&#x20; public: {

&#x20;   Tables: {

&#x20;     reservations:     { Row: Reservation }

&#x20;     payments:         { Row: Payment }

&#x20;     reservation\_logs: { Row: ReservationLog }

&#x20;     sms\_logs:         { Row: SmsLog }

&#x20;     admin\_users:      { Row: AdminUser }

&#x20;   }

&#x20;   Functions: {

&#x20;     get\_next\_reservation\_no: {

&#x20;       Args: { target\_date: string }

&#x20;       Returns: string

&#x20;     }

&#x20;   }

&#x20; }

}



// ──────────────────────────────────────────────────────────────

// 클라이언트 싱글턴 (서버사이드 전용 — service\_role)

// ──────────────────────────────────────────────────────────────



let \_supabase: SupabaseClient<Database> | null = null



/\*\*

&#x20;\* 서버사이드 전용 Supabase 클라이언트

&#x20;\* service\_role 키 사용 → API Route에서만 호출할 것

&#x20;\* 절대로 클라이언트 컴포넌트에서 import하지 말 것

&#x20;\*/

export function getSupabaseClient(): SupabaseClient<Database> {

&#x20; if (\_supabase) return \_supabase



&#x20; const url = process.env.NEXT\_PUBLIC\_SUPABASE\_URL

&#x20; const key = process.env.SUPABASE\_SERVICE\_ROLE\_KEY



&#x20; if (!url || !key) {

&#x20;   throw new Error(

&#x20;     'Supabase 환경변수가 설정되지 않았습니다. ' +

&#x20;     'NEXT\_PUBLIC\_SUPABASE\_URL, SUPABASE\_SERVICE\_ROLE\_KEY를 확인해 주세요.'

&#x20;   )

&#x20; }



&#x20; \_supabase = createClient<Database>(url, key, {

&#x20;   auth: {

&#x20;     autoRefreshToken: false,

&#x20;     persistSession: false,

&#x20;   },

&#x20; })



&#x20; return \_supabase

}



// 편의용 alias

export const supabase = () => getSupabaseClient()

```



\---



\## 6. NextAuth.js 설정 (`src/lib/auth.ts`)



```typescript

import { type NextAuthOptions } from 'next-auth'

import CredentialsProvider from 'next-auth/providers/credentials'

import bcrypt from 'bcryptjs'

import { getSupabaseClient } from '@/lib/supabase'

import { formatInTimeZone } from 'date-fns-tz'



// NextAuth 세션/토큰 타입 확장

declare module 'next-auth' {

&#x20; interface Session {

&#x20;   user: {

&#x20;     id:          string

&#x20;     username:    string

&#x20;     displayName: string

&#x20;   }

&#x20; }

&#x20; interface User {

&#x20;   id:          string

&#x20;   username:    string

&#x20;   displayName: string

&#x20; }

}



declare module 'next-auth/jwt' {

&#x20; interface JWT {

&#x20;   id:          string

&#x20;   username:    string

&#x20;   displayName: string

&#x20; }

}



// 로그인 실패 한도

const MAX\_FAIL\_COUNT  = 5

const LOCK\_MINUTES    = 5



export const authOptions: NextAuthOptions = {

&#x20; providers: \[

&#x20;   CredentialsProvider({

&#x20;     name: 'Credentials',

&#x20;     credentials: {

&#x20;       username: { label: '아이디', type: 'text' },

&#x20;       password: { label: '비밀번호', type: 'password' },

&#x20;     },



&#x20;     async authorize(credentials) {

&#x20;       if (!credentials?.username || !credentials?.password) return null



&#x20;       const db = getSupabaseClient()



&#x20;       // 1. 계정 조회

&#x20;       const { data: admin, error } = await db

&#x20;         .from('admin\_users')

&#x20;         .select('\*')

&#x20;         .eq('username', credentials.username)

&#x20;         .eq('is\_active', true)

&#x20;         .single()



&#x20;       if (error || !admin) return null



&#x20;       // 2. 잠금 여부 확인

&#x20;       if (admin.locked\_until) {

&#x20;         const lockedUntil = new Date(admin.locked\_until)

&#x20;         if (lockedUntil > new Date()) return null  // 잠금 중

&#x20;       }



&#x20;       // 3. 비밀번호 검증

&#x20;       const isValid = await bcrypt.compare(credentials.password, admin.password\_hash)



&#x20;       if (!isValid) {

&#x20;         // 실패 카운트 증가

&#x20;         const newFailCount = admin.login\_fail\_count + 1

&#x20;         const lockedUntil  = newFailCount >= MAX\_FAIL\_COUNT

&#x20;           ? new Date(Date.now() + LOCK\_MINUTES \* 60 \* 1000).toISOString()

&#x20;           : null



&#x20;         await db

&#x20;           .from('admin\_users')

&#x20;           .update({

&#x20;             login\_fail\_count: newFailCount,

&#x20;             locked\_until:     lockedUntil,

&#x20;           })

&#x20;           .eq('id', admin.id)



&#x20;         return null

&#x20;       }



&#x20;       // 4. 로그인 성공 — 카운트 초기화 + last\_login\_at 업데이트

&#x20;       await db

&#x20;         .from('admin\_users')

&#x20;         .update({

&#x20;           login\_fail\_count: 0,

&#x20;           locked\_until:     null,

&#x20;           last\_login\_at:    new Date().toISOString(),

&#x20;         })

&#x20;         .eq('id', admin.id)



&#x20;       return {

&#x20;         id:          admin.id,

&#x20;         username:    admin.username,

&#x20;         displayName: admin.display\_name,

&#x20;         name:        admin.display\_name,  // NextAuth 기본 필드

&#x20;         email:       null,

&#x20;       }

&#x20;     },

&#x20;   }),

&#x20; ],



&#x20; session: {

&#x20;   strategy: 'jwt',

&#x20;   maxAge:   12 \* 60 \* 60,  // 12시간 (초 단위)

&#x20; },



&#x20; callbacks: {

&#x20;   async jwt({ token, user }) {

&#x20;     if (user) {

&#x20;       token.id          = user.id

&#x20;       token.username    = user.username

&#x20;       token.displayName = user.displayName

&#x20;     }

&#x20;     return token

&#x20;   },

&#x20;   async session({ session, token }) {

&#x20;     session.user = {

&#x20;       id:          token.id,

&#x20;       username:    token.username,

&#x20;       displayName: token.displayName,

&#x20;     }

&#x20;     return session

&#x20;   },

&#x20; },



&#x20; pages: {

&#x20;   signIn: '/admin/login',

&#x20;   error:  '/admin/login',

&#x20; },



&#x20; secret: process.env.NEXTAUTH\_SECRET,

}



/\*\*

&#x20;\* BO API Route에서 세션 인증 확인용 헬퍼

&#x20;\* 세션 없으면 null 반환 → 호출 측에서 401 처리

&#x20;\*/

export async function requireAdminSession(

&#x20; req: Request

): Promise<{ id: string; username: string; displayName: string } | null> {

&#x20; const { getServerSession } = await import('next-auth')

&#x20; const session = await getServerSession(authOptions)

&#x20; if (!session?.user?.id) return null

&#x20; return session.user

}

```



\---



\## 7. 공통 유효성 검증 유틸 (`src/lib/validate.ts`)



```typescript

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



/\*\* 전화번호: 01X 시작 11자리 숫자 \*/

export const phoneSchema = z

&#x20; .string()

&#x20; .regex(/^01\[0-9]\\d{7,8}$/, '올바른 휴대폰 번호 형식이 아닙니다.')

&#x20; .length(11, '휴대폰 번호는 11자리여야 합니다.')



/\*\* 이용일: 오늘 \~ 오늘+90일 \*/

export const useDateSchema = z

&#x20; .string()

&#x20; .regex(/^\\d{4}-\\d{2}-\\d{2}$/, '날짜 형식이 올바르지 않습니다. (YYYY-MM-DD)')

&#x20; .refine((date) => {

&#x20;   const todayKst    = parseISO(formatInTimeZone(new Date(), TZ, 'yyyy-MM-dd'))

&#x20;   const maxDate     = addDays(todayKst, 90)

&#x20;   const parsedDate  = parseISO(date)

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

&#x20; customer\_phone: phoneSchema,

})



/\*\* POST /api/reservations/confirm \*/

export const confirmReservationSchema = z.object({

&#x20; payment\_key:    z.string().min(1),

&#x20; toss\_order\_id:  z.string().min(1),

&#x20; amount:         z.literal(5000),

&#x20; terminal:       z.enum(\['T1', 'T2']),

&#x20; direction:      z.enum(\['DEPARTURE', 'ARRIVAL']),

&#x20; use\_date:       useDateSchema,

&#x20; bag\_count:      z.number().int().min(1).max(2),

&#x20; customer\_name:  z.string().min(1).max(20).trim(),

&#x20; customer\_phone: phoneSchema,

})



/\*\* GET /api/admin/reservations (Query Params) \*/

export const adminReservationListSchema = z.object({

&#x20; date:     z.string().regex(/^\\d{4}-\\d{2}-\\d{2}$/).optional(),

&#x20; terminal: z.enum(\['T1', 'T2']).optional(),

&#x20; status:   z.string().optional(),   // "RESERVED,STORED" → 파싱 별도 처리

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

&#x20; if (phone.length !== 11) return phone

&#x20; return `${phone.slice(0, 3)}-\*\*\*\*-${phone.slice(7)}`

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

&#x20; const todayKst   = getTodayKst()

&#x20; return useDate > todayKst   // 이용일 00시 이전 = 이용일 > 오늘

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



\---



\## 8. 토스페이 유틸 (`src/lib/toss.ts`)



```typescript

import crypto from 'crypto'



const TOSS\_API\_BASE = 'https://api.tosspayments.com'



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

&#x20; \[key: string]: unknown  // 나머지 필드 (toss\_response JSONB 저장용)

}



export interface TossErrorResponse {

&#x20; code:    string

&#x20; message: string

}



// ──────────────────────────────────────────────────────────────

// 핵심 함수

// ──────────────────────────────────────────────────────────────



/\*\*

&#x20;\* 토스페이 Basic 인증 헤더 생성

&#x20;\* 시크릿 키 뒤에 콜론(:)을 붙여 Base64 인코딩

&#x20;\*/

function getTossAuthHeader(): string {

&#x20; const secretKey = process.env.TOSS\_SECRET\_KEY

&#x20; if (!secretKey) throw new Error('TOSS\_SECRET\_KEY 환경변수가 설정되지 않았습니다.')

&#x20; const encoded = Buffer.from(`${secretKey}:`).toString('base64')

&#x20; return `Basic ${encoded}`

}



/\*\*

&#x20;\* 결제 승인 API 호출

&#x20;\* POST https://api.tosspayments.com/v1/payments/confirm

&#x20;\*/

export async function confirmTossPayment(

&#x20; params: TossConfirmRequest

): Promise<{ ok: true; data: TossPaymentResponse } | { ok: false; error: TossErrorResponse }> {

&#x20; try {

&#x20;   const response = await fetch(`${TOSS\_API\_BASE}/v1/payments/confirm`, {

&#x20;     method: 'POST',

&#x20;     headers: {

&#x20;       Authorization:  getTossAuthHeader(),

&#x20;       'Content-Type': 'application/json',

&#x20;     },

&#x20;     body: JSON.stringify({

&#x20;       paymentKey: params.paymentKey,

&#x20;       orderId:    params.orderId,

&#x20;       amount:     params.amount,

&#x20;     }),

&#x20;   })



&#x20;   const body = await response.json()



&#x20;   if (!response.ok) {

&#x20;     return { ok: false, error: body as TossErrorResponse }

&#x20;   }



&#x20;   return { ok: true, data: body as TossPaymentResponse }

&#x20; } catch (err) {

&#x20;   return {

&#x20;     ok: false,

&#x20;     error: { code: 'NETWORK\_ERROR', message: '토스페이 API 연결에 실패했습니다.' },

&#x20;   }

&#x20; }

}



/\*\*

&#x20;\* 토스페이 웹훅 서명 검증

&#x20;\* HMAC-SHA256(payload, TOSS\_WEBHOOK\_SECRET)

&#x20;\*/

export function verifyTossWebhookSignature(

&#x20; payload: string,

&#x20; receivedSignature: string

): boolean {

&#x20; const secret = process.env.TOSS\_WEBHOOK\_SECRET

&#x20; if (!secret) throw new Error('TOSS\_WEBHOOK\_SECRET 환경변수가 설정되지 않았습니다.')



&#x20; const expectedSignature = crypto

&#x20;   .createHmac('sha256', secret)

&#x20;   .update(payload, 'utf-8')

&#x20;   .digest('base64')



&#x20; // 타이밍 공격 방지를 위한 상수 시간 비교

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

&#x20;\* 토스페이 orderId 제약: 영문 대소문자·숫자·-·\_ / 최대 64자

&#x20;\*/

export function generateTossOrderId(dateStr: string): string {

&#x20; const randomHex = crypto.randomBytes(4).toString('hex')

&#x20; return `bagdrop\_${dateStr}\_${randomHex}`

}

```



\---



\## 9. 알리고 SMS 유틸 (`src/lib/sms.ts`)



```typescript

import { getSupabaseClient, type SmsEventType } from '@/lib/supabase'

import { formatInTimeZone } from 'date-fns-tz'

import { parseISO } from 'date-fns'



const TZ              = 'Asia/Seoul'

const ALIGO\_SEND\_URL  = 'https://apis.aligo.in/send/'

const IS\_PROD         = process.env.NODE\_ENV === 'production'



// ──────────────────────────────────────────────────────────────

// 메시지 템플릿

// ──────────────────────────────────────────────────────────────



const TERMINAL\_LABEL: Record<string, string> = {

&#x20; T1: '제1여객터미널(T1)',

&#x20; T2: '제2여객터미널(T2)',

}



const DIRECTION\_LABEL: Record<string, string> = {

&#x20; DEPARTURE: '출국',

&#x20; ARRIVAL:   '귀국',

}



function formatDate(dateStr: string): string {

&#x20; // "2026-07-15" → "2026.07.15"

&#x20; return dateStr.replace(/-/g, '.')

}



function formatDateTime(isoStr: string): string {

&#x20; return formatInTimeZone(parseISO(isoStr), TZ, 'HH:mm')

}



interface ReservationSmsData {

&#x20; reservationNo: string

&#x20; terminal:      string

&#x20; direction:     string

&#x20; useDate:       string

&#x20; bagCount:      number

&#x20; checkedInAt?:  string

}



function buildMessage(eventType: SmsEventType, data: ReservationSmsData): string {

&#x20; const terminalLabel  = TERMINAL\_LABEL\[data.terminal]  ?? data.terminal

&#x20; const directionLabel = DIRECTION\_LABEL\[data.direction] ?? data.direction

&#x20; const appUrl         = process.env.NEXT\_PUBLIC\_APP\_URL ?? 'https://bagdrop.kr'



&#x20; switch (eventType) {

&#x20;   case 'RESERVATION\_COMPLETE':

&#x20;     return \[

&#x20;       '\[BagDrop] 예약이 완료됐습니다.',

&#x20;       '',

&#x20;       `예약번호: ${data.reservationNo}`,

&#x20;       `터미널: ${terminalLabel}`,

&#x20;       `이용일: ${formatDate(data.useDate)} / ${directionLabel}`,

&#x20;       `짐 개수: ${data.bagCount}개`,

&#x20;       '',

&#x20;       '부스에서 QR 또는 예약번호를 제시해 주세요.',

&#x20;       `예약 확인: ${appUrl}/my-reservation`,

&#x20;     ].join('\\n')



&#x20;   case 'CHECKIN\_COMPLETE':

&#x20;     return \[

&#x20;       '\[BagDrop] 짐이 접수됐습니다.',

&#x20;       '',

&#x20;       `예약번호: ${data.reservationNo}`,

&#x20;       data.checkedInAt ? `접수 시각: ${formatDateTime(data.checkedInAt)}` : '',

&#x20;       '',

&#x20;       '차량을 가져오신 후 부스에서 짐을 찾아가세요.',

&#x20;     ].filter(Boolean).join('\\n')



&#x20;   case 'CHECKOUT\_COMPLETE':

&#x20;     return \[

&#x20;       '\[BagDrop] 짐이 반환됐습니다.',

&#x20;       '',

&#x20;       `예약번호: ${data.reservationNo}`,

&#x20;       '',

&#x20;       '이용해 주셔서 감사합니다!',

&#x20;     ].join('\\n')



&#x20;   case 'CANCELLATION\_COMPLETE':

&#x20;     return \[

&#x20;       '\[BagDrop] 예약이 취소됐습니다.',

&#x20;       '',

&#x20;       `예약번호: ${data.reservationNo}`,

&#x20;       '',

&#x20;       '결제 금액(5,000원)은 영업일 기준 3\~5일 내 환불됩니다.',

&#x20;     ].join('\\n')



&#x20;   default:

&#x20;     return `\[BagDrop] 예약번호: ${data.reservationNo}`

&#x20; }

}



// ──────────────────────────────────────────────────────────────

// 발송 함수

// ──────────────────────────────────────────────────────────────



interface SendSmsParams {

&#x20; reservationId: string

&#x20; eventType:     SmsEventType

&#x20; phone:         string

&#x20; data:          ReservationSmsData

}



/\*\*

&#x20;\* SMS 발송 + sms\_logs 기록

&#x20;\* 발송 실패는 예외를 던지지 않고 FAILED 로그로만 기록

&#x20;\* (SMS 실패로 예약 트랜잭션을 롤백하지 않음)

&#x20;\*/

export async function sendSms(params: SendSmsParams): Promise<void> {

&#x20; const { reservationId, eventType, phone, data } = params

&#x20; const db      = getSupabaseClient()

&#x20; const message = buildMessage(eventType, data)



&#x20; let result:       'SUCCESS' | 'FAILED' = 'FAILED'

&#x20; let aligoMsgId:   string | null        = null

&#x20; let errorMessage: string | null        = null



&#x20; try {

&#x20;   // 환경변수 검증

&#x20;   const apiKey      = process.env.ALIGO\_API\_KEY

&#x20;   const userId      = process.env.ALIGO\_USER\_ID

&#x20;   const senderPhone = process.env.ALIGO\_SENDER\_PHONE



&#x20;   if (!apiKey || !userId || !senderPhone) {

&#x20;     throw new Error('알리고 SMS 환경변수가 설정되지 않았습니다.')

&#x20;   }



&#x20;   // 알리고 API 호출

&#x20;   const formData = new FormData()

&#x20;   formData.append('key',          apiKey)

&#x20;   formData.append('user\_id',      userId)

&#x20;   formData.append('sender',       senderPhone)

&#x20;   formData.append('receiver',     phone)

&#x20;   formData.append('msg',          message)

&#x20;   formData.append('testmode\_yn',  IS\_PROD ? 'N' : 'Y')  // 개발 환경: 실제 발송 안 함



&#x20;   const response   = await fetch(ALIGO\_SEND\_URL, { method: 'POST', body: formData })

&#x20;   const responseData = await response.json() as { result\_code: string; message: string; msg\_id?: number }



&#x20;   if (responseData.result\_code === '1') {

&#x20;     result     = 'SUCCESS'

&#x20;     aligoMsgId = responseData.msg\_id?.toString() ?? null

&#x20;   } else {

&#x20;     errorMessage = responseData.message

&#x20;   }

&#x20; } catch (err) {

&#x20;   errorMessage = err instanceof Error ? err.message : 'SMS 발송 중 알 수 없는 오류 발생'

&#x20;   console.error('\[SMS] 발송 실패:', { eventType, phone, error: errorMessage })

&#x20; }



&#x20; // sms\_logs INSERT (성공/실패 무관하게 항상 기록)

&#x20; const { error: logError } = await db.from('sms\_logs').insert({

&#x20;   reservation\_id:  reservationId,

&#x20;   event\_type:      eventType,

&#x20;   recipient\_phone: phone,

&#x20;   message\_content: message,

&#x20;   result,

&#x20;   aligo\_msg\_id:    aligoMsgId,

&#x20;   error\_message:   errorMessage,

&#x20; })



&#x20; if (logError) {

&#x20;   console.error('\[SMS] 로그 저장 실패:', logError)

&#x20; }

}



/\*\*

&#x20;\* 비동기 SMS 발송 래퍼

&#x20;\* API 응답 속도에 영향을 주지 않도록 await 없이 호출용

&#x20;\* 내부 오류는 콘솔 로그로만 처리

&#x20;\*/

export function sendSmsAsync(params: SendSmsParams): void {

&#x20; sendSms(params).catch((err) => {

&#x20;   console.error('\[SMS] 비동기 발송 실패:', err)

&#x20; })

}

```



\---



\## 10. 미들웨어 (`src/middleware.ts`)



```typescript

import { NextResponse } from 'next/server'

import type { NextRequest } from 'next/server'

import { getToken } from 'next-auth/jwt'



export async function middleware(request: NextRequest): Promise<NextResponse> {

&#x20; const { pathname } = request.nextUrl



&#x20; // ─── BO 어드민 페이지 접근 제어 ───────────────────────────

&#x20; if (pathname.startsWith('/admin') \&\& pathname !== '/admin/login') {

&#x20;   const token = await getToken({

&#x20;     req:    request,

&#x20;     secret: process.env.NEXTAUTH\_SECRET,

&#x20;   })



&#x20;   if (!token) {

&#x20;     // 로그인 후 원래 경로로 복귀할 수 있도록 callbackUrl 포함

&#x20;     const loginUrl = new URL('/admin/login', request.url)

&#x20;     loginUrl.searchParams.set('callbackUrl', pathname)

&#x20;     return NextResponse.redirect(loginUrl)

&#x20;   }

&#x20; }



&#x20; // ─── 로그인 상태에서 /admin/login 접근 시 대시보드로 리디렉트 ─

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

&#x20; // 미들웨어 적용 대상 경로 (정적 파일, API 제외)

&#x20; matcher: \[

&#x20;   '/admin/:path\*',

&#x20;   // API 경로는 각 route.ts에서 getServerSession()으로 재검증

&#x20; ],

}

```



\---



\## 11. 공통 타입 및 상수 (`src/types/index.ts`, `src/constants/index.ts`)



\### `src/types/index.ts`



```typescript

// API 공통 응답 타입

export interface ApiSuccess<T> {

&#x20; success: true

&#x20; data:    T

}



export interface ApiError {

&#x20; success: false

&#x20; error: {

&#x20;   code:    string

&#x20;   message: string

&#x20; }

}



export type ApiResponse<T> = ApiSuccess<T> | ApiError

```



\### `src/constants/index.ts`



```typescript

export const RESERVATION\_AMOUNT    = 5000      // 예약 1건 요금 (원)

export const MAX\_BAG\_COUNT         = 2         // 예약 1건 최대 짐 개수

export const MAX\_RESERVATION\_DAYS  = 90        // 예약 가능 최대 선납 일수

export const SESSION\_MAX\_AGE       = 12 \* 60 \* 60  // BO 세션 유효 시간 (초)

export const MAX\_LOGIN\_FAIL        = 5         // 최대 로그인 실패 횟수

export const LOGIN\_LOCK\_MINUTES    = 5         // 계정 잠금 시간 (분)

export const TIMEZONE              = 'Asia/Seoul'



export const TERMINAL\_LABEL: Record<string, string> = {

&#x20; T1: '제1여객터미널',

&#x20; T2: '제2여객터미널',

}



export const DIRECTION\_LABEL: Record<string, string> = {

&#x20; DEPARTURE: '출국',

&#x20; ARRIVAL:   '귀국',

}



export const STATUS\_LABEL: Record<string, string> = {

&#x20; RESERVED:  '예약완료',

&#x20; STORED:    '보관중',

&#x20; RETURNED:  '반환완료',

&#x20; CANCELLED: '취소완료',

}



// 보관 경과 시간 경고 기준 (분)

export const STORAGE\_WARN\_ORANGE = 120   // 2시간

export const STORAGE\_WARN\_RED    = 240   // 4시간

```



\---



\## 12. 초기 데이터 시드 (`scripts/seed.ts`)



```typescript

import bcrypt from 'bcryptjs'

import { createClient } from '@supabase/supabase-js'



// 환경변수 직접 로드 (tsx로 실행 시 .env.local 자동 로드)

const supabase = createClient(

&#x20; process.env.NEXT\_PUBLIC\_SUPABASE\_URL!,

&#x20; process.env.SUPABASE\_SERVICE\_ROLE\_KEY!

)



async function seed() {

&#x20; console.log('🌱 시드 시작...')



&#x20; const password     = process.env.ADMIN\_INITIAL\_PASSWORD ?? 'bagdrop2026!'

&#x20; const passwordHash = await bcrypt.hash(password, 12)



&#x20; const { error } = await supabase

&#x20;   .from('admin\_users')

&#x20;   .upsert(

&#x20;     {

&#x20;       username:     'admin',

&#x20;       password\_hash: passwordHash,

&#x20;       display\_name: '관리자',

&#x20;       is\_active:    true,

&#x20;     },

&#x20;     { onConflict: 'username' }

&#x20;   )



&#x20; if (error) {

&#x20;   console.error('❌ 시드 실패:', error.message)

&#x20;   process.exit(1)

&#x20; }



&#x20; console.log('✅ 운영자 계정 시드 완료 (username: admin)')

&#x20; console.log(`⚠️  초기 비밀번호: ${password} — 배포 전 반드시 변경하세요.`)

}



seed()

```



\### `package.json` 스크립트 추가



```json

{

&#x20; "scripts": {

&#x20;   "dev":   "next dev",

&#x20;   "build": "next build",

&#x20;   "start": "next start",

&#x20;   "lint":  "next lint",

&#x20;   "seed":  "tsx -r dotenv/config scripts/seed.ts dotenv\_config\_path=.env.local"

&#x20; }

}

```



```bash

\# tsx, dotenv 설치

npm install -D tsx dotenv



\# 시드 실행

npm run seed

```



\---



\## 13. next.config.js



```javascript

/\*\* @type {import('next').NextConfig} \*/

const nextConfig = {

&#x20; // 외부 패키지를 서버 번들에 포함 (bcryptjs native 빌드 우회)

&#x20; serverExternalPackages: \['bcryptjs'],



&#x20; // 환경변수 런타임 검증

&#x20; env: {

&#x20;   NEXT\_PUBLIC\_APP\_URL: process.env.NEXT\_PUBLIC\_APP\_URL,

&#x20; },

}



module.exports = nextConfig

```



\---



\## 14. .gitignore 추가 항목



```

\# 환경변수

.env.local

.env.\*.local



\# 시크릿

\*.pem

```



\---



\## 15. 개발 시작 순서



```bash

\# 1. 의존성 설치

npm install



\# 2. .env.local 설정 (Supabase, 토스페이, 알리고 키 입력)

cp .env.example .env.local

\# → 각 값 입력



\# 3. Supabase 마이그레이션 실행

\# Supabase Dashboard > SQL Editor 에서 db\_schema.md의 SQL 순서대로 실행

\# (1) Enum → (2) Tables → (3) Index → (4) Trigger → (5) CHECK → (6) Function → (7) RLS



\# 4. 어드민 계정 시드

npm run seed



\# 5. 개발 서버 시작

npm run dev

\# → http://localhost:3000       FO 고객용

\# → http://localhost:3000/admin BO 운영자용

```



\---



\## Session 06 체크리스트



| 항목 | 상태 |

|------|------|

| Next.js 14 프로젝트 설정 | ✅ |

| 패키지 목록 및 설치 명령 | ✅ |

| 전체 폴더 구조 | ✅ |

| .env.local 템플릿 | ✅ |

| lib/supabase.ts (타입 포함) | ✅ |

| lib/auth.ts (로그인 실패 잠금 포함) | ✅ |

| lib/validate.ts (Zod 스키마 + 헬퍼) | ✅ |

| lib/toss.ts (승인 + 웹훅 서명 검증) | ✅ |

| lib/sms.ts (템플릿 + 비동기 발송) | ✅ |

| middleware.ts (BO 접근 제어) | ✅ |

| constants/index.ts | ✅ |

| scripts/seed.ts | ✅ |



\---



\*다음 세션: Session 07 — API Route 구현 (FO: prepare → confirm → 예약 조회/취소)\*

