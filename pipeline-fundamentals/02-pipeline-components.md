# Pipeline 구성 요소 — Trigger/Job/Step/Artifact

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Trigger, Job, Step, Artifact는 각각 어떤 역할을 하고 어떻게 연결되는가?
- GitHub-hosted Runner와 Self-hosted Runner의 실질적 차이는 무엇인가?
- Job은 왜 격리된 환경에서 실행되어야 하는가?
- 한 Job의 결과물을 다른 Job에서 사용하려면 어떻게 해야 하는가?
- Runner 선택이 파이프라인 성능과 보안에 어떤 영향을 주는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

GitHub Actions YAML을 처음 작성할 때 `on:`, `jobs:`, `steps:`를 복붙하면서도 각 블록이 어떤 의미인지 정확히 모르는 경우가 많다. 이 모호함은 파이프라인이 실패했을 때 원인을 찾지 못하게 만들고, 더 나은 구조로 개선하지 못하게 막는다.

각 구성 요소의 역할과 경계를 정확히 알면, 파이프라인을 처음부터 읽을 수 있고 실패 지점을 즉시 특정할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# 흔한 잘못된 패턴 1: 모든 것을 하나의 Job에 넣기
jobs:
  everything:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./gradlew test          # 테스트
      - run: ./gradlew build         # 빌드
      - run: docker build -t app .   # 이미지 빌드
      - run: docker push app         # 이미지 푸시
      - run: kubectl apply -f k8s/   # 배포

# 문제:
#   1. 테스트 실패해도 어디서 실패했는지 한눈에 안 보임
#   2. 테스트만 다시 실행하고 싶어도 전부 다시 실행
#   3. 빌드와 배포를 병렬화할 수 없음
#   4. 배포 승인 단계를 중간에 넣기 어려움
```

```yaml
# 흔한 잘못된 패턴 2: Job 간 파일 공유를 환경으로 해결하려는 시도
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew build
      # "빌드된 파일이 다음 Job에서 필요한데..."
      # → 아무 처리 없이 deploy Job에서 파일을 찾으려 함 → 파일 없음 에러

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ls build/libs/    # ← 비어 있음! 다른 Runner에서 실행되기 때문
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# 올바른 구조: 역할별로 Job 분리, Artifact로 결과물 공유
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  # Job 1: 테스트 (빠른 피드백)
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
      - run: ./gradlew test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-reports
          path: build/reports/tests/

  # Job 2: 빌드 (테스트 통과 후)
  build:
    needs: test              # test Job 완료 후 실행
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
      - run: ./gradlew bootJar
      - uses: actions/upload-artifact@v4
        with:
          name: app-jar           # ← Artifact로 저장
          path: build/libs/*.jar
          retention-days: 7

  # Job 3: 배포 (빌드 완료 후)
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production      # ← 배포 승인 게이트
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-jar           # ← 이전 Job의 Artifact 사용
      - run: ls -la *.jar
      - run: echo "배포 진행..."
```

---

## 🔬 내부 동작 원리

### 구성 요소 계층 구조

```
Workflow (.github/workflows/ci.yml)
  │
  ├── Trigger (on:)
  │     언제 파이프라인을 시작할지 결정
  │
  ├── Job (jobs.test:)
  │     하나의 Runner에서 실행되는 작업 단위
  │     기본적으로 다른 Job과 병렬 실행
  │     needs: 로 순서 제어 가능
  │
  │     ├── Step (steps[0])
  │     │     Job 안에서 순차적으로 실행
  │     │     이전 Step의 파일에 접근 가능 (같은 Runner)
  │     │
  │     ├── Step (steps[1])
  │     └── Step (steps[2])
  │
  └── Artifact
        Job 간 파일 공유 수단
        GitHub 서버에 임시 저장 후 다른 Job에서 다운로드
```

### Trigger — 파이프라인의 시작점

Trigger는 GitHub 이벤트와 파이프라인을 연결한다. 코드가 push되거나, PR이 열리거나, 스케줄 시간이 되거나, 수동으로 버튼을 누르는 것 모두 Trigger가 될 수 있다.

```yaml
on:
  # push 이벤트
  push:
    branches: [main, 'release/**']
    paths-ignore:
      - 'docs/**'        # 문서만 변경 시 파이프라인 스킵
      - '*.md'

  # PR 이벤트
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  # 스케줄 (cron 형식, UTC 기준)
  schedule:
    - cron: '0 2 * * *'  # 매일 새벽 2시 (UTC) — 야간 통합 테스트

  # 수동 실행
  workflow_dispatch:
    inputs:
      environment:
        description: '배포 환경'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
```

내부적으로: GitHub은 레포지토리 이벤트가 발생하면 Webhook을 통해 GitHub Actions 서비스에 알린다. 해당 이벤트가 Workflow의 `on:` 조건과 일치하면 새로운 Workflow Run이 생성된다.

### Job — 격리의 단위

각 Job은 **새로운 Runner 인스턴스에서 실행된다**. 이는 두 가지 의미를 가진다.

첫째, Job 사이에서 파일 시스템이 공유되지 않는다. Job A에서 생성한 파일은 Job B에서 접근할 수 없다. 파일을 공유하려면 반드시 Artifact를 통해야 한다.

둘째, Job은 기본적으로 병렬로 실행된다. `needs:` 없이 여러 Job을 정의하면 동시에 실행된다.

```yaml
jobs:
  lint:                    # 병렬 실행 1
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew checkstyleMain

  test-unit:               # 병렬 실행 2
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew test

  test-integration:        # 병렬 실행 3
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew integrationTest

  build:
    needs: [lint, test-unit, test-integration]  # 위 3개 모두 통과 후 실행
    runs-on: ubuntu-latest
    steps:
      - run: ./gradlew bootJar
```

이 구조에서 `lint`, `test-unit`, `test-integration`은 동시에 실행된다. 전체 파이프라인 시간은 세 Job 중 가장 오래 걸리는 것에 의해 결정된다.

### Step — Job 내의 순차 실행

Step은 같은 Job 안에서 순차적으로 실행된다. 이전 Step에서 생성한 파일이나 환경변수에 접근할 수 있다.

```yaml
steps:
  - name: 1단계 파일 생성
    run: echo "hello" > /tmp/message.txt

  - name: 2단계 파일 읽기 (같은 Runner이므로 가능)
    run: cat /tmp/message.txt    # "hello" 출력

  - name: 3단계 환경변수 설정
    run: echo "VERSION=1.2.3" >> $GITHUB_ENV

  - name: 4단계 환경변수 사용
    run: echo "버전: $VERSION"   # "버전: 1.2.3" 출력
```

Step 간 데이터 전달 방법:
- **파일**: 같은 디렉토리에 파일 생성 후 다음 Step에서 읽기
- **환경변수**: `$GITHUB_ENV` 파일에 추가
- **출력값(Output)**: `$GITHUB_OUTPUT` 파일에 추가 후 `${{ steps.step-id.outputs.변수명 }}`으로 참조

```yaml
steps:
  - name: 버전 계산
    id: version-step          # ← id 지정 필수
    run: |
      VERSION=$(git describe --tags --abbrev=0 2>/dev/null || echo "0.0.1")
      echo "tag=$VERSION" >> $GITHUB_OUTPUT    # ← Output 설정

  - name: 버전 사용
    run: echo "배포 버전: ${{ steps.version-step.outputs.tag }}"
```

### Artifact — Job 간 파일 다리

Artifact는 Job 간 파일을 공유하는 유일한 공식 수단이다. 업로드된 파일은 GitHub 서버에 저장되고, 다른 Job에서 다운로드할 수 있다.

```yaml
# 업로드 (build Job)
- uses: actions/upload-artifact@v4
  with:
    name: app-bundle          # Artifact 이름 (Job 내에서 고유해야 함)
    path: |
      build/libs/*.jar
      config/
    retention-days: 5         # 5일 후 자동 삭제

# 다운로드 (deploy Job)
- uses: actions/download-artifact@v4
  with:
    name: app-bundle          # 동일한 이름
    path: ./deploy/           # 다운로드 위치 (생략 시 현재 디렉토리)
```

---

### GitHub-hosted Runner vs Self-hosted Runner

**GitHub-hosted Runner**

GitHub가 관리하는 가상 머신. push할 때마다 새로운 깨끗한 VM이 할당된다.

```yaml
runs-on: ubuntu-latest       # Ubuntu (무료)
runs-on: ubuntu-22.04        # 특정 버전 고정
runs-on: windows-latest      # Windows
runs-on: macos-latest        # macOS (비싸다)
```

특징:
- 즉시 사용 가능, 관리 불필요
- Job 종료 시 VM 폐기 → 완전한 격리
- `ubuntu-latest`는 2코어 CPU, 7GB RAM, 14GB SSD
- public 레포는 무료, private 레포는 월 2000분 무료 (초과 시 과금)

**Self-hosted Runner**

직접 관리하는 서버에 Runner 에이전트를 설치한다.

```yaml
runs-on: self-hosted          # 레이블로 특정 Runner 선택
runs-on: [self-hosted, linux, x64, high-memory]
```

설치:
```bash
# GitHub 레포 → Settings → Actions → Runners → New self-hosted runner
# 제공되는 스크립트로 설치

./config.sh --url https://github.com/org/repo \
            --token TOKEN

./run.sh    # 또는 서비스로 등록: sudo ./svc.sh install
```

Self-hosted Runner를 선택해야 하는 경우:
- 빌드가 오래 걸려 GitHub-hosted Runner 비용이 과도한 경우
- 특수 하드웨어(GPU, ARM) 또는 대용량 메모리가 필요한 경우
- 내부 네트워크(사내 DB, 사내 레지스트리)에 접근이 필요한 경우
- Docker 레이어 캐시, Gradle 캐시를 Runner에 직접 보관해 빌드를 가속하는 경우

보안 주의사항: 외부 기여자의 PR에서 Self-hosted Runner를 사용하면 악의적인 코드가 내부 인프라에 접근할 수 있다. public 레포에서는 GitHub-hosted Runner를 사용하거나 엄격한 권한 제어가 필요하다.

---

## 💻 실전 실험

### 실험 1: Job 격리 확인

```yaml
name: Job 격리 실험

on: workflow_dispatch

jobs:
  job-a:
    runs-on: ubuntu-latest
    steps:
      - run: echo "job-a의 파일" > /tmp/from-a.txt
      - run: cat /tmp/from-a.txt    # 성공

  job-b:
    needs: job-a
    runs-on: ubuntu-latest
    steps:
      - run: cat /tmp/from-a.txt   # ← 실패! 다른 Runner
      # Error: cat: /tmp/from-a.txt: No such file or directory
```

이 실험을 직접 실행하면 Job 간 파일 시스템이 공유되지 않음을 확인할 수 있다.

### 실험 2: Artifact로 Job 간 파일 전달

```yaml
name: Artifact 전달 실험

on: workflow_dispatch

jobs:
  producer:
    runs-on: ubuntu-latest
    steps:
      - run: echo "job-a의 파일" > result.txt
      - uses: actions/upload-artifact@v4
        with:
          name: my-result
          path: result.txt

  consumer:
    needs: producer
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: my-result
      - run: cat result.txt    # ← 성공! Artifact를 통해 전달됨
```

### 실험 3: Job 출력값으로 이미지 태그 전달

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}   # Job 출력값 선언
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: |
          TAG="v1.0.0-$(git rev-parse --short HEAD)"
          echo "version=$TAG" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "배포할 이미지 태그: ${{ needs.build.outputs.image-tag }}"
          # docker pull myapp:${{ needs.build.outputs.image-tag }}
```

---

## 📊 성능/비용 비교

### Job 구조에 따른 파이프라인 시간 비교

```
단일 Job (순차 실행):
  테스트(5분) → 빌드(3분) → 보안스캔(4분) → 배포(2분) = 14분

분리된 Job (병렬 실행):
  ┌─ 테스트(5분) ──────────┐
  ├─ 보안스캔(4분) ────────┤→ 빌드(3분) → 배포(2분) = 10분
  └─ 정적분석(2분) ────────┘

절약: 4분 (28% 단축)
Runner 비용: GitHub-hosted ubuntu-latest 기준 $0.008/분
  단일 Job: 14분 × $0.008 = $0.112/회
  병렬 Job: (5+3+2+2)분 = 12분 분량 과금 = $0.096/회 (Runner 수만큼 과금)
  → 시간은 줄지만 비용은 비슷하거나 약간 증가할 수 있음
```

| Runner 유형 | 비용 | 빌드 캐시 | 내부망 접근 | 관리 부담 |
|------------|------|---------|-----------|---------|
| GitHub-hosted | 분당 과금 | 매 Job 초기화 | 불가 | 없음 |
| Self-hosted | 서버 유지비 | 디스크에 영구 보관 | 가능 | 높음 |

---

## ⚖️ 트레이드오프

**Job을 너무 잘게 쪼개면**

각 Job은 Runner 프로비저닝, 코드 체크아웃, 의존성 설치 시간이 필요하다. Job을 10개로 쪼개면 각각 30초~1분의 준비 시간이 추가된다. 논리적으로 연결된 작업은 같은 Job의 Step으로 유지하는 것이 낫다.

**Artifact의 크기 제한**

Artifact는 단일 파일 기준 2GB, Workflow 전체 기준 500MB~10GB 제한이 있다 (플랜에 따라 다름). 대용량 파일은 Docker 이미지로 빌드해 레지스트리에 저장하고, Job 간에는 이미지 태그만 전달하는 것이 더 효율적이다.

---

## 📌 핵심 정리

- **Trigger**: GitHub 이벤트를 파이프라인 실행과 연결하는 시작점
- **Job**: 독립된 Runner에서 실행되는 작업 단위, 기본 병렬 실행, `needs:`로 순서 제어
- **Step**: 같은 Job 안에서 순차 실행, 파일 시스템과 환경변수 공유
- **Artifact**: Job 간 유일한 파일 공유 수단, GitHub 서버에 임시 저장
- Job은 항상 다른 Runner에서 실행되므로 **파일 시스템이 공유되지 않음**
- Self-hosted Runner는 성능/비용 최적화와 내부망 접근에 유용하지만 보안 관리가 필요

---

## 🤔 생각해볼 문제

**Q1.** 하나의 Workflow에서 테스트 Job과 배포 Job을 병렬로 실행하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

테스트가 실패해도 배포가 동시에 진행된다. 테스트 실패한 코드가 프로덕션에 배포되는 심각한 문제가 발생한다. `needs: [test]`로 명시적 의존성을 설정해야 테스트 통과 후에만 배포 Job이 실행된다.

</details>

**Q2.** `actions/cache`와 `actions/upload-artifact`의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

`actions/cache`는 의존성 등 재사용 가능한 데이터를 Workflow 실행 간(여러 번의 push 사이)에 보관하기 위한 것이다. 같은 key가 있으면 캐시를 사용하고 없으면 새로 생성한다.

`actions/upload-artifact`는 같은 Workflow Run 내에서 Job 간 결과물을 전달하기 위한 것이다. 빌드된 JAR 파일을 빌드 Job에서 배포 Job으로 전달하는 것이 대표적인 사용 사례다. Workflow Run이 끝나면 설정된 retention 기간 후 삭제된다.

</details>

**Q3.** Self-hosted Runner에서 Job이 실행된 후 파일이 남아 있으면 다음 Job에 영향을 줄 수 있는가?

<details>
<summary>해설 보기</summary>

그렇다. GitHub-hosted Runner는 Job이 끝나면 VM 자체가 폐기되므로 완전한 격리가 보장된다. 반면 Self-hosted Runner는 같은 서버가 재사용되므로 이전 Job에서 남긴 파일, 환경변수, 프로세스가 다음 Job에 영향을 줄 수 있다.

이를 방지하려면 각 Job 시작 시 작업 디렉토리를 초기화하거나, Docker 컨테이너 안에서 Job을 실행하는 방식을 사용한다. GitHub Actions의 Runner는 `--ephemeral` 플래그로 Job 후 자동 종료하고 새 Runner 인스턴스를 사용하도록 설정할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: CI/CD 철학 — 수동 배포가 실패하는 이유](./01-cicd-philosophy.md)** | **[홈으로 🏠](../README.md)** | **[다음: GitHub Actions 내부 동작 ➡️](./03-github-actions-internals.md)**

</div>
