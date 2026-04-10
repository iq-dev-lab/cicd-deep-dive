# Pipeline 성능 최적화 — 병목 단계 식별

## 🎯 핵심 질문

- "우리 CI 파이프라인이 왜 이렇게 오래 걸려요?" — 어디서 시간을 낭비하는가?
- 빌드 내에서 어느 태스크가 가장 오래 걸릴까요?
- 캐시 설정이 정말 효율적일까요? (히트율 확인)
- 테스트를 병렬로 나눠서 실행하면 얼마나 빨라질까요?
- Docker 빌드 캐시는 제대로 작동하고 있나요?

## 🔍 왜 이 개념이 실무에서 중요한가

파이프라인 속도는 **개발 생산성의 직접적인 지표**다. 10분 파이프라인은 1시간 파이프라인과는 완전히 다른 개발 경험을 제공한다.

실제 사례:
- **느린 CI 때문에 배포 망설임**: 검증에 45분 소요 → 급한 버그 수정도 배포 미루기 → 서비스 장애 가능성 증가
- **캐시 미설정**: 매번 npm install에 3분 소요 → 20번 빌드하면 1시간 낭비
- **병렬화 미실시**: 테스트 20분 소요 → 병렬 샤딩으로 5분으로 단축 가능
- **불필요한 스텝**: 모든 빌드마다 Docker 레이어 재빌드 → 캐시 활용으로 1분 단축

**누적 효과:**
팀에 개발자 10명, 하루 평균 5번 빌드 → 하루 50번 빌드 → 느린 파이프라인 = 매달 수백 시간 낭비

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# Before: 최적화가 없는 순수 빌드
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 캐시 없음 → 매번 의존성 재설치
      - name: Install dependencies
        run: npm install
      
      # 캐시 없음 → 매번 빌드 수행
      - name: Build
        run: npm run build
      
      # 캐시 없음 → 매번 Docker 레이어 재빌드
      - name: Build Docker image
        run: docker build -t myapp .
      
      # 순차 테스트 → 병렬화 안 됨
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Run e2e tests
        run: npm run test:e2e
```

**병목:**
```
Total time: 45 minutes
├── npm install: 5 min (매번 동일)
├── npm build: 10 min (캐시 없음)
├── Docker build: 12 min (레이어 재빌드)
├── Unit tests: 8 min (순차 실행)
├── Integration tests: 6 min (순차 실행)
└── E2E tests: 4 min (순차 실행)
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# After: 캐시, 병렬화, 스캔 도구로 최적화
name: Optimized CI

on: [push, pull_request]

env:
  NODE_VERSION: '20'
  GRADLE_OPTS: '-Xmx2g -XX:+HeapDumpOnOutOfMemoryError'

jobs:
  # Stage 1: 빠른 체크만 먼저
  quick-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Lint/Format은 의존성 필요 없음 (가장 빠름)
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Cache node_modules for lint
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint (병렬 실행)
        run: npm run lint -- --max-warnings 0
      
      - name: Format check
        run: npm run format:check

  # Stage 2: 빌드 (lint 성공 후)
  build:
    needs: quick-checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      # npm 캐시 (node_modules 캐시와 분리)
      - name: Cache npm cache directory
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-cache-${{ hashFiles('package-lock.json') }}
      
      - name: Cache node_modules
        uses: actions/cache@v4
        id: node-cache
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        if: steps.node-cache.outputs.cache-hit != 'true'
        run: npm ci --prefer-offline
      
      # Gradle 빌드 스캔으로 성능 분석 (Java 프로젝트 기준)
      - name: Build with Gradle Build Scan
        run: |
          ./gradlew build --scan \
            -x test \
            --build-cache \
            --parallel \
            --max-workers=4
      
      # npm 빌드 캐시 활용
      - name: Build with incremental compilation
        run: npm run build -- --incremental
      
      - name: Cache build output
        uses: actions/cache@v4
        with:
          path: dist/
          key: ${{ runner.os }}-build-${{ github.sha }}

  # Stage 3: 병렬 테스트
  test-unit:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]  # 4개 병렬 샤드
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Restore build cache
        uses: actions/cache@v4
        with:
          path: dist/
          key: ${{ runner.os }}-build-${{ github.sha }}
      
      - name: Restore node_modules
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
      
      # 테스트 샤딩: 전체 테스트를 분산 실행
      - name: Run unit tests (shard ${{ matrix.shard }}/4)
        run: npm test -- --shard=${{ matrix.shard }}/4 --coverage
      
      # 커버리지 리포트 업로드
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-shard-${{ matrix.shard }}
          path: coverage/

  test-integration:
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Restore node_modules
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test

  # Stage 4: Docker 빌드 (캐시 활용)
  docker-build:
    needs: [test-unit, test-integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Docker 빌드 캐시 설정
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      # GitHub Container Registry에 캐시 저장
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      # 여러 플랫폼으로 빌드 (캐시 활용)
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max
          platforms: linux/amd64,linux/arm64

  # Stage 5: 성능 분석 리포트
  performance-report:
    needs: [quick-checks, build, test-unit, test-integration, docker-build]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Collect job timing data
        run: |
          echo "=== Pipeline Performance Report ==="
          echo ""
          echo "Job execution times:"
          # GitHub API로 각 job의 실행 시간 조회
          gh run view ${{ github.run_id }} \
            --json jobs \
            --jq '.jobs[] | "\(.name): \(.completedAt | fromdateiso8601 - .startedAt | fromdateiso8601) seconds"'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Analyze cache efficiency
        run: |
          echo "=== Cache Hit Analysis ==="
          # Cache 로그 분석 (GitHub Actions 로그에서 추출)
          # "Cache hit" 또는 "Cache miss" 카운트
```

## 🔬 내부 동작 원리

### GitHub Actions 캐시 메커니즘

```
Key 생성: hash(파일 내용)
  ↓
캐시 확인: key 존재 여부 확인
  ↓
HIT: 캐시 복원 (빠름)
  또는
MISS: 처음부터 설치 후 캐시 저장
```

**캐시 키 설계:**
```yaml
# 패턴 1: 정적 파일 기반 (권장)
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

# 패턴 2: 날짜 기반 (정기 갱신)
key: ${{ runner.os }}-npm-${{ github.run_id }}-${{ github.run_number }}

# 패턴 3: 복합 (여러 파일)
key: |
  ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}-\
  ${{ hashFiles('*.gradle.kts') }}
```

### Gradle Build Scan 분석

```bash
# Build Scan 활성화 (성능 분석)
./gradlew build --scan

# 출력:
# Build scan published at https://scans.gradle.com/s/XXXX
# 이 URL에서 상세 분석:
# - 각 task 실행 시간
# - 병렬화 효율
# - 캐시 히트율
# - 네트워크 대역폭
```

### 테스트 샤딩(Sharding) 작동 방식

```
전체 테스트 suite: 100개 테스트, 20분 소요

샤딩 없음 (순차):
────────────────────────── (20분)

4-way 샤딩 (병렬):
─────  ─────  ─────  ─────  (5분 × 4 실행 = 20분 벽시간, 하지만 4 job 병렬)
```

**샤딩 전략:**
1. **파일 기반**: 테스트 파일을 분산 (Jest, Vitest)
   ```bash
   npm test -- --shard=1/4 --shardIndex=0
   ```

2. **태그 기반**: @unit, @integration, @e2e로 분류
   ```bash
   npm test -- --tags @unit
   ```

3. **커스텀**: 자체 분류 로직
   ```bash
   # 짝수 인덱스만 실행
   npm test -- --testNamePattern=$(($SHARD % 2 == 0 && "even" || "odd"))
   ```

### Docker 레이어 캐시와 BuildX

```dockerfile
# 레이어 캐시 효율적 작성
FROM node:20-alpine

# Layer 1: 의존성 설치 (자주 안 바뀜)
COPY package*.json ./
RUN npm ci --prefer-offline --no-audit

# Layer 2: 소스 코드 복사 (자주 바뀜)
COPY . .

# Layer 3: 빌드 (소스에 의존)
RUN npm run build

# Layer 4: 실행
CMD ["npm", "start"]
```

**BuildX 캐시 전략:**
```yaml
cache-from: type=registry,ref=ghcr.io/org/repo:buildcache
cache-to: type=registry,ref=ghcr.io/org/repo:buildcache,mode=max
```

각 레이어가 registry에 저장되어 다음 빌드에서 재사용된다.

## 💻 실전 실험 (GitHub Actions YAML, CLI 명령어로 재현 가능)

### 실험 1: 캐시 히트율 측정

```bash
# .github/workflows/cache-analysis.yml
name: Cache Analysis

on:
  schedule:
    - cron: '0 * * * *'  # 매시간

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Cache node_modules
        uses: actions/cache@v4
        id: cache
        with:
          path: node_modules
          key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
      
      # 캐시 히트 여부 출력
      - name: Report cache status
        run: |
          if [ "${{ steps.cache.outputs.cache-hit }}" == "true" ]; then
            echo "✅ Cache HIT: Dependencies restored in milliseconds"
          else
            echo "❌ Cache MISS: Installing dependencies from scratch"
            npm ci --verbose
          fi
      
      # 전체 히트율 분석
      - name: Analyze cache hit rate
        run: |
          gh run list \
            --limit 20 \
            --json startedAt,conclusion \
            --jq '.[] | select(.conclusion == "success") | .startedAt' \
            | head -20 > recent_runs.txt
          
          # 간단한 분석
          wc -l recent_runs.txt
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 실험 2: Step 별 실행 시간 측정

```bash
# 각 step의 실행 시간을 기록
- name: Measure step duration
  id: timing
  run: |
    START=$(date +%s)
    
    # Step 작업 수행
    npm run build
    
    END=$(date +%s)
    DURATION=$((END - START))
    
    echo "build_duration=$DURATION" >> $GITHUB_OUTPUT
    echo "Build completed in ${DURATION}s"

# 결과 사용
- name: Report performance
  run: echo "Build took ${{ steps.timing.outputs.build_duration }}s"
```

### 실험 3: 테스트 샤딩 구현 (Jest 기준)

```bash
# jest.config.js
module.exports = {
  testMatch: ['**/__tests__/**/*.test.js'],
  // 샤딩 설정
  shard: process.env.JEST_SHARD ? {
    index: parseInt(process.env.JEST_SHARD_INDEX || '0'),
    total: parseInt(process.env.JEST_SHARD_TOTAL || '1'),
  } : null,
};

# GitHub Actions에서
- name: Run tests (shard ${{ matrix.shard }}/4)
  env:
    JEST_SHARD_INDEX: ${{ strategy.job-index }}
    JEST_SHARD_TOTAL: 4
  run: npm test
```

### 실험 4: 성능 비교 벤치마크

```bash
# benchmark.yml
name: Performance Benchmark

on:
  workflow_dispatch:
    inputs:
      enable_cache:
        description: Enable caching
        required: true
        type: choice
        options:
          - 'true'
          - 'false'

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      # 조건부 캐시 사용
      - name: Cache dependencies
        if: ${{ github.event.inputs.enable_cache == 'true' }}
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
      
      - name: Record start time
        id: start
        run: echo "start=$(date +%s%3N)" >> $GITHUB_OUTPUT
      
      - name: Install and build
        run: |
          npm ci
          npm run build
      
      - name: Record end time and report
        id: end
        run: |
          END=$(date +%s%3N)
          DURATION=$((END - ${{ steps.start.outputs.start }}))
          echo "Total time: ${DURATION}ms"
          echo "Cache enabled: ${{ github.event.inputs.enable_cache }}"
```

### 실험 5: Docker 빌드 캐시 효율 분석

```bash
# docker-cache-analysis.yml
name: Docker Cache Efficiency

on:
  push:
    branches: [main]

jobs:
  docker-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-buildx-action@v3
      
      - name: Build without cache
        uses: docker/build-push-action@v5
        with:
          context: .
          cache-from: type=local,src=/tmp/.buildx-cache-nocache
          cache-to: type=local,dest=/tmp/.buildx-cache-nocache
          outputs: type=oci,dest=/tmp/image-nocache
      
      - name: Build with cache
        uses: docker/build-push-action@v5
        with:
          context: .
          cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          outputs: type=oci,dest=/tmp/image-cache
      
      # 이미지 크기 비교
      - name: Compare image sizes
        run: |
          SIZE_NOCACHE=$(du -sh /tmp/image-nocache | cut -f1)
          SIZE_CACHE=$(du -sh /tmp/image-cache | cut -f1)
          
          echo "Image without cache: $SIZE_NOCACHE"
          echo "Image with cache: $SIZE_CACHE"
```

## 📊 성능/비용 비교

| 최적화 기법 | 속도 개선 | 구현 난도 | 유지보수 |
|-----------|---------|---------|--------|
| **npm 캐시** | 40-50% | 매우 쉬움 | 거의 없음 |
| **테스트 샤딩** | 60-75% | 중간 | 중간 |
| **Docker 캐시** | 30-60% | 쉬움 | 쉬움 |
| **병렬 Job** | 50-70% | 쉬움 | 쉬움 |
| **Gradle Build Scan** | 10-20% 분석 | 매우 쉬움 | 없음 |

**누적 효과:**
```
최적화 전:     45분
+ npm 캐시:   30분 (33% 개선)
+ 샤딩:       15분 (50% 개선)
+ Docker:     10분 (33% 개선)
= 최적화 후:  10분 (78% 개선)
```

## ⚖️ 트레이드오프

### 캐시 적중률 vs 메모리 사용량

**공격적 캐싱** (모든 파일 캐시):
- 장점: 최고의 속도
- 단점: 저장소 용량 초과 가능

**선택적 캐싱** (주요 파일만):
- 장점: 저장소 효율
- 단점: 약간의 속도 손실

**권장:** package-lock.json과 build 결과물만 캐시.

### 테스트 병렬화 vs 의존성

**무조건 병렬:**
- 장점: 최고의 속도
- 단점: 테스트 간 격리 문제 (DB 상태, 전역 변수)

**선택적 병렬:**
- 장점: 안정성
- 단점: 약간의 속도 손실

**권장:** Unit test는 4-way 병렬, Integration test는 순차.

## 📌 핵심 정리

1. **캐시는 선택이 아닌 필수**: npm install에만 캐시 적용해도 30-40% 시간 절감.

2. **병목을 식별하고 개선**: Gradle Build Scan, GitHub Actions UI로 느린 step 파악.

3. **테스트 병렬화로 극적 개선**: 20분 테스트를 5분으로 줄일 수 있다.

4. **Docker 캐시는 레이어 단위**: Dockerfile 작성 순서가 캐시 효율을 결정.

5. **지속적 모니터링**: 매월 성능 리포트를 생성해 최적화 효과 추적.

## 🤔 생각해볼 문제

### Q1: 캐시 키가 자주 변해서 항상 MISS 되는데, 어떻게 해야 할까요?

<details>
<summary>해설</summary>

캐시 키 설계를 다시 생각하세요:

```yaml
# 나쁜 예: hash가 자주 변함
key: ${{ runner.os }}-npm-${{ hashFiles('src/**/*.js') }}
# → 소스 코드가 바뀔 때마다 MISS

# 좋은 예: 의존성만 hash 하기
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
# → 의존성 변경할 때만 MISS

# 더 좋은 예: fallback 활용
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
restore-keys: |
  ${{ runner.os }}-npm-
# → 정확한 match 없으면 이전 캐시 사용
```

또한 `npm ci` (clean install)를 사용해서 package-lock.json 준수하세요.
</details>

### Q2: 테스트 샤딩을 하는데 일부 shard가 다른 shard보다 훨씬 오래 걸려요.

<details>
<summary>해설</summary>

테스트 샤딩의 불균형은 흔한 문제입니다. 해결책:

```bash
# 1. 각 shard의 실행 시간 측정
for shard in {1..4}; do
  time npm test -- --shard=$shard/4
done

# 2. 테스트 실행 시간 profile 생성
npm test -- --verbose 2>&1 | grep "duration" | sort -t: -k2 -rn

# 3. 느린 테스트를 별도 shard로 이동
# test.js, test.slow.js로 분류
# --testPathPattern으로 분리
```

또는 Jest의 `--bail` 옵션으로 첫 실패 시 중지해서 실행 시간 감소.
</details>

### Q3: 모든 단계를 병렬 실행하면 어떤 문제가 생길까요?

<details>
<summary>해설</summary>

병렬 실행의 문제점:

```yaml
# 문제: build 결과물이 test 단계에서 필요한데, 순서가 없음
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  test:
    # build를 기다려야 함!
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
```

**의존성 그래프:**
```
lint → build → test → deploy
```

각 단계는 이전 단계의 산물에 의존하므로 순서가 필요하다.

**병렬화는 같은 단계 내에서만:**
```
lint (병렬: 코드 스타일, 타입체크)
build (병렬: 불가능, 의존성 있음)
test (병렬: unit/integration/e2e)
deploy (병렬: 불가능, 순서 필요)
```
</details>

---

[⬅️ 이전](./02-pipeline-failure-diagnosis.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./04-environment-management.md)
