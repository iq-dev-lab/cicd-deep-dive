# GitOps 원칙 완전 분해 — 4원칙과 감사 추적성

## 🎯 핵심 질문

- Git이 Source of Truth인데, 그럼 Kubernetes 클러스터에서 `kubectl edit`으로 직접 수정하면 어떻게 되는가?
- 명령형(`kubectl apply`) 배포와 선언형(GitOps) 배포의 근본적인 차이는 무엇인가?
- Git 커밋 히스토리가 배포 Audit Log를 완전히 대체할 수 있는가?
- Pull 모델이 Push 모델보다 보안상 진정 우월한가?

## 🔍 왜 이 개념이 실무에서 중요한가

배포 자동화를 도입한 팀의 70%가 "누가 언제 뭘 배포했는지 추적하기 어렵다"고 호소합니다. 특히 장애 발생 시 "1시간 전엔 정상이었는데..."이라는 말만 있고, 정확한 배포 이력이 없으면 롤백도, 원인 분석도 불가능합니다.

GitOps는 단순한 배포 도구가 아니라, **Git 커밋이 신뢰할 수 있는 감사 추적(Audit Trail)**이 되도록 강제합니다. CI 서버가 자동으로 배포하든, 개발자가 손수 배포하든 모든 변경이 Git 히스토리에 기록되고, 어떤 파일이 언제 누구에 의해 어떻게 변경되었는지 영구적으로 추적됩니다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

### Before: 명령형 배포의 혼란

팀 A는 다음과 같이 배포합니다:

```bash
# 개발자 손으로 직접 배포
$ kubectl set image deployment/app app=myimage:v1.2.3 -n production
deployment.apps/app image updated

# 긴급 패치는 Helm Chart 없이 직접 수정
$ kubectl edit deployment app -n production
# vim으로 직접 편집... 저장

# 누군가는 CI 자동화
$ helm upgrade app ./chart --values prod-values.yaml

# 누군가는 실수로 직접 포드를 수정
$ kubectl patch pod app-xyz -p '{"spec":{"containers":[{"name":"app","image":"myimage:emergency"}]}}'
```

**발생하는 문제:**
- Git 저장소의 `deployment.yaml`과 실제 클러스터 상태가 불일치
- 누가 언제 뭘 수정했는지 추적 불가능
- 배포 히스토리는 로그에 산재, Slack 채널에만 "배포했습니다"라는 메시지만 남음
- 장애 발생 시: "혹시 어제 누가 뭘 건드렸나?" → 수동 조사
- 새 팀원이 온라인 클러스터를 원본으로 착각, 로컬 Git을 버리고 클러스터에서 복사

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### After: GitOps 기반 선언형 배포

```bash
# 모든 변경은 Git 저장소에 커밋 (GitHub/GitLab)
# Git이 유일한 Source of Truth

$ git log --all --graph --oneline -- config/
* 3f2a1c9 (HEAD -> main) Bump app image to v1.2.3 (deployment: app)
* b8d7c4e Fix resource limits for production
* a1e9f6d Add Prometheus annotation for monitoring
* 7c3b9d2 Initial deployment configuration

# 모든 변경의 근거가 PR/Commit message에 남음
$ git show 3f2a1c9
Author: alice@company.com
Date: Thu Apr 10 15:23:45 2026
Message: Bump app image to v1.2.3

  - Fixes critical bug in payment processing
  - Reviewed by: @bob @carol
  - Closes: JIRA-1234

# ArgoCD는 Git의 선언형 상태를 읽어서 클러스터와 자동 동기화
$ argocd app get myapp
Name: myapp
Namespace: argocd
Project: default
Sync Policy: Automated
Sync Status: Synced (Git과 클러스터가 일치)
Last Sync Time: 2026-04-10T15:25:30Z
Repository: https://github.com/company/app-config
Target Revision: main
```

**얻는 이점:**
- Git 커밋이 신뢰할 수 있는 Audit Log
- `git blame` → 누가 언제 변경했는지 즉시 확인
- `git revert` → 문제 커밋 되돌리기 (배포 롤백)
- 클러스터 상태가 항상 Git과 동기화
- 만약 누군가 `kubectl edit`으로 클러스터를 직접 수정하면, ArgoCD가 자동으로 Git 상태로 복원

## 🔬 내부 동작 원리

### 1. Declarative 원칙 — "현재 상태" vs "원하는 상태"

```yaml
# 명령형 (Imperative): "이 작업을 실행하라"
$ kubectl run app --image=myimage:v1.0.0
$ kubectl expose deployment app --port=8080
$ kubectl scale deployment app --replicas=3

# 선언형 (Declarative): "이런 상태이고 싶다"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
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
        image: myimage:v1.0.0
        ports:
        - containerPort: 8080
---
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
```

**차이:**
- 명령형: 각 명령은 무상태(stateless), 같은 명령을 반복하면 오류 발생 (Service 이미 존재 등)
- 선언형: YAML만 관리하면 Kubernetes가 "현재 상태"를 "원하는 상태"로 자동 조정 (멱등성 보장)

### 2. Versioned 원칙 — Git이 Source of Truth

```bash
# Git 저장소 구조
app-config/
├── config/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── values.yaml
├── helm/
│   └── myapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── .gitignore  # ← Secrets는 반드시 제외!
└── README.md

# 모든 변경이 Git 히스토리에 남음
$ git log --all --pretty=format:"%H %an %ad %s" -- config/prod/
3f2a1c9 alice 2026-04-10 Bump app image to v1.2.3
b8d7c4e bob   2026-04-09 Fix resource limits
a1e9f6d carol 2026-04-08 Add Prometheus annotation
```

**이점:**
- `git blame`: 누가, 언제, 무엇을
- `git diff`: 정확히 뭐가 바뀌었는지
- `git revert`: 어떤 커밋든 원복 가능
- Git branch로 환경별 테스트 가능

### 3. Automated 원칙 — CI 서버가 Git을 갱신

```bash
# 개발자는 코드만 수정
$ git commit -m "Fix payment bug"

# CI 파이프라인이 자동으로 배포 설정을 갱신
# (예: 빌드 완료 후 새 이미지 SHA로 deployment.yaml 업데이트)

# Git 저장소에 새 커밋이 자동으로 푸시됨
$ git log --oneline
abc1234 [CI] Bump app image to sha:1a2b3c4d (automated)
def5678 Fix payment bug

# ArgoCD가 Git 변경을 감지하고 클러스터 배포
# (Webhook 또는 3분마다 폴링)
```

**자동화 흐름:**
```
개발자 커밋 → CI 빌드 → 이미지 푸시 → Git 업데이트 
→ ArgoCD 감지 → 클러스터 동기화
```

### 4. Continuously Reconciled 원칙 — "드리프트" 자동 감지 및 복구

```bash
# 현재 상태: Git과 클러스터가 일치 (Synced)
$ argocd app get myapp
Status: Synced
Last Sync Time: 2026-04-10T15:25:30Z

# 누군가 cluster에서 직접 수정 (의도하지 않은 변경)
$ kubectl set image deployment/app app=emergency-hotfix:v999 -n prod

# ArgoCD가 3분마다 체크 (또는 Webhook으로 즉시)
$ argocd app get myapp
Status: OutOfSync
Diff:
  deployment.yaml: 
    spec.template.spec.containers[0].image
    - emergency-hotfix:v999  (실제 클러스터)
    + myimage:v1.2.3         (Git의 선언형)

# ArgoCD가 자동으로 Git 상태로 복원 (Self-Heal)
$ argocd app sync myapp --auto
Synced successfully
# 또는 매뉴얼 명령
$ argocd app sync myapp
```

### 5. 명령형과 선언형의 혼재 — "Drift" 발생

```bash
# ❌ 나쁜 패턴: 선언형 + 명령형 혼재
# 1. Git에는 replicas: 3으로 정의
cat deployment.yaml
spec:
  replicas: 3

# 2. 트래픽 스파이크 → 운영자가 kubectl로 직접 스케일
$ kubectl scale deployment app --replicas=10

# 3. ArgoCD의 다음 자동 동기화에서...
$ argocd app sync myapp --auto
# Replicas: 10 → 3으로 자동 감소 (운영자가 의도한 것 무시)
# 사용자 요청이 갑자기 느려짐...

# ✅ 올바른 패턴: 선언형만 사용
# 1. Git을 수정해서 원하는 상태를 선언
$ git commit -m "Scale app to 10 replicas due to traffic spike"
$ git show HEAD:deployment.yaml | grep replicas
  replicas: 10

# 2. ArgoCD가 이를 읽어서 클러스터 적용
$ argocd app sync myapp
```

### 6. Pull 모델 vs Push 모델 — 보안 아키텍처

**Push 모델 (전통적 CI/CD):**
```
CI 서버 → 클러스터에 직접 kubectl apply
  |
  └─ 문제: CI 서버가 클러스터 접근권한 필요
      - CI 서버 손상 시 전체 클러스터 위험
      - 클러스터 크래덴셜을 CI 서버에 저장해야 함
      - 멀티 클러스터 관리 시 N개 클러스터의 크래덴셜 필요
```

**Pull 모델 (GitOps):**
```
ArgoCD 컨트롤러 (클러스터 내부) ← Git 저장소를 폴링/Webhook
  |
  └─ 장점:
      - 클러스터 내부에서 Git만 읽음 (어떤 권한도 Git에 주지 않음)
      - CI 서버 손상 → Git은 안전
      - Git 토큰이 클러스터 내부의 Secret으로만 존재
      - 멀티 클러스터 각각이 Git을 폴링 (중앙집중식 배포 서버 불필요)
```

```yaml
# 클러스터 내부의 ArgoCD Secret
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
  namespace: argocd
type: Opaque
data:
  url: aHR0cHM6Ly9naXRodWIuY29tL2NvbXBhbnkvYXBwLWNvbmZpZw==  # Base64
  password: Z2hwX1hYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWA==  # GitHub PAT
  # ← 클러스터 내부에서만 존재, 외부에 노출 안 됨
```

## 💻 실전 실험 (재현 가능한 예시)

### 실험 1: ArgoCD 설치 및 기본 앱 배포

```bash
# 1. ArgoCD 네임스페이스 생성 및 ArgoCD 설치
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. ArgoCD CLI 설치
$ curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
$ chmod +x argocd && sudo mv argocd /usr/local/bin/

# 3. ArgoCD API 서버 접근
$ kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# 브라우저: https://localhost:8080

# 4. ArgoCD 로그인 (기본 암호는 argocd Pod 이름)
$ ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
$ argocd login localhost:8080 --username admin --password $ARGOCD_PASS --insecure

# 5. Git 저장소 등록
$ argocd repo add https://github.com/company/app-config \
  --username <github-username> \
  --password <github-token> \
  --insecure-skip-server-verification
```

### 실험 2: Application 리소스 생성 및 Sync 관찰

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/app-config
    targetRevision: main
    path: config/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false      # Git에서 삭제된 리소스 자동제거 안 함
      selfHeal: true    # kubectl edit으로 수정된 것 자동 복원
    syncOptions:
    - CreateNamespace=true
```

```bash
# 적용
$ kubectl apply -f argocd-app.yaml

# 상태 확인
$ argocd app get myapp
Name: myapp
Namespace: argocd
Project: default
Status: Synced
Health: Healthy
Repository: https://github.com/company/app-config
Path: config/prod
Target: main
Sync Policy: Automated
Sync Result:
  Resources: 3 created, 0 updated, 0 deleted
  Revision: 3f2a1c9abc123def456

# Git과 클러스터의 Diff 확인
$ argocd app diff myapp

# 클러스터 상태를 Git으로 복원 (동기화)
$ argocd app sync myapp --prune
# Syncing application details
# ...
# SYNCED
```

### 실험 3: Drift 감지 및 자동 복원

```bash
# 1. 현재 상태 확인 (Synced)
$ argocd app get myapp | grep Status

# 2. 누군가가 cluster에서 직접 수정
$ kubectl set image deployment/app app=myimage:emergency -n production
# 또는
$ kubectl edit deployment app -n production
# (replicas를 5로 수정)

# 3. ArgoCD 상태 확인 (OutOfSync로 표시됨)
$ argocd app get myapp
Status: OutOfSync
Diff:
  deployment.yaml:
    spec.replicas: 5 (실제) vs 3 (Git)
    spec.containers[0].image: myimage:emergency (실제) vs myimage:v1.2.3 (Git)

# 4. 자동 복원 (Self-Heal)
# Self-Heal이 활성화되어 있으면 3분 내에 자동 복원
# 또는 수동으로 즉시 동기화
$ argocd app sync myapp

# 5. 확인
$ argocd app get myapp
Status: Synced
```

### 실험 4: Git Commit 이력 기반 감사

```bash
# 배포 이력 확인 (Git 커밋)
$ git log --all --pretty=format:"%h %an %ad %s" -- config/prod/
3f2a1c9 alice 2026-04-10 Bump app image to v1.2.3
b8d7c4e bob   2026-04-09 Fix resource limits for OOM
a1e9f6d carol 2026-04-08 Add Prometheus scrape annotation

# 특정 배포의 상세 내용
$ git show 3f2a1c9
Author: alice@company.com
Date: Thu Apr 10 15:23:45 2026
Message: Bump app image to v1.2.3

  Security patch for payment processing vulnerability
  Approved by: @security-team
  Jira: SECURITY-1234

diff --git a/config/prod/kustomization.yaml b/config/prod/kustomization.yaml
index abc1234..def5678 100644
--- a/config/prod/kustomization.yaml
+++ b/config/prod/kustomization.yaml
@@ -5,7 +5,7 @@ bases:
 images:
 - name: myimage
-  newTag: v1.2.2
+  newTag: v1.2.3

# 누가 이 파일을 마지막으로 수정했는지
$ git blame config/prod/kustomization.yaml
3f2a1c9 (alice 2026-04-10) spec:
3f2a1c9 (alice 2026-04-10)   replicas: 3
b8d7c4e (bob   2026-04-09)   resources:
a1e9f6d (carol 2026-04-08)   - deployment.yaml

# 롤백 (특정 커밋으로 되돌리기)
$ git revert 3f2a1c9
$ git push origin main
# ArgoCD가 자동으로 감지하고 이전 상태로 복원
```

## 📊 성능/비용 비교

| 항목 | 명령형 (전통 CI/CD) | 선언형 (GitOps) |
|------|------------------|----------------|
| **배포 시간** | 5~10초 (직접 실행) | 5~10초 (동기화) + 3분 폴링 |
| **감사 추적** | 로그 서버에 분산, 보존 기간 제한 | Git 영구 기록, `git blame/log` 즉시 확인 |
| **롤백 시간** | 수동 조사 + 배포 (10~30분) | `git revert` + 자동 배포 (3분 내) |
| **멀티 클러스터** | CI에서 N번 배포 (N×리소스) | 각 클러스터가 Git 폴링 (1×리소스) |
| **보안** | CI 서버에 모든 클러스터 크래덴셜 | 클러스터 내 Git 토큰만 필요 |
| **운영 학습곡선** | 낮음 | 중간 (Git 기본 지식 필수) |
| **비용** | 높음 (CI 서버 리소스 관리) | 낮음 (경량 컨트롤러) |

## ⚖️ 트레이드오프

| 트레이드오프 | GitOps의 선택 | 비용 |
|-----------|-------------|-----|
| **즉시 배포 vs 3분 폴링** | 3분 폴링이 기본, Webhook으로 즉시화 가능 | Webhook 설정 필요 |
| **완전 자동화 vs 수동 승인** | Automated Sync 비활성화 → 수동 검증 후 `sync` | 운영 수고 증가 |
| **Cluster 직접 수정 제한** | Self-Heal로 Git 상태 강제 → 변경 불가 | 긴급 패치 시 Git 먼저 수정해야 함 |
| **Secret 관리** | Git에 평문 저장 불가 → Sealed Secrets/Vault 필수 | 추가 운영 복잡도 |
| **상세한 감사 vs 과도한 Git Commit** | 모든 변경이 커밋 → Git 히스토리 빠르게 증가 | `git log` 관리 필요 |

## 📌 핵심 정리

1. **Declarative**: YAML로 "원하는 상태"를 정의, Kubernetes가 현재 상태를 맞춤
2. **Versioned**: 모든 설정을 Git에서 관리, 히스토리 영구 보존
3. **Automated**: CI가 자동으로 Git을 갱신, ArgoCD가 자동으로 배포
4. **Continuously Reconciled**: ArgoCD가 주기적으로 Git ↔ Cluster 상태 확인, 불일치 자동 복구
5. **Audit Trail**: Git 커밋 = 배포 감사 로그, `git blame/log/revert`로 완전 추적
6. **Pull Model의 보안**: 클러스터가 Git을 폴링, CI 서버 손상 영향 최소화
7. **Drift 방지**: 누군가 `kubectl edit`으로 수정해도 ArgoCD가 자동 복원 (Self-Heal)

## 🤔 생각해볼 문제

### Q1. Git에만 변경을 허용하면, 긴급 패치는 어떻게 적용하나?

<details>
<summary>💡 해설</summary>

**시나리오**: 보안 취약점 발견, 즉시 패치 필요 (Git commit 기다릴 수 없음)

**틀린 접근**: 
```bash
$ kubectl set image deployment/app app=emergency-fix:v999
# Git과 불일치, ArgoCD가 3분 후 원상복구
```

**올바른 접근**:
```bash
# 1. 긴급 변경도 Git에 먼저 커밋 (빠르게)
$ git checkout -b hotfix/security-patch-urgent
$ vim config/prod/kustomization.yaml  # 이미지 버전 변경
$ git add .
$ git commit -m "URGENT: Security patch for CVE-2026-1234"
$ git push origin hotfix/...

# 2. 즉시 PR Merge (승인 간소화 가능)
# GitHub에서 Merge

# 3. ArgoCD가 Webhook으로 즉시 감지, 수초 내 배포
# (폴링이 아닌 Webhook 설정 필수)

# 또는 수동으로 즉시 동기화
$ argocd app sync myapp --prune
```

**왜 이게 더 나은가:**
- Git이 근본 원인 (실제 배포된 이미지 버전)
- 나중에 "누가 왜 emergency-fix:v999를 배포했나?" 추적 가능
- rollback도 `git revert` 한 줄

</details>

### Q2. 모든 설정이 Git에 있으면, Secret은 어디에 둘까?

<details>
<summary>💡 해설</summary>

**금지**: 데이터베이스 비밀번호, API 키를 Git에 평문으로 저장

```bash
# ❌ 절대 금지
$ git add config/secrets.yaml
# 나중에 git log로도 히스토리에서 영구 삭제 어려움
```

**해결책들** (자세한 내용은 Chapter 5-6 참고):

1. **Sealed Secrets**: Git에는 암호화된 파일만 저장
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
spec:
  encryptedData:
    password: AgBc1234abcdef...  # 클러스터의 공개키로 암호화
```

2. **External Secrets Operator**: Git에는 참조만, 실제 Secret은 AWS Secrets Manager
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
spec:
  secretStoreRef:
    name: aws-secrets
  target:
    name: db-password  # 클러스터에 생성될 Secret 이름
  data:
  - secretKey: password
    remoteRef:
      key: prod/db-password
```

3. **SOPS**: KMS/GPG로 파일 암호화, Git에 저장
```bash
$ sops -e secrets.yaml > secrets.enc.yaml
$ git add secrets.enc.yaml
```

</details>

### Q3. 개발(dev) 환경에서는 "꼭 Git으로만" 해야 하나? Manual Sync는 언제 쓸까?

<details>
<summary>💡 해설</summary>

**답**: 아니다. 환경별로 다르게 설정 가능

```yaml
# dev 환경: 자주 테스트, 빠른 피드백 필요
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
spec:
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스 자동 제거
      selfHeal: true    # kubectl edit 자동 복원
    # 3분마다 자동 동기화

---
# prod 환경: 변경 신중함, 수동 검증 필수
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  syncPolicy:
    # automated 없음 = 수동 동기화만
    syncOptions:
    - CreateNamespace=true
    # 배포 전 diff 확인, 명시적으로 sync 실행
```

**Manual Sync 사용 패턴:**
```bash
# 1. Diff 먼저 확인
$ argocd app diff myapp-prod
# 변경사항을 리뷰

# 2. 팀 합의 (Slack, Jira 등)

# 3. 명시적으로 배포
$ argocd app sync myapp-prod --prune

# 또는 UI에서 "SYNC" 버튼 클릭
```

**정리:**
- **dev**: 자동 배포 (빠른 피드백)
- **staging**: 자동 배포 + Self-Heal (검증 환경)
- **prod**: 수동 배포 (변경 신중함)

</details>

### Q4. "Git이 근본 진실"이면, 클러스터를 완전 새로 만들 때는 어떻게 할까?

<details>
<summary>💡 해설</summary>

**시나리오**: 새 클러스터 추가 또는 기존 클러스터 재구성

**장점: Git에서 모든 설정을 가져올 수 있음**

```bash
# 1. 새 클러스터 생성
$ kind create cluster --name new-cluster

# 2. ArgoCD 설치 (자동화 가능)
$ kubectl --context kind-new-cluster create namespace argocd
$ kubectl --context kind-new-cluster apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. ArgoCD에 클러스터 등록
$ argocd cluster add kind-new-cluster

# 4. 모든 앱 설정을 Git에서 배포
$ argocd app create myapp \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://new-cluster-api \
  --dest-namespace production

# 5. 동기화 (Git의 모든 리소스 클러스터에 생성)
$ argocd app sync myapp --prune

# ✅ 몇 분 내에 완전히 동일한 상태의 새 클러스터 완성!
```

**비교: 전통적 수동 배포**
- 클러스터 생성
- Helm Chart 설치
- ConfigMap, Secret 생성
- 모니터링 에이전트 설정
- ... (문서 대로 따라하기, 실수 가능)
- 시간 소요: 1~2시간, 오류 가능성 높음

**GitOps의 이점:**
- 모든 설정이 코드 (재현 가능)
- 수동 오류 없음
- 시간: 10~15분

</details>

---

[⬅️ 이전: Chapter 4 — 배포 파이프라인 전체 흐름](../deployment-strategies/07-deployment-pipeline-flow.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./02-argocd-architecture.md)
