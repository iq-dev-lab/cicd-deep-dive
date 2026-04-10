# ArgoCD 아키텍처 — Reconciliation Loop 내부

## 🎯 핵심 질문

- ArgoCD는 내부적으로 어떻게 Git 저장소와 클러스터 상태를 비교하는가?
- 3분마다 폴링하는데, Webhook으로 즉시 배포할 수 있는 이유는?
- `OutOfSync` vs `Synced` 상태는 정확히 어떻게 판단하는가?
- ArgoCD의 각 컴포넌트(Application Controller, Repo Server, API Server)가 하는 일은?

## 🔍 왜 이 개념이 실무에서 중요한가

ArgoCD를 설치했지만, "왜 배포가 3분 걸리지?", "Webhook이 뭐더라?", "OutOfSync 상태가 자꾸 뜨는데 왜?"라는 질문을 자주 받습니다. 아키텍처를 이해하면:

1. **성능 최적화**: 폴링 주기 조정, Webhook 설정, 빠른 배포
2. **트러블슈팅**: Repo Server 렌더링 실패, API 지연, 상태 불일치 원인 파악
3. **용량 계획**: 많은 Application을 관리할 때 리소스 할당
4. **보안**: 각 컴포넌트의 권한 최소화

## 😱 흔한 실수 (Before — 아키텍처를 모를 때의 접근)

### Before: 아키텍처 모르고 운영하면...

```bash
# "왜 배포가 안 되지?"
$ argocd app sync myapp
# 기다리는데... 5분 후 성공
# 실제 원인? Repo Server가 과부하

# "Git 변경이 감지 안 되는데?"
# Webhook 설정 없이 3분 폴링만 사용
# 매번 최대 3분 기다려야 함

# "OutOfSync인데 sync를 눌러도 안 고쳐져?"
$ argocd app sync myapp
# Application Controller가 리소스 부족해서 느림
# 실제 상태 파악 어려움

# "Secret 렌더링 오류가 뜨는데?"
# Repo Server의 로그를 안 보고 추측만 함
$ argocd app get myapp
# OutOfSync
# Repo server error: secret not found
# → 원인 파악 안 됨

# "갑자기 많은 앱을 추가하려는데 성능이 떨어짐"
# API Server의 메모리 부족, ETCD 성능 저하
# 사전 계획 없음
```

## ✨ 올바른 접근 (After — 아키텍처를 알고 난 설계/운영)

### After: 아키텍처 이해 후 최적화

```bash
# 1. Webhook으로 즉시 배포
# Git 변경 → 자동 배포 (3분 X, 수초 O)

# 2. 각 컴포넌트 모니터링
$ kubectl logs -n argocd deployment/argocd-repo-server | grep -E "render|error"
$ kubectl logs -n argocd deployment/argocd-application-controller

# 3. OutOfSync 상태 상세 파악
$ argocd app get myapp
$ argocd app diff myapp
# 정확히 뭐가 다른지 확인

# 4. Repo Server 성능 최적화
# Kustomize/Helm 캐싱 설정, 폴링 주기 조정

# 5. 사전 용량 계획
# Application 개수, 리소스 제한 설정
```

## 🔬 내부 동작 원리

### ArgoCD 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────┐
│                     ArgoCD (K8s 클러스터)                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Application Controller                      │  │
│  │  (Git과 Cluster 상태 비교, 동기화 결정)               │  │
│  │  - Reconciliation Loop (주기: 3분)                    │  │
│  │  - OutOfSync/Synced 판단                              │  │
│  │  - Sync 작업 실행                                     │  │
│  └──────────────────┬─────────────────────────────────┘  │
│                     │                                      │
│  ┌──────────────────┼─────────────────────────────────┐  │
│  │  Repo Server     │  (Git YAML 렌더링)               │  │
│  │  - Git에서 YAML  │                                  │  │
│  │  - Kustomize,    ├──→ 렌더링된 YAML 반환           │  │
│  │    Helm 렌더링   │  - 캐싱                         │  │
│  └──────────────────────────────────────────────────┘  │
│                     │                                      │
│  ┌──────────────────┼─────────────────────────────────┐  │
│  │  API Server      │                                  │  │
│  │  ├─ CLI 요청 처리 ├──→ Application 상태 조회/수정  │  │
│  │  ├─ UI 요청 처리 │                                 │  │
│  │  └─ 인증/인가    │                                 │  │
│  └──────────────────────────────────────────────────┘  │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              내부 ETCD (Application 메타데이터)        │  │
│  │              - Application 정의                       │  │
│  │              - Sync 히스토리                          │  │
│  │              - 현재 상태 캐시                         │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
              ↓                          ↓
        ┌─────────────┐           ┌──────────────┐
        │ Git         │           │ Kubernetes   │
        │ Repository  │           │ API Server   │
        │             │           │              │
        │ ← 폴링/     │           │ ← 배포       │
        │   Webhook   │           │   (kubectl   │
        │             │           │   apply)     │
        └─────────────┘           └──────────────┘
```

### 1. Application Controller — Reconciliation Loop

**정의**: Git의 선언형 상태와 클러스터의 실제 상태를 주기적으로 비교해서 일치시키는 핵심 로직

```yaml
# Application 리소스 (ArgoCD가 감시하는 객체)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/app-config  # ← Git 저장소
    targetRevision: main
    path: config/prod
  destination:
    server: https://kubernetes.default.svc  # ← 타겟 클러스터
    namespace: production
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

**Reconciliation Loop 작동 과정** (3분마다 또는 Webhook 트리거 시):

```
1. Repo Server에게 요청: "config/prod의 현재 Git 상태는?"
   ↓
2. Repo Server가 Git 클론, Kustomize/Helm 렌더링
   (기본 YAML 반환)
   ↓
3. Kubernetes API 서버에 질의: "클러스터에 실제로 배포된 리소스는?"
   (kubectl get 동일한 작업)
   ↓
4. Git 상태 vs 클러스터 상태 Diff 계산
   (3-way merge: base, local, remote)
   ↓
5. 판단: 같음? → Synced / 다름? → OutOfSync
   ↓
6. AutoSync 활성화? → 자동 배포
   Manual Sync? → 대기 (수동 배포 대기)
   ↓
7. 배포 실행 (kubectl apply):
   a) Sync Wave 순서대로 리소스 생성/업데이트
   b) 상태 모니터링 (Pod ready, Service IP 할당 등)
   c) 완료 → Synced 상태로 전환
```

**코드 레벨로 이해하는 Reconciliation:**

```bash
# ArgoCD가 내부적으로 하는 작업들을 시뮬레이션

# 1. Git의 현재 상태 가져오기
$ git clone https://github.com/company/app-config /tmp/app-config
$ cd /tmp/app-config && git checkout main
$ kustomize build config/prod > /tmp/desired-state.yaml

# 2. 클러스터의 실제 상태 가져오기
$ kubectl get -n production deployment,service,configmap -o yaml > /tmp/actual-state.yaml

# 3. Diff 계산 (ArgoCD의 Application Controller가 이를 수행)
$ diff <(yq eval 'sort_by(.metadata.name)' /tmp/desired-state.yaml) \
       <(yq eval 'sort_by(.metadata.name)' /tmp/actual-state.yaml)
# 차이 있음 → OutOfSync

# 4. 자동 Sync 활성화 시 배포
$ kubectl apply -f /tmp/desired-state.yaml
# 또는 더 정교하게
$ kubectl diff -f /tmp/desired-state.yaml
$ kubectl apply -f /tmp/desired-state.yaml
```

### 2. Repo Server — Git 렌더링 엔진

**역할**: Git 저장소에서 YAML을 가져와서 실제 배포될 매니페스트로 변환

```yaml
# 저장소 구조
app-config/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-resources.yaml
    └── prod/
        ├── kustomization.yaml
        └── patch-resources.yaml

# dev overlay
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1  # dev는 1 replica
resources:
- configmap-dev.yaml

# prod overlay
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5  # prod는 5 replica
resources:
- configmap-prod.yaml
```

**Repo Server 렌더링 과정:**

```bash
# ArgoCD의 Repo Server가 하는 작업

# 1. Git 리포지토리 클론
$ git clone https://github.com/company/app-config /var/lib/argocd/repo/abc123

# 2. targetRevision 체크아웃
$ cd /var/lib/argocd/repo/abc123 && git checkout main

# 3. Path 기반 렌더링
$ cd /var/lib/argocd/repo/abc123/config/prod

# 4. Kustomize 또는 Helm 렌더링
$ kustomize build .
# 또는
$ helm template myapp ./chart --values values-prod.yaml

# 5. 결과물 (최종 YAML)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: production
spec:
  replicas: 5  # prod 설정 반영됨
  selector:
    matchLabels:
      app: app
  template:
    ...
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-prod
  namespace: production
data:
  env: production
  debug: "false"
```

**Repo Server 성능 최적화:**

```yaml
# ArgoCD ConfigMap (argocd-cm)에서 Repo Server 설정
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Kustomize 빌드 캐싱 (기본값: 1 시간)
  reposerver.autoreload: "true"
  
  # 렌더링 타임아웃 (기본값: 30초)
  server.insecure: "false"
  
  # HELM_EXPERIMENTAL_OCI를 활성화 (OCI 레지스트리 지원)
  helm.valuesFileSchemes: file,http
```

### 3. API Server — 외부 요청 처리

**역할**: CLI, UI, Webhook 요청을 받아서 Application 상태 조회/수정

```bash
# API Server가 처리하는 요청들

# 1. CLI 요청 (gRPC)
$ argocd app get myapp
# → API Server가 ETCD에서 Application 메타데이터 조회
# → Application Controller의 현재 상태 반환

# 2. UI 요청 (REST API)
$ curl https://localhost:8080/api/v1/applications/myapp
# → 동일하게 처리

# 3. Sync 요청
$ argocd app sync myapp
# → API Server가 Sync 작업을 ETCD에 큐잉
# → Application Controller가 처리

# 4. Webhook (GitHub/GitLab)
# GitHub가 POST 요청: https://argocd.example.com/api/webhook
# → API Server가 변경된 브랜치/경로 파악
# → 해당 Application 찾아서 즉시 Reconciliation 트리거
```

**RBAC (역할 기반 접근 제어):**

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly  # 기본 역할: 읽기 전용
  policy.csv: |
    # p, 주체, 리소스, 작업, 객체
    p, role:admin, *, *, *  # 관리자: 모든 작업
    p, role:readonly, *, get, *  # 읽기 전용: get만
    
    # g, 사용자, 역할
    g, alice@company.com, role:admin
    g, bob@company.com, role:readonly
    
    # 프로젝트별 권한
    p, dev-team, applications, *, myapp-dev
    p, prod-team, applications, *, myapp-prod
    g, alice@company.com, dev-team
    g, charlie@company.com, prod-team
```

### 4. Reconciliation Loop — 상태 변화

```
Application 상태 머신:

┌─────────────┐
│   Unknown   │  (처음 생성 또는 에러 복구)
└──────┬──────┘
       │
       ↓
┌─────────────────┐
│  Progressing    │  (배포 진행 중)
│  - Resource     │
│    생성/업데이트 │
│  - 상태 모니터링 │
└──────┬──────────┘
       │
    ┌──┴──┐
    │     │
    ↓     ↓
┌────┐ ┌────────┐
│Sync│ │ Error  │  (배포 실패)
│ed  │ │Health  │
└────┘ │Degraded
       └────────┘
    (재시도 또는
     수동 개입)
```

### 5. OutOfSync vs Synced 판단 기준

```bash
# 판단 알고리즘 (3-way merge)

# 1. Base 상태 (Git에서 첫 배포 시)
base_state=$(git show <deploy-commit>:config/prod)

# 2. Desired 상태 (현재 Git)
desired_state=$(git show HEAD:config/prod)

# 3. Actual 상태 (클러스터)
actual_state=$(kubectl get -o yaml)

# 4. 3-way Diff
if desired_state == actual_state:
    status = "Synced"
else:
    status = "OutOfSync"
    
# 5. 자세한 Diff 정보 (어떤 부분이 다른지)
# - spec.replicas: 3 (desired) vs 5 (actual)
# - spec.template.spec.containers[0].image: v1.2.3 vs v1.2.0
```

**구체적 예:**

```yaml
# Git의 desired 상태
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - image: myimage:v1.2.3

---
# 클러스터의 actual 상태
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 5  # ← 다름!
  template:
    spec:
      containers:
      - image: myimage:v1.2.0  # ← 다름!
```

```bash
$ argocd app diff myapp
deployment.apps/app
  spec.replicas: 3 vs 5
  spec.template.spec.containers[0].image: myimage:v1.2.3 vs myimage:v1.2.0

→ OutOfSync 상태!
```

## 💻 실전 실험

### 실험 1: Reconciliation Loop 직접 관찰

```bash
# 1. Application 생성
$ argocd app create myapp \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# 2. Application Controller 로그 실시간 보기
$ kubectl logs -f -n argocd deployment/argocd-application-controller
# 어떤 애플리케이션을 reconcile하는지, 몇 초 걸리는지 보임

2026-04-10T15:00:00Z info ... Processing 'myapp'
2026-04-10T15:00:01Z debug ... Fetching latest git state
2026-04-10T15:00:02Z debug ... Comparing with cluster state
2026-04-10T15:00:03Z info ... Sync required: differences detected
2026-04-10T15:00:05Z info ... Reconciliation complete

# 3. Reconciliation 주기 확인 (기본값: 180초)
$ kubectl get configmap -n argocd argocd-cmd-params-cm -o yaml | grep timeout
  application.instanceLabelKey: argocd.argoproj.io/instance
  # application.autoSync 항목은 없음 (개별 Application에서 설정)

# 4. Reconciliation 주기 커스터마이징
$ kubectl patch configmap argocd-cmd-params-cm -n argocd -p '
{
  "data": {
    "controller.operation.processors": "10",  # 동시 처리 개수
    "controller.selfheal.enabled": "true",
    "server.insecure": "true"
  }
}'
```

### 실험 2: Webhook으로 즉시 배포

```bash
# 1. GitHub Webhook 설정
# GitHub 저장소 → Settings → Webhooks → Add webhook
# - Payload URL: https://argocd.example.com/api/webhook
# - Content type: application/json
# - Events: push 이벤트만 선택

# 2. ArgoCD에 GitHub 저장소 등록
$ argocd repo add https://github.com/company/app-config \
  --username alice \
  --password <github-token>

# 3. Application 생성
$ argocd app create myapp-webhook \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# 4. Webhook 설정 확인
$ kubectl get application -n argocd myapp-webhook -o yaml
# webhook 섹션 확인

# 5. Git에 push
$ echo "# Updated at $(date)" >> README.md
$ git add . && git commit -m "Update config"
$ git push origin main

# 6. ArgoCD API 서버 로그 확인 (Webhook 수신)
$ kubectl logs -f -n argocd deployment/argocd-server | grep webhook
webhook=[payload matched]

# 7. Application 상태 확인
$ argocd app get myapp-webhook
# 수초 내에 Synced 상태로 변함 (3분 폴링이 아님)
```

### 실험 3: Repo Server 렌더링 성능 분석

```bash
# 1. Repo Server 로그 레벨 상향
$ kubectl set env deployment/argocd-repo-server \
  -n argocd \
  ARGOCD_LOG_LEVEL=debug

# 2. 렌더링 시간 측정
$ kubectl logs -n argocd deployment/argocd-repo-server | grep -E "render|completed"

2026-04-10T15:00:00Z debug ... rendering kustomization config/prod
2026-04-10T15:00:02Z debug ... rendering completed (2.1s)

# 3. 여러 Application이 동시에 Reconcile되면
# Repo Server가 경합할 수 있음

# 4. Repo Server 복제본 추가
$ kubectl scale deployment argocd-repo-server -n argocd --replicas=3

# 5. 렌더링 캐시 확인
$ kubectl exec -it deployment/argocd-repo-server -n argocd -- \
  ls -la /tmp/repo
# 캐시된 저장소 리스트

# 6. 렌더링 타임아웃 조정
$ kubectl patch configmap argocd-cmd-params-cm -n argocd -p '{
  "data": {
    "repo.server.timeout.seconds": "60"  # 기본값: 30초
  }
}'
```

### 실험 4: OutOfSync 감지 및 자동 복구

```bash
# 1. Application 생성
$ argocd app create myapp \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --auto-prune \
  --self-heal

# 2. 현재 상태 (Synced)
$ argocd app get myapp | grep "Status"
Status: Synced

# 3. 클러스터에서 직접 수정 (드리프트 발생)
$ kubectl set image deployment/app app=myimage:emergency -n production

# 4. Application Controller가 폴링 중...
# 3분 대기 또는 수동으로 refresh
$ argocd app wait myapp --timeout=10 --operation
# 또는
$ argocd app get myapp --refresh
Status: OutOfSync

# 5. 자동 복구 (Self-Heal 활성화 시)
# 대기 또는 명시적 sync
$ argocd app sync myapp

# 6. 로그로 복구 과정 확인
$ kubectl logs -n argocd deployment/argocd-application-controller | \
  grep -A5 "myapp" | tail -20

2026-04-10T15:05:00Z info ... Reconciling myapp
2026-04-10T15:05:01Z info ... Detected OutOfSync: image mismatch
2026-04-10T15:05:02Z info ... Applying Git state
2026-04-10T15:05:03Z info ... Deployment myapp updated (image: myimage:v1.2.3)
2026-04-10T15:05:04Z info ... Synced
```

## 📊 성능/비용 비교

| 항목 | 폴링 (3분) | Webhook | 성능 |
|------|----------|---------|-----|
| **배포 지연** | 최대 180초 | 수초 | Webhook 5~10배 빠름 |
| **API 요청** | 3분마다 모든 앱 Reconcile | 변경 시만 | Webhook이 90% 감소 |
| **비용** | Repo Server/Controller 계속 운영 | 동일 | Webhook이 조금 효율적 |
| **복잡도** | 간단 (설정 불필요) | Webhook 설정 필요 | 폴링이 간단 |
| **신뢰성** | Git 변경 확실히 감지 | Webhook 실패 가능 | 폴링이 안전 |

**권장사항**: 폴링 + Webhook 동시 사용 (Webhook 실패 시 폴링이 백업)

## ⚖️ 트레이드오프

| 항목 | 선택지 | 트레이드오프 |
|------|--------|-----------|
| **Reconciliation 주기** | 짧음 (30초) | 높은 리소스 사용 |
| **Reconciliation 주기** | 길음 (10분) | 배포 지연, 드리프트 감지 늦음 |
| **Repo Server 복제본** | 많음 (5개) | 높은 비용 |
| **Repo Server 복제본** | 적음 (1개) | 렌더링 병목 |
| **Self-Heal** | 활성화 | kubectl edit 직접 수정 불가 |
| **Self-Heal** | 비활성화 | 드리프트 누적 |
| **Webhook** | 활성화 | 설정 복잡, 외부 요청 허용 |
| **Webhook** | 비활성화 | 폴링만 의존, 배포 지연 |

## 📌 핵심 정리

1. **Application Controller**: Git과 클러스터 상태를 주기적으로 비교, OutOfSync/Synced 판단, 자동 배포
2. **Repo Server**: Git에서 YAML을 가져와 Kustomize/Helm으로 렌더링, 성능 병목 가능성
3. **API Server**: CLI/UI/Webhook 요청 처리, RBAC 적용, Application 메타데이터 관리
4. **Reconciliation Loop**: 3분마다 또는 Webhook으로 즉시 실행, 상태 동기화
5. **3-Way Diff**: base(초기) vs desired(Git) vs actual(cluster) 비교
6. **자동 복구**: Self-Heal 활성화 시 kubectl edit도 Git 상태로 자동 복원
7. **성능 최적화**: Webhook 설정, Repo Server 스케일, 캐싱 활용

## 🤔 생각해볼 문제

### Q1. 만약 Repo Server가 계속 느리면 어떻게 할까?

<details>
<summary>💡 해설</summary>

**진단:**
```bash
# 1. Repo Server 로그 확인
$ kubectl logs -n argocd deployment/argocd-repo-server | grep -i "slow\|timeout\|error"

# 2. Repo Server 리소스 사용 확인
$ kubectl top pod -n argocd -l app.kubernetes.io/name=argocd-repo-server
NAME                              CPU    MEMORY
argocd-repo-server-abc123-def45   900m   512Mi  # CPU 과부하

# 3. Git 작업이 느린지 확인
$ time git clone https://github.com/company/app-config /tmp/test
# 네트워크 지연? SSH vs HTTPS?

# 4. 렌더링이 느린지 확인
$ time kustomize build config/prod
# 복잡한 오버레이? 너무 많은 리소스?

# 5. Helm 렌더링이 느린지 확인
$ time helm template myapp ./chart --values values-prod.yaml
# 복잡한 템플릿? 많은 서브차트?
```

**해결책:**
```bash
# 1. Repo Server 복제본 추가
$ kubectl scale deployment argocd-repo-server -n argocd --replicas=3

# 2. 리소스 요청/제한 증가
$ kubectl set resources deployment argocd-repo-server -n argocd \
  --requests=cpu=500m,memory=512Mi \
  --limits=cpu=2000m,memory=2Gi

# 3. Git 저장소 최적화
# - SSH로 변경 (HTTPS보다 빠를 수 있음)
# - Shallow clone 설정
# - 큰 파일은 Git LFS로 외부 저장

# 4. Kustomize 렌더링 최적화
# - 불필요한 패치 제거
# - 복잡한 로직을 ConfigMap/Secret으로 분리

# 5. Helm 렌더링 최적화
# - 서브차트 최소화
# - values.yaml 단순화
```

</details>

### Q2. OutOfSync 상태가 계속 떴다가 자동으로 Synced가 될 수 있나?

<details>
<summary>💡 해설</summary>

**가능한 시나리오:**

```bash
# 1. Application 생성, 초기에는 OutOfSync (Git 상태를 아직 클러스터에 배포 안 함)
$ argocd app create myapp
Status: OutOfSync

# 2. 몇 초 후 Application Controller가 자동 배포 (AutoSync 활성화 시)
$ argocd app get myapp
Status: Synced

# 3. 또는 리소스 생성 시간이 오래 걸리면
# Progressing → Synced로 변함
```

**예시:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
  # ← AutoSync 활성화
```

```bash
# 시간별 상태 변화
$ for i in {1..5}; do
  echo "Check $i:"
  argocd app get myapp | grep "Status"
  sleep 5
done

Check 1: Status: OutOfSync
Check 2: Status: Progressing  # 배포 중
Check 3: Status: Progressing
Check 4: Status: Synced  # 완료!
Check 5: Status: Synced
```

</details>

### Q3. Repo Server와 Application Controller가 다른 버전이면?

<details>
<summary>💡 해설</summary>

**문제:** ArgoCD를 업그레이드할 때 모든 컴포넌트의 버전이 맞지 않을 수 있음

```bash
# 현재 버전 확인
$ kubectl get deployment -n argocd -o wide
NAME                            READY   VERSION
argocd-application-controller   1/1     v2.8.0
argocd-repo-server              1/1     v2.7.0  # ← 다른 버전!
argocd-api-server               1/1     v2.8.0

# 문제:
# - gRPC 프로토콜 불일치
# - API 메시지 형식 차이
# - 예상치 못한 동작
```

**해결책:**
```bash
# 1. ArgoCD 완전히 재설치 (권장)
$ kubectl delete namespace argocd
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.0/manifests/install.yaml

# 2. 또는 버전 일치하도록 패치
$ kubectl set image deployment/argocd-repo-server -n argocd \
  argocd-repo-server=argoproj/argocd:v2.8.0

# 3. 확인
$ kubectl get deployment -n argocd -o wide
```

</details>

---

[⬅️ 이전](./01-gitops-principles.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./03-sync-strategy.md)
