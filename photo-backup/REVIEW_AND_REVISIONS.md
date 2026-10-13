# 설계 검토 결과 및 수정 기록

## 범위와 판정

요구사항, 아키텍처, API, DB, 보안·운영, 구현 계획을 상호 대조했습니다. 이번 개정은 문서 수준의 설계 수정입니다. 애플리케이션 코드를 구현하거나 런타임 검증한 것은 아닙니다.

## 발견사항 및 조치

### R1 — SQL 인덱스 오류 (수정)
이전 문서는 assets.taken_at 인덱스를 정의했지만 taken_at 컬럼은 asset_metadata에 있었습니다. 빈 DB에서 migration 실행 시 실패하는 문제입니다. assets에서 잘못된 인덱스를 제거하고 asset_metadata에 timestamp 인덱스를 정의했습니다. 실제 JOIN 쿼리는 구현 후 EXPLAIN QUERY PLAN으로 검증해야 합니다.

### R2 — 네이티브 토큰과 웹 쿠키의 불일치 (수정)
이전 인증 규칙은 모든 클라이언트에 token 응답을 암묵적으로 가정하면서 관리자 웹은 HttpOnly cookie를 요구했습니다. Android native bearer token, iOS PWA cookie+CSRF, 관리자 별도 host/cookie/session으로 분리했습니다.

### R3 — 관리자 웹 외부 노출 가능성 (구체화)
Public 사용자 PWA와 관리자 SPA를 호스트·정적 파일·세션 수준에서 분리했습니다. Caddy는 admin host에 설정된 실제 LAN CIDR만 허용하며 FastAPI도 관리자 role을 확인합니다.

### R4 — 중복 업로드 의사결정 API 누락 (수정)
중복을 발견하고 선택을 기다리는 흐름이 있었지만 선택 API가 없었습니다. awaiting_duplicate_choice 상태와 duplicate-choice API를 정의했고, skip/store_separately 선택 및 반복 요청 멱등성을 명시했습니다.

### R5 — EXIF 시간대 해석 (수정)
시간대 없는 EXIF 촬영 시각을 UTC로 추정 변환하지 않습니다. 원본 문자열, 로컬 시각, offset, UTC 값을 나눠 저장하며 UTC는 신뢰할 수 있는 offset이 있을 때만 계산합니다.

### R6 — 영구 삭제 중 crash (수정)
active/trashed만으로는 파일 삭제와 DB 변경 사이의 중단을 복구하기 어렵습니다. purging 상태와 purge_jobs를 추가해 재실행 가능한 처리 단계로 만들었습니다.

### R7 — Mac이 없는 개발 환경 (요구사항 변경)
iOS 네이티브 앱을 PWA로 바꿨습니다. React PWA는 Linux/Windows에서 빌드할 수 있고 Xcode·Mac·IPA 서명이 필요 없습니다. iPhone Safari의 홈 화면 앱 경험을 사용합니다.

PWA 한계도 요구사항에 반영했습니다:
- 지속적인 백그라운드 업로드 보장 없음
- 재실행 후 파일 재선택이 필요할 수 있음
- Photos 라이브러리에 직접 일괄 저장을 보장하지 않음
- 단일 파일의 기본 경로는 Files/다운로드이며 공유 시트는 지원될 때만 선택 제공. Photos 저장 대상은 보장하지 않음. 여러 파일은 ZIP → Files → 사용자 가져오기
- HEIC/MOV/대용량 파일 및 share sheet는 실기기 검증

### R8 — 원본 보존의 정의 (명확화)
서버는 수신 바이트를 자동 변환하지 않고 SHA-256을 검증합니다. 다만 iOS picker가 선택한 리소스가 원본 카메라 리소스와 언제나 바이트 단위로 동일하다고 가정하지 않습니다.

### R9 — 파일과 SQLite 트랜잭션 (보완)
파일시스템과 SQLite를 하나의 원자적 트랜잭션으로 묶을 수 없습니다. upload reconciliation, purge job, 고아 파일 유예 정리를 문서화했습니다. 구현 시 crash injection 테스트가 필요합니다.

### R10 — 전역 여유 공간 기준 (수정)
전역 쓰기 중지 기준을 max(전체 용량의 15%, 100GB)로 정의해 모호함을 없앴습니다. 2TB 장치에서는 약 300GB 여유 공간이 기준입니다.

### R11 — PWA 캐시 개인정보 (추가)
Service worker는 정적 셸만 캐시하며 원본·미리보기·동영상·ZIP·인증 응답은 캐시하지 않도록 했습니다.

### R12 — PWA 다중 다운로드 (추가)
iOS PWA에서 Photos 라이브러리에 직접 다중 파일 저장이 보장되지 않으므로 ZIP 내보내기와 임시 공간 예약, 사용자당 활성 작업 1개, 24시간 TTL을 정의했습니다.

## 남은 실증 검증

- 모든 migration을 빈 DB에서 실행
- Caddy host allowlist와 실제 LAN source IP 검증
- SQLite/파일시스템 crash injection
- iPhone Safari/PWA의 파일 선택, 재선택 재개, HEIC/MOV, share sheet, ZIP-to-Files
- Pi 5에서 대형 동영상 업로드의 CPU/RAM/SSD I/O 측정
- DB backup restore 및 orphan reconciliation

문서 수정 완료는 애플리케이션 구현이나 운영 환경 검증이 완료되었다는 뜻이 아닙니다.
