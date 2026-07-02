\# BagDrop 배포 가이드



> 작성: 개발 에이전트 가온

> 세션: Session 15

> 최종 업데이트: 2026-07-01

> 대상 환경: Supabase (ap-northeast-1) + Vercel + Upstash Redis

> 예상 소요 시간: 약 2\~3시간



\---



\## 사전 준비 (계정)



배포 전 아래 계정이 모두 필요합니다.



| 서비스 | 용도 | URL |

|--------|------|-----|

| GitHub | 코드 저장소 | https://github.com |

| Supabase | DB 호스팅 | https://supabase.com |

| Vercel | 앱 호스팅 | https://vercel.com |

| 토스페이먼츠 | 결제 | https://developers.tosspayments.com |

| 알리고 | SMS 발송 | https://aligo.in |

| Upstash | Redis (Rate Limiting) | https://upstash.com |

| 도메인 등록기 | 도메인 구입 | (가비아, 호스팅케이아이 등) |



\---



\## STEP 1. GitHub 레포지토리 설정



\### 1-1. 레포 생성 및 초기 커밋



```bash

\# 프로젝트 루트에서

cd bagdrop



\# Git 초기화 (이미 되어 있으면 생략)

git init

git add .

git commit -m "feat: BagDrop initial commit"



\# GitHub에 레포 생성 후 push

git remote add origin https://github.com/{your-username}/bagdrop.git

git branch -M main

git push -u origin main

```



\### 1-2. `.gitignore` 확인



아래 항목이 반드시 포함되어 있는지 확인합니다.



```gitignore

\# 환경변수 — 절대 커밋하면 안 됨

.env.local

.env.\*.local



\# Next.js 빌드 아티팩트

.next/

out/



\# Node modules

node\_modules/



\# 기타

\*.pem

.DS\_Store

```



\---



\## STEP 2. Supabase 프로젝트 생성



\### 2-1. 프로젝트 생성



1\. \[https://supabase.com](https://supabase.com) → \*\*New project\*\* 클릭

2\. 설정값 입력:



| 항목 | 값 |

|------|-----|

| \*\*Name\*\* | `bagdrop` |

| \*\*Database Password\*\* | 강력한 비밀번호 생성 (저장 필수) |

| \*\*Region\*\* | \*\*Northeast Asia (Tokyo)\*\* ← 한국 서비스 latency 최소화 |

| \*\*Pricing Plan\*\* | Free (초기 MVP) |



3\. 프로젝트 생성 완료까지 약 1\~2분 대기



\### 2-2. 연결 정보 수집



프로젝트 생성 후 \*\*Settings → API\*\* 에서 아래 값을 복사합니다.



```

Project URL:      https://xxxxxxxxxxxx.supabase.co

anon (public):    eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

service\_role:     eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...  ← 절대 공개 금지

```



> ⚠️ `service\_role` 키는 DB 전체 접근 권한을 가집니다. 절대 프론트엔드 코드나 공개 저장소에 노출하지 마세요.



\### 2-3. Supabase Database Pooler 설정



\*\*Settings → Database → Connection Pooling\*\* 에서:

\- \*\*Mode\*\*: `Transaction` 선택

\- \*\*Port\*\*: `6543`



Vercel 서버리스 함수는 요청마다 DB 커넥션을 새로 생성합니다. Pooler를 사용하면 커넥션 수를 관리할 수 있습니다.



\### 2-4. DB 마이그레이션 실행



\*\*SQL Editor\*\*(`Database → SQL Editor → New query`)를 열고 아래 SQL을 \*\*순서대로\*\* 실행합니다.



\---



\#### 마이그레이션 01 — Enum 타입 생성



```sql

\-- Enum 01: 예약 상태

CREATE TYPE reservation\_status AS ENUM (

&#x20; 'RESERVED',

&#x20; 'STORED',

&#x20; 'RETURNED',

&#x20; 'CANCELLED'

);



\-- Enum 02: 터미널 구분

CREATE TYPE terminal\_type AS ENUM (

&#x20; 'T1',

&#x20; 'T2'

);



\-- Enum 03: 이용 방향

CREATE TYPE travel\_direction AS ENUM (

&#x20; 'DEPARTURE',

&#x20; 'ARRIVAL'

);



\-- Enum 04: 취소 주체

CREATE TYPE canceller\_type AS ENUM (

&#x20; 'CUSTOMER',

&#x20; 'ADMIN'

);



\-- Enum 05: 결제 상태

CREATE TYPE payment\_status AS ENUM (

&#x20; 'PENDING',

&#x20; 'PAID',

&#x20; 'CANCELLED',

&#x20; 'FAILED'

);



\-- Enum 06: SMS 이벤트 유형

CREATE TYPE sms\_event\_type AS ENUM (

&#x20; 'RESERVATION\_COMPLETE',

&#x20; 'CHECKIN\_COMPLETE',

&#x20; 'CHECKOUT\_COMPLETE',

&#x20; 'CANCELLATION\_COMPLETE'

);



\-- Enum 07: SMS 결과

CREATE TYPE sms\_result AS ENUM (

&#x20; 'SUCCESS',

&#x20; 'FAILED'

);

```



실행 후 `Success. No rows returned` 확인.



\---



\#### 마이그레이션 02 — 테이블 생성



```sql

\-- 테이블 01: admin\_users

CREATE TABLE admin\_users (

&#x20; id               UUID          PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; username         VARCHAR(50)   NOT NULL UNIQUE,

&#x20; password\_hash    VARCHAR(255)  NOT NULL,

&#x20; display\_name     VARCHAR(50)   NOT NULL,

&#x20; is\_active        BOOLEAN       NOT NULL DEFAULT true,

&#x20; last\_login\_at    TIMESTAMPTZ   NULL,

&#x20; login\_fail\_count SMALLINT      NOT NULL DEFAULT 0,

&#x20; locked\_until     TIMESTAMPTZ   NULL,

&#x20; created\_at       TIMESTAMPTZ   NOT NULL DEFAULT now(),

&#x20; updated\_at       TIMESTAMPTZ   NOT NULL DEFAULT now()

);



\-- 테이블 02: reservations

CREATE TABLE reservations (

&#x20; id              UUID              PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_no  VARCHAR(20)       NOT NULL UNIQUE,

&#x20; terminal        terminal\_type     NOT NULL,

&#x20; direction       travel\_direction  NOT NULL,

&#x20; use\_date        DATE              NOT NULL,

&#x20; bag\_count       SMALLINT          NOT NULL CHECK (bag\_count BETWEEN 1 AND 2),

&#x20; customer\_name   VARCHAR(20)       NOT NULL,

&#x20; customer\_phone  VARCHAR(11)       NOT NULL,

&#x20; amount          INTEGER           NOT NULL DEFAULT 5000 CHECK (amount = 5000),

&#x20; status          reservation\_status NOT NULL DEFAULT 'RESERVED',

&#x20; cancelled\_by    canceller\_type    NULL,

&#x20; cancel\_reason   TEXT              NULL,

&#x20; checked\_in\_at   TIMESTAMPTZ       NULL,

&#x20; checked\_out\_at  TIMESTAMPTZ       NULL,

&#x20; cancelled\_at    TIMESTAMPTZ       NULL,

&#x20; created\_at      TIMESTAMPTZ       NOT NULL DEFAULT now(),

&#x20; updated\_at      TIMESTAMPTZ       NOT NULL DEFAULT now()

);



\-- 테이블 03: payments

CREATE TABLE payments (

&#x20; id                UUID          PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_id    UUID          NULL REFERENCES reservations(id) ON DELETE RESTRICT,

&#x20; toss\_payment\_key  VARCHAR(200)  NOT NULL UNIQUE,

&#x20; toss\_order\_id     VARCHAR(64)   NOT NULL UNIQUE,

&#x20; amount            INTEGER       NOT NULL CHECK (amount = 5000),

&#x20; method            VARCHAR(50)   NOT NULL,

&#x20; status            payment\_status NOT NULL DEFAULT 'PENDING',

&#x20; toss\_response     JSONB         NULL,

&#x20; cancelled\_amount  INTEGER       NULL,

&#x20; cancelled\_at      TIMESTAMPTZ   NULL,

&#x20; paid\_at           TIMESTAMPTZ   NULL,

&#x20; created\_at        TIMESTAMPTZ   NOT NULL DEFAULT now(),

&#x20; updated\_at        TIMESTAMPTZ   NOT NULL DEFAULT now()

);



\-- 테이블 04: reservation\_logs

CREATE TABLE reservation\_logs (

&#x20; id              UUID                PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_id  UUID                NOT NULL REFERENCES reservations(id) ON DELETE CASCADE,

&#x20; from\_status     reservation\_status  NULL,

&#x20; to\_status       reservation\_status  NOT NULL,

&#x20; actor\_type      VARCHAR(20)         NOT NULL,

&#x20; actor\_id        UUID                NULL REFERENCES admin\_users(id),

&#x20; actor\_label     VARCHAR(50)         NULL,

&#x20; note            TEXT                NULL,

&#x20; created\_at      TIMESTAMPTZ         NOT NULL DEFAULT now()

);



\-- 테이블 05: sms\_logs

CREATE TABLE sms\_logs (

&#x20; id              UUID            PRIMARY KEY DEFAULT gen\_random\_uuid(),

&#x20; reservation\_id  UUID            NOT NULL REFERENCES reservations(id) ON DELETE CASCADE,

&#x20; event\_type      sms\_event\_type  NOT NULL,

&#x20; recipient\_phone VARCHAR(11)     NOT NULL,

&#x20; message\_content TEXT            NOT NULL,

&#x20; result          sms\_result      NOT NULL DEFAULT 'SUCCESS',

&#x20; aligo\_msg\_id    VARCHAR(50)     NULL,

&#x20; error\_message   TEXT            NULL,

&#x20; created\_at      TIMESTAMPTZ     NOT NULL DEFAULT now()

);

```



\---



\#### 마이그레이션 03 — 인덱스 생성



```sql

\-- reservations 인덱스

CREATE UNIQUE INDEX idx\_reservations\_reservation\_no

&#x20; ON reservations (reservation\_no);



CREATE INDEX idx\_reservations\_use\_date\_status

&#x20; ON reservations (use\_date, status);



CREATE INDEX idx\_reservations\_status

&#x20; ON reservations (status);



CREATE INDEX idx\_reservations\_terminal\_use\_date

&#x20; ON reservations (terminal, use\_date);



CREATE INDEX idx\_reservations\_created\_at

&#x20; ON reservations (created\_at DESC);



\-- payments 인덱스

CREATE UNIQUE INDEX idx\_payments\_toss\_payment\_key

&#x20; ON payments (toss\_payment\_key);



CREATE UNIQUE INDEX idx\_payments\_toss\_order\_id

&#x20; ON payments (toss\_order\_id);



CREATE INDEX idx\_payments\_reservation\_id

&#x20; ON payments (reservation\_id);



\-- reservation\_logs 인덱스

CREATE INDEX idx\_reservation\_logs\_reservation\_id

&#x20; ON reservation\_logs (reservation\_id, created\_at ASC);



\-- sms\_logs 인덱스

CREATE INDEX idx\_sms\_logs\_reservation\_id

&#x20; ON sms\_logs (reservation\_id);



\-- admin\_users 인덱스

CREATE UNIQUE INDEX idx\_admin\_users\_username

&#x20; ON admin\_users (username);

```



\---



\#### 마이그레이션 04 — updated\_at 트리거



```sql

\-- 공통 트리거 함수

CREATE OR REPLACE FUNCTION update\_updated\_at\_column()

RETURNS TRIGGER AS $$

BEGIN

&#x20; NEW.updated\_at = now();

&#x20; RETURN NEW;

END;

$$ LANGUAGE plpgsql;



\-- 각 테이블 트리거 등록

CREATE TRIGGER trg\_reservations\_updated\_at

&#x20; BEFORE UPDATE ON reservations

&#x20; FOR EACH ROW EXECUTE FUNCTION update\_updated\_at\_column();



CREATE TRIGGER trg\_payments\_updated\_at

&#x20; BEFORE UPDATE ON payments

&#x20; FOR EACH ROW EXECUTE FUNCTION update\_updated\_at\_column();



CREATE TRIGGER trg\_admin\_users\_updated\_at

&#x20; BEFORE UPDATE ON admin\_users

&#x20; FOR EACH ROW EXECUTE FUNCTION update\_updated\_at\_column();

```



\---



\#### 마이그레이션 05 — CHECK 제약 추가



```sql

\-- 예약번호 형식 검증

ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_reservation\_no\_format

&#x20; CHECK (reservation\_no \~ '^BD-\[0-9]{8}-\[0-9]{4}$');



\-- 상태별 시각 정합성

ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_checked\_in\_at

&#x20; CHECK (

&#x20;   (status = 'STORED' AND checked\_in\_at IS NOT NULL) OR

&#x20;   (status != 'STORED')

&#x20; );



ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_checked\_out\_at

&#x20; CHECK (

&#x20;   (status = 'RETURNED' AND checked\_out\_at IS NOT NULL) OR

&#x20;   (status != 'RETURNED')

&#x20; );



ALTER TABLE reservations

&#x20; ADD CONSTRAINT chk\_cancelled\_by

&#x20; CHECK (

&#x20;   (status = 'CANCELLED' AND cancelled\_by IS NOT NULL) OR

&#x20;   (status != 'CANCELLED')

&#x20; );

```



\---



\#### 마이그레이션 06 — 채번 RPC 함수



```sql

CREATE OR REPLACE FUNCTION get\_next\_reservation\_no(target\_date TEXT)

RETURNS TEXT

LANGUAGE plpgsql

AS $$

DECLARE

&#x20; seq\_name TEXT;

&#x20; next\_val BIGINT;

BEGIN

&#x20; IF target\_date !\~ '^\[0-9]{8}$' THEN

&#x20;   RAISE EXCEPTION 'Invalid date format: %', target\_date;

&#x20; END IF;



&#x20; seq\_name := 'reservation\_seq\_' || target\_date;



&#x20; EXECUTE format(

&#x20;   'CREATE SEQUENCE IF NOT EXISTS %I START WITH 1 INCREMENT BY 1 NO CYCLE',

&#x20;   seq\_name

&#x20; );



&#x20; EXECUTE format('SELECT nextval(%L)', seq\_name) INTO next\_val;



&#x20; RETURN 'BD-' || target\_date || '-' || LPAD(next\_val::TEXT, 4, '0');

END;

$$;

```



\---



\#### 마이그레이션 07 — RLS 정책 활성화



```sql

\-- reservations

ALTER TABLE reservations ENABLE ROW LEVEL SECURITY;



CREATE POLICY "고객: 예약번호로 본인 예약 조회"

&#x20; ON reservations FOR SELECT

&#x20; USING (true);



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON reservations FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- payments

ALTER TABLE payments ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON payments FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- reservation\_logs

ALTER TABLE reservation\_logs ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON reservation\_logs FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- sms\_logs

ALTER TABLE sms\_logs ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON sms\_logs FOR ALL

&#x20; USING (auth.role() = 'service\_role');



\-- admin\_users

ALTER TABLE admin\_users ENABLE ROW LEVEL SECURITY;



CREATE POLICY "서비스 롤: 전체 접근"

&#x20; ON admin\_users FOR ALL

&#x20; USING (auth.role() = 'service\_role');

```



\---



\#### 마이그레이션 완료 확인



SQL Editor에서 아래를 실행해 모든 테이블이 정상 생성됐는지 확인합니다.



```sql

SELECT table\_name

FROM information\_schema.tables

WHERE table\_schema = 'public'

ORDER BY table\_name;



\-- 결과 확인: admin\_users, payments, reservation\_logs, reservations, sms\_logs

```



\---



\## STEP 3. Upstash Redis 설정 (Rate Limiting)



\### 3-1. 데이터베이스 생성



1\. \[https://upstash.com](https://upstash.com) → 회원가입/로그인

2\. \*\*Create Database\*\* 클릭

3\. 설정값:



| 항목 | 값 |

|------|-----|

| \*\*Name\*\* | `bagdrop-ratelimit` |

| \*\*Type\*\* | Regional |

| \*\*Region\*\* | \*\*AP-NORTHEAST-1 (Tokyo)\*\* ← Vercel, Supabase와 동일 리전 |

| \*\*TLS\*\* | 활성화 |



4\. 생성 후 \*\*REST API\*\* 탭에서 아래 값 복사:



```

UPSTASH\_REDIS\_REST\_URL=   https://xxxx.upstash.io

UPSTASH\_REDIS\_REST\_TOKEN= AXXXxxxxxxxxxxxxxxxxxxxxxxxx

```



\---



\## STEP 4. 알리고 SMS 설정



\### 4-1. 계정 가입 및 API 키 발급



1\. \[https://aligo.in](https://aligo.in) → 회원가입

2\. \*\*서비스 신청 → 문자(SMS)\*\* 서비스 신청

3\. \*\*API 설정\*\* → API Key 발급



\### 4-2. 발신 번호 등록



1\. \*\*문자발송 → 발신번호 관리\*\* → 번호 등록

2\. 인증 후 심사 완료까지 1\~2 영업일 소요

3\. 등록된 번호가 `ALIGO\_SENDER\_PHONE` 환경변수 값



\### 4-3. 잔액 충전



문자 발송은 건당 요금이 부과됩니다.

\- 단문(SMS, 80바이트 이하): 약 20\~25원/건

\- 장문(LMS, 80바이트 초과): 약 50원/건



BagDrop 발송 이벤트 4종 모두 LMS 범위이므로 약 50원/건 × 4회 = 예약 1건당 약 200원 소요.



\### 4-4. 테스트 발송 확인



API 설정에서 `testmode\_yn=Y`로 설정된 상태에서 테스트 발송을 먼저 진행합니다.



\---



\## STEP 5. 토스페이먼츠 라이브 키 전환



\### 5-1. 테스트 환경 확인



개발 중 사용한 테스트 키 확인:



```

클라이언트 키: test\_ck\_...

시크릿 키:     test\_sk\_...

```



\### 5-2. 라이브 전환 절차



1\. \[https://developers.tosspayments.com](https://developers.tosspayments.com) 로그인

2\. \*\*내 개발정보\*\* → \*\*라이브 전환\*\* 클릭

3\. 사업자 정보 입력 및 서류 제출 (심사 기간: 1\~3 영업일)

4\. 심사 완료 후 \*\*라이브 탭\*\* 에서 키 발급:



```

클라이언트 키(라이브): live\_ck\_...

시크릿 키(라이브):     live\_sk\_...

```



\### 5-3. 웹훅 설정



\*\*개발정보 → 웹훅\*\* 에서:



| 항목 | 값 |

|------|-----|

| \*\*웹훅 URL\*\* | `https://bagdrop.kr/api/payments/webhook` |

| \*\*이벤트\*\* | `PAYMENT\_STATUS\_CHANGED` 체크 |

| \*\*인증 방식\*\* | HMAC-SHA256 |



\*\*웹훅 시크릿\*\*을 발급받아 `TOSS\_WEBHOOK\_SECRET` 환경변수에 설정합니다.



\### 5-4. 허용 도메인 등록



\*\*개발정보 → 허용 도메인\*\* 에서 `bagdrop.kr` 추가.



\---



\## STEP 6. Vercel 프로젝트 설정 및 배포



\### 6-1. Vercel 프로젝트 생성



1\. \[https://vercel.com](https://vercel.com) 로그인

2\. \*\*New Project\*\* → GitHub 연결 → `bagdrop` 레포 선택

3\. Framework: \*\*Next.js\*\* 자동 감지 확인

4\. \*\*Configure Project\*\* 에서 빌드 설정 확인:



| 항목 | 값 |

|------|-----|

| \*\*Framework\*\* | Next.js |

| \*\*Build Command\*\* | `npm run build` |

| \*\*Output Directory\*\* | `.next` (자동) |

| \*\*Install Command\*\* | `npm install` |



\### 6-2. 환경변수 전체 설정



\*\*Settings → Environment Variables\*\* 에서 아래 변수를 \*\*모두\*\* 추가합니다.



\#### Supabase



| 변수명 | 값 | 환경 |

|--------|-----|------|

| `NEXT\_PUBLIC\_SUPABASE\_URL` | `https://xxxx.supabase.co` | All |

| `NEXT\_PUBLIC\_SUPABASE\_ANON\_KEY` | `eyJ...` (anon key) | All |

| `SUPABASE\_SERVICE\_ROLE\_KEY` | `eyJ...` (service\_role key) | All |



> ⚠️ `SUPABASE\_SERVICE\_ROLE\_KEY`는 `NEXT\_PUBLIC\_` 접두사 없이 설정해야 합니다.



\#### NextAuth.js



| 변수명 | 값 | 환경 |

|--------|-----|------|

| `NEXTAUTH\_URL` | `https://bagdrop.kr` | Production |

| `NEXTAUTH\_URL` | `https://staging.bagdrop.kr` | Preview |

| `NEXTAUTH\_SECRET` | (아래 명령어로 생성) | All |



```bash

\# NEXTAUTH\_SECRET 생성 (로컬에서 실행 후 복사)

openssl rand -base64 32

\# 예: dJ3k8f2mNxPq9vR0tY5wA1bC6eH4iL7o

```



\#### 토스페이먼츠



| 변수명 | 값 | 환경 |

|--------|-----|------|

| `NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY` | `live\_ck\_...` | Production |

| `NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY` | `test\_ck\_...` | Preview/Development |

| `TOSS\_SECRET\_KEY` | `live\_sk\_...` | Production |

| `TOSS\_SECRET\_KEY` | `test\_sk\_...` | Preview/Development |

| `TOSS\_WEBHOOK\_SECRET` | (토스페이 어드민에서 발급) | All |



\#### 알리고 SMS



| 변수명 | 값 | 환경 |

|--------|-----|------|

| `ALIGO\_API\_KEY` | (알리고 API Key) | All |

| `ALIGO\_USER\_ID` | (알리고 사용자 ID) | All |

| `ALIGO\_SENDER\_PHONE` | `01000000000` | All |



\#### Upstash Redis



| 변수명 | 값 | 환경 |

|--------|-----|------|

| `UPSTASH\_REDIS\_REST\_URL` | `https://xxxx.upstash.io` | All |

| `UPSTASH\_REDIS\_REST\_TOKEN` | `AXXXxxxx...` | All |



\#### 앱 설정



| 변수명 | 값 | 환경 |

|--------|-----|------|

| `NEXT\_PUBLIC\_APP\_URL` | `https://bagdrop.kr` | Production |

| `NEXT\_PUBLIC\_APP\_URL` | `https://staging.bagdrop.kr` | Preview |

| `ADMIN\_INITIAL\_PASSWORD` | (초기 비밀번호, 시드 실행 후 삭제) | Production |



\### 6-3. 첫 배포



환경변수 설정 완료 후 \*\*Deploy\*\* 버튼 클릭.



```

빌드 로그에서 확인할 사항:

✅ npm install 완료

✅ next build 성공

✅ Route (앱) 목록 출력

✅ Deployment 완료 → 임시 URL 발급 (https://bagdrop-xxx.vercel.app)

```



빌드 실패 시 로그에서 TypeScript 오류 또는 환경변수 누락을 확인합니다.



\### 6-4. 커스텀 도메인 연결



1\. \*\*Settings → Domains\*\* → `bagdrop.kr` 추가

2\. 도메인 등록기에서 DNS 설정:



```

\# A 레코드 (루트 도메인)

Type: A

Name: @

Value: 76.76.21.21   ← Vercel IP



\# CNAME (www 서브도메인)

Type: CNAME

Name: www

Value: cname.vercel-dns.com

```



3\. DNS 전파까지 최대 24시간 소요 (일반적으로 5\~30분)

4\. Vercel에서 SSL 인증서 자동 발급 확인



\### 6-5. `next.config.js` Runtime 설정 추가



bcryptjs는 Node.js Runtime이 필요합니다. 배포 전 `next.config.js`를 확인합니다.



```javascript

/\*\* @type {import('next').NextConfig} \*/

const nextConfig = {

&#x20; serverExternalPackages: \['bcryptjs'],



&#x20; // API Route 기본 런타임을 Node.js로 명시 (Edge Runtime 방지)

&#x20; // 개별 route.ts에서도 아래를 추가할 수 있음

&#x20; // export const runtime = 'nodejs'



&#x20; env: {

&#x20;   NEXT\_PUBLIC\_APP\_URL: process.env.NEXT\_PUBLIC\_APP\_URL,

&#x20; },

}



module.exports = nextConfig

```



\---



\## STEP 7. 초기 데이터 시드 (어드민 계정 생성)



\### 7-1. 로컬에서 시드 실행



```bash

\# 로컬 .env.local에 프로덕션 Supabase 키 임시 설정 후 실행

\# (주의: 프로덕션 DB에 직접 데이터를 입력하는 작업)



npm run seed



\# 출력 예시:

\# 🌱 시드 시작...

\# ✅ 운영자 계정 시드 완료 (username: admin)

\# ⚠️  초기 비밀번호: bagdrop2026! — 배포 전 반드시 변경하세요.

```



\### 7-2. 초기 비밀번호 즉시 변경



> 시드 실행 직후 반드시 비밀번호를 변경하세요. 운영 서버에 기본 비밀번호를 그대로 두면 보안 위험이 됩니다.



```bash

\# 직접 bcrypt 해시 생성 후 Supabase SQL Editor에서 UPDATE

\# 로컬에서:

node -e "const bcrypt=require('bcryptjs'); bcrypt.hash('새\_비밀번호\_입력', 12).then(h => console.log(h))"



\# 출력된 해시를 복사 후 Supabase SQL Editor에서:

UPDATE admin\_users

SET password\_hash = '$2b$12$복사한\_해시값'

WHERE username = 'admin';

```



\### 7-3. Vercel에서 `ADMIN\_INITIAL\_PASSWORD` 환경변수 삭제



시드 완료 후 Vercel 환경변수에서 `ADMIN\_INITIAL\_PASSWORD`를 삭제합니다.



\---



\## STEP 8. 배포 후 스모크 테스트



배포 완료 직후 아래 체크리스트를 순서대로 실행합니다.



\### 8-1. FO (고객 화면) 스모크 테스트



```

□ \[홈] https://bagdrop.kr 접속 → 홈 화면 정상 로드

□ \[홈] "예약하기" 버튼 → /reserve/step1 정상 이동

□ \[STEP 1] T1 선택 → 출국 선택 → 내일 날짜 선택 → "다음으로" 클릭

□ \[STEP 2] 짐 1개 선택 → "다음으로" 클릭

□ \[STEP 3] 이름 입력 → 전화번호 입력 → "다음으로" 클릭

□ \[STEP 4] 예약 요약 카드 정상 표시

□ \[STEP 4] "토스페이로 결제하기" 클릭 → 결제창 정상 호출

□ \[STEP 4] 토스페이 테스트 결제 완료 (테스트 카드 사용)

□ \[STEP 5] 예약번호 + QR 코드 표시 확인

□ \[SMS] 예약자 전화번호로 예약 완료 SMS 수신 확인

□ \[예약 조회] /my-reservation → 예약번호 입력 → 상세 조회 성공

□ \[예약 취소] 예약 상세에서 취소 버튼 → 취소 확인 → CANCELLED 상태 변경

□ \[SMS] 취소 완료 SMS 수신 확인

□ \[이용 안내] /guide 페이지 정상 로드

```



\### 8-2. BO (운영자 화면) 스모크 테스트



```

□ \[로그인] https://bagdrop.kr/admin/login 접속 → 로그인 화면 정상 로드

□ \[로그인] admin / (설정한 비밀번호) → 로그인 성공 → 대시보드 이동

□ \[대시보드] 현황 카드 3개 수치 표시 확인

□ \[접수] "짐 접수하기" 버튼 → /admin/checkin → QR 스캔 화면 로드

□ \[접수] 카메라 권한 허용 → QR 스캐너 활성 확인

□ \[접수] 위에서 생성한 예약 QR 스캔 → 예약 정보 확인 화면 표시

□ \[접수] "접수 완료 처리" → 성공 피드백 → 자동 복귀

□ \[SMS] 짐 접수 완료 SMS 수신 확인

□ \[보관 현황] /admin/storage → 방금 접수한 예약 목록 표시

□ \[반환] 보관 현황에서 예약 탭 → "반환 처리하기" → 반환 완료

□ \[SMS] 짐 반환 완료 SMS 수신 확인

□ \[예약 관리] /admin/reservations → 오늘 예약 목록 필터 확인

□ \[매출 관리] /admin/sales → 일간 매출 조회 확인

□ \[로그아웃] 로그아웃 버튼 → 로그인 화면 이동

```



\### 8-3. API 직접 확인



```bash

\# 예약 조회 API 응답 확인

curl -X GET https://bagdrop.kr/api/reservations/BD-20260715-0001



\# 웹훅 서명 검증 확인 (잘못된 서명 → 400 반환)

curl -X POST https://bagdrop.kr/api/payments/webhook \\

&#x20; -H "Content-Type: application/json" \\

&#x20; -H "Toss-Signature: invalid" \\

&#x20; -d '{"eventType":"PAYMENT\_STATUS\_CHANGED","data":{}}'

\# 기대 응답: {"success":false,"error":{"code":"INVALID\_REQUEST",...}}



\# Rate Limiting 확인 (분당 5회 초과 시 429 반환)

for i in {1..6}; do

&#x20; curl -s -o /dev/null -w "%{http\_code}\\n" \\

&#x20;   -X POST https://bagdrop.kr/api/reservations/prepare \\

&#x20;   -H "Content-Type: application/json" \\

&#x20;   -d '{"terminal":"T1","direction":"DEPARTURE","use\_date":"2099-01-01","bag\_count":1,"customer\_name":"test","customer\_phone":"01000000000"}'

done

\# 6번째 요청에서 429 반환 확인

```



\### 8-4. 성능 확인



```bash

\# Lighthouse 점수 확인 (Chrome DevTools → Lighthouse)

\# 목표: Performance ≥ 80, FCP ≤ 2.5초 (모바일 기준)



\# Core Web Vitals 확인 (Vercel Analytics에서도 확인 가능)

\# - LCP (Largest Contentful Paint) ≤ 2.5초

\# - FID (First Input Delay) ≤ 100ms

\# - CLS (Cumulative Layout Shift) ≤ 0.1

```



\---



\## STEP 9. 운영 중 모니터링



\### 9-1. Vercel Analytics 활성화



\*\*Vercel Dashboard → Analytics → Enable Analytics\*\*



\- 실시간 방문자 수

\- 페이지별 응답 시간

\- 함수 실행 시간

\- 에러율



\### 9-2. Vercel 함수 로그 확인



\*\*Vercel Dashboard → Functions (또는 Logs)\*\*



실시간으로 API Route 실행 로그를 확인할 수 있습니다.



```

필터 키워드로 빠른 검색:

\[confirm]    → 결제 승인 관련 로그

\[webhook]    → 토스페이 웹훅 로그

\[checkin]    → 짐 접수 로그

\[checkout]   → 짐 반환 로그

\[rate-limit] → Rate Limiting 차단 로그

\[SMS]        → SMS 발송 로그

```



\### 9-3. Supabase 모니터링



\*\*Supabase Dashboard → Database → Performance / Logs\*\*



| 탭 | 확인 사항 |

|----|----------|

| \*\*Logs\*\* | API 요청 로그, 쿼리 실행 내역 |

| \*\*Performance\*\* | 슬로우 쿼리 탐지 |

| \*\*Realtime\*\* | 실시간 커넥션 수 |



\*\*주기적 확인 항목:\*\*



```sql

\-- 1. PENDING 상태 payments 누적 확인 (매일)

SELECT COUNT(\*), MIN(created\_at), MAX(created\_at)

FROM payments

WHERE status = 'PENDING'

&#x20; AND created\_at < NOW() - INTERVAL '1 hour';

\-- 1시간 이상 PENDING이면 결제 플로우 문제 의심



\-- 2. 오늘 SMS 실패 건수 확인 (매일)

SELECT event\_type, COUNT(\*) AS fail\_count

FROM sms\_logs

WHERE result = 'FAILED'

&#x20; AND DATE(created\_at AT TIME ZONE 'Asia/Seoul') = CURRENT\_DATE

GROUP BY event\_type;



\-- 3. 장기 보관 중 예약 확인 (매일 운영 마감 전)

SELECT reservation\_no, customer\_name, customer\_phone, checked\_in\_at,

&#x20;      EXTRACT(EPOCH FROM (NOW() - checked\_in\_at)) / 3600 AS hours\_stored

FROM reservations

WHERE status = 'STORED'

ORDER BY checked\_in\_at ASC;



\-- 4. 일별 매출 확인 (매일)

SELECT

&#x20; DATE(paid\_at AT TIME ZONE 'Asia/Seoul') AS sale\_date,

&#x20; COUNT(\*) FILTER (WHERE r.status != 'CANCELLED') AS paid\_count,

&#x20; COUNT(\*) FILTER (WHERE r.status = 'CANCELLED')  AS cancelled\_count,

&#x20; COUNT(\*) FILTER (WHERE r.status != 'CANCELLED') \* 5000 AS gross\_revenue

FROM payments p

JOIN reservations r ON r.id = p.reservation\_id

WHERE p.status = 'PAID'

&#x20; AND p.paid\_at >= NOW() - INTERVAL '7 days'

GROUP BY sale\_date

ORDER BY sale\_date DESC;

```



\### 9-4. 알리고 발송 현황



\[https://aligo.in](https://aligo.in) → \*\*문자발송 → 발송내역\*\* 에서 일별 발송 건수와 실패 건수를 확인합니다.



잔액이 부족하면 발송이 중단됩니다. 잔액 알림 이메일을 설정해 두세요.



\### 9-5. Upstash Redis 모니터링



\[https://console.upstash.com](https://console.upstash.com) → 대시보드에서:



\- 일별 커맨드 수 (무료 플랜: 일 10,000 커맨드)

\- 429 차단 건수 추이

\- 메모리 사용량



\### 9-6. 토스페이 어드민 모니터링



\[https://admin.tosspayments.com](https://admin.tosspayments.com) (라이브 전환 후 접근 가능)



\- 결제 성공/실패율

\- 취소/환불 현황

\- 미환불 취소 건 수동 처리 여부 확인



\---



\## STEP 10. 배포 이후 운영 절차



\### 10-1. 코드 업데이트 배포



```bash

\# 기능 개발 및 버그 수정

git add .

git commit -m "fix: 버그 수정 내용"

git push origin main

\# → Vercel이 자동으로 Production 배포 시작

```



\### 10-2. Preview 환경 활용



```bash

\# 배포 전 Preview에서 먼저 검증

git checkout -b feature/new-feature

git push origin feature/new-feature

\# → PR 생성 시 Vercel이 자동으로 Preview URL 생성

\# → Preview URL에서 테스트 후 main에 merge

```



\### 10-3. 롤백 절차



```

Vercel Dashboard → Deployments → 이전 배포 선택 → "Redeploy" 클릭

→ 약 30초 내 이전 버전으로 복구

```



\### 10-4. 긴급 장애 대응



| 상황 | 확인 포인트 | 조치 |

|------|------------|------|

| 결제 안 됨 | 토스페이 시스템 상태 확인 (status.tosspayments.com) | 장애 공지 후 복구 대기 |

| SMS 안 옴 | 알리고 잔액 / API 서버 상태 확인 | 잔액 충전, 알리고 고객센터 문의 |

| BO 로그인 안 됨 | Vercel 함수 로그에서 `\[auth]` 키워드 검색 | NEXTAUTH\_SECRET 환경변수 확인 |

| DB 접근 오류 | Supabase Status 페이지 확인 | Supabase 장애 시 대기 |

| 502/503 오류 다수 | Vercel 함수 타임아웃 또는 cold start | 함수 실행 시간 Vercel 로그 확인 |



\---



\## 배포 체크리스트 (최종 확인)



```

Supabase:

□ 프로젝트 도쿄 리전 생성 완료

□ 마이그레이션 01\~07 순서대로 실행 완료

□ 테이블 5개 생성 확인

□ RLS 정책 활성화 확인

□ get\_next\_reservation\_no RPC 함수 생성 확인

□ npm run seed 실행 완료 (어드민 계정 생성)

□ 어드민 초기 비밀번호 변경 완료

□ ADMIN\_INITIAL\_PASSWORD 환경변수 삭제



토스페이:

□ 라이브 키 발급 완료

□ 웹훅 URL 등록 완료 (https://bagdrop.kr/api/payments/webhook)

□ 웹훅 시크릿 발급 완료

□ 허용 도메인 등록 완료 (bagdrop.kr)



알리고:

□ API 키 발급 완료

□ 발신 번호 등록 및 심사 완료

□ 문자 잔액 충전 완료

□ testmode\_yn=N 확인 (NODE\_ENV=production 시 자동)



Upstash:

□ Redis 데이터베이스 생성 완료 (도쿄 리전)

□ REST URL, TOKEN 복사 완료



Vercel:

□ GitHub 레포 연결 완료

□ 환경변수 14개 전체 설정 완료

□ 커스텀 도메인 연결 완료 (bagdrop.kr)

□ SSL 인증서 발급 확인

□ Production 배포 성공

□ NEXTAUTH\_URL이 프로덕션 도메인으로 설정됨



스모크 테스트:

□ FO 예약 완료 플로우 확인

□ SMS 수신 확인

□ BO 로그인 및 접수/반환 확인

□ 웹훅 서명 검증 확인

□ Rate Limiting 429 반환 확인

```



\---



\## 환경변수 전체 요약



| 변수명 | 예시값 | 범위 | 필수 |

|--------|--------|------|------|

| `NEXT\_PUBLIC\_SUPABASE\_URL` | `https://xxxx.supabase.co` | 전체 | ✅ |

| `NEXT\_PUBLIC\_SUPABASE\_ANON\_KEY` | `eyJ...` | 전체 | ✅ |

| `SUPABASE\_SERVICE\_ROLE\_KEY` | `eyJ...` | 전체 | ✅ |

| `NEXTAUTH\_URL` | `https://bagdrop.kr` | Production | ✅ |

| `NEXTAUTH\_SECRET` | `openssl rand -base64 32` 결과 | 전체 | ✅ |

| `NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY` | `live\_ck\_...` | 전체 | ✅ |

| `TOSS\_SECRET\_KEY` | `live\_sk\_...` | 전체 | ✅ |

| `TOSS\_WEBHOOK\_SECRET` | (토스페이 발급) | 전체 | ✅ |

| `ALIGO\_API\_KEY` | (알리고 발급) | 전체 | ✅ |

| `ALIGO\_USER\_ID` | (알리고 ID) | 전체 | ✅ |

| `ALIGO\_SENDER\_PHONE` | `01000000000` | 전체 | ✅ |

| `UPSTASH\_REDIS\_REST\_URL` | `https://xxxx.upstash.io` | 전체 | ✅ |

| `UPSTASH\_REDIS\_REST\_TOKEN` | `AXXXxxxx...` | 전체 | ✅ |

| `NEXT\_PUBLIC\_APP\_URL` | `https://bagdrop.kr` | 전체 | ✅ |

| `ADMIN\_INITIAL\_PASSWORD` | (시드 전용, 이후 삭제) | 일시 | - |

