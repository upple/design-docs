# 설계 결정 및 미결정 항목

## 1. 결정 기록

| ID | 결정 | 이유 |
|---|---|---|
| ADR-001 | Android는 Flutter 네이티브, iOS는 React PWA | Mac/Xcode 없이 iOS 클라이언트 배포 |
| ADR-002 | iOS PWA는 HTTPS에서 홈 화면 앱 형태로 사용 | IPA·서명·스토어 불필요, 브라우저 제약 수용 |
| ADR-003 | FastAPI 단일 서버 | 낮은 운영 복잡도 |
| ADR-004 | SQLite WAL, 로컬 SSD | 5명 이하 단일 서버 |
| ADR-005 | 원본은 파일시스템, 메타데이터는 SQLite | 바이너리·DB 분리 |
| ADR-006 | Caddy HTTPS | 인증서와 host별 라우팅 |
| ADR-007 | public photos host와 LAN-only admin host 분리 | 관리자 공개 방지 |
| ADR-008 | Android token, PWA cookie, admin cookie 분리 | 클라이언트별 인증 방식 일관성 |
| ADR-009 | 서버 수신 원본 바이트 보존 | 서버 자동 변환 방지. 클라이언트 선택기가 반환하는 리소스는 실기기 검증 필요 |
| ADR-010 | 백업과 삭제 동기화 분리 | 휴대폰/서버 원본 삭제 방지 |
| ADR-011 | 휴지통 30일, purging job | 삭제 중 장애 복구 |
| ADR-012 | 사용자별 SHA-256 중복 감지 및 명시적 선택 | 데이터 격리와 사용자 제어 |
| ADR-013 | 공유·자동 백업 MVP 제외 | 핵심 수동 백업 기능 우선 |
| ADR-014 | 사용자당 300GB, 80% 경고 | 전체 저장 공간 여유 |
| ADR-015 | DB 백업 7일 동일 SSD | 논리적 DB 복구. SSD 자체 장애 보호는 아님 |
| ADR-016 | SSD 암호화·외부 사본 미적용 | 운영 편의/무료 우선, 데이터 손실 위험 수용 |
| ADR-017 | ZIP export 24시간 TTL, 사용자당 1개 | iOS PWA의 다중 파일 전달 |

## 2. iOS PWA 영향

네이티브 iOS build/release는 하지 않으며 Flutter iOS target, Xcode, IPA, App Store Connect 및 Apple code signing은 범위에서 제거합니다. 사용자 웹앱은 Linux/Windows에서 Node.js로 빌드하고 public HTTPS host에서 서비스합니다.

iPhone 사용자는 Safari에서 홈 화면에 추가해 웹앱으로 엽니다. 네이티브 앱과 달리 백그라운드 업로드를 보장하지 않고, 페이지 재시작 후 파일 재선택이 필요할 수 있습니다. 단일 파일은 기본적으로 Files/다운로드에 저장하며, 작은 파일은 지원 여부를 확인한 뒤 공유 시트를 선택적으로 제공합니다. 공유 대상에 Photos 저장이 항상 표시된다고 보장하지 않습니다. 다중 파일은 ZIP → Files → 사용자가 Photos로 가져오기 흐름입니다.

공식 자료:
- Apple Safari 홈 화면 웹앱: https://support.apple.com/ko-kr/guide/iphone/iphea86e5236/27/ios/27
- Web Share API: https://developer.mozilla.org/en-US/docs/Web/API/Web_Share_API
- Flutter iOS native build requirements: https://docs.flutter.dev/platform-integration/ios/setup

## 3. 배포 기본값

- public host: photos.<domain>
- admin host: admin.<domain>
- admin LAN CIDR: 실제 내부 네트워크를 확인해 설정
- 내부 DNS override: admin host를 Raspberry Pi LAN IP로 지정
- 비밀번호 최소 12자
- Android access token 15분, refresh session 30일
- 웹 쿠키 host-only, Secure, HttpOnly, SameSite=Lax
- chunk 8MiB, HTTP body limit 약 12MiB
- 업로드 세션 TTL 48시간
- 모바일 데이터 경고 100MiB
- 감사 로그 보존 90일
- export TTL 24시간, 사용자당 동시에 1개
- 여유 공간이 max(총 용량 15%, 100GB)보다 작으면 업로드/export 차단
- 미리보기 긴 변 1600px, JPEG 품질 약 85부터 검증
- 한국어 우선

## 4. 수용 위험

- 단일 SSD: SSD 고장·분실 시 원본 복구 사본 없음
- 암호화 미적용: SSD 물리적 탈취 시 파일 노출 가능
- iOS PWA: Photos 저장과 업로드 재개가 브라우저 제약을 받음. 단일 파일 다운로드도 기본 경로는 Files/다운로드이며 공유 시트의 Photos 대상은 보장하지 않음
- 영구 삭제는 휴지통 30일 만료 뒤 시스템 purge worker만 수행하며 일반 사용자 API에는 즉시 purge 기능이 없음
- 외부 네트워크: 공인 IP가 있어도 라우터·DNS·ISP 영향으로 실접속 검증 필요
- 코덱: 일부 동영상은 스트리밍 불가 시 원본 다운로드 사용

## 5. 이번 검토에서 반영한 수정

- 잘못된 assets.taken_at 인덱스 제거, asset_metadata timestamp 인덱스로 변경
- Android token, PWA cookie, admin cookie 인증 분리
- 중복 업로드 skip/store_separately API와 멱등성 정의
- 시간대 없는 EXIF를 임의 UTC로 변환하지 않도록 데이터 모델 수정
- purging 상태와 purge_jobs로 영구 삭제 복구
- PWA의 Photos 저장/백그라운드 업로드/파일 재선택 제약 명시
- ZIP export, 공간 예약, 24시간 정리 정책 추가
- 전역 공간 기준을 max(15%, 100GB)로 명확화
- PWA service worker에서 개인 미디어/API 캐시 금지

## 6. 남은 검증

- 실제 DNS 및 LAN CIDR
- iPhone 실제 iOS 버전에서 Photos/Files/HEIC/MOV/공유 시트/ZIP 확인
- Android 최소 지원 OS와 갤러리 저장 기능
- 메타데이터 원본별 시간대 규칙
- ZIP 내보내기 취소·공간 부족 UX
- FFmpeg 패키지 및 라이선스
- 외부 사진 백업이 없는 위험 수용의 재확인
