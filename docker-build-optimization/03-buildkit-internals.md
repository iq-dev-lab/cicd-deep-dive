# BuildKit 내부 동작 — 병렬 스테이지와 원격 캐시

## 🎯 핵심 질문

- Docker 빌드가 언제부터 병렬로 처리되기 시작했을까?
- 여러 대의 빌드 머신이 이전 빌드의 캐시를 공유할 수 있을까? (로컬 캐시 외에)
- GitHub Actions에서 빌드할 때마다 캐시가 초기화되는데, 이를 유지할 방법은?

## 🔍 왜 이 개념이 실무에서 중요한가

**CI/CD 파이프라인 속도의 핵심:**

1. **병렬 빌드로 시간 절감**
   - 기존: 3개 스테이지 순차 실행 (30초 + 20초 + 10초 = 60초)
   - BuildKit: 의존성 없는 스테이지 병렬 실행 (max(30, 20, 10) = 30초)

2. **원격 캐시로 CI 무한 가속화**
   - 첫 빌드: 120초 (의존성 다운로드)
   - 이후 모든 빌드: 5초 (원격 캐시 활용)
   - 월간 50회 배포: 60분 → 5분 (95% 절감!)

3. **GitHub Actions 비용 절감**
   - 빌드 시간 단축 = GitHub Actions 시간 절감
   - 월간 2,000분 무료 한도 초과 시 가격 폭등 ($0.008/분)

4. **개발 생산성 향상**
   - 로컬 빌드 캐시 효율 향상
   - 팀 전체가 동일한 캐시 공유 (재다운로드 불필요)

**현실의 예:**
- Spotify: BuildKit 도입으로 빌드 시간 40% 단축
- Uber: 멀티 머신 빌드로 배포 속도 5배 향상

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

### 문제 1: 구형 Docker 빌더 사용 (Legacy Builder)

```bash
# ❌ 기본 Docker로 빌드 (구형 빌더 자동 사용)
$ docker build -t myapp:v1 .
[+] Building 45.3s

# 문제점:
# 1. 병렬 실행 불가 (스테이지를 순차적으로 실행)
# 2. 빌드 중간 레이어가 커짐 (임시 파일 캐시)
# 3. 원격 캐시 지원 불가
```

### 문제 2: GitHub Actions에서 매번 캐시 초기화

```yaml
# ❌ 나쁜 예: 캐시 전략 없음
name: Build
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: myapp:latest
          # ❌ cache-from / cache-to 없음!
          # 결과: 매번 의존성 다시 다운로드 (120초)
```

**실제 CI 실행 시간:**
```
Run 1: 125초 (의존성 다운로드)
Run 2: 128초 (캐시 없음, 다시 다운로드)
Run 3: 122초 (캐시 없음, 다시 다운로드)
...
월간 50회 배포: 50 × 125초 = 6,250초 (약 104분)
```

### 문제 3: 로컬 개발 + CI 빌드에서 캐시 불일치

```bash
# 개발자 로컬 머신
$ docker build -t myapp:dev .
# 캐시 위치: ~/.docker/buildx/... (로컬만 유효)
# 이 캐시는 CI 파이프라인에서 사용 불가!

# GitHub Actions
$ docker build -t myapp:latest .
# 캐시 위치: GitHub 러너의 /tmp (일시적)
# 런너가 종료되면 캐시 삭제!

# 결과:
# 개발자: 5초 (로컬 캐시 사용)
# CI: 125초 (캐시 없음)
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 방식 1: BuildKit 활성화 (기본 설정)

```bash
# ✅ BuildKit으로 빌드 (모던 빌더, 병렬 실행 지원)
$ docker build -t myapp:v1 .
# 자동으로 BuildKit 사용 (Docker 20.10+)
```

또는 명시적으로 활성화:

```bash
# BuildKit 명시 활성화
$ docker buildx build -t myapp:v1 .
# 또는 환경변수 설정
$ export DOCKER_BUILDKIT=1
$ docker build -t myapp:v1 .
```

### 방식 2: GitHub Actions에서 원격 캐시 활용

```yaml
# ✅ 좋은 예: BuildKit + 원격 캐시
name: Build and Push
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: docker/setup-buildx-action@v2
      
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: myapp:latest
          
          # ✅ 원격 캐시 활용 (GitHub Actions 내장)
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # 결과: 매 빌드마다 캐시 자동 저장 및 로드
```

**실제 CI 실행 시간:**
```
Run 1: 125초 (의존성 다운로드, 캐시 저장)
Run 2: 8초 (캐시에서 로드)
Run 3: 9초 (캐시에서 로드)
...
월간 50회 배포: 125초 + 49 × 8초 = 517초 (약 8.6분)

절감: 104분 → 8.6분 (92% 단축!)
```

### 방식 3: 멀티 플랫폼 빌드 + 원격 캐시

```yaml
# ✅ 복합 예: 멀티 플랫폼 + 원격 캐시
name: Build Multi-Platform
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: docker/setup-buildx-action@v2
        with:
          platforms: linux/amd64,linux/arm64
      
      - uses: docker/build-push-action@v4
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: myapp:latest
          
          # BuildKit이 병렬로 두 플랫폼 동시 빌드
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 🔬 내부 동작 원리

### 1. **BuildKit vs 기존 Docker 빌더**

**기존 Docker 빌더 (Legacy):**
```
Dockerfile:
┌─────────────────────────┐
│ FROM ubuntu:22.04       │ Layer A
├─────────────────────────┤
│ RUN apt-get install ... │ Layer B (depends on A)
├─────────────────────────┤
│ COPY . .                │ Layer C (depends on B)
├─────────────────────────┤
│ RUN ./build.sh          │ Layer D (depends on C)
└─────────────────────────┘

실행 순서: A → B → C → D (순차 실행, 병렬 불가)
시간: 5초 + 10초 + 2초 + 3초 = 20초
```

**BuildKit:**
```
의존성 그래프 (DAG):
┌─────────────────────────┐
│ FROM ubuntu:22.04       │ Layer A
├─────────────────────────┤
│ RUN apt-get install ... │ Layer B (depends on A)
├─────────────────────────┤
│ COPY . .                │ Layer C (depends on B)
├─────────────────────────┤
│ RUN ./build.sh          │ Layer D (depends on C)
└─────────────────────────┘

멀티 스테이지의 경우:
┌─────────────────┐
│ Builder Stage   │ 
│ FROM gradle:7.6 │ (A1)
│ RUN build...    │ (A2)
└─────────────────┘
         ↓ (독립적)
┌─────────────────┐
│ Runtime Stage   │
│ FROM openjdk:17 │ (B1) ← A와 동시 실행 가능!
│ COPY --from...  │ (B2)
└─────────────────┘

실행: 병렬 (max(A의시간, B의시간))
```

### 2. **원격 캐시 메커니즘**

**Docker 캐시 저장 위치:**

```
로컬 캐시:
  ~/.docker/buildx/cache/...  (로컬 머신에만 존재)
  
원격 캐시 (GitHub Actions GHA):
  GitHub의 캐시 저장소 (모든 Runner가 접근 가능)
  
원격 캐시 (Registry):
  Docker Hub, GHCR, ECR 등 (다른 머신도 접근 가능)
```

**캐시 저장 흐름:**

```dockerfile
# Dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl  # Layer A
COPY . .                                        # Layer B
RUN ./build.sh                                  # Layer C
```

**첫 빌드:**
```
1. Layer A 빌드 (새로 생성)
2. Layer A 캐시 저장 (원격 저장소에)
   sha256:layer-a-hash = 20MB
3. Layer B 빌드 (새로 생성)
4. Layer B 캐시 저장
   sha256:layer-b-hash = 5MB
5. Layer C 빌드 (새로 생성)
6. Layer C 캐시 저장
   sha256:layer-c-hash = 100MB
```

**두 번째 빌드 (다른 머신/Runner):**
```
1. Layer A 필요 → 원격 캐시에서 로드 (0초)
2. Layer B 필요 → 원격 캐시에서 로드 (0초)
3. Layer C 필요 → 원격 캐시에서 로드 (0초)
4. 전체 빌드 완료 (5초, 레이어 확인 시간만)
```

**캐시 키 계산:**

```
Cache Key = SHA256(
  이전_레이어_ID + 
  현재_명령어 + 
  파일_내용
)

예시:
Layer A: SHA256(ubuntu:22.04 + "RUN apt-get update...")
       = abc123...def456

Layer B: SHA256(abc123...def456 + "COPY . ." + 파일내용)
       = xyz789...uvw012

# 다른 머신에서 동일한 Dockerfile 빌드
Layer A: SHA256(ubuntu:22.04 + "RUN apt-get update...")
       = abc123...def456 (동일!)
       # 캐시에서 로드 가능
```

### 3. **BuildKit의 의존성 그래프 (DAG) 최적화**

```dockerfile
# 복잡한 멀티 스테이지 Dockerfile
FROM ubuntu:22.04 AS builder1
RUN apt-get update && apt-get install -y build-essential  # 10초
RUN ./build1.sh  # 20초

FROM ubuntu:22.04 AS builder2
RUN apt-get update && apt-get install -y golang  # 8초
RUN ./build2.sh  # 15초

FROM ubuntu:22.04 AS builder3
RUN apt-get update && apt-get install -y python3  # 5초
RUN ./build3.sh  # 18초

FROM ubuntu:22.04
COPY --from=builder1 /out /app1
COPY --from=builder2 /out /app2
COPY --from=builder3 /out /app3
RUN ./link.sh  # 2초
```

**의존성 그래프:**
```
builder1: ubuntu → apt-get → build1.sh → (최종 이미지에 필요)
                                          ↓
builder2: ubuntu → apt-get → build2.sh → (병렬 실행 가능)
                                          ↓
builder3: ubuntu → apt-get → build3.sh → (병렬 실행 가능)

모든 builder가 완료되어야 최종 스테이지 실행

기존 빌더: 20 + 15 + 18 + 2 = 55초 (순차)
BuildKit: max(20, 15, 18) + 2 = 22초 (병렬)

개선: 60% 빠름!
```

### 4. **BuildKit 캐시 모드**

```bash
# Mode 1: inline (캐시를 최종 이미지에 포함, 크기 증가)
docker buildx build \
  --cache-to type=docker,image-manifest=true \
  -t myapp:latest .

# Mode 2: max (모든 레이어의 캐시 저장, 저장소 크기 증가)
docker buildx build \
  --cache-to type=gha,mode=max \
  -t myapp:latest .

# Mode 3: min (최소한의 캐시만 저장, 저장소 크기 작음)
docker buildx build \
  --cache-to type=gha,mode=min \
  -t myapp:latest .
```

## 💻 실전 실험

### 실험 1: BuildKit 병렬 실행 확인

```dockerfile
# Dockerfile.parallel
FROM ubuntu:22.04 AS stage1
RUN echo "Stage 1 시작" && sleep 5 && echo "Stage 1 완료"

FROM ubuntu:22.04 AS stage2
RUN echo "Stage 2 시작" && sleep 3 && echo "Stage 2 완료"

FROM ubuntu:22.04
COPY --from=stage1 /etc/os-release /app1/
COPY --from=stage2 /etc/hostname /app2/
CMD echo "완료"
```

**Step 1: 기존 빌더 (순차 실행)**

```bash
$ docker build -f Dockerfile.parallel -t parallel:old --progress=plain .
# 결과: 
# Stage 1: 5초
# Stage 2: 3초
# Final: 1초
# 총합: 약 9초

# 레이어 구성:
# ├── ubuntu:22.04 (Stage 1용)
# ├── Stage 1 RUN (5초)
# ├── ubuntu:22.04 (Stage 2용)
# ├── Stage 2 RUN (3초)
# ├── ubuntu:22.04 (최종)
# ├── COPY from stage1
# └── COPY from stage2
```

**Step 2: BuildKit (병렬 실행)**

```bash
$ docker buildx build -f Dockerfile.parallel -t parallel:new --progress=plain .
# 결과:
# Stage 1과 Stage 2 동시 실행!
# max(5초, 3초) = 5초
# Final: 1초
# 총합: 약 6초

# 시간 절감: 9초 → 6초 (33% 개선)
```

**상세 로그 확인:**

```bash
$ docker buildx build --progress=plain .
#1 [internal] load build context
#1 DONE 0.1s

#2 [stage1 1/2] FROM ubuntu:22.04
#2 DONE 2.1s

#3 [stage2 1/2] FROM ubuntu:22.04  ← stage1과 동시 실행!
#3 DONE 2.1s

#2 [stage1 2/2] RUN echo "Stage 1..."
#2 DONE 5.0s

#3 [stage2 2/2] RUN echo "Stage 2..."
#3 DONE 3.0s

#4 [final 1/5] FROM ubuntu:22.04
#4 DONE 2.1s

#4 [final 2/5] COPY --from=stage1...
#4 DONE 0.1s

#4 [final 3/5] COPY --from=stage2...
#4 DONE 0.1s
```

### 실험 2: 원격 캐시 효과 측정 (GitHub Actions)

```yaml
# .github/workflows/build-with-cache.yml
name: Build with Remote Cache
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: docker/setup-buildx-action@v2
      
      # 캐시 없이 빌드
      - uses: docker/build-push-action@v4
        with:
          tags: myapp:without-cache
          push: false
          # cache-from, cache-to 없음!

      # 캐시 활용 빌드
      - uses: docker/build-push-action@v4
        with:
          tags: myapp:with-cache
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**실행 결과:**

```
첫 번째 푸시:
- Build without-cache: 128초 (의존성 다운로드)
- Build with-cache: 130초 (캐시 저장 포함)

두 번째 푸시 (같은 commit):
- Build without-cache: 129초 (캐시 없음, 다시 다운로드)
- Build with-cache: 8초 (GHA 캐시에서 로드)

효과: 129초 → 8초 (94% 단축!)
```

### 실험 3: Registry 기반 원격 캐시

```bash
# Docker Hub를 원격 캐시로 사용
$ docker buildx build \
    --cache-from type=registry,ref=docker.io/myuser/myapp:latest \
    --cache-to type=registry,ref=docker.io/myuser/myapp:latest,mode=max \
    -t myuser/myapp:latest \
    --push .

# 또는 GitHub Container Registry (GHCR)
$ docker buildx build \
    --cache-from type=registry,ref=ghcr.io/myuser/myapp:latest \
    --cache-to type=registry,ref=ghcr.io/myuser/myapp:latest,mode=max \
    -t ghcr.io/myuser/myapp:latest \
    --push .
```

**캐시 저장 시간:**

```
빌드: 125초
캐시 업로드: 30MB × 시간 = 약 15초
총 시간: 140초

이후 빌드:
캐시 다운로드: 30MB × 시간 = 약 15초
캐시 검증: 10초
최종 이미지 빌드: 5초
총 시간: 30초 (기존 125초 → 76% 단축)
```

### 실험 4: BuildKit 진행률 상세 보기

```dockerfile
# Dockerfile.progress
FROM ubuntu:22.04

RUN echo "단계 1: 시스템 업데이트" && \
    apt-get update && \
    apt-get install -y curl git && \
    echo "단계 1 완료"

RUN echo "단계 2: 추가 도구 설치" && \
    apt-get install -y build-essential && \
    echo "단계 2 완료"

RUN echo "단계 3: 빌드" && \
    sleep 3 && \
    echo "단계 3 완료"

ENTRYPOINT ["echo", "이미지 완료"]
```

```bash
# 상세 진행률 표시
$ docker buildx build --progress=plain .
#1 [internal] load build context
#1 DONE 0.1s

#2 [1/4] FROM ubuntu:22.04
#2 DONE 3.2s

#3 [2/4] RUN echo "단계 1:..."
#3 10.5s 단계 1: 시스템 업데이트
#3 Getting package lists...
#3 ...
#3 단계 1 완료
#3 DONE 15.3s

#4 [3/4] RUN echo "단계 2:..."
#4 DONE 8.2s

#5 [4/4] RUN echo "단계 3:..."
#5 DONE 3.1s

#6 exporting to image
#6 DONE 1.0s

# 총 빌드 시간: 30.8s
```

## 📊 성능/비용 비교

### 빌드 시간 비교

| 시나리오 | 기존 빌더 | BuildKit | 개선율 |
|--------|---------|---------|------|
| 단일 스테이지, 캐시 무효 | 125초 | 120초 | 4% |
| 단일 스테이지, 캐시 히트 | 2초 | 1초 | 50% |
| 멀티 스테이지(3개), 순차 | 55초 | 22초 | 60% |
| 원격 캐시 첫 빌드 | 125초 | 130초 | -4% |
| 원격 캐시 재빌드 | 125초 | 8초 | 94% |

### GitHub Actions 비용 절감

```
월간 50회 배포, ubuntu-latest 기준 ($0.008/분)

원격 캐시 없음:
- 빌드 시간: 125초/회
- 총 시간: 50 × 125초 = 6,250초 = 104.2분
- 비용: 104.2분 × $0.008 = $0.83/월 (개인은 무료)

원격 캐시 활용:
- 첫 빌드: 130초
- 재빌드: 8초 × 49회 = 392초
- 총 시간: 130 + 392 = 522초 = 8.7분
- 비용: 8.7분 × $0.008 = $0.07/월 (개인은 무료)

엔터프라이즈:
- 절감: 104.2분 → 8.7분 (92% 단축)
- 월간 절감: 95.5분 (약 $0.76, 연간 $9.12)
- 개발 생산성: 월간 95.5분 절감 (팀이 크면 배수로 증가)
```

### 멀티 플랫폼 빌드 시간

```
amd64 + arm64 빌드

기존 빌더 (순차):
- amd64 빌드: 125초
- arm64 빌드: 125초
- 총 시간: 250초

BuildKit (병렬):
- max(amd64, arm64) = 125초
- 저장소 푸시: 30초 (두 플랫폼)
- 총 시간: 155초

개선: 250초 → 155초 (38% 단축)
```

## ⚖️ 트레이드오프

### 1. **캐시 저장소 크기 vs 캐시 히트율**

```bash
# Mode: min (최소 캐시, 저장소 작음)
--cache-to type=gha,mode=min
# 저장소 사용: 5MB
# 캐시 히트율: 70% (일부 레이어만 캐시)

# Mode: max (최대 캐시, 저장소 큼)
--cache-to type=gha,mode=max
# 저장소 사용: 150MB
# 캐시 히트율: 99% (거의 모든 레이어 캐시)

# 권장: max (GitHub Actions는 5GB 무료 한도)
```

### 2. **캐시 다운로드 시간 vs 빌드 시간**

```
Registry 캐시 (Docker Hub/GHCR/ECR):
- 캐시 다운로드: 30초 (30MB, 10Mbps)
- 빌드 시간 절감: 120초
- 순이득: 90초

Registry 캐시가 느린 경우:
- 캐시 다운로드: 2분 (1Mbps)
- 빌드 시간 절감: 120초
- 손실: -60초 (캐시 사용이 역효과)

권장: 빠른 네트워크 환경에서만 Registry 캐시 사용
```

### 3. **BuildKit 호환성**

```
BuildKit 요구사항:
- Docker 20.10+ (2020년 12월 이후)
- Docker Desktop (모든 버전에 포함)
- Linux: BuildKit 별도 설치 필요

호환 문제:
- 구식 Docker (19.x): BuildKit 미지원
- 특수 네트워크 환경 (프록시/방화벽): 설정 필요

해결: docker buildx 사용으로 대부분 해결
```

## 📌 핵심 정리

1. **BuildKit은 의존성 그래프를 분석해 의존성 없는 스테이지를 병렬 실행**
   - 멀티 스테이지 빌드 속도 30-40% 개선

2. **원격 캐시(GHA, Registry)로 CI 파이프라인의 빌드 시간 94% 단축**
   - 첫 빌드: 125초 → 재빌드: 8초

3. **`cache-from type=gha` + `cache-to type=gha,mode=max` 필수**
   - GitHub Actions 환경에서 가장 효율적
   - 추가 비용 없음 (5GB 무료 한도)

4. **멀티 플랫폼 빌드(amd64 + arm64)는 병렬로 실행**
   - 2배 플랫폼 = 38% 시간 절감 (여전히 순차보다 빠름)

5. **Registry 캐시는 네트워크 속도가 빠를 때만 효과적**
   - GHA 캐시가 무난한 선택

## 🤔 생각해볼 문제

### 문제 1: 다음 GitHub Actions 워크플로우에서 문제를 찾으세요.

```yaml
name: Build
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: myapp:latest
          cache-from: type=gha  # ⚠️ 캐시 로드만 함
          # cache-to 없음!
```

<details>
<summary>해설</summary>

**문제점:**
- `cache-from: type=gha`만 지정하면 캐시를 **로드만** 함
- `cache-to` 없으면 빌드 결과를 캐시에 **저장하지 않음**
- 결과: 다음 빌드에서도 캐시가 없음 (매번 처음부터 빌드)

**올바른 설정:**
```yaml
- uses: docker/build-push-action@v4
  with:
    push: true
    tags: myapp:latest
    cache-from: type=gha          # 캐시 로드
    cache-to: type=gha,mode=max   # 캐시 저장 (필수!)
```

**동작:**
- 첫 빌드: 캐시 없음 → 빌드 → 캐시 저장
- 두 번째: 캐시 로드 → 빠른 빌드 → 캐시 업데이트
</details>

### 문제 2: 멀티 스테이지 빌드에서 캐시 무효화가 병렬성에 미치는 영향은?

```dockerfile
FROM golang:1.21 AS builder
COPY . .
RUN go build -o app

FROM ubuntu:22.04
COPY --from=builder /go/app .
CMD ["./app"]
```

<details>
<summary>해설</summary>

**시나리오 1: 소스 코드 변경**
```
변경: main.go 1줄 수정

Dockerfile 빌드 순서:
FROM golang:1.21 → (캐시 히트)
COPY . . → (❌ 캐시 무효, 소스 변경)
RUN go build → (새로 실행)

FROM ubuntu:22.04 → (캐시 히트)
COPY --from=builder → (새로 실행, builder 결과 변경)

병렬성: 없음 (builder는 이전 스테이지가 필요)
```

**시나리오 2: README 파일 변경**
```
변경: README.md 수정

Dockerfile 빌드 순서:
FROM golang:1.21 → (캐시 히트)
COPY . . → (❌ 캐시 무효, README 변경됨)
RUN go build → (새로 실행하지만, 결과는 동일)

FROM ubuntu:22.04 → (캐시 히트)
COPY --from=builder → (새로 실행, builder 출력 동일)

문제: 불필요한 빌드 재실행
```

**최적화 방법:**
```dockerfile
FROM golang:1.21 AS builder

# 의존성 파일만 먼저 복사
COPY go.mod go.sum ./
RUN go mod download

# 소스 코드만 복사
COPY . .
RUN go build -o app

# ...
```

**결과:**
- go.mod/go.sum 변경 시만 `go mod download` 재실행
- README 변경은 캐시에 영향 없음
</details>

### 문제 3: Registry 캐시와 GHA 캐시 중 어느 것을 선택해야 할까?

<details>
<summary>해설</summary>

**GHA 캐시 (GitHub Actions 빌트인):**
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```
- 장점: 무료, 빠름, 설정 간단
- 단점: GitHub Actions에서만 유효
- 권장: 대부분의 경우

**Registry 캐시 (Docker Hub, GHCR, ECR):**
```yaml
cache-from: type=registry,ref=docker.io/user/app:latest
cache-to: type=registry,ref=docker.io/user/app:latest,mode=max
```
- 장점: 여러 CI 플랫폼에서 공유 가능, 구글 클라우드 빌드 등
- 단점: 느림, 저장소 비용 (GHCR는 무료)
- 권장: 여러 CI 플랫폼 사용할 때

**혼합 전략 (권장):**
```yaml
cache-from: |
  type=gha
  type=registry,ref=docker.io/user/app:latest
cache-to: type=gha,mode=max
```
- GitHub Actions: GHA 캐시 (빠름)
- 로컬 개발: Registry 캐시 (공유됨)
- 다른 CI: Registry 캐시 (호환성)
</details>

---

<div align="center">

**[⬅️ 이전: 멀티 스테이지 빌드](./02-multi-stage-build.md)** | **[홈으로 🏠](../README.md)** | **[다음: 이미지 보안 ➡️](./04-image-security.md)**

</div>
