# HashiCorp Vault 테스트 및 인프라 연동 환경

WSL2 Ubuntu(`Ubuntu-26.04`) 기반의 Docker 컨테이너 인프라 운영 가이드입니다.  
**본 프로젝트의 핵심 목적은 [HashiCorp Vault]의 기능 검증, 시크릿 수명주기 관리, 그리고 타 서비스(WAS, CI/CD, DBMS)와의 시크릿 연동 테스트를 안전하게 수행하기 위한 격리 환경을 구축하는 것입니다.**

---

## 🎯 프로젝트 목적 (Project Objectives & Purpose)

이 인프라 환경은 단순한 웹 서비스 호스팅이 아닌, **HashiCorp Vault 중심의 엔터프라이즈 보안 및 시크릿 관리 아키텍처를 테스트**하기 위해 설계되었습니다.

```
                  ┌─────────────────────────────────────────┐
                  │    HashiCorp Vault (10.20.0.40:8200)    │
                  │   - KV Secrets / Transit / Dynamic DB   │
                  │   - AppRole / Token 인증 & 권한 정책     │
                  └────────────────────┬────────────────────┘
                                       │ (internal-net 시크릿 주입)
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────────┐
│     Tomcat (WAS)      │  │    Jenkins (CI/CD)    │  │     MySQL (DBMS)      │
│  DB 연결 정보 동적 주입  │  │  빌드 자격 증명 안전 회수 │  │  동적 계정 자동 생성/회수 │
└───────────────────────┘  └───────────────────────┘  └───────────────────────┘
```

### 1. HashiCorp Vault 핵심 기능 테스트
- **정적 시크릿 중앙화 (KV Secrets Engine v2)**: 소스코드에 하드코딩된 API Key, DB 패스워드 등을 Vault에 안전하게 격리 보관하고 버전 관리/롤백 검증
- **동적 자격 증명 (Dynamic Secrets)**: MySQL 시크릿 엔진을 통해 필요할 때마다 임시 DB 사용자 계정을 즉시 생성하고 TTL 만료 시 자동 파기(Revoke)하는 시나리오 테스트
- **데이터 암호화 (Transit Secrets Engine)**: DB에 저장되는 민감 정보를 Vault API를 통해 암호화/복호화하는 EaaS(Encryption as a Service) 테스트
- **인증 및 접근 제어 (Auth Methods & Policies)**: AppRole, Token 기반의 세분화된 접근 권한(ACL Policy) 테스트

### 2. 타 서비스와의 연동 시나리오
- **Tomcat ↔ Vault**: 웹 애플리케이션 시작 시 Vault API 또는 Vault Agent를 통해 DB 접속 자격 증명을 주입받는 테스트
- **Jenkins ↔ Vault**: CI/CD 파이프라인 빌드 중 민감한 배포 키 및 토큰을 Vault에서 안전하게 조회하여 사용하는 테스트
- **MySQL ↔ Vault**: Vault의 Database Secrets Engine의 타깃 DB로 연동되어 동적 계정 발급 대상 역할 수행
- **Nginx**: 외부 SSL 종단 처리 및 향후 Vault API/UI 리버스 프록시 연동 테스트

---

## 🏗️ 시스템 아키텍처 (System Architecture)

```
[ 클라이언트 / 개발자 로컬 PC ]
    │                      │                      │                      │                      │
    │ HTTPS (443)           │ HTTP (8081)          │ HTTP (8080)          │ MySQL (127.0.0.1:3306)│ Vault UI (127.0.0.1:8200)
    ▼                      │                      │                      │                      │
┌──────────────────────┐   │                      │                      │                      │
│ Nginx (10.10.0.10)   │   │                      │                      │                      │
│ - SSL Termination    │   │                      │                      │                      │
│ - 80 -> 443 Redirect │   │                      │                      │                      │
└──────────┬───────────┘   │                      │                      │                      │
           │               ▼                      ▼                      ▼                      ▼
           │       ┌───────────────┐      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
           │       │    Tomcat     │      │    Jenkins    │      │     MySQL     │      │ HashiCorp     │
           │       │ (10.10.0.20)  │      │ (10.10.0.250) │      │ (10.10.0.30)  │      │ Vault (.40)   │
           │       └───────┬───────┘      └───────┬───────┘      └───────┬───────┘      └───────┬───────┘
           │               │                      │                      │                      │
═══════════╪═══════════════╪══════════════════════╪══════════════════════╪══════════════════════╪════════ public-net (10.10.0.0/16)
           │               │                      │                      │                      │
           ▼               ▼                      ▼                      ▼                      ▼
     10.20.0.10       10.20.0.20            10.20.0.250            10.20.0.30             10.20.0.40
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                          internal-net                                          │
│                         (10.20.0.0/16 백엔드 격리 네트워크 / --internal)                         │
│                    * 외부 인터넷 및 호스트 포트 직결 차단, 컨테이너 전용 사설망                    │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📁 디렉터리 구조 (Directory Structure)

```text
/Docker
├── Compose/                           # 서비스별 독립 Docker Compose 명세
│   ├── Jenkins/
│   │   └── docker-compose.yaml        # Jenkins CI/CD (v3.8)
│   ├── MySQL/
│   │   └── docker-compose.yaml        # MySQL 8.0 DBMS (v3.8, 로컬 루프백 바인딩)
│   ├── Nginx/
│   │   ├── conf.d/                    # 임시/백업 설정 디렉터리
│   │   ├── html/                      # 정적 웹 리소스 디렉터리
│   │   ├── logs/                      # Nginx 로그
│   │   └── docker-compose.yaml        # Nginx 리버스 프록시 (v3.8)
│   ├── Tomcat/
│   │   └── docker-compose.yaml        # Tomcat WAS (v3.8)
│   └── Vault/
│       └── docker-compose.yaml        # HashiCorp Vault (v3.8, 파일 스토리지 & UI 활성화)
│
├── Jenkins/                           # Jenkins 영속 데이터 볼륨 마운트
│   ├── Home/                          # Jenkins Root (/var/jenkins_home) 설정, 플러그인, 빌드 기록
│   └── Workspace/                     # 빌드 파이프라인 워크스페이스
│
├── mysql/                             # MySQL 영속 데이터 볼륨 마운트
│   └── data/                          # MySQL DB 데이터 파일 (/var/lib/mysql)
│
├── nginx/                             # Nginx 컨테이너 바인드 마운트
│   ├── conf.d/
│   │   └── default.conf               # 가상 호스트, SSL, 리버스 프록시 설정
│   ├── html/
│   │   └── index.html                 # 기본 인덱스 HTML
│   ├── logs/                          # access.log / error.log
│   └── ssl/                           # SSL 인증서 (nginx.crt) 및 개인키 (nginx.key)
│
├── tomcat/                            # Tomcat 컨테이너 바인드 마운트
│   ├── webapps/                       # 웹 애플리케이션 루트 및 관리자 콘솔 (ROOT, manager, host-manager 등)
│   └── logs/                          # catalina.log, localhost_access_log 등 톰캣 로그
│
├── vault/                             # HashiCorp Vault 영속 볼륨 마운트
│   ├── config/
│   │   └── vault.hcl                  # Vault 서버 설정 (File 스토리지, TCP 리스너, UI 활성화)
│   ├── data/                          # 암호화된 시크릿 키/엔진 영속 저장소 (/vault/data)
│   └── logs/                          # Vault 감사 및 접근 로그 (/vault/logs)
│
└── README.md                          # 프로젝트 인프라 문서 (본 파일)
```

---

## 🌐 도커 네트워크 설계 및 사전 필수 작업 (Prerequisites)

각 서비스(`Compose/*`)가 개별 docker-compose 파일로 분리되어 있으므로, **컨테이너를 기동하기 전 아래 2개의 공용 브리지 네트워크가 호스트 Docker 환경에 반드시 사전 생성**되어 있어야 합니다.

### 1. 네트워크 사전 생성 이유 및 원리
- **개별 Compose 간 통신 보장**: 각 서비스 Compose 파일에서 `networks: { public-net: { external: true }, internal-net: { external: true } }`로 선언하여 동일한 네트워크를 공유합니다.
- **보안 격리 (`--internal`) 강제 적용**: `internal-net`은 생성 시 반드시 `--internal` 플래그를 지정해야 게이트웨이가 제거되어, 외부 인터넷이나 호스트 포트 포워딩을 통한 데이터 유출을 원천 방지할 수 있습니다.

### 2. [필수] 도커 네트워크 사전 생성 명령어

새로운 환경이나 네트워크가 없는 환경에서는 **서비스 기동 전 아래 명령어를 순서대로 실행**해야 합니다:

```bash
# 1. 외부 통신 및 호스트 포트 매핑용 공용 네트워크 생성
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/16 \
  --gateway 10.10.0.1 \
  public-net

# 2. 백엔드 시크릿 전송용 완전 격리 네트워크 생성 (--internal 플래그 필수!)
docker network create \
  --driver bridge \
  --subnet 10.20.0.0/16 \
  --gateway 10.20.0.1 \
  --internal \
  internal-net
```

> **Windows PowerShell에서 실행 시:**
> ```powershell
> wsl -d Ubuntu-26.04 docker network create --driver bridge --subnet 10.10.0.0/16 --gateway 10.10.0.1 public-net
> wsl -d Ubuntu-26.04 docker network create --driver bridge --subnet 10.20.0.0/16 --gateway 10.20.0.1 --internal internal-net
> ```

### 3. 네트워크 생성 확인 및 검증
```bash
# 네트워크 목록 확인
docker network ls

# internal-net 격리 속성(Internal: true) 검증
docker network inspect internal-net --format '{{.Internal}}'
# 출력 결과: true
```

### 4. [필수] Nginx SSL 자체 서명 인증서 생성
보안을 위해 SSL 개인키(`nginx.key`)는 Git 저장소에 커밋되지 않습니다.  
새로운 환경에서 Nginx를 기동하기 전, 아래 명령어로 1초 만에 자체 서명(Self-Signed) 인증서를 생성해 두어야 정상 작동합니다:

```bash
# SSL 디렉터리 생성 및 자체 서명 인증서 발급 (1초 소요)
mkdir -p /Docker/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /Docker/nginx/ssl/nginx.key \
  -out /Docker/nginx/ssl/nginx.crt \
  -subj "/CN=localhost"
```

> **Windows PowerShell (WSL)에서 실행 시:**
> ```powershell
> wsl -d Ubuntu-26.04 sh -c "mkdir -p /Docker/nginx/ssl && openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /Docker/nginx/ssl/nginx.key -out /Docker/nginx/ssl/nginx.crt -subj '/CN=localhost'"
> ```

---

### 📌 컨테이너 고정 IP 및 엔드포인트 맵핑 표

| 서비스 | 컨테이너명 | 이미지 | public-net IP | internal-net IP | 호스트 바인딩 포트 | 접속 URL / 엔드포인트 | 비고 |
|:---|:---|:---|:---:|:---:|:---|:---|:---|
| **Vault** | `vault` | `hashicorp/vault:latest` | `10.10.0.40` | `10.20.0.40` | `127.0.0.1:8200` | `http://127.0.0.1:8200/ui` | **핵심 테스트 대상** (시크릿 중앙 관리) |
| **MySQL** | `mysql` | `mysql:8.0` | `10.10.0.30` | `10.20.0.30` | `127.0.0.1:3306` | `127.0.0.1:3306` | Vault 동적 자격증명 연동 DB |
| **Tomcat** | `tomcat` | `tomcat:9.0-jdk17-temurin` | `10.10.0.20` | `10.20.0.20` | `8081:8080` | `http://localhost:8081/` | WAS (Vault 시크릿 주입 테스트용) |
| **Jenkins** | `jenkins` | `jenkins/jenkins:lts-jdk21` | `10.10.0.250` | `10.20.0.250` | `8080`, `50000` | `http://localhost:8080/` | CI/CD (Vault AppRole 파이프라인 연동) |
| **Nginx** | `nginx` | `nginx:alpine` | `10.10.0.10` | `10.20.0.10` | `80`, `443` | `https://localhost/` | SSL 종단 & 리버스 프록시 |

---

## ⚙️ 서비스별 상세 설정

### 1. HashiCorp Vault (Secrets & Encryption Management) - 🌟 핵심 대상
- **Compose 경로:** `Compose/Vault/docker-compose.yaml`
- **설정 파일 (`vault/config/vault.hcl`):**
  - **스토리지:** `storage "file" { path = "/vault/data" }` (컨테이너 볼륨 영속화)
  - **리스너:** `listener "tcp" { address = "0.0.0.0:8200" tls_disable = 1 }`
  - **UI 지원:** `ui = true` 활성화
- **보안 바인딩 정책 (`127.0.0.1:8200:8200`):**
  - 민감한 시크릿을 다루므로 호스트 외부 노출을 막기 위해 **로컬 루프백(`127.0.0.1`) 전용으로 포트 바인딩**
  - 웹 브라우저에서 `http://127.0.0.1:8200/ui`로 바로 접근 가능
- **보안 역량 (`cap_add`):**
  - `IPC_LOCK` 추가로 프로세스 메모리 스왑 방지 지원

### 2. MySQL (Relational Database)
- **Compose 경로:** `Compose/MySQL/docker-compose.yaml`
- **보안 바인딩:** `127.0.0.1:3306:3306` (로컬 PC 접속만 허용, 외부 네트워크 침입 차단)
- **계정/데이터베이스:** `root` / `rootpassword`, `appdb` (문자셋 `utf8mb4`)
- **Vault 연동:** Vault의 Database Secrets Engine을 통해 동적 DB 사용자 계정을 발급받는 대상 DB로 활용

### 3. Tomcat (Web Application Server)
- **Compose 경로:** `Compose/Tomcat/docker-compose.yaml`
- **JVM 사양:** Heap 256m~512m (`JAVA_OPTS=-Xms256m -Xmx512m -Dfile.encoding=UTF-8`)
- **Vault 연동:** 애플리케이션 시작 시 Vault KV 엔진에서 DB 접속 문자열 및 토큰을 조회하여 사용하는 클라이언트로 활용
- **Host Manager 관리 콘솔 (`/host-manager`):**
  - 가상 호스트(Virtual Host)를 웹 UI에서 동적으로 생성/관리/모니터링하는 관리 애플리케이션
  - Nginx 리버스 프록시(`10.20.0.10`)를 통한 외부 접근 허용을 위해 `webapps/host-manager/META-INF/context.xml`의 `RemoteAddrValve`가 주석 해제(비활성화)되어 있습니다.

### 4. Jenkins (CI/CD 자동화 서버)
- **Compose 경로:** `Compose/Jenkins/docker-compose.yaml`
- **주요 구성:** DooD(Docker-out-of-Docker) 바인딩으로 컨테이너 내 빌드 수행
- **Vault 연동:** Vault 플러그인 또는 CLI/AppRole 인증을 통해 빌드 단계에서 시크릿을 안전하게 주입받는 테스트
- **관리자 인증:** 최초 접속 시 `/var/jenkins_home/secrets/initialAdminPassword`에 기록된 32자리 마스터 패스워드로 인증

### 5. Nginx (리버스 프록시 및 SSL 종단)
- **Compose 경로:** `Compose/Nginx/docker-compose.yaml`
- **주요 설정:** HTTP 80 -> HTTPS 443 301 리다이렉트, SSL/TLS TLSv1.2/1.3 적용, Tomcat 및 향후 Vault 프록시 연동
- **설정 검증 도구:** 설정 수정 후 `docker exec -it nginx nginx -t` 명령으로 문법 유효성 사전 검증 지원

---

## 🚀 운영 및 테스트 가이드 (Operations)

### 1. 권장 서비스 시작 순서

네트워크가 사전 생성된 후, 의존성에 맞추어 순서대로 기동하는 것을 권장합니다:

```bash
# 1단계: 핵심 시크릿 및 데이터베이스 기동
docker compose -f /Docker/Compose/Vault/docker-compose.yaml up -d
docker compose -f /Docker/Compose/MySQL/docker-compose.yaml up -d

# 2단계: 애플리케이션 및 CI/CD 플랫폼 기동
docker compose -f /Docker/Compose/Tomcat/docker-compose.yaml up -d
docker compose -f /Docker/Compose/Jenkins/docker-compose.yaml up -d

# 3단계: 웹 리버스 프록시 기동
docker compose -f /Docker/Compose/Nginx/docker-compose.yaml up -d
```

### 2. Vault 초기화 (최초 1회 필수)

Vault 서버는 최초 기동 시 데이터가 암호화되지 않은 봉인(Sealed) 상태입니다. 브라우저(`http://127.0.0.1:8200/ui`)에서 진행하거나 CLI로 초기화합니다:

```bash
# 1. 초기화 실행 (Key Shares & Root Token 발급)
docker exec -it vault vault operator init

# 2. 발급받은 Unseal Key 3개로 언실 수행 (3회 반복)
docker exec -it vault vault operator unseal <Unseal_Key_1>
docker exec -it vault vault operator unseal <Unseal_Key_2>
docker exec -it vault vault operator unseal <Unseal_Key_3>

# 3. 상태 확인
docker exec -it vault vault status
```

### 3. Nginx 설정 문법 점검 및 무중단 적용 (`nginx -t`)

설정 파일(`nginx/conf.d/default.conf` 등)을 수정한 후, **문법 에러로 인해 Nginx 컨테이너가 비정상 종료(Downtime)되는 것을 방지하기 위해 반드시 사전 검증**을 수행합니다:

```bash
# 1. Nginx 설정 문법 검사 (Syntax OK 여부 확인)
docker exec -it nginx nginx -t

# 2. 문법 정상 확인 후 무중단(Zero-Downtime) 설정 반영
docker exec -it nginx nginx -s reload
```
- **설명:** `nginx -t`는 세미콜론 누락, 괄호 닫힘 오류, 인증서 파일 경로 오류 등을 실제 서비스에 적용하기 전에 미리 파싱하여 테스트해 주는 명령어로, 무중단 운영의 핵심 절차입니다.

### 4. Jenkins 관리자 초기 패스워드 확인 방법

Jenkins 컨테이너 기동 후 웹 콘솔(`http://localhost:8080`)에 처음 접속하면 Unlock Jenkins 화면에서 관리자 패스워드를 요구합니다:

```bash
# 방법 1: 컨테이너 내부 보안 파일에서 직접 확인
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# 방법 2: 컨테이너 초기 기동 로그에서 확인
docker compose -f /Docker/Compose/Jenkins/docker-compose.yaml logs | grep -A 2 -B 2 "initialAdminPassword"
```
- **설명:** 출력되는 32자리의 영문/숫자 조합 키를 복사하여 웹 화면에 입력하면 관리자 계정 생성 및 플러그인 설치 화면으로 진입할 수 있습니다.
- *(참고: 본 환경의 `JAVA_OPTS`에 `-Djenkins.install.runSetupWizard=false`가 지정된 경우 설치 마법사가 자동으로 생략될 수 있습니다.)*

### 5. Tomcat Host Manager 접속 및 원격 허용 점검

Tomcat의 가상 호스트(Virtual Host)를 웹 UI에서 관리하는 애플리케이션으로, 기본 보안 정책으로 인해 원격 접속 허용 작업이 수반됩니다:

- **접속 URL:** `https://localhost/host-manager/html` (Nginx 경유) 또는 `http://localhost:8081/host-manager/html` (Tomcat 직접 접근)
- **역할 및 용도:**
  - `Catalina` 엔진 하위의 호스트 추가/제거, AppBase 디렉터리 매핑, XML 설정 검증 등을 웹 브라우저에서 동적으로 수행
- **접근 권한 설정 (`conf/tomcat-users.xml`):**
  - 접속 시 브라우저 인증 팝업이 뜨며, `admin-gui` (또는 `admin-script`) 역할이 부여된 계정만 로그인이 허용됩니다:
    ```xml
    <role rolename="admin-gui"/>
    <user username="admin" password="adminpassword" roles="admin-gui"/>
    ```
- **원격/프록시 접속 허용 (`context.xml`):**
  - Tomcat은 기본적으로 `127.0.0.1` 로컬 외의 접속을 차단(`403 Forbidden`)하는 `RemoteAddrValve`가 켜져 있습니다.
  - 현재 환경은 Nginx 리버스 프록시(`10.20.0.10`)를 경유하여 접속할 수 있도록 `webapps/host-manager/META-INF/context.xml` 내부의 `RemoteAddrValve`가 올바르게 주석 처리(비활성화)되어 있습니다.

---

## 🛠️ 트러블슈팅 및 팁 (Troubleshooting)

### Q1. `network public-net (or internal-net) declared as external, but could not be found` 오류
- **원인:** 서비스별 Compose 파일이 외부 네트워크(`external: true`)를 참조하도록 설정되어 있으나, 해당 도커 네트워크가 호스트에 아직 생성되지 않았을 때 발생합니다.
- **해결:** 서비스 기동 전, 본 문서의 **[도커 네트워크 사전 생성 명령어]** 섹션에 명시된 `docker network create` 명령어로 두 네트워크를 먼저 생성해 주세요.

### Q2. MySQL / Vault 포트 매핑 시 `ports:`가 무시되는 현상
- **원인:** Docker에서 `--internal` (`"Internal": true`) 속성을 가진 네트워크에만 단독 연결된 컨테이너는 보안 정책상 호스트 포트 포워딩(`-p`, `ports:`)이 자동으로 비활성화됩니다.
- **해결:** `public-net`을 함께 연결하되, 외부 침입을 막기 위해 `ports:`를 **`127.0.0.1:<포트>:<포트>`**로 지정하여 로컬 PC 전용으로 안전하게 개방합니다.

### Q3. Tomcat `/manager` 콘솔 접속 시 404/기동 실패 오류
- **원인:** `/Docker/tomcat/webapps/manager/META-INF/context.xml` 파일 내 닫힘 주석(`-->`) 문법 오류로 인한 XML 파싱 실패
- **해결:** `context.xml` 파일을 열어 불필요한 `-->` 문자를 정리하거나 정상 XML 형태로 수정한 후 톰캣을 재시작합니다.
