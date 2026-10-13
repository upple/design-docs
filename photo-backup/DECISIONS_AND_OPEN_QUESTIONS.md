# 설계 결정 및 미결정 항목

## 1. 기록된 주요 결정

| ID | 결정 | 근거 |
|---|---|---|
| ADR-001 | Flutter로 Android/iOS 앱을 구현 | 두 플랫폼 UI와 API 코드를 최대한 공유 |
| ADR-002 | FastAPI를 단일 API 서버로 사용 | 소규모 단일 서버에 적합한 개발·운영 구조 |
| ADR-003 | SQLite를 로컬 SSD에서 사용 | 최대 5명, 단일 서버, 낮은 운영 복잡도 |
| ADR-004 | 원본은 로컬 파일로 저장하고 DB에는 메타데이터만 저장 | 대용량 파일과 DB 백업을 분리 |
| ADR-005 | Caddy를 HTTPS 진입점으로 사용 | 인증서 발급·갱신 자동화 |
| ADR-006 | 백업과 삭제 동기화를 분리 | 휴대폰 삭제가 서버 백업을 지우지 않도록 보장 |
| ADR-007 | 휴지통은 30일 유지 | 실수 삭제 복구와 용량 회수의 균형 |
| ADR-008 | 중복 검사는 사용자별 SHA-256 | 정확한 동일 파일 감지, 사용자 간 데이터 존재 여부 보호 |
| ADR-009 | 공유 기능은 MVP 제외 | 계정·저장·복구 핵심 기능을 먼저 안정화 |
| ADR-010 | 사진 자동 백업은 MVP 제외 | 수동 업로드에서 무결성과 권한 정책 검증 후 확장 |
| ADR-011 | DB 백업은 같은 SSD에서 최근 7일 보관 | 논리적 DB 손상/잘못된 변경 복구에 도움. SSD 장애 보호는 아님 |
| ADR-012 | SSD 암호화와 외부 데이터 사본은 MVP에서 제외 | 서비스 재부팅 편의 및 무료 운영 우선 |
| ADR-013 | 사용자당 기본 한도는 300GB | 최대 5명, 합계 1.5TB로 2TB 장치의 여유 공간 확보 |
| ADR-014 | 계정 보관·정지는 사진을 기본 보존 | 계정 작업으로 사진을 실수 삭제하지 않도록 함 |
| ADR-015 | 관리자 웹은 내부 네트워크에서만 허용 | 관리자 자격 증명과 운영 기능의 외부 노출 방지 |

## 2. 초기 설정에서 확정할 값

다음 값은 시작 기본값이며 실제 LAN·디스크·네트워크 상태에 맞게 배포 시 설정해야 합니다.

- 실제 LAN CIDR: 예를 들어 192.168.1.0/24이지만 환경을 확인해 설정
- 도메인 또는 DDNS 호스트명
- 사용자 ID 규칙: 기본은 3~32자, 영문 소문자·숫자·점·밑줄·하이픈 허용 여부를 구현 전에 확정
- 비밀번호 최소 길이: 기본 12자 이상 권장
- access token: 15분
- refresh session: 30일, 회전 및 폐기 지원
- 청크 크기: 8MiB
- 업로드 세션 만료: 마지막 활동 후 48시간
- 대용량 모바일 데이터 경고: 100MiB 이상
- 감사 로그 보존: 90일
- 전체 디스크 안전 임계값: 여유 공간 15% 미만 또는 100GB 미만
- 미리보기 최대 긴 변: 1600px, JPEG 품질 약 85부터 시작. 원본 파일에는 적용하지 않음
- EXIF/GPS 검색: GPS 좌표 및 GPS 존재 여부는 저장 가능. 도시·주소로 자동 변환하는 역지오코딩은 MVP 범위 밖

## 3. 수용된 위험

### 원본 데이터의 단일 사본

사진 원본과 DB 사본이 동일 SSD에 저장됩니다. SSD 자체 고장, 분실, 파일시스템 손상 또는 장치 전체 손실을 복구할 외부 사본이 없습니다. 이 상태에서는 사진이 영구적으로 손실될 수 있습니다.

### 디스크 암호화 미적용

SSD가 물리적으로 분실되거나 제거되면 다른 환경에서 데이터를 읽을 위험이 있습니다. 운영 편의성 우선 결정을 따르되 파일 권한과 물리적 접근을 관리해야 합니다.

### 무료 iOS 직접 설치

Personal Team의 무료 서명·설치 방식에는 프로파일 만료와 재설치 부담이 있습니다. 실제 iOS 빌드와 기기 설치를 P5 단계에서 검증합니다.

### 인터넷 연결 안정성

공인 IPv4가 있어도 공유기 포워딩, ISP 정책, DNS, NAT loopback, 방화벽 상태에 따라 외부 접속이 실패할 수 있습니다. 배포 시 외부 모바일 네트워크에서 검증해야 합니다.

### 동영상 코덱

서버 재인코딩을 하지 않으므로 일부 포맷은 브라우저/앱 내 스트리밍이 되지 않을 수 있습니다. 원본 다운로드를 대체 경로로 제공합니다.

## 4. 남은 구현 전 결정

- 실제 사용자 ID 규칙 및 비밀번호 정책
- 관리자 LAN 허용 CIDR과 내부 DNS/NAT loopback 구성
- iOS 빌드용 Mac/Xcode 환경
- 사용자별 사진 GPS 정보 표시 수준 및 GPS 검색 UX
- UI 언어는 MVP에서 한국어 우선
- 접근성·다크 모드·다중 언어는 핵심 기능 검증 후 확장
- 모바일 갤러리 저장 플러그인의 Android/iOS 버전별 동작
- 동영상 미리보기 생성에 필요한 FFmpeg 패키지 및 라이선스 검토

## 5. 공식 문서 참조

- Flutter: https://docs.flutter.dev/
- FastAPI: https://fastapi.tiangolo.com/
- SQLite WAL: https://sqlite.org/wal.html
- SQLite Backup API: https://sqlite.org/backup.html
- Caddy Automatic HTTPS: https://caddyserver.com/docs/automatic-https
- OWASP Password Storage Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- Android WorkManager: https://developer.android.com/develop/background-work/background-tasks/persistent
- Apple Developer Program: https://developer.apple.com/programs/
