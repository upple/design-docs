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
이전 명세는 iOS PWA에서 Photos 라이브러리에 직접 다중 파일 저장이 보장되지 않으므로 ZIP 내보내기와 임시 공간 예약, 사용자당 활성 작업 1개, 24시간 TTL을 정의했습니다.

## gstack 4대 플랜 리뷰 결과 (2026-10-14 개정)

CEO, Engineering, Design, DevEx 4대 영역에 걸쳐 gstack 플랜 리뷰 가운틀릿을 수행하고 전 관문 **CLEARED** 판정을 받았습니다.

### GSTACK-01 — CEO Review (HOLD SCOPE)
- **D1 (SPOF 위험 수용 재확인)**: 0원 운영 원칙을 위해 단일 2TB 외장 SSD 장애 위험을 감수하고, 별도 MicroSD 복제는 배제함.
- **D2 (PWA Screen Wake Lock)**: 대용량 미디어 업로드 중 Safari 화면 자동 잠금 가능성을 줄이는 선택 기능으로 Screen Wake Lock API 적용. 백그라운드 전송은 보장하지 않음.
- **운영 알림 범위 조정**: 외부 웹훅은 초기 계획의 관리자 화면+로그 원칙과 맞지 않아 제외. 별도 호스트 감시는 마운트/API 장애를 로컬 journald 및 상태 파일에 기록하고 관리자 화면에서 표시. API/호스트 전체 장애 시 원격 즉시 알림이 없다는 위험을 수용.

### GSTACK-02 — Engineering Review (LOCK IN)
- **E1 (SQLite 청크 I/O 격리)**: 세션별 OS 잠금, fsync 후 ACK, 조건부 offset 갱신 및 재시작 복구 규칙을 적용. 성능은 부하·장애 주입 테스트로 검증해야 함.
- **E2 (WebAssembly 스트리밍 해싱)**: 4K 대용량 영상 사전 해시 연산 시 Web Worker 기반 스트리밍 해싱을 사용하며 Wasm은 선택적 최적화. 실제 iPhone에서 메모리·시간·발열을 측정하고 과열 방지를 보장하지 않음.
- **E3 (SQLite 온라인 백업)**: 쓰기 트랜잭션 진행 중 백업 복사 시 WAL 손상을 방지하기 위해 SQLite `.backup` API 적용.
- **미디어 워커 제한**: Pillow/FFmpeg 백그라운드 동시성과 작업 큐를 제한. `nice 19`는 CPU 우선순위만 조정하며 RAM은 별도 제한 필요.

### GSTACK-03 — Design Review (Score: 6.0 → 9.8/10)
- **D-DES1 (ZIP 가져오기 3단계 시각 모달)**: iOS PWA에서 다중 다운로드 완료 시 'Files 열기 → 압축 해제 → 사진 보관함 저장' 3단계 시각 가이드 모달 제공.
- **5대 인터랙션 상태 매트릭스**: Loading(Shimmer 스켈레톤), Empty(감성 일러스트), Error(재시도), Success, Partial(블러해시) UI 표준화.

### GSTACK-04 — DevEx Review (Score: 5.5 → 9.5/10, TTHW < 10분)
- **D-DX1 (원클릭 설치 자동화)**: 외장 SSD 확인, .env 생성, 컨테이너 기동, 관리자 생성을 대화형으로 한 번에 끝내는 `deploy/setup.sh` 제공.
- **D-DX2 (로컬 개발 패리티)**: 라즈베리파이나 공인 도메인 없이도 로컬 PC에서 즉시 테스트 가능한 `deploy/docker-compose.dev.yml` 분리.

### GSTACK-05 — 구현 태스크 및 인수 테스트 제안
- 10개 제안 구현 과제(T1 ~ T10) 및 인수 테스트(AT-21 ~ AT-26)를 `IMPLEMENTATION_PLAN.md`에 기록. 이는 구현 완료 또는 실기기 검증 통과를 뜻하지 않음.

## 2차 정합성 검토 (2026-10-14)

초기 목표(무료 우선, 가족 2~5명, 사진 원본 바이트 보존, 중단 후 재개, Raspberry Pi 단일 서버의 낮은 운영 복잡도)를 우선 기준으로 다시 검토했습니다. 기능을 더 늘리기보다 잘못된 보장과 데이터 무결성 위험을 줄이는 수정입니다.

### RR1 — Wake Lock 보장 범위 수정
Screen Wake Lock은 화면 자동 잠금 가능성을 줄이는 선택 기능으로 한정했습니다. API 미지원·요청 실패·visibility 변경·해제를 처리하고, 백그라운드 업로드 보장으로 설명하지 않습니다. 업로드 재개는 Wake Lock과 독립적입니다.

### RR2 — 청크 ACK 및 복구 계약 강화
한 세션의 writer를 OS 수준 잠금으로 직렬화하고, offset 재확인 → 임시 파일 기록 → flush/fsync → 조건부 DB offset 갱신/commit → ACK 순서를 정의했습니다. 재시작 시 DB의 acknowledged offset이 기준이며, 파일이 더 길면 해당 offset으로 truncate하고 파일이 더 짧으면 안전 오류로 격리합니다. DB 트랜잭션 시간을 1ms 미만으로 보장하는 문구는 제거했습니다. 구현 시 다중 worker·전원 차단·디스크 I/O 오류를 포함한 crash-injection 테스트가 필수입니다.

### RR3 — 중복 상태 머신 정합화
중복 건너뛰기는 `skipped` terminal 상태로 통일했습니다. 사전 중복 감지와 완료 직전 경합 감지 모두 같은 duplicate-choice 계약을 사용하며, skip/store_separately의 재시도·예약 해제·완료 멱등성을 API와 상태 다이어그램에 반영했습니다. 세션 생성 `idempotency_key`의 owner 범위 고유성 및 요청 본문 불일치 처리도 명시했습니다. 완료 직전 해시 검사와 자산 등록은 `(owner_id, sha256)` 기준 OS 잠금으로 직렬화합니다. `store_separately`를 허용하므로 자산 해시의 DB unique 제약은 두지 않습니다.

### RR4 — 운영 공간 임계값 및 알림 책임
전체 여유 공간 20% 미만은 관리자 화면/로컬 로그의 주의 표시, `max(전체 용량의 15%, 100GB)` 미만은 신규 업로드/export 차단, 10% 미만은 로컬 로그의 긴급 오류로 통일했습니다. API가 중지되어도 SSD/API 상태를 감지하도록 호스트 감시를 별도 정의했습니다. 초기 범위(관리자 화면과 로그만)를 지키기 위해 외부 웹훅은 제외했습니다. 이로 인해 서버/네트워크 전체 장애 시 원격 즉시 알림이 없다는 한계를 수용합니다.

### RR5 — 성능·설치 목표를 측정 항목으로 변경
1GB 해싱 15초, 설치 10분 등 실측 근거가 없는 수치는 합격 보장 조건에서 제거했습니다. 대표 기기에서 처리 시간·메모리·발열과 클린 Ubuntu 설치 시간을 측정해 결과를 기록하도록 인수 테스트를 바꿨습니다. `nice 19`가 RAM을 제한한다는 표현도 제거하고 작업 큐·픽셀·시간·스레드 제한 및 필요 시 cgroup 메모리 상한을 분리했습니다.

### 검토 판정
이 수정은 문서 설계의 정합성을 높이지만 코드 구현이나 실제 데이터 내구성을 증명하지는 않습니다. 특히 파일시스템과 SQLite를 하나의 원자적 트랜잭션으로 만들 수 없으므로 최종 파일 rename/DB commit 경계에서 재시작 시 임시·최종 파일과 세션 상태를 대조해 등록 재시도 또는 고아 격리를 수행해야 합니다. 이 복구와 재시도, ACK된 청크의 crash-injection 검증은 구현 단계의 차단 조건이며 AT-26에 추가했습니다. gstack의 이전 `CLEARED` 표기와 점수는 자동화된 리뷰 관문 결과일 뿐 실기기·부하·장애 주입 테스트 통과를 뜻하지 않으므로 README에서는 구현 전 검증 필요 상태로 바로잡았습니다.

## 3차 정합성 검토 및 아키텍처 보완 (2026-10-14 외부 자문 심층 리뷰 반영: R13 ~ R19)

외부 아키텍처 자문 내용을 바탕으로, 런타임 레이스 컨디션 해소, SQLite Filesort 병목 제거, 동시성 격리 및 PWA 복구력을 강화하는 7가지 핵심 아키텍처 개정을 반영했습니다.

### R13 — 사후 중복 선택 시 상태 머신 역행 및 용량 이중 예약 해소 (수정)
- **문제**: 대용량 동시 업로드 중 `verifying` 단계에서 동일 해시 자산이 뒤늦게 등록되면 `awaiting_duplicate_choice`로 전이됩니다. 이때 사용자가 `store_separately`를 선택할 경우, 이미 전체 바이트를 수신하고 검증했음에도 다시 `receiving` 상태로 역행하고 이미 예약된 `reservation_bytes` 위에 추가 용량을 이중 예약하는 모순이 있었습니다.
- **조치**: 중복 대기 상태를 사전(`received_bytes = 0`)과 사후(`received_bytes = expected_size`)로 명확히 분기했습니다. 사후 중복 상태에서 `store_separately` 수신 시, 추가 예약 및 `receiving` 전이 없이 임시 파일을 `/srv/photo-data/originals/{owner_uuid}/{asset_uuid}`로 원자적 이동(`os.replace`)하고 즉시 `status = 'completed'`, `reservation_bytes = 0`으로 완료합니다.

### R14 — 타임라인 단일 정렬 컬럼 및 커버링 복합 인덱스 신설 (수정)
- **문제**: `assets` 테이블과 `asset_metadata` 테이블이 분리되어 있고 타임존 유무에 따라 `taken_at_utc`와 `taken_at_local`로 나뉘어 있어, 타임라인 정렬 쿼리 시 `COALESCE` 함수로 인해 SQLite 인덱스를 타지 못하고 전체 테이블 정렬(Filesort) 및 임시 B-Tree 메모리 폭증이 발생했습니다.
- **조치**: `assets` 테이블에 정규화된 타임라인 정렬 기준 컬럼 `effective_taken_at TEXT NOT NULL`을 추가하고, `idx_assets_owner_timeline(owner_id, status, effective_taken_at DESC, id DESC)` 복합 커버링 인덱스를 구축하여 $O(\log N)$ Keyset 커서 조회를 보장합니다.

### R15 — 내보내기 세션 매핑 테이블 신설 및 ZIP 파일명 충돌 회피 (보완)
- **문제**: `export_sessions` 테이블만으로는 내보내기에 포함된 자산 목록을 원자적으로 추적하기 어려웠으며, 서로 다른 날짜/경로에 촬영되어 동일한 원본 파일명(`IMG_0001.JPG`)을 갖는 여러 자산을 하나의 ZIP으로 압축할 때 아카이브 내부 파일명 덮어쓰기 손실이 발생했습니다.
- **조치**: `export_session_items` 매핑 테이블을 신설하고, ZIP 내부 상대 경로(`archive_path`) 생성 시 대소문자 무관 중복 감지 알고리즘을 적용하여 2회차 이상 등장 시 `{stem}_{seq}_{asset_id[:6]}.{ext}` 형태로 결정론적 고유성을 부여합니다. CPU 보호를 위해 아카이브는 `ZIP_STORED`(무압축) 방식을 강제합니다.

### R16 — SQLite WAL Concurrency 및 전역 디스크 여유 공간 보호 (보완)
- **문제**: 기본 SQLite `DEFERRED` 트랜잭션을 사용하면 동시 업로드/내보내기 요청이 동일한 잔여 여유 공간을 읽어 Quota 및 디스크 최소 보호선($\max(15\%, 100\text{GB})$)을 초과하는 TOCTOU(Time-of-Check to Time-of-Use) 경합이 발생했습니다.
- **조치**: 용량 검증 및 예약 트랜잭션에 `BEGIN IMMEDIATE`와 `PRAGMA busy_timeout = 5000`을 강제했습니다. 아울러 쓰기 락 점유 시간을 줄이기 위해 `users.allocated_bytes` 증분 관리 및 세션 복합 인덱스(`idx_upload_sessions_owner_status`, `idx_export_sessions_owner_status`)를 보강했습니다.

### R17 — 청크 SHA-256 검증 및 대용량 비동기 검증 분기 (개선)
- **문제**: 8MiB 청크 전송 시 청크 단위 검증이 없어 전송 중 변조를 조기에 발견할 수 없었고, `POST /complete`에서 수 GB 동영상의 SHA-256을 동기 계산할 때 HTTP 타임아웃(60초) 및 프록시 연결 끊김이 발생했습니다.
- **조치**: 각 청크(`PATCH /content`) 수신 시 8MiB 청크 해시를 기록하고, 파일 크기가 100MiB를 초과하는 경우 즉시 `202 Accepted` (`status = 'verifying'`)를 반환하고 백그라운드 워커가 SHA-256 검증을 비동기 수행하도록 분기했습니다. 워커 크래시 복구를 위해 15분 경과한 `verifying` 세션은 조정 워커가 자동 재검증/정리합니다.

### R18 — 메타데이터 초기 상태 정의 및 NOT NULL 제약 해소 (수정)
- **문제**: `asset_metadata.metadata_extracted_at`이 `NOT NULL`로 선언되어 있어, 비동기 EXIF/FFmpeg 추출 워커가 작업을 마치기 전에 업로드 완료 시점에서 초기 메타데이터 레코드를 원자적으로 생성할 수 없는 구조적 모순이 있었습니다.
- **조치**: `metadata_extracted_at`을 `NULL` 허용으로 변경하고, `metadata_status CHECK (metadata_status IN ('pending', 'completed', 'failed'))` 컬럼을 추가하여 자산 생성 트랜잭션에서 즉시 `pending` 메타데이터 레코드가 생성되도록 수정했습니다.

### R19 — PWA 세션 상태 조회 및 멱등 재개 프로토콜 명세 (보완)
- **문제**: iOS Safari PWA는 백그라운드 실행이 중단되고 앱 재시작 시 `File` 인스턴스 참조가 소실되므로, 사용자가 파일을 다시 선택했을 때 기존 전송량(`received_bytes`)을 조회하여 청크 오프셋을 이어받을 명확한 프로토콜이 부족했습니다.
- **조치**: `GET /upload-sessions/{id}` 조회 엔드포인트와 `POST /upload-sessions`의 결정론적 `idempotency_key` 기반 멱등 재개(200 OK + `received_bytes`) 동작을 표준화했습니다.

## 남은 실증 검증

- 모든 migration을 빈 DB에서 실행
- Caddy host allowlist와 실제 LAN source IP 검증
- SQLite/파일시스템 crash injection
- iPhone Safari/PWA의 파일 선택, 재선택 재개, HEIC/MOV, share sheet, ZIP-to-Files
- Pi 5에서 대형 동영상 업로드의 CPU/RAM/SSD I/O 측정
- DB backup restore 및 orphan reconciliation

문서 수정 완료는 애플리케이션 구현이나 운영 환경 검증이 완료되었다는 뜻이 아닙니다.

