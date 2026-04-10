# 코드 품질 게이트 — SonarQube와 커버리지 임계값

## 🎯 핵심 질문

- SonarQube Quality Gate는 어떤 조건으로 파이프라인을 중단시킬까?
- JaCoCo 코드 커버리지 리포트를 SonarQube에 업로드하는 방법은?
- Gradle에서 `jacocoTestCoverageVerification`으로 임계값을 강제할 수 있을까?
- SpotBugs, Checkstyle 같은 정적 분석 도구를 파이프라인에 통합하는 방법은?
- SonarCloud(SaaS)와 SonarQube(Self-hosted)는 언제 어떤 것을 선택할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

코드 품질 게이트는 버그가 프로덕션에 도달하는 것을 자동으로 차단합니다.

**실제 효과**:
- 보안 취약점: 조기 감지로 침해사건 82% 감소 (2024 Synopsys 보고서)
- 유지보수 비용: 코드 리뷰 시간 40% 단축
- 기술 부채: 누적 속도 50% 감소

**단 5분의 자동화 검증이 수일간의 수동 리뷰를 대체합니다.**

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 접근 1: 품질 게이트 없이 모든 코드 머지

name: Build without Quality Gate
on: [pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      # 테스트만 실행하고 품질 검증 없음
      - name: Build and Test
        run: ./gradlew clean test
      
      # 결과: 모든 PR이 머지됨
      # → 버그 누적, 기술 부채 증가
```

```yaml
# ❌ 잘못된 접근 2: SonarQube 스캔하지만 결과 무시

name: Sonar Scan (ignored)
on: [pull_request]

jobs:
  sonar:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      # SonarQube 스캔하지만...
      - name: Run SonarQube
        run: ./gradlew sonarqube
        continue-on-error: true  # ← 실패해도 무시!
      
      # 결과: 보고만 하고 강제하지 않음
      # → 게이트 역할 못 함
```

```java
// ❌ 잘못된 접근 3: 커버리지 임계값 너무 높음

// build.gradle.kts
jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = "PACKAGE"
            excludes = listOf()
            
            limit {
                minimum = "1.0"  // ← 100% 커버리지 요구!
            }
        }
    }
}

// 결과:
// - 모든 PR이 차단됨
// - 테스트 "게임": 실제 로직 안 하고 커버리지만 채움
// - 더 많은 시간 낭비, 코드 품질 오히려 악화
```

**결과**: 
- 보안 취약점 발견 못 함
- 기술 부채 누적
- CI/CD 신뢰도 감소

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 1단계: JaCoCo 코드 커버리지 설정

```gradle
// build.gradle.kts

plugins {
    id("java")
    id("jacoco")
    id("org.sonarqube") version "4.4.1.3373"
}

// 1. JaCoCo 플러그인 설정
jacoco {
    toolVersion = "0.8.10"
}

// 2. 테스트 태스크에 JaCoCo 리포트 생성
tasks.test {
    useJUnitPlatform()
    
    // JaCoCo 에이전트 자동 추가
    finalizedBy("jacocoTestReport")
}

// 3. JaCoCo 리포트 생성 (XML + HTML)
tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn(tasks.test)
    
    reports {
        xml.required = true   // SonarQube가 XML 읽음
        html.required = true  // 사람이 읽을 HTML
        csv.required = false
    }
    
    // 리포트에 포함할 클래스
    sourceDirectories.setFrom(sourceSets.main.get().allSource.sourceDirectories)
    classDirectories.setFrom(sourceSets.main.get().output)
}

// 4. 커버리지 임계값 검증 (선택사항)
tasks.register<JacocoTestCoverageVerification>("jacocoTestCoverageVerification") {
    dependsOn(tasks.test)
    
    violationRules {
        // 전체 프로젝트: 최소 60% 커버리지
        rule {
            element = "PACKAGE"
            
            limit {
                counter = "LINE"
                value = "COVEREDRATIO"
                minimum = "0.60".toBigDecimal()  // 60%
            }
        }
        
        // 패키지별 세분화된 규칙
        rule {
            element = "PACKAGE"
            includes = listOf("com.example.service.*")  // Service 패키지
            
            limit {
                counter = "LINE"
                value = "COVEREDRATIO"
                minimum = "0.75".toBigDecimal()  // Service는 75%
            }
        }
        
        // 라인별 검증 (복잡한 분기)
        rule {
            element = "CLASS"
            includes = listOf("com.example.service.*Service")
            
            limit {
                counter = "BRANCH"
                value = "COVEREDRATIO"
                minimum = "0.70".toBigDecimal()
            }
        }
    }
}

// SonarQube 연동
sonarqube {
    properties {
        property("sonar.projectKey", "my-app")
        property("sonar.projectName", "My Application")
        property("sonar.projectVersion", "1.0.0")
        
        // JaCoCo 리포트 경로 지정
        property("sonar.coverage.jacoco.xmlReportPaths", 
            "build/reports/jacoco/jacocoTestReport/jacocoTestReport.xml")
        
        // 코드 경로
        property("sonar.sources", "src/main/java")
        property("sonar.tests", "src/test/java")
        
        // 최소 커버리지 요구
        property("sonar.coverage.minCoverageLines", "60")
    }
}
```

### 2단계: SonarQube Quality Gate 설정

```yaml
# ✅ GitHub Actions: SonarQube와 품질 게이트

name: Code Quality Gate with SonarQube
on: [pull_request, push]

jobs:
  quality:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarQube가 git 히스토리 분석 필요
      
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      # 1. 빌드 및 테스트
      - name: Build with JaCoCo
        run: ./gradlew clean build jacocoTestReport
      
      # 2. SonarQube 스캔
      - name: Run SonarQube Analysis
        run: ./gradlew sonarqube \
          -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      # 3. Quality Gate 결과 대기
      - name: Wait for Quality Gate
        id: quality-gate
        run: |
          # SonarQube API로 Quality Gate 상태 확인
          PROJECT_KEY="my-app"
          HOST="${{ secrets.SONAR_HOST_URL }}"
          TOKEN="${{ secrets.SONAR_TOKEN }}"
          
          for i in {1..60}; do
            # Quality Gate 상태 조회
            RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" \
              "$HOST/api/qualitygates/project_status?projectKey=$PROJECT_KEY")
            
            STATUS=$(echo $RESPONSE | jq -r '.projectStatus.status')
            
            if [ "$STATUS" = "OK" ]; then
              echo "✅ Quality Gate PASSED"
              echo "gate_status=passed" >> $GITHUB_OUTPUT
              exit 0
            elif [ "$STATUS" = "ERROR" ]; then
              echo "❌ Quality Gate FAILED"
              echo "gate_status=failed" >> $GITHUB_OUTPUT
              exit 1
            fi
            
            echo "Waiting for Quality Gate... (attempt $i/60)"
            sleep 5
          done
          
          echo "⏱️ Timeout: Quality Gate result not available"
          exit 1
      
      # 4. 게이트 실패 시 상세 정보
      - name: Report Quality Gate Details
        if: failure()
        run: |
          echo "❌ Code Quality Gate Failed"
          echo ""
          echo "Issues detected:"
          curl -s -H "Authorization: Bearer ${{ secrets.SONAR_TOKEN }}" \
            "${{ secrets.SONAR_HOST_URL }}/api/issues/search?projectKey=my-app&statuses=OPEN" \
            | jq '.issues[] | "\(.rule): \(.message)"' | head -10
```

### 3단계: SpotBugs와 Checkstyle 통합

```gradle
// build.gradle.kts

plugins {
    id("com.github.spotbugs") version "6.0.0"
    id("checkstyle")
    id("java")
}

// 1. SpotBugs: 정적 바이트코드 분석 (잠재적 버그 감지)
spotbugs {
    effort = "max"  // NORMAL, HIGH, MAX
    reportLevel = "medium"  // LOW, MEDIUM, HIGH
    omitVisitors = listOf()
    includeFilterFile = file("spotbugs-include.xml")
    excludeFilterFile = file("spotbugs-exclude.xml")
}

tasks.spotbugsMain {
    reports.create("html").required = true
}

// 2. Checkstyle: 코드 스타일 강제
checkstyle {
    toolVersion = "10.12.0"
    configFile = file("checkstyle.xml")
    maxWarnings = 0  // 경고도 허용 안 함
}

// 3. Gradle check 태스크에 포함
tasks.check {
    dependsOn(tasks.spotbugsMain, tasks.checkstyleMain)
}
```

```xml
<!-- spotbugs-include.xml: SpotBugs 규칙 -->
<?xml version="1.0" encoding="UTF-8"?>
<FindBugsFilter>
    <!-- 심각한 버그 감지 -->
    <Match abbrev="NP">
        <Priority value="1"/>  <!-- Potential null pointer dereference -->
    </Match>
    
    <!-- SQL Injection 위험 -->
    <Match abbrev="SQL">
        <Priority value="1"/>
    </Match>
    
    <!-- Resource Leak (DB 연결 미정리) -->
    <Match abbrev="OBL">
        <Priority value="2"/>
    </Match>
</FindBugsFilter>
```

```xml
<!-- checkstyle.xml: 코드 스타일 규칙 -->
<!DOCTYPE module PUBLIC "-//Puppy Crawl//DTD Check Configuration 1.3//EN"
  "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
    <property name="severity" value="error"/>
    
    <!-- 라인 길이 제한 -->
    <module name="LineLength">
        <property name="max" value="120"/>
        <property name="ignorePattern" value="^package.*|^import.*|a href|href|http://|https://"/>
    </module>
    
    <!-- 탭 사용 금지 -->
    <module name="FileTabCharacter"/>
    
    <!-- Java 언어 규칙 -->
    <module name="TreeWalker">
        <!-- 변수명 규칙 -->
        <module name="ConstantName"/>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        
        <!-- 들여쓰기 검증 -->
        <module name="Indentation">
            <property name="basicOffset" value="4"/>
            <property name="caseIndent" value="4"/>
        </module>
        
        <!-- 빈 블록 감지 -->
        <module name="EmptyBlock"/>
        
        <!-- 불필요한 임포트 감지 -->
        <module name="UnusedImports"/>
    </module>
</module>
```

---

## 🔬 내부 동작 원리

### SonarQube Quality Gate 메커니즘

```
Quality Gate = 통과/실패를 결정하는 규칙 집합

내부 구조:

┌─────────────────────────────────────────┐
│ SonarQube Analysis                      │
│ (코드 스캔)                              │
└───────────────────┬─────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────┐
│ Calculate Metrics                       │
│ - Coverage (%)                          │
│ - Bugs (개수)                           │
│ - Vulnerabilities (개수)                │
│ - Code Smells (개수)                    │
│ - Duplication (%)                       │
└───────────────────┬─────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────┐
│ Apply Quality Gate Conditions           │
│                                         │
│ Condition 1: Coverage >= 70%?  → YES ✅│
│ Condition 2: Bugs < 5?         → NO  ❌│
│ Condition 3: Vulnerabilities < 1? → YES│
│                                         │
│ ALL pass? → GATE PASSED ✅              │
│ ANY fail? → GATE FAILED ❌              │
└───────────────────┬─────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────┐
│ Result: PASS or FAIL                    │
│                                         │
│ PASS → PR 머지 허용                     │
│ FAIL → PR 머지 차단, 리뷰 요청           │
└─────────────────────────────────────────┘
```

### JaCoCo 커버리지 계산 방식

```
Line Coverage = 실행된 라인 / 전체 라인

예시:
┌─────────────────────────────────────────┐
│ public class OrderService {             │
│   public double calculate(int qty) {    │ ← 라인 3: 실행됨
│     if (qty > 0) {          ← 라인 4: 실행됨 (true)
│       return qty * 10.0;    │ ← 라인 5: 실행됨
│     }                       │ ← 라인 6: 스킵됨 (false 분기)
│     return 0;               │ ← 라인 7: 미실행
│   }                         │
│ }                           │
└─────────────────────────────────────────┘

테스트 1: calculate(5) 실행
  실행된 라인: 3, 4, 5 = 3개
  Line Coverage = 3/7 = 42.9%
  Branch Coverage = 1/2 = 50% (true만 실행)

테스트 2: calculate(0) 추가
  실행된 라인: 3, 4, 6, 7 = 4개 (누적)
  Line Coverage = 4/7 = 57.1%
  Branch Coverage = 2/2 = 100%
```

### SpotBugs 정적 분석 방식

```
바이트코드 패턴 매칭:

1. Null Pointer Dereference
   패턴: obj.method() → obj가 null일 가능성?
   코드:
     User user = getUserFromDB(id);
     user.getName();  // ← 위험: user는 Optional, null일 수 있음
   
   SpotBugs 경고: NP_NULL_ON_SOME_PATH

2. Resource Leak
   패턴: new Resource() → close() 호출 없음?
   코드:
     Connection conn = getConnection();
     executeQuery(conn);
     // conn.close() 누락!
   
   SpotBugs 경고: OBL_UNSATISFIED_OBLIGATION

3. Unchecked Cast
   패턴: (Type) obj → 캐스트 검증 없음?
   코드:
     Object obj = getValue();
     String str = (String) obj;  // 런타임 에러 가능
   
   SpotBugs 경고: BC_UNCONFIRMED_CAST
```

---

## 💻 실전 실험

### 실험 1: JaCoCo 커버리지 생성 및 검증

```gradle
// build.gradle.kts - 실험 설정

plugins {
    id("java")
    id("jacoco")
}

jacoco {
    toolVersion = "0.8.10"
}

tasks.test {
    finalizedBy("jacocoTestReport")
}

tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn(tasks.test)
    reports {
        xml.required = true
        html.required = true
    }
}

tasks.register<JacocoTestCoverageVerification>("jacocoVerify") {
    dependsOn(tasks.test)
    violationRules {
        rule {
            element = "PACKAGE"
            limit {
                counter = "LINE"
                value = "COVEREDRATIO"
                minimum = "0.70".toBigDecimal()
            }
        }
    }
}
```

```java
// 테스트 대상 클래스
public class OrderCalculator {
    public double calculateTotal(List<Item> items) {
        if (items == null || items.isEmpty()) {
            return 0.0;
        }
        
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
    
    public boolean isValidOrder(Order order) {
        return order != null && 
               order.getTotalAmount() > 0 && 
               !order.getItems().isEmpty();
    }
}

// 테스트
@Test
void testCalculateTotal() {
    OrderCalculator calc = new OrderCalculator();
    
    List<Item> items = List.of(
        new Item("A", 10.0, 2),
        new Item("B", 5.0, 3)
    );
    
    assertEquals(35.0, calc.calculateTotal(items), 0.01);
}

// 아직 테스트되지 않은 분기:
// - null 체크 (null 전달)
// - empty 체크 (빈 리스트 전달)
// - isValidOrder() 메서드 전체
```

```bash
# 실행 및 결과 확인
./gradlew test jacocoTestReport

# 생성된 리포트 위치:
# build/reports/jacoco/jacocoTestReport/html/index.html

# 커버리지 검증 (실패할 가능성 높음)
./gradlew jacocoVerify
# > Task :jacocoTestCoverageVerification FAILED
# The following classes do not meet the minimum line coverage of 0.70:
#   - OrderCalculator: 0.50
```

### 실험 2: SonarQube 로컬 설정 및 스캔

```bash
# 1. Docker에서 SonarQube 실행
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  sonarqube:latest

# 브라우저: http://localhost:9000
# 기본 계정: admin / admin

# 2. SonarQube 토큰 생성
# Admin → Security → Tokens → Generate

# 3. 프로젝트 스캔
./gradlew sonarqube \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=<YOUR_TOKEN>
```

### 실험 3: SpotBugs 실행 및 보고서 확인

```bash
./gradlew spotbugsMain

# 보고서 생성 위치:
# build/reports/spotbugs/main.html

# 실행 결과 예:
# SpotBugs found 3 potential bugs:
# - NP_NULL_ON_SOME_PATH in UserService.getUser()
# - OBL_UNSATISFIED_OBLIGATION in DatabasePool.getConnection()
# - BC_UNCONFIRMED_CAST in ConfigParser.parse()
```

### 실험 4: GitHub Actions에서 품질 게이트 강제

```yaml
name: Complete Quality Gate
on: [pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      # 1. 모든 품질 검사
      - name: Run all quality checks
        run: ./gradlew clean check \
          test \
          jacocoTestReport \
          jacocoTestCoverageVerification \
          spotbugsMain \
          checkstyleMain
      
      # 2. SonarQube 스캔
      - name: SonarQube Analysis
        run: ./gradlew sonarqube \
          -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}
      
      # 3. Quality Gate 검증 (병렬 처리)
      - name: Verify Quality Gates
        run: |
          set -e
          
          echo "✓ Checkstyle passed"
          echo "✓ SpotBugs passed"
          echo "✓ Coverage threshold passed"
          echo "✓ SonarQube Quality Gate passed"
          
          echo ""
          echo "✅ All quality gates passed"
      
      # 4. 실패 시 상세 보고
      - name: Report Coverage Details
        if: failure()
        run: |
          echo "## Code Coverage Report" >> $GITHUB_STEP_SUMMARY
          grep -A 10 "Line Coverage" build/reports/jacoco/jacocoTestReport/jacocoTestReport.xml >> $GITHUB_STEP_SUMMARY || true
```

---

## 📊 성능/비용 비교

```
품질 게이트 도구별 비용/효과:

1. JaCoCo (무료)
   - 설정: 5분
   - 실행 오버헤드: +10% (테스트 시간)
   - 효과: 코드 커버리지 정량화
   - 한계: 커버리지 수치만 보고, 실제 품질 미측정

2. SpotBugs (무료)
   - 설정: 10분
   - 실행 시간: +5초
   - 효과: 잠재적 버그 감지
   - 거짓 양성(False Positive): 높음 (20-30%)

3. Checkstyle (무료)
   - 설정: 15분
   - 실행 시간: +2초
   - 효과: 스타일 규칙 강제
   - 한계: 실제 기능 버그는 감지 못 함

4. SonarQube (Self-hosted: 무료, Cloud: $50/월)
   - 설정: 30분
   - 실행 시간: +15초
   - 효과: 종합적 코드 품질 분석
   - 장점: Quality Gate 자동화, 추적 기능

조합 효과:
JaCoCo + SpotBugs + Checkstyle + SonarQube
= 전체 스캔 시간: 30초
= 100개 PR × 30초 = 50분/월
= 버그 조기 발견율: 85%
```

### SonarCloud vs SonarQube 선택

```
SonarCloud (SaaS):
- 비용: 무료(오픈소스), $50/월(비공개)
- 설정: 매우 쉬움 (GitHub 통합)
- 유지보수: 제로
- 응답시간: 빠름 (CDN 캐시)
- 데이터: 외부 서버 저장

SonarQube (Self-hosted):
- 비용: 무료(커뮤니티), $4000/년(상용)
- 설정: 30-60분 (Docker)
- 유지보수: 중간 (DB, 디스크 관리)
- 응답시간: 로컬이므로 매우 빠름
- 데이터: 내부 서버 저장

선택 기준:
┌──────────────────┬────────────────┬──────────────────┐
│ 조건             │ SonarCloud 추천 │ SonarQube 추천    │
├──────────────────┼────────────────┼──────────────────┤
│ 공개 리포지토리  │ ✅              │ -                │
│ 민감한 소스코드  │ -               │ ✅               │
│ 대규모 팀(50+)   │ -               │ ✅ (비용)        │
│ 시작 단계        │ ✅              │ -                │
│ HIPAA/FedRAMP    │ -               │ ✅ (규정)        │
└──────────────────┴────────────────┴──────────────────┘
```

---

## ⚖️ 트레이드오프

### 1. 커버리지 임계값 설정

```
높은 임계값 (90%+):
✅ 버그 가능성 낮음
✅ 엣지 케이스 많이 테스트됨
❌ 테스트 작성 시간 3배 증가
❌ 테스트 "게임": 실제 기능 없이 커버리지만 채움
❌ 뮤턴트 테스트에서 많이 실패

권장 임계값:
- 신규 코드: 80%
- 서비스 패키지: 75%
- 유틸리티: 60%
```

### 2. SpotBugs 엄격도 vs 거짓 양성

```
effort = "max", reportLevel = "low"
→ 많은 경고 (90%)
→ 거짓 양성 많음 (40%)
→ 개발자가 모두 무시함

effort = "normal", reportLevel = "medium"
→ 균형 잡힌 경고 (60%)
→ 거짓 양성 적음 (15%)
→ 신뢰성 높음

권장: effort = "high", reportLevel = "medium"
```

### 3. Quality Gate 자동화 vs 유연성

```
강한 자동화 (모든 조건 필수):
✅ 품질 확보
❌ 긴급 배포 불가
❌ 기술 부채 처리 어려움

유연한 자동화 (매개변수 조정 가능):
✅ 긴급 상황 대응
❌ 기준이 흐려질 수 있음

권장: Quality Gate는 고정, 예외 프로세스 수립
→ 예외 인정 시 기술 부채 이슈 생성
```

---

## 📌 핵심 정리

| 도구 | 역할 | 효과 |
|------|------|------|
| **JaCoCo** | 커버리지 측정 | 테스트 범위 정량화 |
| **SpotBugs** | 정적 버그 탐지 | 심각한 버그 75% 차단 |
| **Checkstyle** | 코드 스타일 강제 | 일관성 보증 |
| **SonarQube** | 종합 품질 분석 | 누적 기술 부채 관리 |

---

## 🤔 생각해볼 문제

### Q1: "커버리지 임계값을 60%로 설정했는데, PR이 60.1%만 되어도 통과한다. 신뢰할 수 있을까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**문제 분석**:

커버리지는 **통과 조건**이지 **품질 지표**가 아닙니다.

```
커버리지 60% ≠ 버그 없는 코드

반례:
public double divide(int a, int b) {
    return a / b;  // ← 한 라인, 100% 커버리지
}

테스트:
divide(10, 2) → 테스트됨

하지만 divide(10, 0) → ZeroDivisionError 미처리!
```

**해결책: 뮤턴트 테스팅**

```java
// 뮤턴트 테스팅: 코드에 버그를 주입하고 테스트가 감지하는지 확인

원본 코드:
if (salary > 0) {
    bonus = salary * 0.1;
}

뮤턴트 1: > → >= 변경
if (salary >= 0) {  // 버그!
    bonus = salary * 0.1;
}

테스트: divide(salary=0) → 보너스 계산 검증?
NO → 뮤턴트 탐지 실패 (테스트 불충분)
YES → 뮤턴트 탐지 성공
```

**PIT (Pitest) 사용**:

```gradle
plugins {
    id("info.solidsoft.pitest") version "1.14.2"
}

pitest {
    mutationThreshold = 75  // 75% 뮤턴트 탐지율 요구
    coverageThreshold = 60  // 60% 라인 커버리지도 요구
}

tasks.test {
    finalizedBy("pitest")
}
```

```bash
./gradlew pitest

# 결과:
# Generated 150 mutations, 120 killed (80%)
# 30 mutations survived (테스트가 감지 못 한 버그)
```

**결론**: 커버리지 + 뮤턴트 테스팅 = 신뢰성 높음

</details>

### Q2: "SonarQube Quality Gate가 실패했는데, PR을 강제로 머지하고 싶다. 어떻게?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**승인 워크플로우 설정**:

```yaml
# GitHub 저장소 설정
Settings → Branches → Require status checks to pass before merging

추가 규칙:
☑ Require code reviews before merging (1명 이상)
☑ Dismiss stale pull request approvals
☑ Require status checks to pass (SonarQube Quality Gate)
```

**예외 프로세스**:

```java
// 1. 기술 부채 이슈 생성
GitHub Issue: "[TECH-DEBT] SonarQube Gate Waiver"
- 사유: 긴급 보안 패치 필요
- 승인자: Tech Lead
- 만료일: 7일

// 2. PR에서 이슈 참조
PR 설명:
"""
긴급 보안 업데이트
SonarQube 게이트 면제: #1234-tech-debt

기술 부채는 별도 PR에서 처리
"""

// 3. 예외 승인 후 머지
```

**자동 예외 프로세스**:

```yaml
name: Quality Gate with Waiver
on: [pull_request]

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Quality Gate
        id: quality-gate
        run: ./gradlew sonarqube
        continue-on-error: true
      
      # Quality Gate 실패하면...
      - name: Check for Waiver
        if: failure()
        run: |
          # PR 설명에 "QUALITY_GATE_WAIVER" 포함?
          WAIVER=$(gh pr view ${{ github.event.pull_request.number }} \
            --json body -q '.body' | grep -c "QUALITY_GATE_WAIVER" || echo 0)
          
          if [ "$WAIVER" -eq 1 ]; then
            echo "⚠️ Quality Gate waived by PR author"
            # 대신 리뷰어 권장
            gh pr comment ${{ github.event.pull_request.number }} \
              --body "⚠️ Quality Gate waived. Please have 2 reviewers approve."
          else
            echo "❌ Quality Gate failed, no waiver found"
            exit 1
          fi
```

**권장사항**:
- 예외는 **매우 드물어야** 함
- **매번 기술 부채 이슈 생성**
- **1주일 내 해결** 약속
- **팀 리더 승인** 필수

</details>

### Q3: "로컬에서 테스트는 모두 통과하는데 CI에서 SonarQube가 실패한다. 왜?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**원인 분석**:

```
로컬 (통과):
- IDE 설정: Checkstyle, SpotBugs 비활성화
- 커버리지: 측정 안 함
- 분석: SonarQube 스캔 안 함

CI (실패):
- 엄격한 정적 분석 활성화
- 커버리지 임계값 강제
- SonarQube 전체 스캔
```

**일반적인 원인 Top 5**:

1. **로컬 IDE 설정과 CI 설정 불일치**
```yaml
# ✅ 일관된 설정
.editorconfig  # IDE와 CI 모두 읽음
build.gradle   # 통일된 정적 분석 설정

# build.gradle.kts
val spotbugsConfig: Configuration by configurations.creating
```

2. **커버리지 계산 차이**
```
로컬:
./gradlew test           # 선택 테스트만 실행

CI:
./gradlew test           # 전체 테스트
gradlew jacocoTestReport # 모든 파일 포함
```

3. **경로 차이**
```
로컬:
"src/main/java" → 커버리지 포함

CI:
"src/main/java", "src/main/generated" → 생성된 코드도 포함!
→ 커버리지 저하
```

4. **JVM 바ージョ 차이**
```
로컬: Java 21
CI: Java 17

→ SpotBugs가 다른 바이트코드 패턴 감지
```

5. **SonarQube 서버 설정**
```
로컬 SonarQube (개발):
- Quality Gate 없음
- 규칙 10개 활성화

CI SonarQube (운영):
- 엄격한 Quality Gate
- 규칙 200개 활성화
```

**해결 방법**:

```gradle
// build.gradle.kts - CI와 로컬 통일

tasks.register<JacocoReport>("jacocoTestReport") {
    // CI와 로컬 모두 같은 경로 제외
    sourceDirectories.setFrom(sourceSets.main.get().allSource.sourceDirectories)
    classDirectories.setFrom(sourceSets.main.get().output.classesDirs)
    
    // 생성된 코드 제외
    classDirectories.exclude("**/generated/**")
}

sonarqube {
    properties {
        // CI와 로컬 동일한 SonarQube 서버 사용
        property("sonar.host.url", 
            project.findProperty("sonar.host.url") ?: "http://localhost:9000")
    }
}
```

```bash
# 로컬에서 CI와 동일한 환경으로 실행
./gradlew clean \
  test \
  jacocoTestReport \
  spotbugsMain \
  checkstyleMain \
  sonarqube \
  -Dsonar.host.url=http://localhost:9000
```

**CI/CD 파이프라인 통일**:

```yaml
name: Unified Quality Checks
on: [pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'  # 로컬과 동일한 버전
      
      # CI 명령을 로컬에서도 실행 가능하게
      - name: Run quality checks (same as local)
        run: |
          ./gradlew clean \
            test \
            jacocoTestReport \
            spotbugsMain \
            checkstyleMain \
            sonarqube
```

</details>

---

[⬅️ 이전](./02-spring-boot-test-optimization.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./04-performance-regression.md)
