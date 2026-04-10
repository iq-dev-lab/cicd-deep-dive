# 보안 스캐닝 자동화 — CVE 감지와 Secret 노출 방지

## 🎯 핵심 질문

- Trivy로 Docker 이미지의 OS 패키지 및 라이브러리 취약점을 어떻게 스캔할까?
- HIGH/CRITICAL CVE 발견 시 파이프라인을 자동으로 차단할 수 있을까?
- OWASP Dependency-Check로 Java 라이브러리의 알려진 취약점을 감지하는 방법은?
- TruffleHog로 실수로 커밋된 API 키, 비밀번호를 감지하는 방법은?
- `git-secrets` commit hook으로 Secret 패턴을 커밋 전에 차단할 수 있을까?
- SAST(정적)와 DAST(동적) 테스트의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**보안 취약점은 침묵 속에서 번식합니다.**

실제 사건:
- 2022년 Log4Shell: 의존 라이브러리 미패치 → 전 세계 수백만 개 시스템 노출
- 2023년 GitHub 누수: 개발자가 실수로 API 키 커밋 → 계정 탈취
- 2024년 공급망 공격: npm 패키지에 악성 코드 삽입 → 다운스트림 피해

**자동화된 보안 스캔이 방어합니다:**
- 빌드 시간에 취약점 검증: 배포 전 차단
- Secret 누출 방지: 코밋 단계에서 차단
- CVE 자동 업데이트: 신규 취약점 조기 감지

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 접근 1: 보안 스캔 없이 배포

name: Build and Deploy
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - name: Build
        run: ./gradlew build
      
      - name: Deploy
        run: ./deploy.sh
      
      # 결과: 취약점이 있는 라이브러리를 프로덕션에 배포
      # → 보안 침해 위험
```

```bash
# ❌ 잘못된 접근 2: API 키를 코드에 하드코드

# application.properties
api.key=sk-1234567890abcdef
aws.secret=AKIAIOSFODNN7EXAMPLE

# build.gradle (커밋됨)
credentials {
    github.token = "ghp_1234567890abcdefghijklmnop"
}

# 결과: git 히스토리에 영구적으로 남음
# → 누구나 접근 가능
```

```gradle
// ❌ 잘못된 접근 3: 의존성 업데이트 무시

dependencies {
    // 6개월 전 버전 (수십 개의 CVE 있음)
    implementation("org.springframework.boot:spring-boot-starter-web:3.0.0")
    
    // 알려진 취약점 라이브러리
    implementation("log4j:log4j:1.2.17")  // Log4Shell 포함!
}

// 결과: 정기적인 취약점 업데이트 없음
```

**결과**:
- 보안 침해 위험
- 규정 준수 실패 (HIPAA, PCI-DSS)
- 신뢰도 감소

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 1단계: Docker 이미지 취약점 스캔 (Trivy)

```yaml
# ✅ GitHub Actions: Trivy로 이미지 스캔

name: Security Scanning - Trivy
on: [push, pull_request]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 1. Docker 이미지 빌드
      - name: Build Docker Image
        run: |
          docker build -t myapp:latest .
      
      # 2. Trivy 스캔 (취약점 감지)
      - name: Run Trivy Vulnerability Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:latest'
          format: 'sarif'  # GitHub Security tab에 표시
          output: 'trivy-results.sarif'
          exit-code: '1'  # 취약점 발견 시 실패
          # 심각도 임계값
          severity: 'CRITICAL,HIGH'
      
      # 3. 결과 업로드 (GitHub에 표시)
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### 2단계: Java 라이브러리 취약점 스캔 (OWASP Dependency-Check)

```gradle
// build.gradle.kts

plugins {
    id("java")
    id("org.owasp.dependencycheck") version "9.0.0"
}

// Dependency-Check 설정
dependencyCheck {
    // NVD(National Vulnerability Database) 자동 업데이트
    nvdApiKey = System.getenv("NVD_API_KEY")  // 더 빠른 스캔용
    
    // 보고서 형식
    format = "ALL"  // HTML, XML, JSON 모두 생성
    
    // 검사 범위
    skipConfiguration = false
    skipRuntimeScope = false
    skipProvidedScope = false
    
    // 취약점 임계값
    failBuildOnCVSS = 7.0f  // CVSS 7.0 이상이면 빌드 실패
}

tasks.check {
    dependsOn(tasks.dependencyCheckAnalyze)
}
```

```yaml
# GitHub Actions: Dependency-Check 통합

name: Security Scanning - Dependency Check
on: [push, pull_request]

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      # 1. 의존성 취약점 스캔
      - name: Run OWASP Dependency-Check
        run: ./gradlew dependencyCheckAnalyze
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        continue-on-error: true
      
      # 2. 보고서 생성
      - name: Upload Dependency-Check Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: dependency-check-report
          path: build/reports/dependency-check/
      
      # 3. CVSS 점수로 실패 여부 판단
      - name: Check Dependency-Check Results
        run: |
          if grep -q 'CVSS Score.*[7-9]\.' build/reports/dependency-check/dependency-check-report.json; then
            echo "❌ High severity vulnerability detected"
            exit 1
          fi
```

### 3단계: Secret 감지 (TruffleHog + git-secrets)

```yaml
# ✅ GitHub Actions: TruffleHog로 Secret 감지

name: Security Scanning - Secret Detection
on: [push, pull_request]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 분석
      
      # 1. TruffleHog: 리포지토리 전체 스캔
      - name: Run TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --json
        continue-on-error: true
      
      # 2. git-secrets: 커밋 전 차단
      - name: Install git-secrets
        run: |
          git clone https://github.com/awslabs/git-secrets.git /tmp/git-secrets
          cd /tmp/git-secrets && make install
      
      # 3. 기본 패턴 등록
      - name: Configure git-secrets patterns
        run: |
          git secrets --register-aws
          
          # 커스텀 패턴
          git secrets --add -a 'AKIA[0-9A-Z]{16}'  # AWS Access Key
          git secrets --add -a 'sk-[a-zA-Z0-9]{48}'  # OpenAI API Key
          git secrets --add -a 'ghp_[a-zA-Z0-9]{36}'  # GitHub Token
          git secrets --add -a '-----BEGIN RSA PRIVATE KEY'  # 개인 키
      
      # 4. 현재 리포지토리 스캔
      - name: Scan current repo with git-secrets
        run: git secrets --scan
```

```bash
# 로컬 개발 환경에서 git-secrets 설정

# 1. 설치
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets && make install

# 2. 리포지토리에 hook 설치
cd ~/my-project
git secrets --install

# 3. 패턴 등록
git secrets --register-aws
git secrets --add -a 'api[_-]?key'
git secrets --add -a 'secret[_-]?key'

# 4. 테스트: Secret이 포함된 커밋 시도
echo "api_key = sk-1234567890" > config.txt
git add config.txt
git commit -m "Add config"

# 결과:
# ❌ git-secrets: matched one or more blacklisted patterns
# Matched: api_key = sk-1234567890
# Commit rejected!
```

### 4단계: 통합 보안 파이프라인

```yaml
# ✅ 완전한 보안 스캔 파이프라인

name: Complete Security Pipeline
on: [push, pull_request]

jobs:
  # 1. 컨테이너 이미지 취약점
  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t myapp:latest .
      
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:latest'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
  
  # 2. 라이브러리 의존성 취약점
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      - name: OWASP Dependency Check
        run: ./gradlew dependencyCheckAnalyze
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
  
  # 3. Secret 노출 감지
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
  
  # 4. 코드 정적 분석 (SAST)
  sast-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: SonarQube SAST
        run: |
          ./gradlew sonarqube \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}
  
  # 5. 모든 스캔 통과 후 배포
  deploy:
    needs: [container-scan, dependency-scan, secret-scan, sast-scan]
    runs-on: ubuntu-latest
    if: success()
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: ./deploy.sh
```

---

## 🔬 내부 동작 원리

### Trivy 취약점 스캔 메커니즘

```
Trivy 스캔 프로세스:

1. 이미지 분석
   Docker 이미지 압축 해제
   ├─ OS 패키지 목록 추출
   │  ├─ apt (Debian/Ubuntu)
   │  ├─ rpm (RHEL/CentOS)
   │  ├─ apk (Alpine)
   │  └─ etc.
   │
   ├─ Application 라이브러리 목록
   │  ├─ Java JAR (SBOM)
   │  ├─ Python wheel
   │  ├─ npm packages
   │  └─ etc.
   │
   └─ 언어별 빌드 파일
      ├─ pom.xml (Maven)
      ├─ build.gradle (Gradle)
      └─ package.json (Node.js)

2. 데이터베이스 조회
   각 패키지/라이브러리를 NVD(National Vulnerability Database)와 비교
   ├─ Java: org.springframework:spring-core:5.0.0 → CVE-2019-1010023
   ├─ OS: openssl 1.0.2 → CVE-2016-2105
   └─ etc.

3. 결과 생성
   취약점 목록 + 심각도
   ├─ CRITICAL: CVSS >= 9.0
   ├─ HIGH: CVSS 7.0-8.9
   ├─ MEDIUM: CVSS 4.0-6.9
   └─ LOW: CVSS 0.1-3.9

4. 결정
   exit-code 1 → 파이프라인 실패
```

### Dependency-Check 의존성 분석

```
Build Artifact에서 라이브러리 추출:

pom.xml / build.gradle 파싱
     ↓
모든 전이 의존성(Transitive Dependencies) 수집
     ↓
각 라이브러리의 CPE(Common Platform Enumeration) 생성
예: cpe:/a:apache:log4j:1.2.17
     ↓
NVD 데이터베이스 조회
   log4j 1.2.17 → [CVE-2021-44228, CVE-2019-17571, ...]
     ↓
CVSS 점수 계산 및 리포트 생성
```

### TruffleHog 패턴 매칭

```
Secret 탐지 방식:

1. 엔트로피 기반 탐지
   "api_key = sk-1234567890abcdefghijklmnop"
   → 문자 다양성 분석
   → 랜덤 문자열 같음 → 의심

2. 정규식 패턴 매칭
   AWS Access Key: AKIA[0-9A-Z]{16}
   GitHub Token: ghp_[a-zA-Z0-9]{36}
   OpenAI Key: sk-[a-zA-Z0-9]{48}

3. Validator 검증 (API 호출)
   GitHub Token → GitHub API 호출
   → 유효한 토큰 확인
   → 위험도 높음 (실제 사용 가능)

4. Git 히스토리 스캔
   모든 커밋, 브랜치, 태그 조회
   각 파일 버전의 변경사항 검사
   → 삭제된 Secret도 찾음
```

### 정적(SAST) vs 동적(DAST) 분석

```
SAST (Static Application Security Testing):

코드 분석 (실행 없음)
   ↓
패턴 매칭 + 데이터 흐름 분석
   ├─ SQL Injection 위험
   ├─ XSS 위험
   ├─ Hard-coded Credentials
   └─ etc.

장점:
✅ 빠름 (수초)
✅ 비용 낮음
✅ 개발 초기 감지

단점:
❌ 거짓 양성 높음
❌ 실제 취약점인지 확인 필요


DAST (Dynamic Application Security Testing):

실행 중인 애플리케이션 공격 시뮬레이션
   ↓
입력값 변조 → 응답 분석
   ├─ SQL Injection: ' OR '1'='1
   ├─ XSS: <script>alert('xss')</script>
   └─ CSRF: 토큰 검증 확인

장점:
✅ 실제 취약점만 발견
✅ 거짓 양성 낮음

단점:
❌ 느림 (수분)
❌ 비용 높음
❌ 테스트 환경 필요
```

---

## 💻 실전 실험

### 실험 1: Trivy로 로컬 이미지 스캔

```bash
# 1. Trivy 설치
# macOS:
brew install trivy

# Linux:
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# 2. Docker 이미지 빌드
docker build -t vulnerable-app:latest .

# 3. Trivy 스캔
trivy image vulnerable-app:latest

# 출력 예:
# ...
# (base)
# Total: 45 (CRITICAL: 12, HIGH: 20, MEDIUM: 13)
#
# CRITICAL: 12 vulnerabilities
# HIGH: 20 vulnerabilities
```

```dockerfile
# Dockerfile - 의도적으로 취약점 포함

# ❌ 오래된 Alpine (많은 취약점)
FROM alpine:3.10

# ❌ 루트로 실행
USER root

# ❌ 의존성 업데이트 안 함
RUN apk add --no-cache \
    openssl=1.0.2 \
    curl=7.65.3
```

### 실험 2: OWASP Dependency-Check 로컬 실행

```bash
# 1. Dependency-Check 다운로드
wget https://github.com/jeremylong/DependencyCheck_CLI/releases/download/v8.3.1/dependency-check-8.3.1-release.zip
unzip dependency-check-8.3.1-release.zip
cd dependency-check/bin

# 2. 프로젝트 스캔
./dependency-check.sh --project "MyApp" \
  --scan /path/to/project \
  --format HTML \
  --out ./reports

# 출력 예:
# [INFO] Check Started
# [INFO] Indexing Artifacts
# [INFO] Artifact Identification Complete (XX artifacts)
# [INFO] Analysis Starting
# [INFO] Total Dependencies: 45
# [INFO] Vulnerable Dependencies: 8
#
# Severity: HIGH
#   - log4j-core: 2.14.1 → CVE-2021-44228 (CVSS: 10.0)
```

### 실험 3: TruffleHog로 Secret 감지

```bash
# 1. TruffleHog 설치
pip install truffleHog

# 2. 저장소 스캔
trufflehog git https://github.com/my-org/my-repo.git

# 3. JSON 형식으로 상세 보고서
trufflehog git https://github.com/my-org/my-repo.git --json > secrets-report.json

# 결과 예:
# {
#   "Verified": true,
#   "Secret": "ghp_1234567890abcdefghijklmnopqrstuvwxyz",
#   "Type": "GitHub Token",
#   "Commit": "abc123def456"
# }

# 4. 특정 파일만 스캔
trufflehog filesystem ./src --regex

# 5. 커스텀 패턴
trufflehog filesystem ./src \
  --custom-pattern "api[_-]?key\s*=\s*[a-zA-Z0-9_-]{20,}"
```

### 실험 4: git-secrets 로컬 설정

```bash
# 1. 설치
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets
make install

# 2. 프로젝트에 설치
cd ~/my-project
git secrets --install

# 3. 기본 패턴 등록
git secrets --register-aws

# 4. 커스텀 패턴 추가
git secrets --add -a 'BEGIN RSA PRIVATE KEY'
git secrets --add -a 'BEGIN OPENSSH PRIVATE KEY'
git secrets --add -a 'api[_-]?key\s*[=:]\s*'

# 5. 패턴 목록 확인
git config --local core.hooksPath .git/hooks
git secrets --list

# 6. 테스트: Secret 포함 커밋
echo "SECRET_KEY=sk-1234567890abcdef" > secret.txt
git add secret.txt
git commit -m "Add secret"

# 결과:
# ❌ COMMIT REJECTED
# matched one or more blacklisted patterns
#
# Matched: SECRET_KEY=sk-1234567890abcdef
```

### 실험 5: 통합 보안 파이프라인 로컬 테스트

```bash
# 1. 모든 스캔 실행 (CI와 동일)
./run-security-checks.sh

#!/bin/bash
set -e

echo "=== 1. Building Docker Image ==="
docker build -t myapp:test .

echo "=== 2. Trivy Image Scan ==="
trivy image --severity HIGH,CRITICAL \
  --exit-code 1 \
  myapp:test

echo "=== 3. Dependency Check ==="
./gradlew dependencyCheckAnalyze

echo "=== 4. TruffleHog Scan ==="
trufflehog git . --json > /tmp/trufflehog-results.json
if [ -s /tmp/trufflehog-results.json ]; then
  echo "❌ Secrets detected!"
  cat /tmp/trufflehog-results.json
  exit 1
fi

echo "=== 5. git-secrets Verification ==="
git secrets --scan

echo ""
echo "✅ All security checks passed!"
```

---

## 📊 성능/비용 비교

```
보안 스캔 도구별 성능:

1. Trivy (Container):
   시간: 10-30초/이미지
   비용: 무료
   DB 업데이트: 매일
   정확도: 높음 (95%)

2. Dependency-Check (라이브러리):
   시간: 1-3분 (초회), 10초 (캐시)
   비용: 무료
   DB 업데이트: 매일
   정확도: 높음 (95%)

3. TruffleHog (Secret):
   시간: 5-30초 (저장소 크기에 따라)
   비용: 무료
   실시간: 커밋 시점
   정확도: 매우 높음 (99%)

4. git-secrets (로컬):
   시간: 100ms (커밋당)
   비용: 무료
   실시간: 커밋 전 차단
   정확도: 매우 높음 (99%)

월간 비용 (100 PR 기준):
- Trivy: 100 × 30s = 50분 (CI 시간)
- Dependency-Check: 100 × 2m = 200분
- TruffleHog: 100 × 10s = 16분
- 합계: 266분 = 4.4시간 (무료)

보안 위반 방지 효과:
- Secret 노출 방지: 월 1-2건 (개발자 실수)
- CVE 조기 감지: 월 3-5건 (알려진 취약점)
- 침해사건 방지: 연 1건 (큰 비용 절감)
```

---

## ⚖️ 트레이드오프

### 1. 보안 임계값 설정

```
높은 임계값 (모든 취약점 차단):
✅ 최고 보안 수준
❌ 빌드 지연 (3개월 이상 지원 종료 라이브러리)
❌ 라이센스 호환성 문제 (업데이트 불가)

낮은 임계값 (CRITICAL만 차단):
✅ 빠른 개발
❌ MEDIUM, HIGH 취약점 누적

권장: CRITICAL + HIGH 차단, MEDIUM은 30일 내 해결
```

### 2. 스캔 속도 vs 정확도

```
빠른 스캔:
- Trivy (10초): 알려진 CVE만
- 정확도: 90%

정확한 스캔:
- Snyk (2분): 엔트로피 + 정규식 + API 검증
- 정확도: 99%

권장: 로컬은 빠르게, CI는 정확하게
```

### 3. False Positive 처리

```
매우 엄격한 정책:
❌ Secret 같은 문자열에 경고 (랜덤 ID 등)
❌ 모든 경고 해결 필요 (시간 낭비)

합리적인 정책:
✅ 검증된 토큰만 경고 (API 호출 확인)
✅ Known False Positive는 예외 처리
```

---

## 📌 핵심 정리

| 도구 | 역할 | 감지 대상 |
|------|------|----------|
| **Trivy** | 컨테이너 취약점 | OS 패키지, 라이브러리 |
| **Dependency-Check** | 라이브러리 취약점 | Maven, Gradle, npm |
| **TruffleHog** | Secret 노출 | API 키, 토큰, 비밀번호 |
| **git-secrets** | 커밋 차단 | AWS, GitHub, 커스텀 패턴 |
| **SonarQube** | 코드 품질 | 보안 규칙 포함 |

---

## 🤔 생각해볼 문제

### Q1: "의존성 라이브러리에 CVE가 있는데 패치가 없다. 어떻게 할까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**시나리오**:

```
상황:
- 사용 중인 라이브러리: log4j 2.14.1
- CVE-2021-44228 (CVSS 10.0, 가장 위험)
- 패치: 2.15.0 이상

하지만:
- log4j 2.15.0은 주요 API 변경
- 애플리케이션이 2.15.0과 호환 안 함
- 패치하면 빌드 실패
```

**해결 방법**:

1. **대체 라이브러리 사용**
```gradle
// 문제:
implementation("log4j:log4j:2.14.1")

// 해결: Logback 또는 SLF4J로 변경
implementation("ch.qos.logback:logback-classic:1.2.11")
```

2. **Dependency Exclusion (임시)**
```gradle
implementation("some-lib:1.0.0") {
    exclude(group = "log4j", module = "log4j")
}
```

3. **WAF/IDS로 방어**
```
애플리케이션 방어벽:
- ModSecurity WAF: Log4j 공격 패턴 차단
- 입력 검증: JNDI 문자열 금지
```

4. **제한된 권한 실행**
```
Docker:
FROM openjdk:11
USER appuser  # ← 루트 아님
RUN chmod 000 /etc/shadow  # ← 시스템 파일 수정 불가
```

5. **모니터링 강화**
```yaml
- 로그 모니터링: "JNDI" 패턴 감지
- 네트워크 모니터링: 비정상 DNS 조회
- 파일 시스템: 비정상 파일 생성
```

**우선순위**:
1. 패치 가능 → 즉시 패치
2. 대체 라이브러리 → 마이그레이션
3. 보안 패치 불가 → 방어 강화
4. 마지막 수단 → 라이브러리 제거

</details>

### Q2: "Trivy가 거짓 양성으로 취약점을 보고한다. 어떻게 확인할까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**거짓 양성 확인 프로세스**:

```
Trivy 보고:
- Image: ubuntu:22.04
- Package: openssl 3.0.1-1ubuntu1
- CVE-2022-0778: CVSS 5.9

실제로 영향받는가?
1. CVE 상세 확인
   https://nvd.nist.gov/vuln/detail/CVE-2022-0778
   → "Affects: OpenSSL 1.0.2 through 1.0.2zd"
   → Ubuntu 22.04는 OpenSSL 3.0.1
   → 영향받지 않음 (거짓 양성)

2. Trivy 최신 버전 확인
   trivy version
   → 구 버전의 NVD 데이터 사용했을 가능성

3. NVD 데이터 업데이트
   trivy image update
```

**거짓 양성 처리**:

```yaml
# .trivyignore 파일로 무시 목록 관리

# 거짓 양성 (Ubuntu 22.04에 영향 없음)
CVE-2022-0778

# 패치 예정 (다음 스프린트)
CVE-2023-1234

# 알려진 제한 (Open by Design)
CVE-2023-9999  exp:2024-12-31
```

```bash
# Trivy 실행 시 무시 파일 사용
trivy image \
  --ignore-file .trivyignore \
  ubuntu:22.04
```

**확실한 방법**:

```bash
# 1. 직접 패키지 버전 확인
docker run --rm ubuntu:22.04 \
  dpkg -l | grep openssl

# 출력:
# ii  openssl              3.0.1-1ubuntu1

# 2. CVE와 실제 설치 버전 비교
# CVE: 1.0.2 ~ 1.0.2zd 범위
# 설치: 3.0.1
# → 영향 없음 (안전)
```

</details>

### Q3: "git-secrets가 너무 많은 거짓 양성을 생성한다. 패턴을 어떻게 조정할까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**거짓 양성 감소 방법**:

```bash
# 문제: 모든 "password" 문자열에 반응
git secrets --add -a 'password'

# 결과:
# 거짓 양성:
# - password: "initial"  (테스트 코드)
# - password123  (로그인 폼 예제)
# - defaultPassword = "abc123"  (문서)
```

**패턴 개선**:

```bash
# 1. 더 구체적인 패턴으로 변경
# ❌ 너무 넓음:
git secrets --add -a 'password'

# ✅ 더 구체적:
git secrets --add -a 'password["\s]*[:=]["\s]*[a-zA-Z0-9!@#$%^&*()]{8,}'

# 2. 환경 변수 형식만 감지
git secrets --add -a 'export [A-Z_]*_PASSWORD=.*'

# 3. 특정 파일만 검사
git secrets --scan -- --cached *.java *.properties
```

**Known False Positive 예외 처리**:

```bash
# 1. 파일 전체 무시
echo ".env.example" >> .gitignore
git secrets --scan -- ':!.env.example'

# 2. 특정 라인 무시
# 파일 내 마커 추가
echo "password = 'test'  # git-secrets:ignore" >> test.properties

# 3. 전역 allowlist
git config --global secrets.allowed |  # 허용 패턴 목록
```

**권장 패턴 조합**:

```bash
git secrets --register-aws  # AWS 계정 ID, 키

# API 키 (최소 32자 이상)
git secrets --add -a '[a-zA-Z0-9_]{32,}'

# 개인 키
git secrets --add -a 'BEGIN .* PRIVATE KEY'
git secrets --add -a 'BEGIN RSA PRIVATE KEY'

# 특정 서비스 토큰 (더 정확함)
git secrets --add -a 'ghp_[a-zA-Z0-9]{36}'        # GitHub
git secrets --add -a 'sk_live_[a-zA-Z0-9]{24}'    # Stripe
git secrets --add -a 'xoxb-[0-9]{1,13}-[0-9]{1,13}-[a-zA-Z0-9]{24}'  # Slack
```

**CI에서의 처리**:

```yaml
- name: Check for secrets
  run: |
    # TruffleHog with high-confidence 검사만
    trufflehog git . \
      --only-verified \  # 검증된 토큰만 (Validator 실행)
      --json > /tmp/secrets.json
    
    # 검증된 토큰이 실제로 발견됨
    if [ -s /tmp/secrets.json ]; then
      echo "❌ Real secrets detected (verified by API)"
      cat /tmp/secrets.json
      exit 1
    fi
```

</details>

---

[⬅️ 이전](./04-performance-regression.md) | [홈으로 🏠](../README.md) | [다음: Chapter 7 — 배포 추적과 알림 ➡️](../monitoring-operations/01-deployment-tracking.md)
