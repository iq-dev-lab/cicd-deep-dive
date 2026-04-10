# Workflow YAML 구조 분해 — 키워드 처리 방식

## 🎯 핵심 질문

GitHub Actions의 YAML 파일을 작성할 때, `on:`, `jobs:`, `steps:`, `env:`, `secrets:`는 정확히 어떻게 처리될까? 표현식 `${{ }}` 은 언제 평가되고, `if:` 조건은 어느 단계에서 체크될까? 환경변수를 여러 수준에서 설정하면 어떤 우선순위로 적용될까?

## 🔍 왜 이 개념이 실무에서 중요한가

- **디버깅 능력**: 워크플로우가 예상과 다르게 동작할 때, YAML 키워드가 어떻게 처리되는지 알아야 원인을 찾을 수 있다.
- **보안**: 표현식 인젝션으로부터 자격증명을 보호하려면 GitHub의 내부 처리 방식을 이해해야 한다.
- **성능**: 불필요한 스텝을 `if:` 조건으로 스킵할 수 있으면 Runner 시간을 절약할 수 있다.
- **유지보수성**: 환경변수 우선순위를 이해하면 설정 오류를 줄일 수 있다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 예 1: 표현식 인젝션 취약점
name: Unsafe Workflow
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Unsafe secret logging
        run: echo "Token is ${{ secrets.GITHUB_TOKEN }}"
        # 표현식이 먼저 평가된 후 run에 전달되어, 
        # 로그에 토큰이 노출될 수 있다 (마스킹이 작동하지 않을 수 있음)

# ❌ 잘못된 예 2: 환경변수 우선순위 무시
name: Wrong Env Priority
on: [push]
env:
  VERSION: 1.0.0
jobs:
  build:
    env:
      VERSION: 2.0.0
    runs-on: ubuntu-latest
    steps:
      - name: Check version
        env:
          VERSION: 3.0.0
        run: echo $VERSION  # 어느 값이 출력될까?

# ❌ 잘못된 예 3: if 조건을 문자열로 작성
name: Wrong Condition
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: This always runs
        run: echo "Running"
      - name: This should skip
        if: "false"  # 문자열 "false"는 truthy이므로 실행된다!
        run: echo "Skipped?"
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 예 1: 환경변수로 표현식 전달 (인젝션 방지)
name: Safe Workflow
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Safe secret handling
        env:
          TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # 환경변수를 통해 전달하면 GitHub이 마스킹 처리
          echo "Checking with token"
          # TOKEN 값이 로그에 노출되지 않음

# ✅ 올바른 예 2: 명확한 환경변수 우선순위 설정
name: Correct Env Priority
on: [push]
env:
  VERSION: 1.0.0        # Workflow 수준 (가장 낮은 우선순위)
  DEBUG: false
jobs:
  build:
    env:
      VERSION: 2.0.0    # Job 수준 (중간 우선순위, Workflow 덮어씀)
    runs-on: ubuntu-latest
    steps:
      - name: Step level override
        env:
          VERSION: 3.0.0  # Step 수준 (가장 높은 우선순위)
          DEBUG: true
        run: |
          echo "VERSION=$VERSION"  # 3.0.0 출력
          echo "DEBUG=$DEBUG"      # true 출력

# ✅ 올바른 예 3: 조건식을 올바르게 작성
name: Correct Conditions
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: This always runs
        run: echo "Running"
      - name: This skips correctly
        if: false  # 불리언 (마스킹되지 않음)
        run: echo "Skipped"
      - name: Conditional based on event
        if: github.event_name == 'push'
        run: echo "Push event detected"
```

## 🔬 내부 동작 원리

### 1. YAML 파싱 및 키워드 처리 순서

GitHub Actions는 다음 순서로 YAML을 처리합니다:

```
1. YAML 파일 파싱 (on:, jobs:, name: 등을 읽음)
2. 트리거 조건 평가 (on: 섹션 처리)
3. 각 Job을 순차적으로 또는 병렬로 준비
4. Runner 할당 및 Job 시작
5. 각 Step 실행 전에 if: 조건 평가
6. Step 실행 시점에 ${{ }} 표현식 평가
```

### 2. 표현식 `${{ }}` 평가 시점

**서버 측 평가 (GitHub의 서버):**
- `on:` 섹션의 필터 표현식
- Job-level `if:` (암묵적으로)
- 조건부 Job 실행

**Runner 측 평가 (실제 Runner 머신):**
- Step-level `run:` 내부의 표현식
- Step-level `if:` 조건
- 환경변수로 전달된 표현식

예시:
```yaml
on:
  push:
    branches:
      - main
      - ${{ github.ref == 'refs/heads/develop' && 'develop' || '' }}
      # ❌ 위는 작동하지 않음 (on:은 서버에서 미리 파싱됨)

jobs:
  build:
    if: github.event_name == 'push'  # ✅ 서버에서 평가
    runs-on: ubuntu-latest
    steps:
      - run: echo "Event: ${{ github.event_name }}"  
        # ✅ Runner에서 평가됨
```

### 3. 환경변수 우선순위 (Shadowing)

GitHub Actions는 UNIX 셸 규칙을 따릅니다:

```
Step level env > Job level env > Workflow level env > Runner 기본값
```

```yaml
env:
  LOG_LEVEL: info              # Workflow level

jobs:
  test:
    env:
      LOG_LEVEL: debug         # Job level이 Workflow 덮어씀
      API_URL: https://api.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Check variables
        env:
          LOG_LEVEL: verbose   # Step level이 모두 덮어씀
        run: |
          echo $LOG_LEVEL      # verbose 출력
          echo $API_URL        # https://api.example.com 출력
```

### 4. Secrets의 마스킹 메커니즘

GitHub는 Runner로 전달되기 전에 Secret 값을 다음과 같이 처리합니다:

```
1. GitHub 서버에서 Secret 값 읽기
2. Runner에 표현식 ${{ secrets.NAME }}과 실제 값 전송
3. Runner는 Secret 값을 메모리에 기록하지 않음
4. 모든 출력에서 Secret 값을 정규표현식으로 찾아 "***"로 교체
5. 로그에 전송
```

마스킹이 작동하려면:
- Secret을 **환경변수로 전달**해야 함
- Secret을 **직접 echo하면 마스킹이 우회될 수 있음**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Safe way
        env:
          TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: echo "Using token"  # 마스킹됨

      - name: Unsafe way
        run: echo "${{ secrets.GITHUB_TOKEN }}"
        # 문제: 표현식이 평가되고 값이 echo에 전달되므로 
        # 마스킹이 작동하지 않을 수 있음
```

### 5. `if:` 조건의 컨텍스트 범위

`if:` 조건이 평가될 때 접근 가능한 컨텍스트:

```yaml
jobs:
  build:
    if: github.event_name == 'push'  # ✅ github, env 컨텍스트 사용 가능
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - id: version
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT

      - name: Job output access
        if: job.outputs.version == '1.0.0'  # ❌ 실패: 같은 Job의 outputs는 if에서 미사용
        run: echo "Not accessible"

  deploy:
    needs: build
    if: success()  # ✅ 이전 Job의 상태 확인 가능
    runs-on: ubuntu-latest
    steps:
      - name: Use previous output
        run: echo "Version: ${{ needs.build.outputs.version }}"
```

## 💻 실전 실험 (GitHub Actions YAML 재현 가능한 예시)

### 실험 1: 환경변수 우선순위 확인

```yaml
name: Environment Priority Test
on: [push]

env:
  LEVEL: workflow
  DEBUG: false

jobs:
  priority-test:
    env:
      LEVEL: job
      API_KEY: ${{ secrets.API_KEY }}
    runs-on: ubuntu-latest
    steps:
      - name: Workflow level only
        run: echo "DEBUG at workflow level: $DEBUG"

      - name: Job level override
        run: echo "LEVEL at job level: $LEVEL"

      - name: Step level override
        env:
          LEVEL: step
          DEBUG: true
        run: |
          echo "LEVEL at step level: $LEVEL"  # step 출력
          echo "DEBUG at step level: $DEBUG"  # true 출력

      - name: Secret in env var
        env:
          SECRET_VAR: ${{ secrets.API_KEY }}
        run: echo "Secret safe"  # 로그에 Secret이 *** 마스킹됨
```

### 실험 2: 조건부 실행

```yaml
name: Conditional Execution Test
on: 
  push:
    branches:
      - main
      - develop

jobs:
  conditional-steps:
    runs-on: ubuntu-latest
    steps:
      - name: Always run
        run: echo "This always runs"

      - name: Run only on main
        if: github.ref == 'refs/heads/main'
        run: echo "Only on main branch"

      - name: Skip with false condition
        if: false
        run: echo "This is skipped"

      - name: Run only on failures
        if: failure()
        run: echo "Previous step failed"

      - name: Check event type
        if: github.event_name == 'push'
        run: echo "This is a push event"

      - name: Expression evaluation
        if: ${{ 1 + 1 == 2 }}
        run: echo "Math works"
```

### 실험 3: 표현식 인젝션 방지

```yaml
name: Expression Injection Prevention
on: [push]

jobs:
  security-test:
    runs-on: ubuntu-latest
    steps:
      - name: Safe: Use environment variables
        env:
          COMMIT_MSG: ${{ github.event.head_commit.message }}
          USER_INPUT: ${{ github.event.client_payload.user_input }}
        run: |
          echo "Commit message: $COMMIT_MSG"
          echo "User input: $USER_INPUT"
          # 어떤 입력이 와도 환경변수로 격리됨

      - name: Context used safely
        run: |
          # 트러스트된 컨텍스트만 직접 사용
          echo "Actor: ${{ github.actor }}"
          echo "Ref: ${{ github.ref }}"
          echo "Repo: ${{ github.repository }}"

      - name: Avoid dynamic job names or secrets
        env:
          SAFE_ENV: safe_value
        # run: echo "Token: ${{ secrets.GITHUB_TOKEN }}"  # ❌ 피하기
        # run: jobs:
        #   dynamic-${{ github.ref }}: ...  # ❌ 불가능하고 위험
        run: echo "Static approach only"
```

## 📊 성능/비용 비교

| 항목 | 비용/성능 영향 | 설명 |
|------|--------------|------|
| 불필요한 Step 실행 | ❌ 높음 | `if: false`로 스킵 가능 → Runner 분당 비용 절감 |
| 여러 `env:` 레벨 | ✅ 낮음 | 해석 오버헤드 무시할 수준 |
| 표현식 평가 | ✅ 낮음 | 대부분 밀리초 단위 |
| Secret 마스킹 | ✅ 낮음 | 로그 처리 시간 미미 |
| 환경변수 개수 | ⚠️ 중간 | 100개 이상이면 Runner 메모리 영향 |

**최적화 팁:**
- 조건부 실행으로 불필요한 Step 제거: **분당 비용 5~10% 절감 가능**
- Secret을 환경변수로만 전달: **보안과 성능 동시 달성**

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 |
|------|------|------|
| **Step level `env`** | 격리된 환경, 명확함 | 반복적인 설정 |
| **Job level `env`** | 재사용성, 간결함 | 범위 파악 어려움 |
| **Workflow level `env`** | 중앙 관리 | 하위 Step에서 덮어쓰면 혼란 |
| **Secret 직접 사용** | 간단함 | 보안 위험, 마스킹 불확실 |
| **Secret을 env로 감싸기** | 안전함, 마스킹 보장 | 한 단계 더 거쳐야 함 |

## 📌 핵심 정리

1. **YAML 키워드는 계층적으로 처리**: Workflow → Job → Step 순서로 우선순위가 높아짐
2. **표현식 평가 시점이 다름**: 서버(on:) vs Runner(run:)에서 평가되는 시점 차이 이해 필수
3. **환경변수 우선순위**: Step > Job > Workflow (UNIX 셸 규칙)
4. **Secret 마스킹은 환경변수 경유 필수**: 직접 표현식으로 사용하면 마스킹이 작동하지 않을 수 있음
5. **`if:` 조건은 불리언 평가**: 문자열 "false"는 truthy (false 마크업 필수)
6. **표현식 인젝션 방지**: 신뢰되지 않은 입력은 환경변수로 격리

## 🤔 생각해볼 문제

### 문제 1: 다음 YAML에서 3개 echo의 출력은?

```yaml
env:
  MSG: "workflow"

jobs:
  test:
    env:
      MSG: "job"
    runs-on: ubuntu-latest
    steps:
      - name: Check 1
        run: echo $MSG

      - name: Check 2
        env:
          MSG: "step"
        run: echo $MSG

      - name: Check 3
        run: echo $MSG
```

<details>
<summary>해설</summary>

- Check 1: **job** 출력 (Job level env가 Workflow를 덮어씀)
- Check 2: **step** 출력 (Step level env가 최우선)
- Check 3: **job** 출력 (Check 2의 Step env는 그 Step에만 적용됨)

핵심: 환경변수 스코프는 각 Step 단위로 격리됨
</details>

### 문제 2: 이 조건식이 작동할까?

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      status: ${{ job.status }}
    steps:
      - run: exit 0

  deploy:
    needs: build
    if: needs.build.outputs.status == 'success'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
```

<details>
<summary>해설</summary>

❌ **작동하지 않음**

이유:
- `job.status`는 현재 Job의 상태이고, Job이 실행 중일 때는 이미 'success', 'failure' 등으로 확정된 상태
- outputs는 모든 Step이 완료된 후에만 사용 가능
- 더 나은 방법: `if: success()` 또는 Step의 `outputs` 사용

올바른 예:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - id: test
        run: echo "BUILD_RESULT=success" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    if: needs.build.result == 'success'  # Job 결과 확인
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
```
</details>

### 문제 3: Secret을 "안전하게" 사용하는 3가지 방법을 구분하라

```yaml
방법 A) run: echo "${{ secrets.TOKEN }}"
방법 B) env:
        TOKEN: ${{ secrets.TOKEN }}
       run: echo "Using token"
방법 C) run: TOKEN=${{ secrets.TOKEN }}; echo "Using token"
```

<details>
<summary>해설</summary>

**보안 순위:**
1. **방법 B (가장 안전)** ✅
   - 환경변수로 전달하면 GitHub의 마스킹 처리가 보장됨
   - Secret 값이 Command line에 노출되지 않음

2. **방법 C** ⚠️
   - 환경변수로 설정하지만, 같은 라인에서 실행되므로 history에 남을 수 있음
   - 배쉬 히스토리가 기록되면 노출 가능

3. **방법 A** ❌
   - 표현식이 직접 echo에 전달되므로 마스킹이 작동하지 않을 수 있음
   - 로그에 Secret이 노출될 위험이 높음

**권장사항:**
```yaml
# 최고의 방법
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Use secret safely
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          # 이 라인에서 echo를 해도 마스킹됨
          curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
```
</details>

---

<div align="center">
**[홈으로 🏠](../README.md)** | **[다음: Actions 내부 동작 ➡️](./02-actions-internals.md)**
</div>
