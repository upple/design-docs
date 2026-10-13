# Raspberry Pi Photo Backup — 설계 문서

가족·부부 등 최대 5명이 사용하는 개인용 사진·동영상 백업 서비스의 기준 설계입니다.

- 상태: 요구사항 확정 및 gstack 4대 플랜 리뷰(CEO, Eng, Design, DevEx) 전 관문 통과(CLEARED). 10개 구현 과제(T1~T10) 확정.
- 서버: Raspberry Pi 5, RAM 8GB, Ubuntu, 외장 SSD 2TB
- Android 클라이언트: Flutter 네이티브 앱
- iOS 클라이언트: React 기반 반응형 웹앱(PWA), Safari에서 홈 화면에 추가
- 관리자: 별도 호스트의 React 웹 페이지, 내부 네트워크 전용
- 서버: FastAPI, SQLite, Caddy
- 운영 원칙: 무료 우선, 외부 접속은 HTTPS, 사진 원본의 바이트 보존

## 문서 목록

- [REQUIREMENTS.md](REQUIREMENTS.md) — 기능·비기능 요구사항 및 플랫폼별 차이
- [ARCHITECTURE.md](ARCHITECTURE.md) — 시스템 구성, 데이터 흐름, 저장소 구조
- [API_SPEC.md](API_SPEC.md) — 인증·업로드·다운로드·중복 선택·내보내기 API
- [DATA_MODEL.md](DATA_MODEL.md) — DB 모델, 인덱스, 파일/DB 일관성
- [SECURITY_AND_OPERATIONS.md](SECURITY_AND_OPERATIONS.md) — 네트워크·인증·운영·복구 정책
- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) — 단계별 개발과 인수 테스트
- [DECISIONS_AND_OPEN_QUESTIONS.md](DECISIONS_AND_OPEN_QUESTIONS.md) — 결정, 위험 및 기본값
- [REVIEW_AND_REVISIONS.md](REVIEW_AND_REVISIONS.md) — 객관적 검토 결과와 반영 내역

## 플랫폼 결정

| 플랫폼 | 클라이언트 | 빌드·배포 |
|---|---|---|
| Android | Flutter 네이티브 앱 | 기존 PC/CI에서 Android APK 빌드 가능 |
| iOS | 반응형 웹앱(PWA) | Linux/Windows에서 웹 번들 빌드 후 HTTPS 배포. Xcode, Mac, IPA 서명 불필요 |
| 관리자 | React 웹 | Linux/Windows에서 정적 웹 빌드. 관리자 호스트는 LAN만 허용 |

iOS PWA는 네이티브 iOS 앱이 아닙니다. Safari에서 홈 화면에 추가해 앱처럼 열 수 있지만, Photos 라이브러리에 임의로 직접 저장하거나 백그라운드 전송을 보장할 수 없습니다. 단일 파일은 기본적으로 Files/다운로드에 저장합니다. 지원되는 작은 파일은 Web Share API를 사용할 수 있지만 Photos 저장 대상이 항상 제공되는 것은 아닙니다. 여러 파일은 ZIP으로 Files에 내려받은 후 사용자가 사진 앱으로 가져오는 흐름으로 설계합니다.

## 핵심 데이터 정책

1. 휴대폰에서 사진을 삭제해도 서버 백업은 유지합니다.
2. 서버에서 사진을 삭제해도 휴대폰 원본을 삭제하지 않습니다.
3. 서버에서 삭제한 파일은 30일간 휴지통에서 복구할 수 있습니다.
4. 원본 파일은 서버에 수신된 바이트 그대로 보존하고 자동 압축·변환하지 않습니다.
5. 사용자당 기본 할당량은 300,000,000,000바이트이며 사용량 80%부터 경고합니다.
6. 동일 파일 감지는 사용자별 SHA-256으로 수행합니다.
7. 일반 사용자는 본인 소유의 사진·앨범만 접근할 수 있습니다.
8. 관리자 웹과 API는 내부 네트워크에서만 접근할 수 있습니다.
9. 외장 SSD가 마운트되지 않았으면 파일 변경 작업을 차단합니다.
10. DB는 매일 백업하고 최근 7일분을 같은 SSD에 보관합니다. SSD 자체 고장에 대한 백업은 아닙니다.

## 기본 기술 스택

| 영역 | 선택 |
|---|---|
| Android | Flutter / Dart |
| iOS 사용자 웹앱 | React + TypeScript + Vite + PWA |
| 관리자 웹 | React + TypeScript + Vite |
| API | Python + FastAPI + Pydantic |
| 메타데이터 DB | SQLite, WAL 모드 |
| HTTPS / reverse proxy | Caddy |
| 이미지 처리 | Pillow 및 HEIC 지원 모듈 |
| 동영상 메타데이터·썸네일 | FFmpeg / ffprobe |
| 운영 | Docker Compose + systemd 기반 마운트·서비스 보호 |
| 저장 | Ubuntu 로컬 파일시스템의 외장 SSD |

## MVP 포함

가입 신청·관리자 승인, 로그인, 수동 업로드, 재개 가능한 청크 업로드, 중복 선택, 미리보기, 검색, 앨범, 동영상 스트리밍, 다운로드, 휴지통, 용량 제한, 관리자 감사 로그와 서버 상태 확인을 포함합니다.

자동 사진 백업, 사진 공유, 공개 공유 링크, 외부 클라우드 사본, 동영상 재인코딩, AI 기반 분류, 네이티브 iOS 앱은 MVP에서 제외합니다.

## 명시적 위험

원본 사진과 DB 백업이 하나의 SSD에 있으므로 SSD 고장·분실 시 사진 원본을 복구할 외부 사본이 없습니다. 이것은 다중 사본 백업이 아니라 중앙 저장소이며, 물리적 데이터 손실 위험을 수용해야 합니다.

## 참고 자료

- Flutter iOS tooling: https://docs.flutter.dev/platform-integration/ios/setup
- iPhone Safari에서 웹앱으로 열기: https://support.apple.com/ko-kr/guide/iphone/iphea86e5236/27/ios/27
- Web Share API: https://developer.mozilla.org/en-US/docs/Web/API/Web_Share_API
