# Spring Boot 최적화 이미지 — Layered JAR와 Buildpacks

## 🎯 핵심 질문

- Spring Boot 애플리케이션의 JAR 파일 내부는 어떻게 구성되어 있을까?
- 일반 멀티 스테이지 빌드와 Layered JAR 방식의 차이는?
- Cloud Native Buildpacks가 제공하는 장점은 정확히 무엇인가?

## 🔍 왜 이 개념이 실무에서 중요한가

**Spring Boot 빌드 최적화의 핵심:**

1. **JAR 파일의 계층화 (Layering)**
   - 일반 JAR: 모든 라이브러리 + 애플리케이션 코드 섞여 있음
   - Layered JAR: 의존성 / 스냅샷 의존성 / 애플리케이션 코드 분리
   - 결과: 소스 코드 변경 시 라이브러리 레이어 재빌드 불필요

2. **Cloud Native Buildpacks 자동화**
   - 매번 Dockerfile 수작업 작성 필수 (위험, 휴먼 에러)
   - Buildpacks: `./gradlew bootBuildImage` 한 줄로 최적화된 이미지 생성
   - Spring Boot 팀이 권장 (공식 표준)

3. **JVM 메모리 설정의 중대한 문제**
   - 구 버전 JVM: 컨테이너 cgroup 메모리 제한을 인식 못함
   - 결과: cgroup 메모리 초과 → OOMKilled (서비스 중단)
   - 해결: JVM 메모리 옵션 수동 설정 또는 Java 11+

4. **Native Image로 시작 시간 대폭 단축**
   - 일반 JAR: 시작 5초 (JVM 부팅, 클래스 로딩)
   - Native Image: 시작 0.1초 (사전 컴파일)
   - 서버리스 환경 필수 (AWS Lambda, Google Cloud Run)

**실무 사례:**
- Walmart: Buildpacks 도입으로 개발자 생산성 30% 향상
- Google Cloud: Spring Boot 3.x Native Image로 Cold Start 99% 단축

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

### 문제 1: 일반 JAR 파일의 문제점

```dockerfile
# ❌ 나쁜 Dockerfile: Layered JAR 미사용
FROM openjdk:17-jre-slim

WORKDIR /app

COPY build/libs/spring-boot-app.jar .

ENTRYPOINT ["java", "-jar", "spring-boot-app.jar"]
```

**JAR 파일 구조 (일반):**
```
spring-boot-app.jar
├── BOOT-INF/
│   ├── classes/  (2MB, 애플리케이션 코드)
│   └── lib/
│       ├── spring-core-6.0.jar (150MB, 라이브러리)
│       ├── spring-web-6.0.jar
│       ├── spring-data-*.jar
│       └── ... (100+ JAR 파일)
├── META-INF/
│   ├── MANIFEST.MF
│   └── spring.factories
└── org/ (JVM 부트로더)
```

**빌드할 때마다:**
```
코드 1줄 수정 → JAR 재생성 (50MB)
├── COPY build/libs/spring-boot-app.jar . (캐시 무효화)
└── 전체 150MB 재빌드 (라이브러리 포함)

문제: 라이브러리는 변경 없어도 매번 같은 150MB 재다운로드
```

### 문제 2: JVM 메모리 설정 부재

```dockerfile
# ❌ 나쁜 Dockerfile: JVM 메모리 설정 안 함
FROM openjdk:17-jre-slim

WORKDIR /app

COPY app.jar .

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**실제 Kubernetes 배포:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-app
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      limits:
        memory: "512Mi"  # 메모리 제한
```

**문제 발생:**
```
컨테이너 메모리 제한: 512MB
JVM 힙 크기: 제한 인식 못함 → Java 8/11 기본값 (최대 512MB)

실제 할당:
- JVM 메타스페이스: 100MB
- JVM 스택: 50MB
- 애플리케이션 힙: 제한 없음 (462MB 시도)

결과:
JVM 시작: 50MB (OK)
애플리케이션 로딩: 150MB (OK)
런타임 메모리 누적: 512MB 초과 (OOMKilled!)

증상: Kubernetes에서 주기적으로 Pod 재시작
```

### 문제 3: Buildpacks 미사용

```bash
# ❌ 나쁜 방식: 매번 Dockerfile 수작업 작성
$ docker build -t myapp:v1 .
$ docker build -t myapp:v2 .
# 각 버전마다 다른 최적화 적용될 수 있음 (일관성 부족)

# Spring Boot 팀이 권장하는 방식을 모르면:
# → 매번 Dockerfile 수정 필요
# → 최적화 누락 위험
# → 개발자 휴먼 에러
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 방식 1: Layered JAR (공식 권장)

**Step 1: Spring Boot에서 Layered JAR 생성**

```gradle
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.1.0'
}

jar {
    enabled = false  // 일반 JAR 비활성화
}

tasks.named('bootJar') {
    layered {
        enabled = true  // Layered JAR 활성화
    }
}
```

```bash
$ ./gradlew clean build

# 결과:
# build/libs/spring-boot-demo.jar (Layered 구조)
```

**Layered JAR 구조:**
```
spring-boot-demo.jar (Layered)
├── Layer 1: dependencies/ (140MB, 변경 거의 없음)
│   ├── spring-core-6.0.jar
│   ├── spring-web-6.0.jar
│   └── ... (모든 라이브러리)
├── Layer 2: spring-boot-loader/ (1MB, 거의 변경 없음)
│   └── org/springframework/boot/loader/...
├── Layer 3: snapshot-dependencies/ (변경 자주)
│   └── dev-tools, spring-boot-devtools
└── Layer 4: application/ (2MB, 매번 변경)
    ├── BOOT-INF/classes/... (애플리케이션 코드)
    └── META-INF/...
```

**Step 2: Layered JAR 추출하는 Dockerfile**

```dockerfile
# ✅ Stage 1: Builder (Layered JAR 추출)
FROM openjdk:17-jdk AS builder

WORKDIR /build

COPY . .

RUN ./gradlew clean build -x test

# Layered JAR에서 각 계층 추출
# 이 명령어는 Spring Boot 2.3+에서만 작동
RUN java -Djarmode=layertools -jar build/libs/spring-boot-demo.jar extract \
    --destination extracted

# extracted/ 디렉토리 구조:
# extracted/
# ├── dependencies/        (Layer 1)
# ├── spring-boot-loader/  (Layer 2)
# ├── snapshot-dependencies/ (Layer 3)
# └── application/         (Layer 4)


# ✅ Stage 2: Runtime (최적화된 레이어 구성)
FROM openjdk:17-jre-slim

WORKDIR /app

# Layer 1: 의존성 (변경 거의 없음, 캐시 활용)
COPY --from=builder /build/extracted/dependencies/ ./

# Layer 2: Spring Boot 로더 (거의 변경 없음)
COPY --from=builder /build/extracted/spring-boot-loader/ ./

# Layer 3: 스냅샷 의존성 (가끔 변경)
COPY --from=builder /build/extracted/snapshot-dependencies/ ./

# Layer 4: 애플리케이션 (자주 변경)
COPY --from=builder /build/extracted/application/ ./

# ✅ JVM 메모리 설정 (중요!)
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

**빌드 효과:**
```
첫 빌드: 140MB (의존성) + 1MB (로더) + 2MB (앱) = 143MB
코드 수정 후:
  - Layer 1 (의존성): 캐시 히트 (0초)
  - Layer 2 (로더): 캐시 히트 (0초)
  - Layer 3 (스냅샷): 캐시 히트 (0초)
  - Layer 4 (애플리케이션): 재빌드 (2초)
  
  총 빌드 시간: 5초 (기존 50초 → 90% 단축!)
```

### 방식 2: Cloud Native Buildpacks (가장 권장)

**Step 1: Maven 또는 Gradle 설정**

```gradle
// build.gradle (Gradle)
plugins {
    id 'org.springframework.boot' version '3.1.0'
}

springBoot {
    buildpack {
        publish = true
    }
}
```

```xml
<!-- pom.xml (Maven) -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>3.1.0</version>
</plugin>
```

**Step 2: 한 줄의 명령어로 최적화된 이미지 생성**

```bash
# Gradle
$ ./gradlew bootBuildImage --imageName=myapp:latest

# Maven
$ mvn spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest

# 결과:
# ✅ 자동으로 Layered JAR 구조로 최적화
# ✅ JVM 메모리 자동 설정
# ✅ 보안 베스트 프랙티스 적용 (non-root)
# ✅ 성능 최적화 (병렬 레이어)
```

**Buildpack이 자동으로 해주는 것:**
```
1. JVM 탐지 및 선택
2. Layered JAR 자동 생성 및 추출
3. JVM 메모리 옵션 자동 설정
4. Non-root 사용자 자동 생성
5. 캐시 친화적 레이어 구성
6. 보안 스캔 (기본 포함)
```

### 방식 3: Native Image (최고 성능)

```gradle
// build.gradle (Gradle)
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.1.0'
    id 'org.graalvm.buildtools.native' version '0.9.18'
}
```

```bash
# Native Image 빌드 (GraalVM 필요)
$ ./gradlew nativeCompile

# 또는 Buildpacks로 Native Image 생성
$ ./gradlew bootBuildImage --imageName=myapp:native

# 결과:
# - 이미지 크기: 200MB → 100MB (50% 감소)
# - 시작 시간: 5초 → 0.1초 (50배 빠름!)
```

## 🔬 내부 동작 원리

### 1. **Layered JAR의 계층 구조**

```
일반 JAR (Fat JAR):
┌─────────────────────────────────────┐
│ 모든 의존성 + 로더 + 애플리케이션    │
│ (150MB 전체가 하나의 아티팩트)       │
└─────────────────────────────────────┘
코드 변경 → 전체 150MB 재빌드


Layered JAR:
┌─────────────────────────────────┐
│ Layer 4: application (2MB)      │ ← 자주 변경, 캐시 최소화
├─────────────────────────────────┤
│ Layer 3: snapshot-dependencies  │ ← 가끔 변경
├─────────────────────────────────┤
│ Layer 2: spring-boot-loader(1MB)│ ← 거의 변경 안 함
├─────────────────────────────────┤
│ Layer 1: dependencies (140MB)   │ ← 변경 거의 없음, 캐시 활용
└─────────────────────────────────┘
코드 변경 → Layer 4만 2MB 재빌드 (캐시: Layer 1-3)
```

**JAR 분석 도구:**
```bash
$ jar tf build/libs/spring-boot-demo.jar | head -20

# Layered JAR 확인
$ unzip -l build/libs/spring-boot-demo.jar | grep "^-" | head -5

# Spring Boot 로더 확인
$ unzip -l build/libs/spring-boot-demo.jar | grep "org/springframework/boot/loader"
Archive:  build/libs/spring-boot-demo.jar
  Length      Date    Time    Name
---------  ---------- -----   ----
     1234  2024-01-01 12:00   org/springframework/boot/loader/JarLauncher.class
      ...
```

### 2. **Layered JAR 추출 메커니즘**

```bash
# Layered JAR 추출
$ java -Djarmode=layertools -jar app.jar extract --destination extracted

# 이 명령어의 작동 원리:
# 1. JAR 파일 읽기 (ZIP 형식)
# 2. BOOT-INF/layers.idx 파일 파싱
#    └─ 각 계층 정의: dependencies, snapshot-dependencies, etc.
# 3. 각 계층을 별도 디렉토리로 추출
# 4. Dockerfile COPY 명령어로 각 계층을 레이어로 구성

# layers.idx 내용 예:
# dependencies:
#   - "BOOT-INF/lib/"
# spring-boot-loader:
#   - "org/"
# snapshot-dependencies:
#   - "BOOT-INF/lib/*-SNAPSHOT.jar"
# application:
#   - "BOOT-INF/classes/"
#   - "META-INF/"
```

### 3. **JVM 메모리 설정과 cgroup**

**문제: JVM이 cgroup 제한을 모름 (Java 8, 구 Java 11)**

```dockerfile
# ❌ 문제 있는 설정
FROM openjdk:11-jre

ENTRYPOINT ["java", "-jar", "app.jar"]
# JVM이 자동 감지할 때:
# - 호스트 전체 메모리 검색 (16GB)
# - 힙 크기를 16GB의 25% = 4GB로 설정
# - 컨테이너 제한(512MB) 무시
# → OOMKilled!
```

**해결 방법 1: JVM 옵션 명시**

```dockerfile
# ✅ 해결: JVM 옵션 명시
FROM openjdk:17-jre-slim

ENTRYPOINT [
  "java",
  "-XX:MaxRAMPercentage=75.0",    # 컨테이너 메모리의 75%를 힙으로 사용
  "-XX:InitialRAMPercentage=50.0", # 초기 힙 50%로 시작 (GC 효율)
  "-XX:+UseG1GC",                 # G1 가비지 컬렉터 사용
  "-jar", "app.jar"
]
```

**동작:**
```
Kubernetes 메모리 제한: 512MB

JVM 메모리 설정:
MaxRAMPercentage=75.0  → 힙 크기 = 512MB × 75% = 384MB
InitialRAMPercentage=50.0 → 초기 힙 = 512MB × 50% = 256MB

메모리 사용:
- 초기: 256MB (Initialize)
- 런타임: 필요시 384MB까지 확장
- 절대 512MB 초과 안 함 (안전함!)
```

**해결 방법 2: Java 17+ (자동 감지)**

```bash
# Java 17+에서는 자동 감지됨
FROM openjdk:17-jre-slim
# cgroup 제한 자동 감지 + 메모리 자동 설정
```

### 4. **Cloud Native Buildpacks의 최적화**

```
Buildpack 실행 순서:

1. Detect (감지)
   ├─ build.gradle 또는 pom.xml 확인
   └─ "Java 애플리케이션" 확인

2. Restore (이전 캐시 복원)
   ├─ 이전 빌드의 캐시 로드
   └─ Maven/Gradle 의존성 캐시 복원

3. Build (빌드)
   ├─ Maven/Gradle 컴파일
   ├─ 자동으로 Layered JAR 생성
   └─ 각 계층 추출

4. Export (이미지 생성)
   ├─ 각 계층을 Docker 레이어로 매핑
   ├─ JVM 옵션 자동 설정
   ├─ Non-root 사용자 설정
   └─ 최종 이미지 생성

결과:
- 개발자는 Dockerfile 작성 안 함 (버그 위험 없음)
- Spring Boot 팀이 권장하는 최적화 자동 적용
- 모든 프로젝트가 동일한 표준 준수
```

## 💻 실전 실험

### 실험 1: Layered JAR vs 일반 JAR 빌드 비교

**Step 1: 일반 JAR (Layered 미사용)**

```gradle
// build.gradle - Layered 비활성화
jar {
    enabled = true  // 일반 JAR 생성
}

tasks.named('bootJar') {
    layered {
        enabled = false  // Layered 비활성화
    }
}
```

```dockerfile
# Dockerfile.normal (일반 JAR)
FROM openjdk:17-jdk AS builder
WORKDIR /build
COPY . .
RUN ./gradlew clean build -x test

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=builder /build/build/libs/spring-boot-app.jar .
ENTRYPOINT ["java", "-jar", "spring-boot-app.jar"]
```

```bash
$ docker build -f Dockerfile.normal -t myapp:normal .
[+] Building 145.3s (9/9) FINISHED

$ docker images myapp:normal
myapp    normal    abc1234567890    180MB

# 두 번째 빌드 (코드만 수정)
$ docker build -f Dockerfile.normal -t myapp:normal .
[+] Building 52.3s (9/9) FINISHED  # 여전히 오래 걸림
# 이유: COPY --from=builder 전체 JAR 재빌드
```

**Step 2: Layered JAR**

```gradle
// build.gradle - Layered 활성화
jar {
    enabled = false  // 일반 JAR 비활성화
}

tasks.named('bootJar') {
    layered {
        enabled = true  // Layered JAR 활성화
    }
}
```

```dockerfile
# Dockerfile.layered (Layered JAR)
FROM openjdk:17-jdk AS builder
WORKDIR /build
COPY . .
RUN ./gradlew clean build -x test

# Layered JAR 추출
RUN java -Djarmode=layertools -jar build/libs/spring-boot-app.jar extract \
    --destination extracted

FROM openjdk:17-jre-slim
WORKDIR /app

# 각 계층을 별도 레이어로 복사
COPY --from=builder /build/extracted/dependencies/ ./
COPY --from=builder /build/extracted/spring-boot-loader/ ./
COPY --from=builder /build/extracted/snapshot-dependencies/ ./
COPY --from=builder /build/extracted/application/ ./

ENTRYPOINT ["sh", "-c", "java org.springframework.boot.loader.JarLauncher"]
```

```bash
$ docker build -f Dockerfile.layered -t myapp:layered .
[+] Building 150.2s (13/13) FINISHED

$ docker images myapp:layered
myapp    layered   xyz9876543210    180MB  # 크기 동일

$ docker history myapp:layered
# 계층 확인:
# Layer 1: dependencies (140MB)
# Layer 2: spring-boot-loader (1MB)
# Layer 3: snapshot-dependencies (0MB)
# Layer 4: application (2MB)

# 두 번째 빌드 (코드만 수정)
$ docker build -f Dockerfile.layered -t myapp:layered .
[+] Building 8.3s (13/13) FINISHED  # 84% 빨라짐!
# 이유: Layer 1-3은 캐시 히트, Layer 4만 재빌드
```

**빌드 시간 비교:**

| 상황 | 일반 JAR | Layered JAR | 개선 |
|-----|---------|-----------|------|
| 첫 빌드 | 145초 | 150초 | -3% |
| 코드 수정 | 52초 | 8초 | 85% |
| 의존성 변경 | 145초 | 145초 | 0% |

### 실험 2: Cloud Native Buildpacks

```bash
# Step 1: Buildpack으로 이미지 빌드
$ ./gradlew bootBuildImage --imageName=myapp:buildpack

# 자동으로 수행되는 작업:
# 1. Layered JAR 생성
# 2. 각 계층 추출
# 3. 각 계층을 Docker 레이어로 매핑
# 4. JVM 옵션 자동 설정
# 5. Non-root 사용자 설정 (uid=1000)

$ docker images myapp:buildpack
myapp    buildpack    pqr1234567890    180MB

$ docker history myapp:buildpack
# Buildpack이 자동으로 생성한 레이어:
# ├─ Base OS layer (100MB)
# ├─ dependencies (140MB)
# ├─ loader (1MB)
# └─ application (2MB)

# Step 2: 실행
$ docker run myapp:buildpack
# 자동으로 설정된 JVM 옵션:
# - MaxRAMPercentage=75.0
# - InitialRAMPercentage=50.0
# - Non-root 사용자로 실행
```

### 실험 3: Native Image (GraalVM)

```bash
# Step 1: Native Image 빌드 (시간 많이 걸림)
$ ./gradlew nativeCompile
# 약 3-5분 소요 (애플리케이션 사전 컴파일)

# 또는 Buildpack으로
$ ./gradlew bootBuildImage --imageName=myapp:native --build-argument BPE_SPRING_AOT=true

# Step 2: 이미지 확인
$ docker images | grep myapp
myapp    native    uvw5678901234    100MB  # 180MB → 100MB (44% 감소!)

# Step 3: 실행 및 성능 비교
$ time docker run myapp:buildpack
# Spring Boot 앱 시작 시간: 약 3-5초

$ time docker run myapp:native
# Native Image 시작 시간: 약 0.1초 (50배 빠름!)

# 메모리 사용:
$ docker run myapp:buildpack
# 힙 메모리: 약 200MB

$ docker run myapp:native
# 힙 메모리: 약 50MB (4배 적음!)
```

## 📊 성능/비용 비교

### 빌드 시간 비교

| 시나리오 | 일반 JAR | Layered JAR | Buildpack |
|--------|---------|-----------|----------|
| 첫 빌드 | 120초 | 125초 | 130초 |
| 코드 수정 | 50초 | 8초 | 8초 |
| 의존성 변경 | 120초 | 125초 | 130초 |

**월간 빌드 시간 (개발자 1명, 하루 10회):**
```
일반 JAR: 10회 × 50초 = 500초 = 8분/일 × 20일 = 160분/월
Layered JAR: 5회 × 50초 + 5회 × 8초 = 290초 = 5분/일 × 20일 = 100분/월
Buildpack: 5회 × 130초 + 5회 × 8초 = 690초 = 11분/일 × 20일 = 220분/월 (초기)

이후:
Layered JAR: 100분/월
Buildpack: 50분/월 (캐시 풀 활용 후)

절감: 160 → 100분/월 (37% 단축)
```

### 런타임 성능

| 지표 | 일반 JAR | Native Image | 개선 |
|-----|---------|------------|------|
| 시작 시간 | 5초 | 0.1초 | 50배 |
| 초기 메모리 | 300MB | 50MB | 6배 |
| 이미지 크기 | 180MB | 100MB | 44% |
| 최대 메모리 | 1GB | 100MB | 10배 |

### 서버리스 환경 비용 절감 (AWS Lambda)

```
Cold Start (첫 호출):
- 일반 JAR: 5초 초기화 + 5초 실행 = 10초
- Native Image: 0.1초 초기화 + 5초 실행 = 5.1초
- 절감: 4.9초 × $0.0000166667/초 = $0.00008/콜드스타트

월간 1,000회 콜드스타트:
- 일반 JAR: 1,000 × $0.00008 = $0.08/월
- Native Image: 1,000 × $0.00004 = $0.04/월
- 절감: $0.04/월 (50% 감소)
```

## ⚖️ 트레이드오프

### 1. **Layered JAR vs Buildpack vs Native Image**

```dockerfile
# Layered JAR
장점:
- 개발자가 최적화 제어 가능
- Dockerfile 커스터마이징 자유도 높음
단점:
- Dockerfile 작성/관리 필요
- 휴먼 에러 가능성

# Buildpack
장점:
- 자동화 (개발자는 한 줄 명령어만 실행)
- Spring Boot 팀의 공식 권장
- 모든 프로젝트 일관된 최적화
단점:
- Dockerfile 커스터마이징 어려움
- 캐시 초기 설정 필요

# Native Image
장점:
- 최고의 성능 (50배 빠른 시작)
- 최소 이미지 크기
- 서버리스 환경 최적
단점:
- 빌드 시간 길다 (3-5분)
- 메모리 제한 심함 (일부 라이브러리 호환 이슈)
- GraalVM 별도 설치 필요
```

### 2. **JVM 메모리 설정의 트레이드오프**

```dockerfile
# 보수적 설정 (안전하나 성능 저하)
ENV JAVA_OPTS="-XX:MaxRAMPercentage=50.0"
# 힙 사용: 50% (512MB 제한 = 256MB 힙)
# 장점: 안전함 (OOMKilled 위험 적음)
# 단점: 힙이 작아 Full GC 자주 발생 (성능 저하)

# 적극적 설정 (성능 좋으나 위험)
ENV JAVA_OPTS="-XX:MaxRAMPercentage=90.0"
# 힙 사용: 90% (512MB 제한 = 460MB 힙)
# 장점: 힙이 커서 성능 좋음
# 단점: 메타스페이스/스택 초과 위험 (OOMKilled)

# 권장 설정 (균형)
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
# 초기: 375MB, 최대: 460MB (안전과 성능 균형)
```

## 📌 핵심 정리

1. **Layered JAR는 의존성/스냅샷/애플리케이션을 분리**
   - 코드 변경 시 2MB만 재빌드 (기존 50MB → 85% 단축)
   - Spring Boot 2.3+에서 자동 지원

2. **Cloud Native Buildpacks은 최고 생산성 (권장)**
   - `./gradlew bootBuildImage` 한 줄로 최적화된 이미지 생성
   - Dockerfile 작성 불필요
   - Spring Boot 팀 공식 권장

3. **JVM 메모리 옵션 필수 설정**
   - `-XX:MaxRAMPercentage=75.0` 필수
   - 컨테이너 OOMKilled 방지

4. **Native Image는 최고 성능**
   - 시작 시간 50배 빠름 (5초 → 0.1초)
   - 메모리 사용 90% 감소
   - 서버리스 환경 필수

5. **Layered JAR + JVM 옵션 = 안정성과 성능의 최적 조합**
   - 일반 운영 환경 권장
   - 개발자가 이해하고 제어 가능

## 🤔 생각해볼 문제

### 문제 1: Spring Boot Layered JAR에서 `layers.idx` 파일의 역할은?

<details>
<summary>해설</summary>

**layers.idx 파일:**

Spring Boot JAR 내부에 포함된 메타데이터 파일:
```
BOOT-INF/layers.idx
```

**내용 (예시):**
```
dependencies:
  - BOOT-INF/lib/spring-core-6.0.0.jar
  - BOOT-INF/lib/spring-web-6.0.0.jar
  - BOOT-INF/lib/... (모든 라이브러리)

spring-boot-loader:
  - org/springframework/boot/loader/...

snapshot-dependencies:
  - BOOT-INF/lib/*-SNAPSHOT.jar

application:
  - BOOT-INF/classes/
  - META-INF/
```

**역할:**
1. `java -Djarmode=layertools` 명령어가 읽음
2. 각 계층(Layer)이 어떤 파일을 포함하는지 정의
3. 추출 시 파일들을 올바른 디렉토리로 분류

**설정 (gradle):**
```gradle
tasks.named('bootJar') {
    layered {
        enabled = true
        
        // 커스텀 계층 정의 가능
        layer 'my-dependencies' {
            include 'org.springframework:spring-core'
        }
    }
}
```

**없으면?**
- Layered JAR 추출 불가
- 모든 파일이 하나의 디렉토리로 추출됨
- 캐시 효율 저하
</details>

### 문제 2: Native Image에서 Reflection 사용 불가능한 이유는?

<details>
<summary>해설</summary>

**Native Image의 작동 원리:**

```
Spring Boot JAR (런타임 컴파일):
1. JVM이 클래스 로드
2. 런타임에 리플렉션 사용 (Class.forName, getMethod 등)
3. 동적으로 메서드 호출

GraalVM Native Image (사전 컴파일):
1. AOT (Ahead-of-Time) 컴파일: 빌드 시 모든 코드 분석
2. 사용되는 클래스/메서드 미리 파악
3. 미사용 코드 제거 (크기 축소)
4. 리플렉션으로 찾을 수 없는 메서드 = 제거됨!
```

**예시:**

```java
// Spring Data JPA 리포지토리
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email);
}

// Native Image 빌드 시:
// 1. UserRepository 인터페이스 분석
// 2. JpaRepository.findByEmail() 메서드 미존재 (Spring이 동적 생성)
// 3. GraalVM이 미사용 코드 판단 → 제거
// 4. 런타임: findByEmail() 메서드 미존재 (에러!)

// 해결: Spring Boot 3.x의 AOT (Ahead-of-Time) 지원
// Spring이 빌드 시 리플렉션 코드 생성 (GraalVM 사전 분석)
```

**Spring Boot 3.x 해결:**
```gradle
plugins {
    id 'org.graalvm.buildtools.native' version '0.9.18'
}

tasks.named('nativeCompile') {
    options {
        // Spring AOT 활성화
        buildArgs.add('--enable-preview')
    }
}
```

**실제 Spring이 생성하는 AOT 코드:**
```java
// 빌드 시 자동 생성
@Configuration
public class UserRepositoryAotConfiguration {
    @Bean
    public UserRepository userRepository() {
        return // 리플렉션 없이 직접 생성
    }
}
```

**결론:**
- Spring Boot 3.x: Native Image 완벽 지원 (AOT)
- Spring Boot 2.x: Native Image 부분 지원 (일부 기능 제한)
</details>

### 문제 3: Layered JAR에서 `snapshot-dependencies` 계층의 용도는?

<details>
<summary>해설</summary>

**snapshot-dependencies 계층:**

```
spring-boot-demo.jar
├── Layer 1: dependencies (변경 거의 없음)
│   ├── spring-core-6.0.0.jar  (릴리스 버전)
│   └── mysql-connector-8.0.0.jar (릴리스)
├── Layer 3: snapshot-dependencies (변경 자주)
│   ├── spring-data-jpa-3.0.0-SNAPSHOT.jar
│   └── my-internal-lib-1.0-SNAPSHOT.jar
```

**역할:**

SNAPSHOT 버전의 의존성들:
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-data-jpa</artifactId>
    <version>3.0.0-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>com.company</groupId>
    <artifactId>my-lib</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

**왜 별도 계층?**

1. SNAPSHOT은 자주 변경됨 (개발 중 버전)
2. 릴리스 버전과 분리 (캐시 효율 향상)
3. 캐시 전략:
   - Layer 1 (릴리스): 거의 변경 안 함 → 캐시 효율 100%
   - Layer 3 (SNAPSHOT): 자주 변경 → 캐시 효율 50%

**실제 빌드:**
```dockerfile
# 처음 빌드
COPY --from=builder dependencies/ ./           # 125MB (새로)
COPY --from=builder snapshot-dependencies/ ./  # 5MB (새로)
COPY --from=builder application/ ./            # 2MB (새로)

# SNAPSHOT 업그레이드 후 빌드
COPY --from=builder dependencies/ ./           # 125MB (캐시!)
COPY --from=builder snapshot-dependencies/ ./  # 6MB (새로)
COPY --from=builder application/ ./            # 2MB (새로)

절감: 130MB → 8MB (약 94%)
```

**없으면:**
- SNAPSHOT 변경 → 릴리스 라이브러리도 함께 재다운로드
- 캐시 효율 저하
</details>

---

<div align="center">

**[⬅️ 이전: 이미지 보안](./04-image-security.md)** | **[홈으로 🏠](../README.md)** | **[다음: 레지스트리 관리 ➡️](./06-registry-management.md)**

</div>
