# 데이터 모델 및 저장 정책

## 1. 관계 개요

- users 1:N assets
- users 1:N albums
- albums N:M assets (album_items)
- users 1:N upload_sessions
- users 1:N sessions
- users 1:N audit_logs(행위자)
- assets 1:1 asset_metadata
- assets는 active 또는 trashed 상태를 갖고, 영구 삭제 시 관련 미리보기와 메타데이터를 함께 제거합니다.

## 2. 주요 테이블

### users

- id: UUID primary key
- username: 정규화된 유일 사용자 ID
- password_hash: Argon2id 해시
- role: admin 또는 user
- status: pending, active, suspended, archived
- quota_bytes: 기본 300000000000
- must_change_password: boolean
- created_at, approved_at, updated_at, archived_at

관리자 role은 일반 가입 요청으로 생성할 수 없습니다. username은 trim 및 정규화 후 unique 제약을 적용합니다.

### assets

- id: UUID primary key
- owner_id: users.id foreign key
- original_name: 업로드 당시 이름
- object_key: 서버 내부 상대 저장 키
- size_bytes: 검증된 원본 바이트 크기
- sha256: 원본 SHA-256 hex digest
- media_type: photo 또는 video
- mime_type: 서버가 검증한 MIME 유형
- status: active 또는 trashed
- uploaded_at, client_modified_at
- trashed_at, purge_after

인덱스: (owner_id, uploaded_at DESC), (owner_id, sha256), (owner_id, original_name), (owner_id, status, taken_at DESC). hash 인덱스는 사용자 범위로 조회해야 하며 다른 사용자의 데이터 존재 여부를 노출하는 용도로 사용하지 않습니다.

### asset_metadata

- asset_id: assets.id, unique foreign key
- taken_at: EXIF 또는 컨테이너 메타데이터에서 추출한 촬영 시각(UTC 정규화)
- width, height
- duration_seconds: 동영상인 경우
- camera_make, camera_model
- gps_latitude, gps_longitude
- metadata_extracted_at
- metadata_status 및 metadata_error_code(선택)

메타데이터 추출 실패는 원본 업로드 성공 자체를 실패시키지 않습니다. GPS 값은 민감할 수 있으므로 일반 로그에 포함하지 않습니다. 메타데이터 추출 시도는 원본을 변경하지 않습니다.

### albums

- id: UUID primary key
- owner_id: users.id foreign key
- name
- created_at, updated_at

### album_items

- album_id: albums.id
- asset_id: assets.id
- added_at
- primary key (album_id, asset_id)

앱 서비스 계층에서 앨범과 자산의 owner_id가 일치하는지 검증합니다. 트랜잭션 없이 서로 다른 소유자의 자산 관계를 생성할 수 없어야 합니다. 앨범에서 제거해도 원본 파일은 삭제하지 않습니다.

### upload_sessions

- id: UUID primary key
- owner_id: users.id foreign key
- original_name
- expected_size
- received_bytes
- expected_sha256: nullable
- temp_object_key
- status: receiving, verifying, duplicate_detected, completed, cancelled, failed
- created_at, updated_at, expires_at
- reservation_bytes

예약 용량은 미완료 세션의 예상 크기를 포함합니다. 각 청크 쓰기는 현재 오프셋을 DB의 received_bytes와 동기화하고 부분 기록 실패를 복구할 수 있어야 합니다. 만료 세션은 임시 파일을 정리하고 예약 용량을 해제합니다.

### sessions

- id: UUID primary key
- user_id: users.id
- refresh_token_hash
- created_at, last_used_at, expires_at
- revoked_at
- token_family_id 또는 rotation metadata(선택)

리프레시 토큰 원문은 DB에 저장하지 않습니다. 로그아웃, 비밀번호 재설정, 계정 정지 시 기존 세션을 폐기합니다.

### audit_logs

- id: integer 또는 UUID primary key
- actor_user_id: nullable users.id
- action
- target_type, target_id
- source_ip
- created_at
- result, safe_details_json

비밀번호, access/refresh token, 원본 파일 데이터, GPS 좌표는 감사 로그에 쓰지 않습니다. 기본 로그 보존 기간은 90일로 시작하고 설정 가능하게 구현합니다.

### app_settings

- key primary key
- value
- updated_at

할당량, 전체 디스크 안전 임계값, 업로드 세션 TTL, 대용량 모바일 데이터 경고 임계값 등을 관리합니다. 비밀정보는 이 테이블에 평문으로 저장하지 않습니다.

## 3. 용량 계산

- 사용자 사용량 = 해당 사용자 원본 자산(active + trashed)의 size_bytes 합계 + 진행 중 업로드 예약 바이트 합계
- 기본 quota_bytes = 300,000,000,000바이트
- 경고 임계점 = quota_bytes의 80%
- quota 초과 시 새 업로드 세션 생성 또는 크기 증가를 거부
- 휴지통 파일은 영구 삭제되기 전까지 사용자 한도에 포함
- 미리보기·DB·로그·임시 오버헤드는 사용자 원본 할당량에서 별도 계산하되 전체 디스크 안전 검사에는 모두 포함
- duplicate로 새 별도 자산을 만들 경우, 실제 중복 바이트도 사용자 한도에 포함

할당량 판정은 클라이언트 UI가 아니라 서버 트랜잭션에서 수행합니다. 동시 세션이 한도를 넘지 않도록 사용자별 용량 예약 갱신을 직렬화하거나 조건부 UPDATE를 사용합니다.

## 4. 파일 시스템과 DB 일관성

SQLite 트랜잭션만으로 파일시스템 이동을 원자적으로 묶을 수 없습니다. 따라서 업로드 완료 상태 전이를 명시하고, 임시 파일과 DB 행을 서로 점검하는 복구 루틴을 구현해야 합니다.

- DB에 active 자산이 있으나 원본 파일이 없으면 시스템 오류로 표기하고 접근을 막습니다.
- 최종 파일은 있지만 DB 자산이 없는 경우 고아 파일 후보로 기록하고 즉시 삭제하지 않습니다.
- 고아 파일 정리는 일정 유예 기간 후 관리자 확인 또는 안전 규칙을 거칩니다.
- 휴지통 복원은 원본 파일 존재와 DB 상태 확인 후 수행합니다.
- 영구 삭제는 DB 관계, 미리보기, 원본 순서를 고려해 재실행 가능하게 만듭니다.

## 5. 인덱스 및 마이그레이션

필요 인덱스는 owner_id와 검색·정렬 필드를 조합합니다. 원본 파일 저장 키와 사용자 입력 파일명을 혼동하지 않습니다. 스키마 변경은 버전 기반 migration으로 수행하며, 배포 전 DB 백업 및 롤백 절차를 확인합니다.

SQLite WAL 모드와 로컬 SSD 경로를 사용합니다. WAL 파일을 수동 삭제하거나 실행 중인 DB 파일을 임의 복사하지 않습니다. 백업은 SQLite online backup API 또는 일관성을 보장하는 절차를 사용합니다.
