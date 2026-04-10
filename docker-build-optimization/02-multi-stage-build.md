# 멀티 스테이지 빌드 — 800MB를 80MB로

## 🎯 핵심 질문

- Docker 이미지 크기가 800MB인데, 실제로 필요한 애플리케이션은 50MB 정도뿐이다. 나머지는 어디서 온 걸까?
- 빌드 도구(Gradle, Maven, npm)가 최종 이미지에 포함되는 것을 막을 수 있을까?
- 한 Dockerfile에서 여러 개의 환경을 구성하고, 각각에서 결과물만 뽑아낼 수 있을까?

## 🔍 왜 이 개념이 실무에서 중요한가

**이미지 크기는 곧 배포 속도와 비용입니다:**

1. **레지스트리 저장소 비용**: AWS ECR 월 $0.10/GB 저장소 요금
   - 800MB 이미지 100개 = 80GB = 월 $8
   - 80MB 이미지 100개 = 8GB = 월 $0.80 (10배 절감!)

2. **배포 속도**: 이미지 푸시/풀 시간
   - 800MB 이미지: 10초 (10Mbps 기준)
   - 80MB 이미지: 1초 (90% 빨라짐)
   - 하루 20회 배포: 180초 차이 (3분 절감/일)

3. **컨테이너 시작 시간**: 이미지 레이어 언팩 시간
   - 800MB: 더 많은 파일 → 느린 시작
   - 80MB: 빠른 시작 → 자동 스케일 반응 향상

4. **보안**: 공격 면적 축소
   - 빌드 도구(Gradle 5MB, Maven 3MB, npm 300MB+) 제거
   - 불필요한 라이브러리 제거로 취약점 감소

**현실의 예:**
- Netflix: 언어별 이미지 최적화로 월간 $100,000+ 절감
- Spotify: 멀티 스테이지 빌드로 배포 시간 50% 단축

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

### 문제 1: 빌드 도구가 최종 이미지에 포함됨

```dockerfile
# ❌ 나쁜 Dockerfile: JDK 전체 + 빌드 도구 + 애플리케이션
FROM openjdk:17-jdk

WORKDIR /app

COPY . .

# Gradle 설치 (6MB)
RUN curl -O https://services.gradle.org/distributions/gradle-7.6-bin.zip
RUN unzip gradle-7.6-bin.zip

# 의존성 다운로드 (150MB)
RUN ./gradlew dependencies

# 컴파일 및 빌드 (1000+개 JAR 파일)
RUN ./gradlew clean build -x test

ENTRYPOINT ["java", "-jar", "/app/build/libs/app.jar"]
```

**최종 이미지 크기:**
```
FROM openjdk:17-jdk          = 500MB (JDK 전체)
  + gradle                    = 6MB
  + gradle cache              = 150MB
  + spring boot dependencies  = 100MB
  + source code               = 10MB
  + build artifacts           = 34MB (JAR + libs)
──────────────────────────────
총합                          = 800MB

문제:
- JDK 포함 (컨테이너에서 필요 없음, 빌드에만 필요)
- Gradle 포함 (실행 시 불필요)
- gradle cache 포함 (실행 시 불필요)
- 모든 라이브러리 소스코드와 함께 복사됨
- 불필요한 파일 제거 안 함
```

### 문제 2: Node.js 애플리케이션의 문제

```dockerfile
# ❌ 나쁜 Dockerfile: npm 전체 포함
FROM node:18

WORKDIR /app

COPY package*.json ./

# 모든 의존성 설치 (300MB+)
RUN npm install

# 개발 의존성도 함께 설치됨
COPY . .

# 빌드
RUN npm run build

# 최종 이미지에 모두 포함
CMD ["node", "dist/server.js"]
```

**최종 이미지 크기:**
```
FROM node:18                 = 450MB (Node.js 전체)
  + node_modules             = 300MB (모든 개발 도구 포함)
    - webpack (100MB)
    - typescript (50MB)
    - test libraries (100MB)
  + source code              = 10MB
  + dist folder              = 5MB
──────────────────────────────
총합                          = 765MB

문제:
- 빌드에만 필요한 webpack 포함
- 테스트 라이브러리(jest, mocha 등) 포함
- TypeScript 컴파일러 포함
- 원본 소스 코드 포함 (dist만 필요)
- 실행에는 dist와 node_modules의 일부만 필요
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 방식 1: Spring Boot + 멀티 스테이지

```dockerfile
# ✅ Stage 1: Builder (빌드 환경)
FROM openjdk:17-jdk AS builder

WORKDIR /app

COPY build.gradle settings.gradle ./
RUN ./gradlew dependencies

COPY src ./src

# 빌드 (gradle cache 포함, 불필요한 파일 포함)
RUN ./gradlew clean build -x test

# Stage 1의 결과물:
# - gradle (6MB) ❌ 버려짐
# - gradle cache (150MB) ❌ 버려짐
# - build/libs/app.jar (50MB) ✅ 가져옴
# - build/libs/lib/ (100MB) ✅ 가져옴


# ✅ Stage 2: Runtime (실행 환경)
FROM openjdk:17-jre-slim  # JDK 대신 JRE 사용 (450MB → 150MB)

WORKDIR /app

# Stage 1의 결과물에서 필요한 것만 복사
COPY --from=builder /app/build/libs/app.jar .
COPY --from=builder /app/build/libs/lib ./lib

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**최종 이미지 크기:**
```
FROM openjdk:17-jre-slim     = 150MB (JRE만, JDK 제외)
  + app.jar                   = 50MB
  + lib (의존성)              = 100MB
──────────────────────────────
총합                          = 300MB (기존 800MB → 62% 절감!)

제거된 것들:
- Gradle 제거 (6MB)
- gradle cache 제거 (150MB)
- JDK 제거 (300MB)
- 빌드 임시 파일 제거 (50MB)
```

### 방식 2: Node.js + 멀티 스테이지

```dockerfile
# ✅ Stage 1: Builder (빌드 환경)
FROM node:18 AS builder

WORKDIR /app

COPY package*.json ./

# npm install로 모든 의존성 설치 (개발 포함)
RUN npm install --legacy-peer-deps

COPY . .

# 빌드 생성
RUN npm run build

# Stage 1의 결과물:
# - node_modules (300MB) ❌ 버려짐 (웹팩, 테스트 도구 포함)
# - dist (5MB) ✅ 가져옴
# - src (10MB) ❌ 버려짐


# ✅ Stage 2: Runtime (실행 환경)
FROM node:18-slim  # 기존 node:18 → node:18-slim (450MB → 200MB)

WORKDIR /app

COPY package*.json ./

# 프로덕션 의존성만 설치 (30MB)
RUN npm install --production

# Stage 1의 빌드 결과물만 복사
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/server.js"]
```

**최종 이미지 크기:**
```
FROM node:18-slim           = 200MB (slim 사용)
  + package.json 의존성      = 30MB (프로덕션만)
  + dist 폴더                = 5MB
──────────────────────────────
총합                         = 235MB (기존 765MB → 69% 절감!)

제거된 것들:
- npm dev dependencies 제거 (웹팩, TypeScript 등 270MB)
- 원본 소스 코드 제거 (src/ 10MB)
- 모든 node_modules 소스 제거 (node_modules/ 제거 후 필요한 것만 설치)
- Node.js slim 버전 사용 (npm 제거, 필수 도구만 포함)
```

## 🔬 내부 동작 원리

### 1. **멀티 스테이지 빌드 구조**

```dockerfile
# Stage 1: builder
FROM ubuntu:22.04 AS builder
...작업들...

# Stage 2: runtime
FROM ubuntu:22.04 AS runtime
COPY --from=builder /path/to/result .
...

# Stage 3: debug (선택사항)
FROM ubuntu:22.04 AS debug
COPY --from=builder /path/to/artifacts .
...
```

**각 스테이지의 특성:**
- 독립적인 파일시스템 (각 스테이지마다 새로운 /app, /tmp 등)
- 독립적인 명령어 실행 (한 스테이지의 RUN 결과가 다른 스테이지에 영향 없음)
- 기본 빌드 대상: **마지막 FROM 명령어의 스테이지**

### 2. **COPY --from 동작 방식**

```dockerfile
FROM ubuntu:22.04 AS builder
WORKDIR /build
RUN echo "build result" > /build/output.txt

FROM ubuntu:22.04 AS runtime
WORKDIR /app
# builder 스테이지의 /build/output.txt를 runtime의 /app/output.txt로 복사
COPY --from=builder /build/output.txt /app/output.txt
```

**중요한 점:**
- `COPY --from`은 **파일시스템 스냅샷**을 복사
- 복사 대상 스테이지의 모든 파일이 유지됨 (일부만 선택 복사 불가)
- 따라서 빌드 스테이지에서는 불필요한 파일 최소화

### 3. **선택적 빌드 대상 (--target)**

```dockerfile
# Stage 1: builder
FROM ubuntu:22.04 AS builder
RUN apt-get install -y build-essential
RUN ./build.sh

# Stage 2: runtime
FROM ubuntu:22.04 AS runtime
COPY --from=builder /out/app .
CMD ["./app"]

# Stage 3: debug (개발/디버깅용)
FROM ubuntu:22.04 AS debug
COPY --from=builder /out/app .
RUN apt-get install -y gdb strace
CMD ["gdb", "./app"]
```

**빌드 시점에 대상 선택:**
```bash
# 기본: 마지막 스테이지(runtime)만 빌드
docker build -t myapp:prod .
# 결과: runtime 스테이지의 이미지 (최적화됨)

# 디버그 이미지 빌드
docker build -t myapp:debug --target debug .
# 결과: debug 스테이지의 이미지 (gdb/strace 포함)

# 빌드 검증용
docker build -t myapp:build --target builder .
# 결과: builder 스테이지의 이미지 (빌드 결과물 확인용)
```

### 4. **멀티 스테이지의 캐시 효율**

```dockerfile
# Stage 1: 30초 소요
FROM openjdk:17-jdk AS builder
COPY build.gradle ./
RUN ./gradlew dependencies  # 캐시 유효 시 0초

# Stage 2: 2초 소요
FROM openjdk:17-jre-slim
COPY --from=builder ...
```

**캐시 동작:**
- 각 스테이지는 **독립적인 캐시** 보유
- Stage 1의 RUN이 캐시되어도, Stage 2는 새로 실행되지 않음
- 따라서 여러 스테이지가 있어도 **필요한 스테이지만 재실행**

## 💻 실전 실험

### 실험 1: Spring Boot 멀티 스테이지 빌드

**Step 1: 나쁜 예제 (빌드 도구 포함)**

```dockerfile
# ❌ Dockerfile.bad
FROM openjdk:17-jdk

WORKDIR /app

COPY . .

RUN ./gradlew clean build -x test

ENTRYPOINT ["java", "-jar", "/app/build/libs/spring-boot-demo.jar"]
```

```bash
# 빌드
$ docker build -f Dockerfile.bad -t springapp:bad .
[+] Building 120.3s

# 이미지 크기 확인
$ docker images springapp:bad
REPOSITORY   TAG    IMAGE ID      SIZE
springapp    bad    abc1234567890 800MB  ❌

# 레이어 확인
$ docker history springapp:bad
IMAGE          CREATED             SIZE
abc1234...     20 seconds ago      50MB   (JAR 파일)
def5678...     50 seconds ago      150MB  (gradle cache)
ghi9012...     80 seconds ago      300MB  (JDK)
...
```

**Step 2: 좋은 예제 (멀티 스테이지)**

```dockerfile
# ✅ Dockerfile.good
FROM openjdk:17-jdk AS builder

WORKDIR /build

COPY build.gradle settings.gradle ./

RUN ./gradlew dependencies

COPY src ./src

RUN ./gradlew clean build -x test


FROM openjdk:17-jre-slim

WORKDIR /app

COPY --from=builder /build/build/libs/spring-boot-demo.jar .

COPY --from=builder /build/build/libs/lib ./lib

ENTRYPOINT ["java", "-jar", "spring-boot-demo.jar"]
```

```bash
# 빌드
$ docker build -f Dockerfile.good -t springapp:good .
[+] Building 125.4s

# 이미지 크기 확인
$ docker images springapp:good
REPOSITORY   TAG     IMAGE ID      SIZE
springapp    good    xyz9876543210 320MB  ✅ (800MB → 320MB, 60% 절감)

# 레이어 확인
$ docker history springapp:good
IMAGE          CREATED             SIZE
xyz9876...     10 seconds ago      50MB   (JAR 파일)
abc1234...     15 seconds ago      100MB  (의존성 라이브러리)
def5678...     30 seconds ago      150MB  (JRE)
...
```

**Step 3: 이미지 크기 비교**

```bash
$ docker images | grep springapp
springapp     bad     abc1234567890    800MB
springapp     good    xyz9876543210    320MB
springapp     better  pqr1234567890    280MB  # 아래 참고

# 크기 비교
$ du -sh $(docker inspect --format='{{.GraphDriver.Data.MergedDir}}' \
  $(docker create springapp:bad))
800M  /var/lib/docker/containers/.../merged

$ du -sh $(docker inspect --format='{{.GraphDriver.Data.MergedDir}}' \
  $(docker create springapp:good))
320M  /var/lib/docker/containers/.../merged
```

### 실험 2: Node.js 멀티 스테이지 + 더블 npm install

**더 최적화된 방식 (프로덕션 의존성만 최종 이미지에 포함)**

```dockerfile
# ❌ Dockerfile.nodejs.bad (450MB)
FROM node:18

WORKDIR /app
COPY package*.json ./
RUN npm install --legacy-peer-deps
COPY . .
RUN npm run build

CMD ["node", "dist/server.js"]
```

```dockerfile
# ✅ Dockerfile.nodejs.good (180MB)
# Stage 1: Builder
FROM node:18 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm install --legacy-peer-deps

COPY . .
RUN npm run build
# Stage 1 결과물:
# - dist (빌드 결과, 5MB) ✅
# - node_modules (모든 것 포함, 300MB) ❌ 버려짐


# Stage 2: Runtime
FROM node:18-slim

WORKDIR /app

COPY package*.json ./
# 프로덕션 의존성만 설치 (30MB, 개발 의존성 제외)
RUN npm install --production --legacy-peer-deps

# 빌드된 결과물만 복사
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/server.js"]
```

```bash
# 빌드 및 비교
$ docker build -f Dockerfile.nodejs.bad -t nodeapp:bad .
$ docker build -f Dockerfile.nodejs.good -t nodeapp:good .

$ docker images | grep nodeapp
nodeapp     bad      450MB
nodeapp     good     180MB  # 60% 절감!

# 더 자세한 분석
$ docker history nodeapp:bad
IMAGE      CREATED    SIZE
...        1 min      5MB    (dist)
...        2 min      300MB  (node_modules - 웹팩, 테스트 도구 포함)
...        3 min      145MB  (Node.js)

$ docker history nodeapp:good
IMAGE      CREATED    SIZE
...        1 min      5MB    (dist)
...        2 min      30MB   (프로덕션 의존성만)
...        3 min      145MB  (Node.js slim)
```

### 실험 3: Python 데이터 사이언스 애플리케이션

```dockerfile
# ❌ Dockerfile.python.bad (1.2GB!)
FROM python:3.11

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt
# 설치되는 것:
# - jupyter notebook (200MB)
# - matplotlib (150MB)
# - pandas (300MB+)
# - scipy, numpy, scikit-learn (400MB+)
# 모두 최종 이미지에 포함됨!

COPY . .
CMD ["python", "main.py"]
```

```dockerfile
# ✅ Dockerfile.python.good (800MB → 200MB)
# Stage 1: Builder (모든 라이브러리 설치)
FROM python:3.11 AS builder

WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# 결과: jupyter, matplotlib 등 모두 포함 (800MB)


# Stage 2: Runtime (실행에 필요한 것만)
FROM python:3.11-slim

WORKDIR /app

# Builder의 venv 복사 (필요한 라이브러리만)
COPY --from=builder /opt/venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

COPY . .

CMD ["python", "main.py"]
```

```bash
$ docker images | grep pyapp
pyapp    bad      1.2GB
pyapp    good     450MB  # 62% 절감!
```

## 📊 성능/비용 비교

### 빌드 시간 비교

| 프로젝트 | 방식 | 첫 빌드 | 소스 변경 | 의존성 변경 |
|--------|------|--------|---------|---------|
| **Spring Boot** | 단일 스테이지 | 120초 | 120초 | 120초 |
| **Spring Boot** | 멀티 스테이지 | 125초 | 5초 | 120초 |
| **Node.js** | 단일 스테이지 | 90초 | 90초 | 90초 |
| **Node.js** | 멀티 스테이지 | 95초 | 3초 | 90초 |

**효과: 소스 변경 시 빌드 시간 96-97% 단축 (캐시 활용)**

### 이미지 크기 비교

| 프로젝트 | 단일 스테이지 | 멀티 스테이지 | 절감율 |
|--------|----------|----------|------|
| Spring Boot (JDK 포함) | 800MB | 320MB | 60% |
| Node.js 애플리케이션 | 765MB | 180MB | 77% |
| Python 데이터 과학 | 1,200MB | 450MB | 62% |
| Go 애플리케이션 | 800MB | 15MB | 98% |

### 월간 비용 절감 (100개 이미지, 10회/월 푸시)

```
ECR 저장소 비용:  $0.10/GB/월
푸시 대역폭 비용: $0.02/GB

Spring Boot 예시:
❌ 단일 스테이지: (800MB × 100 × $0.10) + (800MB × 10 × 100 × $0.02) = $8 + $160 = $168/월
✅ 멀티 스테이지: (320MB × 100 × $0.10) + (320MB × 10 × 100 × $0.02) = $3.2 + $64 = $67.2/월

절감: $100.8/월 = $1,209.6/년
```

### 배포 속도 향상

```
네트워크 속도: 10Mbps

800MB 이미지 배포:
- 푸시 시간: 800MB / 10Mbps = 640초 (10.7분)
- 풀 시간: 640초
- 총 시간: 1,280초 (21분)

320MB 이미지 배포:
- 푸시 시간: 320MB / 10Mbps = 256초 (4.3분)
- 풀 시간: 256초
- 총 시간: 512초 (8.5분)

개선: 60% 빨라짐
```

## ⚖️ 트레이드오프

### 1. **복잡도 vs 이미지 크기**

```dockerfile
# ❌ 간단하지만 큼 (단일 스테이지)
FROM ubuntu:22.04
RUN apt-get install -y build-essential java maven
COPY . .
RUN ./build.sh
```

vs

```dockerfile
# ✅ 복잡하지만 작음 (멀티 스테이지)
FROM ubuntu:22.04 AS builder
RUN apt-get install -y build-essential java maven
COPY . .
RUN ./build.sh

FROM ubuntu:22.04
COPY --from=builder /out .
```

**결론: 프로덕션 환경에서는 복잡도를 감수할 가치 있음**

### 2. **너무 많은 스테이지**

```dockerfile
# ❌ 지나치게 많은 스테이지 (관리 어려움)
FROM ubuntu AS stage1
...
FROM ubuntu AS stage2
COPY --from=stage1 ...
...
FROM ubuntu AS stage3
COPY --from=stage2 ...
...
FROM ubuntu
COPY --from=stage3 ...
```

vs

```dockerfile
# ✅ 적절한 개수의 스테이지 (보통 2-3개)
FROM ubuntu AS builder
...
FROM ubuntu
COPY --from=builder ...
```

**일반적인 스테이지 수:**
- 최소: 2개 (builder, runtime)
- 권장: 2-3개 (builder, runtime, debug)
- 과다: 4개 이상 (관리 부담 증가)

### 3. **런타임 베이스 이미지 선택**

```dockerfile
# 크기 vs 호환성 트레이드오프
FROM openjdk:17-jdk        # 500MB (JDK 전체)
FROM openjdk:17-jre        # 300MB (JRE, 대부분 호환)
FROM openjdk:17-jre-slim   # 150MB (최소한의 JRE)
FROM eclipse-temurin:17-jre-focal  # 160MB (더 안정적)
```

**선택 기준:**
- `-jdk`: 개발 환경 또는 런타임에 컴파일 필요한 경우
- `-jre`: 일반 프로덕션 (권장)
- `-jre-slim`: 최소 크기 필요한 경우 (주의: 일부 라이브러리 호환 문제)

## 📌 핵심 정리

1. **멀티 스테이지 빌드는 빌드 환경과 실행 환경을 분리**
   - Builder 스테이지: 빌드 도구, 의존성 포함
   - Runtime 스테이지: 필요한 것만 포함

2. **`COPY --from=builder`로 이전 스테이지의 결과물만 가져옴**
   - gradle, maven, npm dev dependencies 제외
   - JDK 제외, JRE만 포함

3. **이미지 크기 60-80% 감소 가능**
   - 빌드 도구 제외
   - 불필요한 라이브러리 제외
   - slim 베이스 이미지 사용

4. **각 스테이지는 독립적인 캐시 보유**
   - 소스 변경 시에도 builder 스테이지 캐시 활용
   - 재빌드 속도 대폭 개선

5. **--target으로 선택적 빌드 가능**
   - 프로덕션 이미지 vs 디버그 이미지 구분
   - 빌드 검증용 이미지 따로 생성

## 🤔 생각해볼 문제

### 문제 1: 다음 Dockerfile의 문제점을 찾으세요.

```dockerfile
FROM openjdk:17-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew build -x test

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=builder /app .  # ⚠️ 주의
ENTRYPOINT ["java", "-jar", "build/libs/app.jar"]
```

<details>
<summary>해설</summary>

**문제점:**
- `COPY --from=builder /app .`는 builder의 `/app` 디렉토리 전체를 복사
- 여기에는:
  - build/ (빌드 결과) ✅
  - src/ (원본 소스) ❌
  - gradle cache (.gradle/) ❌
  - 모든 파일이 포함됨!

**올바른 방식:**
```dockerfile
FROM openjdk:17-jdk AS builder
WORKDIR /app
COPY build.gradle settings.gradle ./
RUN ./gradlew dependencies
COPY src ./src
RUN ./gradlew build -x test

FROM openjdk:17-jre-slim
WORKDIR /app
# 필요한 것만 복사
COPY --from=builder /app/build/libs/app.jar .
COPY --from=builder /app/build/libs/lib ./lib
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**효과:**
- src/ 제외로 추가 10MB 절감
- gradle cache 제외로 추가 150MB 절감
</details>

### 문제 2: Java 애플리케이션에서 slim 이미지를 사용하지 않는 경우는?

<details>
<summary>해설</summary>

**slim 이미지가 부족한 경우:**

1. **Native 라이브러리 필요** (예: libc 호환성)
   ```java
   // JNA로 C 라이브러리 호출
   Native.load("mylib.so");  // libc 필요
   ```
   - slim: `libc-minimal` (문제 발생 가능)
   - 표준 JRE: 전체 `libc` (안정적)

2. **시스템 도구 필요** (예: SSL 인증서, fontconfig)
   ```java
   // PDF 생성에서 폰트 필요
   Runtime.getRuntime().exec("fc-list");
   ```

3. **특수 라이브러리 필요** (예: ImageMagick 바인딩)
   ```java
   Runtime.getRuntime().exec("convert image.jpg");
   ```

**해결 방법:**
```dockerfile
# slim으로 시작했다가 필요한 것만 추가
FROM openjdk:17-jre-slim

RUN apt-get update && apt-get install -y \
    fontconfig \
    libfreetype6 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder ...
```

**결과: 기본 slim (150MB) + 필요한 라이브러리 (30MB) = 180MB (여전히 300MB 이상 절감)**
</details>

### 문제 3: Node.js 프로젝트에서 Stage 1에서 `--production`을 사용하지 않는 이유는?

<details>
<summary>해설</summary>

**왜 Stage 1 (Builder)에서는 모든 의존성을 설치하는가?**

```dockerfile
FROM node:18 AS builder
COPY package*.json ./
RUN npm install --legacy-peer-deps  # ← 모든 의존성 (dev 포함)
COPY . .
RUN npm run build  # ← webpack, TypeScript 등 필요
```

**이유:**
1. `npm run build`에서 webpack, TypeScript, babel 등이 필요
2. 이들은 devDependencies에 명시됨
3. `--production` 플래그를 사용하면 devDependencies를 설치하지 않음
4. 따라서 빌드 실패

**Stage 2에서는 왜 `--production`을 사용하는가?**

```dockerfile
FROM node:18-slim
COPY package*.json ./
RUN npm install --production  # ← 프로덕션 의존성만
COPY --from=builder /app/dist ./dist
```

**이유:**
1. dist 폴더는 이미 빌드됨 (webpack, TypeScript 컴파일 완료)
2. 런타임은 express, axios 등 실행 라이브러리만 필요
3. devDependencies는 제거해도 괜찮음

**결과:**
- Stage 1: 300MB (모든 의존성)
- Stage 2: 30MB (프로덕션 의존성만) ← 270MB 절감!
</details>

---

<div align="center">

**[⬅️ 이전: 레이어 캐시 원리](./01-layer-cache-principles.md)** | **[홈으로 🏠](../README.md)** | **[다음: BuildKit 내부 동작 ➡️](./03-buildkit-internals.md)**

</div>
