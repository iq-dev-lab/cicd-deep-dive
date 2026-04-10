# Actions 내부 동작 — Docker Action vs JavaScript Action

## 🎯 핵심 질문

`uses: actions/checkout@v4`는 정확히 어떻게 실행될까? Docker Action과 JavaScript Action의 차이점은 무엇이고, 어떤 경우에 어떤 타입을 선택해야 할까? Composite Action은 언제 필요한가? 그리고 Reusable Workflow와의 차이는?

## 🔍 왜 이 개념이 실무에서 중요한가

- **Action 선택**: 같은 기능을 여러 Action으로 구현할 수 있는데, 각각의 성능과 보안이 다름
- **Custom Action 개발**: 팀의 배포 프로세스를 자동화하려면 Custom Action을 만들어야 하는데, 올바른 타입 선택이 중요
- **보안과 격리**: Docker Action은 완전히 격리되지만 오버헤드가 크고, JavaScript Action은 빠르지만 Host Runner에서 직접 실행됨
- **디버깅**: Action 내부에서 뭔가 실패할 때 로그 해석 능력이 필수

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 예 1: 느린 Docker Action으로 간단한 작업
name: Inefficient Action Usage
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # Docker로 실행: 약 20초 소요

      - uses: custom/slow-docker-action@v1
        # 간단한 파일 조작을 Docker로: 
        # 이미지 풀(2-5초) + 컨테이너 시작(3-10초) + 실제 작업(0.1초)
        with:
          file: package.json

      - run: echo "Done"
        # JavaScript Action으로 구현했으면 0.5초

# ❌ 잘못된 예 2: Action input/output을 무시
name: Missing Action Metadata
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # 이 Action이 어떤 outputs를 제공하는지 모름

      - name: Try to use output
        run: echo ${{ steps.checkout.outputs.repo_path }}
        # outputs가 정의되지 않았으므로 실패
        # (사실 checkout은 outputs를 제공하지 않음)

# ❌ 잘못된 예 3: Composite Action을 Task로 남용
name: Overusing Composite
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup multiple tools separately
        run: |
          npm install -g yarn
          yarn install
          npm run lint
          npm run build
        # 10줄의 수동 스텝 → 재사용 불가
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 예 1: 작업 성격에 맞는 Action 선택
name: Efficient Action Usage
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # 필수 Action: 코드 체크아웃 (JavaScript 기반, 빠름)

      - uses: actions/setup-node@v4
        with:
          node-version: '18'
        # 시스템 셋업은 공식 Action 사용 (최적화됨)

      - run: npm install
        # 간단한 명령어는 run 사용

      - run: npm run build
        # 단순 작업은 Docker 오버헤드 피함

# ✅ 올바른 예 2: Action outputs 활용
name: Proper Output Usage
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Get version
        id: version
        run: |
          VERSION=$(jq -r '.version' package.json)
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - name: Create release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.version }}
          # 이전 Step의 output을 명시적으로 사용

# ✅ 올바른 예 3: Composite Action으로 재사용 가능한 흐름
name: Using Composite Actions
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup and build
        uses: ./.github/actions/setup-and-build
        with:
          node-version: '18'
          skip-tests: false
        # ./.github/actions/setup-and-build/action.yml에 정의됨
```

## 🔬 내부 동작 원리

### 1. Action 타입별 실행 방식

#### Docker Action (`using: docker`)

```yaml
# action.yml 예시
name: 'My Docker Action'
description: 'Run a containerized task'
runs:
  using: 'docker'
  image: 'docker://myrepo/myimage:latest'
  args:
    - ${{ inputs.arg1 }}
```

**실행 단계:**
```
1. action.yml 파싱 (image URI, args 읽음)
2. Docker 이미지 풀 (레지스트리에서 다운로드)
   - 캐시 있으면 스킵, 없으면 다운로드 (2-10초)
3. 컨테이너 시작
   - 환경변수 주입
   - inputs를 args로 전달
4. 컨테이너 내부에서 entrypoint 실행
5. 컨테이너 종료
6. outputs 파일 읽기 ($GITHUB_OUTPUT에 쓰여진 값)
```

**성능 특성:**
- 초기 이미지 풀: 2-10초
- 컨테이너 시작: 1-3초
- 실제 작업: 가변
- **총 오버헤드: 3-13초**

**보안 특성:**
- ✅ Host Runner와 완전히 격리
- ✅ 파일시스템 제한 가능
- ✅ 네트워크 제한 가능
- ❌ 격리로 인한 오버헤드

#### JavaScript Action (`using: node20`)

```yaml
# action.yml 예시
name: 'My Node Action'
description: 'Run JavaScript code'
runs:
  using: 'node20'
  main: 'dist/index.js'
```

**실행 단계:**
```
1. action.yml 파싱 (main 파일 읽음)
2. Node.js 프로세스로 직접 실행
   - $GITHUB_ACTION_PATH는 Action 디렉토리 경로
   - inputs를 환경변수 또는 process.argv로 전달
3. index.js 실행 (require('@actions/core') 사용)
4. core.setOutput()으로 outputs 설정
5. process.exit()로 종료
```

**성능 특성:**
- 초기 로드: 0.1-0.5초
- 실행: 가변
- **총 오버헤드: 0.1-0.5초 (Docker의 1/20)**

**보안 특성:**
- ❌ Host Runner와 같은 프로세스
- ❌ 파일시스템 제한 없음
- ❌ Runner의 모든 환경변수 접근 가능
- ✅ 간단하고 빠름

#### Composite Action (`using: composite`)

```yaml
# action.yml 예시
name: 'My Composite Action'
description: 'Combine multiple steps'
runs:
  using: 'composite'
  steps:
    - name: Step 1
      run: npm install
      shell: bash
    - name: Step 2
      run: npm run build
      shell: bash
```

**실행 방식:**
```
1. action.yml 파싱
2. steps 배열 반복
3. 각 step을 현재 Job의 Step으로 직접 실행
   - Docker나 Node.js 프로세스 없음
   - 현재 Runner 환경에서 직접 실행
4. 모든 outputs 수집
```

**성능:**
- 최소 오버헤드 (단순 step 실행)
- Docker보다 50배 빠름

**제약:**
- Docker 컨테이너 사용 불가
- 리눅스 기반 명령어만 실행 가능 (Windows와 macOS도 지원하려면 shell 신경써야 함)

### 2. Action Input/Output 메커니즘

**Input 전달:**
```yaml
# 1. Action 정의 (action.yml)
inputs:
  api-key:
    description: 'API Key'
    required: true
  version:
    description: 'Version'
    required: false
    default: '1.0.0'

# 2. Action 사용 (workflow.yml)
- uses: my-action@v1
  with:
    api-key: ${{ secrets.API_KEY }}
    version: 2.0.0

# 3. Action 내부에서 접근
# Docker Action에서는:
# - args로 전달: ["${{ inputs.api-key }}", "${{ inputs.version }}"]
# JavaScript Action에서는:
const core = require('@actions/core');
const apiKey = core.getInput('api-key');  // ${{ secrets.API_KEY }} 값
const version = core.getInput('version'); // '2.0.0'
```

**Output 전달:**
```yaml
# 1. Action 정의 (action.yml)
outputs:
  build-id:
    description: 'Build ID'
    value: ${{ steps.build.outputs.id }}

# 2. Action 내부 (Composite의 경우)
- id: build
  run: echo "id=12345" >> $GITHUB_OUTPUT
  shell: bash

# 3. Workflow에서 사용
- uses: my-action@v1
  id: build-action

- run: echo "Build ID: ${{ steps.build-action.outputs.build-id }}"
```

### 3. Reusable Workflow vs Composite Action 차이

| 항목 | Reusable Workflow | Composite Action |
|------|------------------|-----------------|
| 정의 위치 | `.github/workflows/*.yml` | `.github/actions/*/action.yml` |
| 입력 방식 | `inputs:` | `inputs:` |
| 실행 환경 | 새로운 Job (독립적) | 현재 Job 내 (Step으로 통합) |
| 격리 수준 | ✅ Job 경계로 완전 격리 | ❌ 현재 Step과 같은 프로세스 |
| outputs | `outputs:` → `needs.` 접근 | `outputs:` → `steps.` 접근 |
| 보안 컨텍스트 | 독립적 | 상속 |
| Job 통계 | 추가 Job 카운트 | 추가 카운트 없음 |

**선택 기준:**
- **Composite Action**: 동일 Job 내에서 단계를 재사용하고 싶을 때
- **Reusable Workflow**: 여러 Job을 단위로 재사용하거나 격리가 필요할 때

## 💻 실전 실험 (GitHub Actions YAML 재현 가능한 예시)

### 실험 1: Docker Action 구현 및 사용

**`.github/actions/docker-demo/action.yml`:**
```yaml
name: 'Docker Demo Action'
description: 'Demonstrate Docker action'
inputs:
  message:
    description: 'Message to print'
    required: true
outputs:
  timestamp:
    description: 'Current timestamp'
    value: ${{ steps.docker.outputs.timestamp }}
runs:
  using: 'docker'
  image: 'docker://ubuntu:22.04'
  args:
    - ${{ inputs.message }}
```

**`.github/actions/docker-demo/entrypoint.sh`:**
```bash
#!/bin/bash
set -e

echo "Received: $1"
TIMESTAMP=$(date +%s)
echo "timestamp=$TIMESTAMP" >> $GITHUB_OUTPUT
```

**`.github/workflows/test-docker-action.yml`:**
```yaml
name: Test Docker Action
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run docker action
        id: docker
        uses: ./.github/actions/docker-demo
        with:
          message: "Hello from Docker"

      - name: Use output
        run: echo "Timestamp: ${{ steps.docker.outputs.timestamp }}"
```

### 실험 2: JavaScript Action 구현 및 사용

**`.github/actions/js-demo/action.yml`:**
```yaml
name: 'JavaScript Demo Action'
description: 'Demonstrate JavaScript action'
inputs:
  number:
    description: 'Number to multiply'
    required: true
outputs:
  result:
    description: 'Number * 2'
    value: ${{ steps.js.outputs.result }}
runs:
  using: 'node20'
  main: 'dist/index.js'
```

**`.github/actions/js-demo/index.js`:**
```javascript
const core = require('@actions/core');

try {
  const number = parseInt(core.getInput('number'));
  const result = number * 2;
  
  core.setOutput('result', result);
  core.info(`Multiplied ${number} by 2 = ${result}`);
} catch (error) {
  core.setFailed(error.message);
}
```

**`.github/workflows/test-js-action.yml`:**
```yaml
name: Test JS Action
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run JS action
        id: js
        uses: ./.github/actions/js-demo
        with:
          number: '21'

      - name: Use output
        run: echo "Result: ${{ steps.js.outputs.result }}"
```

### 실험 3: Composite Action으로 복합 워크플로우

**`.github/actions/build-and-test/action.yml`:**
```yaml
name: 'Build and Test'
description: 'Build and test Node.js project'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '18'
outputs:
  test-passed:
    description: 'Whether tests passed'
    value: ${{ steps.test.outputs.passed }}
runs:
  using: 'composite'
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Install dependencies
      run: npm install
      shell: bash

    - name: Build
      run: npm run build
      shell: bash

    - id: test
      name: Run tests
      run: |
        npm test && echo "passed=true" >> $GITHUB_OUTPUT || echo "passed=false" >> $GITHUB_OUTPUT
      shell: bash
```

**`.github/workflows/test-composite.yml`:**
```yaml
name: Test Composite Action
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Build and test
        id: build
        uses: ./.github/actions/build-and-test
        with:
          node-version: '18'

      - name: Check result
        if: steps.build.outputs.test-passed == 'true'
        run: echo "All tests passed!"

      - name: Failed
        if: steps.build.outputs.test-passed == 'false'
        run: echo "Tests failed" && exit 1
```

### 실험 4: Action 성능 비교

**`.github/workflows/performance-test.yml`:**
```yaml
name: Performance Test
on: [push]

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Time Docker Action
        run: |
          START=$(date +%s%N)
          docker run --rm ubuntu:22.04 echo "Hello"
          END=$(date +%s%N)
          echo "Docker overhead: $(( (END - START) / 1000000 ))ms"

      - name: Time JavaScript
        run: |
          START=$(date +%s%N)
          node -e "console.log('Hello')"
          END=$(date +%s%N)
          echo "Node.js overhead: $(( (END - START) / 1000000 ))ms"

      - name: Time shell
        run: |
          START=$(date +%s%N)
          echo "Hello"
          END=$(date +%s%N)
          echo "Shell overhead: $(( (END - START) / 1000000 ))ms"
```

## 📊 성능/비용 비교

| Action 타입 | 오버헤드 | 속도 | 보안 격리 | 사용 사례 |
|-----------|---------|------|----------|---------|
| **Docker** | 3-15초 | 느림 | ✅ 완전 격리 | 의존성 충돌이 많은 작업, 완전 격리 필요 |
| **JavaScript** | 0.1-1초 | 빠름 | ❌ 없음 | 빠른 처리, CLI 도구 실행 |
| **Composite** | <100ms | 매우 빠름 | ❌ 없음 | 단순 step 조합, 재사용 |
| **run (shell)** | 거의 없음 | 최빠름 | ❌ 없음 | 간단한 명령어 |

**비용 계산:**
- Docker Action: 15초 × $0.005/분 ≈ $0.00125 / 실행
- JavaScript Action: 0.5초 × $0.005/분 ≈ $0.000042 / 실행
- **Docker는 JavaScript의 30배 비싼 오버헤드**

월 1000회 실행 시:
- Docker: $1.25
- JavaScript: $0.042

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 |
|------|------|------|
| **Docker Action** | 완전 격리, 어떤 언어든 가능 | 느림, 비쌈, 이미지 관리 필요 |
| **JavaScript Action** | 빠름, 간단함, GitHub 라이브러리 풍부 | 격리 없음, 보안 주의 필요 |
| **Composite Action** | 단순함, 빠름, 버전 관리 용이 | 조합 기능만 가능, 복잡한 로직 불가 |
| **shell run** | 최빠름, 최간단 | 재사용 불가, 읽기 어려움 |

## 📌 핵심 정리

1. **Docker Action**: 완전 격리가 필요할 때만 사용 (오버헤드 크)
2. **JavaScript Action**: 기본 선택지 (빠르고 간단)
3. **Composite Action**: 여러 Step을 재사용 가능한 단위로 만들 때
4. **Input/Output**: Action 간 데이터 전달의 표준 메커니즘
5. **Reusable Workflow**: Job 단위 재사용이 필요할 때
6. **성능**: JavaScript < Docker < Composite < run 순서로 빠름

## 🤔 생각해볼 문제

### 문제 1: 다음 Action을 Docker로 구현해야 할까, JavaScript로 구현해야 할까?

요구사항:
- Python 스크립트 실행 (ML 모델 추론)
- 입력: 이미지 경로
- 출력: 추론 결과 JSON
- 실행 시간: ~10초

<details>
<summary>해설</summary>

**답: Docker Action**

이유:
1. Python 의존성 관리: JavaScript 런타임에서는 Python 실행 불가
2. 실행 시간이 이미 10초: Action 오버헤드 3-10초는 상대적으로 30% 수준
3. 격리 필요: ML 모델 파일, Python 라이브러리 등은 격리된 환경에서 실행 권장

대안:
```yaml
# Docker Action으로 구현
runs:
  using: 'docker'
  image: 'docker://python:3.9'
  args:
    - ${{ inputs.image-path }}
```

반례: 단순 텍스트 처리(1초 이내) + Node.js 활용 → JavaScript Action 권장
</details>

### 문제 2: Composite Action으로 다음을 구현할 수 있을까?

```yaml
inputs:
  docker-image:
    required: true

outputs:
  image-id:
    value: ${{ steps.build.outputs.id }}

steps:
  - uses: actions/checkout@v4
  
  - name: Build Docker image
    id: build
    run: docker build -t ${{ inputs.docker-image }} .
    # 하지만 docker build의 image ID를 output으로 전달하려면?
```

<details>
<summary>해설</summary>

**답: 가능하지만 조심해야 함**

Composite Action 내에서:
```yaml
runs:
  using: 'composite'
  steps:
    - uses: actions/checkout@v4
    
    - id: build
      run: |
        docker build -t ${{ inputs.docker-image }} .
        IMAGE_ID=$(docker image inspect ${{ inputs.docker-image }} --format='{{.ID}}')
        echo "id=$IMAGE_ID" >> $GITHUB_OUTPUT
      shell: bash

outputs:
  image-id:
    value: ${{ steps.build.outputs.id }}
```

**주의:**
- Docker가 설치되어 있어야 함 (ubuntu-latest에는 있음)
- Windows/macOS Runner에서는 작동하지 않음
- 크로스 플랫폼이 필요하면 Docker Action 권장
</details>

### 문제 3: 이 구조에서 outputs은 작동할까?

```yaml
# action.yml
outputs:
  result:
    value: ${{ steps.step1.outputs.value }}

runs:
  using: 'composite'
  steps:
    - id: step1
      run: echo "value=success" >> $GITHUB_OUTPUT
      shell: bash
    
    - id: step2
      run: echo "value from step1: ${{ steps.step1.outputs.value }}"
      shell: bash
```

<details>
<summary>해설</summary>

**답: 완전히 작동함**

Composite Action 내부에서:
- `steps.<id>.outputs`는 같은 Composite 내 이전 step의 output 접근 가능
- Action의 outputs는 마지막 step의 outputs를 전달

시간 순서:
1. step1 실행 → $GITHUB_OUTPUT에 "value=success" 기록
2. step2 실행 → `${{ steps.step1.outputs.value }}`로 "success" 접근 가능
3. Action 종료 시 outputs에서 `${{ steps.step1.outputs.value }}` 평가 → "success"

workflow에서:
```yaml
- uses: ./.github/actions/my-action
  id: myaction

- run: echo "Final result: ${{ steps.myaction.outputs.result }}"
  # "success" 출력
```

**주의:** outputs의 value 필드는 Action 실행 완료 후에 평가되므로, 모든 step의 output이 준비되어 있어야 함
</details>

---

<div align="center">
**[⬅️ 이전: Workflow YAML 구조 분해](./01-workflow-yaml-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Job 의존성과 병렬화 ➡️](./03-job-dependency-parallelism.md)**
</div>
