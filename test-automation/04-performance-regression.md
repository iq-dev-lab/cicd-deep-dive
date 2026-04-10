# 성능 회귀 감지 — k6 Smoke Test와 Baseline 비교

## 🎯 핵심 질문

- 성능 회귀(Performance Regression)란 무엇이고 어떻게 자동으로 감지할까?
- k6 스크립트로 API 응답 시간을 측정하는 방법은?
- GitHub Actions에서 k6를 실행하고 결과를 분석하는 방법은?
- Baseline(이전 버전 성능)과 비교해서 10% 이상 느려지면 자동으로 파이프라인을 차단할 수 있을까?
- k6 Cloud와 Grafana Dashboard로 결과를 시각화하려면?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**성능 회귀는 무지 속에서 발생합니다.**

"이번 코드 변경이 얼마나 느려졌는가?"를 모르면:
- 사용자 경험 악화: 응답 시간 1초 증가 → 전환율 7% 감소 (Google)
- 서버 비용 증가: API 응답이 2배 느려짐 → 인스턴스 2배 필요
- 버그 원인 파악 어려움: "어느 PR에서 느려졌지?"

**자동화된 성능 테스트가 해결합니다:**
- 모든 PR에서 성능 체크: 5분 내 피드백
- 성능 회귀 조기 발견: 배포 전 차단
- 개선 효과 측정: "최적화 후 20% 빨라짐"

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```javascript
// ❌ 잘못된 접근 1: 수동 성능 테스트

// 매번 dev 환경에서 브라우저로 측정
// API 응답 시간: 500ms (체감)
// → 정확하지 않음, 재현 불가능, 일관성 없음

// ❌ 잘못된 접근 2: 테스트 없이 배포

// "코드를 정리했는데 성능에 영향이 없겠지?"
// → 실제로는 부정적 영향 발생
// → 사용자 불만 증가

// ❌ 잘못된 접근 3: 성능 메트릭 관리 안 함

// 운영 환경에서만 모니터링
// → 버그가 프로덕션에 도달한 후 발견
// → 긴급 롤백, 신뢰도 감소
```

**결과**:
- 성능 저하 조기 감지 불가
- 배포 후 성능 문제 발생
- 원인 파악 어려움 (어느 PR의 어느 코드?)

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 1단계: k6 성능 테스트 스크립트 작성

```javascript
// k6-smoke-test.js
// 설치: npm install k6 (또는 바이너리 다운로드)

import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend, Counter, Gauge } from 'k6/metrics';

// 커스텀 메트릭
const errorRate = new Rate('errors');
const apiDuration = new Trend('api_duration');
const successCount = new Counter('success_count');
const concurrentUsers = new Gauge('concurrent_users');

// VU(Virtual User) 설정: 부하 시뮬레이션
export const options = {
    // 시나리오 1: Smoke Test (빠른 검증)
    scenarios: {
        smoke: {
            executor: 'constant-vus',
            vus: 5,           // 동시 5명의 사용자
            duration: '30s',  // 30초 동안 유지
            tags: { name: 'smoke' },
        },
        
        // 시나리오 2: Load Test (부하 테스트)
        load: {
            executor: 'ramping-vus',
            stages: [
                { duration: '2m', target: 10 },   // 0→10 VU (2분)
                { duration: '5m', target: 10 },   // 10 VU 유지 (5분)
                { duration: '2m', target: 0 },    // 10→0 VU (2분)
            ],
            tags: { name: 'load' },
            startTime: '1m',  // 1분 후 시작 (smoke 후)
        },
    },
    
    // 임계값: 이를 초과하면 테스트 실패
    thresholds: {
        'api_duration': ['p(95)<500', 'p(99)<1000'],  // 95%가 500ms 이하
        'errors': ['rate<0.05'],                       // 에러율 5% 이하
        'http_req_failed': ['rate<0.1'],               // HTTP 실패 10% 이하
    },
};

// 테스트 함수
export default function () {
    concurrentUsers.add(1);
    
    group('API - User Operations', () => {
        // 1. 사용자 목록 조회
        const listResponse = http.get(
            'http://localhost:8080/api/users',
            {
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Bearer token123',
                },
                tags: { name: 'GetUsers' },
            }
        );
        
        check(listResponse, {
            'GET /api/users - Status 200': (r) => r.status === 200,
            'GET /api/users - Duration < 500ms': (r) => r.timings.duration < 500,
            'GET /api/users - Has body': (r) => r.body.length > 0,
        });
        
        apiDuration.add(listResponse.timings.duration, 
            { endpoint: '/api/users', method: 'GET' });
        
        errorRate.add(listResponse.status !== 200);
        if (listResponse.status === 200) successCount.add(1);
        
        sleep(1);  // 1초 대기 (자연스러운 사용자 행동)
        
        // 2. 사용자 조회
        const userId = JSON.parse(listResponse.body)[0]?.id || 1;
        const getResponse = http.get(
            `http://localhost:8080/api/users/${userId}`,
            { tags: { name: 'GetUserById' } }
        );
        
        check(getResponse, {
            'GET /api/users/{id} - Status 200': (r) => r.status === 200,
            'GET /api/users/{id} - Has name': (r) => r.json('name') !== undefined,
        });
        
        apiDuration.add(getResponse.timings.duration,
            { endpoint: '/api/users/{id}', method: 'GET' });
        
        sleep(1);
        
        // 3. 사용자 생성
        const createPayload = JSON.stringify({
            name: `LoadTest-${Math.random()}`,
            email: `user-${Date.now()}@example.com`,
        });
        
        const createResponse = http.post(
            'http://localhost:8080/api/users',
            createPayload,
            {
                headers: { 'Content-Type': 'application/json' },
                tags: { name: 'CreateUser' },
            }
        );
        
        check(createResponse, {
            'POST /api/users - Status 201': (r) => r.status === 201,
            'POST /api/users - Has id': (r) => r.json('id') !== undefined,
        });
        
        apiDuration.add(createResponse.timings.duration,
            { endpoint: '/api/users', method: 'POST' });
        
        sleep(2);
    });
    
    group('API - Order Operations', () => {
        // 4. 주문 목록 조회
        const ordersResponse = http.get(
            'http://localhost:8080/api/orders?limit=10',
            { tags: { name: 'GetOrders' } }
        );
        
        check(ordersResponse, {
            'GET /api/orders - Status 200': (r) => r.status === 200,
            'GET /api/orders - Duration < 1000ms': (r) => r.timings.duration < 1000,
        });
        
        apiDuration.add(ordersResponse.timings.duration,
            { endpoint: '/api/orders', method: 'GET' });
        
        sleep(1);
    });
    
    concurrentUsers.add(-1);
}

// 요약 통계 함수
export function handleSummary(data) {
    return {
        'stdout': textSummary(data, { indent: ' ', enableColors: true }),
        'summary.json': JSON.stringify(data),  // CI/CD에서 파싱용
    };
}
```

### 2단계: GitHub Actions에서 k6 실행

```yaml
# ✅ GitHub Actions: k6 성능 테스트

name: Performance Regression Detection
on: [pull_request, push]

jobs:
  # 1단계: 애플리케이션 빌드 및 시작
  build-and-start:
    runs-on: ubuntu-latest
    outputs:
      api-url: ${{ steps.server.outputs.api-url }}
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      - name: Build Application
        run: ./gradlew clean build -x test
      
      - name: Start Application Server
        id: server
        run: |
          # 백그라운드에서 서버 시작
          java -jar build/libs/app.jar &
          
          # 서버가 준비될 때까지 대기
          for i in {1..30}; do
            if curl -s http://localhost:8080/actuator/health; then
              echo "✅ Server ready"
              echo "api-url=http://localhost:8080" >> $GITHUB_OUTPUT
              exit 0
            fi
            echo "Waiting for server... ($i/30)"
            sleep 2
          done
          
          echo "❌ Server failed to start"
          exit 1
  
  # 2단계: Baseline 성능 데이터 가져오기 (이전 버전)
  get-baseline:
    runs-on: ubuntu-latest
    outputs:
      baseline-json: ${{ steps.baseline.outputs.data }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main  # main 브랜치의 성능 데이터
      
      - name: Get Baseline Performance
        id: baseline
        run: |
          # 이전 성공한 테스트의 성능 데이터 다운로드
          if [ -f "performance-baseline.json" ]; then
            cat performance-baseline.json
            echo "data=$(cat performance-baseline.json | jq -c .)" >> $GITHUB_OUTPUT
          else
            echo "No baseline found, using defaults"
            echo 'data={"p95": 500, "p99": 1000}' >> $GITHUB_OUTPUT
          fi
  
  # 3단계: 현재 코드의 성능 테스트
  performance-test:
    needs: [build-and-start, get-baseline]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # k6 설치
      - name: Install k6
        run: |
          sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 \
            --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb https://dl.k6.io/deb stable main" | \
            sudo tee /etc/apt/sources.list.d/k6-stable.list
          sudo apt-get update
          sudo apt-get install -y k6
      
      # 성능 테스트 실행
      - name: Run k6 Performance Test
        id: k6-test
        run: |
          k6 run k6-smoke-test.js \
            --vus=10 \
            --duration=2m \
            --out json=test-results.json \
            --out csv=test-results.csv
        env:
          API_URL: ${{ needs.build-and-start.outputs.api-url }}
        continue-on-error: true
      
      # 결과 분석
      - name: Analyze Performance Results
        id: analysis
        run: |
          SUMMARY=$(cat test-results.json | jq '.metrics')
          
          # 95 percentile 응답 시간 추출
          P95=$(echo $SUMMARY | jq '.api_duration.values.p(95)' || echo "999")
          P99=$(echo $SUMMARY | jq '.api_duration.values.p(99)' || echo "9999")
          ERROR_RATE=$(echo $SUMMARY | jq '.errors.value' || echo "0.1")
          
          echo "Current Performance:"
          echo "  P95: ${P95}ms"
          echo "  P99: ${P99}ms"
          echo "  Error Rate: ${ERROR_RATE}"
          
          echo "p95=${P95}" >> $GITHUB_OUTPUT
          echo "p99=${P99}" >> $GITHUB_OUTPUT
          echo "error_rate=${ERROR_RATE}" >> $GITHUB_OUTPUT
      
      # Baseline과 비교
      - name: Compare with Baseline
        id: regression-check
        run: |
          # Baseline 성능 (이전 버전)
          BASELINE_P95=${{ fromJson(needs.get-baseline.outputs.baseline-json).p95 || 500 }}
          BASELINE_P99=${{ fromJson(needs.get-baseline.outputs.baseline-json).p99 || 1000 }}
          
          # 현재 성능
          CURRENT_P95=${{ steps.analysis.outputs.p95 }}
          CURRENT_P99=${{ steps.analysis.outputs.p99 }}
          
          # 성능 저하율 계산
          P95_DIFF=$(echo "scale=2; ($CURRENT_P95 - $BASELINE_P95) * 100 / $BASELINE_P95" | bc)
          P99_DIFF=$(echo "scale=2; ($CURRENT_P99 - $BASELINE_P99) * 100 / $BASELINE_P99" | bc)
          
          echo "Baseline vs Current:"
          echo "  P95: $BASELINE_P95ms → ${CURRENT_P95}ms (${P95_DIFF}%)"
          echo "  P99: $BASELINE_P99ms → ${CURRENT_P99}ms (${P99_DIFF}%)"
          
          # 10% 이상 느려짐 = 성능 회귀
          REGRESSION=0
          if (( $(echo "$P95_DIFF > 10" | bc -l) )); then
            echo "❌ P95 performance regression detected: ${P95_DIFF}%"
            REGRESSION=1
          fi
          
          if (( $(echo "$P99_DIFF > 10" | bc -l) )); then
            echo "❌ P99 performance regression detected: ${P99_DIFF}%"
            REGRESSION=1
          fi
          
          if [ $REGRESSION -eq 0 ]; then
            echo "✅ Performance within acceptable range"
          fi
          
          echo "regression=${REGRESSION}" >> $GITHUB_OUTPUT
      
      # 결과 저장 (Baseline 업데이트용)
      - name: Save Performance Baseline
        if: success()
        run: |
          cat > performance-baseline.json <<EOF
          {
            "p95": ${{ steps.analysis.outputs.p95 }},
            "p99": ${{ steps.analysis.outputs.p99 }},
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "commit": "${{ github.sha }}"
          }
          EOF
      
      - name: Upload Performance Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: |
            test-results.json
            test-results.csv
            performance-baseline.json
      
      # 회귀 감지 시 실패
      - name: Fail on Performance Regression
        if: steps.regression-check.outputs.regression == '1'
        run: |
          echo "❌ Performance regression detected"
          exit 1
```

### 3단계: k6 Cloud와 Grafana 연동

```bash
# k6 Cloud에 결과 업로드
k6 run k6-smoke-test.js \
  --out cloud \
  --cloud
  # 결과: https://app.k6.io/projects/YOUR_PROJECT_ID

# Grafana 대시보드에서 실시간 모니터링
# 1. k6 Prometheus 메트릭 활성화
# 2. Grafana가 Prometheus 쿼리
# 3. 차트로 성능 추세 시각화
```

```javascript
// k6-grafana-integration.js

import { InfluxDB } from 'k6/x/influxdb';

const influxDB = new InfluxDB({ url: 'http://localhost:8086' });

export const options = {
    scenarios: {
        smoke: {
            executor: 'constant-vus',
            vus: 5,
            duration: '30s',
        },
    },
    
    // Prometheus 출력 설정
    ext: {
        loadimpact: {
            projectID: 3356643,
            name: 'API Performance Test',
        },
    },
};

export default function () {
    const response = http.get('http://localhost:8080/api/users');
    
    // InfluxDB에 메트릭 저장 (Grafana가 시각화)
    influxDB.write('api_response_time', 
        response.timings.duration, 
        { endpoint: '/api/users' });
    
    check(response, {
        'Status 200': (r) => r.status === 200,
    });
}
```

---

## 🔬 내부 동작 원리

### 성능 회귀 감지 메커니즘

```
파이프라인 실행 흐름:

1. 현재 코드 성능 테스트
   └─ k6 스크립트 실행
      └─ 10명 동시 사용자, 2분 부하 테스트
         └─ API 응답 시간 측정
            └─ P95, P99, 평균값 계산

2. Baseline 조회
   └─ 이전 성공한 테스트의 성능 데이터
      └─ performance-baseline.json 파일
         └─ main 브랜치의 마지막 커밋 기준

3. Regression Detection (비교)
   └─ 현재 vs Baseline 비교
      └─ (현재 - Baseline) / Baseline × 100 = 저하율
         └─ 저하율 > 10%? → 회귀 감지
            └─ 파이프라인 차단
```

### k6 메트릭 계산 방식

```
Percentile 계산:

응답 시간: [100, 150, 200, 250, 300, 350, 400, 450, 500, 1000]ms

P50 (중앙값):
  → 위치: (50% × 10) = 5번째
  → 300ms

P95 (상위 5% 제외):
  → 위치: (95% × 10) = 9.5번째
  → 950ms (9.5번째와 10번째 평균)

P99 (상위 1% 제외):
  → 위치: (99% × 10) = 9.9번째
  → 991ms

실제 사용:
- P50: 일반적인 응답 시간
- P95: 대부분의 사용자 경험
- P99: 최악의 경우 (느린 사용자)
```

### 성능 메트릭 수집 과정

```
k6 실행 중:

VU 1 ─┐
      ├─ API 호출 ─┐
      │            ├─ 응답 시간 수집
      ├─ API 호출 ─┤ (다양한 메트릭)
VU 2 ─┤            │
      │ API 호출 ──┤ 메트릭:
      │            ├─ http_req_duration
VU 3 ─┤ API 호출 ──┤ http_req_failed
      │            ├─ http_req_blocked
      ├─ API 호출 ──┤ http_req_connecting
      │            ├─ http_req_tls_handshaking
VU 4 ─┴─ ...      │ http_req_sending
      ...         ├─ http_req_waiting
                  └─ http_req_receiving

집계:

모든 응답 시간:
[100, 120, 150, 180, 200, ... 500, 1000, 5000]ms

통계:
- min: 100ms
- max: 5000ms
- avg: 250ms
- p95: 450ms
- p99: 900ms
```

---

## 💻 실전 실험

### 실험 1: 로컬에서 k6 실행

```bash
# 1. k6 설치
# macOS:
brew install k6

# Linux:
sudo apt-get install k6

# 2. 간단한 k6 스크립트 작성
cat > simple-test.js <<'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    vus: 10,           // 10명 동시 사용자
    duration: '30s',   // 30초 동안
    thresholds: {
        'http_req_duration': ['p(95)<500'],  // P95 < 500ms
    },
};

export default function () {
    const response = http.get('https://httpbin.org/delay/1');
    
    check(response, {
        'Status is 200': (r) => r.status === 200,
        'Response time < 2s': (r) => r.timings.duration < 2000,
    });
    
    sleep(1);
}
EOF

# 3. 실행
k6 run simple-test.js

# 출력 예:
# ✓ Status is 200
# ✓ Response time < 2s
# 
# checks.........................: 100% ✓ 600 ✗ 0
# http_req_duration..............: avg=1.02s min=1s max=1.04s p(95)=1.03s p(99)=1.04s
# http_reqs........................: 300 10/s
# iteration_duration..............: avg=2.02s min=2s max=2.04s p(95)=2.03s p(99)=2.04s
# iterations......................: 300 10/s
# vus...........................: 10
```

### 실험 2: Spring Boot 애플리케이션 성능 테스트

```gradle
// build.gradle.kts - 성능 테스트 간단하게

tasks.register<Test>("performanceTest") {
    useJUnitPlatform {
        includeTags("performance")
    }
    
    doFirst {
        println("Starting application for performance test...")
        // Spring Boot 애플리케이션 시작
    }
}
```

```java
// PerformanceTest.java - k6 대신 JMH 사용 가능

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Thread)
public class UserServicePerformanceTest {
    
    private UserService userService;
    private UserRepository userRepository;
    
    @Setup
    public void setup() {
        // 테스트 환경 초기화
        userService = new UserService(userRepository);
    }
    
    @Benchmark
    public User getUserById() {
        return userService.getUserById(1L);
    }
    
    @Benchmark
    public List<User> listUsers() {
        return userService.listUsers();
    }
}

// 실행:
// ./gradlew jmh
// 결과: 각 메서드의 평균 실행 시간 (nanosecond 단위)
```

### 실험 3: Baseline 저장 및 비교

```bash
# 1. main 브랜치에서 성능 테스트 실행
git checkout main
k6 run k6-smoke-test.js --out json=baseline.json

# 결과 저장
jq '.metrics.api_duration.values' baseline.json > baseline-p95.json
# 출력: {"p(95)": 450, "p(99)": 900}

# 2. 기능 브랜치로 전환
git checkout feature/new-feature

# 3. 새로운 코드로 테스트
k6 run k6-smoke-test.js --out json=current.json

# 4. 비교
jq '.metrics.api_duration.values' current.json > current-p95.json

# 5. 성능 변화 계산
python3 <<'PYTHON'
import json

with open('baseline-p95.json') as f:
    baseline = json.load(f)
    
with open('current-p95.json') as f:
    current = json.load(f)

p95_diff = (current['p(95)'] - baseline['p(95)']) / baseline['p(95)'] * 100
print(f"P95 performance change: {p95_diff:.1f}%")

if p95_diff > 10:
    print("❌ Performance regression detected!")
    exit(1)
else:
    print("✅ Performance within acceptable range")
PYTHON
```

### 실험 4: GitHub Actions에서 성능 테스트 및 Slack 알림

```yaml
name: Performance Test with Slack Notification
on: [pull_request]

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install k6
        run: |
          sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 \
            --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb https://dl.k6.io/deb stable main" | \
            sudo tee /etc/apt/sources.list.d/k6-stable.list
          sudo apt-get update && sudo apt-get install -y k6
      
      # 성능 테스트
      - name: Run k6 test
        id: k6
        run: |
          k6 run k6-smoke-test.js \
            --out json=results.json \
            --out summary=results.txt
          
          # 결과 추출
          jq '.metrics.api_duration.values | "P95: \(.["p(95)"])ms, P99: \(.["p(99)"])ms"' results.json -r
        continue-on-error: true
      
      # Slack 알림 (성공)
      - name: Notify Slack - Success
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "✅ Performance Test Passed",
              attachments: [{
                color: "good",
                text: "P95: 450ms, P99: 900ms"
              }]
            }
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
      
      # Slack 알림 (실패)
      - name: Notify Slack - Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "❌ Performance Regression Detected",
              attachments: [{
                color: "danger",
                text: "P95 increased from 450ms to 550ms (+22%)"
              }]
            }
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 📊 성능/비용 비교

```
성능 테스트 방식별 비용:

1. 수동 테스트:
   시간: 30분/PR
   인력: 1명
   월간: 10시간 (100 PR × 6분)
   문제: 일관성 없음, 재현 불가능

2. k6 자동 테스트:
   시간: 5분/PR (자동)
   인력: 0 (CI/CD 자동)
   비용: $0 (오픈소스)
   월간: 8시간 30분 (CI 서버 시간)
   장점: 일관성 높음, 재현 가능

3. k6 Cloud (SaaS):
   시간: 5분/PR
   월간 비용: $400-2000
   장점: 대규모 부하 테스트, 분석 도구

회귀 감지 효과:
- 성능 저하 조기 발견: 배포 전 차단
- 롤백 비용 절감: 프로덕션 장애 80% 감소
- 사용자 경험 개선: 응답 시간 10% 개선

ROI: 자동화 설정 4시간 < 월간 30시간 절감
```

### Baseline 관리 비용

```
Baseline 저장 위치별:

1. 파일 (performance-baseline.json):
   - 저장소에 커밋
   - 모든 PR에서 비교 가능
   - 수동 업데이트 필요
   - 비용: 무료

2. k6 Cloud:
   - 클라우드에 저장
   - 자동 추적
   - 실시간 비교
   - 비용: $400/월

3. Grafana + Prometheus:
   - 로컬 저장소
   - 자동 메트릭 수집
   - 대시보드 시각화
   - 비용: 무료 (인프라 비용만)

권장: 작은 팀 → 파일, 대규모 팀 → k6 Cloud
```

---

## ⚖️ 트레이드오프

### 1. 성능 임계값 설정

```
엄격한 임계값 (P95 < 300ms):
✅ 높은 성능 기준 유지
❌ 작은 변화도 차단
❌ 최적화하지 않아도 되는 코드 최적화 압박

느슨한 임계값 (P95 < 1000ms):
✅ 자유로운 개발
❌ 성능 저하 놓칠 수 있음
❌ 누적되면 심각한 문제

권장: P95 < 500ms, 10% 이상 회귀 시 차단
```

### 2. 부하 시나리오 vs 테스트 시간

```
가벼운 시나리오 (VU=5, 1분):
✅ 빠른 피드백 (2분)
❌ 실제 부하 재현 못함
❌ 간헐적 문제 감지 못할 수 있음

무거운 시나리오 (VU=100, 10분):
✅ 실제 부하 근접 재현
❌ 느린 피드백 (15분)
❌ CI 비용 높음

권장: 기본은 가볍게 (VU=10, 2분), 야간에는 무겁게
```

### 3. 성능 vs 기능

```
성능 최적화 시간:
- 요구사항: "API 응답 500ms 이상 차단"
- 개발자: 기능 구현 + 성능 최적화 (50% 시간 증가)

성능 선택적 최적화:
- 요구사항: "회귀 10% 이상만 차단"
- 개발자: 기능 구현만 (정상 시간)
- 문제: 누적 성능 저하

권장: 
- 핵심 API: 엄격한 임계값
- 부차 API: 느슨한 임계값
- 차등 관리
```

---

## 📌 핵심 정리

| 개념 | 설명 |
|------|------|
| **Baseline** | 이전 코드의 성능 데이터 (비교 기준) |
| **P95** | 상위 5%를 제외한 응답 시간 (일반적 경험) |
| **Regression** | 성능이 10% 이상 저하된 상태 |
| **VU** | Virtual User (동시 사용자 시뮬레이션) |
| **Threshold** | 테스트 통과 조건 (P95 < 500ms 등) |

---

## 🤔 생각해볼 문제

### Q1: "로컬에서는 API 응답이 100ms인데 CI에서는 500ms다. 왜?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**원인 분석**:

1. **네트워크 지연**
```
로컬:
서버 (localhost:8080)
     ↓
클라이언트 (k6)
응답 시간: 1ms (네트워크 무시할 수 있음)

CI:
GitHub Actions 서버
     ↓ 네트워크 (Docker 컨테이너)
     ↓
애플리케이션 (별도 프로세스/컨테이너)
응답 시간: 50ms (네트워크 포함)
```

2. **리소스 경합**
```
로컬:
- CPU 전용
- 메모리 충분
- 디스크 빠름

CI:
- 여러 작업 동시 실행
- 메모리 제약 (2GB)
- 디스크 느림 (공유 스토리지)
```

3. **데이터베이스 상태**
```
로컬: 캐시된 데이터
CI: 매번 새로 생성 (인덱스 없음, 캐시 없음)
```

**해결책**:

```yaml
# CI에서 로컬과 동일한 환경으로 설정

jobs:
  performance-test:
    runs-on: ubuntu-latest
    services:
      # 데이터베이스를 같은 호스트에서 실행
      postgres:
        image: postgres:16-alpine
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    
    steps:
      - name: Warm up Database
        run: |
          # 캐시 워밍: 인덱스 생성, 캐시 로드
          ./gradlew flywayMigrate
          java -jar app.jar &
          sleep 30  # JVM 워밍업
      
      - name: Run k6 Test
        run: |
          # 3회 실행 (첫 실행은 제외, 2-3회 평균)
          k6 run k6-test.js --iterations=3
```

</details>

### Q2: "성능이 좋아졌는데도 테스트가 실패한다. 왜?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**원인 분석**:

Baseline이 오래되었거나 잘못된 데이터일 수 있습니다.

```
시나리오:
1. main 브랜치: API 응답 500ms (성능 나쁨)
   └─ Baseline: 500ms (기준 설정)

2. feature 브랜치: 최적화 적용
   API 응답 400ms (20% 개선!)
   └─ 하지만 비교: 400ms vs 500ms = -20% ✅

3. 그런데 CI에서는?
   └─ Baseline 읽기: main 브랜치의 performance-baseline.json
   └─ 근데 main 브랜치가 최적화되지 않았음
   └─ 비교 실패 가능성
```

**해결책**:

```yaml
# Baseline을 항상 최신으로 유지

name: Update Baseline
on: [push]

jobs:
  update-baseline:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run performance test
        run: k6 run k6-test.js --out json=results.json
      
      - name: Save baseline
        run: |
          # main 브랜치에 최신 성능 데이터 저장
          jq '.metrics.api_duration.values' results.json > performance-baseline.json
          
          git config user.name "CI/CD"
          git config user.email "ci@example.com"
          git add performance-baseline.json
          git commit -m "Update performance baseline"
          git push
```

**또 다른 원인: Flaky 성능 테스트**

```javascript
// ❌ 문제: 매번 다른 결과

export default function () {
    // 외부 API 호출 (불안정)
    const response = http.get('https://api.external.com/data');
    
    // 응답 시간이 매번 다름
    // 100ms ~ 5000ms
}

// ✅ 해결: Mock 사용 또는 로컬 호출

export default function () {
    // 로컬 API 호출 (일관성)
    const response = http.get('http://localhost:8080/api/data');
}
```

</details>

### Q3: "k6 테스트가 너무 많은 데이터를 생성해서 데이터베이스가 가득 찬다."

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**문제 상황**:

```javascript
// 문제 코드: 데이터 계속 증가

export default function () {
    // 10 VU × 60초 = 600개 사용자 생성
    const payload = JSON.stringify({
        name: `User-${Math.random()}`,
        email: `user-${Date.now()}@example.com`,
    });
    
    http.post('http://localhost:8080/api/users', payload);
}

// 결과:
// 테스트 1: 600개 생성
// 테스트 2: 600개 + 600개 = 1200개 (증가)
// 테스트 3: 1200개 + 600개 = 1800개 (증가)
// → DB 크기 폭증, 성능 저하
```

**해결책**:

```javascript
// 1. 테스트 전 DB 초기화

import exec from 'k6/execution';

export function setup() {
    // 테스트 시작 전 한 번만 실행
    http.request(
        'DELETE',
        'http://localhost:8080/api/users?cleanup=true',
        null
    );
}

export default function () {
    // 실제 테스트
    http.post('http://localhost:8080/api/users', payload);
}

// 2. 기존 사용자 재사용

const USERS = [
    { id: 1, name: 'User1' },
    { id: 2, name: 'User2' },
    { id: 3, name: 'User3' },
];

export default function () {
    // 새로 생성 대신 기존 사용자 사용
    const user = USERS[Math.floor(Math.random() * USERS.length)];
    http.get(`http://localhost:8080/api/users/${user.id}`);
}

// 3. Cleanup 함수

export function teardown(data) {
    // 테스트 끝난 후 정리
    http.request(
        'DELETE',
        'http://localhost:8080/api/users?cleanup=true',
        null
    );
}
```

</details>

---

[⬅️ 이전](./03-code-quality-gate.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./05-security-scanning.md)
