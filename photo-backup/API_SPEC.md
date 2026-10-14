# REST API 명세 초안

## 1. 공통 계약

- 공개 호스트의 prefix: /api/v1
- LAN-only 관리자 호스트의 prefix: /admin-api/v1
- JSON은 UTF-8, 식별자는 서버 생성 UUID
- 모든 자산·앨범·업로드 세션·ZIP export는 요청 사용자와 owner_id를 대조
- 타 사용자의 UUID가 제공되어도 리소스 존재 여부나 해시 중복을 노출하지 않음
- 청크 endpoint에는 예를 들어 12MiB의 개별 HTTP 요청 본문 제한을 둘 수 있음. 이는 전체 원본 파일 크기 제한과 다름
- 오류 응답에 내부 경로·SQL·스택 트레이스·토큰·타 사용자 정보가 포함되지 않음
- 완료 및 상태 변경 endpoint는 재시도에 안전하도록 설계

## 2. Android 인증

| Method | Path | 설명 |
|---|---|---|
| POST | /auth/native/register | 가입 신청 |
| POST | /auth/native/login | 승인 사용자 로그인, access/refresh 발급 |
| POST | /auth/native/refresh | refresh token 회전 |
| POST | /auth/native/logout | 해당 세션 폐기 |
| POST | /auth/native/change-password | 비밀번호 변경 |
| GET | /me | 사용자 정보·상태·용량 |

Android 앱은 Authorization Bearer로 access token을 보내고 refresh token은 Android 보안 저장소에 저장합니다. 가입 신청에는 인증 토큰을 발급하지 않습니다.

## 3. iOS PWA 인증

| Method | Path | 설명 |
|---|---|---|
| POST | /auth/web/register | 가입 신청 |
| POST | /auth/web/login | host-only Secure HttpOnly session cookie 발급 |
| GET | /auth/web/csrf | CSRF 토큰 발급/갱신 |
| POST | /auth/web/refresh | 세션 회전 또는 갱신 |
| POST | /auth/web/logout | 쿠키 세션 폐기 |
| POST | /auth/web/change-password | 비밀번호 변경 |

PWA는 session token을 localStorage/IndexedDB에 저장하지 않습니다. 인증은 cookie 기반이며 상태 변경 요청에는 CSRF token과 Origin 검사를 적용합니다. SameSite 속성도 적용합니다.

## 4. 관리자 인증/API

관리자 웹은 admin host의 별도 cookie/session을 사용하고 공개 사진 호스트의 세션과 공유하지 않습니다. 모든 관리자 API는 Caddy의 정확한 LAN CIDR allowlist와 서버 측 admin role 검사를 통과해야 합니다.

| Method | Path | 설명 |
|---|---|---|
| POST | /admin-api/v1/auth/login | 관리자 로그인 |
| GET | /admin-api/v1/auth/csrf | CSRF 토큰 |
| POST | /admin-api/v1/auth/logout | 세션 폐기 |
| GET | /admin-api/v1/me | 관리자 세션 확인 |
| GET | /admin-api/v1/registrations | 가입 대기 목록 |
| POST | /admin-api/v1/users/{id}/approve | 가입 승인 |
| POST | /admin-api/v1/users/{id}/reject | 가입 거절 |
| POST | /admin-api/v1/users/{id}/suspend | 계정 정지 |
| POST | /admin-api/v1/users/{id}/activate | 계정 활성화 |
| POST | /admin-api/v1/users/{id}/reset-password | 임시 비밀번호 발급·변경 강제 |
| GET | /admin-api/v1/users | 사용자·용량 목록 |
| GET | /admin-api/v1/audit-logs | 감사 로그 |
| GET | /admin-api/v1/system-status | API/DB/마운트/백업 상태 |
| GET | /admin-api/v1/storage-status | 공간 및 사용자 할당량 |

관리자 쿠키는 Secure/HttpOnly/SameSite이며 CSRF token과 Origin 검사를 적용합니다.

## 5. 업로드 세션

| Method | Path | 설명 |
|---|---|---|
| POST | /upload-sessions | 예상 크기·해시로 사전 중복 확인, 필요 시 용량 예약 |
| GET | /upload-sessions/{id} | 상태 및 offset 조회 |
| PATCH | /upload-sessions/{id}/content | 다음 청크 |
| POST | /upload-sessions/{id}/complete | 최종 크기·hash 검증 및 자산 등록 |
| POST | /upload-sessions/{id}/duplicate-choice | skip/store_separately 결정 |
| DELETE | /upload-sessions/{id} | 취소 및 임시 파일 정리 |

### 생성 요청

필드: original_name, expected_size, client_mime_type, client_modified_at(optional), expected_sha256(required), idempotency_key(required). idempotency_key는 owner 범위에서 unique하며, 같은 키·같은 요청은 기존 세션 결과를 반환하고 같은 키·다른 요청 본문은 409를 반환합니다.

클라이언트는 파일 전체를 메모리에 적재하지 않고 incremental SHA-256을 계산합니다. 브라우저는 Web Worker와 스트리밍 해시 구현(Wasm 사용 가능)을 이용하고 진행률을 표시합니다. 이는 메인 스레드 정체와 전체 파일 메모리 적재를 피하기 위한 것이며 CPU 발열·배터리 소모·처리 시간을 보장하지 않습니다. Screen Wake Lock은 지원되는 경우 화면 자동 잠금 가능성을 줄이는 선택적 편의 기능이며 백그라운드 전송이나 네트워크 연속성을 보장하지 않습니다. 서버는 기대 hash를 검증 전 입력으로만 취급합니다.

POST 처리 순서:
1. 계정이 active 상태인지 확인합니다.
2. expected_size, expected_sha256 형식을 검증합니다.
3. 해당 owner_id 범위에서 기존 자산 해시를 사전 조회합니다.
4. 동일 자산이 있으면 업로드 세션 row를 reservation/temp file 없이 awaiting_duplicate_choice 상태로 생성하고 세션 ID와 상태를 반환합니다. 이때 원본 바이트는 아직 업로드하지 않습니다.
5. 중복이 아니면 사용자 한도와 전체 SSD 여유 공간을 확인하고 전체 크기를 reservation으로 잡은 뒤 receiving 세션을 생성합니다.

중복 선택 요청은 body `{"choice":"skip"}` 또는 `{"choice":"store_separately"}`를 사용합니다.
- **사전 중복 (Case A, `received_bytes < expected_size`)**: `skip`은 세션 상태를 terminal 상태인 `skipped`로 바꾸고 멱등 응답합니다. `store_separately`는 사용자 한도와 전역 디스크 공간을 검증한 후 `reservation_bytes = expected_size`를 예약하고 `receiving` 상태로 전환합니다.
- **사후 중복 (Case B, `received_bytes == expected_size`, 완료 검증 중 경합)**: 이미 전체 바이트 수신과 용량 예약이 완료된 상태입니다. `skip`은 임시 파일을 삭제하고 예약을 해제하며 `skipped`로 전이합니다. `store_separately`는 추가 용량 예약이나 `receiving` 역행 없이 즉시 임시 파일을 `originals/`로 원자적 이동(`os.replace`)하고 자산 등록을 완료(`status = 'completed'`, `reservation_bytes = 0`)합니다.

### 청크 규칙

- 기본 청크: 8MiB
- Content-Type: application/offset+octet-stream
- Upload-Offset은 해당 청크의 시작 byte offset입니다.
- 서버 수신 offset과 정확히 일치할 때만 기록합니다.
- 불일치는 409 Conflict와 current offset 반환
- 요청 본문 제한은 개별 청크보다 조금 큰 값(예: 12MiB)으로 구성합니다. 전체 원본 크기 제한과 혼동하지 않습니다.
- 한 업로드 세션의 청크 쓰기는 세션별 OS 파일 잠금으로 직렬화합니다(단일 프로세스 메모리 잠금만으로는 다중 worker를 보호할 수 없음). 잠금 안에서 DB offset을 다시 확인하고 요청 offset이 정확히 일치할 때만 기록합니다.
- 청크 바이트를 임시 파일에 기록한 뒤 파일을 flush/fsync하고, 짧은 DB 트랜잭션에서 조건부 offset 갱신을 커밋한 후에만 클라이언트에 ACK합니다. SQLite는 foreign keys와 busy_timeout을 설정하고 `synchronous=FULL` 등 내구성 설정을 명시적으로 구성·시험합니다. 파일 I/O 동안 DB write transaction을 열어 두지 않습니다.
- 프로세스 시작/세션 복구 시 DB에 커밋된 received_bytes를 기준으로 임시 파일을 검사하고, 파일이 더 길면 마지막 승인 offset까지 truncate+fsync합니다. 파일이 더 짧으면 세션을 안전 오류로 표시해 자동으로 offset을 앞당기지 않습니다. 파일시스템과 DB는 하나의 원자적 트랜잭션이 아니므로 reconciliation이 고아 파일을 기록합니다.
- TTL 기본값은 마지막 활동 후 48시간입니다.
- PWA 재개 시 선택 File 참조가 사라졌으면 같은 파일을 다시 선택해야 합니다. 재선택한 파일의 사전 SHA-256이 세션 값과 다르면 기존 세션으로 재개하지 않습니다.

### 완료 및 동시성

complete는 파일 크기에 따라 동기 또는 비동기 응답을 반환합니다:
- **100MiB 이하**: 동기 검증을 수행하여 `200 OK` (`status: completed` 또는 사후 중복 시 `status: awaiting_duplicate_choice`)를 반환합니다.
- **100MiB 초과**: HTTP 타임아웃 방지를 위해 즉시 `202 Accepted` (`status: verifying`)를 반환하고, 백그라운드 워커가 스트리밍 SHA-256 검증을 수행합니다. 클라이언트는 `GET /upload-sessions/{id}` 폴링으로 최종 완료 여부를 확인합니다.

사전 검사 통과 후 다른 업로드가 같은 해시로 먼저 등록된 경우 complete는 세션을 verifying에서 awaiting_duplicate_choice로 전환하고 선택을 다시 요청합니다. 완료 직전 해시 중복 검사와 자산 등록은 `(owner_id, sha256)` 기준 OS 잠금으로 직렬화합니다. `store_separately`를 허용하므로 자산 테이블에 해시 unique 제약을 두지는 않습니다. 잠금 안에서 기존 자산을 재확인하고 중복이면 파일을 최종 경로로 이동하기 전에 선택 대기 상태로 commit합니다. 기존 임시 파일과 reservation은 선택이 끝날 때까지 유지하며 TTL로 제한합니다. skip은 skipped terminal 상태로 전환하고 reservation·임시 파일을 멱등적으로 해제합니다. 완료 직전 경합에서 store_separately를 선택하면 이미 잡힌 reservation을 중복 생성하지 않고 해당 reservation을 유지한 채 검증·등록을 재개합니다. 최초 세션 생성 때 중복이 발견된 경우에만 선택 후 새 reservation을 만듭니다. 같은 선택 요청 재시도는 같은 최종 결과를 반환하며 completed_asset_id로 자산을 중복 등록하지 않습니다. 클라이언트가 전송한 MIME 값은 신뢰하지 않고 서버에서 형식·시그니처를 확인합니다.

## 6. 자산·검색·휴지통

| Method | Path | 설명 |
|---|---|---|
| GET | /assets | 본인 자산 목록·검색 |
| GET | /assets/{id} | 메타데이터 |
| GET | /assets/{id}/preview | 미리보기 |
| GET | /assets/{id}/content | 원본 다운로드/동영상 스트리밍 |
| DELETE | /assets/{id} | 휴지통 이동 |
| GET | /trash | 휴지통 목록 |
| POST | /trash/{id}/restore | 복원 |

검색 필터: media_type, taken_from, taken_to, filename, camera_make, camera_model, has_gps, cursor. 시간대 미확정 EXIF 시각을 UTC로 오인하지 않도록 taken_at_local, taken_at_utc, timezone_known, source를 구분합니다.

영구 삭제 API는 일반 사용자에게 노출하지 않습니다. 서버 스케줄러가 30일 경과한 파일을 purge job으로 처리합니다. 인증 미디어에는 private/no-store 캐시를 적용합니다. 동영상은 HTTP Range를 지원하고 전체 파일을 메모리에 올리지 않습니다.

## 7. ZIP 내보내기

| Method | Path | 설명 |
|---|---|---|
| POST | /exports | 본인 자산 ID 목록으로 ZIP 작업 생성 |
| GET | /exports/{id} | 상태/진행률 |
| GET | /exports/{id}/content | 완료 ZIP 다운로드 |
| DELETE | /exports/{id} | 취소 및 임시 파일 제거 |

- 작업 생성 시 모든 asset owner를 검증. 타 사용자 ID가 섞이면 전체 요청 거부
- 예상 ZIP 크기(무압축)를 계산해 공간 예약
- 사용자당 활성 export 기본 1개, TTL 24시간
- ZIP은 SSD export-tmp에 저장하고 만료 시 정리
- iOS PWA에서는 ZIP을 Files에 다운로드한 후 사용자가 사진 앱으로 가져옴
- export에서 참조 중인 자산은 작업 종료 전까지 purge를 보류
- 전역 여유 공간이 쓰기 차단 임계값 아래로 내려가면 신규 export를 거부합니다. 이미 시작된 export는 예약된 공간을 포함해 시작 전에 확보한 여유 공간을 보호하며, 실제 공간 부족 또는 I/O 오류 시 안전하게 실패 처리하고 임시 파일·예약을 멱등적으로 정리합니다.

## 8. 앨범

| Method | Path | 설명 |
|---|---|---|
| GET | /albums | 본인 앨범 |
| POST | /albums | 생성 |
| PATCH | /albums/{id} | 이름 변경 |
| DELETE | /albums/{id} | 관계 삭제, 원본 유지 |
| POST | /albums/{id}/assets | 자산 추가 |
| DELETE | /albums/{id}/assets/{assetId} | 앨범에서 제거 |

앨범과 자산 owner가 같아야 합니다.

## 9. 오류 계약

- 400 잘못된 요청
- 401 인증 실패/만료
- 403 권한 거부
- 404 대상 없음 또는 타 사용자 리소스
- 409 offset 충돌, 중복 선택 대기, 잘못된 상태 전이
- 413 개별 청크 요청 크기 초과
- 422 파일 형식/메타데이터 검증 실패
- 429 로그인 제한
- 507 사용자 quota 또는 전역 저장 공간 부족
