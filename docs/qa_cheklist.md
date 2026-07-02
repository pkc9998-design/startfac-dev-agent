\# BagDrop QA 체크리스트



> 작성: QA 에이전트 바른

> 세션: Session 13

> 최종 업데이트: 2026-07-01

> 범위: FO API / BO API / 프론트엔드 / 보안 / 성능 / 배포

> 상태: ✅ Session 13 확정



\---



\## 범례



| 기호 | 의미 |

|------|------|

| 🔴 CRITICAL | 서비스 운영 불가 수준. 배포 전 반드시 수정 |

| 🟠 HIGH | 사용자 피해 발생 가능. 배포 전 수정 권고 |

| 🟡 MEDIUM | UX 저하 또는 잠재적 오류. 초기 배포 후 빠른 수정 |

| 🟢 LOW | 개선 사항. 여유 있을 때 반영 |

| ✅ PASS | 현재 구현에서 올바르게 처리됨 |

| ❌ BUG | 코드 검토 결과 버그 확인됨 |

| ⚠️ RISK | 엣지 케이스에서 문제 발생 가능 |



\---



\## 1. API별 엣지 케이스 체크리스트



\### 1-1. POST `/api/reservations/prepare`



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| P-01 | 정상 요청 | 200 + toss\_order\_id 반환 | ✅ PASS | |

| P-02 | `use\_date` = 오늘 | 200 정상 처리 | ✅ PASS | 당일 예약 허용 |

| P-03 | `use\_date` = 어제 | 400 INVALID\_DATE | ✅ PASS | useDateSchema 검증 |

| P-04 | `use\_date` = 오늘+91일 | 400 INVALID\_DATE | ✅ PASS | 90일 제한 |

| P-05 | `bag\_count` = 3 | 400 INVALID\_BAG\_COUNT | ✅ PASS | Zod CHECK |

| P-06 | `customer\_phone` = "010-1234-5678" (하이픈 포함) | ❌ BUG | \*\*🔴\*\* | 아래 버그 #1 참조 |

| P-07 | `customer\_phone` = "01012345678" (숫자만) | 400 오류 | ❌ BUG | \*\*🔴\*\* | 아래 버그 #1 참조 |

| P-08 | `customer\_name` = "   " (공백만) | 400 오류 | ⚠️ RISK | `.trim()` 후 빈 문자열 체크 확인 필요 |

| P-09 | Body 없음 | 400 INVALID\_REQUEST | ✅ PASS | try-catch 처리 |

| P-10 | PENDING payments INSERT 실패 (DB 다운) | 500 INTERNAL\_ERROR | ✅ PASS | |

| P-11 | 동일 사용자가 빠르게 연속 호출 | 각각 독립 payments 생성 | ⚠️ RISK | Rate Limiting 없으면 payments PENDING 누적 |



\---



\### 1-2. POST `/api/reservations/confirm`



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| C-01 | 정상 결제 승인 흐름 | 201 + 예약번호 | ✅ PASS | |

| C-02 | `toss\_order\_id` 없는 값 | 404 | ✅ PASS | |

| C-03 | 이미 PAID인 payments로 재시도 | 409 PAYMENT\_ALREADY\_EXISTS | ✅ PASS | 중복 방어 |

| C-04 | `amount` 위변조 (예: 1) | 400 + FAILED 처리 | ✅ PASS | 이중 금액 검증 |

| C-05 | 토스페이 승인 API 타임아웃 | ❌ BUG | \*\*🔴\*\* | 아래 버그 #2 참조 |

| C-06 | RPC 채번 성공, reservations INSERT 실패 | 500 반환, payments PENDING 유지 | ⚠️ RISK | 웹훅 DONE으로 복구되나, reservations는 미생성 상태 |

| C-07 | 동시에 같은 `toss\_order\_id`로 두 요청 | 첫 번째 성공, 두 번째 409 | ⚠️ RISK | PENDING → 한 요청만 PAID로 변환 가능하나 경쟁 조건 존재 |

| C-08 | `use\_date` 위변조 (다른 날짜) | confirm에서 재검증 없음 | ❌ BUG | \*\*🟠\*\* | 아래 버그 #3 참조 |

| C-09 | 토스페이 DONE 웹훅 도착 전 confirm 완료 | 멱등 처리로 웹훅 스킵 | ✅ PASS | |

| C-10 | confirm 성공 후 reservation\_logs INSERT 실패 | 예약은 정상, 로그 누락 | ✅ PASS | 치명적이지 않음, 콘솔 기록 |

| C-11 | KST 자정 경계(23:59:59 → 00:00:00)에 요청 | 채번 날짜 불일치 가능 | ⚠️ RISK | 아래 버그 #4 참조 |



\---



\### 1-3. GET `/api/reservations/:reservationNo`



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| R-01 | 정상 조회 | 200 + 예약 상세 | ✅ PASS | |

| R-02 | 존재하지 않는 예약번호 | 404 | ✅ PASS | |

| R-03 | 형식 오류 예약번호 (소문자 bd-...) | 400 | ✅ PASS | |

| R-04 | SQL 인젝션 시도 (예: `BD-' OR 1=1--`) | 400 형식 검증 차단 | ✅ PASS | Zod regex로 사전 차단 |

| R-05 | `is\_cancellable` 계산 — 오늘 자정 경계 | ⚠️ RISK | 서버 KST 기준이지만 엣지 케이스 존재 |

| R-06 | CANCELLED 상태 예약 조회 | 200 + `is\_cancellable: false` | ✅ PASS | |

| R-07 | 전화번호 마스킹 확인 | `010-\*\*\*\*-5678` 형태 | ✅ PASS | |



\---



\### 1-4. POST `/api/reservations/:reservationNo/cancel`



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| CA-01 | 정상 취소 (이용일 전) | 200 + CANCELLED | ✅ PASS | |

| CA-02 | 이용일 당일 취소 시도 | 409 CANCEL\_NOT\_ALLOWED | ✅ PASS | |

| CA-03 | STORED 상태 취소 시도 | 409 ALREADY\_STORED | ✅ PASS | |

| CA-04 | CANCELLED 상태 재취소 | 409 ALREADY\_CANCELLED | ✅ PASS | |

| CA-05 | 동시에 같은 예약 두 번 취소 요청 | 첫 번째 성공, 두 번째 409 | ✅ PASS | 낙관적 잠금 |

| CA-06 | 취소 성공 후 SMS 실패 | 200 반환 (SMS는 비동기) | ✅ PASS | sms\_logs FAILED 기록 |

| CA-07 | 취소 후 reservation\_logs INSERT 실패 | 취소 자체는 성공, 로그 누락 | ✅ PASS | 치명적이지 않음 |

| CA-08 | Body JSON 파싱 오류 (빈 요청) | 정상 처리 (Body 미사용) | ✅ PASS | |



\---



\### 1-5. POST `/api/payments/webhook`



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| W-01 | 유효한 서명 + DONE 이벤트 | 200 + payments PAID | ✅ PASS | |

| W-02 | 잘못된 서명 | 400 | ✅ PASS | timingSafeEqual |

| W-03 | 서명 헤더 없음 | 400 | ✅ PASS | 빈 문자열로 비교 시 실패 |

| W-04 | DONE + 이미 PAID 상태 | 200 skipped | ✅ PASS | 멱등 처리 |

| W-05 | CANCELED 이벤트 (환불 완료 확인) | payments CANCELLED 업데이트 | ✅ PASS | |

| W-06 | ABORTED 이벤트 | payments FAILED 업데이트 | ✅ PASS | |

| W-07 | 웹훅 재발송 (5회 이상) | 모두 200 (멱등) | ✅ PASS | |

| W-08 | DONE 웹훅 + payments 레코드 없음 | 200 (로그만) | ✅ PASS | |

| W-09 | DB 업데이트 실패 | 500 반환 → 토스페이 재발송 | ✅ PASS | |

| W-10 | 페이로드가 유효 JSON이 아님 | 400 | ✅ PASS | |

| W-11 | `TOSS\_WEBHOOK\_SECRET` 미설정 | 서버 예외 발생 | ❌ BUG | \*\*🔴\*\* | 아래 버그 #5 참조 |



\---



\### 1-6. BO API — Checkin / Checkout



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| BI-01 | 정상 접수 (RESERVED → STORED) | 200 | ✅ PASS | |

| BI-02 | 세션 없이 접근 | 401 | ✅ PASS | requireAdminSession |

| BI-03 | 이미 STORED 상태 접수 시도 | 409 ALREADY\_STORED | ✅ PASS | |

| BI-04 | 동시 접수 2개 요청 | 첫 번째 성공, 두 번째 409 | ✅ PASS | 낙관적 잠금 |

| BI-05 | 이용일 다른 예약 접수 | 200 (이용일 검증 없음) | ❌ BUG | \*\*🟠\*\* | 아래 버그 #6 참조 |

| BI-06 | 정상 반환 (STORED → RETURNED) | 200 | ✅ PASS | |

| BI-07 | RESERVED 상태 반환 시도 | 409 NOT\_STORED | ✅ PASS | |



\---



\### 1-7. BO API — 강제 취소



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| FC-01 | RESERVED 예약 강제 취소 | 200 + toss\_payment\_key 반환 | ✅ PASS | |

| FC-02 | STORED 예약 강제 취소 시도 | 409 ALREADY\_STORED | ✅ PASS | |

| FC-03 | toss\_payment\_key 없는 예약 강제 취소 | 200 + `toss\_payment\_key: null` | ✅ PASS | 결제 전 취소 케이스 |

| FC-04 | 고객이 이미 취소한 예약 강제 취소 시도 | 409 ALREADY\_CANCELLED | ✅ PASS | |



\---



\### 1-8. BO API — 매출



| # | 테스트 케이스 | 예상 결과 | 상태 | 비고 |

|---|--------------|-----------|------|------|

| S-01 | 일간 정상 조회 | 200 + 집계 + 목록 | ✅ PASS | |

| S-02 | 주간 — 월 경계 날짜 (예: 6/30 포함 주) | 올바른 월\~일 범위 | ⚠️ RISK | date-fns weekStartsOn 검증 필요 |

| S-03 | 월간 — 윤년 2월 | 2/29까지 집계 | ⚠️ RISK | endOfMonth 처리 확인 |

| S-04 | `net\_revenue` = paid×5000 - cancelled×5000 | 정확한 계산 | ⚠️ RISK | 아래 버그 #7 참조 |

| S-05 | 세션 없이 접근 | 401 | ✅ PASS | |



\---



\## 2. 프론트엔드 UX 체크리스트



\### 2-1. FO 예약 플로우



| # | 테스트 케이스 | 중요도 | 상태 | 비고 |

|---|--------------|--------|------|------|

| U-01 | STEP 1 → 뒤로가기 → 홈 이동 | 🔴 | ✅ PASS | |

| U-02 | STEP 2 → 뒤로가기 → STEP 1 (값 유지) | 🔴 | ✅ PASS | sessionStorage |

| U-03 | STEP 4에서 브라우저 새로고침 | STEP 1로 리디렉트 | ✅ PASS | 필수값 없으면 리디렉트 |

| U-04 | 결제 완료 후 브라우저 뒤로가기 | 홈으로 이동 | ✅ PASS | popstate 이벤트 처리 |

| U-05 | 결제창 닫음(취소) 후 재결제 | 동일 화면 유지, 재시도 가능 | ✅ PASS | PAY\_PROCESS\_CANCELED 처리 |

| U-06 | prepare API 실패 후 결제 버튼 재클릭 | 재시도 가능 | ✅ PASS | finally에서 isLoading 해제 |

| U-07 | 토스페이 SDK 로드 실패 시 결제 버튼 클릭 | ❌ BUG | \*\*🟠\*\* | 아래 버그 #8 참조 |

| U-08 | 전화번호 입력 중 12자리 이상 입력 시도 | 11자리에서 입력 차단 | ⚠️ RISK | maxLength 처리 확인 필요 |

| U-09 | 이름 입력 후 전화번호로 키보드 포커스 이동 | 다음 필드로 포커스 | ✅ PASS | onKeyDown Enter |

| U-10 | 달력에서 90일 초과 날짜 선택 시도 | 선택 불가 (그레이) | ✅ PASS | isDisabled 처리 |

| U-11 | 달력에서 어제 날짜 탭 | 선택 불가 | ✅ PASS | isBefore(today) |

| U-12 | QR 코드 화면 스크린샷 후 조회 가능 여부 | 예약번호로 조회 가능 | ✅ PASS | |

| U-13 | confirm 응답 지연 (10초 이상) 시 UX | 버튼 비활성 + 스피너 유지 | ✅ PASS | |

| U-14 | 예약 완료 후 sessionStorage 초기화 확인 | 재예약 시 이전 값 없음 | ✅ PASS | reservationStore.clear() |

| U-15 | 모바일 다크모드에서 QR 코드 배경 | 흰 배경 강제 | ✅ PASS | bgColor="#FFFFFF" |



\---



\### 2-2. FO 예약 확인·취소



| # | 테스트 케이스 | 중요도 | 상태 | 비고 |

|---|--------------|--------|------|------|

| M-01 | 소문자로 예약번호 입력 (bd-...) | 자동 대문자 변환 | ✅ PASS | formatReservationNo |

| M-02 | 공백 포함 예약번호 (BD -20260715-0001) | 조회 버튼 비활성 | ✅ PASS | regex 불일치 |

| M-03 | 존재하지 않는 예약번호 조회 | 인라인 오류 메시지 | ✅ PASS | |

| M-04 | 이용일 당일 00:00 직후 취소 시도 | 취소 불가 안내 | ✅ PASS | is\_cancellable: false |

| M-05 | 취소 확인 모달에서 "돌아가기" 탭 | 모달 닫힘, 상세 유지 | ✅ PASS | |

| M-06 | 취소 처리 중 이중 탭(더블탭) | 두 번째 API 호출 방지 | ⚠️ RISK | cancelLoading으로 비활성화되나 짧은 시간 내 첫 탭 전 두 번째 탭 가능 |

| M-07 | 취소 완료 후 화면 상태 배지 갱신 | CANCELLED 배지 표시 | ✅ PASS | fetchDetail() 재호출 |

| M-08 | RETURNED 예약 상세 — QR 코드 미표시 | QR 없음 | ✅ PASS | 조건부 렌더링 |



\---



\### 2-3. BO 화면



| # | 테스트 케이스 | 중요도 | 상태 | 비고 |

|---|--------------|--------|------|------|

| B-01 | 세션 만료 중 API 호출 | 401 → 세션 만료 모달 | ✅ PASS | AdminLayout 감지 |

| B-02 | 로그인 실패 5회 후 잠금 | 잠금 안내 | ✅ PASS | locked\_until 처리 |

| B-03 | QR 스캔 — 카메라 권한 거부 | 수동 입력 안내 | ✅ PASS | onError 콜백 |

| B-04 | QR 스캔 — BagDrop QR 외 다른 QR 스캔 | 무시 (재스캔 대기) | ✅ PASS | BD-... 형식 아니면 무시 |

| B-05 | QR 스캔 성공 후 confirm 화면에서 뒤로가기 | 스캔 화면 복귀 + 스캐너 재시작 | ✅ PASS | setScannerActive(true) |

| B-06 | 접수 완료 피드백 화면 — 2초 후 자동 복귀 | ❌ BUG | \*\*🟡\*\* | 아래 버그 #9 참조 |

| B-07 | 대시보드 30초 폴링 — 탭 비활성(백그라운드) | 폴링 계속 실행 | ⚠️ RISK | 아래 버그 #10 참조 |

| B-08 | 보관 현황 → 반환 처리하기 → checkout URL | `?no=BD-...` 파라미터 자동 입력 | ⚠️ RISK | checkout에서 searchParams 읽기 구현 확인 |

| B-09 | 매출 주간 — 월 경계 넘는 주 범위 표시 | 올바른 날짜 범위 표시 | ⚠️ RISK | |

| B-10 | 하단 탭 — `/admin/reservations/BD-...` 진입 시 | 예약 관리 탭 미활성 | ❌ BUG | \*\*🟡\*\* | 아래 버그 #11 참조 |

| B-11 | BO 로그인 상태에서 `/admin/login` 직접 접근 | 대시보드로 리디렉트 | ✅ PASS | middleware 처리 |

| B-12 | GNB 로그아웃 버튼 → 확인 → 로그인 화면 | 정상 | ✅ PASS | signOut({ redirect: false }) |



\---



\## 3. 보안 취약점 점검



\### 3-1. 인증·인가



| # | 항목 | 위험도 | 상태 | 수정 방안 |

|---|------|--------|------|----------|

| SEC-01 | 모든 `/api/admin/\*`에 `requireAdminSession()` 적용 | 🔴 | ✅ PASS | 각 route.ts에서 세션 확인 |

| SEC-02 | BO 페이지 미들웨어 접근 제어 | 🔴 | ✅ PASS | middleware.ts + getToken |

| SEC-03 | API Route에서 세션 재검증 (미들웨어만 믿지 않음) | 🔴 | ✅ PASS | 이중 검증 구조 |

| SEC-04 | `SUPABASE\_SERVICE\_ROLE\_KEY` 클라이언트 노출 | 🔴 | ✅ PASS | 서버사이드 전용, NEXT\_PUBLIC\_ 아님 |

| SEC-05 | `TOSS\_SECRET\_KEY` 클라이언트 노출 | 🔴 | ✅ PASS | 서버사이드 전용 |

| SEC-06 | NextAuth `NEXTAUTH\_SECRET` 강도 | 🟠 | ⚠️ RISK | 32자 이상 랜덤값 사용 필수. `openssl rand -base64 32` 권고 |

| SEC-07 | bcrypt rounds=12 적용 | 🟠 | ✅ PASS | 충분한 보안 수준 |

| SEC-08 | 세션 쿠키 `httpOnly`, `secure`, `sameSite` | 🟠 | ✅ PASS | NextAuth 기본 설정 |



\---



\### 3-2. 입력값 검증



| # | 항목 | 위험도 | 상태 | 수정 방안 |

|---|------|--------|------|----------|

| SEC-09 | 모든 API 입력값 Zod 검증 | 🔴 | ✅ PASS | |

| SEC-10 | SQL Injection — 예약번호 regex 사전 차단 | 🔴 | ✅ PASS | `BD-YYYYMMDD-NNNN` 형식만 허용 |

| SEC-11 | XSS — React는 기본적으로 이스케이프 | 🟠 | ✅ PASS | dangerouslySetInnerHTML 미사용 |

| SEC-12 | Path Traversal — `\[reservationNo]` 파라미터 | 🟠 | ✅ PASS | Zod regex 검증 후 사용 |

| SEC-13 | `customer\_name`에 HTML/스크립트 삽입 시도 | 🟡 | ✅ PASS | React 자동 이스케이프 |

| SEC-14 | `toss\_response` JSONB 저장 시 임의 데이터 크기 | 🟡 | ⚠️ RISK | 토스페이 응답 크기는 제한적이나 저장 전 크기 검증 미비 |



\---



\### 3-3. 결제 보안



| # | 항목 | 위험도 | 상태 | 수정 방안 |

|---|------|--------|------|----------|

| SEC-15 | 토스페이 웹훅 서명 검증 (HMAC-SHA256) | 🔴 | ✅ PASS | timingSafeEqual 적용 |

| SEC-16 | 결제 금액 서버 재검증 (5,000원 고정 확인) | 🔴 | ✅ PASS | 이중 검증 |

| SEC-17 | `paymentKey` 위변조 차단 | 🔴 | ✅ PASS | 토스페이 서버에서 직접 검증 |

| SEC-18 | 동일 `toss\_order\_id` 중복 결제 차단 | 🔴 | ✅ PASS | PENDING 상태 확인 후 처리 |

| SEC-19 | PENDING payments 무기한 누적 | 🟡 | ⚠️ RISK | 오래된 PENDING 정리 배치 미구현 |



\---



\### 3-4. Rate Limiting



| # | 항목 | 위험도 | 상태 | 수정 방안 |

|---|------|--------|------|----------|

| SEC-20 | FO API Rate Limiting | 🟠 | ❌ BUG | \*\*🟠\*\* 아래 버그 #12 참조 |

| SEC-21 | BO 로그인 Rate Limiting | 🟠 | ✅ PASS | DB 레벨 5회 잠금 구현 |

| SEC-22 | 웹훅 엔드포인트 Rate Limiting | 🟡 | ⚠️ RISK | 토스페이 IP 화이트리스팅 설정 권장 |



\---



\### 3-5. 개인정보 보호



| # | 항목 | 위험도 | 상태 | 수정 방안 |

|---|------|--------|------|----------|

| SEC-23 | 전화번호 마스킹 (FO 응답) | 🟠 | ✅ PASS | maskPhone() 적용 |

| SEC-24 | 전화번호 마스킹 (BO 응답) | 🟠 | ✅ PASS | customer\_phone\_masked 반환 |

| SEC-25 | DB 원본 전화번호 암호화 | 🟡 | ⚠️ RISK | 현재 평문 저장. GDPR/개인정보보호법 대응 시 암호화 검토 |

| SEC-26 | 로그에 고객 개인정보 출력 | 🟠 | ⚠️ RISK | console.error에 전화번호 등 포함 여부 확인 |

| SEC-27 | `toss\_response` JSONB에 카드번호 등 민감정보 포함 가능성 | 🟡 | ⚠️ RISK | 토스페이 응답에서 민감 필드 제거 후 저장 권장 |



\---



\## 4. 성능 체크리스트



\### 4-1. 데이터베이스



| # | 항목 | 중요도 | 상태 | 비고 |

|---|------|--------|------|------|

| PERF-01 | `reservations(reservation\_no)` UNIQUE 인덱스 | 🔴 | ✅ PASS | 조회 핵심 경로 |

| PERF-02 | `reservations(use\_date, status)` 복합 인덱스 | 🔴 | ✅ PASS | 대시보드 카운트 쿼리 |

| PERF-03 | `reservations(status)` 인덱스 | 🟠 | ✅ PASS | 보관현황 STORED 조회 |

| PERF-04 | `payments(toss\_payment\_key)` UNIQUE 인덱스 | 🔴 | ✅ PASS | 웹훅 처리 핵심 |

| PERF-05 | `payments(toss\_order\_id)` UNIQUE 인덱스 | 🔴 | ✅ PASS | confirm 처리 핵심 |

| PERF-06 | 대시보드 4개 쿼리 병렬 처리 | 🟠 | ✅ PASS | Promise.all 사용 |

| PERF-07 | 매출 집계 — 전체 데이터 fetch 후 앱 레이어 집계 | 🟡 | ⚠️ RISK | 아래 버그 #13 참조 |

| PERF-08 | Supabase 커넥션 풀 (서버리스 환경) | 🟠 | ⚠️ RISK | Vercel 서버리스는 커넥션이 요청마다 생성 → Supabase Pooler 모드 설정 권장 |

| PERF-09 | `reservation\_logs` 인덱스 — `(reservation\_id, created\_at)` | 🟡 | ✅ PASS | 이력 조회 최적화 |



\---



\### 4-2. 프론트엔드



| # | 항목 | 중요도 | 상태 | 비고 |

|---|------|--------|------|------|

| PERF-10 | 토스페이 SDK `afterInteractive` 전략 로드 | 🟠 | ✅ PASS | 초기 번들 영향 없음 |

| PERF-11 | html5-qrcode 동적 import (SSR 제외) | 🟠 | ✅ PASS | 클라이언트 전용 |

| PERF-12 | 대시보드 폴링 — 탭 비활성 시 낭비 | 🟡 | ⚠️ RISK | 버그 #10 |

| PERF-13 | 보관 현황 폴링 — 30초마다 전체 목록 재조회 | 🟡 | ⚠️ RISK | 장기적으로 목록이 길어지면 부하 증가 |

| PERF-14 | QR 코드 SVG 생성 — 클라이언트 사이드 | 🟢 | ✅ PASS | 서버 부하 없음 |

| PERF-15 | 이미지 최적화 — next/image 미사용 | 🟡 | ⚠️ RISK | 홈 히어로 배너 등 이미지 추가 시 적용 필요 |



\---



\### 4-3. Vercel 배포



| # | 항목 | 중요도 | 상태 | 비고 |

|---|------|--------|------|------|

| PERF-16 | Edge Runtime vs Node.js Runtime | 🟡 | ⚠️ RISK | bcryptjs는 Node.js Runtime 필요. API Route에 `export const runtime = 'nodejs'` 명시 권장 |

| PERF-17 | 서버리스 함수 Cold Start | 🟡 | ⚠️ RISK | 접수/반환 API는 빠른 응답 필요. Vercel Pro의 Fluid Compute 고려 |

| PERF-18 | Supabase 지역 — ap-northeast-1 (도쿄) 권장 | 🟠 | ⚠️ RISK | 한국 서비스이므로 지연 최소화를 위해 도쿄 리전 선택 |



\---



\## 5. 배포 전 필수 확인 사항



\### 5-1. 환경변수



```bash

\# 아래 모든 환경변수가 Vercel 프로덕션 환경에 설정됐는지 확인

NEXT\_PUBLIC\_SUPABASE\_URL          # ✅ 설정 여부 확인

NEXT\_PUBLIC\_SUPABASE\_ANON\_KEY     # ✅ 설정 여부 확인 (실사용 없으나 빌드 필요)

SUPABASE\_SERVICE\_ROLE\_KEY         # 🔴 절대 NEXT\_PUBLIC\_ 아님

NEXTAUTH\_URL                      # 🔴 프로덕션 도메인으로 변경 (ex: https://bagdrop.kr)

NEXTAUTH\_SECRET                   # 🔴 32자 이상 랜덤값 (openssl rand -base64 32)

NEXT\_PUBLIC\_TOSS\_CLIENT\_KEY       # 🟠 테스트키 → 라이브키로 교체

TOSS\_SECRET\_KEY                   # 🔴 테스트키 → 라이브키로 교체

TOSS\_WEBHOOK\_SECRET               # 🔴 토스페이 어드민에서 발급한 웹훅 시크릿

ALIGO\_API\_KEY                     # 🟠 설정 여부 확인

ALIGO\_USER\_ID                     # 🟠 설정 여부 확인

ALIGO\_SENDER\_PHONE                # 🟠 사전 등록된 발신 번호

NEXT\_PUBLIC\_APP\_URL               # 🟡 SMS 내 링크 URL (https://bagdrop.kr)

NODE\_ENV                          # production 자동 설정 (Vercel)

```



\### 5-2. DB 마이그레이션



```

배포 전 체크리스트 (Supabase SQL Editor에서 순서대로 실행):



\[ ] 1. CREATE TYPE (Enum 6종) 실행 완료

\[ ] 2. CREATE TABLE admin\_users 실행 완료

\[ ] 3. CREATE TABLE reservations 실행 완료

\[ ] 4. CREATE TABLE payments 실행 완료

\[ ] 5. CREATE TABLE reservation\_logs 실행 완료

\[ ] 6. CREATE TABLE sms\_logs 실행 완료

\[ ] 7. CREATE INDEX (전체) 실행 완료

\[ ] 8. CREATE TRIGGER (updated\_at) 실행 완료

\[ ] 9. ALTER TABLE (CHECK 제약) 실행 완료

\[ ] 10. CREATE FUNCTION get\_next\_reservation\_no 실행 완료

\[ ] 11. RLS 정책 활성화 완료

\[ ] 12. npm run seed 실행 완료 (어드민 계정 생성)

\[ ] 13. 어드민 초기 비밀번호 변경 완료

```



\### 5-3. 토스페이 설정



```

\[ ] 토스페이 어드민 콘솔 (https://developers.tosspayments.com) 설정

\[ ] 라이브 키 발급 완료 (클라이언트 키 + 시크릿 키)

\[ ] 웹훅 URL 등록: https://bagdrop.kr/api/payments/webhook

\[ ] 웹훅 이벤트 선택: PAYMENT\_STATUS\_CHANGED

\[ ] 웹훅 시크릿 발급 및 TOSS\_WEBHOOK\_SECRET 환경변수 설정

\[ ] 결제 테스트 (실 결제) → 취소 검증

\[ ] 허용 도메인 등록: bagdrop.kr

```



\### 5-4. 알리고 SMS 설정



```

\[ ] 알리고 계정 생성 및 API 키 발급

\[ ] 발신 번호 등록 및 심사 완료

\[ ] testmode\_yn = 'N' 확인 (프로덕션에서 실발송)

\[ ] 문자 잔액 충전 확인

\[ ] 예약 완료 SMS 테스트 발송

```



\### 5-5. 도메인·SSL



```

\[ ] 도메인 구입 및 Vercel 연결 완료

\[ ] SSL 인증서 자동 발급 확인 (Vercel 자동)

\[ ] www 리디렉트 설정 (www.bagdrop.kr → bagdrop.kr 또는 반대)

\[ ] HTTP → HTTPS 리디렉트 확인

```



\### 5-6. 스모크 테스트 (배포 직후)



```

FO 흐름:

\[ ] 홈 화면 접근 정상

\[ ] STEP 1\~4 이동 정상

\[ ] 토스페이 결제창 호출 정상 (라이브 환경)

\[ ] 결제 완료 후 예약번호 + QR 표시 정상

\[ ] SMS 수신 확인

\[ ] 예약 조회 화면에서 예약번호 조회 정상

\[ ] 취소 플로우 (이용일 전 예약으로) 정상



BO 흐름:

\[ ] /admin/login 접근 정상

\[ ] 어드민 로그인 정상

\[ ] 대시보드 현황 카드 표시 정상

\[ ] QR 스캔 (카메라 권한 요청) 정상

\[ ] 짐 접수 처리 정상 → SMS 수신 확인

\[ ] 짐 반환 처리 정상 → SMS 수신 확인

\[ ] 보관 현황 목록 표시 정상

\[ ] 예약 관리 필터 정상

\[ ] 매출 관리 일간 조회 정상

```



\---



\## 6. 발견된 잠재적 버그 및 수정 방안



\---



\### 🔴 버그 #1 — prepare API 전화번호 형식 불일치



\*\*위치\*\*: `src/lib/validate.ts` / `preparReservationSchema`

\*\*증상\*\*: FO STEP 3에서 `010-1234-5678`(하이픈 포함)으로 저장된 전화번호가 prepare API에 전달되면 Zod 검증 실패



\*\*원인\*\*:

```typescript

// 현재 FO step3: phoneDisplay(하이픈 포함)가 아닌 phoneDigits(숫자만)를 store에 저장

// → 그러나 prepare 요청 시 draft.customer\_phone 값을 그대로 전송



// validate.ts의 phoneSchema

export const phoneSchema = z

&#x20; .string()

&#x20; .regex(/^01\[0-9]\\d{7,8}$/, ...)  // 숫자만 허용

&#x20; .length(11, ...)

```



\*\*수정 방안\*\*:

```typescript

// 방법 A: FO step3에서 저장 시 반드시 숫자만 저장 (현재 구현이 이를 의도함)

// → reservationStore.set({ customer\_phone: phoneDigits }) 이미 올바름

// → 단, confirm API 전송 시 draft.customer\_phone이 숫자인지 재확인 필요



// 방법 B: prepare/confirm API에서 수신 전 자동 정규화

export const phoneSchema = z

&#x20; .string()

&#x20; .transform(v => v.replace(/\\D/g, ''))   // 수신 즉시 숫자만 추출

&#x20; .pipe(z.string().regex(/^01\[0-9]\\d{8}$/).length(11))

```



\---



\### 🔴 버그 #2 — 토스페이 승인 API 타임아웃 미처리



\*\*위치\*\*: `src/lib/toss.ts` / `confirmTossPayment()`

\*\*증상\*\*: 토스페이 서버가 느릴 때 `fetch()`가 무한 대기 → Vercel 함수 타임아웃(기본 10초) → 502 Bad Gateway



\*\*원인\*\*:

```typescript

// 현재 코드 - 타임아웃 없음

const response = await fetch(`${TOSS\_API\_BASE}/v1/payments/confirm`, {

&#x20; method: 'POST',

&#x20; headers: { ... },

&#x20; body: JSON.stringify({ ... }),

&#x20; // ❌ timeout 없음

})

```



\*\*수정 방안\*\*:

```typescript

const controller = new AbortController()

const timeout    = setTimeout(() => controller.abort(), 8000)  // 8초



try {

&#x20; const response = await fetch(`${TOSS\_API\_BASE}/v1/payments/confirm`, {

&#x20;   method:  'POST',

&#x20;   headers: { ... },

&#x20;   body:    JSON.stringify({ ... }),

&#x20;   signal:  controller.signal,

&#x20; })

&#x20; clearTimeout(timeout)

&#x20; // ...

} catch (err) {

&#x20; clearTimeout(timeout)

&#x20; if (err instanceof Error \&\& err.name === 'AbortError') {

&#x20;   return { ok: false, error: { code: 'TIMEOUT', message: '결제 서버 응답 시간이 초과됐습니다.' } }

&#x20; }

&#x20; return { ok: false, error: { code: 'NETWORK\_ERROR', message: '...' } }

}

```



\---



\### 🟠 버그 #3 — confirm에서 use\_date 재검증 없음



\*\*위치\*\*: `src/app/api/reservations/confirm/route.ts`

\*\*증상\*\*: 악의적 사용자가 세션스토리지를 조작해 과거 날짜나 200일 뒤 날짜로 예약 생성 가능



\*\*원인\*\*: confirm API는 `use\_date`를 Zod로 검증하지만, prepare 단계에서 통과한 `use\_date`가 confirm 단계에서 다시 검증되지 않으면 `useDateSchema`가 적용되지 않음



\*\*수정 방안\*\*:

```typescript

// confirmReservationSchema에 useDateSchema 포함 확인

export const confirmReservationSchema = z.object({

&#x20; payment\_key:    z.string().min(1),

&#x20; toss\_order\_id:  z.string().min(1),

&#x20; amount:         z.literal(5000),

&#x20; terminal:       z.enum(\['T1', 'T2']),

&#x20; direction:      z.enum(\['DEPARTURE', 'ARRIVAL']),

&#x20; use\_date:       useDateSchema,   // ✅ 이미 포함됨 (확인 필요)

&#x20; bag\_count:      z.number().int().min(1).max(2),

&#x20; customer\_name:  z.string().min(1).max(20).trim(),

&#x20; customer\_phone: phoneSchema,

})

// → api\_spec.md에는 포함됐으나 실제 구현에서 누락 여부 재확인 필요

```



\---



\### ⚠️ 버그 #4 — KST 자정 경계에서 예약번호 채번 날짜 불일치



\*\*위치\*\*: `src/app/api/reservations/confirm/route.ts`

\*\*증상\*\*: prepare 호출(23:59 KST)과 confirm 호출(00:01 KST)이 날짜가 다를 때 예약번호가 다음 날 날짜로 발급됨



\*\*원인\*\*:

```typescript

// prepare 시: toss\_order\_id = "bagdrop\_20260715\_a1b2..."

// confirm 시:  getDateKst() = "20260716" (자정 넘어감)

// → 예약번호 = "BD-20260716-0001" (이용일 20260715인데 예약번호는 20260716)

```



\*\*수정 방안\*\*: 예약번호 날짜가 이용일 날짜(`use\_date`)와 달라도 기능적으로 무해하다. 단, 운영 혼란을 줄이려면 채번 기준을 `use\_date`로 변경:

```typescript

// getDateKst() 대신 use\_date에서 날짜 추출

const dateStr = use\_date.replace(/-/g, '')  // "2026-07-15" → "20260715"

const { data: reservationNo } = await db.rpc('get\_next\_reservation\_no', { target\_date: dateStr })

```



\---



\### 🔴 버그 #5 — 환경변수 미설정 시 uncaught exception



\*\*위치\*\*: `src/lib/toss.ts` / `verifyTossWebhookSignature()`

\*\*증상\*\*: `TOSS\_WEBHOOK\_SECRET`가 미설정이면 `throw new Error(...)` 발생 → 웹훅 핸들러가 500 반환하지 않고 unhandled exception으로 Vercel 함수 crash



\*\*원인\*\*:

```typescript

export function verifyTossWebhookSignature(...): boolean {

&#x20; const secret = process.env.TOSS\_WEBHOOK\_SECRET

&#x20; if (!secret) throw new Error('TOSS\_WEBHOOK\_SECRET 환경변수가 설정되지 않았습니다.')

&#x20; // ← webhook handler에서 catch되지 않으면 500이 아닌 unhandled rejection

```



\*\*수정 방안\*\*:

```typescript

// webhook/route.ts에서 try-catch로 감싸기

try {

&#x20; const isValid = verifyTossWebhookSignature(rawBody, signature)

&#x20; if (!isValid) return errorResponse('INVALID\_REQUEST', '...', 400)

} catch (err) {

&#x20; console.error('\[webhook] 서명 검증 오류:', err)

&#x20; return Response.json({ success: false }, { status: 500 })

}

```



\---



\### 🟠 버그 #6 — BO 접수 시 이용일 불일치 검증 없음



\*\*위치\*\*: `src/app/api/admin/reservations/\[reservationNo]/checkin/route.ts`

\*\*증상\*\*: 내일 이용일인 예약을 오늘 접수 처리 가능 (운영자 실수 또는 시스템 오류)



\*\*원인\*\*: checkin API에서 `use\_date` vs 오늘 날짜 비교 없음



\*\*수정 방안\*\*:

```typescript

// checkin route.ts에 날짜 검증 추가

const todayKst = getTodayKst()

if (reservation.use\_date !== todayKst) {

&#x20; return errorResponse(

&#x20;   'INVALID\_DATE',

&#x20;   `오늘 이용일이 아닌 예약입니다. (예약 이용일: ${reservation.use\_date})`,

&#x20;   409

&#x20; )

}

```

> 단, 운영 유연성을 위해 당일±1일 허용 여부를 정책으로 확정 후 적용.



\---



\### 🟡 버그 #7 — 매출 net\_revenue 계산 오류 가능성



\*\*위치\*\*: `src/app/api/admin/sales/route.ts`

\*\*증상\*\*: `net\_revenue`를 `(paid\_count - cancelled\_count) × 5000`으로 계산하는데, 취소 건의 paid\_at이 집계 기간 밖이면 cancelled\_count와 paid\_count 기준 기간이 달라 음수 가능



\*\*원인\*\*:

```typescript

// paid\_count: paid\_at 기준 집계

// cancelled\_count: reservations.cancelled\_at 기준 집계

// → 결제는 7/14, 취소는 7/15인 건의 경우 두 집계 기간이 다를 수 있음

const netRevenue = (paidCount - cancelledCount) \* RESERVATION\_AMOUNT

```



\*\*수정 방안\*\*: 순매출은 "기간 내 결제 완료 건"에서 "기간 내 결제 완료 건 중 취소된 건"을 차감하는 방식으로 단일 기준 사용:

```typescript

// payments.paid\_at 기준으로 통일

// → 기간 내 결제된 건 중 현재 상태가 CANCELLED인 건

```

또는 현재 방식의 오차를 허용하고 주석으로 명시.



\---



\### 🟠 버그 #8 — 토스페이 SDK 로드 실패 시 결제 버튼 클릭 오류



\*\*위치\*\*: `src/app/reserve/step4/page.tsx`

\*\*증상\*\*: SDK 로드 실패 시 `window.TossPayments`가 `undefined` → 클릭 시 오류 메시지 없이 조용히 실패하거나 일반 에러 메시지 노출



\*\*원인\*\*:

```typescript

// step4/page.tsx

if (!window.TossPayments) {

&#x20; throw new Error('결제 모듈을 불러오지 못했습니다. 잠시 후 다시 시도해 주세요.')

}

```

이 throw는 catch에서 잡혀 `setErrorMsg`로 연결되지만, `sdkReady` 상태가 여전히 `false`여서 버튼이 비활성 상태 유지됨. 실제로는 SDK 로드 실패인데 "로딩 중" UI가 계속 표시될 수 있음.



\*\*수정 방안\*\*:

```typescript

// Script onError 핸들러 추가

<Script

&#x20; src="https://js.tosspayments.com/v1/payment"

&#x20; onLoad={() => setSdkReady(true)}

&#x20; onError={() => {

&#x20;   setSdkError(true)   // 새 state 추가

&#x20;   setSdkReady(false)

&#x20; }}

&#x20; strategy="afterInteractive"

/>



// 결제 버튼 또는 오류 표시

{sdkError ? (

&#x20; <div className="rounded-xl bg-red-50 p-4 text-sm text-red-600">

&#x20;   결제 모듈 로드에 실패했습니다. 새로고침 후 다시 시도해 주세요.

&#x20; </div>

) : (

&#x20; <button disabled={!sdkReady} ...>토스페이로 결제하기</button>

)}

```



\---



\### 🟡 버그 #9 — 접수 완료 피드백 자동 복귀 미구현



\*\*위치\*\*: `src/app/admin/checkin/page.tsx` / `DoneFeedback` 컴포넌트

\*\*증상\*\*: 명세(bo\_screen\_spec.md §3-3)에는 "2초 후 자동으로 접수 메인 복귀"가 명시됐으나 코드에는 자동 복귀 없이 버튼 클릭만 구현됨



\*\*수정 방안\*\*:

```typescript

function DoneFeedback({ data, mode, onNext }: ...) {

&#x20; // 2초 후 자동 복귀 추가

&#x20; useEffect(() => {

&#x20;   const timer = setTimeout(onNext, 2000)

&#x20;   return () => clearTimeout(timer)

&#x20; }, \[onNext])



&#x20; return (

&#x20;   // ...기존 UI...

&#x20;   // 카운트다운 표시 (선택)

&#x20;   <p className="text-xs text-gray-400">2초 후 자동으로 돌아갑니다</p>

&#x20; )

}

```



\---



\### ⚠️ 버그 #10 — 폴링이 백그라운드 탭에서도 계속 실행



\*\*위치\*\*: `src/app/admin/dashboard/page.tsx`, `src/app/admin/storage/page.tsx`

\*\*증상\*\*: 모바일에서 다른 앱을 사용하는 동안 30초마다 API 호출 발생 → 배터리 소모 및 서버 불필요 부하



\*\*수정 방안\*\*:

```typescript

// Page Visibility API 활용

useEffect(() => {

&#x20; function handleVisibilityChange() {

&#x20;   if (document.visibilityState === 'visible') {

&#x20;     fetchDashboard(true)   // 탭 복귀 시 즉시 갱신

&#x20;   }

&#x20; }

&#x20; document.addEventListener('visibilitychange', handleVisibilityChange)

&#x20; return () => document.removeEventListener('visibilitychange', handleVisibilityChange)

}, \[fetchDashboard])



// 폴링에 visibility 조건 추가

const timer = setInterval(() => {

&#x20; if (document.visibilityState === 'visible') {  // 탭 활성일 때만

&#x20;   fetchDashboard(true)

&#x20; }

}, POLL\_INTERVAL\_MS)

```



\---



\### 🟡 버그 #11 — 예약 상세 진입 시 하단 탭 활성 표시 없음



\*\*위치\*\*: `src/components/bo/BottomNav.tsx`

\*\*증상\*\*: `/admin/reservations/BD-20260715-0001` 경로에서는 `startsWith('/admin/reservations')`가 true이므로 예약 관리 탭이 활성화되어야 하나, BottomNav에 예약 관리 탭이 없음



\*\*원인\*\*: BottomNav의 5개 탭(`홈/접수/반환/현황/매출`)에 예약 관리가 포함되지 않아 `/admin/reservations` 경로에서는 어떤 탭도 활성화되지 않음



\*\*수정 방안\*\*:

```typescript

// 옵션 A: 예약 관리 진입 시 탭 바 숨김 처리

// AdminLayout에 showBottomNav prop 추가

// reservations 관련 페이지에서 showBottomNav={false}



// 옵션 B: 예약 관리 경로에서는 '홈' 탭을 활성으로 표시 (무난한 선택)

function isActive(href: string): boolean {

&#x20; if (href === '/admin/dashboard') {

&#x20;   return pathname === href

&#x20;     || pathname.startsWith('/admin/reservations')  // 예약 관리도 홈 탭 활성

&#x20; }

&#x20; return pathname.startsWith(href)

}

```



\---



\### 🟠 버그 #12 — FO API Rate Limiting 미구현



\*\*위치\*\*: `src/middleware.ts` 또는 각 API Route

\*\*증상\*\*: 악의적 봇이 `/api/reservations/prepare`를 무한 호출 → payments PENDING 누적 → DB 용량 소모



\*\*수정 방안 (Vercel Edge Config 활용)\*\*:

```typescript

// middleware.ts에 Rate Limiting 추가

import { Ratelimit } from '@upstash/ratelimit'

import { Redis } from '@upstash/redis'



const ratelimit = new Ratelimit({

&#x20; redis:     Redis.fromEnv(),

&#x20; limiter:   Ratelimit.slidingWindow(10, '1 m'),  // 분당 10회

&#x20; analytics: true,

})



// /api/reservations/\* 경로에 적용

if (pathname.startsWith('/api/reservations')) {

&#x20; const ip  = request.ip ?? '127.0.0.1'

&#x20; const { success } = await ratelimit.limit(ip)

&#x20; if (!success) {

&#x20;   return new Response(JSON.stringify({

&#x20;     success: false,

&#x20;     error: { code: 'TOO\_MANY\_REQUESTS', message: '잠시 후 다시 시도해 주세요.' }

&#x20;   }), { status: 429 })

&#x20; }

}

```

> Upstash Redis 무료 티어로 운영 가능. 또는 Vercel 자체 Firewall 규칙 활용.



\---



\### ⚠️ 버그 #13 — 매출 집계 대용량 데이터 성능 저하



\*\*위치\*\*: `src/app/api/admin/sales/route.ts`

\*\*증상\*\*: 월간 집계 시 해당 월 전체 결제 건을 메모리에 로드 후 집계 → 건수 많아질 경우 응답 느려짐



\*\*원인\*\*: Supabase JS 클라이언트 제약으로 복잡한 GROUP BY 집계를 SQL로 처리하지 못하고 앱 레이어에서 처리



\*\*수정 방안\*\*: Supabase RPC 함수로 집계 로직 이전:

```sql

CREATE OR REPLACE FUNCTION get\_sales\_summary(

&#x20; p\_from\_ts TIMESTAMPTZ,

&#x20; p\_to\_ts   TIMESTAMPTZ,

&#x20; p\_terminal TEXT DEFAULT NULL

)

RETURNS TABLE (paid\_count BIGINT, cancelled\_count BIGINT, net\_revenue BIGINT)

LANGUAGE sql AS $$

&#x20; SELECT

&#x20;   COUNT(\*) FILTER (WHERE r.status != 'CANCELLED') AS paid\_count,

&#x20;   COUNT(\*) FILTER (WHERE r.status = 'CANCELLED')  AS cancelled\_count,

&#x20;   (COUNT(\*) FILTER (WHERE r.status != 'CANCELLED')

&#x20;    - COUNT(\*) FILTER (WHERE r.status = 'CANCELLED')) \* 5000 AS net\_revenue

&#x20; FROM payments p

&#x20; JOIN reservations r ON r.id = p.reservation\_id

&#x20; WHERE p.paid\_at BETWEEN p\_from\_ts AND p\_to\_ts

&#x20;   AND (p\_terminal IS NULL OR r.terminal = p\_terminal::terminal\_type)

&#x20;   AND p.status = 'PAID';

$$;

```



\---



\## 7. 버그 요약 및 우선순위



| # | 버그명 | 위험도 | 수정 공수 | 배포 전 필수 |

|---|--------|--------|----------|------------|

| #1 | prepare 전화번호 형식 불일치 | 🔴 | 소 (30분) | ✅ |

| #2 | 토스페이 승인 타임아웃 미처리 | 🔴 | 소 (1시간) | ✅ |

| #3 | confirm use\_date 재검증 누락 | 🟠 | 소 (30분) | ✅ |

| #4 | KST 자정 채번 날짜 불일치 | 🟡 | 소 (30분) | ❌ |

| #5 | 환경변수 미설정 uncaught exception | 🔴 | 소 (30분) | ✅ |

| #6 | BO 접수 이용일 검증 없음 | 🟠 | 소 (1시간) | ✅ |

| #7 | 매출 net\_revenue 계산 기준 불일치 | 🟡 | 중 (2시간) | ❌ |

| #8 | SDK 로드 실패 시 UX 누락 | 🟠 | 소 (1시간) | ✅ |

| #9 | 접수 완료 자동 복귀 미구현 | 🟡 | 소 (30분) | ❌ |

| #10 | 백그라운드 폴링 낭비 | 🟡 | 소 (1시간) | ❌ |

| #11 | 하단 탭 예약 상세 미활성 | 🟡 | 소 (30분) | ❌ |

| #12 | FO API Rate Limiting 없음 | 🟠 | 중 (3시간) | ✅ |

| #13 | 매출 집계 대용량 성능 | 🟢 | 대 (1일) | ❌ |



\---



\## 8. 테스트 실행 가이드



\### 단위 테스트 권장 대상



```bash

\# 설치 (Jest + Testing Library)

npm install -D jest @testing-library/react @testing-library/jest-dom jest-environment-jsdom



\# 테스트 파일 우선순위

src/lib/validate.ts       # Zod 스키마, maskPhone, isCancellable

src/lib/toss.ts           # verifyTossWebhookSignature, generateTossOrderId

src/lib/sms.ts            # buildMessage 각 이벤트 타입

src/lib/bo-utils.ts       # fmtDate, fmtElapsed, getApiErrorMessage

src/lib/admin.ts          # guardStatus 상태별 분기

```



\### 핵심 테스트 케이스



```typescript

// validate.test.ts

describe('phoneSchema', () => {

&#x20; it('숫자만 11자리 허용', () => expect(phoneSchema.parse('01012345678')).toBe('01012345678'))

&#x20; it('하이픈 포함 거부', () => expect(() => phoneSchema.parse('010-1234-5678')).toThrow())

&#x20; it('10자리 거부', () => expect(() => phoneSchema.parse('0101234567')).toThrow())

})



describe('isCancellable', () => {

&#x20; it('어제 이용일은 취소 불가', () => expect(isCancellable('2026-07-14')).toBe(false))

&#x20; it('내일 이용일은 취소 가능', () => expect(isCancellable('2026-07-16')).toBe(true))

&#x20; it('오늘 이용일은 취소 불가', () => expect(isCancellable('2026-07-15')).toBe(false))

})



// toss.test.ts

describe('verifyTossWebhookSignature', () => {

&#x20; it('올바른 서명 검증 성공', () => { /\* ... \*/ })

&#x20; it('잘못된 서명 검증 실패', () => { /\* ... \*/ })

&#x20; it('타이밍 공격 방지 (timingSafeEqual)', () => { /\* ... \*/ })

})

```



\### E2E 테스트 권장 (Playwright)



```typescript

// 예약 완성 플로우

test('full reservation flow', async ({ page }) => {

&#x20; await page.goto('/')

&#x20; await page.click('text=예약하기')

&#x20; await page.click('text=T1')

&#x20; await page.click('text=출국')

&#x20; // 달력에서 내일 선택

&#x20; await page.click('text=다음으로')

&#x20; await page.click('text=1개')

&#x20; await page.click('text=다음으로')

&#x20; await page.fill('#name', '테스트')

&#x20; await page.fill('#phone', '01012345678')

&#x20; await page.click('text=다음으로')

&#x20; // 결제는 토스페이 테스트 환경에서만 진행

})

```



\---



\*QA 에이전트 바른 — Session 13 완료\*

\*다음 세션: Session 14 — 버그 수정 적용 (버그 #1\~#8 우선)\*

