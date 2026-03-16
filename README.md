# 🛡️ Sentinel Manager — 서버 보안 모니터링 클라이언트 (POC)

> **서버의 보안 상태를 실시간으로 감시하고 보안 정책을 실행하는 클라이언트 프로그램.**  
> **레거시 MFC 기반의 Windows 전용 구조를 Modern C++17 + Qt6로 전환하여,**  
> **Windows/Linux 크로스 플랫폼 서버 로그 수집 시스템 및 Docker 배포 자동화를 실증한 Shadow Project.**

---

## 🎯 핵심 성과 한눈에 보기

| 항목 | 이전 (Legacy) | 이후 (Modern) | 상태 |
| :--- | :--- | :--- | :---: |
| **빌드 시스템** | Visual Studio `.sln` (수동) | `CMake` + 커맨드라인 자동화 | ✅ 완료 |
| **프레임워크** | MFC (Windows 전용) | Qt 6.x (Windows/Linux 지원) | ✅ 완료 |
| **언어 표준** | C++98/03 | C++17 (스마트 포인터, RAII) | ✅ 완료 |
| **Windows 로그 수집** | 없음 | `Windows Event Log API (winevt.h)` 실시간 수집 | ✅ 완료 |
| **Linux 로그 수집** | 없음 | `systemd-journald API (sd-journal.h)` 실시간 수집 | ✅ 완료 |
| **클라이언트-서버 통신** | 없음 | Qt Socket 기반 로그 스트리밍 (10초 주기) | ✅ 완료 |
| **Docker 배포** | 없음 | 멀티 스테이지 `Dockerfile` + `.dockerignore` | ✅ 완료 |
| **On-Premise Git (Gitea)** | SVN 또는 없음 | Docker 기반 Gitea 서버 구축 (폐쇄망 지원) | ✅ 완료 |
| **CI/CD 자동화** | 없음 | Gitea Actions + Self-hosted Runner (W/L 이중 빌드) | ✅ 완료 |
| **코드 품질 분석** | 없음 | SonarQube (Docker) + `sonar-project.properties` | ✅ 완료 |
| **단위 테스트** | 없음 | Google Test (`FetchContent`) 환경 구축 | ✅ 완료 |

---

## 🏗️ 시스템 아키텍처

```
[ 보안 담당자 / 관리자 ]
        │
        │ (클라이언트 실행)
        ▼
[ Sentinel Manager Client (Qt6 GUI) ]
  - 서버 로그인 및 인증
  - 실시간 보안 로그 모니터링
  - 보안 정책 실행 요청
        │
        │  TCP Socket (Qt Network)
        │  LOG_REQ (10초 주기)
        │
        ▼
┌────────────────────────────────────────┐
│     Sentinel Server (TCP 포트 8080)     │
│     (관리 대상 서버에 설치)              │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │         ILogProvider (인터페이스) │  │
│  └────────┬─────────────────────────┘  │
│           │                            │
│    ┌──────┴──────┐                     │
│    │             │                     │
│    ▼             ▼                     │
│  [Windows]    [Linux]                  │
│  WinEvent     LinuxJournal             │
│  LogProvider  Provider                 │
│  (winevt.h)   (sd-journal.h)           │
└──────────────────────────────────────-─┘
        │
        │  LOG_RES (패킷)
        │  - timestamp (KST)
        │  - source, level, content
        ▼
[ 클라이언트 화면에 실시간 보안 로그 표시 ]
```

---

## 📦 프로젝트 구조

```
RedCastle-Manager/
├── Manager_Qt_Version/
│   ├── CMakeLists.txt              # 메인 빌드 설정 (Windows/Linux 분기)
│   ├── src/
│   │   ├── core/
│   │   │   ├── network/
│   │   │   │   ├── logworker.h/cpp      # 백그라운드 로그 요청 워커
│   │   │   │   ├── packetparser.h/cpp   # 패킷 파싱 로직
│   │   │   │   └── qtsocketclient.h/cpp # Qt 소켓 클라이언트
│   │   │   ├── storage/                 # 사용자 데이터 저장 (JSON)
│   │   │   └── registrymanager.h/cpp    # 설정값 관리
│   │   ├── ui/
│   │   │   ├── dialog.h/cpp             # 메인 UI (로그 테이블 포함)
│   │   │   └── serverlogindialog.h/cpp  # 서버 접속 다이얼로그
│   │   └── utils/
│   │       └── logger.h/cpp             # spdlog 기반 로거
│   └── MockServer/
│       ├── CMakeLists.txt               # MockServer 빌드 설정
│       ├── Dockerfile                   # 멀티 스테이지 Docker 빌드
│       ├── mockserver.h/cpp             # 서버 진입점 및 핸들러
│       ├── logprovider.h                # ILogProvider 인터페이스
│       ├── wineventlogprovider.h/cpp    # Windows 이벤트 로그 수집
│       └── linuxjournalprovider.h/cpp   # Linux journald 수집
├── .dockerignore                        # Docker 빌드 최적화
└── README.md
```

---

## ✅ 단계별 구현 내용

### Phase 1 · 빌드 시스템 현대화 (CMake)

Visual Studio 없이 커맨드라인만으로 빌드 가능한 환경을 구축했습니다.

- `CMakeLists.txt`를 작성하여 Windows(MinGW) / Linux(GCC) 분기 처리
- `FetchContent`를 사용하여 `spdlog`, `GoogleTest`를 외부 의존성 없이 자동 다운로드
- `pkg-config`를 통한 `libsystemd` 감지 및 조건부 링크

```bash
# Windows 빌드
cmake -S Manager_Qt_Version -B build && cmake --build build

# Linux 빌드 (WSL/Ubuntu)
mkdir build_linux && cd build_linux
cmake ../Manager_Qt_Version && make -j$(nproc)
```

---

### Phase 2 · Qt6 UI 및 클라이언트-서버 로그 스트리밍

MFC Dialog를 Qt6 Widget으로 재구성하고, 서버와 실시간으로 통신하는 구조를 설계했습니다.

**데이터 흐름:**
```
[10초 타이머] → LOG_REQ 패킷 전송 → [Sentinel Server] 로그 수집
    → LOG_RES 패킷 수신 → [Qt 테이블] 행 추가 → 최신 로그로 자동 스크롤
```

**주요 구현:**

| 클래스 | 역할 |
| :--- | :--- |
| `LogWorker` | `QThread`에서 10초마다 서버에 로그 요청 |
| `QtSocketClient` | Qt `QTcpSocket` 기반 연결 및 패킷 I/O |
| `PacketParser` | 공통 패킷 포맷 파싱 (Header + Body) |
| `Dialog::onLogReceived` | 수신 로그를 화면 하단 테이블에 렌더링 |

---

### Phase 3 · 크로스 플랫폼 로그 수집 (핵심 기능)

단일 인터페이스(`ILogProvider`)를 통해 OS별 로그 수집 구현을 교체하는 **전략 패턴**을 적용했습니다.

```cpp
// 플랫폼에 따라 자동으로 적합한 구현체가 선택됨
#if defined(Q_OS_WIN)
    auto provider = std::make_unique<WinEventLogProvider>();
#elif defined(Q_OS_LINUX)
    auto provider = std::make_unique<LinuxJournalProvider>();
#endif
```

#### Windows: WinEventLogProvider
- Windows 이벤트 로그 API(`EvtQuery`, `EvtRender`) 사용
- XPath 쿼리로 당일 00:00:00 이후 로그를 소급 수집
- XML 파싱으로 시간, 소스, 레벨, 내용 추출

#### Linux: LinuxJournalProvider
- `libsystemd`의 `sd-journal` API 사용
- `sd_journal_seek_tail` → `sd_journal_wait` → `sd_journal_process` 패턴으로 실시간 감지
- UTC 타임스탬프를 **한국 시간(KST) UTC+9**으로 자동 변환

| OS | API | 로그 소스 | 권한 |
| :--- | :--- | :--- | :--- |
| Windows | `winevt.h` | Windows Event Log | Administrator |
| Linux | `sd-journal.h` | systemd-journald | systemd-journal 그룹 |

---

### Phase 4 · Docker 배포 자동화

멀티 스테이지 빌드로 이미지 크기를 최소화하고, KST 시간대를 컨테이너에 적용했습니다.

```
┌─────────────────────────────────┐
│ Stage 1: builder (ubuntu:22.04) │  <- cmake, g++, qt6-base-dev, libsystemd-dev
│   COPY 소스코드                  │
│   RUN cmake && make             │  -> /build/MockServer 바이너리 생성
└─────────────────┬───────────────┘
                  │  COPY --from=builder (바이너리만 복사)
┌─────────────────▼───────────────┐
│ Stage 2: runtime (ubuntu:22.04) │  <- libqt6core6, libqt6network6, libsystemd0
│   ENV TZ=Asia/Seoul             │  <- KST 시간대 설정
│   EXPOSE 8080                   │
│   ENTRYPOINT ["/usr/local/bin/MockServer"]
└─────────────────────────────────┘
```

**실행 방법:**
```bash
# 이미지 빌드
docker build -t mockserver:latest -f Manager_Qt_Version/MockServer/Dockerfile .

# 컨테이너 실행 (호스트 journal 로그 마운트)
docker run -d \
  --name mockserver \
  -p 8080:8080 \
  -v /var/log/journal:/var/log/journal:ro \
  mockserver:latest
```

---

## 🔬 검증 환경

| 환경 | 역할 | 검증 내용 |
| :--- | :--- | :--- |
| Windows 10 (MinGW-w64 13.1.0) | 클라이언트 개발 & 테스트 | Qt6 빌드, WinEventLog 수집, GUI 렌더링 |
| Windows Server 2025 (VM) | 서버 테스트 | Windows Server 네이티브 환경 로그 수집 검증 |
| Ubuntu 24.04 (VM) | 클라이언트 테스트 | Linux 환경에서의 Qt6 GUI 동작 및 서버 연동 확인 |
| WSL Ubuntu 22.04 | 리눅스 서버 테스트 | LinuxJournalProvider 빌드 및 실시간 저널 수집 |
| Docker Desktop (Ubuntu 22.04) | 배포 자동화 검증 | 멀티 스테이지 빌드 및 컨테이너 실행 |

**실제 테스트 완료된 시나리오:**
1. WSL Ubuntu에서 `Sentinel Server` 빌드 및 실행 (`port 8080`)
2. Windows Qt 클라이언트에서 WSL IP(`172.25.x.x`)로 접속
3. `logger -p user.info "메시지"` 명령 입력 → Qt 화면 테이블에 실시간 표시 ✅

---

## 🛠️ 기술 스택

| 분류 | 기술 |
| :--- | :--- |
| **언어** | C++17 |
| **프레임워크** | Qt 6.x (Core, Network, Widgets) |
| **빌드** | CMake 3.22+ |
| **컴파일러** | MinGW-w64 GCC 13.1 (Windows), GCC 11 (Linux) |
| **로깅** | spdlog 1.14 (`FetchContent`) |
| **테스트** | Google Test (`FetchContent`) |
| **컨테이너** | Docker (멀티 스테이지 빌드, Ubuntu 22.04) |
| **Linux 로그** | libsystemd (`sd-journal.h`) |
| **Windows 로그** | Windows Event Log API (`winevt.h`) |

---

## 🔄 배포 전략 (Two-Track)

> OS 버전에 따른 하위 호환성을 유지하면서 신규 환경에는 최신 버전을 제공하는 이원화 전략

| 구분 | Track 1 (Modern) | Track 2 (Legacy) |
| :--- | :--- | :--- |
| **타겟 OS** | Windows 10/11, Linux | Windows 구형 OS 환경 |
| **기술 스택** | Qt 6 + C++17 | MFC + C++03 |
| **업데이트** | 신규 기능 개발 | 보안 패치만 (Frozen) |
| **배포** | Docker 자동화 빌드 | 수동 레거시 파이프라인 |

---

## 🏭 DevOps 인프라 구축

### On-Premise Git 서버 (Gitea + Docker)

인터넷 연결 없는 폐쇄망(Air-gapped) 환경에서도 웹 기반 코드 관리 및 CI/CD를 사용할 수 있도록 **Gitea**를 Docker로 구축했습니다.

```yaml
# docker-compose.yml (Gitea)
services:
  gitea:
    image: gitea/gitea:latest
    ports:
      - "3000:3000"   # 웹 UI (http://localhost:3000)
      - "2222:22"     # SSH
    volumes:
      - ./gitea-data:/data
```

```bash
docker-compose up -d
# → 브라우저에서 http://localhost:3000 접속, 초기 설정 후 즉시 사용 가능
```

**폐쇄망 오프라인 배포:**
```bash
# 외부망: 이미지 저장
docker save gitea/gitea:latest -o gitea_image.tar

# 폐쇄망: USB로 반입 후 로드
docker load -i gitea_image.tar
```

---

### CI/CD 자동화 (Gitea Actions)

GitHub Actions와 100% 호환되는 **Gitea Actions**를 이용하여 코드를 Push하면 Windows/Linux 빌드가 자동으로 수행됩니다.

#### 전체 구조 ("One Source, Two Builds")

```
개발자
  │
  │  git push → main
  ▼
┌────────────────────────────────────────┐
│    Gitea Server (On-Premise, :3000)    │
│    "빌드 작업 생성 & 배분 (관제탑)"    │
└─────────────┬──────────────────────────┘
              │
    ┌─────────┴──────────┐
    │                    │
    ▼                    ▼
┌──────────────┐  ┌───────────────────────┐
│ Windows PC   │  │ Linux 서버 (Docker)   │
│ act_runner   │  │ act_runner            │
│ (MinGW/Qt6)  │  │ (GCC/Ubuntu 22.04)    │
│              │  │                       │
│ → App.exe    │  │ → App (ELF 바이너리)  │
└──────────────┘  └───────────────────────┘
```

#### Gitea Actions Runner 설정

| 단계 | Windows Runner | Linux Runner |
| :--- | :--- | :--- |
| **실행 환경** | Host(내 PC) 직접 실행 | Docker Container 사용 |
| **Executor** | PowerShell (Shell) | Ubuntu 22.04 이미지 |
| **빌드 도구** | 호스트에 설치된 MinGW, Qt | 컨테이너 내 apt로 설치 |
| **등록 라벨** | `windows-latest` | `ubuntu-latest` |
| **상시 실행** | NSSM으로 Windows 서비스 등록 | systemd 또는 데몬 실행 |

#### CI 워크플로우 예시 (`.gitea/workflows/build.yaml`)

```yaml
name: Build and Test
on:
  push:
    branches: [main]

jobs:
  windows_build:
    runs-on: windows-latest          # Windows Runner가 처리
    steps:
      - uses: actions/checkout@v3
      - name: Configure CMake
        run: cmake -S . -B build -G "MinGW Makefiles"
      - name: Build
        run: cmake --build build

  linux_build:
    runs-on: ubuntu-latest           # Linux Runner (Docker)가 처리
    container:
      image: ubuntu:22.04
    steps:
      - uses: actions/checkout@v3
      - run: apt-get update && apt-get install -y build-essential cmake qt6-base-dev
      - run: cmake -S . -B build && cmake --build build
```

---

### 코드 품질 분석 (SonarQube + Docker)

코드 복잡도, 중복 코드, 잠재적 버그를 시각화하는 **SonarQube**를 Docker Compose로 로컬에 구축했습니다.

```yaml
# docker/sonarqube/docker-compose.yml
services:
  sonarqube:
    image: sonarqube:community
    ports:
      - "9000:9000"     # 웹 대시보드: http://localhost:9000
    depends_on:
      - db
  db:
    image: postgres:15  # SonarQube 전용 DB
```

```bash
# SonarQube 실행
docker-compose -f docker/sonarqube/docker-compose.yml up -d

# 분석 실행 (sonar-scanner 필요)
sonar-scanner -Dsonar.host.url=http://localhost:9000
```

**`sonar-project.properties` 설정:**
```properties
sonar.projectKey=sentinel-manager
sonar.projectName=Sentinel Manager
sonar.sources=Manager_Qt_Version/src,Manager_Qt_Version/MockServer
sonar.language=cpp
```

---

## 💡 MFC → Qt 코드 변환 치트시트

| 항목 | Legacy (MFC) | Modern (Qt6) |
| :--- | :--- | :--- |
| 문자열 | `CString` (MBCS/CP949) | `QString::fromLocal8Bit()` → 내부 UTF-16 |
| 메모리 | `new` / `delete` | `std::make_unique<T>()` |
| 스레드 | `CreateThread` | `QThread` / `std::thread` |
| 동기화 | `CRITICAL_SECTION` | `std::mutex` + `std::lock_guard` |
| 파일 경로 | `CFile`, `GetModuleFileName` | `std::filesystem::path` |
| UI 이벤트 | `ON_BN_CLICKED(...)` | `connect(btn, &QPushButton::clicked, ...)` |
| 설정 저장 | Registry (`GetProfileString`) | `QSettings` (INI/JSON) |

---

## 📈 기대 효과

- **안정성** — 스마트 포인터와 RAII로 메모리 누수 및 Race Condition 원천 제거
- **확장성** — OS 종속성 제거로 클라우드·리눅스 서버 환경 진출 용이
- **자동화** — Gitea Actions + Docker로 코드 Push 한 번에 Windows/Linux 동시 빌드 검증
- **품질** — SonarQube를 통한 코드 복잡도·중복 코드 지속 모니터링
- **보안성** — 폐쇄망 환경에서 자체 Git 서버 및 CI/CD 파이프라인 구축 가능성 실증
