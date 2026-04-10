# Pipeline 설계 원칙 — Fast Fail과 병렬 실행

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Fast Fail 원칙이 왜 파이프라인 설계의 핵심인가?
- 단계별 게이트(Quality Gate)는 어떻게 구성하고, 무엇이 파이프라인을 멈추는가?
- Job을 병렬화할 때 고려해야 할 의존성과 비용은 무엇인가?
- 피드백 루프(Feedback Loop)가 길어지면 어떤 문제가 발생하는가?
- 테스트, 보안 스캔, 빌드, 배포를 어떤 순서로 배치해야 최적인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파이프라인을 "일단 동작하게" 만드는 것은 쉽다. 하지만 팀이 빠르게 개발하면서도 품질을 유지하는 파이프라인은 설계가 필요하다. 잘못 설계된 파이프라인은 두 가지 극단 중 하나로 빠진다: 너무 느려서 개발자가 기다리다가 집중력을 잃거나, 너무 빠르지만 문제를 잡지 못해 프로덕션 장애로 이어지거나.

Fast Fail과 병렬 실행은 이 두 극단 사이에서 최적의 균형을 찾는 원칙이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# 잘못된 패턴 1: 느린 작업을 먼저 배치
jobs:
  pipeline:
    steps:
      - run: ./gradlew integrationTest   # 10분 (느림)
      - run: ./gradlew test              # 3분
      - run: ./gradlew checkstyleMain    # 30초

# 문제: 간단한 코드 스타일 오류가 있어도
#       10분짜리 통합 테스트를 다 기다려야 알 수 있음
# 개발자 경험: push 후 10분 기다림 → "스타일 오류입니다" → 분노

# 잘못된 패턴 2: 모든 것을 직렬로 연결
jobs:
  lint:
    ...
  unit-test:
    needs: lint       # lint 끝나야 unit-test 시작
  integration-test:
    needs: unit-test  # unit-test 끝나야 integration-test 시작
  security-scan:
    needs: integration-test  # 순서 없이 그냥 직렬
  build:
    needs: security-scan
  deploy:
    needs: build

# 총 시간: 0.5 + 3 + 10 + 5 + 3 + 2 = 23.5분
# lint와 security-scan은 독립적인데 왜 직렬인가?
```

```yaml
# 잘못된 패턴 3: 배포 전 품질 게이트 없음
jobs:
  build-and-deploy:
    steps:
      - run: ./gradlew bootJar    # 테스트 없이 빌드
      - run: docker build -t app .
      - run: kubectl apply -f k8s/deployment.yaml   # 바로 배포
# 코드 품질과 무관하게 빌드만 성공하면 배포
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# 올바른 패턴: 빠른 것 먼저, 독립적인 것은 병렬로
name: Optimized Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # 레이어 1: 빠른 검사 (병렬, 30초~1분)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'gradle' }
      - run: ./gradlew checkstyleMain spotbugsMain

  # 레이어 1: 단위 테스트 (병렬, 2~3분)
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'gradle' }
      - run: ./gradlew test

  # 레이어 1: 보안 스캔 (병렬, 1~2분)
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Secret 스캔
        uses: trufflesecurity/trufflehog@v3
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}

  # 레이어 2: 통합 테스트 (레이어 1 통과 후, 8~10분)
  integration-test:
    needs: [lint, unit-test, security-scan]   # 레이어 1 전체 통과 필수
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'gradle' }
      - run: ./gradlew integrationTest

  # 레이어 3: 빌드 (레이어 2 통과 후, 3분)
  build:
    needs: integration-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'gradle' }
      - run: ./gradlew bootJar
      - uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar

  # 레이어 4: 배포 (빌드 통과 후, 2분)
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production   # 배포 승인 게이트
    if: github.ref == 'refs/heads/main'   # main 브랜치만 배포
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: app-jar }
      - run: echo "배포 진행..."

# 총 시간: max(0.5, 3, 2) + 10 + 3 + 2 = 18분 (병렬 최적화)
# vs 직렬 23.5분 대비 5.5분 단축 (23% 개선)
```

---

## 🔬 내부 동작 원리

### Fast Fail 원칙 — 빠른 피드백의 경제학

Fast Fail의 핵심 통찰: **버그를 늦게 발견할수록 고치는 비용이 기하급수적으로 증가한다.**

```
버그 발견 시점별 수정 비용 (상대적):

개발 중 발견:          비용 1배   (즉시 수정, 컨텍스트 유지)
CI 실패로 발견:         비용 5배   (다른 작업으로 전환 후 복귀)
코드 리뷰에서 발견:     비용 10배  (PR 수정, 재리뷰 요청)
스테이징에서 발견:      비용 50배  (전체 배포 프로세스 반복)
프로덕션에서 발견:      비용 100배 (긴급 대응, 고객 영향, 장애 보고)
```

이것이 "테스트는 느리다"는 불평에도 불구하고 파이프라인에 반드시 넣어야 하는 이유다. 테스트 5분이 프로덕션 장애 수 시간보다 훨씬 저렴하다.

**Fast Fail 구현 원칙:**

1. **가장 빠른 검사를 먼저** — 30초짜리 린트가 10분짜리 통합 테스트보다 먼저 와야 한다
2. **실패 즉시 중단** — 이미 실패한 파이프라인의 후속 단계를 계속 실행하는 것은 낭비다
3. **실패 이유를 명확하게** — "파이프라인 실패"가 아니라 "LoginService 단위 테스트 실패"가 개발자에게 전달되어야 한다

```yaml
# 실패 시 즉시 중단 (기본 동작)
steps:
  - run: ./gradlew test
  # 위 Step 실패 시 아래는 실행 안 됨

# 실패해도 계속 실행 (리포트 업로드 등에 활용)
steps:
  - run: ./gradlew test
    continue-on-error: false   # 기본값

  - uses: actions/upload-artifact@v4
    if: always()               # 테스트 실패해도 리포트 업로드
    with:
      name: test-results
      path: build/reports/
```

### 단계별 게이트(Quality Gate) 설계

게이트는 파이프라인의 "통과 조건"이다. 각 단계가 어떤 조건을 검증하는지 명확히 정의해야 한다.

```
Pipeline Gate 구조:

[코드 게이트] ────────────────────────────
  ✓ 컴파일 성공 (빌드 가능)
  ✓ 코드 스타일 통과 (Checkstyle)
  ✓ 정적 분석 통과 (SpotBugs)
  ✓ Secret 미포함 확인 (TruffleHog)

[테스트 게이트] ───────────────────────────
  ✓ 단위 테스트 100% 통과
  ✓ 코드 커버리지 80% 이상
  ✓ 통합 테스트 통과

[빌드 게이트] ─────────────────────────────
  ✓ Docker 이미지 빌드 성공
  ✓ 이미지 취약점 스캔 통과 (Trivy)

[배포 게이트] ─────────────────────────────
  ✓ 스테이징 스모크 테스트 통과
  ✓ 수동 승인 (프로덕션)
```

```yaml
# GitHub Environment 활용한 배포 게이트
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging           # 환경 정의 (Settings → Environments)
    steps:
      - run: kubectl apply -f k8s/staging/

  smoke-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: 스테이징 스모크 테스트
        run: |
          sleep 30  # 배포 완료 대기
          curl --fail https://staging.myapp.com/health || exit 1
          curl --fail https://staging.myapp.com/api/ping || exit 1

  deploy-production:
    needs: smoke-test
    runs-on: ubuntu-latest
    environment: production        # required_reviewers 설정으로 수동 승인
    steps:
      - run: kubectl apply -f k8s/production/
```

GitHub Environment에서 "Required reviewers" 설정 시, `deploy-production` Job 실행 전 지정된 리뷰어가 승인해야 한다.

### 병렬 실행 최적화 — 의존성 그래프 분석

```
의존성 없는 Job 찾기:
  질문: "이 Job이 다른 Job의 결과물에 의존하는가?"
  
  코드 스타일 검사 → 의존성 없음 (소스만 필요)
  단위 테스트    → 의존성 없음 (소스만 필요)
  보안 스캔      → 의존성 없음 (소스만 필요)
  통합 테스트    → 위 세 개 통과가 전제 (품질 보장 후 실행)
  이미지 빌드    → 테스트 통과 후 의미 있음
  배포           → 이미지 빌드 완료 후

최적 병렬화 구조:
  레이어 1 (병렬): lint | unit-test | security-scan
  레이어 2 (순차): integration-test
  레이어 3 (순차): build
  레이어 4 (순차): deploy
```

**Matrix Strategy — 다중 환경 병렬 테스트:**

```yaml
jobs:
  test:
    strategy:
      matrix:
        java-version: [17, 21]
        os: [ubuntu-latest, windows-latest]
        # 4가지 조합: ubuntu/17, ubuntu/21, windows/17, windows/21
      fail-fast: true    # 하나 실패 시 나머지 즉시 취소

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
      - run: ./gradlew test
```

---

## 💻 실전 실험

### 실험 1: 파이프라인 시간 측정

```yaml
name: 파이프라인 시간 분석

on: workflow_dispatch

jobs:
  # 각 Job의 실행 시간을 측정하려면
  # GitHub Actions UI에서 Job 클릭 → 각 Step 옆에 소요 시간 표시
  
  measure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 단위 테스트 시간 측정
        run: |
          start=$(date +%s)
          ./gradlew test
          end=$(date +%s)
          echo "단위 테스트: $((end-start))초"

      - name: 통합 테스트 시간 측정
        run: |
          start=$(date +%s)
          ./gradlew integrationTest
          end=$(date +%s)
          echo "통합 테스트: $((end-start))초"
```

### 실험 2: 테스트 병렬화로 속도 향상

```yaml
# Gradle 병렬 테스트 실행
jobs:
  parallel-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]    # 4개 Runner에 분산

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'gradle' }

      - name: 테스트 샤드 실행
        run: |
          # 전체 테스트를 4조각으로 분할
          ./gradlew test \
            --tests "*Test*" \
            -PtestShard=${{ matrix.shard }} \
            -PtotalShards=4
```

```kotlin
// build.gradle.kts — 샤드 기반 테스트 분할
val testShard = project.findProperty("testShard")?.toString()?.toInt() ?: 1
val totalShards = project.findProperty("totalShards")?.toString()?.toInt() ?: 1

tasks.test {
    filter {
        // 테스트 클래스를 해시로 분산
        includeTestsMatching { descriptor ->
            Math.abs(descriptor.className.hashCode()) % totalShards == testShard - 1
        }
    }
}
```

### 실험 3: 느린 파이프라인 병목 찾기

```yaml
name: 병목 분석

on: workflow_dispatch

jobs:
  bottleneck-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'gradle' }

      - name: Gradle 빌드 스캔 활성화
        run: |
          # 빌드 스캔으로 각 태스크 소요 시간 분석
          ./gradlew build --scan
          # → 출력된 URL에서 태스크별 시간 확인 가능
```

---

## 📊 성능/비용 비교

### 파이프라인 구조별 총 소요 시간

```
시나리오: lint(1분), unit-test(3분), security(2분), integration(10분), build(3분), deploy(2분)

구조 1 — 완전 직렬:
  1 + 3 + 2 + 10 + 3 + 2 = 21분
  Runner 비용: 21분

구조 2 — 레이어별 병렬:
  max(1, 3, 2) + 10 + 3 + 2 = 18분
  Runner 비용: (1+3+2) + 10 + 3 + 2 = 21분

구조 3 — 최적 병렬 + 캐시:
  max(0.5, 2, 1) + 8 + 2 + 1 = 13.5분  (캐시로 각 단계 50% 단축)
  Runner 비용: 비슷하지만 개발자 대기 시간 35% 절약

핵심 지표: 개발자 피드백 루프 (push 후 결과 알기까지 시간)
  목표: 5분 내 기본 피드백, 15분 내 전체 파이프라인 완료
```

| 파이프라인 특성 | 권장 구조 |
|--------------|---------|
| 단위 테스트만 있음 | 단일 Job, 직렬 |
| 단위 + 통합 테스트 | 2레이어, 테스트 병렬 |
| 전체 CI/CD | 4레이어, 레이어 내 병렬 |
| 다중 언어/환경 | Matrix Strategy |

---

## ⚖️ 트레이드오프

**병렬화의 숨겨진 비용**

Job 병렬화는 Runner 비용을 늘린다. 3개 Job을 병렬로 실행하면 3개 Runner가 동시에 과금된다. 시간은 줄지만 비용은 비슷하거나 약간 늘어날 수 있다. 개발자 생산성 향상(대기 시간 감소)과 Runner 비용 증가를 비교해 결정한다.

**캐시 활용과 파이프라인 복잡도**

캐시를 잘 사용하면 파이프라인을 크게 가속할 수 있지만, 캐시 무효화 로직이 잘못되면 "캐시된 이전 결과로 테스트 통과"하는 상황이 생긴다. 의존성 캐시(Gradle, Maven, npm)는 안전하지만, 테스트 결과 자체를 캐싱하는 것은 위험하다.

**너무 엄격한 게이트**

커버리지 90% 이상, 모든 SpotBugs 경고 0개 같은 과도하게 엄격한 게이트는 파이프라인을 항상 실패시킨다. 팀이 게이트를 무시하거나 우회하는 방법을 찾게 된다. 현실적인 기준을 설정하고 점진적으로 높이는 것이 낫다.

---

## 📌 핵심 정리

- **Fast Fail**: 빠른 검사를 앞에 배치해 개발자가 빠르게 피드백을 받도록 설계한다
- **Quality Gate**: 각 단계를 통과하는 조건을 명확히 정의해 품질을 강제한다
- **병렬 실행**: 의존성 없는 Job은 동시에 실행해 전체 파이프라인 시간을 단축한다
- 최적 순서: `빠른 검사(병렬) → 느린 테스트 → 빌드 → 배포`
- 피드백 루프 목표: **5분 내 기본 피드백, 15분 내 전체 파이프라인 완료**
- 캐시로 의존성 다운로드 시간을 제거하면 각 단계를 50% 이상 가속할 수 있다

---

## 🤔 생각해볼 문제

**Q1.** PR에서는 빠른 피드백을 위해 통합 테스트를 생략하고, main 머지 시에만 통합 테스트를 실행하는 전략의 장단점은 무엇인가?

<details>
<summary>해설 보기</summary>

장점: PR 단계에서 빠른 피드백(단위 테스트만 실행 → 3분), 개발자가 기다리는 시간 감소, PR 리뷰 속도 향상.

단점: 통합 문제가 main 머지 후에 발견된다. 여러 PR이 동시에 머지되면 어느 PR이 통합 테스트를 깼는지 특정이 어렵다. "항상 배포 가능한 상태"라는 CI 원칙이 약해진다.

절충점: PR에서도 통합 테스트를 실행하되, 단위 테스트 실패 시 통합 테스트는 건너뛰는 방식(`needs`로 단위 테스트를 통합 테스트의 전제로)을 사용하면 실패한 PR에서의 낭비를 줄일 수 있다.

</details>

**Q2.** Matrix Strategy로 Java 17/21 × Ubuntu/Windows 4조합 테스트 중 Windows/21 조합만 실패했다. `fail-fast: true`와 `fail-fast: false`일 때 각각 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

`fail-fast: true` (기본값): Windows/21이 실패하면 나머지 3개 조합(아직 실행 중인 것들)을 즉시 취소한다. 빠르게 실패를 인지하고 불필요한 Runner 시간을 줄인다. 단, 다른 조합의 결과를 볼 수 없다.

`fail-fast: false`: Windows/21이 실패해도 나머지 3개 조합은 끝까지 실행된다. 모든 환경에서의 결과를 볼 수 있어 "이 문제가 Windows에만 발생하는가"를 한 번에 확인할 수 있다. Runner 비용은 더 들지만 디버깅에 유리하다.

CI 파이프라인에서는 보통 `fail-fast: true`가 적합하고, 호환성 매트릭스 분석 목적이라면 `fail-fast: false`가 적합하다.

</details>

**Q3.** 파이프라인이 간헐적으로 실패(Flaky)한다. 어떻게 접근해야 하는가?

<details>
<summary>해설 보기</summary>

간헐적 실패의 흔한 원인: 테스트 간 공유 상태(DB, 파일), 타임아웃에 민감한 테스트, 실행 순서에 의존하는 테스트, 네트워크 의존 테스트.

접근 방법:
1. 실패 로그를 수집해 패턴을 찾는다 (특정 테스트, 특정 시간대, 특정 Runner)
2. `continue-on-error: true`로 실패를 기록하면서 개발을 막지 않도록 임시 처리
3. 테스트 격리 — 각 테스트가 독립적인 상태를 사용하도록 수정 (Testcontainers로 DB 격리)
4. `retry-on-failure` 옵션으로 일시적 실패를 자동 재시도 (근본 해결이 아닌 임시방편)
5. 근본 원인 수정 후 `continue-on-error` 제거

</details>

---

<div align="center">

**[⬅️ 이전: GitHub Actions 내부 동작](./03-github-actions-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: GitOps 원칙 — Git이 Single Source of Truth ➡️](./05-gitops-principles.md)**

</div>
