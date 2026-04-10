# GitOps 원칙 — Git이 시스템 상태의 Single Source of Truth

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GitOps의 4원칙(Declarative, Versioned, Automated, Continuously Reconciled)은 각각 무엇을 의미하는가?
- 명령형 배포(`kubectl apply`)와 선언형 배포(GitOps)는 실제로 무엇이 다른가?
- Push 모델이 Pull 모델보다 보안상 불리한 이유는 무엇인가?
- Git 커밋이 어떻게 Audit Log(감사 추적)를 대체할 수 있는가?
- GitOps를 도입했는데 팀원이 `kubectl edit`으로 직접 수정하면 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

ArgoCD나 Flux를 "YAML을 Git에 올리면 자동으로 배포되는 도구" 정도로 이해하는 팀이 많다. 하지만 GitOps는 단순한 배포 자동화 도구가 아니다. 시스템의 "원하는 상태(Desired State)"를 Git에 선언적으로 정의하고, 실제 시스템이 그 상태를 향해 지속적으로 수렴하도록 강제하는 **운영 모델**이다.

이 차이를 이해하지 못하면 GitOps 도구를 도입해도 `kubectl edit`으로 직접 수정하거나, Helm 명령어로 수동 배포하는 관행이 사라지지 않는다. 도구는 GitOps지만 방식은 여전히 명령형인 이상한 상태가 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: ArgoCD를 도입했지만 여전히 발생하는 혼란

문제 1: "급하니까 kubectl edit으로 직접 수정"
  → ArgoCD가 3분 후 Git 상태로 되돌림
  → 팀원: "내가 수정한 게 왜 원래대로 돌아왔지?"
  → 원인 모름 → "ArgoCD가 이상하다"

문제 2: Production만 ArgoCD, Staging은 수동 kubectl apply
  → 환경 간 설정 불일치 발생
  → "스테이징에서 됐는데 프로덕션에서 안 됨"
  → GitOps의 이점 반감

문제 3: Kubernetes Secret을 Git에 평문 저장
  → git log로 히스토리 조회 가능
  → 나중에 삭제해도 히스토리에 영구 기록
  → 보안 감사에서 지적

문제 4: 이미지 태그를 latest로 설정
  # k8s/deployment.yaml
  image: myapp:latest     # latest가 무엇을 가리키는지 Git만 봐서는 알 수 없음
  → "어제 배포된 버전이 뭔지 모르겠다"
  → GitOps의 Versioned 원칙 위반
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# GitOps 올바른 운영 패턴

# 1. 모든 변경은 Git을 통해서만
# ❌ kubectl edit deployment myapp
# ✅ Git에서 deployment.yaml 수정 → PR → 머지 → ArgoCD가 자동 적용

# 2. 이미지 태그는 항상 구체적인 버전
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:main-abc1234   # ← Git SHA 포함 태그
          # latest 대신 재현 가능한 태그 사용

# 3. 배포 이력이 Git 커밋 히스토리로 추적됨
# git log k8s/deployment.yaml
# → 누가, 언제, 무엇을 배포했는지 코드 리뷰 기록과 함께 남음
```

---

## 🔬 내부 동작 원리

### GitOps 4원칙 (OpenGitOps v1.0)

**원칙 1: Declarative (선언적)**

"이렇게 해라"(명령형)가 아닌 "이런 상태여야 한다"(선언형)로 시스템을 정의한다.

```bash
# 명령형 (Imperative):
kubectl scale deployment myapp --replicas=5   # "5개로 늘려라"
kubectl set image deployment/myapp myapp=v2   # "이미지를 변경해라"

# 선언형 (Declarative):
# k8s/deployment.yaml에 원하는 상태를 정의
spec:
  replicas: 5
  template:
    spec:
      containers:
        - image: myapp:v2
# → "이 상태여야 한다"고 선언, 어떻게 달성할지는 Kubernetes가 결정
```

명령형의 문제: 명령을 실행한 사람, 시간, 이유가 기록되지 않는다. 시스템이 지금 어떤 상태인지 코드를 봐도 알 수 없다.

선언형의 이점: Git에 있는 YAML이 시스템의 "의도된 상태"다. 실제 상태와 비교해 차이를 자동으로 감지할 수 있다.

**원칙 2: Versioned and Immutable (버전화와 불변성)**

원하는 상태는 Git에 저장된다. Git의 모든 커밋은 불변이고(내용이 변경되면 SHA가 바뀜), 완전한 히스토리를 가진다.

```bash
# 배포 이력 조회 — kubectl history보다 훨씬 풍부한 정보
git log --oneline k8s/deployment.yaml
# abc1234 feat: scale up to 5 replicas for Black Friday (2024-11-29)
# def5678 fix: rollback to v1.2.3 due to OOM issue (2024-11-28)
# ghi9012 feat: deploy v1.3.0 with new checkout flow (2024-11-28)
# jkl3456 feat: add readiness probe for /health endpoint (2024-11-27)

# 특정 시점 상태 확인
git show abc1234:k8s/deployment.yaml

# 누가 무슨 이유로 변경했는지 PR 링크까지
git log --format="%H %ae %s" k8s/deployment.yaml
```

이것이 Git이 Audit Log를 대체하는 방식이다. 별도의 배포 이력 시스템 없이 `git log`가 모든 이력을 제공한다.

**원칙 3: Pulled Automatically (자동화된 Pull)**

변경이 Git에 반영되면 자동으로 시스템에 적용된다. 사람이 명령어를 실행하지 않는다.

```
Push 모델 (CI/CD가 직접 배포):
  CI 서버 → kubectl apply → Kubernetes 클러스터
  
  문제: CI 서버가 Kubernetes 클러스터에 직접 접근 권한 필요
    → kubeconfig, 서비스 어카운트 토큰을 CI 서버에 저장
    → CI 서버 침해 시 클러스터 전체 접근 가능

Pull 모델 (ArgoCD가 Git을 감시):
  Git 저장소 ← ArgoCD가 주기적으로 폴링 (또는 Webhook)
  ArgoCD → kubectl apply → Kubernetes 클러스터 (클러스터 내부에서)
  
  이점: 
    - CI 서버는 Git에만 접근하면 됨
    - 클러스터 자격증명이 외부에 노출되지 않음
    - ArgoCD는 클러스터 내부에서 실행 (클러스터 접근이 자연스러움)
```

**원칙 4: Continuously Reconciled (지속적 수렴)**

시스템은 "원하는 상태(Git)"와 "현재 상태(Kubernetes)"를 지속적으로 비교하고, 차이가 있으면 자동으로 수정한다.

```
Reconciliation Loop (ArgoCD 기준):

  매 3분 (또는 Git Webhook 즉시):
    1. Git에서 k8s/ 디렉토리의 모든 YAML 읽기
    2. Kubernetes 현재 상태 조회 (kubectl get)
    3. 차이 감지 (Diff)
       - OutOfSync: Git과 다름 → 동기화 필요
       - Synced: 동일 → 조치 없음

  Self-Heal 활성화 시:
    OutOfSync 감지 → 즉시 Git 상태로 복원 (kubectl apply)
    
  Self-Heal 비활성화 시:
    OutOfSync 감지 → 알림만 발생, 수동 Sync 필요
```

---

### 명령형 vs 선언형 배포 비교

```
실제 운영 시나리오: 장애 발생 → 원인 추적

명령형 배포 후 장애:
  "어제 배포된 버전이 뭔가요?"
  → kubectl get deployment myapp -o yaml | grep image
  → "main-abc1234"라는 태그는 알겠는데...
  → "이 배포가 언제 실행됐나요?" → Slack 검색, 이메일 검색
  → "누가 승인했나요?" → 구두로 물어봄
  → 추적에 30분 소요

GitOps 배포 후 장애:
  "어제 배포된 버전이 뭔가요?"
  → git log k8s/deployment.yaml → PR #234 확인
  → "누가 리뷰했나요?" → PR 리뷰어 목록
  → "왜 배포됐나요?" → PR 설명, 연결된 Issue
  → "이전 버전으로 롤백" → git revert → 자동 배포
  → 추적에 5분 소요
```

---

### GitOps 저장소 구조 — App 코드와 설정 분리

```
# 권장 구조: 두 개의 저장소

저장소 1: 애플리케이션 코드 (myapp)
  myapp/
  ├── src/
  ├── build.gradle.kts
  └── Dockerfile
  # CI 파이프라인: 코드 변경 → 이미지 빌드 → 이미지 태그 업데이트

저장소 2: Kubernetes 설정 (myapp-config)
  myapp-config/
  ├── base/
  │   ├── deployment.yaml
  │   ├── service.yaml
  │   └── kustomization.yaml
  └── overlays/
      ├── dev/
      │   ├── kustomization.yaml
      │   └── replica-patch.yaml
      ├── staging/
      │   └── kustomization.yaml
      └── production/
          ├── kustomization.yaml
          └── resource-patch.yaml

# CI 파이프라인이 이미지 태그 업데이트
# .github/workflows/cd.yml (myapp 저장소)
- name: 이미지 태그 업데이트
  run: |
    cd ../myapp-config
    yq e '.images[0].newTag = "${{ github.sha }}"' \
      -i overlays/production/kustomization.yaml
    git commit -am "deploy: update myapp to ${{ github.sha }}"
    git push
# ArgoCD가 myapp-config 저장소를 감시 → 자동 배포
```

두 저장소로 분리하는 이유:
- 애플리케이션 코드 변경과 인프라 설정 변경의 히스토리 분리
- 인프라 설정 저장소에 더 엄격한 접근 제어 적용 가능
- 설정만 변경 시 코드 CI 없이 바로 배포 가능

---

## 💻 실전 실험

### 실험 1: Self-Heal 동작 확인

```bash
# 1. ArgoCD Application에서 Self-Heal 활성화 확인
argocd app get myapp | grep "Auto Sync"
# Auto Sync:   Enabled (Prune=false, SelfHeal=true)

# 2. Kubernetes에서 직접 수정 (GitOps 원칙 위반)
kubectl scale deployment myapp --replicas=10

# 3. ArgoCD의 반응 관찰 (3분 내)
kubectl get deployment myapp -w
# NAME    READY   UP-TO-DATE   AVAILABLE
# myapp   10/10   10           10        ← 직접 수정 후
# myapp   3/3     3            3         ← ArgoCD가 Git 상태(replicas:3)로 복원

# 4. ArgoCD 이벤트 확인
argocd app get myapp | grep "Events"
# → "SelfHeal: Synced to HEAD" 이벤트 확인
```

### 실험 2: GitOps로 롤백

```bash
# 이전 커밋으로 롤백 (git revert)
git log --oneline k8s/deployment.yaml
# abc1234 deploy: update myapp to main-abc1234 (broken)
# def5678 deploy: update myapp to main-def5678 (stable)

# 방법 1: git revert (커밋 히스토리 보존)
git revert abc1234
git push origin main
# → ArgoCD가 감지 → 자동으로 def5678 버전 배포

# 방법 2: 특정 커밋으로 직접 복원
git checkout def5678 -- k8s/deployment.yaml
git commit -m "rollback: revert to def5678 due to OOM"
git push origin main
# → ArgoCD 자동 적용

# 방법 3: ArgoCD에서 직접 이전 상태로 Sync
argocd app rollback myapp def5678
```

### 실험 3: 로컬에서 ArgoCD 실험 환경 구성

```yaml
# docker-compose.yml (간소화 버전)
version: '3.8'
services:
  gitea:
    image: gitea/gitea:latest
    ports:
      - "3000:3000"
    environment:
      - GITEA_APP_NAME=Local GitOps
    volumes:
      - gitea-data:/data

volumes:
  gitea-data:
```

```bash
# Kind 클러스터에 ArgoCD 설치
kind create cluster --name gitops-lab

kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD UI 접근
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 초기 비밀번호 확인
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d

# ArgoCD Application 생성 (로컬 Gitea 연동)
argocd app create myapp \
  --repo http://gitea:3000/myorg/myapp-config.git \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal
```

---

## 📊 성능/비용 비교

| 항목 | 명령형 배포 | GitOps (ArgoCD) |
|------|-----------|----------------|
| 배포 추적 | 별도 시스템 필요 | git log로 즉시 |
| 롤백 | 수동 명령어 실행 | git revert + 자동 배포 |
| 환경 드리프트 감지 | 수동 점검 | 자동 (3분 주기) |
| 보안 (자격증명) | CI 서버에 kubeconfig 저장 | ArgoCD만 클러스터 접근 |
| 감사 추적 | 별도 로그 시스템 | Git 커밋 히스토리 |
| 신규 환경 구성 | 문서 보고 수동 | Git 저장소 복사 → 자동 적용 |

---

## ⚖️ 트레이드오프

**GitOps의 제약 — "모든 변경은 Git을 통해"**

긴급 장애 상황에서 `kubectl edit`으로 즉시 수정하고 싶어도, GitOps에서는 PR을 만들고 머지해야 한다. 긴급 상황을 위한 "Break Glass" 절차 (Self-Heal을 일시 비활성화 → 직접 수정 → Git 동기화 → Self-Heal 재활성화)가 필요하다.

**이미지 태그 자동 업데이트의 복잡성**

CI 파이프라인이 이미지를 빌드하고 설정 저장소의 태그를 자동으로 업데이트해야 한다. 이 과정이 실패하면 배포가 멈춘다. 두 저장소 간의 조율이 추가적인 복잡도를 만든다.

**Reconciliation 지연**

기본 3분 폴링 주기로 인해 Git push 후 최대 3분 지연이 발생한다. Webhook을 설정하면 즉시 반응하지만, 설정과 유지 관리가 필요하다.

---

## 📌 핵심 정리

- GitOps는 단순 배포 자동화가 아닌 **운영 모델** — Git이 시스템의 Single Source of Truth
- **4원칙**: Declarative(선언적) / Versioned(버전화) / Automated(자동화) / Continuously Reconciled(지속적 수렴)
- **Pull 모델**: ArgoCD가 Git을 감시해 적용 — CI 서버가 클러스터에 직접 접근하지 않아 보안상 유리
- **Self-Heal**: kubectl edit으로 직접 수정해도 자동으로 Git 상태로 복원
- Git 커밋 히스토리가 배포 Audit Log를 대체 — 누가, 언제, 왜 배포했는지 PR로 추적
- 앱 코드 저장소와 설정 저장소를 분리하는 것이 대규모 운영에서 권장

---

## 🤔 생각해볼 문제

**Q1.** GitOps를 도입했는데 팀원이 ArgoCD UI에서 수동으로 Sync를 자주 실행한다. 이것은 GitOps 원칙에 어긋나는가?

<details>
<summary>해설 보기</summary>

어긋나지 않는다. GitOps의 핵심은 "Git이 상태의 원천"이라는 것이지 "자동 Sync가 필수"라는 것이 아니다. Manual Sync는 Git에 정의된 상태를 클러스터에 적용하는 것이므로 선언형 원칙에 위배되지 않는다. 

오히려 자동 Sync(Automated + Self-Heal)는 팀이 준비됐을 때 점진적으로 도입하는 것이 안전하다. Git에 잘못된 YAML이 올라가면 자동으로 프로덕션에 적용되기 때문이다. 처음에는 Manual Sync로 시작해 파이프라인에 대한 신뢰가 쌓이면 자동화한다.

</details>

**Q2.** 동일한 설정 저장소를 dev/staging/prod 세 환경이 공유할 때 발생할 수 있는 문제는?

<details>
<summary>해설 보기</summary>

세 환경이 하나의 저장소를 공유하는 것은 괜찮지만, 브랜치 전략이나 디렉토리 구조로 환경을 분리해야 한다.

문제 시나리오: `overlays/dev/`를 수정하다가 실수로 `overlays/production/`을 함께 커밋하는 경우. 또는 staging에서 테스트 중인 설정이 production에 적용되는 경우.

해결책: Kustomize overlay로 각 환경 설정을 별도 디렉토리에 관리하고, ArgoCD Application을 환경별로 분리해서 각각 다른 경로를 바라보게 한다. production 설정 변경에는 추가 승인자 설정 (Branch Protection Rule)을 적용한다.

</details>

**Q3.** ArgoCD의 Reconciliation Loop가 3분마다 실행된다면, 긴급 배포 시 최대 3분을 기다려야 하는가?

<details>
<summary>해설 보기</summary>

Webhook을 설정하면 Git push 즉시 ArgoCD가 반응한다. GitHub/GitLab/Gitea에서 ArgoCD의 Webhook 엔드포인트(`https://argocd.mycompany.com/api/webhook`)로 push 이벤트를 전송하도록 설정하면 된다.

또한 ArgoCD CLI나 UI에서 수동으로 즉시 Sync를 트리거할 수도 있다:
```bash
argocd app sync myapp
```

3분 폴링은 Webhook이 실패했을 때의 보험이고, 실제 운영에서는 Webhook으로 즉시 반응한다.

</details>

---

<div align="center">

**[⬅️ 이전: Pipeline 설계 원칙 — Fast Fail과 병렬 실행](./04-pipeline-design-principles.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — Workflow YAML 구조 분해 ➡️](../github-actions/01-workflow-yaml-internals.md)**

</div>
