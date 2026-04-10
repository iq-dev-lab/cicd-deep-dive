# 파이프라인 장애 진단 — 간헐적 실패의 원인 분석

## 🎯 핵심 질문

- "로컬에서는 성공하는데 CI에서만 실패하는 이유가 뭘까요?"
- 간헐적 실패(Flaky test)는 어떻게 찾아내고 증명할까요?
- 깊은 디버그 로그를 어떻게 활성화할까요?
- 실패한 시점의 Runner 환경을 직접 접속해서 살펴볼 수 있을까요?
- 비슷한 실패 패턴을 분석해서 근본 원인을 찾으려면?

## 🔍 왜 이 개념이 실무에서 중요한가

CI 파이프라인 실패는 배포 블로킹을 의미한다. 특히 **간헐적 실패(Flaky)**는 악순환을 만든다:

실제 사례:
- **간헐적 테스트 실패**: 테스트가 60% 확률로 통과한다. 개발자는 "뭐 이상하긴 한데"하고 재실행한다. 원인은 1년간 미해결.
- **타이밍 문제 미발견**: 로컬에서는 빠른 환경이라 테스트가 통과하지만, CI 환경은 느려서 타이밍 버그가 드러난다.
- **캐시 충돌**: 이전 빌드 캐시가 남아서 다음 빌드가 실패한다. 디버그 로그가 없으면 원인을 모른다.
- **네트워크 장애**: 배포 단계에서 레지스트리 다운 또는 타임아웃. 원인을 알 수 없으면 "그냥 다시 실행하자" 반복.

**문제의 핵심:**
1. 개발자 로컬 환경 ≠ CI 환경
2. 일회성 네트워크 오류 vs 실제 코드 버그 구분 필요
3. Runner 환경을 실시간으로 살펴볼 수 없음

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# Before: 실패하면 그냥 다시 실행
name: CI Pipeline

on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        run: npm test
        
      # 실패했을 때 무엇이 잘못됐는지 알 수 없다
      # 로그가 너무 적어서 디버깅 불가능
```

**문제점:**
1. 일반 로그만 있어서 내부 동작을 알 수 없음
2. 간헐적 실패의 패턴을 분석할 방법이 없음
3. 실패한 Runner에 접속할 수 없음
4. Step별 실행 시간 분석 안 됨
5. 실패 이력 조회 어려움

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# After: 체계적인 디버깅과 실패 패턴 분석
name: CI Pipeline with Diagnostics

on:
  pull_request:
  workflow_dispatch:
    inputs:
      debug_enabled:
        description: Enable debug logging
        required: false
        type: boolean
        default: false

env:
  # 기본 디버그 활성화
  ACTIONS_STEP_DEBUG: ${{ secrets.ACTIONS_STEP_DEBUG || 'false' }}
  ACTIONS_RUNNER_DEBUG: ${{ secrets.ACTIONS_RUNNER_DEBUG || 'false' }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 조건부 디버그 로깅 활성화
      - name: Enable debug logging if needed
        if: github.event.inputs.debug_enabled == 'true'
        run: |
          echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV
          echo "ACTIONS_RUNNER_DEBUG=true" >> $GITHUB_ENV
      
      # 환경 정보 수집 (디버그 용)
      - name: Capture runner environment
        id: env_info
        run: |
          echo "== System Information =="
          uname -a
          echo ""
          echo "== CPU Information =="
          nproc
          echo ""
          echo "== Memory Information =="
          free -h
          echo ""
          echo "== Disk Space =="
          df -h
          echo ""
          echo "== Node.js Version =="
          node --version
          npm --version
      
      - name: Cache dependencies
        uses: actions/cache@v4
        id: cache
        with:
          path: node_modules
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-
      
      # 캐시 히트 여부 기록 (성능 분석용)
      - name: Report cache status
        run: |
          if [ "${{ steps.cache.outputs.cache-hit }}" == "true" ]; then
            echo "Cache HIT: Dependencies restored from cache"
          else
            echo "Cache MISS: Installing fresh dependencies"
          fi
      
      - name: Install dependencies
        run: npm ci
      
      # 테스트를 여러 번 실행해서 간헐적 실패 감지
      - name: Run tests with retry logic
        id: test
        continue-on-error: true
        run: |
          set -e
          
          MAX_RETRIES=3
          for attempt in $(seq 1 $MAX_RETRIES); do
            echo "Test attempt $attempt/$MAX_RETRIES"
            
            if npm test -- --coverage --verbose; then
              echo "Tests passed on attempt $attempt"
              exit 0
            else
              echo "Tests failed on attempt $attempt"
              if [ $attempt -lt $MAX_RETRIES ]; then
                echo "Retrying in 10 seconds..."
                sleep 10
              fi
            fi
          done
          
          exit 1
      
      # 테스트 실패 시 진단 정보 수집
      - name: Collect diagnostics on failure
        if: failure()
        run: |
          echo "=== Test Failure Diagnostics ==="
          echo ""
          echo "=== Process List ==="
          ps aux
          echo ""
          echo "=== Network Connections ==="
          netstat -tuln || ss -tuln
          echo ""
          echo "=== DNS Resolution Test ==="
          nslookup google.com || echo "DNS lookup failed"
          echo ""
          echo "=== Node Process Memory ==="
          ps aux | grep node
          echo ""
          echo "=== Disk Space ==="
          df -h
          echo ""
          echo "=== npm Cache ==="
          npm cache verify --verbose
      
      # tmate SSH를 통한 대화형 디버깅 (선택사항)
      - name: Setup tmate session if test failed
        if: failure() && github.event_name == 'workflow_dispatch'
        uses: mxschmitt/action-tmate@v3
        with:
          limit-access-to-actor: true
          sudo: true
          # tmate에 접속할 수 있는 시간 (분)
          timeout-minutes: 30
      
      # 실패 이력 조회 및 패턴 분석
      - name: Analyze failure patterns
        if: always()
        run: |
          echo "=== Recent Pipeline History ==="
          
          # GitHub CLI로 최근 10개 실행 조회
          gh run list \
            --repo ${{ github.repository }} \
            --workflow ${{ github.workflow }} \
            --limit 10 \
            --json conclusion,name,createdAt,url \
            --template '{{range .}}{{.conclusion}}{{printf "\t"}}{{.createdAt}}{{printf "\t"}}{{.url}}{{printf "\n"}}{{end}}'
          
          # 실패 횟수 계산
          FAILURES=$(gh run list \
            --repo ${{ github.repository }} \
            --workflow ${{ github.workflow }} \
            --limit 10 \
            --json conclusion \
            --template '{{range .}}{{if eq .conclusion "failure"}}1{{end}}{{end}}' | wc -c)
          
          echo ""
          echo "Failure count in last 10 runs: $((FAILURES - 1))"
          
          if [ $((FAILURES - 1)) -ge 3 ]; then
            echo "⚠️  High failure rate detected! This might be a flaky test."
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      # 테스트 커버리지 리포트
      - name: Upload coverage reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 30
      
      # 테스트 결과를 상세히 기록
      - name: Comment test results on PR
        if: always() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            let summary = '## Test Results\n\n';
            
            if (fs.existsSync('coverage/coverage-summary.json')) {
              const coverage = JSON.parse(fs.readFileSync('coverage/coverage-summary.json', 'utf8'));
              const total = coverage.total;
              
              summary += `| Metric | Coverage |\n`;
              summary += `|--------|----------|\n`;
              summary += `| Statements | ${total.statements.pct}% |\n`;
              summary += `| Branches | ${total.branches.pct}% |\n`;
              summary += `| Functions | ${total.functions.pct}% |\n`;
              summary += `| Lines | ${total.lines.pct}% |\n`;
            }
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: summary
            });
```

## 🔬 내부 동작 원리

### GitHub Actions 디버그 모드 작동 방식

GitHub Actions에는 두 가지 디버그 레벨이 있다:

**레벨 1: `ACTIONS_STEP_DEBUG`**
```bash
# Step 레벨의 상세 로그
ACTIONS_STEP_DEBUG=true

# 출력되는 정보:
# - 각 step 시작/종료 시간
# - 환경 변수
# - 네트워크 요청
# - 파일 시스템 작업
```

**레벨 2: `ACTIONS_RUNNER_DEBUG`**
```bash
# Runner 수준의 깊은 디버그 로그
ACTIONS_RUNNER_DEBUG=true

# 출력되는 정보:
# - Runner 초기화 과정
# - 액션 다운로드
# - 컨테이너 생성/삭제
# - 네트워크 연결
```

### 간헐적 실패(Flaky) 원인 분류

```
Flaky Failures
├── Network-related (20-30%)
│   ├── DNS resolution timeout
│   ├── Registry pull rate limit
│   └── Transient connection errors
├── Timing-related (40-50%)
│   ├── Race condition in code
│   ├── Async operation not awaited
│   └── Test execution order dependency
├── Resource-related (15-25%)
│   ├── Insufficient memory
│   ├── Disk space exhaustion
│   └── Port conflicts
└── Cache-related (10%)
    ├── Stale cache
    └── Cache collision
```

### tmate로 실패한 Runner에 직접 접속

tmate는 SSH를 통해 Runner 환경에 실시간 접속을 제공한다.

```yaml
- name: Setup tmate
  uses: mxschmitt/action-tmate@v3
  if: failure()
  with:
    limit-access-to-actor: true  # 본인만 접속 가능
    timeout-minutes: 30
```

실행 후 GitHub Actions 로그에 SSH 접속 주소가 표시된다:
```
SSH: ssh -i <key> ubuntu@<host>
```

실제 Runner에 접속해서:
```bash
# 현재 디렉토리 확인
pwd
ls -la

# 환경 변수 확인
env

# 프로세스 상태 확인
ps aux

# 메모리/디스크 상태
free -h
df -h

# 최근 로그
cat /tmp/github-actions.log

# 테스트 다시 실행
npm test
```

## 💻 실전 실험 (GitHub Actions YAML, CLI 명령어로 재현 가능)

### 실험 1: Flaky 테스트 작성 및 감지

```bash
# 간헐적으로 실패하는 테스트 작성 (재현 가능)

# test/flaky.test.js
describe('Flaky Test Example', () => {
  it('should pass when timing is right', async () => {
    // 50% 확률로 실패하는 테스트
    const random = Math.random();
    
    if (random < 0.5) {
      // 비동기 작업 완료를 기다리지 않음 (타이밍 버그)
      setTimeout(() => {
        expect(true).toBe(true);
      }, 100);
    } else {
      expect(true).toBe(true);
    }
  });
  
  it('should fail intermittently with network', async () => {
    // 네트워크 타임아웃 시뮬레이션
    const response = await fetch('http://api.example.com/test', {
      timeout: 100  // 매우 짧은 타임아웃
    });
    expect(response.status).toBe(200);
  });
});
```

### 실험 2: 디버그 Secret 설정 및 활성화

```bash
# Repository Settings → Secrets and variables → Actions

# Secret 추가
ACTIONS_STEP_DEBUG=true
ACTIONS_RUNNER_DEBUG=true

# 또는 workflow에서 직접 설정
name: Debug Test

env:
  ACTIONS_STEP_DEBUG: true
  ACTIONS_RUNNER_DEBUG: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Debug logging is enabled"
```

### 실험 3: 실패 패턴 분석 CLI

```bash
# GitHub CLI로 최근 실패 분석
gh run list \
  --repo owner/repo \
  --workflow test.yml \
  --limit 50 \
  --json conclusion,createdAt,durationMinutes \
  --query '.[] | select(.conclusion=="failure")'

# 출력 예시:
# conclusion  createdAt             durationMinutes
# failure     2024-01-15T10:30:00Z  5
# failure     2024-01-14T14:22:00Z  7
# failure     2024-01-13T16:45:00Z  5

# 특정 workflow 실행의 로그 다운로드
gh run download <run-id> --dir ./logs

# 로그에서 "error" 또는 "timeout" 검색
grep -r "timeout\|error" logs/
```

### 실험 4: 캐시 히트율 분석

```yaml
# .github/workflows/cache-analytics.yml
name: Cache Hit Analysis

on:
  schedule:
    - cron: '0 0 * * *'  # 매일 자정

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Analyze recent cache performance
        run: |
          echo "=== Cache Hit Analysis (Last 30 runs) ==="
          
          gh run list \
            --limit 30 \
            --json conclusion,durationMinutes,createdAt \
            | jq '.[] | 
              {
                date: .createdAt,
                duration: .durationMinutes,
                passed: (.conclusion == "success")
              }' > cache-stats.json
          
          # Python으로 분석
          python3 << 'EOF'
          import json
          
          with open('cache-stats.json') as f:
            data = [json.loads(line) for line in f if line.strip()]
          
          total = len(data)
          passed = sum(1 for d in data if d['passed'])
          avg_duration = sum(d['duration'] for d in data) / total
          
          print(f"Success rate: {passed/total*100:.1f}%")
          print(f"Average duration: {avg_duration:.1f} min")
          
          # 실패 경우 분석
          failures = [d for d in data if not d['passed']]
          if failures:
              print(f"Failure count: {len(failures)}")
          EOF
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 실험 5: Network 타임아웃 감지

```bash
# network-timeout.yml
name: Detect Network Issues

on:
  pull_request:

jobs:
  network_test:
    runs-on: ubuntu-latest
    steps:
      - name: Test DNS resolution
        run: |
          # 10초 타임아웃으로 DNS 테스트
          timeout 10 nslookup registry.npmjs.org || {
            echo "DNS resolution failed"
            exit 1
          }
      
      - name: Test registry connectivity
        run: |
          # npm registry 핑 테스트
          npm ping || echo "Registry unreachable"
      
      - name: Test with retry logic
        run: |
          MAX_RETRIES=3
          
          for i in $(seq 1 $MAX_RETRIES); do
            if npm install; then
              echo "Install succeeded on attempt $i"
              exit 0
            fi
            
            if [ $i -lt $MAX_RETRIES ]; then
              WAIT=$((2 ** i))
              echo "Attempt $i failed, waiting ${WAIT}s..."
              sleep $WAIT
            fi
          done
          
          echo "Install failed after $MAX_RETRIES attempts"
          exit 1
```

## 📊 성능/비용 비교

| 방법 | 디버그 정보량 | 실행 시간 | 비용 | 학습곡선 |
|-----|-------------|---------|------|---------|
| **기본 로그** | 낮음 | 빠름 | 무료 | 쉬움 |
| **STEP_DEBUG** | 중간 | 보통 | 무료 | 쉬움 |
| **RUNNER_DEBUG** | 매우 높음 | 느림 | 무료 | 중간 |
| **tmate SSH** | 매우 높음 | 수동 | 무료 | 높음 |
| **Artifact 업로드** | 중간 | 보통 | 무료 | 쉬움 |

**권장 조합:**
- 개발 중: STEP_DEBUG + Artifact
- 프로덕션: 기본 로그 + 실패 시 진단 Job
- 긴급 디버깅: tmate SSH

## ⚖️ 트레이드오프

### 항상 디버그 활성화 vs 필요할 때만

**항상 활성화:**
- 장점: 실패 시 즉시 디버그 정보 확보
- 단점: 로그가 너무 커서 읽기 어려움, 스토리지 비용

**필요할 때만:**
- 장점: 깔끔한 로그, 비용 절감
- 단점: 초기 실패 후 다시 실행해야 함

**권장:** Secret `ACTIONS_STEP_DEBUG`를 기본 false로 두고, 필요할 때 `true`로 변경한다.

### 재시도 vs 즉시 실패

**재시도 3회:**
- 장점: 간헐적 실패 극복
- 단점: 총 실행 시간 3배, 비용 증가

**즉시 실패:**
- 장점: 빠른 피드백
- 단점: 네트워크 일시 오류에도 실패

**권장:** 네트워크 작업(npm install, docker pull)은 재시도, 테스트는 재시도 안 함.

## 📌 핵심 정리

1. **두 가지 디버그 모드**: STEP_DEBUG는 CI 환경별 세부 정보, RUNNER_DEBUG는 그 이상.

2. **Flaky 테스트를 체계적으로 추적**: 최근 실행 이력에서 실패율을 계산해 패턴 파악.

3. **실패 시 자동 진단**: 네트워크, 메모리, 디스크 상태를 자동으로 수집.

4. **tmate로 실시간 디버깅**: SSH 접속으로 실패한 Runner 환경을 직접 조사.

5. **실패 이력과 패턴 분석**: 같은 Step에서 반복 실패하면 구조적 문제일 가능성.

## 🤔 생각해볼 문제

### Q1: "로컬에서는 성공하는데 CI만 실패"하는 원인은?

<details>
<summary>해설</summary>

다음 차이점을 확인하세요:

```bash
# 1. 환경 변수 비교
# 로컬
env | grep -E "NODE|NPM|PATH"

# CI에서
- name: Compare environments
  run: env | grep -E "NODE|NPM|PATH"

# 2. Node.js 버전
# 로컬
node --version

# CI에서
- name: Check Node version
  run: node --version

# 3. 운영체제
# 로컬: macOS, Windows, Linux 혼재
# CI: 지정된 runner (ubuntu-latest)

# 4. 캐시 상태
# 로컬: node_modules 영속 존재
# CI: 매번 초기화 가능

# 5. 네트워크
# 로컬: 신뢰할 수 있는 인터넷
# CI: 레이트 제한, 방화벽 있을 수 있음
```

해결책:
```yaml
- name: Match CI environment locally
  run: |
    # Docker로 ubuntu 환경 시뮬레이션
    docker run -it ubuntu:latest bash
    # 안에서 테스트 실행
```
</details>

### Q2: 간헐적 실패를 한 번에 감지할 수 있을까요?

<details>
<summary>해설</summary>

테스트를 여러 번 반복 실행해서 간헐적 실패를 감지하세요:

```bash
# 테스트를 10번 실행
for i in {1..10}; do
  npm test || echo "Failed on run $i"
done | grep "Failed" | wc -l
```

또는 GitHub Actions에서:

```yaml
- name: Repeat tests to find flakiness
  run: |
    FAILURES=0
    TOTAL_RUNS=10
    
    for i in $(seq 1 $TOTAL_RUNS); do
      if ! npm test; then
        FAILURES=$((FAILURES + 1))
      fi
    done
    
    FLAKINESS=$((FAILURES * 100 / TOTAL_RUNS))
    echo "Flakiness: $FLAKINESS% ($FAILURES/$TOTAL_RUNS failures)"
    
    if [ $FAILURES -gt 0 ]; then
      echo "::warning::Flaky test detected!"
    fi
```

하지만 이 방법은 CI 시간과 비용을 크게 증가시킨다. 더 나은 방법은:

```yaml
# 매일 밤 flaky test 탐지 workflow 실행
schedule:
  - cron: '0 2 * * *'

# 그 안에서 테스트를 50번 반복 실행
```
</details>

### Q3: 배포 직전 단계에서만 디버그를 활성화하려면?

<details>
<summary>해설</summary>

특정 branch나 tag에서만 디버그 활성화:

```yaml
env:
  # main 브랜치일 때만 디버그 활성화
  ACTIONS_STEP_DEBUG: ${{ github.ref == 'refs/heads/main' && 'true' || 'false' }}

# 또는 수동으로 선택
on:
  workflow_dispatch:
    inputs:
      debug_mode:
        description: Enable debug
        required: false
        type: choice
        options:
          - 'false'
          - 'true'
        default: 'false'

env:
  ACTIONS_STEP_DEBUG: ${{ github.event.inputs.debug_mode }}
```

이렇게 하면 필요할 때만 상세 로그를 남길 수 있다.
</details>

---

[⬅️ 이전](./01-deployment-tracking.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./03-pipeline-performance-optimization.md)
