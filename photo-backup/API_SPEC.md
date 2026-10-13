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

필드: original_name, expected_size, client_mime_type, client_modified_at(optional), expected_sha256(required).

클라이언트는 파일 전체를 메모리에 적재하지 않고 incremental SHA-256을 계산합니다. 브라우저 구현은 Web Worker와 incremental hashing을 사용하고 준비 진행률을 표시합니다. 서버는 기대 hash를 검증 전 입력으로만 취급합니다.

POST 처리 순서:
1. 계정이 active 상태인지 확인합니다.
2. expected_size, expected_sha256 형식을 검증합니다.
3. 해당 owner_id 범위에서 기존 자산 해시를 사전 조회합니다.
4. 동일 자산이 있으면 awaiting_duplicate_choice로 응답합니다. 이때 원본 바이트를 아직 업로드하지 않았으며 임시 파일 공간은 만들지 않습니다.
5. 중복이 아니면 사용자 한도와 전체 SSD 여유 공간을 확인하고 전체 크기를 reservation으로 잡은 뒤 receiving 세션을 생성합니다.

중복 선택 요청은 body {"choice":"skip"} 또는 {"choice":"store_separately"}를 사용합니다. skip은 세션을 완료 처리하고 예약을 해제합니다. store_separately는 사용자 한도와 전역 공간을 다시 확인한 후 reservation을 생성하고 receiving 상태로 전환합니다. 모든 중복 검사는 사용자 범위 안에서만 합니다.

### 청크 규칙

- 기본 청크: 8MiB
- Content-Type: application/offset+octet-stream
- Upload-Offset은 해당 청크의 시작 byte offset입니다.
- 서버 수신 offset과 정확히 일치할 때만 기록합니다.
- 불일치는 409 Conflict와 current offset 반환
- 요청 본문 제한은 개별 청크보다 조금 큰 값(예: 12MiB)으로 구성합니다. 전체 원본 크기 제한과 혼동하지 않습니다.
- 청크는 임시 파일에 스트리밍하고, 파일 쓰기 이후 DB received_bytes를 갱신합니다.
- 장애 시 임시 파일 길이와 DB offset을 대조해 완전하게 기록된 청크 경계로 복구합니다. 부분 청크를 완료된 것으로 간주하지 않습니다.
- TTL 기본값은 마지막 활동 후 48시간입니다.
- PWA 재개 시 선택 File 참조가 사라졌으면 같은 파일을 다시 선택해야 합니다. 재선택한 파일의 사전 SHA-256이 세션 값과 다르면 기존 세션으로 재개하지 않습니다.

### 완료 및 동시성

complete는 completed, awaiting_duplicate_choice, integrity_failed, storage_unavailable, quota_exceeded 중 하나를 반환합니다. 서버는 수신 파일의 실제 크기와 SHA-256을 독립 계산합니다. expected_size/hash 불일치면 자산 등록을 거부하고 임시 파일 및 용량 예약을 정리하거나 실패 상태로 안전하게 보존한 뒤 명시적 만료 정책에 따라 정리합니다.

사전 검사 통과 후 다른 업로드가 같은 해시로 먼저 등록된 경우 완료 시점에 중복 선택을 다시 요청합니다. 중복 선택과 complete는 멱등적으로 동작해야 하며 completed_asset_id로 같은 자산을 두 번 등록하지 않습니다. 클라이언트가 전송한 MIME 값은 신뢰하지 않고 서버에서 형식·시그니처를 확인합니다.

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
- 전역 여유 공간 임계값에 도달하면 시작·진행 중인 export를 안전하게 중단 가능

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
