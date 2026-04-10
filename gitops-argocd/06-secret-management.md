# Secret 관리 — Git에 저장하면 안 되는 이유

## 🎯 핵심 질문

- Git 히스토리에 평문 Secret이 한 번 커밋되면 진짜 완전히 제거 불가능한가?
- Sealed Secrets는 어떤 암호화 방식으로 Git에 저장되나?
- AWS Secrets Manager, Vault, SOPS 중 어떤 걸 선택해야 하나?
- Secret을 환경별로 다르게 관리할 수 있나?

## 🔍 왜 이 개념이 실무에서 중요한가

GitOps는 **모든 설정을 Git에 저장**하는 원칙이지만, Secret(데이터베이스 암호, API 키, TLS 인증서)은 **절대로 평문으로 Git에 저장하면 안 됩니다**:

1. **Git 히스토리는 영구적**: `git log`에 한 번 기록되면 `git filter-branch`로도 완전 제거 어려움
2. **Public 저장소 위험**: 실수로 공개 저장소에 푸시하면 누구나 접근 가능
3. **감사 추적성**: Secret 변경이 Git에 로그로 남으면 너무 민감한 정보 노출

**대안들:**
- **Sealed Secrets**: 클러스터의 공개키로 암호화 → Git에 저장
- **External Secrets Operator**: AWS Secrets Manager/Vault와 동기화
- **Vault Agent Injector**: Pod 사이드카로 런타임에 Secret 주입
- **SOPS**: KMS/GPG로 파일 암호화

## 😱 흔한 실수 (Before — Secret 관리 안 할 때)

### Before: 평문 Secret을 Git에 커밋

```bash
# ❌ 절대 금지: 평문 Secret을 Git에 저장
$ cat > config/secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
stringData:
  username: admin
  password: SuperSecret123!  # ← 평문!
  connection-string: "postgres://admin:SuperSecret123!@db.internal:5432/mydb"
EOF

# Git에 커밋 (실수!)
$ git add config/secret.yaml
$ git commit -m "Add database secret"
$ git push origin main

# 나중에 Secret 변경 필요
$ git rm config/secret.yaml
$ git commit -m "Remove database secret"
$ git push

# 하지만 이미 늦음!
# 누군가 git log를 보면:
$ git log --all -- config/secret.yaml
# 이전 커밋에서 평문 Secret을 볼 수 있음

# git filter-branch로 제거 시도
$ git filter-branch --tree-filter 'rm -f config/secret.yaml' HEAD
# But: 
# 1. 강제 푸시 필요 (팀 전체 재작업)
# 2. GitHub 자체적으로도 히스토리 보관 가능
# 3. 누군가 이미 복사했을 수도 있음!
```

**결과:** Secret 완전 탈취!

### Before: 환경별 Secret을 수동으로 관리

```bash
# ❌ 나쁜 패턴: Secret을 Git에서 제외, 수동으로 관리

# .gitignore
secrets/
.env*

# 그러면 배포 시 어떻게?
# 1. CI 파이프라인에서 Secret을 클러스터에 직접 주입?
# 2. Operator가 수동으로 Secret을 만들기?
# 3. Secret이 누락되어 배포 실패?

# 문제:
# - Secret 변경 이력 추적 불가능 (감사 추적성 없음)
# - 누가 언제 뭘 변경했는지 알 수 없음
# - 팀원들이 Secret을 직접 관리 (보안 위험)
# - 새 환경 구성 시 Secret 재생성 (실수 가능)
```

## ✨ 올바른 접근 (After — Secret을 암호화해서 Git에 저장)

### After: Sealed Secrets로 암호화 저장

```bash
# ✅ 올바른 방법: Sealed Secrets

# 1. 평문 Secret 작성 (임시)
$ cat > secret-plain.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-password
  namespace: production
type: Opaque
stringData:
  username: admin
  password: SuperSecret123!
EOF

# 2. sealing (암호화)
$ sealedsecrets seal secret-plain.yaml -o yaml > secret-sealed.yaml

# 3. 암호화된 Secret 확인
$ cat secret-sealed.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production
spec:
  encryptedData:
    username: AgBc1a2b3c4d5e...  # ← 클러스터의 공개키로 암호화됨
    password: AgXyz9v8w7u6t...

# 4. 암호화된 파일을 Git에 저장 (안전!)
$ git add secret-sealed.yaml
$ git commit -m "Add encrypted database secret"
$ git push

# 5. 클러스터에 배포하면 자동으로 복호화 (Sealed Secrets 컨트롤러)
$ kubectl apply -f secret-sealed.yaml
sealedsecret.bitnami.com/db-password created

# 6. 클러스터에서는 일반 Secret으로 표현됨
$ kubectl get secret db-password -n production -o yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-password
data:
  username: YWRtaW4=  # base64
  password: U3VwZXJTZWNyZXQxMjMh

# 7. 평문 secret-plain.yaml는 삭제
$ rm secret-plain.yaml
```

### After: External Secrets Operator로 AWS와 동기화

```bash
# ✅ 외부 Secret 저장소 활용

# 1. AWS Secrets Manager에 Secret 저장 (ArgoCD 외부)
$ aws secretsmanager create-secret \
  --name prod/db-password \
  --secret-string '{"username":"admin","password":"SuperSecret123!"}'

# 2. Git에는 참조만 저장
$ cat > config/prod/external-secret.yaml <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  secretStoreRef:
    name: aws-secrets
  target:
    name: db-password  # ← 클러스터에 생성될 Secret 이름
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: prod/db-password
      property: username
  - secretKey: password
    remoteRef:
      key: prod/db-password
      property: password
EOF

# 3. Git에는 암호화된 정보 없음 (참조만)
$ git add config/prod/external-secret.yaml
$ git commit -m "Add external secret reference"
$ git push

# 4. ArgoCD가 배포하면 External Secrets Operator가
#    AWS Secrets Manager에서 실제 값을 가져와서 Secret 생성
```

## 🔬 내부 동작 원리

### 1. Sealed Secrets — 클러스터 기반 암호화

**작동 방식:**

```
클러스터 설치 시 생성되는 키 쌍:
├─ Sealing Key (공개키): 암호화에 사용
└─ Private Key (비밀키): 복호화에 사용
   (클러스터 etcd에만 저장)

워크플로우:
1. 운영자가 sealing 도구 실행 (공개키 사용)
   $ sealedsecrets seal secret.yaml
   
2. 암호화된 SealedSecret 파일 생성
   → Git에 저장 안전!

3. Sealed Secrets 컨트롤러가 cluster에서 실행
   → Pod이 SealedSecret을 읽으면
   → Private Key로 복호화
   → 일반 Secret으로 변환
```

**암호화 범위:**

```yaml
# 특정 Namespace만 복호화 가능
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production  # ← 이 namespace에서만 복호화됨!
spec:
  encryptedData:
    password: AgBc...
```

```bash
# 다른 namespace에서는 복호화 불가능
$ kubectl apply -f sealed-secret.yaml -n staging
# 복호화 실패 → Secret 생성 안 됨 (보안 보장)
```

**장점:**
- 클러스터 기반, 별도 외부 서비스 불필요
- 간단한 설정
- Git에 암호화된 파일 저장 가능

**단점:**
- 클러스터별로 다른 Private Key (환경 분리 어려움)
- Private Key를 클러스터에서 내보내야 할 경우 관리 복잡

### 2. External Secrets Operator — 외부 저장소 연동

**지원하는 저장소:**
- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- HashiCorp Vault
- Google Cloud Secret Manager
- Azure Key Vault
- Kubernetes Secret
- SOPS

**작동 방식:**

```
┌─────────────────────────────────┐
│ External Secret Store (AWS)     │
│ - prod/db-password              │
│ - prod/api-key                  │
│ - prod/tls-cert                 │
└─────────────────────────────────┘
            ↑ (읽기)
            │
┌─────────────────────────────────┐
│ External Secrets Operator       │
│ (클러스터 내부)                  │
│                                 │
│ 정기적으로 외부 저장소에서       │
│ Secret을 동기화                  │
│ (Polling 또는 Webhook)          │
└─────────────────────────────────┘
            ↓ (쓰기)
┌─────────────────────────────────┐
│ Kubernetes Secret               │
│ (클러스터 etcd)                 │
└─────────────────────────────────┘

워크플로우:
1. ExternalSecret 리소스를 Git에 저장
   (참조만, Secret 값은 없음)

2. ArgoCD가 배포
   $ kubectl apply -f external-secret.yaml

3. External Secrets Operator가 감지
   → AWS Secrets Manager에 쿼리
   → 최신 값을 Kubernetes Secret으로 생성

4. Pod이 Secret을 사용
```

**환경별 설정:**

```yaml
# prod 환경: AWS Secrets Manager 사용
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  secretStoreRef:
    name: aws-secrets
  target:
    name: db-password
  data:
  - secretKey: password
    remoteRef:
      key: prod/db-password
      property: password

---
# dev 환경: 로컬 Kubernetes Secret 사용 (AWS 구독 필요 없음)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: dev
spec:
  secretStoreRef:
    name: local-secret-store  # ← 다른 store
  target:
    name: db-password
  data:
  - secretKey: password
    remoteRef:
      key: dev-db-password
```

### 3. Vault Agent Injector — Pod 사이드카 주입

**작동 방식:**

```
┌──────────────────────────────────┐
│ HashiCorp Vault                  │
│ (중앙 Secret 저장소)              │
│ - secret/db-password             │
│ - secret/api-key                 │
└──────────────────────────────────┘
       ↑ (문의)
       │
┌──────────────────────────────────┐
│ Vault Agent Sidecar              │
│ (Pod 옆에서 실행)                │
│                                  │
│ 1. Vault에서 Secret 조회         │
│ 2. Secret 파일로 저장 (/vault/secrets/)
│ 3. Pod 컨테이너에서 접근         │
└──────────────────────────────────┘
       ↑ (마운트)
       │
┌──────────────────────────────────┐
│ Application Pod                  │
│ - 메인 컨테이너 (앱)             │
│ - Vault Agent 사이드카           │
│ - Shared Volume (/vault/secrets) │
└──────────────────────────────────┘
```

**Pod 주입:**

```yaml
# Vault Injector 주석으로 자동 주입
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: myapp
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/database"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/database" -}}
          export DB_USERNAME="{{ .Data.data.username }}"
          export DB_PASSWORD="{{ .Data.data.password }}"
          {{- end }}
    spec:
      containers:
      - name: myapp
        image: myapp:v1.2.3
        command:
        - sh
        - -c
        - |
          source /vault/secrets/database  # ← Vault Agent가 생성한 파일
          ./run.sh  # 환경 변수로 Secret을 받아서 실행
```

**장점:**
- Pod 시작 시마다 최신 Secret 조회 (변경 반영 빠름)
- Secret을 환경 변수나 파일로 유연하게 주입
- Vault에서 감사 추적성 제공

**단점:**
- Vault 구독 필요 (Self-hosted 가능하지만 운영 복잡)
- 추가 sidecar 메모리 사용

### 4. SOPS — 파일 레벨 암호화

**작동 방식:**

```
SOPS(Secrets Operation)는 YAML/JSON 파일 일부만 암호화

원본 파일:
config/secret.yaml
├─ metadata: (평문)
├─ spec:
│  ├─ replicas: (평문)
│  └─ env:
│     └─ DB_PASSWORD: "secret123" (암호화됨)

암호화 도구: AWS KMS 또는 GPG Key
```

**사용법:**

```bash
# 1. SOPS 설정 파일 (.sops.yaml)
$ cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: config/.*
    kms: "arn:aws:kms:us-east-1:123456789:key/12345"
EOF

# 2. SOPS로 암호화
$ sops -e config/secret.yaml > config/secret.enc.yaml

# 3. 암호화된 파일 확인
$ cat config/secret.enc.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-password
  sops:
    kms:
    - arn: arn:aws:kms:us-east-1:123456789:key/12345
      created_at: '2026-04-10T15:00:00Z'
      enc: AQIDAHhz...
      aws_profile: ""
stringData:
  password: ENC[AES256_GCM,data:abc123,iv:xyz,tag:def,type:str]
  username: admin  # 일부만 암호화 가능

# 4. Git에 저장 (암호화된 상태)
$ git add config/secret.enc.yaml
$ git commit -m "Add encrypted database secret"
$ git push

# 5. ArgoCD 배포 시 SOPS 플러그인이 자동 복호화
$ argocd app create myapp --repo ... --path config
# ArgoCD가 secret.enc.yaml을 감지하면
# KMS(AWS)로 복호화해서 배포
```

**장점:**
- 파일의 특정 부분만 암호화 (설정과 Secret 혼합 가능)
- Git 히스토리에서도 암호화 상태 유지
- 특정 도구가 아닌 표준 포맷

**단점:**
- KMS/GPG 키 관리 필수
- ArgoCD에 SOPS 플러그인 설치 필요
- 렌더링 시 KMS/GPG 호출 오버헤드

## 💻 실전 실험

### 실험 1: Sealed Secrets 설치 및 암호화

```bash
# 1. Sealed Secrets 컨트롤러 설치
$ kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# 2. sealedsecrets CLI 설치
$ wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/kubeseal-0.18.0-linux-amd64.tar.gz
$ tar xfz kubeseal-0.18.0-linux-amd64.tar.gz
$ sudo mv kubeseal /usr/local/bin/

# 3. 평문 Secret 작성
$ cat > secret-plain.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-password
  namespace: production
type: Opaque
stringData:
  username: admin
  password: MySecurePassword123!
EOF

# 4. 암호화 (클러스터의 공개키 사용)
$ kubeseal -f secret-plain.yaml -w secret-sealed.yaml

# 5. 암호화된 파일 확인
$ cat secret-sealed.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production
spec:
  encryptedData:
    username: AgBc1234abcdef5678...
    password: AgXyzABC9v8w7u6t...

# 6. 평문 파일 삭제
$ rm secret-plain.yaml

# 7. Git에 암호화된 파일 저장
$ git add secret-sealed.yaml
$ git commit -m "Add encrypted database secret"
$ git push

# 8. ArgoCD가 배포
$ kubectl apply -f secret-sealed.yaml

# 9. 클러스터에서 일반 Secret으로 복호화됨
$ kubectl get secret db-password -n production -o jsonpath='{.data.password}' | base64 -d
MySecurePassword123!
```

### 실험 2: External Secrets Operator로 AWS 연동

```bash
# 1. External Secrets Operator 설치
$ helm repo add external-secrets https://charts.external-secrets.io
$ helm install external-secrets external-secrets/external-secrets \
  -n external-secrets-system --create-namespace

# 2. AWS Secrets Manager에 Secret 저장
$ aws secretsmanager create-secret \
  --name prod/db-password \
  --secret-string '{"username":"admin","password":"SecurePass123"}'

# 3. IAM 역할/정책 설정 (Pod이 AWS에 접근)
# (생략 - IRSA 또는 IAM User 키 필요)

# 4. SecretStore 생성 (AWS 연결)
$ cat > secretstore.yaml <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
EOF
$ kubectl apply -f secretstore.yaml

# 5. ExternalSecret 생성 (참조만)
$ cat > external-secret.yaml <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  refreshInterval: 1h  # 1시간마다 동기화
  secretStoreRef:
    name: aws-secrets
  target:
    name: db-password  # 생성될 Secret 이름
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: prod/db-password
      property: username
  - secretKey: password
    remoteRef:
      key: prod/db-password
      property: password
EOF
$ kubectl apply -f external-secret.yaml

# 6. Git에 저장 (암호화된 정보 없음)
$ git add external-secret.yaml
$ git commit -m "Add external secret reference for AWS"
$ git push

# 7. ArgoCD가 배포하면 자동으로 Secret 생성
$ kubectl get secret db-password -n production -o jsonpath='{.data.password}' | base64 -d
SecurePass123

# 8. AWS에서 Secret 변경
$ aws secretsmanager update-secret \
  --secret-id prod/db-password \
  --secret-string '{"username":"admin","password":"NewPassword456"}'

# 9. 1시간 후 자동으로 동기화 (또는 수동 강제 동기화)
$ kubectl annotate externalsecret db-password -n production \
  force-sync="$(date +%s)" --overwrite
```

### 실험 3: SOPS로 파일 암호화

```bash
# 1. SOPS 설치
$ curl -L https://github.com/mozilla/sops/releases/download/v3.7.0/sops-v3.7.0.linux.amd64 \
  -o /usr/local/bin/sops
$ chmod +x /usr/local/bin/sops

# 2. SOPS 설정 파일
$ cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: config/prod/.*
    kms: "arn:aws:kms:us-east-1:123456789:key/abcdef12-3456-7890-abcd-ef1234567890"
  - path_regex: config/dev/.*
    unencrypted_regex: ^kind$|^apiVersion$|^metadata$
    kms: "arn:aws:kms:us-east-1:123456789:key/dev-key-id"
EOF

# 3. 평문 Secret 작성
$ cat > config/prod/secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
stringData:
  username: admin
  password: SuperSecret123!
EOF

# 4. SOPS로 암호화
$ sops -e config/prod/secret.yaml > config/prod/secret.enc.yaml

# 5. 암호화된 파일 확인
$ cat config/prod/secret.enc.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
stringData:
  username: admin  # ← 일부는 평문 (unencrypted_regex 설정)
  password: ENC[AES256_GCM,data:abc123xyz,iv:def456,tag:ghi789,type:str]
sops:
  kms:
  - arn: arn:aws:kms:us-east-1:123456789:key/abcdef12...
    created_at: '2026-04-10T15:00:00Z'
    enc: AQIDAH9aMxyz...

# 6. Git에 저장
$ git add config/prod/secret.enc.yaml
$ git commit -m "Add encrypted database secret with SOPS"
$ git push

# 7. ArgoCD에 SOPS 플러그인 설정
# (argocd-repo-server에 sops 바이너리와 KMS 권한 필요)

# 8. ArgoCD가 배포 시 자동 복호화
$ argocd app create myapp --repo ... --path config/prod
# ArgoCD의 Repo Server가:
# 1. secret.enc.yaml 감지
# 2. SOPS로 복호화 (AWS KMS 호출)
# 3. 평문 Secret으로 변환 후 배포
```

## 📊 성능/비용 비교

| 항목 | Sealed Secrets | External Secrets | Vault | SOPS |
|------|----------------|------------------|-------|------|
| **설정 복잡도** | 낮음 | 중간 | 높음 | 중간 |
| **외부 서비스** | 불필요 | 필요 (AWS/GCP/Azure) | 필수 | 필요 (KMS/GPG) |
| **감사 추적** | 클러스터 etcd | 외부 저장소 | Vault logs | Git + KMS 로그 |
| **환경별 관리** | 클러스터마다 다른 키 | 저장소에서 관리 | Vault에서 관리 | 키별 다른 폴더 |
| **렌더링 성능** | 빠름 | 중간 (외부 호출) | 중간 | 느림 (KMS 호출) |
| **비용** | 낮음 (추가 비용 없음) | 중간 (저장소 비용) | 높음 (Vault Enterprise) | 낮음~중간 |

## ⚖️ 트레이드오프

| 선택지 | 주요 특징 | 권장 환경 |
|--------|---------|----------|
| Sealed Secrets | 간단, 클러스터 자체 포함 | 소규모 팀, Single cluster |
| External Secrets | 강력한 통합, 중앙 관리 | Enterprise, Multi-cluster |
| Vault | 완전한 Secret 관리, 복잡 | 매우 큰 조직 |
| SOPS | 유연한 암호화, Git 친화적 | 팀에 KMS 접근 가능 |

## 📌 핵심 정리

1. **평문 Secret 금지**: Git에는 절대 평문 Secret 저장하면 안 됨
2. **Sealed Secrets**: 클러스터 공개키로 암호화, 가장 간단한 방법
3. **External Secrets**: AWS/Vault와 실시간 동기화, 중앙 관리
4. **Vault**: 엔터프라이즈급 Secret 관리, 복잡하지만 강력함
5. **SOPS**: 파일 레벨 암호화, KMS/GPG로 유연한 관리
6. **감사 추적성**: Secret 변경도 Git 또는 저장소에서 추적 가능해야 함

## 🤔 생각해볼 문제

### Q1. Sealed Secret의 Private Key가 노출되면 어떻게 할까?

<details>
<summary>💡 해설</summary>

**위험 상황:**
- 클러스터 etcd 백업이 도둑맞음
- Private Key 파일이 실수로 유출됨
- 운영자가 Private Key를 외부에 공개함

**해결 방법:**

```bash
# 1. 새로운 키 쌍 생성 및 적용
$ kubectl delete secret -l sealedsecrets.bitnami.com/status=active -n kube-system

# 새 Secret이 자동으로 생성됨 (새로운 키 쌍)

# 2. 모든 SealedSecret을 새로운 공개키로 다시 암호화
$ for sealed in $(kubectl get sealedsecrets -A -o name); do
  # 각 SealedSecret을 re-encrypt
  # (이 과정은 복잡하므로 주의 필요)
done

# 3. Git의 모든 SealedSecret을 재생성
$ for f in $(find . -name "*sealed*" -type f); do
  # 새로운 공개키로 다시 seal
done

# 4. 모든 변경사항을 새 Private Key 생성으로 다시 적용
```

**권장: 정기적 키 로테이션**

```bash
# Sealed Secrets 컨트롤러 설정에서 키 로테이션 주기 설정
# (ROTATION_PERIOD 환경 변수)
```

</details>

### Q2. Dev와 Prod 환경이 다른 Sealed Secret Key를 쓸 수 있나?

<details>
<summary>💡 해설</summary>

**기본 설정: 같은 키**

```bash
# 한 클러스터에 한 쌍의 Sealed Secrets 키
# → 모든 SealedSecret이 같은 Private Key로 복호화
```

**서로 다른 키 사용:**

```bash
# 1. Dev 환경 (별도 키)
$ KUSER=dev kubeseal -f secret.yaml
# dev-public-key로 암호화됨

# 2. Prod 환경 (별도 키)
$ KUSER=prod kubeseal -f secret.yaml
# prod-public-key로 암호화됨

# 3. 각 환경의 Private Key는 다름
# dev-private-key (dev 클러스터만 가짐)
# prod-private-key (prod 클러스터만 가짐)

# 결과: 한 SealedSecret이 특정 환경에서만 복호화 가능!
```

**고급: Namespaced Sealing**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: production
spec:
  encryptedData:
    password: AgBc...
  # scope: namespace (기본값)
  # → production namespace에서만 복호화됨!
```

</details>

### Q3. Secret이 자주 바뀌면 Git 커밋이 너무 많아지지 않나?

<details>
<summary>💡 해설</summary>

**문제:** 예를 들어 API 토큰이 매주 로테이션되면?

```bash
# Week 1
$ git commit -m "Update API token token123"

# Week 2
$ git commit -m "Update API token token456"

# Week 3
$ git commit -m "Update API token token789"

# → Git 히스토리가 Secret 변경으로 가득 참
# → 감사 추적성도 떨어짐 (뭐가 진짜 배포 변경인지 알 수 없음)
```

**해결책: External Secrets 사용**

```yaml
# Git에는 참조만 저장
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-token
spec:
  secretStoreRef:
    name: aws-secrets
  refreshInterval: 1h  # 1시간마다 자동 동기화
  data:
  - secretKey: token
    remoteRef:
      key: api-token
```

```bash
# Git 커밋은 한 번만
$ git commit -m "Add external secret reference for API token"

# AWS Secrets Manager에서 토큰 로테이션
$ aws secretsmanager update-secret --secret-id api-token --secret-string "new-token"

# 1시간 후 자동으로 동기화 (Git 커밋 없음!)
# External Secrets Operator가 AWS에서 최신 토큰 조회
# → Kubernetes Secret 업데이트
# → Pod이 자동으로 새 토큰 사용
```

**또는 Vault와 TTL (Time-To-Live)**

```yaml
# Vault에서 동적 Secret 생성
# - TTL: 24시간
# - 자동으로 새 Secret 생성 및 이전 Secret 제거
# → Git 커밋 불필요
```

</details>

---

[⬅️ 이전](./05-multi-cluster-management.md) | [홈으로 🏠](../README.md) | [다음: Chapter 6 — 테스트 피라미드와 Pipeline ➡️](../test-automation/01-test-pyramid-pipeline.md)
