# GitHub Actions 내부 동작 — Webhook에서 Job 실행까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `git push` 이후 GitHub Actions Runner가 Job을 실행하기까지 어떤 과정이 일어나는가?
- Runner는 어떻게 Job을 할당받는가? 대기 중인 Job은 어디에 쌓이는가?
- `GITHUB_TOKEN`은 어디서 생성되고, 어떤 권한을 가지며, 어떻게 만료되는가?
- Job 격리는 어떻게 구현되는가 — VM인가 컨테이너인가?
- Workflow YAML의 표현식 `${{ }}` 는 언제, 어디서 평가되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

GitHub Actions를 수개월 사용해도 "Runner가 Job을 어떻게 받아가는가"를 모르는 개발자가 많다. 이 모름은 장애 상황에서 치명적으로 드러난다. Runner가 응답 없이 Job이 대기 중일 때, `GITHUB_TOKEN`으로 API를 호출했는데 권한 오류가 날 때, Workflow가 예상과 다른 순서로 실행될 때 — 내부 구조를 알면 5분 안에 원인을 찾고, 모르면 수 시간 방황한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# 실수 1: GITHUB_TOKEN 권한 오류를 이해 못 하는 경우
jobs:
  create-release:
    runs-on: ubuntu-latest
    steps:
      - name: Release 생성
        run: |
          gh release create v1.0.0 \
            --title "Release 1.0.0" \
            --notes "Release notes"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
# 오류: "Resource not accessible by integration"
# 원인 파악 못 함 → 토큰이 왜 권한이 없는지 모름

# 실수 2: Self-hosted Runner 연결 안 됨 → 이유 모름
jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - run: echo "배포"
# Job이 계속 Queued 상태 → "Runner가 왜 안 잡히지?"
# Runner 프로세스가 죽어있는지 확인할 생각을 못 함

# 실수 3: 표현식이 예상과 다르게 동작
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo ${{ github.event.pull_request.head.sha }}
# pull_request 이벤트가 아닌 push 이벤트에서 실행하면 빈 값
# 왜 비어있는지 이해 못 함
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# GITHUB_TOKEN 권한 명시적 설정
permissions:
  contents: write      # Release 생성에 필요
  packages: write      # 패키지 레지스트리 push에 필요
  pull-requests: write # PR 코멘트에 필요

jobs:
  create-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Release 생성
        run: |
          gh release create v1.0.0 \
            --title "Release 1.0.0" \
            --notes "Release notes"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# Self-hosted Runner 상태 확인 방법
# GitHub 레포 → Settings → Actions → Runners
# "Idle" = 대기 중 (정상)
# "Offline" = 프로세스 죽음 → 서버에서 ./run.sh 재실행

# 이벤트별 컨텍스트 확인
jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - name: 전체 컨텍스트 출력
        run: echo '${{ toJSON(github) }}'
```

---

## 🔬 내부 동작 원리

### git push → Job 실행까지 전체 흐름

```
1. 개발자: git push origin main

2. GitHub 서버:
   - 레포지토리 업데이트
   - push 이벤트 생성
   - .github/workflows/*.yml 파일 스캔
   - Trigger 조건 매칭 확인 (on.push.branches)

3. Workflow Run 생성:
   - 각 매칭된 Workflow 파일에 대해 Run 생성
   - Run ID 할당 (숫자)
   - Job 목록 생성 (DAG 순서 고려)
   - 초기 상태: Queued

4. Runner 선택:
   [GitHub-hosted]
   - GitHub의 Runner 풀에서 사용 가능한 VM 할당
   - ubuntu-latest: Azure VM에서 프로비저닝 (약 10~30초)
   - Job과 매칭되는 Runner 레이블 확인

   [Self-hosted]
   - 레포지토리/조직/엔터프라이즈 수준 Runner 목록 확인
   - Idle 상태인 Runner에게 Job 할당
   - Runner가 없으면 대기 (기본 6시간 타임아웃)

5. Runner에서 Job 실행:
   a. Runner 에이전트가 GitHub으로부터 Job 수신
   b. 작업 디렉토리 초기화
   c. GITHUB_TOKEN 발급 (Job 시작 시 생성)
   d. steps 순서대로 실행

6. Job 완료:
   - 결과 (success/failure/cancelled) 보고
   - GITHUB_TOKEN 만료
   - GitHub-hosted: VM 폐기
   - Self-hosted: 다음 Job 대기 상태로 전환
```

### Runner 아키텍처 — Poll vs Webhook

Runner가 Job을 받는 방식은 **Poll(풀링)** 이다. Push(푸시)가 아니다.

```
GitHub 서버                    Runner 에이전트
    │                               │
    │  ← GET /job_requests (롱폴링) │
    │    (30초마다 연결 시도)        │
    │                               │
    │  Job 있으면:                  │
    │  → 202 Accepted + Job 데이터 →│
    │                               │
    │  Job 없으면:                  │
    │  → 60초 대기 후 재연결        │
```

Runner는 GitHub 서버를 향해 "Job 있어요?"라고 계속 묻는다. 이 방식의 장점은 Runner가 방화벽 뒤에 있어도 동작한다는 것이다. Runner → GitHub 방향 아웃바운드 연결만 필요하고, 인바운드 포트를 열 필요가 없다.

이것이 Self-hosted Runner를 설치할 때 별도의 포트 포워딩이 필요 없는 이유다.

### GITHUB_TOKEN — 자동 발급되는 임시 토큰

```
Job 시작 시:
  GitHub Actions 서비스 → GITHUB_TOKEN 생성
  - 해당 레포지토리에 대한 권한 포함
  - Job이 실행되는 동안만 유효
  - Job 완료 또는 1시간 후 자동 만료

기본 권한 (레포 설정에 따라 다름):
  contents: read       # 코드 읽기
  metadata: read       # 레포 메타데이터 읽기
  packages: read       # 패키지 읽기
  # 쓰기 권한은 기본으로 없거나 제한됨

명시적 권한 설정:
```

```yaml
# Workflow 수준 (모든 Job에 적용)
permissions:
  contents: write
  packages: write

# Job 수준 (특정 Job에만 적용)
jobs:
  build:
    permissions:
      contents: read
      packages: write
    runs-on: ubuntu-latest
```

레포지토리 기본 토큰 권한 설정: Settings → Actions → General → Workflow permissions

- "Read repository contents and packages permissions" (기본, 더 안전)
- "Read and write permissions" (편하지만 덜 안전)

최소 권한 원칙: 필요한 권한만 Job 수준에서 명시적으로 선언한다.

### Job 격리 — VM vs 컨테이너

GitHub-hosted Runner의 격리 방식:

```
runs-on: ubuntu-latest
  → Azure VM (Standard_DS2_v2 또는 동급)
  → 각 Job마다 새로운 VM 인스턴스
  → Job 완료 후 VM 삭제
  → 완전한 OS 수준 격리

runs-on: ubuntu-latest + container:
  jobs:
    build:
      runs-on: ubuntu-latest
      container:
        image: node:20-alpine    # 이 컨테이너 안에서 Step 실행
        options: --user 1000
      steps:
        - run: node --version    # 컨테이너 안의 Node.js 사용
```

`container:` 옵션을 사용하면 VM 위에서 Docker 컨테이너를 실행하고, Step들이 그 컨테이너 안에서 실행된다. 특정 런타임 버전이나 환경이 필요할 때 유용하다.

### 표현식 `${{ }}` 의 평가 시점

```yaml
# 표현식의 종류와 평가 시점

# 1. 서버 측 평가 (Job 시작 전)
if: github.event_name == 'push'   # Workflow 실행 여부 결정에 사용

# 2. Runner 측 평가 (Step 실행 시)
run: echo "커밋: ${{ github.sha }}"

# 3. 평가 시점 차이가 중요한 경우
jobs:
  build:
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    steps:
      # 이 시점에 build Job의 outputs가 평가됨
      - run: echo "버전: ${{ needs.build.outputs.version }}"
```

보안 주의사항 — 표현식 인젝션:

```yaml
# 위험한 패턴: PR 제목을 직접 run에서 사용
- run: echo "${{ github.event.pull_request.title }}"
# PR 제목이 "hello; rm -rf /"이면 명령어 인젝션 발생

# 안전한 패턴: 환경변수를 거쳐서 사용
- name: 안전한 사용
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "$PR_TITLE"
# 환경변수로 전달하면 쉘 해석 없이 문자열로 처리됨
```

---

## 💻 실전 실험

### 실험 1: Workflow 실행 전체 과정 관찰

```yaml
name: 내부 동작 관찰

on: workflow_dispatch

jobs:
  observe:
    runs-on: ubuntu-latest
    steps:
      - name: Runner 환경 정보
        run: |
          echo "=== Runner 정보 ==="
          echo "OS: $RUNNER_OS"
          echo "Architecture: $RUNNER_ARCH"
          echo "Runner Name: $RUNNER_NAME"
          echo "Workspace: $GITHUB_WORKSPACE"

      - name: GITHUB_TOKEN 정보 (토큰 자체는 출력 안 됨)
        run: |
          echo "=== 토큰 정보 ==="
          # 토큰 값 자체는 마스킹됨
          echo "Token 존재: ${{ secrets.GITHUB_TOKEN != '' }}"
          # API로 토큰 권한 확인
          curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
               https://api.github.com/repos/${{ github.repository }}/actions/permissions \
               | jq '.permissions'

      - name: GitHub 컨텍스트 전체 출력
        run: echo '${{ toJSON(github) }}'
```

### 실험 2: GITHUB_TOKEN 권한 제한 실험

```yaml
name: 권한 테스트

on: workflow_dispatch

# 최소 권한으로 시작
permissions:
  contents: read

jobs:
  test-permissions:
    runs-on: ubuntu-latest
    steps:
      - name: Issue 생성 시도 (권한 없음)
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            -H "Content-Type: application/json" \
            https://api.github.com/repos/${{ github.repository }}/issues \
            -d '{"title":"테스트 이슈"}' | jq '.message'
          # → "Resource not accessible by integration" 예상

      - name: 코드 읽기 (권한 있음)
        run: |
          curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${{ github.repository }}/contents/README.md \
            | jq '.message // "성공"'
          # → 성공 예상
```

### 실험 3: Runner 선택과 레이블 확인

```yaml
jobs:
  check-runner:
    runs-on: ubuntu-22.04    # 특정 버전 고정
    steps:
      - run: |
          echo "Ubuntu 버전:"
          lsb_release -a
          echo ""
          echo "사전 설치된 도구:"
          java -version
          docker --version
          kubectl version --client 2>/dev/null || echo "kubectl 없음"
```

GitHub-hosted Ubuntu Runner에 사전 설치된 도구 목록: https://github.com/actions/runner-images

---

## 📊 성능/비용 비교

### GitHub-hosted Runner 프로비저닝 시간

```
ubuntu-latest:   약 10~20초 (가장 빠름)
windows-latest:  약 30~60초
macos-latest:    약 30~90초 (비용도 10배)

Self-hosted Runner (Idle 상태):
  Job 수신 즉시 실행 (0~3초)
  → 대용량 캐시가 Runner에 보관된 경우 추가 이득
```

### 과금 기준

```
GitHub Free/Pro (private 레포):
  ubuntu-latest:  $0.008/분
  windows-latest: $0.016/분 (2배)
  macos-latest:   $0.080/분 (10배)

월 무료 한도:
  Free: 2,000분/월
  Pro:  3,000분/월
  Team: 3,000분/월

public 레포: 무료
Self-hosted: Runner 서버 비용만 부담
```

---

## ⚖️ 트레이드오프

**GitHub-hosted Runner 한계**

- Job당 최대 6시간 실행 시간 제한
- 2코어 CPU, 7GB RAM 제한 (대용량 빌드에 부족)
- Job마다 새로운 환경이므로 Docker 레이어 캐시 등을 재사용하기 어려움
- `actions/cache`로 보완하지만 완전하지 않음

**Self-hosted Runner 위험**

- 외부 기여자 PR에서 실행 시 악의적 코드가 내부 인프라에 접근 가능
- Runner 서버 관리 필요 (패치, 업데이트, 모니터링)
- Runner 프로세스가 다운되면 Job이 대기 상태로 멈춤

**GITHUB_TOKEN vs Personal Access Token**

| | GITHUB_TOKEN | PAT (Personal Access Token) |
|-|-------------|----------------------------|
| 수명 | Job 완료 시 만료 | 설정한 기간 (최대 1년) |
| 권한 | 레포 한정 | 설정 범위대로 |
| 노출 위험 | 낮음 (자동 만료) | 높음 (장기 유효) |
| 권장 | 레포 내 작업 | 다른 레포/서비스 접근 시 |

---

## 📌 핵심 정리

- `git push` → Webhook → Workflow 매칭 → Run 생성 → Runner 할당 → Job 실행의 순서로 진행된다
- Runner는 **Pull(롱폴링)** 방식으로 Job을 받아간다 — 방화벽 뒤에서도 동작하는 이유
- `GITHUB_TOKEN`은 Job 시작 시 자동 발급되는 임시 토큰으로, Job 완료 시 만료된다
- 권한은 최소 원칙으로, `permissions:` 블록에서 필요한 것만 명시한다
- GitHub-hosted Runner는 Job마다 새 VM, Self-hosted는 같은 서버 재사용
- 표현식 `${{ }}` 를 `run:` 에서 직접 사용하면 인젝션 취약점 — 환경변수를 거쳐 사용한다

---

## 🤔 생각해볼 문제

**Q1.** Self-hosted Runner를 Kubernetes Pod로 실행하면 어떤 이점이 있는가?

<details>
<summary>해설 보기</summary>

Kubernetes에서 Runner를 실행하면 Job이 들어올 때 Pod를 동적으로 생성하고 완료 시 삭제할 수 있다 (Actions Runner Controller 사용). 이를 통해 GitHub-hosted Runner처럼 Job마다 깨끗한 환경을 제공하면서, Self-hosted Runner의 이점(내부망 접근, 캐시 유지, 커스텀 스펙)도 누릴 수 있다. Pod 당 Runner 방식으로 완전한 격리도 가능하다.

</details>

**Q2.** `${{ github.event.pull_request.head.sha }}`가 push 이벤트에서 빈 값이 나오는 이유는?

<details>
<summary>해설 보기</summary>

`github.event`는 Workflow를 트리거한 이벤트의 페이로드다. push 이벤트의 페이로드에는 `pull_request` 키가 없다. 따라서 `github.event.pull_request`는 null이고, `.head.sha`는 빈 문자열이 된다. 커밋 SHA를 얻으려면 `github.sha`를 사용하면 어떤 이벤트에서도 일관되게 동작한다.

</details>

**Q3.** Job 실행 중 Runner 서버가 갑자기 재시작되면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Self-hosted Runner 기준으로, Runner 프로세스가 죽으면 GitHub 서버는 약 1~2분 후 Runner가 오프라인임을 감지한다. 실행 중이던 Job은 실패(Failure)로 표시된다. Job이 자동으로 재시도되지는 않는다. Workflow에 `retry-on-failure` 설정이 없다면 수동으로 재실행해야 한다. 중요한 배포 Job이라면 `if: failure()` Step으로 알림을 보내고, GitHub-hosted Runner를 사용하거나 Runner를 Kubernetes Pod로 관리해 자동 복구를 구성하는 것이 좋다.

</details>

---

<div align="center">

**[⬅️ 이전: Pipeline 구성 요소](./02-pipeline-components.md)** | **[홈으로 🏠](../README.md)** | **[다음: Pipeline 설계 원칙 — Fast Fail과 병렬 실행 ➡️](./04-pipeline-design-principles.md)**

</div>
