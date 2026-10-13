# 보안 및 운영 정책

## 1. 네트워크 보안 경계

공개 호스트 photos.<domain>은 PWA와 일반 사용자 API만 제공합니다. 관리자 호스트 admin.<domain>은 별도 관리자 웹과 admin API만 제공합니다. Caddy는 admin host에 대해 설치 시 설정한 실제 LAN CIDR만 허용하고, FastAPI도 admin role을 검증합니다. 공개 호스트에서 관리자 정적 파일을 제공하지 않습니다.

공유기에서 Raspberry Pi로 TCP 80/443만 전달합니다. SSH, DB, FastAPI 직접 포트 및 관리자 별도 포트는 외부 포워딩하지 않습니다. 내부 DNS override로 관리자 호스트가 Pi의 LAN IP로 해석되도록 설정할 수 있습니다. 인증서 발급과 외부·내부 접근 제한은 실제 배포에서 검증합니다.

## 2. 클라이언트별 인증

### iOS PWA
- host-only cookie, Secure, HttpOnly, SameSite=Lax
- 쿠키에 Domain 속성을 설정해 다른 서브도메인으로 공유하지 않음
- 서버 DB에는 opaque cookie secret의 hash만 저장
- 상태 변경 요청에 CSRF token 및 Origin 검사
- token/secret을 localStorage 또는 IndexedDB에 저장하지 않음
- Service worker는 앱 정적 셸만 캐시
- 사진·동영상·미리보기·ZIP·인증 응답은 캐시하지 않고 private/no-store

### Android
- access token 기본 수명 15분
- refresh token은 Android 보안 저장소에 보관하고 서버는 token hash/rotation 상태 관리
- 로그아웃·비밀번호 재설정·계정 정지 시 refresh session 폐기

### 관리자 웹
- 별도 admin host-only cookie/session
- 공개 사용자 세션과 분리
- Secure/HttpOnly/SameSite, CSRF token 및 Origin 검사
- 관리자 작업에 감사 로그와 role 검사

## 3. 비밀번호와 로그인

- Argon2id와 고유 salt
- 비밀번호 최소 12자 권장
- IP와 사용자명 기준 rate limit. 초기값 예: 15분 실패 5회부터 점증적 지연
- pending/suspended 계정 상태가 응답으로 추측되지 않도록 함
- bootstrap secret은 Git에 저장하지 않음
- 최초 관리자 로그인에서 비밀번호 변경 강제
- 비밀번호·token·CSRF secret을 로그에 남기지 않음

## 4. 업로드 및 미디어 처리

- 파일명은 저장 경로에 사용하지 않고 server UUID 사용
- MIME 값을 신뢰하지 않고 파일 시그니처·실제 형식 확인
- 8MiB 청크 요청에 맞춰 개별 HTTP body 제한 설정. 전체 파일 크기 제한과 분리
- 이미지 디코더의 픽셀 수·처리 시간·동시성 및 작업 큐를 제한. nice 19는 CPU 우선순위만 낮추며 RAM 상한을 보장하지 않으므로 필요 시 cgroup/container memory limit을 별도 설정
- FFmpeg는 shell command 문자열 대신 인자 배열로 실행
- 임시 파일은 공개 자산 API로 노출하지 않음
- 원본·미리보기·검색·ZIP 모두 소유권 확인
- 인증 미디어의 캐시 우회 방지를 위해 private/no-store
- GPS, 원본 파일 경로, 비밀번호와 token은 로그 제외

## 5. iOS PWA 제약 및 개선

- HTTPS에서만 제공하며 iPhone Safari의 홈 화면 추가 기능을 활용
- 수동 업로드만 지원하며 iOS가 페이지를 종료/중단해도 전송이 계속된다고 보장하지 않음
- 지원되는 경우 업로드 중 Screen Wake Lock을 요청해 화면 자동 잠금 가능성을 줄임. 백그라운드 업로드·네트워크 지속성을 보장하지 않으며 업로드 재개는 이 기능과 독립적으로 작동
- 대용량 미디어 사전 해시는 전체 파일 메모리 적재 없이 스트리밍 SHA-256으로 계산. Wasm은 선택적 최적화이며 CPU/배터리 절감은 보장하지 않고 실기기 측정으로 평가
- 재실행 후 File 객체가 유지되지 않으면 사용자가 파일을 다시 선택해야 함
- 최종 SHA-256 불일치 시 자산 등록 거부
- 기본 다운로드는 Files/다운로드에 저장합니다. 작은 파일의 공유 시트는 navigator.canShare로 확인하되 Photos 저장 대상은 보장하지 않습니다.
- 다중 파일은 ZIP을 Files에 다운로드하며, 다운로드 완료 시 '사진 앱으로 가져오는 3단계 시각 가이드 모달'(Files 열기 → 압축 해제 → 사진 보관함 저장)을 노출하여 사용자 혼선을 방지
- 실제 iOS 버전에서 HEIC/MOV/대용량 파일/공유 시트/ZIP을 테스트

## 6. SSD 마운트와 공간 보호

- 외장 SSD UUID로 /srv/photo-data 마운트
- systemd 서비스 시작 전에 실제 mountpoint 검사
- 컨테이너 volume을 명시적으로 마운트하고 OS root 경로 fallback 금지
- 파일 생성·이동·삭제 전 mount 및 free-space 검사
- 장치 제거 또는 I/O 오류 시 쓰기와 purge를 중지하고 상태 경고 기록
- 사용자 한도 300GB, 80% 경고, 한도 초과 시 신규 업로드 차단
- 여유 공간 20% 미만은 주의 알림, max(전체 용량의 15%, 100GB) 미만은 신규 업로드/export 차단, 10% 미만은 긴급 알림
- export reservation도 전체 공간 계산에 포함
- SSD 암호화는 적용하지 않으므로 물리적 접근과 디렉터리 권한 관리 필요

## 7. 삭제 및 복구

- 휴지통은 30일
- 일반 사용자용 즉시 영구 삭제 API를 제공하지 않음. 시스템 정리 작업만 기간 만료 데이터를 purge
- 영구 삭제 전 purging 상태와 purge job 기록
- purge는 원본·미리보기·관계 삭제 후 최종 row 제거, 재실행 가능
- export가 참조 중인 자산은 export 종료 전까지 purge 보류
- DB row는 있으나 파일이 없으면 접근 차단 및 경고
- DB row 없이 파일이 있으면 고아 파일 후보를 기록하고 즉시 삭제하지 않음
- SQLite DB 매일 백업, 최근 7일 같은 SSD에 보관 (트랜잭션 일관성을 보장하기 위해 SQLite online backup API 사용)
- 월 1회 별도 위치 복원 테스트 권장
- 같은 SSD 백업은 SSD 하드웨어 고장 시 보호 수단이 아니며, 단일 SSD 위험은 운영 원칙(무료/단순성)에 따라 수용 (D1 결정)

## 8. 운영·로그 및 장애 알림

- 관리자 상태 화면: API/DB/마운트/디스크/DB 백업 및 실패 작업
- SSD 마운트 해제와 API 프로세스 장애는 API 자체가 중단되어도 감지할 수 있도록 systemd timer 등 별도 호스트 감시 작업이 journald와 로컬 상태 파일에 기록합니다. 관리자 상태 화면은 API가 정상일 때 해당 상태를 표시합니다. 외부 웹훅·SMS·푸시 알림은 MVP에 포함하지 않으며, 서버/네트워크 전체 장애 시 원격 즉시 통보가 없다는 한계를 수용합니다.
- 감사 로그 90일 보존
- 정기 Ubuntu 및 의존성 보안 업데이트, DB migration과 롤백 검토
