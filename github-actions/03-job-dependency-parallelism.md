# Job 의존성과 병렬화 — DAG와 Matrix Strategy

## 🎯 핵심 질문

`needs:` 키워드는 Job 간 의존성을 어떻게 구성하고, GitHub는 어떻게 병렬로 실행할 Job을 결정할까? Matrix Strategy로 여러 환경을 동시에 테스트할 때 내부적으로 몇 개의 Job이 생성되는가? 순환 의존성이 발생하면 어떻게 될까? Job outputs를 다음 Job에 전달하는 패턴은?

## 🔍 왜 이 개념이 실무에서 중요한가

- **CI/CD 속도**: Job 간 의존성을 올바르게 설정하면 병렬 처리로 전체 워크플로우 시간을 절반 이상 줄일 수 있다.
- **비용 최적화**: 불필요한 Job 실행을 피할 수 있다 (앞단 테스트 실패 시 배포 Job 스킵).
- **복잡한 테스트**: Matrix Strategy로 10개의 환경조합을 테스트하는 것이 10배 비싼가, 아니면 병렬이라 같은 비용인가?
- **배포 안정성**: 여러 빌드 Job이 성공했을 때만 배포하도록 구성해야 한다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 예 1: 모든 Job을 순차적으로 실행
name: Sequential Jobs
on: [push]
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  test-integration:
    runs-on: ubuntu-latest
    needs: [test-unit]  # 꼭 필요 없는데 needs 추가
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:integration

  test-e2e:
    runs-on: ubuntu-latest
    needs: [test-integration]
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e

  deploy:
    runs-on: ubuntu-latest
    needs: [test-e2e]
    steps:
      - run: echo "Deploying"
  # 총 시간: unit(5분) + integration(10분) + e2e(15분) + deploy(2분) = 32분
  # 실제로는 unit, integration, e2e가 병렬 가능하면 15분 가능

# ❌ 잘못된 예 2: Matrix 전체 실패로 Job 스킵
name: Bad Matrix Handling
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [14, 16, 18]
    fail-fast: true  # 하나 실패 시 전체 취소
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test

  deploy:
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
  # Node 16에서 실패하면 Node 18 테스트를 안 하고 배포도 못 함

# ❌ 잘못된 예 3: outputs 접근 오류
name: Wrong Output Access
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
    steps:
      - run: echo "version=1.0.0" >> $GITHUB_OUTPUT

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Version: ${{ needs.build.outputs.version }}"
        # Matrix Job의 outputs에 접근 불가
        # (2개의 Job이 생성됐는데, 어느 Job의 output을 쓸지 모호)
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 예 1: 병렬 가능한 Job은 병렬로 구성
name: Parallel Jobs
on: [push]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  test-unit:
    runs-on: ubuntu-latest
    # lint과 병렬 실행
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  test-integration:
    runs-on: ubuntu-latest
    # lint, test-unit과 병렬 실행
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:integration

  test-e2e:
    needs: [test-unit, test-integration]  # 단위테스트 먼저
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e

  deploy:
    needs: [lint, test-e2e]  # 모든 테스트 통과 필수
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
  # 최적화된 시간: max(5, 5, 10) + 15 + 2 = 22분 (순차 32분 vs 병렬 22분)

# ✅ 올바른 예 2: 독립적 테스트는 fail-fast 사용 금지
name: Matrix with fail-fast control
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [14, 16, 18, 20]
      fail-fast: false  # 모든 버전을 다 테스트
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test

  deploy:
    needs: [test]
    if: success()  # 모든 matrix job 성공했을 때만
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"

# ✅ 올바른 예 3: Matrix Job 간 선택적 outputs 사용
name: Conditional Deployment
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-latest]  # 배포는 ubuntu에서만
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - id: version
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT

  deploy:
    needs: [build]
    if: always()  # build의 결과와 관계없이 실행 가능하게
    runs-on: ubuntu-latest
    steps:
      - run: echo "Version: ${{ needs.build.outputs.version }}"
```

## 🔬 내부 동작 원리

### 1. Job DAG(방향 비순환 그래프) 구성

GitHub Actions는 `needs:` 키워드로 Job 간의 의존성을 DAG로 구성합니다:

```
needs: []          needs: [A]          needs: [A, B]
    A                  B                    C
   / \                 |                   / \
  B   C                C                  D   E
  |   |                |
  D   E                D
```

**DAG 검증:**
```
1. 모든 job의 needs 키워드 수집
2. 순환 의존성(Cycle) 검사
   - A → B → A (불가)
   - A → B → C → A (불가)
3. 위상 정렬(Topological Sort)
   - 의존성이 없는 Job부터 실행 가능 표시
4. 병렬 가능한 Job 식별
```

**예시:**
```yaml
jobs:
  A: { }            # level 0: 즉시 실행
  B: { needs: [A] } # level 1: A 완료 후
  C: { needs: [A] } # level 1: A 완료 후 (B와 병렬)
  D: { needs: [B, C] }  # level 2: B, C 모두 완료 후
```

### 2. Runner 할당 및 병렬 실행

**Runner 풀 관리:**
```
Request: 
  A (ubuntu-latest) → 할당 가능한 ubuntu Runner 찾음
  B (ubuntu-latest) → 할당 가능한 ubuntu Runner 찾음 (다른 Runner)
  C (ubuntu-latest) → 할당 가능한 ubuntu Runner 찾음 (또 다른 Runner)

Result:
  Runner #1: A 실행
  Runner #2: B 실행 (A와 병렬)
  Runner #3: C 실행 (A, B와 병렬)
```

**자원 제약:**
- GitHub-hosted: 무제한 Runner (계정 플랜별 동시 Job 수 제한)
- Self-hosted: 유한한 Runner (설정된 수만큼만 동시 실행)

### 3. Matrix Strategy의 내부 동작

Matrix는 **여러 Job을 생성**합니다:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node-version: [16, 18, 20]
```

**생성되는 Job:**
```
test[ubuntu-latest, 16]
test[ubuntu-latest, 18]
test[ubuntu-latest, 20]
test[macos-latest, 16]
test[macos-latest, 18]
test[macos-latest, 20]
test[windows-latest, 16]
test[windows-latest, 18]
test[windows-latest, 20]

총 9개의 Job 생성 (3 × 3 = 9)
```

**Runner 할당:**
```
ubuntu-latest Job들: ubuntu Runner 풀에서 할당
macos-latest Job들: macOS Runner 풀에서 할당
windows-latest Job들: Windows Runner 풀에서 할당
```

**fail-fast 동작:**
```
fail-fast: true (기본)
  - ubuntu[16] 실패 → 모든 ubuntu 남은 Job 취소
  - macos[16], windows[16] 계속 실행 (다른 OS는 독립적)

fail-fast: false
  - 모든 9개 Job 완료될 때까지 실행
  - 실패 여부와 관계없이 진행
```

### 4. Job Output과 Artifact 전달

**같은 Job 내 Step 간:**
```yaml
steps:
  - id: build
    run: echo "version=1.0.0" >> $GITHUB_OUTPUT
  
  - run: echo "Build version: ${{ steps.build.outputs.version }}"
```

**다른 Job 간 (needs 사용):**
```yaml
jobs:
  build:
    steps:
      - id: version
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT
    outputs:
      app-version: ${{ steps.version.outputs.version }}

  deploy:
    needs: [build]
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.app-version }}"
```

**Matrix Job의 outputs:**
```yaml
build:
  strategy:
    matrix:
      node-version: [16, 18]
  outputs:
    # ❌ outputs는 Matrix Job에서 사용 불가
    # 여러 Job이 생성되므로 어느 Job의 output을 쓸지 모호

  # ✅ 대신 Artifact 사용:
  steps:
    - run: echo "artifact" > output.txt
    - uses: actions/upload-artifact@v3
      with:
        name: build-${{ matrix.node-version }}
        path: output.txt

deploy:
  needs: [build]
  steps:
    - uses: actions/download-artifact@v3
      with:
        name: build-16  # 특정 버전의 artifact 다운로드
```

### 5. Job 상태와 조건부 실행

**Job 상태 전파:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  test:
    needs: [build]
    if: success()  # build 성공했을 때만
    steps:
      - run: npm test

  deploy:
    needs: [build, test]
    if: always()  # test 실패해도 실행 가능
    steps:
      - run: ./deploy.sh
```

**가능한 상태:**
- `success()`: 이전 Job 성공
- `failure()`: 이전 Job 실패
- `always()`: 항상 (성공/실패 무관)
- `cancelled()`: 취소됨
- `github.ref == 'refs/heads/main'`: 조건식 결합 가능

## 💻 실전 실험 (GitHub Actions YAML 재현 가능한 예시)

### 실험 1: Job 의존성과 병렬화

```yaml
name: Job Dependency Test
on: [push]

jobs:
  # Level 0: 병렬 가능
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Linting..." && sleep 2

  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Unit tests..." && sleep 5

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Integration tests..." && sleep 3

  # Level 1: lint, test-unit, test-integration 완료 후
  build:
    needs: [lint, test-unit]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Building..." && sleep 2
    outputs:
      build-id: ${{ steps.build.outputs.id }}

  # Level 1: test-integration 완료 후 (build와 병렬 가능)
  test-e2e:
    needs: [test-integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "E2E tests..." && sleep 4

  # Level 2: build, test-e2e 완료 후
  deploy:
    needs: [build, test-e2e]
    if: success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying with build: ${{ needs.build.outputs.build-id }}"

# 실행 순서:
# T=0s: lint, test-unit, test-integration 시작 (병렬)
# T=5s: lint(T=2s), test-integration(T=3s) 완료 → build, test-e2e 시작 (병렬)
# T=7s: test-unit(T=5s) 완료 (build는 이미 lint, test-unit 완료했으므로 무시)
# T=9s: build(T=2s) 완료
# T=9s: test-e2e(T=4s) 완료
# T=9s: deploy 시작 (build, test-e2e 모두 완료)
# T=11s: deploy 완료
# 총 시간: ~11초 (순차 16초 vs 병렬 11초)
```

### 실험 2: Matrix Strategy와 fail-fast

```yaml
name: Matrix Test
on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node-version: [16, 18, 20]
      fail-fast: false  # 모든 조합 테스트

    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - run: echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node-version }}"
      
      - run: |
          # 의도적으로 일부 실패
          if [[ "${{ matrix.os }}" == "macos-latest" && "${{ matrix.node-version }}" == "16" ]]; then
            echo "Node 16 on macOS is not supported"
            exit 1
          fi
          npm test

  # 6개 Job 중 1개 실패해도 deploy 진행
  deploy:
    needs: [test]
    if: success()  # 모든 test Job 성공했을 때
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"

# 생성되는 Job:
# test[ubuntu-latest, 16]
# test[ubuntu-latest, 18]
# test[ubuntu-latest, 20]
# test[macos-latest, 16]  ← 실패
# test[macos-latest, 18]
# test[macos-latest, 20]
#
# fail-fast: false이므로 모두 실행됨
# deploy는 하나 실패했으므로 스킵
```

### 실험 3: outputs와 다른 Job 연결

```yaml
name: Job Output Test
on: [push]

jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      app-version: ${{ steps.version.outputs.version }}
      build-date: ${{ steps.version.outputs.date }}
    steps:
      - id: version
        run: |
          VERSION=$(date +%Y.%m.%d)
          DATE=$(date -u)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "date=$DATE" >> $GITHUB_OUTPUT

  build:
    needs: [version]
    runs-on: ubuntu-latest
    outputs:
      artifact-url: ${{ steps.upload.outputs.url }}
    steps:
      - run: echo "Building version: ${{ needs.version.outputs.app-version }}"
      
      - id: upload
        run: echo "url=s3://bucket/app-${{ needs.version.outputs.app-version }}.tar.gz" >> $GITHUB_OUTPUT

  deploy:
    needs: [version, build]
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Version: ${{ needs.version.outputs.app-version }}"
          echo "Built on: ${{ needs.version.outputs.build-date }}"
          echo "Artifact: ${{ needs.build.outputs.artifact-url }}"

  # 의존성 체인:
  # version (outputs: app-version, build-date)
  #    ↓
  # build (needs: version, outputs: artifact-url)
  #    ↓
  # deploy (needs: version, build)
```

### 실험 4: 복잡한 DAG 구성

```yaml
name: Complex DAG
on: [push]

jobs:
  # Layer 0
  checkout:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

  # Layer 1 (checkout과 병렬)
  lint:
    needs: [checkout]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Linting"

  test-unit:
    needs: [checkout]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Unit tests"

  test-integration:
    needs: [checkout]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Integration tests"

  # Layer 2 (lint, test-unit, test-integration 완료 후)
  build:
    needs: [lint, test-unit]
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.build.outputs.version }}
    steps:
      - id: build
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT

  analyze-coverage:
    needs: [test-unit, test-integration]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Coverage analysis"

  # Layer 3
  docker-build:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Docker build v${{ needs.build.outputs.version }}"

  # Layer 3 (build, analyze-coverage와 병렬)
  e2e-test:
    needs: [build, test-integration]
    runs-on: ubuntu-latest
    steps:
      - run: echo "E2E test"

  # Layer 4
  deploy:
    needs: [docker-build, e2e-test, analyze-coverage]
    if: success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

## 📊 성능/비용 비교

| 전략 | 실행 시간 | 비용 | 사용 시기 |
|------|---------|------|---------|
| **모두 직렬** | 5+10+15+2 = 32분 | 32 Runner분 | 자원 제약 있을 때 |
| **병렬 (의존성만)** | max(5,10,15)+2 = 17분 | 51 Runner분 (3개 동시) | 일반적 권장 |
| **Matrix (3개 환경)** | 5분 × 3 = 15분 | 45 Runner분 (3개 동시) | 테스트 다양화 |
| **Matrix + fail-fast** | 5분 이하 (조기 실패) | 15-45 Runner분 | CI 빠른 피드백 필요 |

**비용 계산:**
- GitHub-hosted Runner: $0.005 / 분
- 병렬 3개: 15분 × 3개 = 45분 × $0.005 = $0.225
- 직렬: 32분 × $0.005 = $0.16

**결론:** 시간은 47% 단축, 비용은 40% 증가 (대부분의 프로젝트에서 시간 가치 > 비용 증가분)

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 |
|------|------|------|
| **needs로 직렬화** | 안정적, 저비용 | 느림, 중단 단계 명확 |
| **병렬 실행** | 빠름, CI 피드백 빠름 | 자원 사용 많음, 복잡도 증가 |
| **Matrix 전체 테스트** | 포괄적 테스트 | 느림, 비쌈, 실패율 높음 |
| **fail-fast: true** | 빠른 피드백 | 일부 환경 테스트 안 할 수 있음 |
| **fail-fast: false** | 모든 환경 테스트 | 실패해도 모두 실행, 시간 낭비 |

## 📌 핵심 정리

1. **DAG 구성**: `needs:`로 Job 간 의존성을 명시 → GitHub이 병렬 가능한 Job 식별
2. **Runner 할당**: 의존성 없는 Job들은 자동 병렬 실행 (Runner 수만큼)
3. **Matrix 생성**: `matrix:` 키워드는 조합 수만큼 Job 생성 (3×3 = 9개 Job)
4. **fail-fast 제어**: `fail-fast: true` (빠른 피드백) vs `false` (포괄 테스트)
5. **Output 전달**: Matrix Job은 outputs 불가 → Artifact 사용
6. **조건부 실행**: `if:` 조건으로 이전 Job 상태 기반 결정

## 🤔 생각해볼 문제

### 문제 1: 이 DAG에서 총 실행 시간은?

```yaml
jobs:
  A: # 5분
  B: needs: [A]  # 3분
  C: needs: [A]  # 4분
  D: needs: [B, C]  # 2분
  E: needs: [D]  # 1분
```

<details>
<summary>해설</summary>

**위상 정렬:**
```
Layer 0: A (5분)
Layer 1: B (3분), C (4분) - 병렬, 최대 4분
Layer 2: D (2분) - B, C 완료 후
Layer 3: E (1분) - D 완료 후

총 시간: 5 + 4 + 2 + 1 = 12분

시각화:
T=0   T=5   T=9   T=11  T=12
|--A--|  B(3)  |--D--|--E--|
       |---C(4)---|
```

**순차 시간:** 5+3+4+2+1 = 15분
**병렬 시간:** 12분 (20% 단축)
</details>

### 문제 2: Matrix로 6개 Job이 생성되는데, 1개 실패 시 deploy는?

```yaml
test:
  strategy:
    matrix:
      os: [ubuntu, macos]
      version: [16, 18, 20]
  fail-fast: false
  steps:
    - run: npm test

deploy:
  needs: [test]
  if: success()
  steps:
    - run: ./deploy.sh
```

<details>
<summary>해설</summary>

**생성되는 Job:**
- test[ubuntu, 16]
- test[ubuntu, 18]
- test[ubuntu, 20]
- test[macos, 16]
- test[macos, 18] ← 실패
- test[macos, 20]

**deploy 실행 여부:**

`needs: [test]`는 Job 이름이므로, 실제로는:
- 모든 test[*] Job의 결합 상태 확인
- `success()`: **모든 Job이 성공했을 때만**
- 1개라도 실패 → deploy 스킵

**결과:** deploy는 **스킵됨** (test[macos, 18] 실패)

**해결책:**
```yaml
deploy:
  needs: [test]
  if: always()  # 또는 실패해도 진행
  steps:
    - run: ./deploy.sh
```

또는 실패한 Job만 제외:
```yaml
deploy:
  needs: [test]
  if: |
    !contains(needs.test.result, 'failure')
  steps:
    - run: ./deploy.sh
```
</details>

### 문제 3: 다음 구조에서 outputs는 작동할까?

```yaml
build:
  strategy:
    matrix:
      node-version: [16, 18]
  outputs:
    version: ${{ steps.version.outputs.version }}
  steps:
    - id: version
      run: echo "version=${{ matrix.node-version }}" >> $GITHUB_OUTPUT

deploy:
  needs: [build]
  steps:
    - run: echo "Version: ${{ needs.build.outputs.version }}"
```

<details>
<summary>해설</summary>

**답: 작동하지만 예측 불가능**

**문제:**
- build[16]과 build[18] 2개 Job 생성
- 각각 version output 설정: "16", "18"
- deploy는 어느 값을 받을까? **Last to complete**

GitHub의 동작:
```
build[16] 완료: version=16
build[18] 완료: version=18 (마지막 완료)

deploy 실행: version=18 (마지막 값)
```

**권장 방법:**
```yaml
build:
  strategy:
    matrix:
      node-version: [16, 18]
  steps:
    - id: version
      run: |
        VERSION=${{ matrix.node-version }}
        echo "version=$VERSION" >> $GITHUB_OUTPUT
    
    - uses: actions/upload-artifact@v3
      with:
        name: version-${{ matrix.node-version }}
        path: version.txt

deploy:
  needs: [build]
  steps:
    - uses: actions/download-artifact@v3
      with:
        name: version-18  # 명시적으로 선택
```

또는 Aggregate Job:
```yaml
aggregate-versions:
  needs: [build]
  runs-on: ubuntu-latest
  outputs:
    all-versions: ${{ steps.collect.outputs.versions }}
  steps:
    - uses: actions/download-artifact@v3
    
    - id: collect
      run: echo "versions=$(ls -1 version-* | tr '\n' ',')" >> $GITHUB_OUTPUT

deploy:
  needs: [aggregate-versions]
  steps:
    - run: echo "All versions: ${{ needs.aggregate-versions.outputs.all-versions }}"
```
</details>

---

<div align="center">
**[⬅️ 이전: Actions 내부 동작](./02-actions-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Secrets와 환경 변수 ➡️](./04-secrets-env-oidc.md)**
</div>
