# 배포 파이프라인 전체 흐름 — 코드 Push부터 헬스체크까지

## 🎯 핵심 질문
- 코드 Push부터 프로덕션 배포까지 정확히 몇 단계가 필요한가?
- CI와 CD의 경계는 어디인가?
- 각 단계에서 실패했을 때 파이프라인이 자동 중단되는 조건은?
- 배포 후 자동 스모크 테스트와 롤백은 어떻게 구현하는가?

## 🔍 왜 이 개념이 실무에서 중요한가

지금까지 배운 배포 전략들(Rolling Update, Blue-Green, Canary 등)은 **"배포 방식"** 에 관한 것입니다.

하지만 그 전에 물어야 할 것이 있습니다:
- **"배포된 이미지는 어디서 왔는가?"** (CI 파이프라인)
- **"누가 언제 배포를 트리거하는가?"** (GitOps vs 수동)
- **"배포 후 뭔가 잘못되면 누가 감지하는가?"** (모니터링)
- **"배포 성공/실패를 어디에 기록하는가?"** (GitHub Deployment API)

전체 그림이 없으면:
1. **CI/CD 파이프라인이 느려서 배포가 1시간 걸림** (불필요한 단계)
2. **배포 후 30분 뒤에야 에러 감지** (모니터링 부재)
3. **누가 배포했는지, 뭐가 배포됐는지 기록이 없음** (버전 관리 부재)

이 모든 것을 통합한 **완전한 배포 파이프라인**을 이해해야 합니다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```bash
# ❌ 수동으로 하나하나 하는 파이프라인
cd ~/myapp
git pull origin main

# 1. 코드 빌드
mvn clean package

# 2. 이미지 빌드
docker build -t myapp:v1.0 .

# 3. 레지스트리 푸시
docker push docker.io/myapp:v1.0

# 4. Kubernetes 업데이트
kubectl set image deployment/myapp app=docker.io/myapp:v1.0

# 5. 배포 완료 확인
kubectl rollout status deployment/myapp

# 6. 수동 테스트
curl http://myapp.example.com/health

# 문제점:
# 1. 과정이 길고 에러 가능성 높음 (사람이 하는 과정)
# 2. 각 단계별 실패 처리 없음 (빌드 실패 → 배포까지 진행?)
# 3. 배포 성공/실패 기록 없음
# 4. 야간 배포 시 누가 이 모든 과정을 수행?
# 5. 배포 후 모니터링 부재 (수동 헬스체크만)
```

**실제 발생하는 일:**
```
09:00 개발자 A가 배포 시작
      git pull → 실패 (merge conflict)
      
      재시도... 해결
      
09:05 mvn clean package
      빌드 실패 (테스트 오류)
      
      그동안 다른 개발자 B가 코드 변경
      
09:10 최신 코드로 다시 빌드
      성공
      
09:15 docker build → docker push
      이미지 푸시 중 네트워크 오류
      
      재시도...
      
09:25 마침내 이미지 푸시 완료
      하지만 어느 이미지가 푸시됐는지 기록 없음
      
09:30 kubectl set image
      배포 시작
      
09:35 배포 완료 (kubectl rollout status)
      
09:45 사용자 리포트: "앱이 느려졌어!"
      
      원인: 메모리 누수가 있는 버전이 배포됨
      
10:00 긴급 롤백 (수동)
      원인 분석 시작 (어느 버전이 문제였나?)
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 전체 파이프라인 구조

```
코드 Push
  ↓
Git Webhook
  ↓
CI 파이프라인 (GitHub Actions)
  ├─ 코드 체크아웃
  ├─ 단위 테스트 (실패하면 중단)
  ├─ 통합 테스트 (실패하면 중단)
  ├─ 정적 분석 (SAST, 실패하면 중단)
  ├─ 이미지 빌드
  ├─ 이미지 보안 스캔 (CVE, 실패하면 중단)
  ├─ 레지스트리 푸시
  └─ GitHub Deployment API 호출 (상태: pending)
  
  ↓ (성공 시에만 진행)
  
CD 파이프라인 (ArgoCD)
  ├─ Git 폴링 (kubernetes-manifests repo)
  ├─ YAML 변경 감지
  ├─ kubectl apply (Deployment 업데이트)
  ├─ Rolling Update / Canary 실행
  └─ 배포 완료
  
  ↓
헬스체크 및 모니터링
  ├─ Prometheus: 에러율, 응답시간 수집
  ├─ 자동 스모크 테스트 실행
  ├─ 임계값 초과 → 자동 롤백
  └─ 모니터링 alert 발송
  
  ↓
GitHub Deployment API 업데이트
  └─ 상태: success 또는 failure 기록
```

### 각 단계별 구체적 구현

```yaml
# 1. GitHub Actions CI 파이프라인
name: CI Pipeline

on:
  push:
    branches: [ main ]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    # 단위 테스트
    - name: Run unit tests
      run: |
        mvn test
        if [ $? -ne 0 ]; then
          echo "Unit tests failed"
          exit 1
        fi
    
    # 통합 테스트
    - name: Run integration tests
      run: |
        mvn verify
        if [ $? -ne 0 ]; then
          echo "Integration tests failed"
          exit 1
        fi
    
    # 이미지 빌드
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
    
    # 보안 스캔 (Trivy)
    - name: Scan image for vulnerabilities
      run: |
        trivy image myapp:${{ github.sha }}
        if [ $? -ne 0 ]; then
          exit 1
        fi
    
    # 레지스트리 푸시
    - name: Push to registry
      env:
        REGISTRY: docker.io
        REGISTRY_USER: ${{ secrets.DOCKER_USER }}
        REGISTRY_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      run: |
        docker login -u $REGISTRY_USER -p $REGISTRY_PASSWORD
        docker tag myapp:${{ github.sha }} $REGISTRY/myapp:latest
        docker push $REGISTRY/myapp:latest
        docker push $REGISTRY/myapp:${{ github.sha }}
    
    # GitHub Deployment API 호출
    - name: Create deployment
      uses: actions/github-script@v6
      with:
        script: |
          const deployment = await github.rest.repos.createDeployment({
            owner: context.repo.owner,
            repo: context.repo.repo,
            ref: context.ref,
            environment: 'production',
            description: 'Deploying with image ${{ github.sha }}',
            auto_merge: false,
            required_contexts: []
          });
          
          console.log('Deployment created:', deployment.data.id);
          
          // 상태: pending
          await github.rest.repos.createDeploymentStatus({
            owner: context.repo.owner,
            repo: context.repo.repo,
            deployment_id: deployment.data.id,
            state: 'pending',
            description: 'Deployment in progress'
          });
---
# 2. ArgoCD를 통한 CD 파이프라인
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/kubernetes-manifests.git
    targetRevision: main
    path: myapp/
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # 자동 동기화
  syncPolicy:
    automated:
      prune: true      # 삭제된 리소스 정리
      selfHeal: true   # 수동 변경 감지 후 복구
    syncOptions:
    - CreateNamespace=true
  
  # 배포 후 헬스체크
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # 자동 스케일링 무시
```

## 🔬 내부 동작 원리

### CI와 CD의 명확한 경계

```
CI (Continuous Integration):
  책임: 코드 품질 보증
  
  1. 코드 체크아웃
  2. 빌드 (컴파일, 테스트)
  3. 검증 (SAST, SCA)
  4. 아티팩트 생성 (Docker 이미지)
  5. 아티팩트 저장 (Registry)
  
  출력: 검증된 Docker 이미지
  위치: Docker Registry (docker.io/myapp:v1.0)

CD (Continuous Deployment):
  책임: 배포 자동화
  
  1. 배포 스크립트 감지 (kubernetes-manifests repo)
  2. 현재 상태와 목표 상태 비교
  3. 필요한 변경 적용 (kubectl apply)
  4. Pod 업데이트 (Rolling Update / Canary)
  5. 배포 완료
  
  입력: Kubernetes YAML (kubernetes-manifests repo)
  출력: 실행 중인 Pod (Kubernetes 클러스터)

경계:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ← CI 파이프라인 → ← Docker Registry → ← CD 파이프라인 →
  
  GitHub Actions     docker.io           ArgoCD
  (GitHub)           (Docker Hub)        (Kubernetes)
```

### GitHub Deployment API의 역할

```
GitHub Deployment API: 배포 상태 추적

1. Deployment 객체 생성 (CI 파이프라인)
   → GitHub에 "배포 준비 중"임을 알림
   
2. Deployment 상태 업데이트 (CD 파이프라인)
   ├─ "pending": 배포 진행 중
   ├─ "in_progress": 배포 중
   ├─ "success": 배포 성공
   ├─ "failure": 배포 실패
   └─ "error": 시스템 오류
   
3. Release Note / 추적
   → GitHub의 "Releases" 탭에서 조회 가능
   → PR에서 어느 버전이 배포되었는지 확인
   → 배포 히스토리 추적

예시:
  PR #123 → Merged
       → CI 파이프라인 실행
       → Deployment #456 생성 (image: myapp:abc123def)
       → CD 파이프라인 실행
       → Deployment #456 상태 업데이트
       → Deployment #456 성공 (environment: production)
       
사용자 입장:
  PR #123 보면 → "배포됨" 상태 확인 가능 ✅
```

### 자동 롤백의 메커니즘

```
배포 후 모니터링:

1단계: 메트릭 수집 (Prometheus)
   - 에러율: prometheus.io/errors_total
   - 응답시간: prometheus.io/request_duration
   - 메모리: container_memory_usage_bytes
   
2단계: 임계값 비교
   에러율 > 5%?
   ├─ Yes → 자동 롤백 트리거
   └─ No → 계속 진행
   
   응답시간 P99 > 1초?
   ├─ Yes → 자동 롤백 트리거
   └─ No → 계속 진행
   
3단계: 자동 롤백 실행
   kubectl rollout undo deployment/myapp
   또는
   ArgoCD에서 이전 commit으로 복구
   
4단계: 알림 발송
   Slack: "@channel 배포가 자동 롤백되었습니다"
   GitHub: Deployment 상태 → "failure"
   PagerDuty: Incident 생성 (원인 분석 필요)
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 실험 1: 전체 배포 파이프라인

```bash
#!/bin/bash
# deploy-pipeline.sh - 전체 배포 파이프라인 통합 스크립트

set -e  # 에러 발생 시 중단

REPO="myapp"
ENVIRONMENT="production"
DEPLOYMENT_NAME="myapp"

echo "=== Deployment Pipeline Started ==="

# 단계 1: CI 파이프라인 시뮬레이션
echo "1. Running CI pipeline..."

# 1-1. 코드 체크아웃
echo "   - Checking out code"
git clone --depth 1 https://github.com/myorg/$REPO.git

# 1-2. 테스트 실행
echo "   - Running tests"
cd $REPO
npm test || { echo "❌ Tests failed"; exit 1; }

# 1-3. 이미지 빌드
echo "   - Building Docker image"
IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t $REPO:$IMAGE_TAG .
docker tag $REPO:$IMAGE_TAG $REPO:latest

# 1-4. 이미지 푸시
echo "   - Pushing to registry"
docker push docker.io/$REPO:$IMAGE_TAG
docker push docker.io/$REPO:latest

echo "✓ CI pipeline completed (image: $IMAGE_TAG)"

# 단계 2: CD 파이프라인 트리거
echo ""
echo "2. Triggering CD pipeline..."

# 2-1. Kubernetes manifests 업데이트
echo "   - Updating deployment manifest"
cd ../kubernetes-manifests
sed -i "s/image: .*/image: docker.io\/$REPO:$IMAGE_TAG/" deployment.yaml

# 2-2. Git commit & push
git add deployment.yaml
git commit -m "Deploy $REPO:$IMAGE_TAG"
git push origin main

echo "✓ Manifest updated, ArgoCD will sync automatically"

# 단계 3: ArgoCD 배포 진행 대기
echo ""
echo "3. Waiting for deployment to complete..."

# ArgoCD가 감지할 시간 제공 (보통 3-5초)
sleep 5

# 배포 상태 확인
kubectl rollout status deployment/$DEPLOYMENT_NAME -n $ENVIRONMENT --timeout=5m

if [ $? -eq 0 ]; then
  echo "✓ Deployment completed successfully"
else
  echo "❌ Deployment failed"
  exit 1
fi

# 단계 4: 헬스체크
echo ""
echo "4. Running health checks..."

# 4-1. Pod 상태 확인
READY_PODS=$(kubectl get deployment $DEPLOYMENT_NAME -n $ENVIRONMENT \
  -o jsonpath='{.status.readyReplicas}')
TOTAL_PODS=$(kubectl get deployment $DEPLOYMENT_NAME -n $ENVIRONMENT \
  -o jsonpath='{.status.replicas}')

echo "   - Pod status: $READY_PODS/$TOTAL_PODS ready"

if [ "$READY_PODS" != "$TOTAL_PODS" ]; then
  echo "❌ Not all pods are ready"
  exit 1
fi

# 4-2. Service 헬스체크
echo "   - Running HTTP health check"
POD=$(kubectl get pod -l app=$DEPLOYMENT_NAME -n $ENVIRONMENT \
  -o name | head -1)

kubectl exec $POD -n $ENVIRONMENT -- \
  curl -f http://localhost:8080/health || {
    echo "❌ Health check failed"
    exit 1
  }

# 4-3. 메트릭 확인
echo "   - Checking metrics"
ERROR_RATE=$(kubectl exec $POD -n $ENVIRONMENT -- \
  curl -s http://prometheus:9090/api/v1/query \
  --data-urlencode 'query=rate(errors_total[5m])' | jq '.data.result[0].value[1]')

echo "   - Error rate: $ERROR_RATE"

if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
  echo "❌ Error rate too high, rolling back"
  kubectl rollout undo deployment/$DEPLOYMENT_NAME -n $ENVIRONMENT
  exit 1
fi

echo "✓ All health checks passed"

# 단계 5: GitHub Deployment API 업데이트
echo ""
echo "5. Updating GitHub Deployment status..."

GITHUB_TOKEN=$GITHUB_TOKEN
DEPLOYMENT_ID=$(curl -s \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/myorg/$REPO/deployments \
  | jq '.[-1].id')

curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/myorg/$REPO/deployments/$DEPLOYMENT_ID/statuses \
  -d '{
    "state": "success",
    "description": "Deployment completed",
    "environment_url": "https://myapp.example.com",
    "auto_inactive": false
  }'

echo "✓ GitHub Deployment marked as success"

# 단계 6: 알림 발송
echo ""
echo "6. Sending notifications..."

curl -X POST $SLACK_WEBHOOK \
  -d '{
    "text": "✅ Deployment successful",
    "attachments": [{
      "color": "good",
      "fields": [
        {"title": "App", "value": "'$REPO'", "short": true},
        {"title": "Image", "value": "'$IMAGE_TAG'", "short": true},
        {"title": "Environment", "value": "'$ENVIRONMENT'", "short": true}
      ]
    }]
  }'

echo "✓ Notifications sent"

echo ""
echo "=== Deployment Pipeline Completed Successfully ==="
```

### 실험 2: GitHub Actions 워크플로우

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [ main ]

env:
  REGISTRY: docker.io
  IMAGE_NAME: myapp

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
    - uses: actions/checkout@v3
    
    # 1. 단위 테스트
    - name: Run unit tests
      run: npm test
    
    # 2. 통합 테스트
    - name: Run integration tests
      run: npm run test:integration
    
    # 3. 정적 분석
    - name: Run SAST (SonarQube)
      uses: SonarSource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    
    # 4. 이미지 빌드
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to registry
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix={{branch}}-
          type=ref,event=branch
          type=semver,pattern={{version}}
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
    
    # 5. 이미지 스캔 (Trivy)
    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.meta.outputs.tags }}
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    # 6. GitHub Deployment 생성
    - name: Create GitHub Deployment
      id: deployment
      uses: actions/github-script@v6
      with:
        script: |
          const deployment = await github.rest.repos.createDeployment({
            owner: context.repo.owner,
            repo: context.repo.repo,
            ref: context.ref,
            environment: 'production',
            description: 'Deploying ${{ env.IMAGE_NAME }}:${{ github.sha }}',
            required_contexts: [],
            auto_merge: false
          });
          
          core.setOutput('deployment-id', deployment.data.id);
          
          // 상태 업데이트: pending
          await github.rest.repos.createDeploymentStatus({
            owner: context.repo.owner,
            repo: context.repo.repo,
            deployment_id: deployment.data.id,
            state: 'pending',
            description: 'Deployment in progress',
            auto_inactive: false
          });

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    environment:
      name: production
      url: https://myapp.example.com
    
    steps:
    - uses: actions/checkout@v3
      with:
        repository: myorg/kubernetes-manifests
        token: ${{ secrets.GH_TOKEN }}
    
    - name: Update deployment manifest
      run: |
        sed -i 's|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|' deployment.yaml
    
    - name: Commit and push
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add deployment.yaml
        git commit -m "Deploy ${{ env.IMAGE_NAME }}:${{ github.sha }}"
        git push
    
    - name: Wait for ArgoCD sync
      run: |
        # ArgoCD API를 통해 sync 상태 대기
        for i in {1..30}; do
          STATUS=$(curl -s -H "Authorization: Bearer $ARGOCD_TOKEN" \
            https://argocd.example.com/api/v1/applications/myapp \
            | jq -r '.status.operationState.phase')
          
          if [ "$STATUS" == "Succeeded" ]; then
            echo "✓ Deployment completed"
            break
          fi
          
          sleep 10
        done
    
    - name: Update deployment status
      if: always()
      uses: actions/github-script@v6
      with:
        script: |
          const deployment_id = ${{ needs.build-and-test.outputs.deployment-id }};
          
          const state = context.job.status === 'success' ? 'success' : 'failure';
          
          await github.rest.repos.createDeploymentStatus({
            owner: context.repo.owner,
            repo: context.repo.repo,
            deployment_id: deployment_id,
            state: state,
            description: state === 'success' ? 'Deployment succeeded' : 'Deployment failed',
            environment_url: 'https://myapp.example.com',
            auto_inactive: false
          });
```

### 실험 3: 배포 후 자동 롤백

```bash
#!/bin/bash
# post-deployment-monitoring.sh - 배포 후 자동 모니터링 및 롤백

DEPLOYMENT="myapp"
NAMESPACE="production"
MONITORING_DURATION=300  # 5분
INTERVAL=10              # 10초마다 체크

echo "=== Post-Deployment Monitoring Started ==="
echo "Duration: $MONITORING_DURATION seconds"
echo "Check interval: $INTERVAL seconds"

START_TIME=$(date +%s)
ERROR_RATE_THRESHOLD=0.05  # 5%
LATENCY_THRESHOLD=1000     # 1000ms

while true; do
  CURRENT_TIME=$(date +%s)
  ELAPSED=$((CURRENT_TIME - START_TIME))
  
  if [ $ELAPSED -gt $MONITORING_DURATION ]; then
    echo "✓ Monitoring completed successfully"
    break
  fi
  
  # 1. 에러율 확인 (Prometheus)
  ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
    --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[5m])' \
    | jq -r '.data.result[0].value[1] // 0')
  
  # 2. 응답시간 확인 (Prometheus)
  LATENCY=$(curl -s "http://prometheus:9090/api/v1/query" \
    --data-urlencode 'query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))' \
    | jq -r '.data.result[0].value[1] // 0')
  
  echo "[$ELAPSED/$MONITORING_DURATION] Error rate: $ERROR_RATE, P99 latency: ${LATENCY}ms"
  
  # 3. 임계값 초과 여부 확인
  if (( $(echo "$ERROR_RATE > $ERROR_RATE_THRESHOLD" | bc -l) )); then
    echo "❌ Error rate exceeded threshold ($ERROR_RATE > $ERROR_RATE_THRESHOLD)"
    echo "Initiating automatic rollback..."
    
    # 이전 버전으로 즉시 롤백
    kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
    
    # 롤백 상태 확인
    kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE
    
    # 알림 발송
    curl -X POST $SLACK_WEBHOOK \
      -d '{
        "text": "🚨 Automatic rollback initiated",
        "attachments": [{
          "color": "danger",
          "fields": [
            {"title": "Deployment", "value": "'$DEPLOYMENT'"},
            {"title": "Reason", "value": "Error rate too high ('$ERROR_RATE')"},
            {"title": "Action", "value": "Rolled back to previous version"}
          ]
        }]
      }'
    
    # GitHub Deployment API 업데이트
    curl -X POST https://api.github.com/repos/myorg/myapp/deployments/123/statuses \
      -d '{
        "state": "failure",
        "description": "Automatic rollback due to high error rate"
      }' \
      -H "Authorization: token $GITHUB_TOKEN"
    
    exit 1
  fi
  
  if (( $(echo "$LATENCY > $LATENCY_THRESHOLD" | bc -l) )); then
    echo "⚠️  Latency too high ($LATENCY > $LATENCY_THRESHOLD ms)"
    echo "Initiating automatic rollback..."
    
    kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
    exit 1
  fi
  
  sleep $INTERVAL
done

echo "✓ Post-deployment monitoring completed"
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 항목 | 수동 배포 | 자동 CI | 자동 CI+CD | 자동 CI+CD+모니터링 |
|------|---|---|---|---|
| **배포 시간** | 30분 | 5분 | 3분 | 8분 (포함) |
| **실패 감지** | 수동 | 자동 | 자동 | 자동 |
| **자동 롤백** | ❌ | ❌ | ❌ | ✅ |
| **배포 기록** | ❌ | ✅ | ✅ | ✅ |
| **야간 배포** | ❌ | ✅ | ✅ | ✅ |
| **배포 1회 비용** | $30 (인건비) | $5 (리소스) | $5 (리소스) | $10 (모니터링) |

### 비용 절감 계산

```
수동 배포 vs 자동 배포

수동 배포:
  배포마다: 30분 × $100/시간 = $50
  일일 배포: 2회 × $50 = $100/일
  월간: $100 × 20 = $2,000

자동 배포:
  CI/CD 인프라: $200/월
  모니터링: $100/월
  월간: $300

절감액: $2,000 - $300 = $1,700/월

ROI:
  월 2회 이상 배포면 자동화 비용 회수 가능
```

## ⚖️ 트레이드오프

### 배포 속도 vs 안정성

```
수동 배포 (30분):
  ✅ 인간의 판단 (복잡한 상황 대응)
  ❌ 느린 배포 (병목)
  ❌ 야간 배포 불가능

자동 배포 (5분):
  ❌ 정해진 규칙만 따름 (복잡한 상황 처리 불가)
  ✅ 빠른 배포
  ✅ 야간 배포 가능
  ✅ 일관된 프로세스
```

### 모니터링 정확도 vs 오탐 위험

```
엄격한 모니터링 (에러율 > 1% → 롤백):
  ✅ 빠른 감지
  ❌ 오탐 많음 (정상 변동까지 롤백)
  ❌ 실제 배포 성공률 낮음

느슨한 모니터링 (에러율 > 10% → 롤백):
  ✅ 오탐 적음
  ❌ 느린 감지
  ❌ 사용자 영향 크림

권장: 다중 지표 조합
  에러율 > 5% AND 응답시간 > 2초 → 롤백
```

## 📌 핵심 정리

1. **전체 파이프라인의 6단계:**
   ```
   1. CI: 코드 검증 → 이미지 빌드 → 레지스트리 푸시
   2. GitHub Deployment API: 배포 기록 (pending)
   3. ArgoCD: YAML 감지 → kubectl apply
   4. Rolling Update/Canary: Pod 업데이트
   5. 헬스체크: 메트릭 수집, 스모크 테스트
   6. 모니터링: 임계값 초과 → 자동 롤백
   ```

2. **CI와 CD의 경계:**
   - CI: 코드 품질 보증 (Build, Test, Scan)
   - CD: 배포 자동화 (Deploy, Monitor, Rollback)

3. **각 단계별 실패 조건:**
   ```
   테스트 실패 → 파이프라인 중단 ✅
   빌드 실패 → 파이프라인 중단 ✅
   보안 스캔 실패 → 파이프라인 중단 ✅
   Pod 준비 안 됨 → 배포 실패 (자동 롤백) ✅
   에러율 높음 → 자동 롤백 ✅
   ```

4. **GitHub Deployment API의 역할:**
   - 배포 상태 추적 (pending → success/failure)
   - PR/Release에서 배포 내역 확인
   - 배포 히스토리 영구 기록

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: CI 파이프라인은 성공했는데 CD 파이프라인이 실패했다. 누가 책임지는가?

```
가능한 원인:
1. Kubernetes 클러스터 다운
2. ArgoCD 오류
3. kubectl apply 실패 (리소스 부족)
4. Pod이 Ready되지 않음
```

<details>
<summary>정답 및 해설</summary>

**책임 분리:**
```
CI 파이프라인 (성공):
  - GitHub Actions: ✅ 정상
  - 이미지: ✅ 레지스트리에 저장됨
  - 책임: 없음 (한 일 다 함)

CD 파이프라인 (실패):
  - ArgoCD: ❓ 설정 오류?
  - Kubernetes: ❓ 리소스 부족?
  - 네트워크: ❓ 레지스트리 접근 불가?
  - 책임: 인프라팀 (클러스터 관리)
```

**추적 방법:**
```
1. ArgoCD 로그 확인
   kubectl logs -n argocd deployment/argocd-application-controller
   
2. Deployment 이벤트 확인
   kubectl describe deployment myapp -n production
   
3. Pod 상태 확인
   kubectl get pods -l app=myapp
   
4. Kubernetes 메모리/리소스 확인
   kubectl top nodes
```

**GitHub Deployment API 상태:**
```
상태: "failure"
설명: "CD pipeline failed: Pod failed to reach ready state"

이를 보면 개발팀이 인프라팀에 리포트할 수 있음
```

</details>

### 문제 2: 배포 후 5초 후에는 정상이었는데 10분 뒤에 에러 발생했다면?

```
모니터링 윈도우: 300초 (5분)

배포 직후: 정상
5분 뒤: 여전히 정상 (모니터링 완료)
10분 뒤: 갑자기 에러 발생

자동 롤백이 트리거되지 않음 (모니터링 윈도우 초과)
```

<details>
<summary>정답 및 해설</summary>

**원인 분석:**
```
일반적인 원인:
1. 메모리 누수 (시간이 지나면서 메모리 증가)
2. DB 연결 풀 고갈 (누적된 요청)
3. 캐시 크기 증가
4. 외부 API 레이트 제한 (시간이 지나면서 증가)
```

**해결책:**
```
1. 모니터링 윈도우 확대
   MONITORING_DURATION=600  # 10분
   
2. 배포 후 지속적 모니터링
   # 배포 후 모니터링이 아닌
   # 지속적 헬스 모니터링 추가
   
3. 프로메테우스 알터트 규칙
   alert: HighErrorRate
   for: 5m  # 5분 지속 시 알림
   
4. Sentry/LogRocket 같은 추가 모니터링
   - 실시간 에러 추적
   - 사용자 영향도 분석
```

**모니터링 시스템 구조:**
```
배포 후 5분 모니터링 (자동 롤백 가능)
         ↓
배포 후 24시간 기본 모니터링 (알림)
         ↓
프로덕션 지속적 모니터링 (알림)
```

</details>

### 문제 3: 자동 롤백 중에 또 다른 배포 요청이 들어왔다면?

```
시나리오:
09:00 배포 1 시작
09:05 배포 1이 에러율 높음 → 자동 롤백 중
09:06 개발자가 배포 2 요청

배포 1 롤백이 끝나지 않았는데
배포 2가 시작되면?
```

<details>
<summary>정답 및 해설</summary>

**ArgoCD의 동작:**
```
배포 1: main 브랜치 → image v1.0 → (에러) → 롤백 시작
        Deployment: replicas 2개만 실행 (롤백 중)

배포 2: main 브랜치 업데이트 → image v2.0
        ArgoCD가 감지 → kubectl apply v2.0

결과:
  ❌ 롤백이 중단됨
  ❌ v1.0 복구 실패
  ❌ v2.0이 배포됨 (원치 않는 상황)
```

**해결책 1: 배포 큐 (Sequential)**
```
배포 1이 진행 중 → 배포 2는 큐에 대기
배포 1 완료 (또는 롤백 완료) → 배포 2 시작

구현: 파이프라인 레벨에서 Lock 사용
```

**해결책 2: 배포 정책 (Prevent Concurrent)**
```
ArgoCD Application:
  spec:
    syncPolicy:
      syncOptions:
      - PrunePropagationPolicy=background
      # 동시 sync 방지
```

**권장 방식:**
```
GitHub Actions에서 배포 잠금:
  - 배포 1 시작 → Lock 획득
  - 배포 2 요청 → Lock 대기
  - 배포 1 완료 → Lock 해제
  - 배포 2 시작

bash 예시:
  LOCK_FILE="/tmp/deployment.lock"
  
  while [ -f $LOCK_FILE ]; do
    sleep 1
  done
  
  touch $LOCK_FILE
  # 배포 진행
  rm $LOCK_FILE
```

</details>

---

<div align="center">

**[⬅️ 이전: Zero-Downtime 배포 조건](./06-zero-downtime.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — GitOps 원칙 완전 분해 ➡️](../gitops-argocd/01-gitops-principles.md)**

</div>
