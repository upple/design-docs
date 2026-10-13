# 구현 계획 및 인수 테스트

## P0 — 저장소 및 환경 골격

- 구조: apps/android, apps/ios-web, apps/admin-web, services/api, deploy
- FastAPI/SQLite migration, React PWA, React 관리자 웹, Flutter Android 앱
- 환경 변수와 비밀정보 분리
- 외장 SSD mount guard 및 readiness/health check
- Linux CI에서 API와 웹 빌드

완료 조건: SSD mount 확인 후에만 쓰기 기능이 활성화되고 iOS 클라이언트는 Mac/Xcode 없이 웹 번들 빌드 가능.

## P1 — 인증·권한

- CLI 기반 관리자 bootstrap
- 가입 신청 및 승인·거절
- Android bearer token 인증
- iOS PWA cookie session + CSRF
- 관리자 전용 host/cookie/session
- 소유권 검사와 감사 로그
- 로그인 rate limit, 비밀번호 재설정, 세션 폐기

완료 조건: pending 로그인 차단, 타 사용자 자산 접근 차단, 외부 admin host 차단, 세션 유형 혼용 방지.

## P2 — 파일 저장·업로드 재개

- 업로드 세션과 할당량 예약
- 8MiB 청크 및 offset 검증/복구
- SHA-256 무결성 검사
- 중복 감지와 duplicate-choice API
- 멱등 완료/취소
- 원본 다운로드 및 HTTP Range

완료 조건: Android와 PWA의 재개 동작을 각각 시험. PWA에서는 재선택 기반 재개가 가능하고 최종 hash 불일치 파일은 등록되지 않아야 함.

## P3 — 미리보기·EXIF·검색·앨범

- 원본 불변 미리보기 생성
- metadata extraction과 시간대 보존
- 날짜/파일명/카메라/GPS 여부 검색
- 앨범 CRUD
- 이미지 처리 픽셀 수/시간/동시성 제한

완료 조건: 시간대 없는 EXIF를 UTC로 오인하지 않고 미리보기 실패가 원본에 영향을 주지 않음.

## P4 — 휴지통·quota·ZIP 내보내기·운영

- 30일 휴지통, purging 상태와 purge jobs
- 사용자 quota 300GB, 80% 경고, 전역 SSD guard
- 일일 DB 백업 및 7일 보존
- 관리자 audit log와 상태 화면
- ZIP export, 공간 예약, 24시간 TTL, 정리
- purge/export 경합 처리

완료 조건: 삭제 중 프로세스 종료 후 purge 재개, ZIP export와 업로드 동시 실행 시 공간 보호, DB 복원 테스트 통과.

## P5 — 사용자 경험

- Flutter Android: 가입/로그인, 업로드, 탐색, 앨범, 스트리밍/다운로드, 갤러리 저장
- React PWA: Safari 파일 선택, 청크 업로드, 재개, 탐색·검색·앨범, 쿠키/CSRF
- iOS PWA 단일 파일 공유 시트/다운로드, 다중 ZIP to Files UX
- React 관리자 웹: 가입, 사용자, quota, audit, system status
- service worker는 정적 셸만 캐시
- Android/iPhone 실기기 테스트

완료 조건: 실제 Android와 iPhone Safari에서 파일 선택, HEIC/MOV, 대용량 업로드, 공유 시트, ZIP-to-Files 흐름 확인.

## P6 — 외부 공개와 안정화

- 도메인/DDNS와 Caddy HTTPS
- 공유기 80/443 전달
- public photos host와 LAN-only admin host 분리
- 실제 LAN CIDR 및 내부 DNS override
- 외부 모바일망 테스트
- SSD 제거·마운트 오류·디스크 부족·재부팅·I/O 오류 테스트
- DB 복원과 orphan reconciliation

완료 조건: 외부망에서는 일반 사용자 서비스만 사용 가능하며 관리자 host/API는 차단. SSD 미마운트 상태에서 OS root에 사진이 저장되지 않음.

## 인수 테스트

| ID | 시나리오 | 예상 결과 |
|---|---|---|
| AT-01 | pending 계정 로그인 | 거부 |
| AT-02 | 타 사용자 자산 조회/삭제 | 거부 또는 404 |
| AT-03 | 외부에서 admin host/API | 차단 |
| AT-04 | 업로드 전후 SHA-256 | 동일 |
| AT-05 | 청크 offset 충돌·프로세스 중단 | 일관된 경계에서 재개 또는 안전 실패 |
| AT-06 | duplicate skip/store_separately 재시도 | 멱등, 중복 자산 과생성 없음 |
| AT-07 | 시간대 없는 EXIF | 임의 UTC 변환 없음 |
| AT-08 | preview 작업 실패 | 원본 유지, 재시도 가능 |
| AT-09 | quota 80%/100% | 경고/신규 업로드 차단 |
| AT-10 | 전역 SSD 임계값 | 업로드와 export 차단 |
| AT-11 | purge 도중 서버 종료 | purge job 재개 |
| AT-12 | 휴지통 복원 | 유효 기간 내 복원 |
| AT-13 | iOS PWA 대용량 업로드 | 메모리 폭증 없이 청크 전송 |
| AT-14 | PWA 종료 후 재개 | 파일 재선택 후 전체 hash 일치 시에만 재개; hash 불일치 거부 |
| AT-15 | iOS 단일 파일 저장 | 다운로드·공유 시트 대체 경로 안내 |
| AT-16 | iOS 다중 내보내기 | ZIP 다운로드 및 Files에서 추출 |
| AT-17 | service worker 검사 | 개인 미디어/API 캐시 없음 |
| AT-18 | DB 백업 복원 | 복원본 SQLite 무결성 통과 |
| AT-19 | 빈 DB migration | 존재하지 않는 컬럼 인덱스 오류 없음 |
| AT-20 | 감사 및 서버 로그 | 비밀정보 없음 |
| AT-21 | PWA Screen Wake Lock | 대용량 업로드 중 화면 꺼짐 방지 및 UI 인디케이터 표시 |
| AT-22 | Wasm 스트리밍 해싱 | 1GB 비디오 사전 해시 연산 15초 이내 완료 및 브라우저 안정성 유지 |
| AT-23 | 치명적 장애 웹훅 알림 | SSD 언마운트 또는 디스크 90% 초과 시 Telegram/Discord 알림 즉시 수신 |
| AT-24 | 원클릭 setup.sh 실행 | 클린 Ubuntu 환경에서 설치부터 관리자 부트스트랩까지 10분 내 완료 |
| AT-25 | 로컬 dev 프로필 구동 | docker-compose.dev.yml로 실기기/공인 도메인 없이 즉시 로컬 테스트 가능 |

## gstack 플랜 리뷰 도출 구현 태스크 (Tasks: T1 ~ T10)

| ID | 우선순위 | 영역 | 작업 내용 | 대상 파일 | 검증 기준 |
|---|---|---|---|---|---|
| **T1** | P1 | `apps/ios-web` | Screen Wake Lock API 연동 및 화면 켜짐 유지 인디케이터 | `src/hooks/useUpload.ts`, `src/components/UploadIndicator.tsx` | AT-21 통과 |
| **T4** | P1 | `apps/ios-web` | WebAssembly 기반 가속 스트리밍 해싱(`hash-wasm`) 도입 | `src/workers/hashWorker.ts`, `package.json` | AT-22 통과 |
| **T5** | P1 | `services/api` | SQLite 커넥션 하드닝(`busy_timeout=5s`, FK=ON) 및 청크 I/O 격리 | `app/db/session.py`, `app/services/upload.py` | AT-05 다중 전송 부하 통과 |
| **T7** | P1 | `apps/ios-web` | iOS PWA ZIP 다운로드 후 사진 앱 가져오기 3단계 시각 가이드 모달 | `src/components/ExportGuideModal.tsx` | AT-16 가이드 UI 검증 |
| **T9** | P1 | `deploy` | 라즈베리파이 원클릭 대화형 설치 스크립트 (`setup.sh`) | `deploy/setup.sh`, `deploy/scripts/bootstrap-admin.sh` | AT-24 통과 |
| **T2** | P2 | `services/api` | 치명적 운영 장애(SSD 탈락, 디스크 90%, 백업 실패) 무료 웹훅 알림 | `app/services/alert.py`, `app/config.py` | AT-23 통과 |
| **T3** | P2 | `services/api` | Pillow/FFmpeg 백그라운드 썸네일 워커 동시성 1개 제한(`nice 19`) | `app/services/media.py` | 5개 동시 인제스천 CPU 안정성 |
| **T6** | P2 | `deploy` | SQLite 온라인 백업 API(`.backup`) 일일 백업 스크립트 적용 | `deploy/scripts/backup-db.sh` | AT-18 무결성 검증 |
| **T8** | P2 | `apps/ios-web` | 5대 인터랙션 상태 표준 컴포넌트(스켈레톤, 빈 화면, 에러 재시도) | `src/components/TimelineStates.tsx`, `EmptyState.tsx` | UI 매트릭스 렌더링 확인 |
| **T10** | P2 | `deploy` | 로컬 개발용 Docker Compose 프로필 및 가상 스토리지 환경 구성 | `deploy/docker-compose.dev.yml`, `app/config.py` | AT-25 통과 |

## 완료 정의

- API 단위/통합 테스트와 DB migration 통과
- Android 및 iPhone 실기기에서 주요 흐름 검증
- 웹 빌드가 Mac/Xcode 없이 재현 가능
- 네트워크 경계, 데이터 소유권, hash, 삭제 복구, SSD guard 검증
- T1~T10 구현 태스크 및 AT-01~AT-25 인수 테스트 통과
- 설치·업데이트·DB 복구 운영 문서 작성
