# 데이터 모델 및 저장 정책

## 1. 관계

- users 1:N assets
- users 1:N albums
- albums N:M assets via album_items
- users 1:N upload_sessions, export_sessions, sessions
- assets 1:1 asset_metadata
- purge_jobs는 영구 삭제를 재시도할 수 있게 관리
- 사진·동영상 바이너리는 SQLite에 넣지 않음

## 2. users

- id UUID primary key
- username 정규화 후 unique
- password_hash Argon2id
- role admin/user
- status pending/active/suspended/archived
- quota_bytes 기본 300000000000
- must_change_password boolean
- created_at, approved_at, updated_at, archived_at

관리자 role은 공개 가입 API에서 생성할 수 없고 DB constraint 및 서비스 계층에서 보호합니다.

## 3. assets

- id UUID PK
- owner_id FK users
- original_name 표시용 원본 파일명
- object_key 서버 생성 상대 저장 키
- size_bytes 검증된 원본 크기
- sha256 서버 계산 hash
- media_type photo/video
- mime_type 서버 검증 유형
- status active/trashed/purging
- effective_taken_at 정규화된 타임라인 정렬 기준 시각 (ISO-8601 문자열, NOT NULL)
- uploaded_at, client_modified_at, trashed_at, purge_after, created_at

인덱스:
- CREATE INDEX idx_assets_owner_timeline ON assets(owner_id, status, effective_taken_at DESC, id DESC);
- CREATE INDEX idx_assets_owner_uploaded ON assets(owner_id, uploaded_at DESC);
- CREATE INDEX idx_assets_owner_status ON assets(owner_id, status, uploaded_at DESC);
- CREATE INDEX idx_assets_owner_hash ON assets(owner_id, sha256);
- CREATE INDEX idx_assets_owner_name ON assets(owner_id, original_name);

중요: 단일 타임라인 조회의 $O(\log N)$ 성능 보장을 위해 assets.effective_taken_at 정규화 컬럼과 복합 커버링 인덱스를 사용하며, asset_metadata의 세부 타임존 컬럼은 메타데이터 상세 표시에 활용합니다.

## 4. asset_metadata

- asset_id UUID unique FK
- taken_at_raw 원본 EXIF/컨테이너 날짜 문자열
- taken_at_local 시간대 없이 정규화한 로컬 wall-clock 값, nullable
- taken_at_offset_minutes source가 명시한 offset, nullable
- taken_at_utc offset이 신뢰 가능할 때 계산하는 UTC 값, nullable
- taken_at_timezone_known boolean
- taken_at_source exif_datetime_original/quicktime/file_fallback 등
- width, height, duration_seconds
- camera_make, camera_model
- gps_latitude, gps_longitude
- metadata_extracted_at NULL 허용 (비동기 추출 전 초기 pending 상태 지원)
- metadata_status pending/completed/failed
- metadata_error_code

시간대 없는 EXIF를 임의로 UTC 변환하지 않습니다. 원시 값과 offset 여부를 보존합니다.

인덱스:
- CREATE INDEX idx_meta_taken_local ON asset_metadata(taken_at_local, asset_id);
- CREATE INDEX idx_meta_taken_utc ON asset_metadata(taken_at_utc, asset_id);
- CREATE INDEX idx_meta_camera ON asset_metadata(camera_make, camera_model, asset_id);

소유자별 날짜 검색은 assets를 owner/status로 필터한 뒤 metadata와 JOIN합니다. 구현 후 EXPLAIN QUERY PLAN으로 실제 검색 쿼리를 점검합니다.

## 5. albums / album_items

albums: id UUID, owner_id FK, name, created_at, updated_at
album_items: album_id FK, asset_id FK, added_at, PK(album_id, asset_id)

album owner와 asset owner가 같아야 합니다. 앨범 삭제는 관계만 삭제하고 assets 및 원본은 유지합니다.

## 6. upload_sessions

- id UUID
- owner_id FK
- original_name
- expected_size
- received_bytes
- expected_sha256 NOT NULL
- temp_object_key
- status receiving/verifying/awaiting_duplicate_choice/completed/skipped/cancelled/failed
- created_at, updated_at, expires_at
- reservation_bytes
- completed_asset_id nullable
- idempotency_key NOT NULL, unique with owner_id

인덱스:
- CREATE INDEX idx_upload_sessions_owner_status ON upload_sessions(owner_id, status);

idempotency_key는 NOT NULL이며 owner_id와 복합 unique 제약을 둡니다. 같은 키에 다른 요청 본문이 오면 409를 반환합니다. status=skipped는 사용자가 중복 파일 건너뛰기를 선택한 terminal 상태이며 재시도 시 같은 결과를 반환합니다. expected_sha256는 세션 생성 시 필수입니다. 클라이언트는 스트리밍 방식으로 사전 해시를 계산하고 서버는 수신 파일을 독립적으로 계산합니다. 두 값이 다르면 자산 등록을 거부합니다. PWA 재개 시 재선택한 파일의 사전 해시가 세션 값과 일치하지 않으면 전송을 재개하지 않습니다. 사전 중복 감지는 소유자 범위에서만 수행하고, 완료 직전에도 검사해 동시 업로드 경합을 처리합니다. 완료 요청을 재시도할 때 completed_asset_id를 반환해 중복 등록을 막습니다.

파일 쓰기와 DB offset 변경은 원자적이지 않습니다. 세션별 OS 잠금으로 동시 writer를 직렬화하고, 파일 기록+fsync 후 조건부 DB offset 갱신/commit을 완료한 경우에만 청크 ACK를 보냅니다. 재시작 복구는 DB의 acknowledged offset을 기준으로 파일이 더 길면 truncate+fsync하고, 파일이 더 짧으면 안전 오류로 격리해 세션을 자동 완료하지 않습니다. 부분 청크를 성공으로 간주하거나 offset을 자동으로 앞당기지 않습니다.

## 7. export_sessions

- id UUID
- owner_id FK
- status queued/creating/ready/downloading/failed/cancelled/expired
- expected_size_bytes 및 reserved_bytes
- object_key nullable
- created_at, updated_at, expires_at
- error_code nullable

인덱스:
- CREATE INDEX idx_export_sessions_owner_status ON export_sessions(owner_id, status);

ZIP은 압축하지 않는 것(무압축 STORED)을 기본으로 합니다. 예상 파일 크기와 전체 free space를 확인하고 사용자당 active export는 1개, TTL은 24시간입니다. export가 참조하는 자산은 export 종료 전까지 purge하지 않습니다.

## 8. export_session_items

- export_id UUID FK export_sessions ON DELETE CASCADE
- asset_id UUID FK assets ON DELETE CASCADE
- archive_path TEXT NOT NULL (ZIP 내부 상대 경로, 대소문자 무관 중복 시 {stem}_{seq}_{asset_id[:6]}.{ext} 규칙 적용)
- size_bytes INTEGER NOT NULL
- PK(export_id, asset_id)
- UNIQUE(export_id, archive_path)

## 9. sessions

- id UUID
- user_id FK
- client_type android_native/web_pwa/admin_web
- session_secret_hash 또는 refresh_token_hash
- csrf_token_hash nullable
- created_at, last_used_at, expires_at, revoked_at
- token_family_id 및 rotation metadata(native)

Web/PWA 및 관리자 웹은 raw cookie secret을 저장하지 않고 hash만 보관합니다. Android는 refresh token hash와 rotation 상태를 보관합니다. 관리자 세션은 별도 type/host로 분리합니다.

## 10. audit_logs

- id INTEGER/UUID PK
- actor_user_id nullable
- action, target_type, target_id, source_ip, created_at, result, safe_details_json

비밀번호, token, GPS, 원본 경로는 기록하지 않습니다. 기본 보존 기간은 90일입니다.

## 11. purge_jobs

- id UUID
- asset_id UUID (intentionally not a foreign key, so the job retains its diagnostic asset UUID after the asset row is purged)
- status queued/running/failed/completed
- attempt_count, last_error_code
- created_at, updated_at, completed_at

휴지통 기간 만료 시 DB에 purging 상태와 job을 먼저 기록합니다. Worker는 원본·미리보기를 멱등적으로 지운 뒤 관계와 자산 row를 제거합니다. 중간 종료 후 queued/running 작업을 재실행합니다. 이미 파일이 삭제된 경우의 재시도도 안전해야 합니다.

## 12. 용량 계산

- 사용자 사진 quota 사용량 = active + trashed + purging 원본 크기 합계 + 미완료 upload reservation
- export reservation은 기존 사진을 내려받기 위한 임시 파일 공간이므로 사용자 사진 quota에는 포함하지 않고 전체 SSD 여유 공간에만 포함
- 사용자 기본 한도 300,000,000,000 byte
- 80% 경고
- 신규 업로드 예약은 트랜잭션/조건부 갱신으로 직렬화
- 휴지통 파일은 purge 완료 전까지 quota 포함
- store_separately duplicate는 물리 중복 크기도 반영
- 미리보기·DB·로그·임시파일은 사용자 quota와 별도지만 전체 SSD 공간 계산에는 포함
- 전역 신규 쓰기 차단 기준 = 전체 용량의 15%와 100GB 중 더 큰 여유 공간 기준
- 공간 정책: 여유 공간 20% 미만은 주의 알림, 차단 임계값 미만은 신규 업로드/export 차단, 여유 공간 10% 미만은 긴급 알림. 알림은 best-effort이며 호스트 장애 감지는 API와 독립된 감시 작업이 담당

## 13. SQLite 및 마이그레이션

SQLite WAL은 local filesystem에서 사용합니다. 실행 중인 DB 파일이나 WAL 파일을 임의 복사·삭제하지 않습니다. SQLite online backup API 또는 일관성을 보장하는 방법으로 백업하며, 스키마 변경은 버전 migration으로 관리합니다.
