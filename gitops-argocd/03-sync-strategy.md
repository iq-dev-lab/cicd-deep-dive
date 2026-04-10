# Sync 전략 — Self-Heal과 Pruning

## 🎯 핵심 질문

- Manual Sync와 Automated Sync는 정확히 언제 어떻게 다르게 동작하는가?
- "Self-Heal"은 좋은 건가? 아니면 운영자의 변경을 무시하는 문제는 없나?
- Pruning이 기본으로 비활성화된 이유는 무엇인가?
- Sync Wave와 Sync Phase는 언제 필요하고, 어떻게 작동하는가?

## 🔍 왜 이 개념이 실무에서 중요한가

Sync 전략은 **"언제, 어떻게, 어떤 순서로 배포할 것인가"**를 결정합니다. 잘못된 Sync 전략은:

1. **Automated Sync + 무분별한 Pruning**: 실수로 중요한 리소스 삭제
2. **Manual Sync만 사용**: 변경 반영이 느려서 장애 대응 시간 증가
3. **Sync Wave 무시**: 데이터베이스 마이그레이션 전에 앱이 실행, 서비스 중단
4. **Self-Heal 비활성화**: 운영자의 긴급 패치가 계속 되돌려짐

## 😱 흔한 실수 (Before — 전략을 모를 때의 접근)

### Before: 무분별한 설정

```yaml
# ❌ 나쁜 예 1: 너무 자동화
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      prune: true       # ← Git에서 삭제된 리소스 자동 제거
      selfHeal: true    # ← kubectl edit으로 수정한 것 자동 복원
```

```bash
# 문제 상황 1: 실수로 Git에서 ConfigMap 삭제
$ git rm config/prod/configmap.yaml
$ git commit -m "Remove old config"
$ git push

# ArgoCD가 자동으로 클러스터의 ConfigMap도 삭제
# → 애플리케이션이 ConfigMap을 못 찾아서 크래시

# 문제 상황 2: 운영자가 긴급 환경 변수 추가
$ kubectl set env deployment/app DEBUG=true -n production

# 3분 후 ArgoCD가 Git 상태로 자동 복원
# DEBUG=true가 사라짐
# → 운영자의 조치가 무시됨
```

```yaml
# ❌ 나쁜 예 2: 동기화 순서 무시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myimage:v1.2.3
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-password
              key: password
---
apiVersion: v1
kind: Secret
metadata:
  name: db-password
data:
  password: base64encodedvalue
```

```bash
# 문제: Secret이 아직 생성되지 않았는데 Deployment가 먼저 생성될 수 있음
# → Pod가 Secret을 못 찾아서 대기 또는 실패
```

## ✨ 올바른 접근 (After — 전략을 이해한 설계)

### After: 환경별 맞춤 전략

```yaml
# ✅ dev 환경: 자주 배포, 빠른 피드백
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
spec:
  syncPolicy:
    automated:
      prune: true       # 개발 환경은 빠른 정리
      selfHeal: true    # 직접 수정해도 Git 상태 유지
    syncOptions:
    - CreateNamespace=true

---
# ✅ prod 환경: 신중함, 수동 검증
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  syncPolicy:
    # automated 없음 = 수동 배포
    syncOptions:
    - CreateNamespace=true
    # Diff 확인 후 수동으로 sync
```

### After: Sync Wave로 순서 제어

```yaml
# ✅ 올바른 순서: DB 마이그레이션 → 앱 배포 → 스모크 테스트

# 1. PreSync: DB 마이그레이션 (실행 우선, 동기화 전)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync        # ← PreSync Hook
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      serviceAccountName: db-migrator
      containers:
      - name: migrate
        image: migrate:latest
        command: ["migrate", "-path", "/migrations", "-database", "$DATABASE_URL", "up"]
      restartPolicy: Never
  backoffLimit: 1

---
# 2. Sync: 실제 앱 배포 (기본값, wave=0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"      # ← 0번 웨이브 (기본값)
spec:
  replicas: 3
  # ... 나머지 설정

---
# 3. PostSync: 스모크 테스트 (Sync 후 실행)
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync       # ← PostSync Hook
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      serviceAccountName: test-runner
      containers:
      - name: test
        image: test:latest
        command: ["sh", "-c", "curl -f http://myapp:8080/health || exit 1"]
      restartPolicy: Never
  backoffLimit: 1
```

## 🔬 내부 동작 원리

### 1. Manual Sync vs Automated Sync

**Manual Sync (기본값, 안전함):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    # automated 없음 = Manual Sync
```

```bash
# 워크플로우
1. Git 변경 → ArgoCD 감지
2. Application 상태: OutOfSync
3. 운영자가 확인: argocd app diff myapp
4. 운영자 판단: "뭐가 바뀌었는지, 배포해도 되나?"
5. 수동 배포: argocd app sync myapp
6. 또는 UI에서 SYNC 버튼 클릭

# 장점:
# - 배포 전 diff 확인 가능
# - 실수로 삭제되는 리소스 방지
# - 운영자의 의도적 변경 가능

# 단점:
# - 배포가 느림 (3분 폴링 후 수동 개입)
# - 변경 놓치기 쉬움
```

**Automated Sync (편함, 위험할 수 있음):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:      # ← 자동 배포 활성화
      prune: false  # Git 삭제 리소스 제거 안 함 (안전)
      selfHeal: true  # kubectl edit 자동 복원
```

```bash
# 워크플로우
1. Git 변경 → ArgoCD 감지
2. Application 상태: OutOfSync (순간, 폴링이나 Webhook에서)
3. Application Controller가 자동으로 Sync 시작
4. 클러스터에 자동 배포, 상태: Progressing
5. 배포 완료, 상태: Synced

# 장점:
# - 빠른 배포 (수초 내)
# - 자동화된 워크플로우
# - 운영자 개입 불필요

# 단점:
# - diff 확인 불가능 (자동이라)
# - 실수 배포 위험 (잘못된 Git 커밋도 자동 배포)
# - 긴급 수정 불가능 (자동 복원됨)
```

### 2. Self-Heal — kubectl edit 방지

**Self-Heal 비활성화 (기본값):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      selfHeal: false  # ← kubectl edit 변경이 유지됨
```

```bash
# 시나리오
1. Git의 replicas: 3

2. 운영자가 트래픽 증가로 kubectl 직접 수정
$ kubectl scale deployment/app --replicas=10

3. ArgoCD 폴링 시 상태 확인
$ argocd app get myapp
Status: OutOfSync
  deployment.spec.replicas: 10 (actual) vs 3 (desired)

4. 하지만 ArgoCD는 아무 조치 안 함
# 트래픽이 많으므로 replicas=10으로 유지 가능

5. 다음날, 트래픽이 정상화되면
$ git commit -m "Scale down to 3 replicas"
$ git push

6. 그 후 ArgoCD가 Sync해서 replicas=3으로 조정
```

**Self-Heal 활성화:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      selfHeal: true  # ← 자동 복원 활성화
```

```bash
# 시나리오
1. Git의 replicas: 3

2. 운영자가 트래픽 증가로 kubectl 직접 수정
$ kubectl scale deployment/app --replicas=10

3. ArgoCD 폴링 (3분 또는 Webhook) 또는 수동 refresh
$ argocd app get myapp --refresh
Status: OutOfSync
  deployment.spec.replicas: 10 (actual) vs 3 (desired)

4. Self-Heal 활성화되어 있으므로 자동 복원
$ kubectl scale deployment/app --replicas=3  # 자동으로 실행

5. 상태: Synced
# 운영자의 트래픽 대응 수동 수정이 무시됨!

# 문제: 급하게 필요한 스케일링도 3분 후 되돌려짐
```

### 3. Pruning — 리소스 자동 삭제

**Pruning 비활성화 (기본값, 안전함):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      prune: false  # ← Git에서 삭제된 리소스 유지
```

```bash
# 시나리오
1. Git에 이런 리소스들이 있었음:
   - Deployment myapp
   - Service myapp
   - ConfigMap myapp-config
   - Secret db-password

2. ConfigMap이 더 이상 필요없어서 Git에서 삭제
$ git rm config/prod/configmap.yaml
$ git push

3. ArgoCD Sync 후
# Deployment, Service는 동기화됨
# 하지만 ConfigMap은 클러스터에 그대로 남음!

4. 이점:
   - 실수로 삭제되는 상황 방지
   - 안전함
   - 운영자가 필요하면 따로 정리 가능

5. 단점:
   - 실수로 만든 리소스가 계속 남음 (좀비 리소스)
   - 클러스터가 점점 부정확해짐
```

**Pruning 활성화 (위험함, 주의필수):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      prune: true  # ← Git에서 삭제된 리소스 자동 제거
```

```bash
# 시나리오
1. Git에서 ConfigMap 삭제
$ git rm config/prod/configmap.yaml
$ git push

2. ArgoCD Sync 시 (또는 자동 배포)
# Git에 없는 ConfigMap을 클러스터에서도 삭제!
$ kubectl delete configmap myapp-config -n production

3. 앱이 ConfigMap에서 설정을 읽으면
# 설정을 못 찾아서 오류 발생!

4. 권장: 필요할 때만 활성화, Dry-Run으로 확인
$ argocd app sync myapp --prune --dry-run
# 실제로는 실행하지 않고, 뭐가 삭제될지만 보여줌
```

### 4. Sync Wave — 순서 제어

**웨이브 개념:**

```
Wave -2: 초기화 작업 (ConfigMap, Secret 미리 생성)
   ↓
Wave -1: DB 마이그레이션
   ↓
Wave 0: 일반 리소스 배포 (Deployment, Service 등)
   ↓
Wave 1: 의존하는 리소스 (Ingress 등)
   ↓
Wave 2: 헬스 체크 (Job, Pod)
```

**구현 방법:**

```yaml
# Wave -1: DB 마이그레이션 (첫 번째 실행)
apiVersion: batch/v1
kind: Job
metadata:
  name: flyway-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # ← 가장 먼저 실행
spec:
  template:
    spec:
      containers:
      - name: flyway
        image: flyway:latest
        command:
        - sh
        - -c
        - flyway migrate
      restartPolicy: Never

---
# Wave 0: 애플리케이션 배포 (기본값)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # 또는 생략 (기본값)
spec:
  replicas: 3
  # ...

---
# Wave 0: 서비스 (같은 웨이브에서 병렬 처리)
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  # ...

---
# Wave 1: Ingress (Deployment, Service 완료 후)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # ← 마지막에 실행
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 8080
```

**실행 순서:**

```
1. Wave -1의 모든 리소스 생성 (Job 완료 대기)
2. Wave 0의 모든 리소스 병렬 생성 (Deployment, Service)
3. Wave 0의 모든 리소스가 Ready 상태로 대기
4. Wave 1의 모든 리소스 생성 (Ingress)
5. 배포 완료
```

### 5. Sync Phase — 배포 훅 (Hook)

**PreSync, Sync, PostSync:**

```yaml
# PreSync: 배포 전 준비 작업
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-deploy-setup
  annotations:
    argocd.argoproj.io/hook: PreSync  # ← Sync 전 실행
    argocd.argoproj.io/sync-wave: "-1"  # 웨이브와 함께 사용 가능
spec:
  template:
    spec:
      serviceAccountName: deployer
      containers:
      - name: setup
        image: setup:latest
        command: ["sh", "-c", "echo 'Preparing deployment...'"]
      restartPolicy: Never

---
# Sync: 일반 배포 (훅 아님)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  # ...

---
# PostSync: 배포 후 검증 작업
apiVersion: batch/v1
kind: Job
metadata:
  name: post-deploy-verify
  annotations:
    argocd.argoproj.io/hook: PostSync  # ← Sync 후 실행
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # 성공 시 자동 삭제
spec:
  template:
    spec:
      serviceAccountName: verifier
      containers:
      - name: verify
        image: curl:latest
        command:
        - sh
        - -c
        - |
          sleep 30  # 앱 시작 대기
          curl -f http://myapp:8080/health || exit 1
      restartPolicy: Never
  backoffLimit: 3
```

**Hook 삭제 정책:**

```yaml
# 1. HookSucceeded: 성공한 훅만 자동 삭제
argocd.argoproj.io/hook-delete-policy: HookSucceeded

# 2. HookFailed: 실패한 훅만 자동 삭제
argocd.argoproj.io/hook-delete-policy: HookFailed

# 3. BeforeHookCreation: 새 훅 생성 전에 이전 것 삭제
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation

# 4. 삭제 안 함 (기본값)
# (아무것도 지정 안 하면 Job이 클러스터에 남음)
```

## 💻 실전 실험

### 실험 1: Manual vs Automated Sync 비교

```bash
# 1. Manual Sync Application 생성
$ cat > myapp-manual.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-manual
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
    # automated 없음 = Manual
EOF
$ kubectl apply -f myapp-manual.yaml

# 2. Git 변경
$ git checkout -b test/manual-sync
$ sed -i 's/replicas: 3/replicas: 5/' config/prod/deployment.yaml
$ git add . && git commit -m "Scale to 5" && git push

# 3. Application 상태 확인
$ argocd app get myapp-manual
Status: OutOfSync  # Git 변경 감지, 하지만 자동 배포 안 함

# 4. Diff 확인
$ argocd app diff myapp-manual
spec.replicas: 3 vs 5

# 5. 수동으로 배포
$ argocd app sync myapp-manual
Syncing...
SYNCED

# 6. 상태 확인
$ argocd app get myapp-manual
Status: Synced
```

```bash
# 7. Automated Sync Application 생성
$ cat > myapp-auto.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-auto
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
      prune: false
      selfHeal: true
EOF
$ kubectl apply -f myapp-auto.yaml

# 8. Git 변경 (다시)
$ sed -i 's/replicas: 5/replicas: 7/' config/prod/deployment.yaml
$ git add . && git commit -m "Scale to 7" && git push

# 9. 즉시 상태 확인
$ argocd app get myapp-auto
Status: OutOfSync  # 아직 감지 안 됨 (폴링 주기 아님)

# 10. Webhook으로 즉시 감지 시뮬레이션
$ argocd app wait myapp-auto --operation  # Webhook 대기 또는
$ sleep 5 && argocd app get myapp-auto --refresh
Status: Synced  # 자동으로 배포됨!

# 11. 비교
# Manual: 배포 전 diff 확인, 수동 개입 필요
# Auto: 자동 배포, diff 확인 없음
```

### 실험 2: Self-Heal 효과

```bash
# 1. Application 생성 (Self-Heal 활성화)
$ argocd app create myapp-heal \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --auto-prune \
  --self-heal

# 2. 현재 replicas 확인 (Git: 3)
$ kubectl get deployment app -n production -o jsonpath='{.spec.replicas}'
3

# 3. 운영자가 수동으로 스케일업
$ kubectl scale deployment app -n production --replicas=10
deployment.apps/app scaled

# 4. 확인
$ kubectl get deployment app -n production -o jsonpath='{.spec.replicas}'
10

# 5. ArgoCD가 감지
$ argocd app get myapp-heal
Status: OutOfSync

# 6. 폴링 주기 또는 수동 refresh 대기
$ sleep 180  # 3분 폴링 주기 대기
# 또는 수동으로 강제 refresh
$ argocd app get myapp-heal --refresh

# 7. Self-Heal 자동 실행
$ kubectl get deployment app -n production -o jsonpath='{.spec.replicas}'
3  # ← 자동으로 3으로 복원됨!

# 8. 상태 확인
$ argocd app get myapp-heal
Status: Synced
```

### 실험 3: Pruning 안전성

```bash
# 1. Application 생성 (Pruning 비활성화 - 안전)
$ argocd app create myapp-no-prune \
  --repo https://github.com/company/app-config \
  --path config/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# 2. ConfigMap이 Git에 있는지 확인
$ git ls-tree -r main config/prod/ | grep configmap
abc1234 blob xyz configmap.yaml

# 3. 클러스터에 배포
$ argocd app sync myapp-no-prune
SYNCED

$ kubectl get configmap -n production
myapp-config

# 4. Git에서 ConfigMap 삭제
$ git rm config/prod/configmap.yaml
$ git add . && git commit -m "Remove old config"
$ git push

# 5. 다시 동기화
$ argocd app sync myapp-no-prune
SYNCED

# 6. ConfigMap이 여전히 클러스터에 있나?
$ kubectl get configmap -n production
myapp-config  # ← 여전히 존재! (Pruning 비활성화)
```

### 실험 4: Sync Wave 순서 확인

```yaml
# config/prod/waves.yaml
---
# Wave -1
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: busybox
        command: ["sh", "-c", "echo 'DB Migration' && sleep 2"]
      restartPolicy: Never
  backoffLimit: 1

---
# Wave 0
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  annotations:
    argocd.argoproj.io/sync-wave: "0"
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
        image: busybox
        command: ["sh", "-c", "sleep 1000"]

---
# Wave 1
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: test
        image: busybox
        command: ["sh", "-c", "echo 'Smoke Test' && sleep 2"]
      restartPolicy: Never
  backoffLimit: 1
```

```bash
# 배포 후 순서 확인
$ argocd app sync myapp
# 로그에서 순서 보기
# 1. Job db-migrate (wave -1) 생성
# 2. Job 완료 대기
# 3. Deployment app (wave 0) 생성
# 4. Pod Ready 대기
# 5. Job smoke-test (wave 1) 생성
# 6. 완료

# Job 생성 시간으로 순서 확인
$ kubectl get job -n production --sort-by=.metadata.creationTimestamp
NAME          COMPLETIONS   DURATION   AGE
db-migrate    1/1           2s         5m
smoke-test    1/1           2s         3m
# db-migrate가 먼저, smoke-test가 나중에 생성됨
```

## 📊 성능/비용 비교

| 항목 | Manual | Automated |
|------|--------|-----------|
| **배포 시간** | 3분+ (수동 대기) | 수초 (자동) |
| **실수 위험** | 낮음 (diff 확인) | 높음 (자동) |
| **운영 수고** | 많음 (매번 sync) | 없음 |
| **긴급 대응** | 가능 (kubectl edit) | 불가능 (자동 복원) |
| **리소스 정리** | 수동 | 자동 (Pruning) |

## ⚖️ 트레이드오프

| 선택지 | 자동화 수준 | 안정성 | 권장 환경 |
|--------|----------|--------|---------|
| Manual Sync + Pruning=false | 낮음 | 높음 | Production |
| Automated + Pruning=false + SelfHeal=false | 중간 | 중간 | Staging |
| Automated + Pruning=true + SelfHeal=true | 높음 | 낮음 | Dev |

## 📌 핵심 정리

1. **Manual Sync**: 배포 전 확인, 실수 방지 (Prod 권장)
2. **Automated Sync**: 빠른 배포, 자동화 (Dev 권장)
3. **Self-Heal**: kubectl edit 변경 자동 복원, 운영자 의도 무시 가능
4. **Pruning**: Git에서 삭제된 리소스 자동 제거, 위험하므로 신중히
5. **Sync Wave**: PreSync → Sync → PostSync 순서로 배포 제어
6. **Sync Phase**: 배포 전/후 작업 (DB 마이그레이션, 헬스 체크)

## 🤔 생각해볼 문제

### Q1. Prod는 Manual Sync, Dev는 Automated인데, 둘 다 같은 Git 저장소에서 관리하면?

<details>
<summary>💡 해설</summary>

**구조:**
```
app-config/
├── config/
│   ├── dev/
│   │   └── application.yaml  # Automated Sync
│   ├── staging/
│   │   └── application.yaml  # Automated Sync
│   └── prod/
│       └── application.yaml  # Manual Sync
```

**구현:**
```yaml
# dev/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# prod/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  syncPolicy:
    # automated 없음 = Manual
```

**이점:**
- 환경별로 다른 정책 적용 가능
- Git 구조가 명확함
- 각 환경이 독립적

</details>

### Q2. Sync Wave=-1, 0, 1이 있는데, Wave 2는 뭘까?

<details>
<summary>💡 해설</summary>

**추가 웨이브의 용도:**

```yaml
# Wave -2: 초기화 (가장 먼저)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    argocd.argoproj.io/sync-wave: "-2"

---
# Wave -1: 의존성 (ConfigMap, Secret)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

---
# Wave 0: 일반 리소스
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# Wave 1: 진입점 (Service, Ingress)
apiVersion: v1
kind: Service
metadata:
  name: app
  annotations:
    argocd.argoproj.io/sync-wave: "1"

---
# Wave 2: 외부 연동 (제3자 시스템)
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-external-system
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    spec:
      containers:
      - name: notify
        image: curl
        command: ["curl", "-X", "POST", "https://external-system.com/api/deployed"]
      restartPolicy: Never
```

**순서:**
```
Wave -2 → Wave -1 → Wave 0 → Wave 1 → Wave 2
Namespace → Config → App → Service → External notify
```

</details>

### Q3. PostSync Job이 실패하면 배포가 실패한 거나?

<details>
<summary>💡 해설</summary>

**동작:**

```bash
# PostSync Job이 실패하면

# 1. Deployment는 이미 배포됨 (Progressing 또는 Synced)
$ kubectl get deployment myapp -n production
NAME   READY   UP-TO-DATE   AVAILABLE
myapp  3/3     3            3

# 2. PostSync Job만 실패
$ kubectl get job -n production
NAME              COMPLETIONS   STATUS
post-deploy-test  0/1           Failed

# 3. ArgoCD Application 상태는?
$ argocd app get myapp-prod
Status: Degraded  # ← 배포 실패!
# (또는 상황에 따라 Synced)

# Health status: Degraded
# Sync Status: Synced
# → 배포는 되었지만 검증 실패
```

**처리:**
```bash
# 1. PostSync Job 로그 확인
$ kubectl logs job/post-deploy-test -n production
curl: (7) Failed to connect to myapp:8080

# 2. 이유 파악
# - 앱이 아직 준비 안 됨
# - 네트워크 오류
# - 등등

# 3. 재시도
$ argocd app sync myapp-prod  # 다시 배포 (PreSync, Sync, PostSync 반복)

# 또는 수동으로 Job 재실행
$ kubectl delete job post-deploy-test -n production
$ kubectl apply -f post-deploy-test-job.yaml
```

**권장: PostSync의 Sleep 추가**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-test
  annotations:
    argocd.argoproj.io/hook: PostSync
spec:
  template:
    spec:
      containers:
      - name: test
        image: busybox
        command: ["sh", "-c", "sleep 30 && curl -f http://myapp:8080/health"]  # ← 30초 대기
      restartPolicy: Never
  backoffLimit: 3
```

</details>

---

[⬅️ 이전](./02-argocd-architecture.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./04-kustomize-helm-integration.md)
