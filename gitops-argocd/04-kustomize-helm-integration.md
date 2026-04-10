# Kustomize와 Helm 통합 — 환경별 설정 분리

## 🎯 핵심 질문

- ArgoCD에서 Kustomize overlay는 정확히 어떻게 동작하는가?
- Helm Chart와 values.yaml을 Git으로 관리할 때 장단점은?
- 왜 템플릿 렌더링이 클러스터가 아닌 Repo Server에서 일어나는가?
- dev/staging/prod 환경별 설정을 어떻게 효율적으로 분리할까?

## 🔍 왜 이 개념이 실무에서 중요한가

dev, staging, prod 환경에서 **거의 동일하지만 약간씩 다른** 설정을 관리해야 합니다:
- Image tag: dev=latest, staging=v1.2.0-rc1, prod=v1.2.0
- Replicas: dev=1, staging=2, prod=5
- Resource limits: dev=낮음, staging=중간, prod=높음
- 환경 변수, Secret 참조, Ingress 도메인 등...

**문제:**
- 3개 환경 × 10개 manifest = 30개 파일 수동 관리
- dev 버그 수정 → staging, prod에도 반영? → DRY 위반
- 복사 붙여넣기 오류 발생

**해결:**
- **Kustomize**: Base + Overlay 구조로 중복 제거
- **Helm**: 템플릿과 values로 변수화

## 😱 흔한 실수 (Before — 패턴을 모를 때의 접근)

### Before: 무분별한 복사 붙여넣기

```bash
# 디렉토리 구조 (❌ 나쁜 예)
app-config/
├── dev/
│   ├── deployment.yaml      # dev용 replicas=1
│   ├── service.yaml
│   ├── configmap.yaml
│   └── ingress.yaml
├── staging/
│   ├── deployment.yaml      # staging용 replicas=2 (복사본!)
│   ├── service.yaml         # 똑같음
│   ├── configmap.yaml       # 약간 다름
│   └── ingress.yaml
└── prod/
    ├── deployment.yaml      # prod용 replicas=5 (복사본!)
    ├── service.yaml         # 또 복사본
    ├── configmap.yaml       # 또 복사본
    └── ingress.yaml         # 또 복사본

# 문제 1: Deployment 버그 수정 → 3개 파일 모두 수정 필요
# 문제 2: 복사 과정에서 실수 (prod의 replicas를 5로 안 했음 등)
# 문제 3: Ingress 도메인이 다르면? 각각 수정, DRY 위반
```

```yaml
# dev/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myimage:latest
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
# staging/deployment.yaml (dev의 복사본)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 2  # ← 다르지만 수동 변경, 실수 가능
  # ... 나머지는 동일
---
# prod/deployment.yaml (또 복사본)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: app
        image: myimage:v1.2.3  # ← 다른 이미지 태그인데 수동 관리
```

**결과:**
- 3개 파일의 버그를 3번 수정
- 환경별 차이를 3배로 관리
- 새로운 manifest 추가 시 3배 작업

## ✨ 올바른 접근 (After — 패턴을 알고 난 설계)

### After: Kustomize + ArgoCD로 효율적 관리

```bash
# ✅ 올바른 구조: Base + Overlay
app-config/
├── base/                      # 모든 환경에 공통인 부분
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── overlays/                  # 환경별 차이만 정의
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── patch-deployment.yaml  # replicas=1로 override
    └── staging/
        ├── kustomization.yaml
        ├── patch-deployment.yaml  # replicas=2로 override
    └── prod/
        ├── kustomization.yaml
        ├── patch-deployment.yaml  # replicas=5로 override
        └── patch-image.yaml       # 이미지 태그 override
```

**이점:**
- Base: 하나의 deployment.yaml만 관리
- Overlay: 환경별 차이만 작게 정의
- 버그 수정: Base를 수정 → 모든 환경에 자동 반영
- 새 manifest: Base에만 추가

## 🔬 내부 동작 원리

### 1. Kustomize 개념 — Base + Patches

**Base 구조:**

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

labels:
- includeSelectors: true
  pairs:
    app: myapp
    managed-by: argocd

images:
- name: myimage
  newName: myimage  # 기본값
  newTag: latest    # 기본값

commonLabels:
  app: myapp
  version: "1.0"
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1  # 기본값 (overlay에서 override 가능)
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myimage:latest
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
```

**Overlay: dev 환경**

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

bases:
- ../../base

# Namespace 바꾸기
namePrefix: dev-

# 리소스 제한 조정
patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 256Mi

# 환경 변수 주입
configMapGenerator:
- name: app-config
  literals:
  - ENVIRONMENT=dev
  - DEBUG=true
  - LOG_LEVEL=debug

# 이미지 태그 (선택적)
images:
- name: myimage
  newTag: latest
```

**Overlay: prod 환경**

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod

bases:
- ../../base

replicas:
- name: app
  count: 5  # Deployment replicas를 5로

patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources
      value:
        requests:
          memory: "512Mi"
          cpu: "1000m"
        limits:
          memory: "1Gi"
          cpu: "2000m"

configMapGenerator:
- name: app-config
  literals:
  - ENVIRONMENT=prod
  - DEBUG=false
  - LOG_LEVEL=error

images:
- name: myimage
  newTag: v1.2.3  # 프로덕션 버전
```

**렌더링 결과:**

```bash
# Dev 환경의 최종 YAML
$ kustomize build overlays/dev/
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-app  # namePrefix 적용
  namespace: dev
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: myimage:latest
        resources:
          limits:
            memory: 256Mi  # ← 패치 적용

---
# Prod 환경의 최종 YAML
$ kustomize build overlays/prod/
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: prod
spec:
  replicas: 5  # ← replicas override
  template:
    spec:
      containers:
      - name: app
        image: myimage:v1.2.3  # ← 프로덕션 이미지
        resources:
          limits:
            memory: 1Gi  # ← 프로덕션 리소스
```

### 2. Helm 개념 — 템플릿 + Values

**Helm Chart 구조:**

```
myapp/
├── Chart.yaml              # Chart 메타데이터
├── values.yaml             # 기본값
├── values-dev.yaml         # Dev 오버라이드
├── values-staging.yaml     # Staging 오버라이드
├── values-prod.yaml        # Prod 오버라이드
└── templates/
    ├── deployment.yaml     # 템플릿 ({{ }} 사용)
    ├── service.yaml
    ├── configmap.yaml
    └── _helpers.tpl        # 헬퍼 함수
```

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.2.3"
description: My Application Chart
```

```yaml
# values.yaml (기본값)
replicaCount: 1

image:
  repository: myimage
  tag: latest
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"

environment: production
debug: false
```

```yaml
# values-dev.yaml (Dev 오버라이드)
replicaCount: 1

image:
  tag: latest

resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"

environment: dev
debug: true
```

```yaml
# values-prod.yaml (Prod 오버라이드)
replicaCount: 5

image:
  tag: v1.2.3

resources:
  requests:
    memory: "512Mi"
    cpu: "1000m"
  limits:
    memory: "1Gi"
    cpu: "2000m"

environment: production
debug: false
```

**템플릿:**

```yaml
# templates/deployment.yaml (템플릿)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          value: "{{ .Values.environment }}"
        - name: DEBUG
          value: "{{ .Values.debug }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

**렌더링 결과:**

```bash
# Dev 환경
$ helm template myapp ./myapp --namespace dev --values values-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-app
  namespace: dev
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: "myimage:latest"
        env:
        - name: ENVIRONMENT
          value: "dev"
        - name: DEBUG
          value: "true"
        resources:
          requests:
            memory: 64Mi
            cpu: 100m
          limits:
            memory: 128Mi
            cpu: 200m

---
# Prod 환경
$ helm template myapp ./myapp --namespace prod --values values-prod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-app
  namespace: prod
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: app
        image: "myimage:v1.2.3"
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: DEBUG
          value: "false"
        resources:
          requests:
            memory: 512Mi
            cpu: 1000m
          limits:
            memory: 1Gi
            cpu: 2000m
```

### 3. ArgoCD에서의 렌더링 위치

**중요한 개념: 렌더링은 클러스터 외부에서 일어남**

```
┌─────────────────────────────────────────────────┐
│ ArgoCD (클러스터)                                │
├─────────────────────────────────────────────────┤
│                                                 │
│ ┌──────────────────────────────────────────┐  │
│ │ Repo Server (클러스터 내부)                 │  │
│ │                                          │  │
│ │ 1. Git에서 Kustomize/Helm 가져오기        │  │
│ │ 2. 렌더링 (YAML 생성)  ← 여기서 일어남!   │  │
│ │ 3. 최종 YAML 반환                        │  │
│ └──────────────────────────────────────────┘  │
│         ↓                                      │
│ ┌──────────────────────────────────────────┐  │
│ │ Application Controller                    │  │
│ │                                          │  │
│ │ 1. Repo Server에서 최종 YAML 가져오기    │  │
│ │ 2. 클러스터 상태와 비교                   │  │
│ │ 3. kubectl apply (배포)                  │  │
│ └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
           ↓           ↓
    ┌─────────┐   ┌────────────┐
    │ Git     │   │ Kubernetes │
    │         │   │ API Server │
    │ (읽기)  │   │ (쓰기)     │
    └─────────┘   └────────────┘
```

**왜 클러스터 외부에서 렌더링?**
1. Kustomize/Helm은 실행 파일이 필요 → 클러스터에 설치 불필요
2. 렌더링 실패 시 → 클러스터 영향 없음
3. 성능 → 렌더링 후 최종 YAML만 네트워크 전송

## 💻 실전 실험

### 실험 1: Kustomize Base + Overlay 구조

```bash
# 1. Base 디렉토리 구조 생성
mkdir -p app-config/base app-config/overlays/{dev,staging,prod}

# 2. Base 파일 생성
cat > app-config/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: default

resources:
- deployment.yaml
- service.yaml

commonLabels:
  app: myapp
  managed-by: argocd

images:
- name: myimage
  newName: myimage
  newTag: latest
EOF

cat > app-config/base/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myimage:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

cat > app-config/base/service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
EOF

# 3. Dev Overlay
cat > app-config/overlays/dev/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

bases:
- ../../base

patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1

images:
- name: myimage
  newTag: latest
EOF

# 4. Prod Overlay
cat > app-config/overlays/prod/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod

bases:
- ../../base

replicas:
- name: app
  count: 5

patches:
- target:
    kind: Deployment
    name: app
  patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 1Gi

images:
- name: myimage
  newTag: v1.2.3
EOF

# 5. 렌더링 확인
$ kustomize build app-config/overlays/dev/
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
    managed-by: argocd
  name: app
  namespace: dev
spec:
  # ... dev 설정 적용됨

---
$ kustomize build app-config/overlays/prod/
# ... prod 설정 적용됨 (replicas=5, image=v1.2.3)

# 6. Git에 커밋
$ git add app-config/
$ git commit -m "Add Kustomize base + overlays"
```

### 실험 2: ArgoCD에서 Kustomize 렌더링

```bash
# 1. Dev Application 생성 (Kustomize)
$ argocd app create myapp-dev \
  --repo https://github.com/company/app-config \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev

# 2. Prod Application 생성
$ argocd app create myapp-prod \
  --repo https://github.com/company/app-config \
  --path overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod

# 3. Diff 확인 (Kustomize 자동 렌더링)
$ argocd app diff myapp-dev
# Kustomize가 base + dev overlay를 렌더링해서 보여줌

$ argocd app diff myapp-prod
# Base + prod overlay를 렌더링

# 4. Repo Server 로그 확인 (Kustomize 렌더링 과정)
$ kubectl logs -f -n argocd deployment/argocd-repo-server | grep -i kustomize
2026-04-10T15:00:00Z debug ... rendering kustomization overlays/dev
2026-04-10T15:00:01Z debug ... running kustomize build overlays/dev
2026-04-10T15:00:02Z debug ... rendering completed (1.2s)

# 5. 배포
$ argocd app sync myapp-dev
$ argocd app sync myapp-prod

# 6. 배포된 리소스 확인
$ kubectl get deployment -n dev
NAME   READY   UP-TO-DATE
app    1/1     1

$ kubectl get deployment -n prod
NAME   READY   UP-TO-DATE
app    5/5     5  # ← Prod는 replicas=5
```

### 실험 3: Helm Chart + ArgoCD

```bash
# 1. Helm Chart 구조 생성
mkdir -p myapp-chart/{templates,charts}

cat > myapp-chart/Chart.yaml <<EOF
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.2.3"
EOF

cat > myapp-chart/values.yaml <<EOF
replicaCount: 1
image:
  repository: myimage
  tag: latest
resources:
  limits:
    memory: 128Mi
    cpu: 200m
EOF

cat > myapp-chart/values-prod.yaml <<EOF
replicaCount: 5
image:
  tag: v1.2.3
resources:
  limits:
    memory: 1Gi
    cpu: 2000m
EOF

cat > myapp-chart/templates/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
EOF

# 2. Git에 커밋
$ git add myapp-chart/
$ git commit -m "Add Helm chart"
$ git push

# 3. ArgoCD에 Helm Application 생성
$ argocd app create myapp-helm-dev \
  --repo https://github.com/company/app-config \
  --path myapp-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --helm-set replicaCount=1

$ argocd app create myapp-helm-prod \
  --repo https://github.com/company/app-config \
  --path myapp-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --helm-set replicaCount=5 \
  --helm-set image.tag=v1.2.3

# 4. Repo Server의 Helm 렌더링 확인
$ kubectl logs -f -n argocd deployment/argocd-repo-server | grep -i helm
rendering helm template myapp-chart

# 5. 배포
$ argocd app sync myapp-helm-dev
$ argocd app sync myapp-helm-prod

# 6. 결과 확인
$ kubectl get deployment -n dev myapp -o jsonpath='{.spec.replicas}'
1

$ kubectl get deployment -n prod myapp -o jsonpath='{.spec.replicas}'
5
```

### 실험 4: app diff로 배포 전 확인

```bash
# 1. Git에서 Deployment 변경
$ git checkout -b test/image-bump
$ sed -i 's/tag: latest/tag: v1.2.4/' app-config/overlays/prod/kustomization.yaml
$ git add . && git commit -m "Bump image to v1.2.4" && git push

# 2. ArgoCD에서 변경사항 확인
$ argocd app diff myapp-prod
Deployment myapp-prod/app
  spec.template.spec.containers[0].image: myimage:v1.2.3 -> myimage:v1.2.4

# 3. 내용 검토 후 배포 결정
$ argocd app sync myapp-prod
```

## 📊 성능/비용 비교

| 항목 | Kustomize | Helm |
|------|-----------|------|
| **학습곡선** | 낮음 (JSON Patch) | 중간 (Go 템플릿) |
| **복잡한 로직** | 어려움 (제한적) | 쉬움 (함수 풍부) |
| **라이브러리 사용** | 안 함 | 지원 (subcharts) |
| **커뮤니티** | 점점 증가 | 매우 큼 (사실상 표준) |
| **렌더링 속도** | 빠름 | 중간 (템플릿 처리) |

## ⚖️ 트레이드오프

| 선택지 | 단순성 | 유연성 | 권장 |
|--------|--------|--------|-----|
| Kustomize | 높음 | 낮음~중간 | 간단한 설정 |
| Helm | 중간 | 높음 | 복잡한 애플리케이션 |
| 혼합 | 낮음 | 높음 | 특수 케이스 |

## 📌 핵심 정리

1. **Base + Overlay**: 공통 부분 한 번만 관리, 환경별 차이만 정의
2. **Kustomize**: JSON Patch로 간단한 오버라이드, 학습곡선 낮음
3. **Helm**: Go 템플릿으로 복잡한 로직, 차트 재사용 가능
4. **Repo Server 렌더링**: 클러스터 외부에서 일어나므로 실행 파일 불필요
5. **app diff**: 배포 전에 정확히 뭐가 바뀔지 확인 가능
6. **values 파일**: 환경별로 다른 values 파일로 쉬운 관리

## 🤔 생각해볼 문제

### Q1. Kustomize와 Helm을 섞어 쓰면 어떻게 될까?

<details>
<summary>💡 해설</summary>

**가능하지만 권장 안 함:**

```bash
# 시나리오: Helm Chart를 Kustomize로 또 overlay

# 폴더 구조
app-config/
├── charts/
│   └── myapp/       # Helm Chart
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-values.yaml  # ← Helm Chart를 Kustomize로 패치
    └── prod/
```

**문제:**
```yaml
# overlays/dev/kustomization.yaml
bases:
- ../../charts/myapp  # ← Helm Chart를 base로 사용?

patches:
- target:
    kind: Deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1
```

**렌더링 순서가 불명확:**
1. Helm이 먼저 렌더링? (values.yaml 기준)
2. Kustomize가 Helm을 패치?
3. 둘 다? 복잡함...

**권장 방식:**
- **Helm만**: Helm values 파일로 모든 환경 관리
- **Kustomize만**: Kustomize overlay로 모든 환경 관리
- **혼합**: Helm Chart를 생성, Kustomize로 패치 아닌 별도 구조

```yaml
# ✅ 좋은 혼합: Helm + 별도 Kustomize
app-config/
├── charts/
│   └── myapp/           # Helm Chart (순수)
├── helm-values/
│   ├── values-dev.yaml
│   └── values-prod.yaml
└── post-helm/           # Helm 이후 추가 리소스 (Kustomize)
    ├── dev/
    │   ├── kustomization.yaml
    │   └── additional-configmap.yaml
    └── prod/
```

</details>

### Q2. Base를 여러 번 중첩하면 (base of base)?

<details>
<summary>💡 해설</summary>

**구조:**

```bash
app-config/
├── bases/
│   ├── core/              # 최상위 base (모든 공통)
│   │   ├── kustomization.yaml
│   │   └── deployment.yaml
│   ├── observability/     # Core를 상속한 base
│   │   ├── kustomization.yaml
│   │   └── prometheus.yaml
│   └── networking/        # Core를 상속한 또 다른 base
│       ├── kustomization.yaml
│       └── ingress.yaml
└── overlays/
    ├── dev/               # 모든 base를 상속
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

**구현:**

```yaml
# bases/core/kustomization.yaml
resources:
- deployment.yaml
- service.yaml

---
# bases/observability/kustomization.yaml
bases:
- ../core  # ← core를 base로 함

resources:
- prometheus.yaml
- grafana.yaml

---
# overlays/prod/kustomization.yaml
bases:
- ../../bases/observability  # ← observability를 base로 (core도 자동 포함)

patches:
- target:
    kind: Deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

**렌더링:**
```bash
$ kustomize build overlays/prod/
# 1. bases/core의 모든 리소스
# 2. bases/observability의 추가 리소스
# 3. overlays/prod의 패치 적용
```

**장점:**
- 공통 부분의 재사용
- 기능 단위로 조합 가능 (observability, networking, security 등)

**단점:**
- 렌더링 순서 이해하기 복잡
- 디버깅 어려움

**권장:**
- 2단계까지만 (core → overlay)
- 3단계 이상은 Helm으로 차라리 관리

</details>

### Q3. values.yaml을 환경별로 여러 개 만들 때, 한 번에 적용하는 순서는?

<details>
<summary>💡 해설</summary>

**Helm에서 여러 values 파일 적용:**

```bash
$ helm template myapp ./myapp-chart \
  -f values.yaml \                 # 기본값 먼저
  -f values-prod.yaml \            # 그 다음 override
  -f values-secrets.yaml           # 마지막 override (높은 우선순위)
```

**병합 순서:**
```
values.yaml (기본)
  ↓
values-prod.yaml (prod 특화, 덮어쓰기)
  ↓
values-secrets.yaml (비밀 값, 최고 우선순위)
```

**예시:**

```yaml
# values.yaml
replicaCount: 1
environment: default

# values-prod.yaml
replicaCount: 5
environment: prod

# values-secrets.yaml
image:
  pullSecrets:
  - name: docker-secret  # ← 추가되는 항목
```

**최종 병합 결과:**
```yaml
replicaCount: 5  # ← values-prod에서 override
environment: prod
image:
  pullSecrets:
  - name: docker-secret
```

**ArgoCD에서:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  source:
    repoURL: https://github.com/company/app-config
    path: myapp-chart
    helm:
      valueFiles:
      - values.yaml           # 기본
      - values-prod.yaml      # override
      - values-secrets.yaml   # 최고 우선순위
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
```

</details>

---

[⬅️ 이전](./03-sync-strategy.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./05-multi-cluster-management.md)
