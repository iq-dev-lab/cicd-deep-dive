# 환경 관리 전략 — Dev/Staging/Prod 분리

## 🎯 핵심 질문

- "Staging과 Production의 차이는 뭐예요?" — 각 환경의 역할은?
- "개발자가 마음대로 Staging에 배포하면 안 되나요?" — 보호 규칙의 필요성?
- "데이터베이스는 각 환경별로 다르게 설정해야 하나요?" — 환경별 설정 전략?
- "PR마다 테스트 환경을 자동으로 만들 수 있을까요?" — Feature 환경 구축?
- "Staging 설정이 Production과 달라져서 실패했어요" — 환경 Drift 감지?

## 🔍 왜 이 개념이 실무에서 중요한가

많은 팀이 **"로컬 개발 → 바로 프로덕션"** 또는 **"Staging도 있지만 대충 운영"** 같은 방식으로 운영한다. 이것은 매우 위험하다.

실제 사례:
- **Staging이 Production과 달랐다**: 메모리 부족으로 Staging에서는 실패하지 않았지만 Production에서 크래시. 장애 20분.
- **환경별 설정을 하드코딩했다**: DNS, API Key를 Dockerfile에 넣음. 본의 아니게 공개.
- **PR마다 테스트 환경이 없었다**: 개발자가 로컬에서만 테스트 → Production에서 실패.
- **배포 권한 통제 불가**: 누구나 언제든 Staging 배포 가능 → 서로 다른 버전이 배포되는 혼란.

**핵심 원칙:**
```
로컬 < Dev < Staging (= Production) < Production
         ↑                      ↑
    빠른 개발          실제와 동일한 환경
```

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# Before: 환경 분리 없음
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 환경이 고정됨 (Production이나 dev나 같음)
      - name: Deploy
        env:
          DATABASE_URL: "postgres://prod-db.example.com"
          API_KEY: "secret-key-123"  # 하드코딩!
          REDIS_URL: "redis://prod-cache.example.com"
        run: |
          kubectl apply -f deploy.yaml
          
      # 누가 배포했는지, 언제 배포했는지 추적 불가
      # 배포 권한 없음
      # Staging에서 테스트할 환경 없음
```

**문제점:**
1. 환경 변수가 코드에 하드코딩됨
2. 배포 권한 통제 없음
3. 각 환경에 맞는 설정을 따로 관리할 수 없음
4. Staging이 실제 환경과 다를 수 있음
5. 데이터베이스 마이그레이션을 어느 환경부터 할지 불명확

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# After: 환경별 분리 및 보호 규칙 적용
name: Multi-Environment Deployment

on:
  push:
    branches:
      - main
      - develop
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options:
          - development
          - staging
          - production

jobs:
  # 환경 결정 (branch 또는 수동 선택)
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      env-name: ${{ steps.env.outputs.name }}
      env-url: ${{ steps.env.outputs.url }}
      requires-approval: ${{ steps.env.outputs.requires-approval }}
    steps:
      - name: Determine environment
        id: env
        run: |
          ENV_NAME="development"
          ENV_URL="https://dev-api.example.com"
          REQUIRES_APPROVAL="false"
          
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            ENV_NAME="staging"
            ENV_URL="https://staging-api.example.com"
            REQUIRES_APPROVAL="false"
          fi
          
          if [ "${{ github.event_name }}" == "workflow_dispatch" ] && \
             [ "${{ github.event.inputs.environment }}" == "production" ]; then
            ENV_NAME="production"
            ENV_URL="https://api.example.com"
            REQUIRES_APPROVAL="true"
          fi
          
          echo "name=$ENV_NAME" >> $GITHUB_OUTPUT
          echo "url=$ENV_URL" >> $GITHUB_OUTPUT
          echo "requires-approval=$REQUIRES_APPROVAL" >> $GITHUB_OUTPUT

  # Dev 환경 배포 (자동)
  deploy-to-dev:
    if: github.ref == 'refs/heads/develop' || github.event.inputs.environment == 'development'
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Development
        run: |
          echo "Deploying to Development environment"
          kubectl set image deployment/api api=myapp:${{ github.sha }} \
            --namespace=development
        env:
          # 개발 환경 설정 (Secret에서)
          DATABASE_URL: ${{ secrets.DEV_DATABASE_URL }}
          REDIS_URL: ${{ secrets.DEV_REDIS_URL }}
          API_KEY: ${{ secrets.DEV_API_KEY }}
          LOG_LEVEL: "debug"
      
      - name: Run smoke tests in Dev
        run: |
          curl -f http://dev-api.example.com/health

  # Staging 환경 배포 (자동, Production과 동일한 사양)
  deploy-to-staging:
    if: github.ref == 'refs/heads/main' || github.event.inputs.environment == 'staging'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging-api.example.com
    steps:
      - uses: actions/checkout@v4
      
      # Staging 환경은 Production과 동일한 설정 사용
      - name: Deploy to Staging
        run: |
          echo "Deploying to Staging (Production-like) environment"
          
          # 스테이징도 프로덕션과 같은 메모리, CPU 설정
          kubectl set resources deployment/api \
            --requests=cpu=1,memory=1Gi \
            --limits=cpu=2,memory=2Gi \
            --namespace=staging
          
          kubectl set image deployment/api api=myapp:${{ github.sha }} \
            --namespace=staging
        env:
          # 스테이징 환경 설정 (프로덕션과 동일)
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
          REDIS_URL: ${{ secrets.STAGING_REDIS_URL }}
          API_KEY: ${{ secrets.STAGING_API_KEY }}
          LOG_LEVEL: "info"  # 프로덕션과 동일
          ENABLE_PROFILING: "true"  # 성능 분석용
      
      # Staging에서 통합 테스트 실행
      - name: Run integration tests in Staging
        run: |
          npm run test:integration
        env:
          TEST_ENV_URL: https://staging-api.example.com
      
      # Staging 설정과 Production 설정 비교
      - name: Check for environment drift
        run: |
          echo "=== Checking for Configuration Drift ==="
          
          # Staging과 Production의 Deployment 정의 비교
          kubectl get deployment api -n staging -o json > staging-deployment.json
          kubectl get deployment api -n production -o json > prod-deployment.json
          
          # image 버전만 비교 (다른 것은 같아야 함)
          STAGING_IMAGE=$(jq '.spec.template.spec.containers[0].image' staging-deployment.json)
          PROD_IMAGE=$(jq '.spec.template.spec.containers[0].image' prod-deployment.json)
          
          echo "Staging image: $STAGING_IMAGE"
          echo "Production image: $PROD_IMAGE"
          
          # 리소스 비교
          STAGING_CPU=$(jq '.spec.template.spec.containers[0].resources.limits.cpu' staging-deployment.json)
          PROD_CPU=$(jq '.spec.template.spec.containers[0].resources.limits.cpu' prod-deployment.json)
          
          if [ "$STAGING_CPU" != "$PROD_CPU" ]; then
            echo "⚠️  WARNING: CPU limits differ between Staging and Production"
            echo "Staging: $STAGING_CPU, Production: $PROD_CPU"
          fi

  # Production 환경 배포 (수동 승인 필요)
  deploy-to-production:
    if: github.event.inputs.environment == 'production'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.example.com
    steps:
      - uses: actions/checkout@v4
      
      # Production 배포 전 체크리스트
      - name: Pre-deployment checklist
        run: |
          echo "=== Production Deployment Checklist ==="
          echo ""
          echo "1. Tag 검증"
          git describe --tags --exact-match ${{ github.sha }} || {
            echo "❌ This commit is not tagged. Production requires a tag."
            exit 1
          }
          echo "✅ Commit is tagged"
          
          echo ""
          echo "2. Staging에서 최근 테스트 확인"
          # Staging에서 최근 배포 이후 실패 없는지 확인
          TEST_COUNT=$(gh run list \
            --repo ${{ github.repository }} \
            --workflow staging.yml \
            --limit 1 \
            --json conclusion \
            --jq '.[0].conclusion')
          
          if [ "$TEST_COUNT" != "success" ]; then
            echo "❌ Recent Staging deployment failed"
            exit 1
          fi
          echo "✅ Staging tests passed"
      
      - name: Deploy to Production
        run: |
          echo "Deploying to Production..."
          
          # Blue-green 배포 (무중단 배포)
          # 1. 새 이미지로 blue deployment 생성
          kubectl set image deployment/api-blue api=myapp:${{ github.sha }} \
            --namespace=production
          
          # 2. blue 배포의 health 확인
          kubectl rollout status deployment/api-blue -n production --timeout=5m
          
          # 3. 트래픽 전환 (green → blue)
          kubectl patch service api -n production \
            -p '{"spec":{"selector":{"deployment":"blue"}}}'
          
          echo "Production deployment completed"
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
          REDIS_URL: ${{ secrets.PROD_REDIS_URL }}
          API_KEY: ${{ secrets.PROD_API_KEY }}
          LOG_LEVEL: "warn"
      
      - name: Verify production health
        run: |
          # 3분 동안 health check
          for i in {1..18}; do
            if curl -f https://api.example.com/health; then
              echo "✅ Production is healthy"
              exit 0
            fi
            sleep 10
          done
          echo "❌ Production health check failed"
          exit 1
      
      - name: Notify deployment
        if: always()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-Type: application/json' \
            -d @- <<EOF
          {
            "blocks": [{
              "type": "section",
              "text": {
                "type": "mrkdwn",
                "text": "*Production Deployment*\nCommit: ${{ github.sha }}\nStatus: ${{ job.status }}"
              }
            }]
          }
          EOF

  # PR마다 자동 생성되는 Feature 환경 (선택사항)
  feature-environment:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    environment:
      name: pr-${{ github.event.number }}
      url: https://pr-${{ github.event.number }}.example.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Create feature environment
        run: |
          PR_NUMBER=${{ github.event.number }}
          NAMESPACE="pr-$PR_NUMBER"
          
          # 새로운 namespace 생성
          kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
          
          # 설정 복사
          kubectl cp default:database-config $NAMESPACE:database-config || true
          
          # 배포
          kustomize edit set image api=myapp:${{ github.sha }}
          kubectl apply -f deploy.yaml -n $NAMESPACE
          
          echo "Feature environment created: pr-$PR_NUMBER"
      
      - name: Run tests in feature environment
        run: |
          npm run test:integration
        env:
          TEST_ENV_URL: https://pr-${{ github.event.number }}.example.com
      
      - name: Comment PR with environment URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Feature environment created: https://pr-${{ github.event.number }}.example.com'
            });
      
      - name: Cleanup on PR close
        if: github.event.action == 'closed'
        run: |
          PR_NUMBER=${{ github.event.number }}
          kubectl delete namespace pr-$PR_NUMBER --ignore-not-found=true
          echo "Feature environment cleaned up"
```

## 🔬 내부 동작 원리

### GitHub Environments 구조

```
Repository
└── Environments
    ├── development
    │   ├── Secrets (DEV_DATABASE_URL, DEV_API_KEY, ...)
    │   ├── Variables (LOG_LEVEL: debug)
    │   └── Protection rules (권한 설정)
    ├── staging
    │   ├── Secrets (STAGING_DATABASE_URL, ...)
    │   ├── Variables (LOG_LEVEL: info)
    │   └── Protection rules (승인자 지정 가능)
    └── production
        ├── Secrets (PROD_DATABASE_URL, ...)
        ├── Variables (LOG_LEVEL: warn)
        └── Protection rules (반드시 승인자 필요)
```

**보호 규칙 설정:**
1. **배포 보호 규칙**: "main 브랜치에서만 배포 허용"
2. **필수 검수자**: "2명 이상 승인 필요"
3. **타이머**: "최대 72시간 승인 대기"

### 환경별 Secret 관리 구조

```yaml
# Repository Settings → Secrets and variables → Actions

Secrets (암호화됨):
├── PROD_DATABASE_URL=postgres://prod-db...
├── STAGING_DATABASE_URL=postgres://staging-db...
├── DEV_DATABASE_URL=postgres://localhost...
└── API_KEYS...

Variables (평문, 공개 가능):
├── PROD_LOG_LEVEL=warn
├── STAGING_LOG_LEVEL=info
└── DEV_LOG_LEVEL=debug
```

**접근 방법:**
```yaml
env:
  # Environment-specific Secret
  DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
  # Environment-specific Variable
  LOG_LEVEL: ${{ vars.STAGING_LOG_LEVEL }}
```

### 환경 Drift 감지 메커니즘

```
Desired State (코드)
  ↓
실제 Staging 상태
  ↓ (비교)
실제 Production 상태

일치하지 않으면 Drift 감지 후 알림
```

**Drift 원인:**
1. 수동 배포/설정 변경
2. Hot-fix 패치
3. Kubernetes 자동 스케일링
4. 외부 시스템 변경

## 💻 실전 실험 (GitHub Actions YAML, CLI 명령어로 재현 가능)

### 실험 1: GitHub Environments 설정

```bash
# 1. 웹 UI에서 설정
# Repository Settings → Environments → New environment

# 2. 또는 terraform으로 자동화
resource "github_repository_environment" "production" {
  repository       = "my-repo"
  environment      = "production"
  
  reviewers {
    teams = [github_team.devops.id]
  }
}
```

### 실험 2: 환경별 배포 트리거

```bash
# 특정 환경으로 배포
gh workflow run deploy.yml \
  -f environment=staging \
  --ref main

# 또는 수동 승인으로 production
gh workflow run deploy.yml \
  -f environment=production \
  --ref main
```

### 실험 3: Kustomize로 환경별 설정 관리

```bash
# 디렉토리 구조
overlays/
├── dev/
│   ├── kustomization.yaml
│   └── deployment.yaml (dev-specific)
├── staging/
│   ├── kustomization.yaml
│   └── deployment.yaml (staging = production)
└── production/
    ├── kustomization.yaml
    └── deployment.yaml
```

```yaml
# overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

replicas:
  - name: api
    count: 3  # Staging도 production처럼 3개 replica

commonLabels:
  environment: staging

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=info
      - ENABLE_PROFILING=true
```

```bash
# Staging 배포
kubectl apply -k overlays/staging/

# Production 배포
kubectl apply -k overlays/production/
```

### 실험 4: 환경 Drift 감지 스크립트

```bash
#!/bin/bash
# check-drift.sh

echo "=== Detecting Environment Drift ==="

# Staging과 Production의 deployment 비교
STAGING=$(kubectl get deployment api -n staging -o json)
PRODUCTION=$(kubectl get deployment api -n production -o json)

# CPU/Memory 비교
STAGING_CPU=$(echo $STAGING | jq '.spec.template.spec.containers[0].resources.limits.cpu')
PROD_CPU=$(echo $PRODUCTION | jq '.spec.template.spec.containers[0].resources.limits.cpu')

STAGING_MEM=$(echo $STAGING | jq '.spec.template.spec.containers[0].resources.limits.memory')
PROD_MEM=$(echo $PRODUCTION | jq '.spec.template.spec.containers[0].resources.limits.memory')

# 환경 변수 비교
STAGING_ENV=$(echo $STAGING | jq '.spec.template.spec.containers[0].env')
PROD_ENV=$(echo $PRODUCTION | jq '.spec.template.spec.containers[0].env')

if [ "$STAGING_CPU" != "$PROD_CPU" ] || [ "$STAGING_MEM" != "$PROD_MEM" ]; then
  echo "⚠️  Resource drift detected!"
  echo "Staging: $STAGING_CPU / $STAGING_MEM"
  echo "Production: $PROD_CPU / $PROD_MEM"
fi

if ! diff <(echo $STAGING_ENV | jq 'sort') <(echo $PROD_ENV | jq 'sort') > /dev/null; then
  echo "⚠️  Environment variable drift detected!"
fi
```

### 실험 5: PR마다 Feature 환경 자동 생성

```yaml
name: Feature Environment

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  create-feature-env:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Create feature namespace
        run: |
          PR_NUM=${{ github.event.number }}
          NAMESPACE="pr-$PR_NUM"
          
          kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
          
          # Database 포드 생성
          kubectl run postgres \
            --image=postgres:15 \
            --env=POSTGRES_PASSWORD=test \
            --port=5432 \
            -n $NAMESPACE
          
          # App 배포
          kubectl apply -f deploy.yaml -n $NAMESPACE
      
      - name: Wait for readiness
        run: |
          kubectl wait --for=condition=ready pod \
            -l app=api \
            -n pr-${{ github.event.number }} \
            --timeout=300s
      
      - name: Post feature URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✨ Feature environment: https://pr-${{ github.event.number }}.dev.example.com'
            });
```

## 📊 성능/비용 비교

| 환경 | 목적 | 비용 | 리소스 | 트래픽 |
|-----|------|------|--------|--------|
| **Development** | 빠른 개발 | 낮음 | 최소 (1개 instance) | 개발자만 |
| **Staging** | Production 테스트 | 중간 | 중간 (3개 instance) | 제한된 사용자 |
| **Production** | 실제 운영 | 높음 | 높음 (5+개 instance) | 전체 사용자 |
| **Feature 환경** | PR 검증 | 낮음 | 최소 (1개 instance) | 개발자만 |

**비용 절감:**
- Dev/Feature는 auto-scaling 비활성화
- Staging은 夜間에 auto-scaling 축소
- Production은 24/7 유지

## ⚖️ 트레이드오프

### Staging = Production vs 경제성

**Staging = Production:**
- 장점: 정확한 테스트
- 단점: 비용 2배

**축소 Staging:**
- 장점: 저비용
- 단점: 성능 문제 놓칠 수 있음

**권장:** Staging은 최소 2개 instance, 프로덕션과 동일한 설정.

### Feature 환경 자동 생성 vs 관리 복잡도

**자동 생성:**
- 장점: PR마다 자동으로 테스트 환경 제공
- 단점: 리소스 낭비, 정리 복잡

**수동 요청:**
- 장점: 필요할 때만 생성
- 단점: 개발자가 매번 요청해야 함

**권장:** PR 열려있는 동안만 자동 생성, 닫히면 자동 삭제.

## 📌 핵심 정리

1. **환경은 최소 3개**: Dev (빠른 개발), Staging (프로덕션과 동일), Production (실제 운영).

2. **Staging ≈ Production**: 다르면 안 된다. 메모리, CPU, 의존성 모두 동일해야 한다.

3. **Secret은 환경별로**: DATABASE_URL, API_KEY 같은 민감 정보는 environment별로 관리.

4. **배포 권한 통제**: Production은 반드시 승인자 지정. Dev는 자동배포.

5. **Drift 감지**: 정기적으로 Staging과 Production 설정을 비교해 차이 감지.

## 🤔 생각해볼 문제

### Q1: Staging에서 테스트했는데 Production에서 실패해요. 뭐가 다를까요?

<details>
<summary>해설</summary>

다음 항목을 체크하세요:

```bash
# 1. 환경 변수
kubectl get deployment api -n staging -o jsonpath='{.spec.template.spec.containers[0].env}' | jq
kubectl get deployment api -n production -o jsonpath='{.spec.template.spec.containers[0].env}' | jq

# 2. 리소스 제한
kubectl describe deployment api -n staging
kubectl describe deployment api -n production

# 3. 데이터베이스 상태
# Staging과 Production의 데이터 다를 수 있음
# 특히 migration 상태

# 4. 외부 서비스 의존성
# Staging: 테스트용 API
# Production: 실제 API

# 5. 네트워크 정책
kubectl get networkpolicies -n staging
kubectl get networkpolicies -n production
```

가장 흔한 원인은 **메모리 부족**이다. Staging은 충분하지만 Production에서 부족할 수 있다.

```bash
# Staging: 2GB 메모리로 충분
# Production: 실제 트래픽 때문에 5GB 필요

# 메모리 증가
kubectl set resources deployment/api \
  --limits=memory=5Gi \
  -n production
```
</details>

### Q2: PR마다 Feature 환경을 만드니까 리소스가 너무 많이 든다면?

<details>
<summary>해설</summary>

비용 최적화:

```yaml
# 1. 최소 리소스로 시작
- name: Create minimal feature env
  run: |
    kubectl apply -f deploy-minimal.yaml -n pr-$PR_NUM
    # deploy-minimal.yaml: 1개 pod, 최소 메모리

# 2. 자동 정리 타이머
- name: Schedule cleanup
  run: |
    # 3일 후 자동 삭제
    # CronJob으로 구현

# 3. 선택적 생성
# PR에 "create-env" 라벨이 있을 때만 생성
if: contains(github.event.pull_request.labels.*.name, 'create-env')

# 4. 공유 환경
# 모든 PR이 하나의 staging을 공유
# (하지만 테스트 격리 문제 발생 가능)
```

**권장:** 활성 PR이 10개 이상이면 "선택적 생성"으로 변경.
</details>

### Q3: 환경 변수가 많아지니까 관리하기 어려워요.

<details>
<summary>해설</summary>

ConfigMap과 Secret을 활용하세요:

```yaml
# kubernetes/overlays/production/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: warn
  DATABASE_POOL_SIZE: "20"
  CACHE_TTL: "3600"
  ENABLE_METRICS: "true"

---
# kubernetes/overlays/production/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: ...
  API_KEY: ...
```

그리고 Deployment에서:

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: DATABASE_URL
```

또는 환경 파일 사용:

```bash
# .env.production
LOG_LEVEL=warn
DATABASE_POOL_SIZE=20

# 배포 시
kubectl create configmap app-config --from-env-file=.env.production
```
</details>

---

[⬅️ 이전](./03-pipeline-performance-optimization.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./05-incident-recovery.md)
