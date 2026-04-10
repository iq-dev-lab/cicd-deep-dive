# Argo Rollouts — 메트릭 기반 자동 승급과 롤백

## 🎯 핵심 질문
- Argo Rollouts는 Kubernetes Deployment와 정확히 무엇이 다른가?
- `Rollout` 리소스의 구조는 어떻게 되어 있는가?
- `AnalysisTemplate`로 Prometheus 메트릭을 확인하고 자동 승급/롤백하는 메커니즘은?
- `kubectl argo rollouts` 명령어로 무엇을 할 수 있는가?

## 🔍 왜 이 개념이 실무에서 중요한가

지금까지 배운 배포 전략들의 한계:
- **Rolling Update**: 문제 감지 후 수동으로 롤백 필요
- **Blue-Green**: 에러율 모니터링은 하지만 자동 롤백 없음
- **Canary**: 단계적 배포지만 롤백 시점을 수동으로 결정

**Argo Rollouts의 차별점:**
```
Prometheus 메트릭 조회 → 에러율 높음? → 자동 롤백
                      → 응답시간 증가? → 자동 롤백
                      → 정상? → 자동 승급
```

이것이 진정한 **자동 배포** (Automatic Deployment)입니다.

실무에서:
1. **배포 담당자의 모니터링 부담 감소** (자동으로 판단)
2. **야간/주말 배포도 안전** (사람의 개입 없이 진행)
3. **장애 발생 시 즉시 롤백** (수동 개입 전에 자동 처리)

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 그냥 Canary를 수동으로 관리
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

**배포 후:**
```
08:00 배포 시작, weight: 5%로 설정
      → 모니터링 중...
08:05 에러율 2%, 괜찮아 보임 → weight: 20% (수동 변경)
08:10 에러율 3%, 여전히 괜찮음 → weight: 50% (수동 변경)
08:15 에러율 급증 12%!! 문제 발생!
      → weight: 0% 로 긴급 변경 (수동, 2분 지연)
08:17 안정화 (하지만 이미 사용자들은 에러 경험)

문제점:
1. 모니터링 담당자가 항상 대기해야 함
2. 대응 속도가 느림 (사람이 판단하는 시간)
3. 야간/주말 배포 시 누가 모니터링?
4. 일관성 없음 (담당자마다 다른 판단)
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ Argo Rollouts: 자동 메트릭 기반 승급/롤백
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    canary:
      steps:
      - setWeight: 5      # 단계 1: 5% 트래픽
      - pause: {duration: 5m}  # 5분 대기 (메트릭 수집)
      
      - setWeight: 20     # 단계 2: 20% 트래픽
      - analysis:         # 메트릭 분석 (자동!)
          templates:
          - name: error-rate
      
      - setWeight: 50     # 단계 3: 50% 트래픽
      - analysis:
          templates:
          - name: error-rate
      
      - setWeight: 100    # 단계 4: 100% 트래픽 (완료)
      
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
# AnalysisTemplate: Prometheus 메트릭 정의
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
spec:
  metrics:
  - name: error-rate
    initialDelay: 30s    # Pod 시작 후 30초 대기
    interval: 1m         # 1분마다 체크
    count: 5             # 5회 체크 (5분)
    failureLimit: 2      # 2회 이상 실패하면 롤백
    
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(irate(http_requests_total{status=~"5.."}[1m])) /
          sum(irate(http_requests_total[1m]))
        
        # 롤백 조건
        thresholds:
          error: "0.05"   # 에러율 > 5% → 롤백
          success: "0.01" # 에러율 < 1% → 승급 (성공)
```

**자동 배포 프로세스:**
```
08:00 배포 시작 (weight: 5%)
  ↓
Argo가 Prometheus에서 에러율 조회 (자동)
  ↓
08:01 에러율: 2% (< 5%) → 성공
      weight: 5% → 20% (자동 진행)
  ↓
08:06 에러율: 3% (< 5%) → 성공
      weight: 20% → 50% (자동 진행)
  ↓
08:11 에러율: 12% (> 5%) → 실패
      weight: 50% → 0% (자동 롤백!)
  ↓
08:12 배포 중단, Rollout 상태: Failed
      사람이 원인 분석 후 재배포 또는 롤백

결과:
- 사람의 개입 없음
- 일관된 판단 기준 (Prometheus 메트릭)
- 빠른 대응 (자동 감지 → 자동 롤백)
```

## 🔬 내부 동작 원리

### Rollout vs Deployment 차이

```
Kubernetes Deployment:
  spec:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 0

    → Pod을 순차적으로 교체
    → 배포 완료 조건: replicas 수의 Pod이 Ready
    
Argo Rollout:
  spec:
    strategy:
      canary:
        steps:
        - setWeight: 5
        - analysis:
            templates:
            - name: error-rate

    → Pod을 배포하되, 트래픽을 점진적으로 증가
    → 배포 완료 조건: 모든 step 통과 + 메트릭 분석 성공
```

### AnalysisTemplate의 동작 메커니즘

```
AnalysisRun 생성 시작
  ↓
Prometheus에 쿼리 전송
  ↓
query 결과값 수집: X
  ↓
X < error (성공): 성공 카운터 ++
X > error (실패): 실패 카운터 ++
X == success (우수): 우수 카운터 ++
  ↓
interval 대기 (다시 쿼리)
  ↓
모든 count 회 반복?
  ├─ successCount >= count → AnalysisRun 성공
  ├─ failureCount > failureLimit → AnalysisRun 실패 → 롤백
  └─ 진행 중 → 계속 대기

Rollout 입장:
  AnalysisRun 성공?
  ├─ Yes → 다음 step 진행
  └─ No → 배포 중단 + 롤백
```

### Step별 Pod 관리

```
Step 1: setWeight: 5
  Stable Pods: 3개 (95% 트래픽)
  Canary Pods: 1개 (5% 트래픽)
  → VirtualService (Istio) 또는 Service (Nginx)로 가중치 제어
  
Step 2: pause
  그냥 대기 (메트릭 수집 시간 확보)
  
Step 3: analysis
  Prometheus 쿼리 실행
  결과에 따라 계속 진행 또는 롤백
  
결과:
  ✅ 성공: step 계속 진행
  ❌ 실패: rollback step으로 이동 (weight: 0)
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 사전 준비: Argo Rollouts 설치

```bash
# Argo Rollouts 설치
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f \
  https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 설치 확인
kubectl get deployment -n argo-rollouts

# kubectl 플러그인 설치
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# 플러그인 확인
kubectl argo rollouts version
```

### 실험 1: 기본 Rollout 생성

```bash
# 1. Prometheus 설치 (메트릭 수집용)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --set prometheus.prometheusSpec.retention=2h

# 2. Prometheus Service 확인
kubectl get svc prometheus-kube-prometheus-prometheus -o wide

# 3. Rollout 리소스 생성
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause:
          duration: 2m
      - setWeight: 20
      - pause:
          duration: 2m
      - setWeight: 50
      - pause:
          duration: 2m
  
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:1.19
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# 4. Rollout 상태 확인
kubectl argo rollouts get rollout myapp --watch

# 5. Pod 생성 확인
kubectl get pods -l app=myapp
```

### 실험 2: 메트릭 기반 자동 분석

```bash
# 1. AnalysisTemplate 생성
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 30s
    count: 3  # 3회 체크 (약 90초)
    failureLimit: 1  # 1회 이상 실패하면 롤백
    
    provider:
      prometheus:
        address: http://prometheus-kube-prometheus-prometheus:9090
        query: |
          sum(irate(http_requests_total{status=~"2.."}[1m])) /
          sum(irate(http_requests_total[1m]))
        
        # Prometheus 쿼리 결과가 0-1 사이의 값이어야 함 (success rate)
        # 0.95 = 95% 성공률
        
        thresholds:
          error: "0.8"    # 성공률 < 80% → 롤백
          success: "0.9"  # 성공률 >= 90% → 승급
EOF

# 2. AnalysisRun 수동 실행 (테스트)
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AnalysisRun
metadata:
  name: test-analysis
spec:
  templates:
  - name: success-rate
  
  args:
    parameters:
    - name: service
      value: myapp
EOF

# 3. AnalysisRun 진행 상황 확인
kubectl get analysisrun test-analysis -o wide
kubectl describe analysisrun test-analysis

# 로그 확인
kubectl logs -l app=argo-rollouts-analysis-runner -n argo-rollouts --tail=50
```

### 실험 3: Rollout 업데이트 및 배포

```bash
# 1. 새 버전 배포 (image 변경)
kubectl argo rollouts set image myapp myapp=nginx:1.20

# 2. 배포 진행 상황 확인 (실시간)
kubectl argo rollouts get rollout myapp --watch

# 출력 예:
# NAME    KIND    CURRENT STEP   PHASE         STATUS
# myapp   Rollout 1/6             Progressing   ProgressDeadlineExceeded

# 3. 각 step 진행 시간 측정
kubectl get rollout myapp -o json | \
  jq '.status.conditions[] | {type, reason, lastUpdateTime}'

# 4. ReplicaSet 상태 (Stable vs Canary)
kubectl get rs -l app=myapp -o wide
# 출력:
# NAME                    DESIRED   CURRENT   READY   AGE
# myapp-abc123 (stable)   2         2         2       3m
# myapp-def456 (canary)   1         1         1       1m
```

### 실험 4: 배포 중 강제 승급

```bash
# 1. 배포 중 상태 확인
kubectl argo rollouts get rollout myapp

# 2. pause 단계에서 수동 승급
kubectl argo rollouts promote myapp

# 3. 다음 step으로 즉시 진행
kubectl argo rollouts promote myapp --full

# 4. 배포 취소 (rollback)
kubectl argo rollouts abort myapp

# 5. 배포 재개
kubectl argo rollouts retry myapp
```

### 실험 5: 조건부 AnalysisRun

```yaml
# 정교한 분석 template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: detailed-metrics
spec:
  metrics:
  # 1. 에러율 체크
  - name: error-rate
    interval: 30s
    count: 3
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus-kube-prometheus-prometheus:9090
        query: |
          sum(irate(http_requests_total{status=~"5.."}[1m])) /
          sum(irate(http_requests_total[1m]))
        thresholds:
          error: "0.05"
          success: "0.01"
  
  # 2. 응답시간 체크
  - name: latency-p99
    interval: 30s
    count: 3
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus-kube-prometheus-prometheus:9090
        query: |
          histogram_quantile(0.99, irate(http_request_duration_seconds_bucket[1m]))
        thresholds:
          error: "1.0"    # 1초 이상 → 롤백
          success: "0.5"  # 500ms 이하 → 승급
  
  # 3. CPU 사용률 체크
  - name: cpu-usage
    interval: 30s
    count: 3
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus-kube-prometheus-prometheus:9090
        query: |
          sum(rate(container_cpu_usage_seconds_total{pod=~"myapp-.*"}[1m]))
        thresholds:
          error: "4.0"    # 4 cores 이상 → 롤백
          success: "2.0"  # 2 cores 이하 → 승급
---
# Rollout에서 이 template 사용
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-advanced
spec:
  replicas: 3
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - analysis:
          templates:
          - name: detailed-metrics
            requiredForCompletion: true
      
      - setWeight: 50
      - analysis:
          templates:
          - name: detailed-metrics
  
  selector:
    matchLabels:
      app: myapp-advanced
  template:
    metadata:
      labels:
        app: myapp-advanced
    spec:
      containers:
      - name: app
        image: nginx:1.20
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### 실험 6: 자동 롤백 시나리오

```bash
# 1. 에러를 발생시키는 이미지로 배포
kubectl argo rollouts set image myapp myapp=nginx:broken

# 2. 배포 진행 중 에러 감지
# 만약 에러율이 5%를 초과하면 AnalysisRun 실패
# → Rollout 자동 롤백

# 3. Rollout 상태 확인
kubectl argo rollouts get rollout myapp
# 출력:
# Phase: Failed
# Reason: AnalysisRun failed

# 4. 이전 버전으로 원상복구 (자동)
kubectl get rs -l app=myapp
# Stable (이전 버전) 다시 활성화됨

# 5. 배포 히스토리 확인
kubectl argo rollouts history myapp
# 출력:
# Revision 1: (current) nginx:1.19
# Revision 2: nginx:1.20
# Revision 3: nginx:broken → failed

# 6. 특정 revision으로 롤백
kubectl argo rollouts rollback myapp 1
```

### 실험 7: Istio와의 통합 (고급)

```yaml
# Istio VirtualService를 통한 가중치 제어
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-istio
spec:
  replicas: 3
  strategy:
    canary:
      # Istio VirtualService로 가중치 제어
      trafficManagement:
        istio:
          virtualService:
            name: myapp-vs
            routes:
            - primary
            - canary
      
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
  
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:2.0
---
# Istio VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*internal.*"
    route:
    - destination:
        host: myapp
        port:
          number: 80
    timeout: 5s
  - route:
    - destination:
        host: myapp
        port:
          number: 80
        subset: primary
      weight: 95
    - destination:
        host: myapp
        port:
          number: 80
        subset: canary
      weight: 5
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 항목 | Rolling Update | Canary (수동) | Canary (Argo) | Blue-Green |
|------|---|---|---|---|
| **배포 시간** | 30초 | 5분 | 5-10분 | 1-2분 |
| **자동 롤백** | ❌ | ❌ | ✅ | ❌ |
| **메트릭 분석** | ❌ | 수동 | ✅ 자동 | 수동 |
| **실패 감지** | 없음 | 사람이 | 자동 | 사람이 |
| **야간 배포** | ✅ | ❌ (사람 필요) | ✅ | ✅ |
| **인프라 비용** | 낮음 | 낮음 | 낮음 | 높음 (2배) |
| **운영 복잡도** | 낮음 | 높음 | 중간 | 높음 |

### 비용 절감 계산

```
Canary 수동 운영:
- 배포마다 담당자 대기 5분
- 담당자 시간당 비용: $100
- 일일 배포 4회: 20분 = $33/일
- 월간: $33 × 20 = $660

Argo Rollouts 자동 운영:
- 초기 설정: 2시간 = $200 (일회성)
- 운영: 거의 비용 없음 (자동)
- 월간: $0

결론:
- 월 4회 이상 배포 시 Argo가 더 저렴
- 무중단성 + 자동 롤백까지 제공
```

## ⚖️ 트레이드오프

### 자동화 vs 제어

```
Argo Rollouts (완전 자동):
  ✅ 배포 시간 일정
  ✅ 야간/주말 배포 가능
  ✅ 빠른 롤백
  ❌ 복잡한 메트릭 설정 필요
  ❌ Prometheus 의존성
  ❌ 메트릭 오류로 인한 误 롤백 위험

수동 모니터링 (빠른 판단):
  ✅ 복잡한 상황 대응 가능
  ✅ 도메인 지식 활용
  ❌ 사람이 항상 대기해야 함
  ❌ 대응 속도 느림
```

### 메트릭 기반 자동 롤백의 위험

```
시나리오: 정상적인 요청 스파이크

08:00 배포 시작 (weight: 5%)
08:05 갑자기 요청 증가 (홍보 캠페인)
08:06 Prometheus 메트릭 보면:
      - 에러율: 정상 (1%)
      - 응답시간: 1.5초 (정상보다 높음)
      - CPU: 90% (매우 높음)
08:07 설정된 임계값 초과 → 자동 롤백
결과: 배포 실패 (정상이었는데도 롤백됨)

해결책:
1. 메트릭 임계값을 충분히 높게 설정
2. 여러 메트릭 조합 (AND 조건)
3. 초기 안정화 시간 설정 (initialDelay)
```

## 📌 핵심 정리

1. **Argo Rollouts의 핵심:**
   - Kubernetes Rollout 리소스 (Deployment와 유사)
   - 배포 step 정의 (setWeight, pause, analysis)
   - AnalysisTemplate으로 자동 메트릭 분석
   - 실패 시 자동 롤백

2. **AnalysisTemplate 구성:**
   ```
   metrics: [ ]
     ├─ name: 메트릭 이름
     ├─ interval: 체크 주기
     ├─ count: 몇 회 체크할 것인가
     ├─ failureLimit: 몇 회 실패하면 롤백
     └─ provider: Prometheus 쿼리
   ```

3. **배포 프로세스:**
   ```
   setWeight(5%) → pause(메트릭 수집)
   → analysis(자동 판단)
   → 성공하면 다음 step
   → 실패하면 자동 롤백
   ```

4. **언제 Argo를 쓸 것인가:**
   - ✅ 빈번한 배포 (일일 10회 이상)
   - ✅ 야간/주말 배포 필요
   - ✅ 자동 롤백 필수
   - ❌ Prometheus 없음
   - ❌ 배포 빈도 낮음 (월 1회)

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: Argo Rollouts 배포 중 AnalysisRun이 실패했다. 자동으로 롤백되는가?

<details>
<summary>정답 및 해설</summary>

**답:** 설정에 따라 다름

```yaml
# 배포 중단 (rollback은 아님)
- setWeight: 20
- analysis:
    templates:
    - name: error-rate
    # requiredForCompletion 설정 없음
    
→ AnalysisRun 실패 → 배포 중단
→ 기존 버전 유지 (자동 롤백 아님)
→ Rollout 상태: Failed

# 자동 롤백 설정
- setWeight: 20
- analysis:
    templates:
    - name: error-rate
    requiredForCompletion: true  # ← 이 설정이 중요
    
→ AnalysisRun 실패 → 자동 롤백
→ 모든 트래픽을 stable로 되돌림
→ Rollout 상태: Failed (하지만 자동 복구됨)
```

**결과:**
```
자동 롤백 단계:
1. AnalysisRun 실패 감지
2. Canary Pod의 weight를 0%로 설정
3. 모든 트래픽 stable로 이동 (초 단위)
4. Canary Deployment 스케일 다운 (또는 유지)
```

</details>

### 문제 2: Prometheus 쿼리 결과가 잘못되었다면?

```
Prometheus 쿼리:
  sum(irate(http_requests_total{status=~"5.."}[1m])) /
  sum(irate(http_requests_total[1m]))

실제 결과:
  0.5 (50% 에러율)
  
임계값 설정:
  error: "0.05" (5% 이상 에러 → 롤백)
  
결과: 배포 자동 롤백

문제: 쿼리가 잘못되었다면?
```

<details>
<summary>정답 및 해설</summary>

**가능한 쿼리 오류:**

1. **레이블 오류**
   ```
   # ❌ 잘못된 쿼리
   http_requests_total{status=~"5.."}  # 이 메트릭 없음
   
   # ✅ 올바른 쿼리
   http_request_total{status=~"5.."}   # 정확한 메트릭명
   ```

2. **나누기 0**
   ```
   # ❌ 분모가 0일 수 있음
   sum(irate(http_requests_total{status=~"5.."}[1m])) /
   sum(irate(http_requests_total[1m]))
   
   → 요청이 없으면 NaN → 메트릭 오류 처리
   
   # ✅ 분모 체크
   (sum(irate(http_requests_total{status=~"5.."}[1m])) /
    sum(irate(http_requests_total[1m])) > 0)
    or 0
   ```

3. **시계열 없음**
   ```
   # ❌ 아직 메트릭이 수집되지 않음
   irate(http_requests_total[1m])
   
   → 처음 1분은 데이터 없음 → NaN
   
   # ✅ 초기 안정화 시간 설정
   initialDelay: 60s  # 1분 후 메트릭 조회 시작
   ```

**해결책:**
```yaml
spec:
  metrics:
  - name: error-rate
    initialDelay: 30s      # ← Pod 시작 후 안정화 대기
    interval: 30s
    count: 3
    failureLimit: 1
    
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          (sum(irate(http_requests_total{status=~"5.."}[1m])) /
          sum(irate(http_requests_total[1m])) > 0)
          or 0
        
        # NaN 처리
        successCriteria: "isNumber($value)"
        
        thresholds:
          error: "0.05"
```

</details>

### 문제 3: Rollout 배포 중 Pod crash가 발생했다. Argo는 어떻게 처리하는가?

<details>
<summary>정답 및 해설</summary>

**시나리오:**
```
setWeight: 20 → canary Pod 에러 발생

canary Pod의 실제 상태:
- 이미지: myapp:2.0 (문제 있는 코드)
- Pod 상태: CrashLoopBackOff
- Readiness Probe: 실패
```

**Argo의 동작:**
```
1단계: Pod이 Ready 상태 되기를 대기
       initialDelaySeconds 경과 → readinessProbe 체크
       → 실패 (Pod이 crash 중)
       
2단계: Rollout이 이를 감지
       Pod 수가 desired 수만큼 Ready가 아님
       
3단계: progressDeadlineSeconds 초과 (기본값 600초)
       → Rollout 상태: Failed
       
4단계: 자동 롤백?
       → 설정에 따라 다름
       
5단계: Pod이 계속 crash 상태
       kubelet이 재시작 시도 (backoff)
```

**최악의 경우:**
```
- 배포 중 5분 경과
- Pod은 여전히 CrashLoopBackOff
- 트래픽은 여전히 stable으로 흐름 (안전)
- 하지만 canary Pod은 계속 리소스 낭비
```

**해결책:**
```yaml
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 5m}      # ← Pod crash 감지 시간
      - analysis:
          templates:
          - name: pod-health       # Pod 상태 체크
  
  progressDeadlineSeconds: 300  # ← 5분 초과 시 실패
  
---
# Pod 상태 분석 template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: pod-health
spec:
  metrics:
  - name: pod-ready
    interval: 30s
    count: 5
    failureLimit: 1
    provider:
      kubernetes:
        query: "podReadyCount"  # 준비된 Pod 수
```

**결론:**
```
Pod crash는 AnalysisTemplate에서 감지해야 함
Argo Rollouts 자체는 Pod crash를 "배포 실패"로만 봄
따라서 Pod 상태를 명시적으로 모니터링 필요
```

</details>

---

<div align="center">

**[⬅️ 이전: Canary 배포 — 트래픽을 단계적으로](./03-canary-deployment.md)** | **[홈으로 🏠](../README.md)** | **[다음: 롤백 전략 — 언제 되돌릴 수 없는가 ➡️](./05-rollback-strategy.md)**

</div>
