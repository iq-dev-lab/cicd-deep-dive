# 아티팩트와 리포트 — Job 간 결과물 공유

## 🎯 핵심 질문

`upload-artifact`와 `download-artifact`는 내부적으로 어떻게 저장되고 공유될까? 테스트 결과를 GitHub PR에 자동으로 코멘트로 추가하려면? JUnit XML 리포트를 GitHub Actions에서 시각화하는 방법은? 아티팩트는 8GB 제한이 있는데, 대용량 빌드 결과는 어떻게 관리할까?

## 🔍 왜 이 개념이 실무에서 중요한가

- **테스트 추적성**: 실패한 테스트의 로그와 스크린샷을 PR에서 직접 확인 가능
- **빌드 결과 공유**: 빌드 성과물을 Job 간에 전달하는 표준 방식
- **저장소 비용**: 아티팩트는 저장소이고 비용이 발생하므로 최적화 필수
- **감사 추적**: 어떤 버전이 배포되었는지, 테스트가 통과했는지 기록

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 예 1: 모든 파일을 아티팩트로 업로드
name: Inefficient Artifact
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test > test-output.log 2>&1

      - name: Upload everything
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: .  # 전체 디렉토리!
          # node_modules, .git 등 불필요한 파일 포함
          # 아티팩트 크기: 500MB (실제 필요한 것: 1MB)

# ❌ 잘못된 예 2: 아티팩트 보존 정책 무시
name: Expensive Artifacts
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

      - name: Upload build output
        uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: dist/
          # retention-days 미설정 → 기본 90일 보존
          # 매일 빌드하면 90개 아티팩트 × 200MB = 18GB (비용 발생!)

# ❌ 잘못된 예 3: 아티팩트 없이 Job 간 파일 전달 시도
name: No Artifact Transfer
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "version=1.0.0" > version.txt

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - run: cat version.txt  # ❌ 실패: 다른 Runner에서는 파일 없음

# ❌ 잘못된 예 4: 테스트 리포트를 자동 코멘트 하지 않음
name: No Test Report Comments
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: npm test -- --junit > test-report.xml

      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: junit-reports
          path: test-report.xml
        # 아티팩트는 업로드하지만 
        # PR에 자동 코멘트 안 함 → 개발자가 확인 안 함
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 예 1: 필요한 아티팩트만 선택적 업로드
name: Optimized Artifact
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test -- --reporter=junit --outputFile=test-results.xml

      - name: Upload test results only
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.node-version }}
          path: test-results.xml  # 1KB만
          retention-days: 7  # 1주일만 보존

# ✅ 올바른 예 2: 보존 정책으로 비용 최적화
name: Cost-Optimized Artifacts
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm run build

      - name: Upload build output
        uses: actions/upload-artifact@v3
        with:
          name: build-dist-${{ github.sha }}
          path: dist/
          retention-days: 1  # main 브랜치라도 1일만 보존
          # 또는 PR 브랜치는 더 짧게:
          # retention-days: ${{ github.ref == 'refs/heads/main' && '3' || '1' }}

      - name: Upload build metadata
        run: |
          echo "built-at=$(date -u)" > build.json
          echo "commit=$(git rev-parse --short HEAD)" >> build.json

# ✅ 올바른 예 3: 아티팩트로 Job 간 파일 전달
name: Artifact Transfer Pattern
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build version
        run: echo "version=1.0.0" > version.txt

      - name: Upload version
        uses: actions/upload-artifact@v3
        with:
          name: build-metadata
          path: version.txt
          retention-days: 1

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - name: Download version
        uses: actions/download-artifact@v3
        with:
          name: build-metadata

      - name: Deploy
        run: |
          VERSION=$(cat version.txt)
          echo "Deploying version: $VERSION"

# ✅ 올바른 예 4: 테스트 리포트를 PR 코멘트로 자동 추가
name: Test Report in PR
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test -- --reporter=json > test-results.json

      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test-results.json

      - name: Publish test results to PR
        if: always()  # 성공/실패 무관
        uses: EnricoMi/publish-unit-test-result-action@v2
        with:
          files: test-results.json
          check_name: Test Results

      - name: Comment test summary
        if: github.event_name == 'pull_request' && always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('test-results.json', 'utf8'));
            
            const summary = `
            ## Test Results
            - Passed: ${results.stats.passes}
            - Failed: ${results.stats.failures}
            - Duration: ${results.stats.duration}ms
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: summary
            });
```

## 🔬 내부 동작 원리

### 1. 아티팩트 저장 메커니즘

**저장 위치:**
```
GitHub의 Blob Storage
  ├─ Repository ID
  ├─ Run ID
  ├─ Artifact 이름
  └─ 파일 데이터 (gzip 압축)

접근 경로:
  GitHub UI: Actions tab → Artifacts section
  API: GET /repos/{owner}/{repo}/actions/artifacts
  CLI: gh run download {run-id}
```

**저장 프로세스:**
```
1. Runner에서 파일 수집
   path: dist/
   → dist 디렉토리의 모든 파일 목록

2. 파일 압축 및 업로드
   - 개별 파일 압축
   - GitHub 서버로 업로드
   - 메타데이터 저장 (크기, 해시, 생성 시간)

3. 보존 정책 적용
   - retention-days: 7 → 7일 후 자동 삭제
   - 만료 시 스케줄러가 자동 정리

4. 저장소 할당량 계산
   - GitHub Free: 500MB (月) 포함
   - GitHub Pro/Team: 2GB 월 포함
   - 초과 시: $0.25 / GB 추가 비용
```

**아티팩트 크기 제한:**
```
단일 파일: 최대 2GB
전체 workflow run: 최대 10GB
유효성: 생성 후 자동 정리 전까지만
```

### 2. 아티팩트 다운로드와 캐시의 차이

| 항목 | Artifact | Cache |
|------|----------|-------|
| 용도 | Job 간 결과물 공유 | 의존성 빠른 복구 |
| 저장소 | GitHub 서버 | Runner의 로컬 저장소 |
| 접근 | `download-artifact` | 자동 복구 |
| 만료 | 7~90일 (설정 가능) | 7일 (고정) |
| 비용 | 초과분 유료 | 무료 |
| 속도 | 느림 (네트워크) | 빠름 (로컬) |

### 3. 테스트 리포트 시각화

**JUnit XML 형식:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="AuthService" tests="3" failures="1" skipped="0">
    <testcase name="login_success" classname="AuthService"/>
    <testcase name="login_invalid_password" classname="AuthService">
      <failure message="Expected password to be hashed">
        AssertionError: Expected hashed password...
      </failure>
    </testcase>
    <testcase name="logout_clears_session" classname="AuthService"/>
  </testsuite>
</testsuites>
```

**GitHub에서 처리:**
```
1. JUnit XML 파일 감지
2. 테스트 수, 실패율 파싱
3. Check Suite 생성 (PR의 Checks 탭에 표시)
4. 실패한 테스트의 상세 정보 표시
5. PR의 코멘트로 자동 추가 가능 (actions/github-script 사용)
```

### 4. Artifact vs Docker 이미지 레지스트리

**Artifact 사용:**
```
장점:
  - GitHub 내장, 별도 레지스트리 불필요
  - 보존 기간 세밀 제어
  - 빠른 설정

단점:
  - 이미지 저장 용도로 비효율적
  - 크기 제한 (10GB/run)
  - 다른 Runner에서 재사용 어려움
```

**Docker 레지스트리 (GHCR, ECR, Docker Hub):**
```
장점:
  - 이미지 레이어 캐싱 (부분 재사용)
  - 무제한 저장 (비용은 발생)
  - CI/CD 표준

단점:
  - 별도 인증 관리
  - 이미지 빌드 시간 오버헤드
```

**선택 기준:**
```
Artifact 사용:
  - 단일 바이너리 (< 100MB)
  - 테스트 결과 및 로그
  - 빌드 메타데이터
  - 보존 기간 짧음

Docker 이미지 사용:
  - 대용량 빌드 결과 (> 500MB)
  - 여러 환경(로컬, 클라우드)에서 재사용
  - 의존성 통째로 캡슐화
  - 배포 대상이 Docker 런타임
```

### 5. PR 댓글 자동 추가

**방법 1: actions/github-script**
```javascript
// 스크립트에서 context.issue.number 사용
github.rest.issues.createComment({
  issue_number: context.issue.number,
  owner: context.repo.owner,
  repo: context.repo.repo,
  body: '## Test Results...'
});
```

**방법 2: Dedicated Action (예: publish-unit-test-result-action)**
```yaml
- uses: EnricoMi/publish-unit-test-result-action@v2
  with:
    files: 'test-results.xml'
    # 자동으로:
    # 1. XML 파싱
    # 2. Check Run 생성
    # 3. PR 코멘트 추가
    # 4. 실패 테스트 상세 표시
```

**액세스 제어:**
```
PR 코멘트 작성:
  - 권한 필요: contents: read, pull-requests: write
  - Workflow 권한은 기본적으로 제한됨
  - secrets.GITHUB_TOKEN 사용 (자동 생성)
```

## 💻 실전 실험 (GitHub Actions YAML 재현 가능한 예시)

### 실험 1: 기본 아티팩트 업로드/다운로드

```yaml
name: Artifact Transfer
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build application
        run: |
          mkdir -p dist
          echo "console.log('App v1.0.0')" > dist/app.js
          echo "{ \"version\": \"1.0.0\" }" > dist/package.json

      - name: Upload build artifact
        uses: actions/upload-artifact@v3
        with:
          name: dist-${{ github.sha }}
          path: dist/
          retention-days: 7

  test:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v3
        with:
          name: dist-${{ github.sha }}
          path: dist/

      - name: Verify artifact
        run: |
          echo "Contents of dist:"
          ls -la dist/
          cat dist/package.json

  deploy:
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - name: Download for deployment
        uses: actions/download-artifact@v3
        with:
          name: dist-${{ github.sha }}

      - name: Deploy
        run: echo "Deploying from dist/"
```

### 실험 2: 테스트 리포트 생성 및 댓글

```yaml
name: Test Report
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm install

      - name: Run tests with coverage
        run: |
          npm test -- \
            --coverage \
            --watchAll=false \
            --testResultsProcessor=jest-junit
        continue-on-error: true  # 실패해도 리포트 생성

      - name: Generate test report
        run: |
          # 테스트 결과를 JSON으로 변환
          npm test -- --json > test-results.json

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results-${{ github.run_id }}
          path: test-results.json
          retention-days: 30

      - name: Publish to PR
        if: github.event_name == 'pull_request' && always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const testResults = JSON.parse(
              fs.readFileSync('test-results.json', 'utf8')
            );

            const stats = testResults.testResults.reduce((acc, suite) => {
              acc.passed += suite.numPassingTests;
              acc.failed += suite.numFailingTests;
              acc.skipped += suite.numPendingTests;
              return acc;
            }, { passed: 0, failed: 0, skipped: 0 });

            const comment = `
## Test Results Summary
- ✅ Passed: ${stats.passed}
- ❌ Failed: ${stats.failed}
- ⏭️ Skipped: ${stats.skipped}

Total: ${stats.passed + stats.failed + stats.skipped} tests
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

### 실험 3: 선택적 아티팩트 업로드 (조건부)

```yaml
name: Conditional Artifacts
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          npm run build
          npm run test

      - name: Upload only on failure
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: debug-logs-${{ github.run_id }}
          path: |
            build/logs/
            test-results/
          retention-days: 7

      - name: Upload on success (smaller retention)
        if: success()
        uses: actions/upload-artifact@v3
        with:
          name: build-dist
          path: dist/
          retention-days: 1
```

### 실험 4: 대용량 바이너리 관리

```yaml
name: Large Binary Management
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build large binary
        run: |
          # 500MB 바이너리 생성 (시뮬레이션)
          dd if=/dev/zero of=app-binary bs=1M count=500

      - name: Docker image vs Artifact decision
        run: |
          SIZE=$(du -sh app-binary | cut -f1)
          echo "Binary size: $SIZE"
          
          if [[ $(stat -f%z app-binary) -gt 104857600 ]]; then
            echo "Too large for artifact (>100MB)"
            echo "Pushing to container registry instead"
          fi

      - name: Push to GitHub Container Registry
        if: github.ref == 'refs/heads/main'
        run: |
          # 실제 구현:
          # docker build -t ghcr.io/owner/repo:latest .
          # docker push ghcr.io/owner/repo:latest
          echo "Would push to GHCR"

      - name: Minimal artifact (metadata only)
        uses: actions/upload-artifact@v3
        with:
          name: build-metadata
          path: |
            build.log
            VERSION
            COMMIT_SHA
          retention-days: 7
```

### 실험 5: 아티팩트 스토리지 모니터링

```yaml
name: Artifact Storage Monitor
on: [push, workflow_dispatch]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      actions: write
    steps:
      - name: List artifacts
        uses: actions/github-script@v7
        with:
          script: |
            const artifacts = await github.rest.actions.listArtifactsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
            });

            console.log(`Total artifacts: ${artifacts.data.artifacts.length}`);
            
            let totalSize = 0;
            artifacts.data.artifacts.forEach(artifact => {
              totalSize += artifact.size_in_bytes;
              console.log(`- ${artifact.name}: ${(artifact.size_in_bytes / 1024 / 1024).toFixed(2)}MB`);
            });

            console.log(`Total size: ${(totalSize / 1024 / 1024 / 1024).toFixed(2)}GB`);

      - name: Delete old artifacts
        uses: actions/github-script@v7
        with:
          script: |
            const artifacts = await github.rest.actions.listArtifactsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
            });

            // 7일 이상 된 아티팩트 삭제
            const now = new Date();
            const sevenDaysAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);

            for (const artifact of artifacts.data.artifacts) {
              const createdDate = new Date(artifact.created_at);
              if (createdDate < sevenDaysAgo) {
                await github.rest.actions.deleteArtifact({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  artifact_id: artifact.id,
                });
                console.log(`Deleted: ${artifact.name}`);
              }
            }
```

## 📊 성능/비용 비교

| 방식 | 저장 비용 | 조회 속도 | 보존 기간 | 추천 용도 |
|------|---------|---------|---------|---------|
| **Artifact (1일)** | ~$0.01 | 빠름 | 1일 | 임시 빌드 결과 |
| **Artifact (7일)** | ~$0.10 | 빠름 | 7일 | 테스트 로그 |
| **Artifact (90일)** | ~$2 | 빠름 | 90일 | 릴리스 바이너리 |
| **Docker 레지스트리** | ~$0.20/GB/월 | 느림 | 무제한 | 대용량 이미지 |
| **S3 저장소** | ~$0.023/GB | 중간 | 무제한 | 아카이브 |

**계산 예시 (500MB 아티팩트):**
- 1일 보존: $0.004 / 일 = $0.13 / 월 (1회/일)
- 7일 보존: $0.04 / 주 = $0.17 / 월 (1회/일)
- Docker GHCR: $0.20 / 월 (무제한)

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 |
|------|------|------|
| **Artifact** | 간단, 빠름, GitHub 통합 | 저장소 비용, 크기 제한 |
| **Docker GHCR** | 레이어 캐싱, 표준 | 관리 복잡, 빌드 시간 |
| **S3/GCS** | 무제한 저장, 저렴 | 별도 구성, 느린 조회 |
| **짧은 보존 기간** | 비용 절감 | 오래된 빌드 조회 불가 |
| **긴 보존 기간** | 감사 추적 용이 | 비용 증가 |

## 📌 핵심 정리

1. **Artifact 용도**: Job 간 결과물 공유, 테스트 로그, 빌드 메타데이터
2. **크기와 비용**: 1일 보존 권장 (비용 최소), 필요시 7일 이상
3. **테스트 리포트**: PR 코멘트로 자동 추가 (actions/github-script 또는 dedicated actions)
4. **대용량 결과**: 500MB 이상이면 Docker 레지스트리 사용 권장
5. **저장소 관리**: 불필요한 아티팩트 정기적 정리 (GitHub Script로 자동화)
6. **감사 추적**: 중요한 빌드는 최소 7일, 릴리스는 90일 보존

## 🤔 생각해볼 문제

### 문제 1: 이 아티팩트 설정의 문제는?

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

      - uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: dist/
          retention-days: 90

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: build-output

      - run: npm run deploy
```

프로젝트가 활발하면 (매일 빌드) 3개월 후 비용은?

<details>
<summary>해설</summary>

**비용 계산:**

- 매일 빌드: 1개 / 일
- dist/ 크기: 50MB (보통)
- 90일 보존: 90개 동시 저장

**저장소 사용량:**
```
90개 × 50MB = 4.5GB
GitHub Free: 500MB 月
초과: 4GB = 4 × $0.25 = $1/월
```

**월별 누적:**
- 월 1: $0 (500MB 내)
- 월 2: $0.12 (90일 × 50MB)
- 월 3+: $0.12 - $0.25/월 (계속)

**개선안:**
```yaml
retention-days: 7  # 1주일만
# 70MB 저장 → 무료 범위

# 또는 main 브랜치만 긴 보존
retention-days: |
  ${{ github.ref == 'refs/heads/main' && 30 || 1 }}

# 결과: 대부분 1일, main만 30일
# 예상 비용: ~$0.01/월
```
</details>

### 문제 2: 테스트가 실패했을 때 PR 댓글이 추가되지 않는다. 왜?

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - run: npm test

      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '## Tests passed!'
            });
```

<details>
<summary>해설</summary>

**문제 1: npm test 실패 시 실행 중단**
```yaml
- run: npm test
  # 실패 → 다음 step 미실행 → 댓글 안 생김
```

**해결책 1: `continue-on-error`**
```yaml
- run: npm test
  continue-on-error: true
  # 실패해도 계속 진행
```

**문제 2: context.issue.number가 없을 수 있음**
```yaml
# PR이 아닌 push 이벤트면:
# context.issue.number = undefined
# → 댓글 생성 실패
```

**해결책 2: 조건 추가**
```yaml
- name: Comment on PR
  if: github.event_name == 'pull_request' && always()
  uses: actions/github-script@v7
  with:
    script: |
      if (!context.issue || !context.issue.number) {
        console.log('Not a PR');
        return;
      }
      
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: '## Tests'
      });
```

**문제 3: 권한 없음**
```yaml
# Workflow 권한 확인:
# pull-requests: write 필요

jobs:
  test:
    permissions:
      pull-requests: write  # 추가
    steps:
      - uses: actions/github-script@v7
        # 이제 작동
```

**올바른 버전:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - run: npm test
        continue-on-error: true

      - uses: actions/github-script@v7
        if: github.event_name == 'pull_request' && always()
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'Test completed'
            });
```
</details>

### 문제 3: 여러 Job에서 같은 이름으로 아티팩트 업로드하면?

```yaml
jobs:
  test-unit:
    strategy:
      matrix:
        node-version: [16, 18]
    steps:
      - run: npm test > test-report.txt

      - uses: actions/upload-artifact@v3
        with:
          name: test-results  # 같은 이름!
          path: test-report.txt

  test-e2e:
    steps:
      - run: npm run test:e2e > e2e-report.txt

      - uses: actions/upload-artifact@v3
        with:
          name: test-results  # 같은 이름!
          path: e2e-report.txt
```

<details>
<summary>해설</summary>

**결과: 아티팩트 병합됨**

GitHub Actions은 같은 이름의 아티팩트를 병합합니다:

```
test-results/
├─ node-16/
│  └─ test-report.txt
├─ node-18/
│  └─ test-report.txt
└─ e2e-report.txt
```

**문제점:**
1. 파일 경로가 길어짐
2. 다운로드 시 전체 병합본 받음 (불필요한 파일도)
3. Job 간 충돌 가능성

**올바른 버전:**
```yaml
jobs:
  test-unit:
    strategy:
      matrix:
        node-version: [16, 18]
    steps:
      - run: npm test > test-report.txt

      - uses: actions/upload-artifact@v3
        with:
          name: test-results-unit-node${{ matrix.node-version }}
          path: test-report.txt

  test-e2e:
    steps:
      - run: npm run test:e2e > e2e-report.txt

      - uses: actions/upload-artifact@v3
        with:
          name: test-results-e2e
          path: e2e-report.txt

  aggregate:
    needs: [test-unit, test-e2e]
    steps:
      - uses: actions/download-artifact@v3
        # 모든 test-results-* 아티팩트 다운로드

      - run: |
          echo "Combining test results..."
          ls -la test-results-*
```
</details>

---

<div align="center">
**[⬅️ 이전: 캐시 전략](./05-cache-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — Dockerfile 레이어 캐시 원리 ➡️](../docker-build-optimization/01-layer-cache-principles.md)**
</div>
