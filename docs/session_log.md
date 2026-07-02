## Session 15 | 2026-07-01 | Vercel 배포 가이드 작성

### 작업 내용
- Supabase 프로젝트 생성 및 DB 마이그레이션 7단계 가이드
- Upstash Redis 설정 가이드
- 알리고 SMS 설정 가이드
- 토스페이 라이브 키 전환 가이드
- Vercel 환경변수 14개 전체 설정 가이드
- 배포 후 스모크 테스트 체크리스트
- 운영 중 모니터링 SQL 쿼리 및 방법
- 긴급 장애 대응 절차
- docs/deployment_guide.md 작성 완료

### 주요 결정사항
- Supabase 리전: ap-northeast-1 (도쿄) — 한국 서비스 latency 최소화
- Upstash 리전: ap-northeast-1 (도쿄) — 동일 리전 권장
- 롤백: Vercel 이전 배포 Redeploy (30초 내 복구)
- 어드민 초기 비밀번호: 시드 후 즉시 변경 필수

---

## Session 14 | 2026-07-01 | 배포 전 버그 수정

### 작업 내용
- 버그 #1: phoneSchema transform으로 전화번호 자동 정규화
- 버그 #2: confirmTossPayment AbortController 8초 타임아웃
- 버그 #3: confirmReservationSchema use_date 포함 확인 (기존 정상)
- 버그 #5: verifyTossWebhookSignature throw → false 반환 + try-catch 이중 방어
- 버그 #6: checkin API 이용일 vs 오늘 날짜 비교 추가
- 버그 #8: step4 sdkError state + Script onError 핸들러
- 버그 #9: DoneFeedback 2초 자동 복귀 + 카운트다운 표시
- 버그 #10: usePolling 공통 훅 + Page Visibility API 적용
- 버그 #11: BottomNav isActive /admin/reservations → 홈 탭 활성
- 버그 #12: middleware.ts Upstash Rate Limiting 추가
- docs/bug_fixes.md 작성 완료

### 주요 결정사항
- Rate Limiting: 환경변수 미설정 시 스킵 (개발 환경 편의)
- Rate Limiting 오류: try-catch로 감싸 서비스 중단 방지
- 이용일 검증: 당일만 허용 (±1일은 정책 확정 후 변경)
- usePolling: onPollRef로 함수 안정화 (불필요한 인터벌 재설정 방지)

### 다음 세션 예고
- Session 15: Vercel 배포 + Supabase 마이그레이션
  - 담당 에이전트: 개발 (가온)

---

## Session 13 | 2026-07-01 | QA 체크리스트 작성

### 작업 내용
- API별 엣지 케이스 체크리스트 (FO/BO 전체)
- 프론트엔드 UX 체크리스트
- 보안 취약점 점검 (인증/입력값/결제/Rate Limiting/개인정보)
- 성능 체크리스트 (DB 인덱스/프론트/Vercel)
- 배포 전 필수 확인 사항 (환경변수/DB/토스페이/알리고/스모크테스트)
- 버그 13개 발견 및 수정 방안 제시
- docs/qa_checklist.md 작성 완료

### 발견된 주요 버그 (배포 전 필수 수정)
- #1 🔴 prepare 전화번호 형식 불일치
- #2 🔴 토스페이 승인 타임아웃 미처리
- #5 🔴 환경변수 미설정 시 uncaught exception
- #3 🟠 confirm use_date 재검증 누락
- #6 🟠 BO 접수 이용일 검증 없음
- #8 🟠 SDK 로드 실패 시 UX 누락
- #12 🟠 FO API Rate Limiting 없음

### 다음 세션 예고
- Session 14: 배포 전 필수 버그 수정 (#1~#8)
  - 담당 에이전트: 개발 (가온)

---

## Session 12 | 2026-07-01 | BO 나머지 화면 구현

### 작업 내용
- bo-utils.ts: 포맷/레이블/에러 메시지 헬퍼 통합
- QRScanner.tsx: html5-qrcode 동적 import + 스캔 가이드 오버레이
- checkin/page.tsx: scan→confirm→done 3단계 화면 상태
- checkout/page.tsx: CheckInOutPage mode="checkout" 재사용
- storage/page.tsx: 30초 폴링 + 경과 시간 경고 + Bottom Sheet
- reservations/page.tsx: 3중 필터 + URL 파라미터 초기 세팅
- reservations/[id]/page.tsx: 강제 취소 모달 + 처리 이력 타임라인
- sales/page.tsx: 기간 탭 + 날짜 네비게이터 + 매출 요약
- docs/bo_remaining_code.md 작성 완료

### 주요 결정사항
- QRScanner: SSR 오류 방지 위해 html5-qrcode 동적 import
- CheckInOutPage: mode prop으로 접수/반환 코드 중복 제거
- storage → checkout 연동: URL 파라미터로 예약번호 전달
- 매출 날짜 이동: 달력 없이 shiftDate() 헬퍼로 처리

### 다음 세션 예고
- Session 13: QA 및 버그 수정
- Session 14: Vercel 배포 + Supabase 마이그레이션

---


## Session 11 | 2026-07-01 | FO 예약 확인/상세/이용 안내 구현

### 작업 내용
- StatusBadge 컴포넌트 (4가지 상태 색상 + 크기 변형)
- 예약 조회 페이지 (대문자 변환/형식 검증/API 조회/오류 인라인)
- 예약 상세 페이지
  - 스켈레톤 UI / 상태 배지 / QR (RESERVED·STORED만)
  - 예약번호 복사 토스트
  - 예약·결제 정보 / 처리 이력 타임라인
  - 취소 케이스 4가지 분기
  - 취소 확인 모달 + 인라인 오류 처리
- 이용 안내 페이지 (4단계/부스위치/FAQ 아코디언)
- docs/fo_mypage_code.md 작성 완료

### 주요 결정사항
- is_cancellable: 서버 반환값 사용 (클라이언트 날짜 재계산 안 함)
- QR 표시: RESERVED/STORED 상태만
- 취소 모달 오류: 모달 내부 인라인 표시 (닫지 않음)
- FAQ: 서버 컴포넌트 + FaqItem만 use client 분리
- 타임라인: 마지막 항목 blue-500 강조

### 다음 세션 예고
- Session 12: BO 나머지 화면 구현
  - 접수/반환/보관현황/예약관리/매출
  - 담당 에이전트: 개발 (가온)

---

## Session 10 | 2026-07-01 | FO 예약 플로우 프론트엔드 구현

### 작업 내용
- reservation-store.ts: sessionStorage 상태 관리 유틸
- FO 홈 페이지 (히어로/이용방법/CTA)
- StepBar 컴포넌트 (진행률 바 + 뒤로가기)
- STEP 1: 터미널/방향 카드 선택 + 미니 달력 (90일 제한)
- STEP 2: 짐 개수 카드 + 초과 안내 + 요금 표시
- STEP 3: 이름/전화번호 + 실시간 포맷 + 유효성 검증
- STEP 4: 예약 요약 + 토스페이 SDK + prepare→결제→confirm
- ReservationQR 컴포넌트 (qrcode.react SVG)
- STEP 5: QR 표시 + 복사 + Web Share API + 뒤로가기 차단
- docs/fo_frontend_code.md 작성 완료

### 주요 결정사항
- sessionStorage: 단일 키 JSON 직렬화 방식
- 달력: 외부 라이브러리 미사용 (직접 구현)
- 토스페이: Promise 방식 (리디렉트 방식 미사용)
- QR 오류 정정: Level M (15%)
- 완료 화면 뒤로가기: popstate 이벤트 → 홈 replace

### 다음 세션 예고
- Session 11: FO 예약 확인/상세 페이지 구현
  - /my-reservation, /my-reservation/:id
  - 담당 에이전트: 개발 (가온)

---

## Session 09 | 2026-07-01 | BO 프론트엔드 구현

### 작업 내용
- NextAuth App Router 핸들러 작성
- BO 로그인 페이지 (폼/에러/로딩/callbackUrl)
- BottomNav 컴포넌트 (5개 탭/활성 판단/safe-area)
- AdminLayout 컴포넌트 (GNB/로그아웃/세션만료 모달)
- 대시보드 페이지 (현황카드/빠른액션/대기목록/30초 폴링)
- docs/bo_frontend_code.md 작성 완료

### 주요 결정사항
- 폴링: useEffect + setInterval 30초 (cleanup 필수)
- 활성탭: startsWith 매칭 (대시보드만 정확 매칭)
- pb-safe: tailwind.config.ts에 safe-area-inset-bottom 추가
- AdminLayout: use client (useSession 클라이언트 훅 필요)

### 다음 세션 예고
- Session 10: FO 예약 플로우 프론트엔드 구현
  - 홈 → STEP

## Session 08 | 2026-07-01 | BO API Route 구현

### 작업 내용
- BO API Route 8개 TypeScript 코드 작성
  - dashboard/route.ts: 병렬 쿼리 4개 + 터미널 필터
  - reservations/route.ts: 동적 필터 + 페이징
  - reservations/[id]/route.ts: toss_payment_key 포함
  - checkin/route.ts: RESERVED → STORED + 낙관적 잠금
  - checkout/route.ts: STORED → RETURNED + 낙관적 잠금
  - cancel/route.ts: 강제 취소 + 환불 안내
  - storage/route.ts: 경과 시간 + 경고 레벨
  - sales/route.ts: 일/주/월 매출 집계
- lib/admin.ts 공통 헬퍼 작성
- docs/bo_api_code.md 작성 완료

### 주요 결정사항
- lib/admin.ts: getAdminSession / fetchReservation / guardStatus 분리
- 낙관적 잠금: UPDATE WHERE status = expected 패턴 전체 적용
- sales: payments ↔ reservations !inner JOIN 활용
- 취소 건수: reservations.cancelled_at 기준 (payments 수동 환불 대응)

### 다음 세션 예고
- Session 09: NextAuth.js 설정 완성 및 BO 로그인 페이지 구현
  - 담당 에이전트: 개발 (가온)

---

## Session 06 | 2026-07-01 | 개발 환경 구성 및 초기 셋업

### 작업 내용
- Next.js 14 App Router 프로젝트 설정
- 패키지 목록 및 설치 명령 정의
- 전체 폴더 구조 확정
- 환경변수 템플릿 작성
- lib/supabase.ts (타입 정의 포함)
- lib/auth.ts (로그인 실패 잠금 포함)
- lib/validate.ts (Zod 스키마 + 헬퍼)
- lib/toss.ts (결제 승인 + 웹훅 서명 검증)
- lib/sms.ts (템플릿 + 비동기 발송)
- middleware.ts (BO 접근 제어)
- scripts/seed.ts (어드민 초기 계정)
- docs/dev_setup.md 작성 완료

## Session 07 | 2026-07-01 | FO API Route 구현

### 작업 내용
- FO API Route 5개 TypeScript 코드 작성
  - prepare/route.ts: toss_order_id 발급 + PENDING INSERT
  - confirm/route.ts: 토스페이 승인 + 예약 확정 + SMS
  - [reservationNo]/route.ts: 예약 조회 + 마스킹
  - [reservationNo]/cancel/route.ts: 취소 처리
  - webhook/route.ts: 웹훅 서명 검증 + 멱등 처리
- docs/fo_api_code.md 작성 완료

### 주요 결정사항
- Supabase 트랜잭션 미지원 → 순차 실행 + 웹훅 복구 방식 채택
- 낙관적 잠금: UPDATE WHERE status = 'RESERVED' 조건 포함
- 전화번호 마스킹: API 응답 조립 시점에서만 적용
- SMS 발송: sendSmsAsync() 비동기 호출 (응답 블로킹 없음)
- 웹훅 멱등 처리: 이미 처리된 건 스킵 후 200 반환

### 다음 세션 예고
- Session 08: BO API Route 구현
  - dashboard / checkin / checkout / cancel / storage / sales
  - 담당 에이전트: 개발 (가온)

---

### 주요 결정사항
- 런타임: Node.js 20 LTS / Next.js 14 App Router
- 유효성 검증: Zod 스키마 통일
- SMS 발송 실패: 예약 롤백 없음 / sms_logs FAILED 기록
- bcryptjs 사용 (native 빌드 불필요)
- 개발 환경: 알리고 testmode_yn=Y (실제 발송 안 함)

### 다음 세션 예고
- Session 07: API Route 구현
  - FO: prepare → confirm → 예약 조회/취소
  - 담당 에이전트: 개발 (가온)

## Session 05 | 2026-07-01 | API 엔드포인트 설계

### 작업 내용
- Next.js 14 App Router 기준 API 설계
- FO API 5종, BO API 10종 상세 정의
- 토스페이 결제 연동 흐름 설계
- 알리고 SMS 연동 유틸 설계
- 미들웨어 및 인증 처리 설계
- API 파일 구조 정의
- docs/api_spec.md 작성 완료

### 주요 결정사항
- FO API: 인증 없음 / BO API: NextAuth.js 세션 필수
- Supabase service_role 키: 서버사이드 전용 (클라이언트 노출 금지)
- SMS 발송 실패: 예약 롤백 없음 / sms_logs에 FAILED 기록
- 강제 취소 환불: 토스페이 콘솔 수동 처리 (API 자동 호출 없음)
- toss_order_id 형식: bagdrop_{YYYYMMDD}_{8자리 랜덤 hex}

### 다음 세션 예고
- Session 06: 디자인 에이전트 다올 착수 또는 개발 에이전트 가온 착수

---

## Session 04 | 2026-07-01 | DB 스키마 설계

### 작업 내용
- Supabase PostgreSQL 기준 DB 스키마 설계
- 테이블 5종 정의 (reservations, payments, reservation_logs, sms_logs, admin_users)
- Enum 타입 6종 정의
- 인덱스 설계
- RLS 정책 설계
- 초기 시드 스크립트 작성
- docs/db_schema.md 작성 완료

### 주요 결정사항
- 예약번호 채번: PostgreSQL 날짜별 Sequence + RPC 함수
- 모든 DB 접근: Next.js API Routes → service_role 키 전용
- FO/BO 클라이언트에 anon 키 미노출
- 강제 취소 환불: 토스페이 콘솔 수동 처리 (DB는 상태 전환만)

### 다음 세션 예고
- Session 05: API 엔드포인트 설계 또는 디자인 에이전트 다올 착수

---

## Session 03 | 2026-07-01 | BO 화면별 상세 요구사항 명세

### 작업 내용
- BO 전체 8개 화면 상세 요구사항 명세 작성
  - 로그인, 대시보드, 짐 접수, 짐 반환
  - 보관현황, 예약관리, 예약상세, 매출관리
- 공통 정의 (GNB, 하단탭, 오류처리, 로딩)
- docs/bo_screen_spec.md 작성 완료

### 주요 결정사항
- 어드민 계정 단일화 (T1/T2 통합 조회/처리)
- PG사 토스페이 단일 사용 (카카오페이 제거)
- 강제 취소: 상태 CANCELLED 전환만 수행
- 실제 환불은 토스페이 어드민 콘솔 수동 처리

### 다음 세션 예고
- Session 04: DB 스키마 설계 또는 디자인 에이전트 다올 착수
  - 담당 에이전트: 기획(여울) 또는 디자인(다올)

---

## Session 02 | 2026-07-01 | FO 화면별 상세 요구사항 명세

### 작업 내용
- FO 전체 8개 화면 상세 요구사항 명세 작성
  - 홈, STEP1~5, 예약확인, 예약상세
- 화면별 레이아웃, 컴포넌트, 유효성 검증 정의
- 취소 케이스 4가지 분기 처리 정의
- 공통 정의 (스텝바, 뒤로가기, 로딩, 오류처리)
- docs/fo_screen_spec.md 작성 완료

### 주요 결정사항
- 예약번호 형식: BD-YYYYMMDD-NNNN
- QR 크기: 최소 200×200px
- 휴대폰 번호 마스킹: 중간 4자리
- 취소 버튼: RESERVED + 이용일 00시 이전만 활성
- 브라우저 백버튼: 예약완료 화면에서 홈으로 이동

### 다음 세션 예고
- Session 03: BO 화면별 상세 요구사항 명세
  - 로그인 / 대시보드 / 짐 접수·반환 우선
  - 담당 에이전트: 기획 (여울)

---

## Session 01 | 2026-07-01 | 서비스 기획 확정

### 작업 내용
- BagDrop 서비스 개요 및 핵심 시나리오 2종 확정
- FO/BO 전체 기능 목록 작성 (우선순위 분류 포함)
- FO/BO 화면 IA 초안 작성
- 예약 상태 흐름 4단계 확정
- 기술 스택 제안 확정
- docs/feature_spec.md 작성 완료

### 주요 결정사항
- 예약 1건당 짐 최대 2개 / 초과 시 별도 예약
- T1/T2 독립 운영 / BO 계정 터미널별 분리
- 취소 정책: 이용일 00시 이전 100% 환불 / 이후 취소 불가
- 현장 수기 접수 불가 / 온라인 예약 고객만 접수 가능
- 기술 스택: Next.js + Supabase + 카카오페이/토스페이 + Vercel

### 다음 세션 예고
- Session 02: 화면별 상세 요구사항 명세 (FO 예약 플로우 우선)
  - 담당 에이전트: 기획 (여울)

---

# 통합 세션 로그

> 모든 세션의 주요 의사결정 및 작업 이력을 최신순으로 기록합니다.

---

## Session 00 | 2025-06-24 | 프로젝트 초기 설정

### 작업 내용
- GitHub repo 생성 (startfac-dev-agent)
- 폴더 구조 생성 (.agents/rules, docs)
- 에이전트 규칙 파일 5종 작성 완료
  - pm.md (한결): 일정/세션 관리
  - planner.md (여울): 기획/문서
  - designer.md (다올): UI/UX
  - developer.md (가온): 코드 구현
  - qa.md (바른): 테스트/검증
- session_roadmap.md 작성 완료
- session_log.md 작성 완료
- README.md 작성 완료 (BagDrop 서비스 개요)

### 주요 결정사항
- 서비스명: BagDrop
- 서비스 개요: 인천공항 터미널 입구 짐 보관 서비스
- 요금: 왕복 1회 5,000원
- 구성: 고객용 모바일웹(FO) + 운영자용 어드민(BO)
- 에이전트 크루 5명 역할 확정
- 기획 우선 원칙 채택 (기획 완료 후 개발 착수)
- 브랜치 전략 확정 (feat/fix/docs prefix 사용)
- main 브랜치 직접 커밋 금지

### 다음 세션 예고
- Session 01: BagDrop 주요 기능 목록 도출 및 기술 스택 확정
  - 담당 에이전트: 기획 (여울)

---

## 로그 작성 가이드

새 세션 시작 시 아래 형식으로 위에 추가하세요 (최신순 유지):

## Session 번호 | 날짜 | 세션 제목

### 작업 내용
- 완료한 작업 목록

### 주요 결정사항
- 이번 세션에서 확정된 사항

### 다음 세션 예고
- 다음에 할 작업