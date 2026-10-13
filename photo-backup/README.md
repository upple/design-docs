# Raspberry Pi Photo Backup — 설계 문서

가족·부부 등 최대 5명이 사용하는 개인용 사진·동영상 백업 서비스의 기준 설계입니다.

- 상태: 요구사항 확정 및 설계 초안 완료. 애플리케이션 구현은 아직 시작하지 않았습니다.
- 서버: Raspberry Pi 5, RAM 8GB, Ubuntu, 외장 SSD 2TB
- 모바일: Flutter 기반 Android / iOS 앱
- 관리자: 내부 네트워크 전용 웹 페이지
- 서버 운영 원칙: 무료 운영 우선, 외부 접속은 HTTPS, 사진 원본의 바이트 보존

## 핵심 정책

1. 휴대폰 사진 앱에서 원본을 삭제해도 서버의 백업은 유지합니다.
2. 서버에서 사진을 삭제해도 휴대폰의 원본을 삭제하지 않습니다.
3. 서버에서 삭제한 파일은 30일간 휴지통에서 복구할 수 있습니다.
4. 파일 원본과 미리보기는 분리해 보관하며 원본을 자동 압축·변환하지 않습니다.
5. 사용자당 기본 할당량은 300GB이며 사용량 80%부터 경고합니다.
6. 중복 검사는 사용자별 SHA-256 해시를 사용합니다.
7. 일반 사용자는 본인 소유의 사진·앨범만 접근할 수 있습니다.
8. 관리자 웹 및 API는 같은 내부 네트워크에서만 접근할 수 있습니다.
9. 외장 SSD가 마운트되지 않았을 때는 파일 저장·삭제 작업을 중단합니다.
10. 데이터베이스 백업은 매일 수행하고 최근 7일분을 같은 SSD에 보관합니다. 이 백업은 SSD 자체 고장을 방지하지 못합니다.

## 문서 목록

- [REQUIREMENTS.md](REQUIREMENTS.md) — 확정된 기능·비기능 요구사항 및 기본값
- [ARCHITECTURE.md](ARCHITECTURE.md) — 시스템 구성, 데이터 흐름, 저장소 구조
- [API_SPEC.md](API_SPEC.md) — REST API와 이어 올리기 프로토콜
- [DATA_MODEL.md](DATA_MODEL.md) — DB 모델, 관계, 제약 조건 및 용량 계산
- [SECURITY_AND_OPERATIONS.md](SECURITY_AND_OPERATIONS.md) — 네트워크·인증·운영·복구 정책
- [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) — 구현 순서, 테스트 및 인수 조건
- [DECISIONS_AND_OPEN_QUESTIONS.md](DECISIONS_AND_OPEN_QUESTIONS.md) — 설계 결정, 명시적 위험, 미결정 항목

## 기본 기술 선택

| 영역 | 선택 |
|---|---|
| 모바일 | Flutter / Dart |
| 관리자 웹 | React + TypeScript + Vite |
| API | Python + FastAPI + Pydantic |
| 메타데이터 DB | SQLite, WAL 모드 |
| HTTPS 및 프록시 | Caddy |
| 미리보기 | Pillow 및 HEIC 지원 모듈 |
| 동영상 정보·썸네일 | FFmpeg / ffprobe |
| 배포 | Docker Compose + systemd 기반 마운트·서비스 보호 |
| 저장 | Ubuntu 로컬 파일시스템의 외장 SSD |

## MVP 범위

MVP는 가입 신청·승인, 로그인, 수동 업로드, 이어 올리기, 중복 경고, 미리보기, 검색, 앨범, 스트리밍, 일괄 다운로드, 휴지통, 용량 제한, 관리자 감사 로그와 상태 확인을 포함합니다.

자동 사진 백업, 사진 공유, 공개 공유 링크, 외부 클라우드 백업, 동영상 재인코딩 및 AI 기반 사진 분류는 범위에서 제외합니다.

## 저장소 및 데이터 손실 고지

현재 설계는 사진 원본과 DB 백업을 같은 SSD에 저장합니다. SSD 장애, 분실, 파일시스템 손상 또는 서버 전체 손실이 발생하면 사진 원본을 복구할 외부 사본이 없습니다. 별도 백업이 없는 위험은 운영자가 명시적으로 수용해야 합니다.
