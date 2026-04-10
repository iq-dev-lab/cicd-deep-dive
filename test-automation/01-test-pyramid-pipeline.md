# 테스트 피라미드와 Pipeline — 어디에 배치할 것인가

## 🎯 핵심 질문

- 단위 테스트, 통합 테스트, E2E 테스트를 파이프라인의 어느 단계에 배치해야 할까?
- 전체 테스트 시간을 단축하면서도 품질을 유지하려면?
- 간헐적 실패(Flaky Test)는 왜 발생하고 어떻게 격리할까?
- 병렬 실행으로 파이프라인 실행 시간을 10분 이상 단축할 수 있을까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

테스트 피라미드는 개발 생산성과 품질 보증 사이의 균형을 결정합니다.

**빠른 피드백 루프**: PR 제출 후 5분 내에 피드백을 받으면 개발자는 즉시 수정할 수 있습니다. 30분 후 피드백을 받으면 이미 다른 작업을 시작했을 가능성이 높고, 컨텍스트 스위칭 비용이 발생합니다.

**인프라 비용**: 모든 테스트를 동시에 실행하면 CI/CD 빌드 시간은 짧지만 병렬 실행 리소스 비용이 급증합니다. 선택적 배치 실행으로 평균 빌드 비용 30% 절감 가능합니다.

**품질 게이트 신뢰성**: E2E 테스트가 간헐적으로 실패하면 "모르겠으니 다시 실행"이 반복되어, 진정한 버그를 놓치게 됩니다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 파이프라인: 모든 테스트를 순차 실행
name: Test Pipeline (Bad)
on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - name: Build
        run: ./gradlew clean build
      
      # 문제점 1: 단위 테스트가 끝날 때까지 통합 테스트 대기
      - name: Unit Tests
        run: ./gradlew test
        timeout-minutes: 15  # 이미 느린데 순차 실행
      
      # 문제점 2: 모든 통합 테스트 순차 실행
      - name: Integration Tests
        run: ./gradlew integrationTest
        timeout-minutes: 30  # 너무 길다!
      
      # 문제점 3: E2E 테스트가 불안정한 네트워크 환경에서 간헐적 실패
      - name: E2E Tests
        run: ./gradlew e2eTest
        timeout-minutes: 45  # 파이프라인이 1시간 이상 소요
```

**결과**: 파이프라인이 60분 이상 소요되므로:
- 개발자가 PR 피드백을 받기 전 다른 작업으로 이동
- "flaky" 테스트가 여러 번 재실행되어 전체 시간 2배 증가
- CI/CD 병목으로 인한 배포 지연

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 파이프라인: 테스트 피라미드와 점진적 실행

name: Test Pyramid Pipeline (Good)
on: [pull_request, push]

jobs:
  # 스테이지 1: 빠른 피드백 (5분 이내)
  fast-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - name: Setup Build Cache
        uses: actions/cache@v3
        with:
          path: ~/.gradle/caches
          key: gradle-${{ hashFiles('**/*.gradle*') }}
      
      # 단위 테스트만 병렬 실행 (Matrix Strategy)
      - name: Unit Tests (Parallel)
        run: ./gradlew test --parallel --max-workers=4
        timeout-minutes: 10

  # 스테이지 2: 통합 테스트 (단위 테스트 통과 후 실행)
  integration-tests:
    needs: fast-tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        # Sharding: 통합 테스트를 3개 병렬 작업으로 분산
        shard: [1, 2, 3]
    services:
      # 이 작업에서만 PostgreSQL 컨테이너 실행 (비용 절감)
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - name: Integration Tests (Shard ${{ matrix.shard }}/3)
        run: |
          ./gradlew integrationTest \
            --parallel \
            -Dspring.datasource.url=jdbc:postgresql://localhost:5432/testdb \
            -Dspring.redis.host=localhost \
            -Dtest.shard=${{ matrix.shard }} \
            -Dtest.shard.total=3
        timeout-minutes: 15
        env:
          SPRING_DATASOURCE_PASSWORD: postgres

  # 스테이지 3: E2E 테스트 (전 스테이지 통과 후)
  e2e-tests:
    needs: [fast-tests, integration-tests]
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      # E2E 테스트는 불안정하므로 3회 재시도
      - name: E2E Tests (with Retry)
        run: |
          ./gradlew e2eTest \
            -De2e.retry.max=3 \
            -De2e.timeout=60000
        timeout-minutes: 30
```

**개선 효과**:
- 전체 파이프라인: 60분 → 25분 (58% 단축)
- PR 피드백: 5분 내 신속한 응답
- 개발자 생산성: 컨텍스트 스위칭 감소

---

## 🔬 내부 동작 원리

### 테스트 피라미드 구조

```
       E2E 테스트 (5%)
      /            \
    UI 자동화      API 엔드-투-엔드
    (느림, 비쌈)    (10초/테스트)
    
      통합 테스트 (15%)
     /            \
  DB 통합       API 통합
  (1초/테스트)  (500ms/테스트)
  
    단위 테스트 (80%)
   /            \
메서드 단위      클래스 단위
(10ms/테스트)  (50ms/테스트)
```

### 각 층의 특징

| 테스트 유형 | 속도 | 비용 | 개수 | 신뢰성 | 배치 위치 |
|-----------|------|------|------|--------|----------|
| 단위 | 10-50ms | 낮음 | 매우 많음 (80%) | 높음 | PR 검증 |
| 통합 | 500ms-2s | 중간 | 중간 (15%) | 높음 | main 머지 후 |
| E2E | 5-30s | 높음 | 적음 (5%) | 낮음 | 스테이징 배포 후 |

**왜 이 순서인가?**

1. **빠른 테스트를 먼저**: "실패할 확률이 높은 빌드를 최대한 빨리 차단"
2. **느린 테스트를 나중에**: "이미 통과한 빌드에서만 비싼 테스트 실행"
3. **E2E는 마지막**: "품질이 보증된 후 실제 환경에서 한 번 더 검증"

### Flaky Test 원인과 격리

```
Flaky Test = 같은 코드인데 때론 실패, 때론 성공

원인:
1. 시간에 의존한 테스트
   - Thread.sleep(1000) 후 상태 확인 → 느린 CI에선 실패
   
2. 순서에 의존한 테스트
   - Test A 후 Test B 순서 가정 → 병렬 실행 시 실패
   
3. 네트워크 불안정성
   - 외부 API 호출 → 간헐적 타임아웃
   
4. 동시성 버그
   - Race condition → 특정 조건에만 노출
```

**격리 전략**:

```java
// ❌ Flaky: 시간 가정
@Test
public void testAsyncOperation() {
    service.startAsync();
    Thread.sleep(1000);  // 느린 CI에서 실패할 수 있음
    assertEquals("completed", service.getStatus());
}

// ✅ Reliable: Polling 사용
@Test
public void testAsyncOperation() {
    service.startAsync();
    
    // 최대 10초 동안 상태 확인 (100ms 간격)
    await()
        .atMost(Duration.ofSeconds(10))
        .pollInterval(Duration.ofMillis(100))
        .untilAsserted(() -> {
            assertEquals("completed", service.getStatus());
        });
}

// ✅ Reliable: Callback 기반
@Test
public void testAsyncOperationWithCallback(WaitingAssertion assertion) {
    service.startAsync(result -> {
        assertion.assertEquals("completed", result.getStatus());
    });
    
    assertion.awaitCompletion();
}
```

---

## 💻 실전 실험

### 실험 1: 기존 파이프라인 성능 측정

```bash
# 테스트 그룹별 소요 시간 측정
echo "=== Unit Tests ==="
time ./gradlew test -x integrationTest -x e2eTest

# 출력 예:
# real    4m35.123s
# user    12m15.234s

echo "=== Integration Tests ==="
time ./gradlew integrationTest -x test -x e2eTest

# 출력 예:
# real    8m12.456s

echo "=== Total Sequential ==="
echo "4m35s + 8m12s = 12m47s"
```

### 실험 2: 병렬 실행으로 최적화

```gradle
// build.gradle.kts
tasks.test {
    // 병렬 실행 활성화
    useJUnitPlatform {
        includeEngines("junit-jupiter")
    }
    
    // 최대 4개 워커로 병렬 실행
    maxParallelForks = Runtime.getRuntime().availableProcessors().div(2)
    
    // 테스트 격리: 각 포크는 독립적인 프로세스
    forkEvery = 100  // 100개 테스트마다 새 JVM 생성
    
    // 타임아웃 설정
    timeout = Duration.ofSeconds(60)
}

tasks.register<Test>("integrationTest") {
    useJUnitPlatform {
        includeTags("integration")
    }
    
    // 시스템 속성 전달
    systemProperty("spring.jpa.hibernate.ddl-auto", "create-drop")
    systemProperty("spring.h2.console.enabled", "true")
    
    mustRunAfter("test")
}
```

### 실험 3: GitHub Actions Matrix로 Sharding

```yaml
name: Matrix Sharding Test
on: [push]

jobs:
  test-matrix:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-shard: [0, 1, 2, 3]  # 4개 병렬 작업
        total-shards: [4]
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - name: Run Integration Tests (Shard ${{ matrix.test-shard }}/${{ matrix.total-shards }})
        run: |
          # JUnit 5 자체 sharding 기능 사용
          ./gradlew integrationTest \
            --tests="**/integration/**" \
            -Djunit.jupiter.execution.parallel.enabled=true \
            -Djunit.jupiter.execution.parallel.mode.default=concurrent \
            -Djunit.jupiter.execution.parallel.mode.classes.default=concurrent
        env:
          TEST_SHARD_INDEX: ${{ matrix.test-shard }}
          TEST_SHARD_COUNT: ${{ matrix.total-shards }}

  report:
    needs: test-matrix
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Report Results
        run: |
          echo "All shards completed"
          # 여기서 결과를 집계
```

### 실험 4: Flaky Test 감지 및 격리

```yaml
name: Flaky Test Detection
on: [pull_request]

jobs:
  detect-flaky:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      # 3회 반복 실행하여 불안정한 테스트 감지
      - name: Run Tests 3 Times
        run: |
          for run in {1..3}; do
            echo "=== Run $run ==="
            ./gradlew test --tests="**/service/**" \
              -Dspring.jpa.hibernate.ddl-auto=create-drop \
              -Dspring.jpa.show-sql=false \
              2>&1 | tee test-run-$run.log
          done
      
      # 결과 비교
      - name: Compare Results
        run: |
          # 각 실행에서 실패한 테스트 추출
          grep "FAILED" test-run-*.log | sort | uniq -c | grep -v "1 " \
            && echo "⚠️ Flaky tests detected!" \
            || echo "✅ All tests stable"

  quarantine-flaky:
    needs: detect-flaky
    runs-on: ubuntu-latest
    if: failure()
    steps:
      - uses: actions/checkout@v4
      - name: Mark Flaky Tests
        run: |
          # Flaky로 표시된 테스트에 @Disabled 또는 @DisabledIfEnvironmentVariable 추가
          find src -name "*.java" -exec grep -l "@Test" {} \; | while read file; do
            sed -i '/@Test/{N;/@Disabled/!s/@Test/@Disabled(reason = "Flaky test - needs investigation")\n    @Test/}' "$file"
          done
```

---

## 📊 성능/비용 비교

### 순차 vs 병렬 실행

```
순차 실행:
├─ Unit Tests (4m35s)
├─ Integration Tests (8m12s)
└─ E2E Tests (12m00s)
────────────────────
총 소요: 24m47s

병렬 실행 (Matrix Sharding):
├─ Unit Tests (4m35s)  ─────────────────────────┐
├─ Integration Tests (2m44s × 3) ──────────────┤ (병렬)
├─ E2E Tests (6m00s) ───────────────────────────┤
─────────────────────────────────────────────────
총 소요: 13m19s (46% 단축)

CI/CD 비용 (GitHub Actions 분/월):
순차: 24.47분 × 100개 PR = 2,447분/월 (~41시간)
병렬: 13.19분 × 100개 PR = 1,319분/월 (~22시간)
절감: 1,128분 (46% 비용 절감)
```

### Docker 컨테이너 활용

```
비용 비교 (1000개 테스트 기준):

1. In-Memory DB (H2):
   - 테스트 시간: 8분
   - 환경 설정 오류: 높음 (20% 확률)
   - 네트워크 오류: 없음
   - 실제 DB 호환성: 낮음

2. Testcontainers (PostgreSQL):
   - 테스트 시간: 12분 (Docker 오버헤드)
   - 환경 설정 오류: 낮음 (2% 확률)
   - 네트워크 오류: 낮음
   - 실제 DB 호환성: 매우 높음 (100%)
   
   → 오류율 18% 감소 → 재실행 비용 절감
   → 추가 4분 > 재실행 비용 절감
```

---

## ⚖️ 트레이드오프

### 1. 테스트 피라미드 비율 조정

```
조직 특성별 권장 비율:

Startups (빠른 반복):
단위 60% / 통합 25% / E2E 15%
→ 빠른 피드백이 우선

Enterprise (안정성 중시):
단위 70% / 통합 20% / E2E 10%
→ 검증된 코드 품질 보증

Legacy 시스템 (E2E 의존성):
단위 50% / 통합 30% / E2E 20%
→ 단위 테스트 작성 어려움
```

### 2. 병렬 실행의 부작용

```
장점:
✅ 전체 시간 단축 (50% 가능)
✅ 빠른 피드백

단점:
❌ 병렬 실행 추가 메모리 사용 (+30% RAM)
❌ 데이터베이스 연결 수 증가 (동시성 제약)
❌ 테스트 격리 어려움 (공유 리소스 접근)

해결책:
- 최대 워커 수 제한: maxParallelForks = 4
- 격리된 DB 인스턴스 사용: Testcontainers
- 테스트 순서 의존성 제거
```

### 3. Flaky Test 격리 비용

```
격리 비용:
- 개발 시간: 각 테스트당 30분~1시간
- 유지보수: 특수한 대기 로직 관리

이익:
- 신뢰성 향상 (+25%)
- 파이프라인 재실행 감소 (80% 감소)
- CI/CD 비용 절감

ROI: Flaky Test 5개 격리 → 월 평균 10시간 절감
```

---

## 📌 핵심 정리

| 원칙 | 설명 |
|------|------|
| **빠른 것 먼저** | 5분 내 피드백 → 단위 테스트 우선 배치 |
| **점진적 엄격화** | 각 단계 통과한 것만 다음 단계로 진행 |
| **병렬화** | 스테이지 간 순서 의존성 제거 → Matrix 활용 |
| **신뢰성** | Flaky 테스트 격리 → 재실행 최소화 |
| **격리** | 각 테스트는 독립적인 환경에서 실행 |

---

## 🤔 생각해볼 문제

### Q1: "우리 팀은 E2E 테스트가 80%인데, 피라미드를 어떻게 재구성할까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**문제 분석**:
- 단위 테스트 부족 → 빌드 시간 매우 김
- E2E 의존도 높음 → 높은 관리 비용

**점진적 개선 방안**:

1단계 (1개월): 현재 E2E 테스트 중 "비즈니스 로직 검증"은 단위 테스트로 변환
```java
// E2E에서 분리할 단위 테스트
@Test
public void testOrderCalculationLogic() {
    Order order = new Order(List.of(
        new Item("ITEM1", 100, 2),
        new Item("ITEM2", 50, 1)
    ));
    
    assertEquals(250, order.calculateTotal());  // DB 없이 테스트
}
```

결과: E2E 60% → 50%, 단위 테스트 20% → 30%

2단계 (2개월): DB 통합 테스트를 Testcontainers로 자동화
```gradle
tasks.register<Test>("integrationTest") {
    useJUnitPlatform { includeTags("integration") }
    // Testcontainers 자동 시작
}
```

결과: E2E 50% → 35%, 통합 테스트 10% → 30%

3단계 (진행 중): E2E는 핵심 사용자 흐름만 유지
- 로그인 → 상품 구매 → 결제 → 이메일 수신
- 약 10-15개 테스트만 유지

**예상 효과**:
- 파이프라인: 45분 → 15분 (67% 단축)
- 유지보수: 50개 E2E → 15개 E2E (70% 감소)
- 신뢰성: Flaky 확률 80% → 20% (75% 개선)

</details>

### Q2: "병렬 실행 중 테스트가 간헐적으로 실패한다. 원인은?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**의심 원인 Top 5**:

1. **공유 리소스 접근** (확률 60%)
```java
// ❌ 공유 리소스: 모든 테스트가 같은 파일 수정
@BeforeEach
void setup() {
    FileUtils.writeStringToFile(
        new File("/tmp/shared-config.txt"), 
        "test-data"
    );
}

// ✅ 격리된 리소스: 각 테스트가 고유한 파일
@BeforeEach
void setup() {
    String uniquePath = "/tmp/test-" + System.nanoTime() + "/";
    Files.createDirectories(Paths.get(uniquePath));
}
```

2. **데이터베이스 동시성** (확률 30%)
```yaml
# ❌ 문제: 모든 테스트가 같은 DB 인스턴스
spring:
  datasource:
    url: jdbc:h2:mem:testdb  # 모든 테스트가 같은 메모리 DB

# ✅ 해결: Testcontainers로 격리
spring:
  datasource:
    url: jdbc:postgresql://localhost:${DB_PORT}/test_db_${TEST_INSTANCE}
```

3. **정적 상태** (확률 7%)
```java
// ❌ 문제: 정적 변수에 상태 저장
public class TestContext {
    private static Map<String, String> cache = new HashMap<>();  // 공유됨!
}

// ✅ 해결: ThreadLocal 사용
public class TestContext {
    private static ThreadLocal<Map<String, String>> cache = 
        ThreadLocal.withInitial(HashMap::new);
}
```

4. **타이밍 문제** (확률 2%)
5. **외부 API 호출** (확률 1%)

**진단 명령**:
```bash
# 병렬 실행에서만 실패하는지 확인
./gradlew test --parallel              # 실패
./gradlew test -x integrationTest      # 성공

# 문제 테스트 격리
./gradlew test --tests="**/SuspiciousTest" -Dtest.single.fork=true
```

</details>

### Q3: "CI/CD 비용을 줄이면서도 품질을 유지하려면?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**3가지 전략**:

**1. 선택적 테스트 실행** (비용 40% 절감)
```yaml
name: Smart Test Selection
on: [pull_request]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      unit-tests: ${{ steps.changes.outputs.unit }}
      integration-tests: ${{ steps.changes.outputs.integration }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 필요
      
      - name: Detect Changed Files
        id: changes
        run: |
          # main과의 차이 비교
          CHANGED=$(git diff origin/main --name-only)
          
          # 변경된 파일 유형별로 테스트 결정
          if echo "$CHANGED" | grep -qE "^src/main/java/.*Service\.java"; then
            echo "integration=true" >> $GITHUB_OUTPUT
          fi
          if echo "$CHANGED" | grep -qE "^src/test/java"; then
            echo "unit=true" >> $GITHUB_OUTPUT
          fi
  
  unit-tests:
    needs: detect-changes
    if: needs.detect-changes.outputs.unit-tests == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - run: ./gradlew test

  integration-tests:
    needs: detect-changes
    if: needs.detect-changes.outputs.integration-tests == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - run: ./gradlew integrationTest
```

**2. 캐시 활용** (비용 30% 절감)
```yaml
- name: Cache Gradle
  uses: actions/cache@v3
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
      ~/.m2/repository
    key: gradle-${{ hashFiles('**/*.gradle.kts') }}
    restore-keys: gradle-

- name: Cache Docker Images
  uses: actions/cache@v3
  with:
    path: /var/lib/docker
    key: docker-${{ hashFiles('docker-compose.yml') }}
```

**3. Nightly 테스트** (비용 50% 절감)
```yaml
# PR: 빠른 테스트만 (5분)
name: Fast PR Tests
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew test --parallel

# Nightly: 전체 테스트 (30분)
name: Nightly Full Tests
on:
  schedule:
    - cron: '0 2 * * *'  # 매일 2시 AM UTC
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew test integrationTest e2eTest
```

**결합 효과**:
- 선택적 실행: -40%
- 캐시: -30%
- Nightly 분리: -50%
- **전체: -62% 비용 절감**

</details>

---

[⬅️ 이전: Chapter 5 — Secret 관리](../gitops-argocd/06-secret-management.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./02-spring-boot-test-optimization.md)
