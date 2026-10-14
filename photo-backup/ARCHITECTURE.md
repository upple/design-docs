# 시스템 아키텍처

## 1. 플랫폼 구성

- Android: Flutter 네이티브 앱
- iOS: React + TypeScript 기반 PWA. Safari에서 HTTPS 웹사이트를 홈 화면에 추가해 사용
- 관리자: React + TypeScript 기반 별도 웹앱
- API: FastAPI
- DB: SQLite WAL 모드
- HTTPS/라우팅: Caddy
- 파일: Raspberry Pi의 Ubuntu 로컬 외장 SSD

iOS는 네이티브 iOS 앱이 아니므로 Xcode·Mac·IPA 빌드가 필요하지 않습니다. 웹 클라이언트는 Windows/Linux에서도 Node.js로 빌드할 수 있습니다. iPhone에서 앱과 유사한 경험을 제공하되 OS 파일·사진 접근은 Safari 웹 API의 범위에 따릅니다.

## 2. 네트워크와 호스트 분리

예시 호스트명이며 실제 도메인은 배포 시 설정합니다.

- photos.example.net: PWA와 일반 사용자 API, 인터넷 공개 HTTPS
- admin.example.net: 관리자 웹과 관리자 API, 내부 LAN만 허용

외부 인터넷에서 admin host로 요청하면 Caddy에서 차단합니다. 내부에서는 내부 DNS override로 admin host를 Raspberry Pi의 LAN IP로 해석하도록 권장합니다. 실제 외부/내부 경계는 배포 중 검증합니다.

```mermaid
flowchart TD
    subgraph Clients["클라이언트 계층"]
        Android["Android 네이티브 앱<br/>(Flutter / Bearer Token)"]
        iOS["iOS Safari PWA<br/>(React / Cookie + Wasm Hash + Wake Lock)"]
        AdminWeb["관리자 웹 SPA<br/>(React / LAN 전용 Cookie)"]
    end

    subgraph ReverseProxy["리버스 프록시 계층 (Caddy)"]
        PhotosDomain["공개 호스트: photos.&lt;domain&gt;<br/>(공인 인터넷 HTTPS)"]
        AdminDomain["관리자 호스트: admin.&lt;domain&gt;<br/>(내부 LAN CIDR Allowlist)"]
    end

    subgraph ServiceLayer["애플리케이션 계층 (FastAPI Container)"]
        AuthMiddleware["인증/인가 미들웨어<br/>(소유권 검증 & CSRF 방어)"]
        UploadService["업로드 세션 관리자<br/>(8MiB 청크 / Wasm 해시 검증 / 할당량 예약)"]
        ExportService["내보내기 엔진<br/>(무압축 ZIP 스트리밍 & 24h TTL)"]
        MediaWorker["미디어 백그라운드 워커<br/>(동시성 1개 제한 / nice 19 / Pillow·FFmpeg)"]
        LocalHealthLog["로컬 상태 기록<br/>(journald + 상태 파일)"]
        Reconciliation["정리 및 복구 워커<br/>(고아 파일 정리 / 30일 휴지통 purge)"]
    end

    subgraph StorageLayer["데이터 및 스토리지 계층 (외장 2TB SSD)"]
        MountGuard["systemd Mount Guard<br/>(UUID 기반 /srv/photo-data 마운트 검증)"]
        SQLiteDB[("SQLite WAL 모드<br/>(app.sqlite3 & 일일 .backup)")]
        subgraph Filesystem["로컬 파일시스템 (/srv/photo-data/)"]
            Originals["originals/{owner_uuid}/{asset_uuid}<br/>(무손실 원본 바이트 보존)"]
            Previews["previews/{asset_uuid}.jpg<br/>(1600px 축소 미리보기)"]
            UploadTmp["upload-tmp/{session_uuid}.part<br/>(청크 업로드 임시 버퍼)"]
            ExportTmp["export-tmp/{export_uuid}.zip<br/>(내보내기 임시 파일)"]
        end
    end

    Android -->|HTTPS| PhotosDomain
    iOS -->|HTTPS| PhotosDomain
    AdminWeb -->|HTTPS| AdminDomain

    PhotosDomain --> AuthMiddleware
    AdminDomain --> AuthMiddleware

    AuthMiddleware --> UploadService
    AuthMiddleware --> ExportService

    UploadService -->|청크 I/O 분리 스트리밍| UploadTmp
    UploadService -->|fsync 후 조건부 offset commit| SQLiteDB
    UploadService -.->|완료 후 썸네일 트리거| MediaWorker

    MediaWorker --> Previews
    MediaWorker --> SQLiteDB

    ExportService --> ExportTmp
    Reconciliation --> SQLiteDB
    Reconciliation --> Filesystem

    MountGuard -.->|마운트 실패 시 서비스 차단| ServiceLayer
    StorageLayer -.->|I/O 오류 / 공간 임계값| LocalHealthLog
```

공개 호스트에는 관리자 정적 파일 번들이나 관리자 API를 제공하지 않습니다. 관리자 role 검사와 네트워크 접근 제한을 모두 적용합니다.

## 3. 기술 스택

| 계층 | 선택 | 원칙 |
|---|---|---|
| Android | Flutter/Dart | 네이티브 갤러리·다운로드 통합 |
| iOS 사용자 웹 | React + TypeScript + Vite + PWA | 반응형 파일 선택·업로드·탐색, PWA 정적 셸 캐시 |
| 관리자 웹 | React + TypeScript + Vite | 별도 빌드 및 관리자 호스트 |
| API | FastAPI + Pydantic | 소유권 검증과 파일 처리 |
| DB | SQLite WAL | 단일 호스트의 로컬 파일시스템 |
| Reverse proxy | Caddy | HTTPS 및 host별 라우팅 |
| 이미지 | Pillow + HEIC decoder | 미리보기 전용, 원본은 변경하지 않음 |
| 동영상 | FFmpeg/ffprobe | 메타데이터·대표 이미지, 재인코딩 없음 |
| 운영 | Docker Compose + systemd | 외장 SSD mount guard |

## 4. 저장 경로

    /srv/photo-data/
      originals/{owner_uuid}/{asset_uuid}
      previews/{asset_uuid}.jpg
      upload-tmp/{upload_session_uuid}.part
      export-tmp/{export_uuid}.zip
      db/app.sqlite3
      db-backups/app-YYYY-MM-DD.sqlite3

- 저장 경로는 서버 생성 UUID이며 파일명은 DB 표시용입니다.
- 원본, 미리보기, 업로드 임시 파일, ZIP 내보내기를 분리합니다.
- ZIP은 기본 24시간 TTL 후 정리합니다.
- 임시 저장 공간을 전체 SSD 여유 공간 계산에 포함합니다.
- SSD 미마운트 시 OS 루트로 저장하는 fallback은 금지합니다.

## 5. 웹/PWA 캐시 및 업로드 재개

- Service worker는 HTML/CSS/JS/아이콘 같은 버전이 있는 정적 리소스만 캐시합니다.
- API, 인증, 원본, 미리보기, 동영상, ZIP은 캐시하지 않고 private/no-store 정책을 적용합니다.
- PWA는 File.slice()로 파일을 읽고 전송하며 대용량 파일 전체를 메모리에 적재하지 않습니다. 세션 생성 전 Web Worker 기반 스트리밍 SHA-256을 계산해 재선택 파일 일치 여부를 확인하고 진행률을 표시합니다. Wasm은 선택 가능한 구현 최적화일 뿐이며 고정 처리 시간·발열·배터리 절감을 보장하지 않습니다.
- 업로드 중 지원되는 경우 Screen Wake Lock API(navigator.wakeLock.request('screen'))를 요청해 화면 자동 잠금 가능성을 줄이고 UI에 실제 활성 상태를 표시합니다. 요청 실패·visibility 변경·잠금 해제를 처리하며 백그라운드 전송 보장으로 설명하지 않습니다. 업로드 재개는 Wake Lock과 독립적입니다.
- 업로드 세션 ID와 acknowledged offset은 IndexedDB에 저장할 수 있습니다.
- 브라우저 종료 후 File 객체가 유지된다고 가정하지 않습니다. 파일 재선택을 요구할 수 있으며 최종 서버 SHA-256 불일치 시 자산 등록을 거부합니다.
- 백그라운드 지속 업로드를 보장하지 않습니다.

## 6. 업로드 흐름

1. 클라이언트가 파일을 전체 메모리에 적재하지 않고 읽어 expected SHA-256을 계산하며 진행률을 표시합니다.
2. 클라이언트가 original_name, expected_size, 필수 expected_sha256로 업로드 세션을 생성합니다.
3. 서버는 사용자 상태와 같은 사용자 소유의 기존 자산을 해시로 사전 조회합니다. 중복이면 파일 바이트 전송 전에 awaiting_duplicate_choice로 응답합니다.
4. skip이면 세션을 완료 처리하고 예약·임시 자원을 해제합니다. store_separately이면 할당량과 전체 여유 공간을 확인하고 receiving으로 전환합니다.
5. 새 파일이면 서버가 사용자 한도와 전체 SSD 여유 공간을 확인하고 예상 크기를 원자적으로 예약합니다.
6. 클라이언트가 8MiB 청크를 보내고, 서버가 예상 offset과 실제 offset이 같은지 검증합니다.
7. complete 단계에서 파일 크기가 100MiB 이하인 경우 동기 검증을 수행하고, 100MiB 초과인 경우 202 Accepted (status = 'verifying')를 반환하여 백그라운드 워커가 SHA-256 검증을 비동기 수행합니다.
8. 완료 시점에 동일 해시가 이미 등록된 경우(사후 중복), awaiting_duplicate_choice로 전이합니다. 이때 사용자가 store_separately를 선택하면(Case B) 추가 용량 예약이나 receiving 역행 없이 즉시 최종 경로로 이동 후 자산 등록을 완료합니다.
9. 승인된 파일을 UUID 기반 최종 경로로 원자적 이동(os.replace)하고 DB 자산(effective_taken_at 포함) 및 초기 메타데이터(pending)를 한 트랜잭션으로 등록합니다. 완료 응답과 completed_asset_id를 저장해 반복 요청을 멱등 처리합니다.
10. 제한된 백그라운드 작업(동시성 1개 제한, nice 19)으로 메타데이터 추출(EXIF/FFmpeg) 및 미리보기를 생성해 라즈베리파이 5 부하를 방어합니다. 메타데이터 완료 시 effective_taken_at을 최종 보정합니다.

### 6.1 청크 업로드 시퀀스 다이어그램

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant App as 클라이언트 (PWA / Flutter)
    participant Worker as Web Worker (Wasm)
    participant API as FastAPI 서버
    participant Storage as 외장 SSD (upload-tmp)
    participant DB as SQLite (WAL)

    User->>App: 사진/동영상 파일 선택
    activate App
    opt PWA에서 Screen Wake Lock 지원
        App->>App: 화면 잠금 방지 요청 (실패 가능)
    end
    opt PWA 클라이언트
        App->>Worker: File.slice() 전달
        activate Worker
        Worker-->>App: 스트리밍 SHA-256 진행률
        Worker-->>App: expected_sha256 완료
        deactivate Worker
    end
    opt Flutter Android 클라이언트
        App->>App: 플랫폼 해시 API로 스트리밍 SHA-256 계산
    end

    App->>API: POST /api/v1/upload-sessions (expected_size, expected_sha256)
    activate API
    API->>DB: 사용자 소유 자산 중복 해시 조회
    alt 동일 해시 자산 이미 존재
        API->>DB: reservation 없이 awaiting_duplicate_choice 세션 생성
        API-->>App: 200 OK (session_id, status: awaiting_duplicate_choice)
        App->>User: 중복 알림 및 건너뛰기/별도저장 선택 요청
        User->>App: 선택 (skip 또는 store_separately)
        App->>API: POST /upload-sessions/{id}/duplicate-choice
        alt skip
            API->>DB: 상태를 skipped로 기록 (멱등)
            API-->>App: 200 OK (status: skipped)
            break 업로드 건너뛰기
                App-->>User: 중복 파일 건너뜀
            end
        else store_separately
            API->>DB: quota·전역 공간 확인 후 reservation 생성
            API-->>App: 200 OK (status: receiving, offset: 0)
        end
    else 중복 아님
        API->>DB: 사용자 quota·전역 공간 검사 및 예약
        API-->>App: 201 Created (upload_session_id, received_bytes: 0)
    end
    deactivate API

    loop 모든 8MiB 청크 전송 완료까지
        App->>API: PATCH /upload-sessions/{id}/content (Offset, 8MiB 청크)
        activate API
        API->>Storage: 세션별 OS 잠금 하에 예상 offset에 청크 기록
        API->>Storage: flush + fsync로 청크 데이터 동기화
        Note over API,DB: 파일 I/O 중 DB write transaction을 유지하지 않음
        API->>DB: 조건부 UPDATE (received_bytes = 요청 offset) 및 commit
        API-->>App: 200 OK (received_bytes)
        deactivate API
    end

    App->>API: POST /upload-sessions/{id}/complete
    activate API
    API->>Storage: 수신 완료된 .part 파일의 실제 크기 및 SHA-256 독립 검증
    alt 무결성 검증 통과
        API->>Storage: 최종 경로로 원자적 rename (디렉터리 fsync 포함)
        API->>DB: INSERT assets/metadata 및 completed_asset_id/status를 한 트랜잭션으로 commit
        API-->>App: 200 OK (asset_id 등록 완료)
        API-)API: 백그라운드 워커에 미리보기 생성 큐 전달 (nice 19)
    else 해시 불일치 (Integrity Failure)
        API->>Storage: 손상된 .part 임시 파일 삭제
        API->>DB: UPDATE upload_sessions SET status = 'failed'
        API-->>App: 422 Unprocessable Entity (무결성 검증 실패)
    end
    deactivate API
    opt Wake Lock이 활성 상태
        App->>App: Screen Wake Lock 해제
    end
    deactivate App
```

### 6.2 업로드 세션 및 자산 상태 머신 전이도

```mermaid
stateDiagram-v2
    [*] --> receiving: POST /upload-sessions (신규 파일, reservation=size)
    [*] --> awaiting_duplicate_choice_pre: POST /upload-sessions (사전 동일 해시 감지, reservation=0)

    awaiting_duplicate_choice_pre --> skipped: choice = skip (정리 및 멱등 응답)
    awaiting_duplicate_choice_pre --> receiving: choice = store_separately (공간 재검증 후 reservation=size)
    skipped --> [*]

    receiving --> receiving: PATCH /content (8MiB 청크 순차 수신)
    receiving --> verifying: POST /complete (크기 > 100MiB 비동기 검증 전환, 202 Accepted)
    receiving --> completed: POST /complete (크기 <= 100MiB 동기 검증 통과 & 단독 자산)
    receiving --> awaiting_duplicate_choice_post: POST /complete (사후 동시 완료 중복 감지)
    receiving --> cancelled: DELETE /upload-sessions (사용자 취소)
    receiving --> failed: 해시 불일치 / TTL 만료 / I/O 오류

    verifying --> completed: 백그라운드 검증 성공 & 단독 자산 (원자적 CAS 전이)
    verifying --> awaiting_duplicate_choice_post: 백그라운드 검증 성공 & 사후 중복 감지 (reservation=size 유지)
    verifying --> failed: 백그라운드 해시 불일치 / 15분 타임아웃 만료

    awaiting_duplicate_choice_post --> skipped: choice = skip (임시파일 삭제 & reservation 해제)
    awaiting_duplicate_choice_post --> completed: choice = store_separately (추가 예약 없이 원자적 originals 이동 & 등록)

    state "자산(Asset) 생애주기" as AssetLifecycle {
        [*] --> active: 업로드 완료 등록 (effective_taken_at 설정)
        active --> trashed: DELETE /assets/{id} (휴지통 이동)
        trashed --> active: POST /trash/{id}/restore (복원)
        trashed --> purging: 30일 경과 (Purge Worker 시작)
        purging --> [*]: 원본/미리보기 파일 삭제 & DB 영구 제거 완료
    }
```

사전 해시는 대용량 파일의 시작 지연과 CPU/배터리 사용을 늘릴 수 있습니다. Web Worker는 메인 스레드 정체를 줄이지만 전체 기기 성능을 보장하지 않으므로 실제 iPhone에서 크기별 시간·메모리·발열을 측정합니다.

청크 쓰기는 세션별 OS 잠금으로 직렬화합니다. 파일 바이트를 기록하고 fsync한 뒤 DB offset을 조건부 갱신하고 commit한 경우에만 ACK합니다. DB commit 전 장애로 파일이 더 길면 재시작 시 DB의 승인 offset까지 truncate합니다. DB offset보다 파일이 짧으면 자동으로 offset을 앞당기지 않고 안전 오류로 격리합니다. 파일시스템과 DB는 원자적으로 묶이지 않으므로 최종 rename/DB commit 경계의 고아 파일을 reconciliation으로 탐지합니다. 용량 계산 및 세션 생성/완료 트랜잭션은 `BEGIN IMMEDIATE`와 `PRAGMA busy_timeout = 5000`으로 SQLite 쓰기 락 경합을 방어하며, 트랜잭션 시간에 1ms 같은 비현실적인 보장값을 두지 않습니다.

## 7. 원본 다운로드와 ZIP 내보내기

- 단일 사진·동영상은 소유권 확인 후 streaming response로 전달합니다.
- 동영상은 HTTP Range 지원을 사용해 부분 요청을 처리합니다.
- Android는 순차 다운로드 후 갤러리에 저장합니다.
- iOS PWA 단일 파일은 기본적으로 Files/다운로드에 저장합니다. 작은 파일은 navigator.canShare 검사 후 Web Share API를 선택적으로 사용할 수 있지만 Photos 저장 대상의 제공 여부는 OS가 결정합니다.
- iOS PWA 다중 선택은 서버 ZIP 내보내기를 사용합니다. ZIP은 Files에 저장되며, 다운로드 완료 시 사용자에게 '3단계 시각 가이드 모달'(Files 앱 열기 → 압축 해제 → 사진 보관함으로 공유/저장)을 노출하여 직관적인 사진 임포트를 지원합니다.
- ZIP은 무압축으로 CPU를 아끼고 예상 크기를 예약해 SSD를 보호합니다. ZIP 임시 공간은 사용자 사진 quota에는 포함하지 않고 전체 SSD 여유 공간 검사에 포함합니다.
- 사용자별 활성 export는 기본 1개, TTL은 24시간입니다.
- export가 참조하는 자산은 export 완료·취소·만료 전까지 영구 purge를 보류합니다.

```mermaid
flowchart TD
    Start["사용자가 다중 사진/동영상 선택 후 [다운로드] 클릭"] --> CheckCount{"선택 파일 수"}
    
    CheckCount -->|단일 파일| SingleFlow["단일 파일 다운로드"]
    SingleFlow --> CanShare{"navigator.canShare 검사"}
    CanShare -->|지원됨| ShareSheet["iOS 시스템 공유 시트 표시<br/>(사용자가 '이미지 저장' 선택 가능)"]
    CanShare -->|미지원/대용량| DirectDownload["Safari Files/다운로드 폴더에 직접 저장"]

    CheckCount -->|다중 파일| MultiFlow["POST /api/v1/exports (자산 ID 목록)"]
    MultiFlow --> ServerZip["FastAPI: 무압축 ZIP 백그라운드 패키징<br/>(CPU 절약 & SSD 임시 공간 예약)"]
    ServerZip --> ZipReady["GET /exports/{id}/content<br/>(Safari를 통해 Files 앱으로 ZIP 다운로드)"]
    
    ZipReady --> ShowModal["3단계 시각 가이드 모달 자동 표시 (D-DES1)"]
    
    subgraph StepGuide["사용자 3단계 조작 가이드"]
        Step1["1단계: iPhone '파일(Files)' 앱 열기"]
        Step2["2단계: 다운로드된 ZIP 파일을 탭하여 압축 해제"]
        Step3["3단계: 풀린 폴더에서 [전체 선택] → [공유] → [사진 저장] 탭"]
    end
    
    ShowModal --> Step1 --> Step2 --> Step3
    Step3 --> Finish["iOS 사진 보관함(Photos) 가져오기 완료 🎉"]
```

## 8. 파일·DB 일관성 및 운영 상태

- 업로드 완료는 멱등적으로 처리하고 completed_asset_id를 보존합니다.
- 영구 삭제는 purging 상태와 재시도 가능한 purge job을 사용합니다.
- DB 상태를 먼저 기록하고 원본·미리보기를 멱등적으로 삭제한 뒤 DB 관계를 제거합니다.
- 서버 재시작 후 남은 purge job을 재실행합니다.
- DB row가 있으나 파일이 없으면 접근을 차단하고 오류로 표시합니다.
- DB row 없이 파일이 있으면 고아 파일 후보로 기록하고 즉시 삭제하지 않습니다.
- DB 백업은 SQLite online backup API(.backup)를 사용하여 트랜잭션 도중 복사로 인한 WAL 손상을 방지합니다.
- 공간 정책은 전체 용량 기준 20% 여유 미만에서 관리자 상태 화면과 로컬 로그에 주의 표시, max(전체 용량의 15%, 100GB) 미만에서 신규 업로드/export 차단, 10% 미만에서 로컬 로그에 긴급 기록으로 통일합니다. SSD 마운트/API 프로세스 상태는 systemd timer 등 API와 독립된 호스트 감시 작업이 journald 및 로컬 상태 파일에 기록합니다. 관리자 화면은 해당 상태를 노출하지만 외부 웹훅·SMS·푸시 알림은 제공하지 않습니다. API/호스트 전체 장애 시 원격 즉시 알림이 없다는 한계를 수용합니다.

```mermaid
flowchart LR
    subgraph Host["Raspberry Pi 5 (Ubuntu 24.04 LTS)"]
        Docker["Docker Compose 환경"]
        Systemd["systemd Mount Unit<br/>(UUID 기반 /srv/photo-data 마운트)"]
    end

    subgraph SSD["외장 2TB SSD (/srv/photo-data)"]
        subgraph DataDirs["디렉터리 구조"]
            Orig["/originals (원본 무손실 보존)"]
            Prev["/previews (1600px 썸네일)"]
            UpTmp["/upload-tmp (임시 청크 버퍼)"]
            ExpTmp["/export-tmp (무압축 ZIP, 24h TTL)"]
            DBDir["/db (app.sqlite3, WAL 모드)"]
            DBBackup["/db-backups (일일 온라인 .backup)"]
        end
    end

    subgraph HealthMonitoring["로컬 장애 감지 및 상태 기록"]
        DiskCheck{"디스크 여유 공간<br/>검사"}
        MountCheck{"외장 SSD<br/>마운트 검사"}
        BackupCheck{"SQLite .backup<br/>성공 여부"}
        
        LocalHealthLog["journald + 로컬 상태 파일"]
        AdminStatus["관리자 상태 화면 (API 정상 시)"]
    end

    Systemd -->|"마운트 성공 시에만"| Docker
    Docker --> Orig
    Docker --> DBDir

    DiskCheck -->|"여유 공간 20% 미만 (주의)"| LocalHealthLog
    DiskCheck -->|"여유 공간 max(15%, 100GB) 미만"| BlockWrite["신규 업로드 및 export 차단"]
    DiskCheck -->|"여유 공간 10% 미만 (긴급)"| LocalHealthLog
    MountCheck -->|"I/O 에러 / 언마운트 감지"| LocalHealthLog
    MountCheck -->|"미마운트 시"| EmergencyStop["FastAPI 파일 쓰기 즉시 중단 (OS Root 오염 방지)"]
    BackupCheck -->|"백업 실패 시"| LocalHealthLog

    LocalHealthLog --> AdminStatus
```

## 9. 날짜/EXIF 처리

시간대 정보가 있는 촬영 시각만 UTC로 변환합니다. 시간대 없는 EXIF DateTimeOriginal을 임의로 UTC로 간주하지 않습니다. 원본 문자열, 로컬 시각, offset 여부, UTC 값을 구분해 저장하고 시간대 미확정 데이터가 잘못 이동된 것처럼 표시되지 않도록 합니다.

## 10. 확장 경계

MVP는 단일 서버·SQLite·로컬 파일시스템입니다. PWA와 관리자 웹은 React 기술을 공유해도 빌드 산출물·호스트·세션 정책은 분리합니다. 사용자 증가나 복잡한 동시 쓰기 요구가 생기기 전까지 PostgreSQL, 객체 스토리지, 별도 작업 큐는 도입하지 않습니다.
