# REST API 명세 초안

## 1. 공통 규칙

- 기본 prefix: /api/v1
- 인코딩: UTF-8 JSON. 바이너리 업로드·다운로드는 별도 Content-Type 사용
- 시간: ISO 8601 UTC
- 식별자: 서버 생성 UUID
- 페이지네이션: cursor 기반을 권장하며 최대 page size 기본값은 100
- 모든 자산·앨범·업로드 세션 요청은 인증 사용자와 owner_id를 대조합니다.
- 사용자 A가 사용자 B의 UUID를 알아도 조회·변경할 수 없어야 합니다.
- 성공 상태뿐 아니라 400, 401, 403/404, 409, 413, 422, 429, 507의 오류 계약을 정의합니다.
- 존재 자체를 노출하지 않아야 하는 타 사용자 자산은 404로 통일할 수 있습니다.

## 2. 인증·가입

| Method | Path | 설명 | 권한 |
|---|---|---|---|
| POST | /auth/register | 가입 신청 | 비로그인 |
| POST | /auth/login | 로그인 | 비로그인 |
| POST | /auth/refresh | 액세스 토큰 갱신 및 리프레시 토큰 회전 | 유효한 refresh token |
| POST | /auth/logout | 현재 세션 폐기 | 로그인 |
| POST | /auth/change-password | 비밀번호 변경 | 로그인 |
| GET | /me | 사용자 정보, 상태, 용량 반환 | 일반 사용자 |
| GET | /me/storage | 원본·휴지통·예약 중 사용량 반환 | 일반 사용자 |

가입 신청 시 서버는 계정 상태를 pending으로 만들고 세션 토큰을 발급하지 않습니다. 로그인 응답에는 짧은 수명의 access token과 갱신 가능한 refresh token을 제공합니다. 클라이언트 저장소에 토큰을 평문 파일로 기록하지 않습니다.

## 3. 업로드 세션

| Method | Path | 설명 |
|---|---|---|
| POST | /upload-sessions | 업로드 세션 생성 및 예상 용량 예약 |
| GET | /upload-sessions/{id} | 현재 상태와 서버 수신 오프셋 조회 |
| PATCH | /upload-sessions/{id}/content | 다음 청크 업로드 |
| POST | /upload-sessions/{id}/complete | 검증·최종 등록 요청 |
| DELETE | /upload-sessions/{id} | 세션 취소 및 임시 파일 정리 |

### 세션 생성 요청 필드

- original_name: 원본 파일명
- expected_size: 전체 파일 크기(byte)
- mime_type: 클라이언트가 추정한 MIME 타입. 서버 검증 전까지 신뢰하지 않음
- expected_sha256: 선택 필드. 최종 무결성 검증은 서버가 수행
- client_modified_at: 선택 필드. 파일시스템 수정 시간이며 촬영 시간과 별도로 저장

서버는 업로드 세션을 생성하기 전에 사용자 계정 상태, 사용자 한도, 전역 디스크 여유 공간을 확인합니다. 동시에 여러 업로드가 진행되어도 예약 바이트를 포함한 합계가 한도를 초과하지 않도록 DB 트랜잭션 또는 동기화 보호를 둡니다.

### 청크 프로토콜

- 기본 청크 크기: 8MiB
- PATCH 요청은 application/offset+octet-stream을 사용합니다.
- Upload-Offset 헤더는 청크가 기록될 시작 오프셋입니다.
- 서버는 전달받은 오프셋이 현재 수신 오프셋과 같을 때만 청크를 기록합니다.
- 오프셋 불일치는 409 Conflict로 응답하고 현재 오프셋을 반환합니다.
- 응답에는 Upload-Offset과 세션 상태를 포함합니다.
- 전체 파일을 메모리에 올리지 않고 임시 파일에 스트리밍 기록합니다.
- 세션 TTL 기본값은 마지막 활동 기준 48시간입니다. 연장 가능한 정책은 설정으로 관리합니다.
- 앱 재시작 시 세션 ID와 서버 오프셋을 조회한 후 이어 올립니다.

### 완료 처리

서버는 임시 파일 크기와 서버에서 계산한 SHA-256을 확인합니다. 형식 및 사용자 용량을 검증하고 성공한 경우 최종 경로로 이동한 뒤 DB에 자산을 등록합니다. 완료 요청은 재시도 가능해야 하며, 이미 완료한 세션에 반복 요청이 오면 같은 완료 결과를 반환합니다.

동일 사용자에게 동일 해시가 있는 경우 기본 결과는 duplicate_detected입니다. 사용자가 별도 자산 저장을 명시한 경우 새 asset UUID와 object key로 등록합니다. 다른 사용자의 해시 중복 여부는 결과에 포함하지 않습니다.

## 4. 자산·검색·다운로드

| Method | Path | 설명 |
|---|---|---|
| GET | /assets | 본인 자산 목록 및 필터 검색 |
| GET | /assets/{id} | 자산 메타데이터 |
| GET | /assets/{id}/preview | 미리보기 |
| GET | /assets/{id}/content | 원본 다운로드·동영상 스트리밍 |
| DELETE | /assets/{id} | 서버 라이브러리에서 휴지통 이동 |
| GET | /trash | 휴지통 목록 |
| POST | /trash/{id}/restore | 휴지통 자산 복원 |
| DELETE | /trash/{id}/purge | 자산 영구 삭제 |

검색 필터 초안: media_type, taken_from, taken_to, filename query, camera_make, camera_model, has_gps, 정렬 필드 및 cursor. GPS 기반의 자연어 지명 검색은 MVP에 포함하지 않습니다. 역지오코딩은 외부 서비스 전송에 따른 개인정보 영향을 검토한 후 별도 결정합니다.

원본 응답은 Content-Disposition을 안전하게 설정합니다. 동영상은 Range 요청을 처리하고, 206 Partial Content와 Content-Range를 올바르게 반환합니다. 미리보기와 원본은 동일한 인증 및 소유권 검사를 통과해야 합니다.

## 5. 앨범

| Method | Path | 설명 |
|---|---|---|
| GET | /albums | 본인 앨범 목록 |
| POST | /albums | 앨범 생성 |
| PATCH | /albums/{id} | 앨범 이름 변경 |
| DELETE | /albums/{id} | 앨범 및 항목 관계 삭제. 자산 원본은 유지 |
| POST | /albums/{id}/assets | 자산 여러 개 추가 |
| DELETE | /albums/{id}/assets/{assetId} | 해당 앨범에서 자산 제거 |

## 6. 관리자 API

모든 경로는 LAN 제한, 관리자 role, CSRF 보호(쿠키 세션을 쓸 경우), 감사 로그 대상입니다.

| Method | Path | 설명 |
|---|---|---|
| GET | /admin/registrations | 승인 대기 목록 |
| POST | /admin/users/{id}/approve | 가입 승인 |
| POST | /admin/users/{id}/reject | 가입 거절 |
| POST | /admin/users/{id}/suspend | 계정 정지 |
| POST | /admin/users/{id}/activate | 계정 재활성화 |
| POST | /admin/users/{id}/reset-password | 임시 비밀번호 발급·변경 강제 |
| GET | /admin/users | 사용자·용량 목록 |
| GET | /admin/audit-logs | 감사 로그 조회 |
| GET | /admin/system-status | API, DB, 마운트 및 시스템 상태 |
| GET | /admin/storage-status | 사용자 한도, 총 디스크 여유 공간 |

관리자 경로가 공개 진입점으로 요청되면 애플리케이션의 응답과 무관하게 Caddy에서 거부해야 합니다.

## 7. 오류 계약 초안

- 400: 잘못된 요청 형식
- 401: 인증 필요 또는 세션 만료
- 403: 권한 없음
- 404: 없는 자산 또는 접근할 수 없는 타 사용자 자산
- 409: 업로드 오프셋 불일치, 중복 선택 필요, 상태 충돌
- 413: 프록시 또는 API 설정상 요청 본문 제한 초과. 파일 크기별 제품 제한과는 구별
- 422: 파일 형식·메타데이터 검증 오류
- 429: 로그인 시도 제한
- 507: 사용자 한도 또는 전역 저장 공간 부족

오류 응답에는 내부 경로, 스택 트레이스, SQL 문, 다른 사용자 정보가 포함되어서는 안 됩니다.
