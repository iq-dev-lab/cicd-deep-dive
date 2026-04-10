# Dockerfile 레이어 캐시 원리 — 순서가 전부다

## 🎯 핵심 질문

- Docker 빌드가 왜 갑자기 느려졌을까? 코드 한 줄만 바꿨는데 모든 것을 처음부터 다시 빌드하는 이유는?
- `docker build` 명령어가 "Using cache" 메시지를 보여주는데, 무엇을 캐시하는 걸까?
- Dockerfile의 명령어 순서를 바꾸는 것이 빌드 속도에 정말 큰 영향을 미칠까?

## 🔍 왜 이 개념이 실무에서 중요한가

Docker 빌드 시간은 **개발 생산성의 직결**입니다. 매번 빌드에 5분이 걸린다면, 하루 20번 빌드할 때 100분(1시간 40분)의 시간을 낭비합니다. 

더 심각한 것은 **CI/CD 파이프라인의 병목**입니다. GitHub Actions에서 5분 빌드는 월간 GitHub Actions 시간을 빠르게 소진하고, 배포 속도를 느리게 만듭니다.

그런데 **Dockerfile 작성 순서 하나만 바꿔도** 빌드 시간을 30초로 단축할 수 있습니다. 이것이 바로 레이어 캐시 원리를 아는 것이 중요한 이유입니다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```dockerfile
# ❌ 잘못된 Dockerfile: 소스 코드가 변경될 때마다 모든 단계를 다시 실행
FROM openjdk:17-slim

WORKDIR /app

# 모든 파일을 한 번에 복사 (의존성 + 소스 코드)
COPY . .

# 의존성을 다운로드하고 컴파일 시작
RUN ./gradlew clean build -x test

# 최종 JAR 파일 실행
ENTRYPOINT ["java", "-jar", "/app/build/libs/app.jar"]
```

**문제점:**
1. 소스 코드 한 줄을 수정하면 `COPY . .` 레이어 캐시가 무효화됨
2. 그 다음 `RUN ./gradlew clean build` 부터 **모든 의존성을 다시 다운로드**
3. 빌드 시간: 의존성 다운로드(3분) + 컴파일(2분) = 5분
4. 이것이 매번 반복됨

실제로 이렇게 하면:
```bash
$ docker build -t myapp:v1 .
Step 1/4 : FROM openjdk:17-slim
 ---> 1234567890ab
Step 2/4 : COPY . .
 ---> Using cache  # 처음은 캐시 없음
 ---> 5678901234cd
Step 3/4 : RUN ./gradlew clean build -x test
 ---> Running in abcdef123456  # ❌ 캐시를 사용하지 않음! 모두 다시 실행
Downloading gradle-7.6...
....(3분 소요)
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```dockerfile
# ✅ 올바른 Dockerfile: 의존성을 먼저 캐시, 소스 코드는 나중에
FROM openjdk:17-slim

WORKDIR /app

# Step 1: 의존성 파일만 먼저 복사 (변경이 거의 없음)
COPY build.gradle settings.gradle ./

# Step 2: 의존성 다운로드 및 캐시
RUN ./gradlew dependencies

# Step 3: 소스 코드 복사 (자주 변경됨)
COPY src ./src

# Step 4: 컴파일 (의존성 캐시를 활용하므로 빠름)
RUN ./gradlew clean build -x test

ENTRYPOINT ["java", "-jar", "/app/build/libs/app.jar"]
```

**장점:**
1. 소스 코드 수정 시 의존성 다운로드를 **건너뜀**
2. `COPY . .` 대신 필요한 파일만 순서대로 복사
3. 빌드 시간: 의존성(3분, 캐시됨) + 컴파일(20초) = 20초

실제 빌드 과정:
```bash
$ docker build -t myapp:v2 .
Step 1/5 : FROM openjdk:17-slim
 ---> 1234567890ab
Step 2/5 : COPY build.gradle settings.gradle ./
 ---> Using cache  # ✅ 캐시 히트!
Step 3/5 : RUN ./gradlew dependencies
 ---> Using cache  # ✅ 의존성 캐시 히트!
Step 4/5 : COPY src ./src
 ---> 새 레이어 생성 (소스 코드가 변경됨)
Step 5/5 : RUN ./gradlew clean build -x test
 ---> Running in xyz789...
Compiling... (20초만 소요!)
```

## 🔬 내부 동작 원리

### 1. **레이어의 불변성**

Docker 이미지는 여러 개의 **읽기 전용 레이어**로 구성됩니다. 각 Dockerfile 명령어는 새로운 레이어를 생성합니다:

```
┌─────────────────────────────────────┐
│   Layer 5: RUN ./gradlew build      │ (컴파일 결과)
├─────────────────────────────────────┤
│   Layer 4: COPY src ./src           │ (소스 코드)
├─────────────────────────────────────┤
│   Layer 3: RUN ./gradlew deps       │ (의존성 라이브러리)
├─────────────────────────────────────┤
│   Layer 2: COPY build.gradle ./     │ (의존성 정의)
├─────────────────────────────────────┤
│   Layer 1: FROM openjdk:17-slim     │ (베이스 OS)
└─────────────────────────────────────┘
```

**중요한 특성:**
- 각 레이어는 **내용물의 SHA256 해시**를 기반으로 식별됨
- 동일한 입력(Dockerfile 명령어 + 파일 내용)이면 동일한 레이어 ID
- 한 레이어라도 변경되면 이후 **모든 레이어가 재생성**

### 2. **캐시 무효화 연쇄 반응**

```dockerfile
# 예시 Dockerfile
FROM python:3.11-slim          # Layer A (base image)
COPY requirements.txt .        # Layer B (dependencies definition)
RUN pip install -r requirements.txt  # Layer C (dependencies cached)
COPY app.py .                  # Layer D (application code)
RUN python app.py              # Layer E (execution)
```

**시나리오 1: requirements.txt 변경 없음**
```
$ docker build -t myapp:v1 .
Step 1: Layer A (cache hit)
Step 2: Layer B (cache hit, requirements.txt 동일)
Step 3: Layer C (cache hit, requirements.txt 동일하면 dependencies도 동일)
Step 4: Layer D (cache hit, app.py 동일)
Step 5: Layer E (cache hit)
결과: 모두 캐시 사용 (빌드 시간 < 1초)
```

**시나리오 2: app.py만 변경**
```
$ docker build -t myapp:v2 .
Step 1: Layer A (cache hit)
Step 2: Layer B (cache hit)
Step 3: Layer C (cache hit)
Step 4: Layer D (❌ cache miss! app.py가 변경됨)
Step 5: Layer E (❌ 새로 실행, C의 캐시를 참고)
결과: 4번부터 새로 빌드 (app.py 변경만 하므로 빠름)
```

**시나리오 3: requirements.txt 변경**
```
$ docker build -t myapp:v3 .
Step 1: Layer A (cache hit)
Step 2: Layer B (❌ cache miss! requirements.txt가 변경됨)
Step 3: Layer C (❌ 새로 실행, pip install 다시 수행)
Step 4: Layer D (새로 실행)
Step 5: Layer E (새로 실행)
결과: 2번부터 새로 빌드 (전체 빌드 시간이 오래 걸림)
```

### 3. **캐시 키(Cache Key) 계산 방식**

Docker는 각 레이어의 캐시 여부를 다음과 같이 결정합니다:

```
이전 레이어의 ID + 현재 명령어 + 파일 내용 = 새 레이어 ID
```

**예시:**
```dockerfile
# 첫 번째 빌드
FROM ubuntu:22.04     # Layer ID = hash(ubuntu:22.04)
COPY app.txt .        # Layer ID = hash(layer_id_1 + "COPY app.txt ." + "file content ABC")
RUN echo "hello"      # Layer ID = hash(layer_id_2 + "RUN echo hello" + no file changes)

# 두 번째 빌드 (app.txt의 내용은 동일)
FROM ubuntu:22.04     # Layer ID 동일 ✅
COPY app.txt .        # Layer ID 동일 ✅ (파일 내용 ABC는 변경 없음)
RUN echo "hello"      # Layer ID 동일 ✅
```

**파일 내용이 한 글자라도 변경되면:**
```
# 세 번째 빌드 (app.txt에 한 글자 추가)
FROM ubuntu:22.04     # Layer ID 동일
COPY app.txt .        # Layer ID 변경 ❌ (파일 내용 ABC→ABCx로 변경)
RUN echo "hello"      # Layer ID 변경 ❌ (이전 레이어가 변경되었으므로)
```

## 💻 실전 실험

### 실험 1: 레이어 캐시 동작 확인

**Step 1: 첫 번째 빌드 (모든 레이어 새로 생성)**

```dockerfile
# Dockerfile.cache-demo
FROM ubuntu:22.04

WORKDIR /app

# 의존성 파일만 복사 (변경 거의 없음)
COPY package.json .

# 의존성 설치 (시간 소요)
RUN apt-get update && apt-get install -y \
    curl \
    git \
    wget \
    && rm -rf /var/lib/apt/lists/*

# 소스 코드 복사 (자주 변경)
COPY src/ ./src/

# 애플리케이션 실행
CMD ["cat", "./src/main.py"]
```

```bash
# 첫 번째 빌드: 모든 레이어 새로 생성
$ docker build -f Dockerfile.cache-demo -t demo:v1 .
[+] Building 45.2s (7/7) FINISHED
 => [internal] load build context                              0.1s
 => [1/5] FROM ubuntu:22.04                                   15.3s
 => [2/5] COPY package.json .                                 0.0s
 => [3/5] RUN apt-get update && apt-get install...           28.6s
 => [4/5] COPY src/ ./src/                                    0.0s
 => [5/5] CMD ["cat", "./src/main.py"]                        0.0s
=> exporting to image                                          1.2s

# 빌드 시간: 약 45초
```

**Step 2: 두 번째 빌드 (아무 변경 없음, 모든 레이어 캐시 사용)**

```bash
$ docker build -f Dockerfile.cache-demo -t demo:v2 .
[+] Building 0.8s (7/7) FINISHED
 => [internal] load build context                              0.1s
 => [1/5] FROM ubuntu:22.04                                   0.0s
 => [2/5] COPY package.json .                                 0.0s
 => [3/5] RUN apt-get update && apt-get install...            0.0s  (CACHED!)
 => [4/5] COPY src/ ./src/                                    0.0s
 => [5/5] CMD ["cat", "./src/main.py"]                        0.0s

# 빌드 시간: 약 0.8초 (45배 빠름!)
```

**Step 3: 소스 코드만 수정 후 빌드 (Step 4부터 새로 실행)**

```bash
# src/main.py 파일 수정
$ echo "print('hello world')" > src/main.py

# 세 번째 빌드
$ docker build -f Dockerfile.cache-demo -t demo:v3 .
[+] Building 3.2s (7/7) FINISHED
 => [internal] load build context                              0.1s
 => [1/5] FROM ubuntu:22.04                                   0.0s  (CACHED!)
 => [2/5] COPY package.json .                                 0.0s  (CACHED!)
 => [3/5] RUN apt-get update && apt-get install...            0.0s  (CACHED!)
 => [4/5] COPY src/ ./src/                                    0.0s  (새 레이어)
 => [5/5] CMD ["cat", "./src/main.py"]                        0.0s

# 빌드 시간: 약 3.2초 (의존성은 캐시, 소스만 새로 복사)
```

### 실험 2: 의존성 캐시 활용 (Gradle 프로젝트)

**Before: 의존성을 매번 다시 다운로드**

```dockerfile
# ❌ Dockerfile.bad
FROM openjdk:17-slim
WORKDIR /app

# 모든 파일을 한 번에 복사
COPY . .

# 의존성을 매번 다시 다운로드
RUN ./gradlew dependencies
RUN ./gradlew build -x test
```

```bash
$ time docker build -f Dockerfile.bad -t myapp:bad .
# 첫 빌드: 약 120초 (의존성 다운로드 포함)

$ time docker build -f Dockerfile.bad -t myapp:bad .
# 두 번째 빌드: 약 118초 (⚠️ 여전히 오래 걸림!)
# 이유: src/ 파일이 포함되어 있고, 의존성 캐시가 무효화됨
```

**After: 의존성을 캐시하고 소스는 별도 복사**

```dockerfile
# ✅ Dockerfile.good
FROM openjdk:17-slim
WORKDIR /app

# Step 1: 의존성 정의 파일만 복사 (변경 거의 없음)
COPY build.gradle settings.gradle gradle.properties ./

# Step 2: 의존성 다운로드 (캐시됨)
RUN ./gradlew dependencies

# Step 3: 소스 코드 복사 (자주 변경)
COPY src ./src

# Step 4: 빌드 (의존성은 캐시 활용)
RUN ./gradlew build -x test
```

```bash
$ time docker build -f Dockerfile.good -t myapp:good .
# 첫 빌드: 약 120초 (의존성 다운로드 포함)

$ time docker build -f Dockerfile.good -t myapp:good .
# 두 번째 빌드: 약 2초! (의존성 캐시 사용)

# src/main/java/App.java 수정 후
$ time docker build -f Dockerfile.good -t myapp:good .
# 세 번째 빌드: 약 15초 (컴파일만, 의존성 캐시 사용)
```

### 실험 3: `docker history`로 레이어 분석

```bash
$ docker history myapp:latest
IMAGE               CREATED             CREATED BY          SIZE
abc1234567890       10 seconds ago      /bin/sh -c ./grad   150MB
def2345678901       20 seconds ago      /bin/sh -c #(nop)   1.2KB
ghi3456789012       30 seconds ago      /bin/sh -c #(nop)   2.5MB
...
```

각 레이어의 SHA를 확인하여 캐시 여부를 추적할 수 있습니다:

```bash
# 빌드 후 레이어 ID 확인
$ docker build -t myapp:v1 --progress=plain .
...
#3 COPY src ./src
#3 digest: sha256:abc123...
#3 name: layer_src

#4 RUN ./gradlew build -x test
#4 digest: sha256:def456...
#4 name: layer_build

# 다시 빌드 시
$ docker build -t myapp:v2 --progress=plain .
...
#3 COPY src ./src
#3 digest: sha256:abc123...  # 동일한 SHA = 캐시 사용!
```

## 📊 성능/비용 비교

### 빌드 시간 비교

| 시나리오 | 방법 | 첫 빌드 | 의존성 변경 | 소스 변경 |
|--------|------|--------|---------|--------|
| ❌ 나쁜 예 (모든 파일 한번에) | `COPY . .` 후 RUN | 120초 | 120초 | 120초 |
| ✅ 좋은 예 (의존성 먼저) | 의존성 별도 → 소스 별도 | 120초 | 125초 | 20초 |

**월간 비용 절감 (5명 개발자, 하루 20회 빌드):**
- 나쁜 예: 20회 × 120초 = 2,400초 = 40분/일 × 5명 = 200분/일
- 좋은 예: 10회(소스만) × 20초 + 10회(의존성) × 120초 = 200초 + 1,200초 = 1,400초 = 약 23분/일 × 5명 = 115분/일
- **절감량: 약 85분/일** = 7시간/주 = 30시간/월 (개발 생산성 향상)

### GitHub Actions 시간 절감

```
월간 GitHub Actions 무료 한도: Ubuntu 2,000분

❌ 나쁜 예:
- CI/CD 한 번: 빌드 2분 + 테스트 3분 + 푸시 1분 = 6분
- 월간 50회 배포: 50 × 6분 = 300분 소비 (한도의 15%)

✅ 좋은 예:
- CI/CD 한 번: 빌드 0.5분(캐시) + 테스트 3분 + 푸시 1분 = 4.5분
- 월간 50회 배포: 50 × 4.5분 = 225분 소비 (한도의 11%)
- 절감: 75분/월 (25% 절감)
```

## ⚖️ 트레이드오프

### 1. **복잡도 vs 성능**

```dockerfile
# 간단하지만 느림
FROM ubuntu:22.04
COPY . .
RUN apt-get install -y build-essential
RUN ./build.sh
```

vs

```dockerfile
# 복잡하지만 빠름
FROM ubuntu:22.04
COPY package.txt .
RUN apt-get install -y build-essential
COPY src ./src
COPY scripts ./scripts
RUN ./build.sh
```

**결론: 프로젝트 크기가 크고 개발이 활발하면 복잡도를 감수할 가치 있음**

### 2. **너무 세분화된 레이어**

```dockerfile
# ❌ 지나치게 세분화 (관리 복잡, 이미지 크기 증가)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get install -y wget
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*
```

vs

```dockerfile
# ✅ 적절히 그룹화 (캐시 효율 + 관리 용이)
RUN apt-get update && apt-get install -y \
    curl \
    git \
    wget \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

**결론: 변경 빈도가 높은 것들만 별도 레이어로 분리**

### 3. **캐시 무효화 임계값**

- 패키지 추가: 캐시 무효화해도 괜찮음 (의존성 설치 시간 < 30초)
- 소스 코드 변경: 빈번하므로 캐시 무효화 최소화 필요
- 베이스 이미지: 거의 변경 없으므로 캐시 효율 높음

## 📌 핵심 정리

1. **Dockerfile은 위에서 아래로 실행되며, 각 줄이 하나의 레이어를 생성**
   - 레이어는 불변이며 SHA 해시로 식별됨

2. **한 레이어의 캐시가 무효화되면 이후 모든 레이어가 재실행됨**
   - `COPY . .`가 캐시 무효화의 주범

3. **변경 빈도가 낮은 것을 먼저, 높은 것을 나중에 배치**
   - 의존성 → 설정 → 소스 코드 순서가 최적

4. **의존성을 별도로 캐시하면 5배 이상 빌드 속도 개선**
   - 첫 빌드: 120초 → 재빌드(소스만): 20초

5. **`docker history`로 레이어 분석 및 캐시 상태 확인 가능**

## 🤔 생각해볼 문제

### 문제 1: 다음 Dockerfile에서 캐시 문제를 찾으세요.

```dockerfile
FROM node:18-slim
WORKDIR /app

COPY . .
RUN npm install
RUN npm run build
CMD ["npm", "start"]
```

<details>
<summary>해설</summary>

**문제점:**
1. `COPY . .`가 가장 먼저 실행되어 소스 코드 변경 시 의존성 캐시가 무효화됨
2. 매번 `npm install`이 실행됨 (시간 낭비)

**개선 방안:**
```dockerfile
FROM node:18-slim
WORKDIR /app

# Step 1: 의존성 파일만 복사
COPY package.json package-lock.json ./

# Step 2: 의존성 설치 (캐시됨)
RUN npm install

# Step 3: 소스 코드 복사
COPY src ./src

# Step 4: 빌드
RUN npm run build

CMD ["npm", "start"]
```

**효과:**
- 첫 빌드: 45초 (의존성 다운로드)
- 소스 수정 후: 5초 (의존성 캐시 사용)
</details>

### 문제 2: 아래 두 Dockerfile의 캐시 동작을 비교하세요.

```dockerfile
# Dockerfile.A
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
```

```dockerfile
# Dockerfile.B
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl git
```

<details>
<summary>해설</summary>

**Dockerfile.A의 캐시:**
- 첫 빌드: 3개 레이어 생성 (RUN 3개)
- apt-get install -y curl 추가 시: 이전 RUN 캐시 무효화되어 git도 다시 설치

**Dockerfile.B의 캐시:**
- 첫 빌드: 1개 레이어 생성 (RUN 1개)
- 패키지 추가 시: 전체 RUN이 다시 실행 (동일 결과)

**핵심 차이:**
- Dockerfile.A는 각 명령어가 독립적인 레이어이지만, 한 명령어 추가 시 이전 레이어도 무효화될 수 있음
- Dockerfile.B는 단일 레이어이므로 더 간결하고 예측 가능함

**결론: 캐시 목적이 아니라면 Dockerfile.B가 낫다. 이미지 크기도 더 작음.**
</details>

### 문제 3: `build.gradle`과 `settings.gradle`을 별도로 복사하는 이유는?

<details>
<summary>해설</summary>

```dockerfile
# ✅ 왜 이렇게 해야 하는가?
COPY build.gradle settings.gradle ./
RUN ./gradlew dependencies
COPY src ./src
RUN ./gradlew build
```

**이유:**
1. `build.gradle`과 `settings.gradle`은 거의 변경되지 않음
2. 이 두 파일을 먼저 복사하면 `RUN ./gradlew dependencies` 캐시 히트율 높음
3. 소스 코드(`src/`)는 자주 변경되므로 나중에 복사
4. 결과: 소스 변경 시 의존성 다운로드를 건너뜀

**만약 `gradle.properties`도 있다면:**
```dockerfile
COPY build.gradle settings.gradle gradle.properties ./
RUN ./gradlew dependencies
COPY src ./src
```

**캐시 효율이 떨어지는 경우:**
- `gradle.properties`에 버전 번호 변경 → `RUN ./gradlew dependencies` 캐시 무효화
- 해결: `gradle.properties`와 `build.gradle`을 따로 관리하거나, 버전 변경이 적은 `gradle.properties`를 `build.gradle`과 함께 복사
</details>

---

<div align="center">

**[⬅️ 이전: Chapter 2 — 아티팩트와 리포트](../github-actions/06-artifacts-reports.md)** | **[홈으로 🏠](../README.md)** | **[다음: 멀티 스테이지 빌드 ➡️](./02-multi-stage-build.md)**

</div>
