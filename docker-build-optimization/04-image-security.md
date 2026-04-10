# 이미지 보안 — 루트 탈출과 공격 면적 최소화

## 🎯 핵심 질문

- Docker 컨테이너의 root 사용자가 왜 위험한가? 호스트 OS의 root와 다른 걸까?
- 쉘이나 패키지 매니저가 없는 이미지를 실행할 수 있을까?
- 불필요한 라이브러리를 제거하면 보안이 정말 향상될까?

## 🔍 왜 이 개념이 실무에서 중요한가

**컨테이너 보안의 중대한 위협:**

1. **컨테이너 탈출 취약점**
   - CVE-2018-1000001 (glibc): root가 setuid를 우회해 호스트 root 탈출
   - CVE-2019-5736 (runc): 컨테이너 root → 호스트 root (즉시 패치 필수)
   - 최근 kernel 취약점: root 권한이 있으면 호스트 커널 exploit 가능

2. **공급망 공격 (Supply Chain Attack)**
   - npm 패키지 1,000개 중 일부가 악성 코드 포함
   - Docker 이미지에 필요 없는 패키지 200개 포함 = 취약점 200개 잠재

3. **규정 준수 (Compliance)**
   - PCI-DSS: root 실행 금지, 취약점 정기 스캔 필수
   - HIPAA: 불필요한 라이브러리 제거 필수
   - SOC 2: 이미지 보안 정책 문서화

4. **실무 사례**
   - Capital One (2019): 컨테이너 탈출로 100만 고객 정보 유출
   - Tesla AWS (2019): 공개 ECR 저장소에 민감 정보 노출

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

### 문제 1: Root 사용자로 애플리케이션 실행

```dockerfile
# ❌ 나쁜 Dockerfile: root로 실행
FROM openjdk:17-jre-slim

WORKDIR /app

COPY app.jar .

# USER 지정 안 함 = root로 실행!
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
$ docker run myapp:bad

# 컨테이너 내부에서
$ whoami
root  # ❌ root 권한으로 실행 중!

$ id
uid=0(root) gid=0(root) groups=0(root)

# 컨테이너 탈출 후 호스트에서
$ id
uid=0(root) gid=0(root)  # 호스트도 root!
```

**위험성:**
- 컨테이너 탈출 시 호스트 root 권한 획득
- 다른 컨테이너 침투 가능
- 호스트 파일시스템 전체 접근 가능

### 문제 2: 전체 OS 레이어 포함

```dockerfile
# ❌ 나쁜 Dockerfile: 불필요한 도구 포함
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    curl \
    git \
    wget \
    openssh-client \
    telnet \
    telnetd \
    netcat \
    socat \
    nmap \
    vim \
    nano \
    mysql-client \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

COPY app.jar .

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**문제점:**
```
이미지 크기: 500MB+

포함된 패키지:
- openssh-client (30MB) - SSH 클라이언트, 애플리케이션에 불필요
- telnet/telnetd (5MB) - 오래된 프로토콜, 보안 위험
- netcat (2MB) - 네트워크 유틸리티, 공격자 도구
- nmap (50MB) - 포트 스캐닝 도구, 공격자 선호 도구
- vim/nano (100MB) - 텍스트 에디터, 런타임에 불필요
- MySQL/PostgreSQL 클라이언트 (80MB) - 특정 용도만 필요

취약점: 200개+ (설치된 패키지별 잠재적 CVE)
공격자의 침투 도구: 대부분 포함됨 (nmap, netcat 등)
```

### 문제 3: 취약점 스캔 없음

```dockerfile
# ❌ 나쁜 Dockerfile: 취약점 미검사
FROM ubuntu:22.04  # 특정 버전 명시 안 함
RUN apt-get install -y \
    openssl \
    libssl-dev \
    # 버전 핀 안 함, 자동 업그레이드로 취약점 패치 어려움
...
```

**문제점:**
- OpenSSL 버전 1.1.1 (사용 중단됨, 취약점 지속 발견)
- 컨테이너 빌드 시마다 다른 버전 설치 가능
- 취약점 발견 후 빠른 패치 불가

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 방식 1: Non-root 사용자 생성

```dockerfile
# ✅ 좋은 Dockerfile: non-root 사용자
FROM openjdk:17-jre-slim

# Step 1: non-root 사용자 생성
RUN groupadd -r appgroup && \
    useradd -r -g appgroup appuser

WORKDIR /app

COPY app.jar .

# Step 2: 파일 소유권 변경
RUN chown -R appuser:appgroup /app

# Step 3: non-root 사용자로 전환
USER appuser

# Step 4: 애플리케이션 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
$ docker run myapp:good

# 컨테이너 내부에서
$ whoami
appuser  # ✅ non-root 사용자

$ id
uid=1000(appuser) gid=1000(appgroup) groups=1000(appgroup)

# 컨테이너 탈출 시
$ id
uid=1000(appuser) gid=1000(appgroup)  # 호스트도 일반 사용자

# 호스트 파일 접근 불가
$ cat /etc/shadow
Permission denied
```

### 방식 2: Distroless 이미지 (Google)

```dockerfile
# ✅ 최적화된 Dockerfile: distroless 베이스
FROM openjdk:17-jdk AS builder

WORKDIR /build

COPY . .

RUN ./gradlew clean build -x test


# Distroless 이미지 사용 (쉘도 패키지 매니저도 없음!)
FROM gcr.io/distroless/java17-nonroot

WORKDIR /app

# non-root 사용자가 기본 (UID 65532)
COPY --from=builder --chown=65532:65532 \
    /build/build/libs/app.jar .

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Distroless의 특성:**
```
크기: 150MB (openjdk:17-jre) → 70MB (distroless)

포함된 것:
- Java 런타임
- 기본 라이브러리 (libc 등)
- 인증서

포함되지 않은 것:
- 쉘 (/bin/sh 없음)
- 패키지 매니저 (apt 없음)
- 불필요한 도구 (curl, wget, vim 등)
- UID 0 (root) 사용자

보안:
- 쉘이 없어 interactive 접근 불가
- 공격자 도구 설치 불가
- 최소한의 라이브러리만 포함
```

### 방식 3: Alpine Linux (최소 크기)

```dockerfile
# ✅ Alpine 사용: 매우 작은 이미지
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
# Alpine: 150MB (Ubuntu: 500MB)

WORKDIR /build

COPY . .

RUN mvn clean package -DskipTests


FROM eclipse-temurin:17-jre-alpine

# non-root 사용자 생성
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=builder --chown=appuser:appgroup \
    /build/target/app.jar .

USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Alpine의 특성:**
```
크기: 150MB (openjdk:17-jre) → 50MB (alpine)
패키지 매니저: apk (apt보다 작음)
C 라이브러리: musl libc (glibc 없음, 호환성 이슈 가능)

장점:
- 매우 작은 크기
- 적당한 패키지 관리
- 보안 업데이트 활발

단점:
- musl libc 호환성 이슈 (특정 라이브러리)
- 일부 Go/Rust 프로젝트에서 문제
```

## 🔬 내부 동작 원리

### 1. **컨테이너 탈출 메커니즘**

```
호스트 OS (Linux Kernel 6.1)
├── PID Namespace (격리된 프로세스)
├── Network Namespace (격리된 네트워크)
├── User Namespace (격리된 사용자)
│   ├── 호스트 UID 0 = 호스트 root
│   └── 컨테이너 UID 0 = 컨테이너 root (하지만 호스트에선?)
├── Mount Namespace (격리된 파일시스템)
│   └── 컨테이너 / = 호스트 /var/lib/docker/containers/.../rootfs/
└── cgroup (리소스 제한)
```

**취약점: User Namespace 매핑 없음**

```
기본 설정 (위험):
┌─────────────────┐     ┌──────────────┐
│ 호스트          │     │ 컨테이너      │
│ UID 0 (root)    │ <-- │ UID 0 (root) │
│ UID 1000 (user) │     │ UID 1000     │
└─────────────────┘     └──────────────┘

컨테이너 탈출 → 호스트 root 권한 획득!

안전한 설정 (userns-remap):
┌──────────────────────┐     ┌──────────────┐
│ 호스트                │     │ 컨테이너      │
│ UID 0 (root)         │     │ UID 0 (root) │
│ UID 100000~109999    │ <-- │ UID ~        │
│ (subuid mapping)     │     │              │
└──────────────────────┘     └──────────────┘

컨테이너 탈출 → 호스트 subuid (100000~109999) 권한만 획득
호스트 root이 아님!
```

### 2. **이미지 레이어의 권한 문제**

```dockerfile
FROM ubuntu:22.04

RUN apt-get install -y openssh-client  # /usr/bin/ssh (파일 소유: root)
COPY app.jar .                          # /app/app.jar (파일 소유: root)
ENTRYPOINT ["java", "-jar", "app.jar"] # 프로세스: root로 실행
```

**파일 권한 확인:**
```bash
$ docker run -it myapp:bad /bin/bash
root@container# ls -la /
drwxr-xr-x  1 root root ...
drwxr-xr-x  1 root root ... app/
-rw-r--r--  1 root root ... app.jar

root@container# cat /etc/passwd | grep root
root:x:0:0:root:/root:/bin/nologin

# 모든 파일과 프로세스가 root 소유
```

**올바른 권한:**
```dockerfile
FROM ubuntu:22.04

RUN groupadd -r appgroup && \
    useradd -r -g appgroup appuser

WORKDIR /app

COPY --chown=appuser:appgroup app.jar .

USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
$ docker run -it myapp:good /bin/bash
appuser@container$ ls -la /
drwxr-xr-x  1 root root ...
drwxr-xr-x  1 appuser appgroup ... app/
-rw-r--r--  1 appuser appgroup ... app.jar

appuser@container$ cat /etc/passwd | grep appuser
appuser:x:1000:1000::/home/appuser:/sbin/nologin

# 애플리케이션 파일은 appuser 소유
```

### 3. **취약점 스캔 원리**

```
Trivy 취약점 스캔:

이미지 분석:
├── 베이스 이미지 식별: ubuntu:22.04 (ubuntu/22.04)
├── 설치된 패키지 목록: openssl, curl, git, ... (dpkg database)
└── 각 패키지 버전: openssl=3.0.2-0ubuntu1

CVE 데이터베이스 검색:
├── openssl 3.0.2: CVE-2022-3602 (높음 심각도)
├── curl 7.81: CVE-2023-27533 (낮음)
└── git 2.34: CVE-2023-22490 (높음)

결과:
HIGH:    3개 CVE (즉시 패치)
MEDIUM:  5개 CVE
LOW:     12개 CVE

권장:
- openssl 3.0.6 이상으로 업그레이드
- curl 7.88 이상으로 업그레이드
```

## 💻 실전 실험

### 실험 1: Root vs Non-root 권한 차이

**Step 1: Root로 실행하는 컨테이너**

```dockerfile
# Dockerfile.root
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y curl

WORKDIR /app

COPY app.sh .

# USER 지정 안 함 = root
ENTRYPOINT ["./app.sh"]
```

```bash
$ docker run myapp:root /bin/id
uid=0(root) gid=0(root) groups=0(root)

# root는 다른 사용자의 파일 읽기 가능
$ docker run myapp:root cat /etc/shadow
root:!:19500:0:99999:7:::
appuser:!:19500:0:99999:7:::
# ✅ 비밀번호 해시 읽음 (보안 위반)

# root는 호스트 디렉토리 마운트 후 접근 가능
$ docker run -v /etc:/host-etc myapp:root cat /host-etc/shadow
# ❌ 호스트 /etc/shadow 접근 가능!
```

**Step 2: Non-root로 실행하는 컨테이너**

```dockerfile
# Dockerfile.nonroot
FROM ubuntu:22.04

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

RUN apt-get update && apt-get install -y curl

WORKDIR /app

RUN chown -R appuser:appgroup /app

COPY --chown=appuser:appgroup app.sh .

USER appuser

ENTRYPOINT ["./app.sh"]
```

```bash
$ docker run myapp:nonroot /bin/id
uid=1000(appuser) gid=1000(appgroup) groups=1000(appgroup)

# appuser는 /etc/shadow 읽기 불가
$ docker run myapp:nonroot cat /etc/shadow
cat: /etc/shadow: Permission denied
# ✅ 접근 거부 (보안 강화)

# appuser는 호스트 파일 접근 불가
$ docker run -v /etc:/host-etc myapp:nonroot cat /host-etc/shadow
cat: /host-etc/shadow: Permission denied
# ✅ 호스트 파일 보호됨
```

### 실험 2: 취약점 스캔 (Trivy)

**Step 1: 취약점 많은 이미지**

```dockerfile
# Dockerfile.vulnerable
FROM ubuntu:22.04  # 기본 버전 사용

RUN apt-get update && apt-get install -y \
    curl \
    wget \
    openssl \
    git \
    openssh-client
```

```bash
# Trivy 설치
$ curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# 이미지 스캔
$ docker build -t vulnerable:latest .
$ trivy image vulnerable:latest

vulnerable:latest (ubuntu 22.04)
====================================
Total: 28 Vulnerabilities

HIGH:    8
MEDIUM:  12
LOW:     8

[HIGH] CVE-2023-3817 (OpenSSL)
  Installed Version: 3.0.2
  Fixed Version: 3.0.7

[HIGH] CVE-2023-22490 (Git)
  Installed Version: 2.34.1
  Fixed Version: 2.41.0
```

**Step 2: 취약점 없는 이미지**

```dockerfile
# Dockerfile.secure
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    curl=7.88-1ubuntu1 \
    openssl=3.0.7-1ubuntu1 \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY --chown=appuser:appgroup app.jar .

USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
$ docker build -t secure:latest .
$ trivy image secure:latest

secure:latest (ubuntu 22.04)
====================================
Total: 0 Vulnerabilities

✅ 모든 취약점 해결!

이미지 크기 비교:
vulnerable:latest   800MB
secure:latest       300MB (62% 감소)
```

### 실험 3: Distroless vs Alpine vs Standard

```bash
# 이미지 빌드
$ docker build -t myapp:standard -f Dockerfile.standard .
$ docker build -t myapp:alpine -f Dockerfile.alpine .
$ docker build -t myapp:distroless -f Dockerfile.distroless .

# 크기 비교
$ docker images | grep myapp
myapp    standard      800MB
myapp    alpine        120MB
myapp    distroless    80MB

# 취약점 스캔
$ trivy image myapp:standard
Total: 45 Vulnerabilities (HIGH: 8)

$ trivy image myapp:alpine
Total: 2 Vulnerabilities (HIGH: 0)

$ trivy image myapp:distroless
Total: 0 Vulnerabilities
```

**각 이미지에서 쉘 접근 가능 여부:**

```bash
# Standard 이미지
$ docker run -it myapp:standard /bin/bash
root@container# whoami
root
# ✅ 쉘 접근 가능, root 권한

# Alpine 이미지
$ docker run -it myapp:alpine /bin/sh
appuser@container# whoami
appuser
# ✅ 쉘 접근 가능, appuser 권한

# Distroless 이미지
$ docker run -it myapp:distroless /bin/bash
docker: Error response from daemon: OCI runtime create failed: container_linux.go:380: 
  starting container process caused: exec: "/bin/bash": stat /bin/bash: no such file or directory

# ❌ 쉘이 없음 (보안 강화)
```

### 실험 4: GitHub Actions에서 Trivy 자동 스캔

```yaml
# .github/workflows/scan.yml
name: Security Scan
on: push

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: docker/setup-buildx-action@v2
      
      - uses: docker/build-push-action@v4
        with:
          tags: myapp:latest
          push: false  # 로컬에만 저장
          load: true   # Docker에 로드
      
      # ✅ Trivy 스캔
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:latest
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      # ✅ 취약점 결과 업로드 (GitHub Security 탭에 표시)
      - uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      # ✅ HIGH 이상 취약점 발견 시 파이프라인 중단
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:latest
          exit-code: '1'  # 취약점 발견 시 1 반환
          severity: 'HIGH,CRITICAL'  # HIGH 이상만 검사
```

## 📊 성능/비용 비교

### 이미지 크기 비교

| 베이스 | 패키지 | Non-root | 크기 | 취약점 |
|-------|-------|---------|------|-------|
| ubuntu:22.04 | 모두 | No | 800MB | 45개 |
| ubuntu:22.04 | 최소 | Yes | 300MB | 5개 |
| alpine:3.18 | 필수 | Yes | 120MB | 2개 |
| distroless | 최소 | Yes | 80MB | 0개 |

### 취약점 수정 비용

```
High/Critical 취약점 발견 시:

Standard 이미지 (취약점 45개):
- 패치 검증: 2시간
- 재빌드/배포: 1시간
- 테스트: 3시간
- 총 비용: 6시간/취약점 = 270시간/월 (주 1회 발견)

Minimal 이미지 (취약점 5개):
- 패치 검증: 30분
- 재빌드/배포: 30분
- 테스트: 1시간
- 총 비용: 2시간/취약점 = 10시간/월 (월 1회)

절감: 260시간/월 (약 $5,200/월, 개발자 월급 기준)
```

### 배포 속도

```
네트워크 속도: 10Mbps

Standard 이미지 (800MB):
- 푸시: 640초
- 풀: 640초
- 언팩: 30초
- 총: 1,310초 (21분)

Distroless 이미지 (80MB):
- 푸시: 64초
- 풀: 64초
- 언팩: 3초
- 총: 131초 (2분)

개선: 90% 빨라짐
```

## ⚖️ 트레이드오프

### 1. **보안 vs 디버깅 용이성**

```dockerfile
# ❌ 디버깅 쉬우나 보안 약함
FROM ubuntu:22.04
RUN apt-get install -y curl wget vim
ENTRYPOINT ["java", "-jar", "app.jar"]

# 장점:
# - 문제 발생 시 /bin/bash로 접근해 로그 확인
# - docker exec로 디버깅 가능

# 단점:
# - 공격 면적 크다
# - 취약점 많다


# ✅ 보안 강하나 디버깅 어려움
FROM distroless/java17-nonroot
COPY --chown=65532:65532 app.jar .
ENTRYPOINT ["java", "-jar", "app.jar"]

# 장점:
# - 쉘 접근 불가 (공격자 도구 설치 못 함)
# - 취약점 0개

# 단점:
# - docker exec로 접근 불가
# - 컨테이너 내 로그 확인 어려움
```

**해결: 멀티 스테이지 + 선택적 빌드**
```dockerfile
FROM ubuntu:22.04 AS debug-image
RUN apt-get install -y curl wget vim

FROM distroless/java17-nonroot
COPY app.jar .

# 프로덕션: distroless로 배포
# 디버깅: docker build --target debug-image로 디버그 이미지 빌드
```

### 2. **최소 이미지 vs 호환성**

```dockerfile
# ❌ Distroless (최소, 호환성 이슈 가능)
FROM gcr.io/distroless/java17-nonroot

# 포함: Java 런타임, libc
# 제외: curl, wget, openssl CLI, /bin/sh

# 문제: 일부 Java 라이브러리가 쉘 스크립트 실행 필요
Runtime.getRuntime().exec("bash -c 'command'");  // ❌ /bin/sh 없음!


# ✅ Alpine (작지만 호환성 좋음)
FROM eclipse-temurin:17-jre-alpine

# 포함: Java 런타임, /bin/sh, apk 패키지 매니저
# 제외: 불필요한 도구

# 호환성: 대부분의 자바 애플리케이션 동작
```

**선택 기준:**
- Distroless: 자바 애플리케이션만 실행 (권장)
- Alpine: 초기화 스크립트나 쉘 필요한 경우
- Debian Slim: 특수한 라이브러리 필요한 경우

## 📌 핵심 정리

1. **Root 사용자는 컨테이너 탈출 시 호스트 root 권한 획득 가능**
   - 항상 non-root 사용자 사용 필수
   - `USER appuser` 명령어로 전환

2. **불필요한 패키지 제거는 취약점 감소 + 이미지 크기 축소**
   - Standard (800MB, 45개 취약점) → Distroless (80MB, 0개)
   - 월간 $5,200+ 비용 절감 (취약점 패치 비용)

3. **Trivy로 CI 파이프라인에 자동 취약점 스캔 통합**
   - GitHub Actions: `aquasecurity/trivy-action`
   - HIGH/CRITICAL 취약점 발견 시 배포 중단

4. **Distroless vs Alpine vs Standard 선택**
   - Distroless: 최고 보안 (쉘 없음), 자바 전용
   - Alpine: 절충안 (작음, 호환성 좋음)
   - Standard: 호환성 최고 (보안 약함)

5. **다중 스테이지로 프로덕션과 디버그 이미지 분리**
   - 프로덕션: distroless (보안 강)
   - 디버그: 도구 포함 (디버깅 용이)

## 🤔 생각해볼 문제

### 문제 1: 다음 Dockerfile의 보안 문제를 찾으세요.

```dockerfile
FROM openjdk:17-jre-slim

WORKDIR /app

COPY app.jar .

# 파일 소유권을 appuser로 변경했으나...
RUN useradd -r appuser && \
    chown appuser:appuser app.jar

# USER를 마지막에 지정
USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

<details>
<summary>해설</summary>

**문제점:**
1. appuser가 /app 디렉토리 소유권이 없음 (root 소유)
   - appuser는 /app 디렉토리 접근 불가능 (x 권한 없음)
   - app.jar는 읽을 수 있어도 다른 파일은 불가

2. /app 디렉토리 소유권이 root (755 권한)
   - appuser는 /app 하위에 파일 쓰기 불가

**올바른 방식:**
```dockerfile
FROM openjdk:17-jre-slim

RUN groupadd -r appgroup && \
    useradd -r -g appgroup appuser

WORKDIR /app

# COPY 후 소유권 변경
COPY --chown=appuser:appgroup app.jar .

USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**검증:**
```bash
$ docker run -it myapp /bin/bash
appuser@app$ ls -la /app
drwxr-xr-x 1 appuser appgroup ... /app  # ✅ appuser 소유
-rw-r--r-- 1 appuser appgroup ... app.jar
```
</details>

### 문제 2: Distroless 이미지에서 로깅이 작동하지 않습니다. 원인은?

```dockerfile
FROM gcr.io/distroless/java17-nonroot

COPY app.jar .

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
$ docker logs myapp
# 아무것도 출력 안 됨
```

<details>
<summary>해설</summary>

**원인:**
- Distroless에는 `/bin/sh`가 없어 stdout/stderr 리다이렉션 불가
- Java 애플리케이션이 파일 로깅을 시도하면 권한 부족

**해결 방법 1: Java 로깅 설정**
```dockerfile
FROM gcr.io/distroless/java17-nonroot

COPY app.jar .

# Java 시스템 속성으로 로깅 설정
ENTRYPOINT [
  "java",
  "-Djava.util.logging.config.file=/etc/logging.properties",
  "-jar", "app.jar"
]
```

**해결 방법 2: 로그 파일 위치 변경**
```dockerfile
FROM gcr.io/distroless/java17-nonroot

COPY app.jar .

# 로그를 stdout으로 출력 (Spring Boot 기본)
ENV SPRING_OUTPUT_ANSI_ENABLED=always

# logs 디렉토리에 쓰기 권한 설정
RUN mkdir -p /logs && chmod 777 /logs

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**해결 방법 3: Alpine으로 변경**
```dockerfile
FROM eclipse-temurin:17-jre-alpine

COPY app.jar .

ENTRYPOINT ["java", "-jar", "app.jar"]
# 쉘 사용 가능하고 로깅도 정상 작동
```
</details>

### 문제 3: 사용자 네임스페이스(userns-remap) 설정이 왜 권장되는가?

<details>
<summary>해설</summary>

**User Namespace Remapping (userns-remap):**

기본 설정:
```
호스트 UID 0 = 컨테이너 UID 0
호스트 UID N = 컨테이너 UID N

컨테이너 root가 호스트 root!
```

userns-remap 설정:
```
/etc/docker/daemon.json:
{
  "userns-remap": "dockeruser"
}

호스트:
- dockeruser UID 165536~165999 (subuid 할당)

컨테이너 UID 0 = 호스트 UID 165536 (dockeruser)
컨테이너 UID 1 = 호스트 UID 165537
```

**효과:**
1. 컨테이너 탈출 → 호스트의 subuid (165536~165999) 권한만 획득
2. 호스트 root 파일 접근 불가
3. 다른 사용자 파일 접근 불가

**설정 방법:**
```bash
# dockeruser 생성
useradd -r -s /sbin/nologin dockeruser

# /etc/subuid, /etc/subgid에 자동 등록
# 또는 수동으로:
echo "dockeruser:165536:65536" >> /etc/subuid
echo "dockeruser:165536:65536" >> /etc/subgid

# Docker 데몬 설정
cat > /etc/docker/daemon.json <<EOF
{
  "userns-remap": "dockeruser"
}
EOF

systemctl restart docker
```

**주의:**
- 기존 컨테이너와 데이터 호환되지 않음 (파일 권한 변경 필요)
- 성능 약간 저하 (권한 확인)
</details>

---

<div align="center">

**[⬅️ 이전: BuildKit 내부 동작](./03-buildkit-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Boot 최적화 이미지 ➡️](./05-spring-boot-image.md)**

</div>
