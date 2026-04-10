# 멀티 클러스터 관리 — ApplicationSet과 환경별 프로모션

## 🎯 핵심 질문

- 여러 클러스터에 같은 앱을 배포할 때 Application 리소스를 N번 만들어야 하나?
- ApplicationSet은 Application의 템플릿 엔진인가?
- 클러스터별 설정을 어떻게 다르게 적용할까?
- dev → staging → prod로 프로모션할 때 자동화는?

## 🔍 왜 이 개념이 실무에서 중요한가

현대의 엔터프라이즈는 다중 클러스터 환경을 운영합니다:

1. **지역별 분산** (Multi-Region):
   - 미국 (us-east, us-west)
   - 유럽 (eu-west)
   - 아시아 (ap-south)

2. **환경별 분리** (Multi-Environment):
   - dev (개발자용 1개)
   - staging (QA용 1개)
   - prod (실서비스 3개 이상)

3. **클러스터별 특성** (Per-Cluster):
   - 다른 네트워크 정책
   - 다른 리소스 할당
   - 다른 Ingress 도메인

**문제:**
- 10개 클러스터 × 5개 애플리케이션 = 50개 Application 리소스 수동 관리
- 새 클러스터 추가 → 5개 Application 다시 생성 (실수 가능)
- 설정 변경 → 모든 클러스터에 수동으로 적용

**해결:**
- **ApplicationSet**: 조건에 따라 자동으로 Application 생성
- **Generator**: 클러스터 목록, Git 브랜치, 환경 변수로 자동 생성

## 😱 흔한 실수 (Before — ApplicationSet을 모를 때의 접근)

### Before: Application을 클러스터마다 수동으로 생성

```bash
# ❌ 나쁜 패턴: 수동으로 각 클러스터마다 Application 생성

# 1. Dev 클러스터용
$ argocd app create myapp-dev \
  --repo https://github.com/company/app-config \
  --path config/dev \
  --dest-server https://dev-k8s.example.com \
  --dest-namespace default

# 2. Staging 클러스터용 (복사해서 조금 수정)
$ argocd app create myapp-staging \
  --repo https://github.com/company/app-config \
  --path config/staging \
  --dest-server https://staging-k8s.example.com \
  --dest-namespace default

# 3. Prod 클러스터 1
$ argocd app create myapp-prod-us \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://prod-us-k8s.example.com \
  --dest-namespace default

# 4. Prod 클러스터 2
$ argocd app create myapp-prod-eu \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://prod-eu-k8s.example.com \
  --dest-namespace default

# ... 총 10개 클러스터 × 20개 앱 = 200개 Application 수동 생성!

# 문제:
# - 실수로 replica 수를 다르게 설정
# - 새 클러스터 추가 시 20개 Application 다시 만들기
# - 클러스터 별칭 변경 → 모든 Application 수정
```

## ✨ 올바른 접근 (After — ApplicationSet 활용)

### After: ApplicationSet으로 자동 생성

```yaml
# ✅ ApplicationSet: 한 번의 정의로 모든 클러스터에 배포

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - name: dev
        server: https://dev-k8s.example.com
        path: config/dev
      - name: staging
        server: https://staging-k8s.example.com
        path: config/staging
      - name: prod-us
        server: https://prod-us-k8s.example.com
        path: config/prod
      - name: prod-eu
        server: https://prod-eu-k8s.example.com
        path: config/prod
  template:
    metadata:
      name: 'myapp-{{ name }}'  # dev → myapp-dev, prod-us → myapp-prod-us
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/company/app-config
        targetRevision: main
        path: '{{ path }}'  # 각 클러스터의 path로 자동 설정
      destination:
        server: '{{ server }}'  # 각 클러스터의 server로 자동 설정
        namespace: default
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
```

**결과:**
```bash
# ApplicationSet을 적용하면 자동으로 4개 Application 생성!
$ kubectl apply -f applicationset.yaml
$ argocd app list
NAME           CLUSTER                  NAMESPACE
myapp-dev      https://dev-k8s...       default
myapp-staging  https://staging-k8s...   default
myapp-prod-us  https://prod-us-k8s...   default
myapp-prod-eu  https://prod-eu-k8s...   default

# 새 클러스터 추가? → ApplicationSet에만 추가, 자동 생성됨!
```

## 🔬 내부 동작 원리

### 1. ApplicationSet의 Generator 타입

**A. List Generator — 명시적 목록**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - list:
      elements:
      - cluster: dev
        url: https://dev-k8s.example.com
      - cluster: staging
        url: https://staging-k8s.example.com
      - cluster: prod
        url: https://prod-k8s.example.com
  template:
    metadata:
      name: 'myapp-{{ cluster }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: 'config/{{ cluster }}'
      destination:
        server: '{{ url }}'
        namespace: default
```

**생성되는 Application:**
```
myapp-dev
myapp-staging
myapp-prod
```

**B. Cluster Generator — 등록된 모든 클러스터**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          argocd.argoproj.io/secret-type: cluster
          deployment: prod  # ← label selector
  template:
    metadata:
      name: 'myapp-{{ name }}'  # cluster secret의 name
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: 'config/prod'
      destination:
        server: '{{ server }}'  # cluster secret의 server URL
        namespace: default
```

**작동 방식:**
```bash
# 1. ArgoCD가 등록한 모든 클러스터를 스캔
$ kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
NAME                              TYPE   AGE
cluster-us-east                   Opaque 5d
cluster-us-west                   Opaque 5d
cluster-eu-west                   Opaque 5d
cluster-ap-south                  Opaque 5d

# 2. label="deployment: prod"인 클러스터 필터링
$ kubectl get secret -n argocd \
  -l argocd.argoproj.io/secret-type=cluster,deployment=prod
NAME                              TYPE
cluster-us-east    # ← prod 클러스터만
cluster-us-west
cluster-eu-west

# 3. 각 클러스터마다 Application 자동 생성
myapp-cluster-us-east
myapp-cluster-us-west
myapp-cluster-eu-west
```

**C. Git Generator — Git의 디렉토리/브랜치**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - git:
      repoURL: https://github.com/company/app-config
      revision: main
      directories:
      - path: 'config/*'  # config/dev, config/staging, config/prod 등
  template:
    metadata:
      name: 'myapp-{{ path.basename }}'  # dev, staging, prod
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: '{{ path.path }}'  # config/dev 등
      destination:
        server: https://kubernetes.default.svc
        namespace: default
```

**작동 방식:**
```bash
# Git 저장소의 디렉토리 구조
config/
├── dev/          # ← path 매칭
│   ├── deployment.yaml
│   └── kustomization.yaml
├── staging/      # ← path 매칭
├── prod/         # ← path 매칭
└── other/        # ← 제외 (* 패턴이므로)

# 생성되는 Application:
myapp-dev      (config/dev 경로)
myapp-staging  (config/staging 경로)
myapp-prod     (config/prod 경로)
```

### 2. 클러스터별 설정 차이 (Patch)

**Cluster Generator + Patch 오버라이드:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          managed-by: argocd
  template:
    metadata:
      name: 'myapp-{{ name }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: config/prod
      destination:
        server: '{{ server }}'
        namespace: default
      syncPolicy:
        automated: {}
  
  syncPolicy:
    preserveResourcesOnDeletion: true
```

**클러스터별 커스터마이징:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - list:
      elements:
      - cluster: us-east
        region: us-east-1
        replicas: 5
      - cluster: us-west
        region: us-west-2
        replicas: 5
      - cluster: eu-west
        region: eu-west-1
        replicas: 3  # 유럽은 리소스 적게
  template:
    metadata:
      name: 'myapp-{{ cluster }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: config/prod
        helm:  # ← Helm values로 환경별 설정
          parameters:
          - name: replicaCount
            value: '{{ replicas }}'
          - name: region
            value: '{{ region }}'
      destination:
        server: '{{ server }}'
        namespace: default
```

**Kustomize Patch로 오버라이드:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          managed-by: argocd
  template:
    metadata:
      name: 'myapp-{{ name }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: config/prod
        kustomize:
          patches:  # ← 클러스터별 패치
          - target:
              kind: Deployment
            patch: |-
              - op: add
                path: /spec/template/metadata/labels/cluster
                value: {{ name }}
      destination:
        server: '{{ server }}'
        namespace: default
```

### 3. 환경별 프로모션 (Progressive Delivery)

**Dev → Staging → Prod 자동 프로모션:**

```bash
# 아이디어: Git 브랜치로 환경 표현

app-config/
├── main (Prod)      # ← 프로덕션 버전
├── staging          # ← Staging 버전 (main의 한 발 뒤)
└── dev              # ← Dev 버전 (최신 실험)
```

**프로모션 워크플로우:**

```
Dev 코드 변경
  ↓
dev 브랜치에 merge
  ↓
CI/CD가 자동 테스트
  ↓
(수동 승인)
  ↓
staging 브랜치로 cherry-pick
  ↓
Staging에서 테스트
  ↓
(수동 승인)
  ↓
main 브랜치로 merge
  ↓
Production 배포!
```

**ApplicationSet으로 구현:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-promotion
spec:
  generators:
  - list:
      elements:
      - env: dev
        branch: dev
        cluster: dev-k8s.example.com
        syncPolicy: automated
      - env: staging
        branch: staging
        cluster: staging-k8s.example.com
        syncPolicy: manual
      - env: prod
        branch: main  # main 브랜치 = 프로덕션
        cluster: prod-k8s.example.com
        syncPolicy: manual
  template:
    metadata:
      name: 'myapp-{{ env }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        branch: '{{ branch }}'
        path: config
      destination:
        server: '{{ cluster }}'
        namespace: default
      syncPolicy:
        # syncPolicy 값을 동적으로 설정하는 방법은 제한적
        # 대신 Application 생성 후 수동 조정 필요
```

## 💻 실전 실험

### 실험 1: List Generator로 여러 클러스터에 배포

```bash
# 1. ArgoCD에 여러 클러스터 등록
$ argocd cluster add kind-dev --name dev
$ argocd cluster add kind-staging --name staging
$ argocd cluster add kind-prod --name prod

# 2. 클러스터 확인
$ argocd cluster list
SERVER                     NAME
https://kubernetes.default.svc  in-cluster
https://dev.../...         dev
https://staging.../...     staging
https://prod.../...        prod

# 3. ApplicationSet 생성
$ cat > applicationset.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: dev
        server: https://dev-api.example.com
        path: config/dev
      - cluster: staging
        server: https://staging-api.example.com
        path: config/staging
      - cluster: prod
        server: https://prod-api.example.com
        path: config/prod
  template:
    metadata:
      name: 'myapp-{{ cluster }}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/company/app-config
        targetRevision: main
        path: '{{ path }}'
      destination:
        server: '{{ server }}'
        namespace: default
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
EOF

# 4. ApplicationSet 적용
$ kubectl apply -f applicationset.yaml
applicationset.argoproj.io/myapp created

# 5. 자동으로 생성된 Application 확인
$ argocd app list
NAME           CLUSTER       NAMESPACE   STATUS
myapp-dev      dev           default     Synced
myapp-staging  staging       default     Synced
myapp-prod     prod          default     Synced

# 6. 각 Application의 상태 확인
$ argocd app get myapp-dev
$ argocd app get myapp-staging
$ argocd app get myapp-prod
```

### 실험 2: Cluster Generator로 자동 감지

```bash
# 1. 클러스터에 label 추가
$ kubectl patch secret cluster-dev -n argocd -p \
  '{"metadata":{"labels":{"env":"dev"}}}'

$ kubectl patch secret cluster-prod-us -n argocd -p \
  '{"metadata":{"labels":{"env":"prod","region":"us"}}}'

$ kubectl patch secret cluster-prod-eu -n argocd -p \
  '{"metadata":{"labels":{"env":"prod","region":"eu"}}}'

# 2. ApplicationSet with Cluster Generator
$ cat > applicationset-clusters.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-clusters
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: prod  # ← prod 레이블만 선택
  template:
    metadata:
      name: 'myapp-{{ name }}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/company/app-config
        targetRevision: main
        path: config/prod
      destination:
        server: '{{ server }}'
        namespace: default
      syncPolicy:
        automated:
          selfHeal: true
EOF

# 3. 적용
$ kubectl apply -f applicationset-clusters.yaml

# 4. 자동으로 생성된 Application (prod 클러스터만)
$ argocd app list
NAME                  CLUSTER          NAMESPACE
myapp-cluster-prod-us https://prod-us.../... default
myapp-cluster-prod-eu https://prod-eu.../... default
```

### 실험 3: Git Generator로 디렉토리 구조 자동 감지

```bash
# 1. Git 저장소 구조 (이미 만들었다고 가정)
# config/
# ├── dev/
# │   └── kustomization.yaml
# ├── staging/
# │   └── kustomization.yaml
# └── prod/
#     └── kustomization.yaml

# 2. ApplicationSet with Git Generator
$ cat > applicationset-git.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-git
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/company/app-config
      revision: main
      directories:
      - path: 'config/*'
  template:
    metadata:
      name: 'myapp-{{ path.basename }}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/company/app-config
        targetRevision: main
        path: '{{ path.path }}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{ path.basename }}'  # dev/staging/prod 네임스페이스
      syncPolicy:
        automated:
          selfHeal: true
EOF

# 3. 적용
$ kubectl apply -f applicationset-git.yaml

# 4. 자동으로 생성된 Application
$ argocd app list
NAME         NAMESPACE   STATUS
myapp-dev    dev         Synced
myapp-staging staging     Synced
myapp-prod   prod        Synced

# 5. 새로운 디렉토리 추가 시 자동으로 Application 추가됨
$ mkdir config/qa
$ git add config/qa/
$ git commit -m "Add QA environment"
$ git push

# ApplicationSet이 자동으로 감지해서 myapp-qa Application 생성!
$ argocd app list
NAME         NAMESPACE   STATUS
myapp-dev    dev         Synced
myapp-staging staging     Synced
myapp-prod   prod        Synced
myapp-qa     qa          Synced  # ← 새로 추가됨!
```

### 실험 4: 클러스터별 Replica 수 다르게 설정

```bash
# 1. ApplicationSet with 클러스터별 설정
$ cat > applicationset-replicas.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-replicas
spec:
  generators:
  - list:
      elements:
      - cluster: dev
        server: https://dev.../...
        replicas: 1
      - cluster: staging
        server: https://staging.../...
        replicas: 2
      - cluster: prod-us
        server: https://prod-us.../...
        replicas: 5
      - cluster: prod-eu
        server: https://prod-eu.../...
        replicas: 5
  template:
    metadata:
      name: 'myapp-{{ cluster }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        path: config/prod
        helm:
          parameters:
          - name: replicaCount
            value: '{{ replicas }}'
      destination:
        server: '{{ server }}'
        namespace: default
EOF

# 2. 적용
$ kubectl apply -f applicationset-replicas.yaml

# 3. 생성된 Application 확인
$ argocd app get myapp-dev
# Helm이 replicas=1로 렌더링됨

$ argocd app get myapp-prod-us
# Helm이 replicas=5로 렌더링됨
```

## 📊 성능/비용 비교

| 항목 | 수동 Application | ApplicationSet |
|------|-----------------|----------------|
| **생성 시간** | N개 앱 × 시간 | 1번의 ApplicationSet |
| **유지보수** | N개 수정 필요 | 1곳만 수정 |
| **새 클러스터 추가** | 애플리케이션당 수동 생성 | ApplicationSet에만 추가 |
| **실수 가능성** | 높음 (복사 붙여넣기) | 낮음 (템플릿 기반) |
| **복잡성** | 낮음 | 중간 (Generator 이해 필요) |

## ⚖️ 트레이드오프

| 선택지 | 유연성 | 복잡성 | 권장 상황 |
|--------|--------|--------|---------|
| List Generator | 중간 | 낮음 | 클러스터 수 고정 |
| Cluster Generator | 높음 | 중간 | 동적 클러스터 추가/제거 |
| Git Generator | 높음 | 중간 | Git 구조가 명확한 경우 |
| 조합 Generator | 최고 | 높음 | 복잡한 멀티 클러스터 |

## 📌 핵심 정리

1. **ApplicationSet**: 하나의 정의로 여러 클러스터에 여러 Application 자동 생성
2. **List Generator**: 명시적 목록으로 Application 생성
3. **Cluster Generator**: 등록된 클러스터 라벨로 필터링해서 자동 생성
4. **Git Generator**: Git 디렉토리/브랜치로 자동 감지
5. **클러스터별 설정**: Helm parameters 또는 Kustomize patches로 오버라이드
6. **프로모션 파이프라인**: 여러 브랜치를 ApplicationSet으로 관리해서 dev→staging→prod 자동화

## 🤔 생각해볼 문제

### Q1. ApplicationSet을 삭제하면 생성된 Application도 자동으로 삭제되나?

<details>
<summary>💡 해설</summary>

**기본 동작:**

```bash
# ApplicationSet 삭제
$ kubectl delete applicationset myapp -n argocd

# 기본적으로 생성된 Application은 유지됨!
$ argocd app list
NAME           STATUS
myapp-dev      Synced
myapp-staging  Synced
myapp-prod     Synced
# ← 여전히 존재!
```

**이유:** ApplicationSet 삭제가 의도하지 않은 리소스 삭제를 방지

**자동 삭제 활성화:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  syncPolicy:
    preserveResourcesOnDeletion: false  # ← false로 설정
  generators:
  - list:
      elements:
      - cluster: dev
        server: ...
  template:
    # ...
```

```bash
# ApplicationSet 삭제 시 생성된 Application도 함께 삭제
$ kubectl delete applicationset myapp -n argocd
applicationset.argoproj.io/myapp deleted

$ argocd app list
# myapp-dev, myapp-staging 등이 모두 삭제됨
```

**권장:**
- 기본값(`preserveResourcesOnDeletion: true`)이 안전
- 명시적으로 Application을 제거해야 할 때만 `false`로 설정

</details>

### Q2. ApplicationSet의 Generator를 여러 개 조합하면?

<details>
<summary>💡 해설</summary>

**예: 클러스터 선택 + Git 디렉토리**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-combined
spec:
  generators:
  # Generator 1: Cluster로 필터링
  - clusters:
      selector:
        matchLabels:
          env: prod
  
  # Generator 2: Git 디렉토리 선택
  - git:
      repoURL: https://github.com/company/app-config
      directories:
      - path: 'apps/*'
  
  template:
    metadata:
      # 어느 Generator의 변수를 사용?
      name: 'app-{{ path.basename }}-{{ name }}'
    spec:
      # ...
```

**조합 방식:**

```
Generator 1 결과: cluster 변수 (name, server 등)
Generator 2 결과: git 변수 (path.path, path.basename 등)

조합 방식 1: Cross Product (기본값)
  cluster 1 × git 1
  cluster 1 × git 2
  cluster 2 × git 1
  cluster 2 × git 2
  → 모든 조합

조합 방식 2: Merge
  cluster 정보 + git 정보 동시에 (이름 충돌 시 조심)
```

**Cross Product 예:**

```yaml
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
      # → prod-us, prod-eu 클러스터
  - git:
      directories:
      - path: 'apps/*'
      # → apps/app1, apps/app2, apps/app3

  # 결과: 2 clusters × 3 apps = 6개 Application
  # app1-prod-us, app1-prod-eu, app2-prod-us, app2-prod-eu, ...
```

</details>

### Q3. 프로덕션 배포 시 ApplicationSet을 Manual Sync로 설정하면?

<details>
<summary>💡 해설</summary>

**구조:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-prod
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
  template:
    metadata:
      name: 'myapp-{{ name }}'
    spec:
      source:
        repoURL: https://github.com/company/app-config
        branch: main
        path: config/prod
      destination:
        server: '{{ server }}'
        namespace: default
      syncPolicy:
        # ← automated 없음 = Manual Sync
```

**프로덕션 배포 절차:**

```bash
# 1. 코드 변경, PR, 승인
$ git commit -m "Fix payment bug"
$ git push origin feature/payment-fix

# 2. PR 승인 후 main에 merge
# → ApplicationSet이 이미 생성한 Application들이 OutOfSync 상태로 변함

# 3. 모든 Production 클러스터에 동시에 배포
$ argocd app sync myapp-prod-us
$ argocd app sync myapp-prod-eu
$ argocd app sync myapp-prod-ap

# 또는 한 번에
$ argocd app list --selector env=prod -o json | \
  jq -r '.items[].metadata.name' | \
  xargs -I {} argocd app sync {}

# 4. 각 배포 상태 모니터링
$ watch argocd app list --selector env=prod
```

**더 나은 접근: 순차 배포**

```bash
# 1. 한 클러스터에 먼저 배포
$ argocd app sync myapp-prod-us
$ argocd app wait myapp-prod-us --sync

# 2. 모니터링 (메트릭 확인, 로그 확인)
# 10분 대기

# 3. 정상이면 다음 클러스터 배포
$ argocd app sync myapp-prod-eu
$ argocd app wait myapp-prod-eu --sync

# 4. 마지막 배포
$ argocd app sync myapp-prod-ap
```

**또는 Promotion 전략:**

```yaml
# config/
# ├── dev/
# ├── staging/
# └── prod/

# ApplicationSet은 3개 환경 동시 관리
spec:
  generators:
  - list:
      elements:
      - env: dev
        branch: dev
        clusters: [dev]
        syncPolicy: automated
      - env: staging
        branch: staging
        clusters: [staging]
        syncPolicy: automated
      - env: prod
        branch: main
        clusters: [prod-us, prod-eu, prod-ap]
        syncPolicy: manual

# 승격: dev (자동) → staging (자동) → prod (수동)
```

</details>

---

[⬅️ 이전](./04-kustomize-helm-integration.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./06-secret-management.md)
