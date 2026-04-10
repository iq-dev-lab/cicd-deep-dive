# Canary 배포 — 트래픽을 단계적으로 이동하기

## 🎯 핵심 질문
- Canary 배포는 정확히 어떤 원리로 작동하는가?
- Kubernetes Ingress의 canary annotation이 정확히 무엇인가?
- 트래픽을 5% → 20% → 50% → 100%로 단계적으로 이동할 때 어떤 일이 발생하는가?
- 에러율 모니터링 후 자동 롤백을 구현할 수 있는가?

## 🔍 왜 이 개념이 실무에서 중요한가

Rolling Update와 Blue-Green 배포의 중간 지점이 Canary 배포입니다:
- **Rolling Update는 빠르지만 혼재 기간이 길다** (최소 readiness 시간 × replicas)
- **Blue-Green은 무중단이지만 비용 2배** (두 환경 동시 유지)
- **Canary는 위험을 최소화하면서 비용도 절감** (작은 트래픽으로 테스트 후 확대)

문제는:
1. **5% 사용자에게만 새 버전 노출했는데 어떻게 트래픽을 정확히 분산하는가?**
2. **에러 감지 후 자동으로 롤백할 수 있는가?**
3. **특정 사용자 (VIP, 내부 팀)만 새 버전을 먼저 테스트할 수 있는가?**

Canary의 메커니즘을 이해하면 이 모든 것이 가능합니다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 이해: Deployment를 2개 만들고 끝
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: app
        image: myapp:1.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # "1개만 배포하니까 5% 정도?"
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: myapp:2.0
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    # version label 없음 → stable + canary 모두 매치
  ports:
  - port: 80
    targetPort: 8080
```

**이 방식의 문제점:**
1. **트래픽 분산이 불명확함**
   - stable Pod 3개 + canary Pod 1개 = 4개 Pod
   - 트래픽: 3/4 = 75% stable, 1/4 = 25% canary (5% 아님!)

2. **트래픽 비율 조정 불가능**
   - 5%로 조정하려면? Pod 수를 조정? (19개 필요)
   - Pod 수 조정은 부하를 같다고 가정 (실제로는 아님)

3. **Ingress의 가중치 기반 라우팅 미사용**
   - 정확한 비율 제어 불가능

4. **자동 롤백 메커니즘 없음**
   - 에러 발생 시 수동으로 canary Pod 제거해야 함

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 1단계: Nginx Ingress 기반 Canary 배포

```yaml
# 먼저 Stable (기존) 버전 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# 새 Canary (실험) 버전 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # 최소한의 Pod (부하 가중을 통해 실제로는 5% 정도)
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: myapp:2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Stable용 Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-stable
spec:
  selector:
    app: myapp
    version: stable
  ports:
  - port: 80
    targetPort: 8080
---
# Canary용 Service (별도)
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary
spec:
  selector:
    app: myapp
    version: canary
  ports:
  - port: 80
    targetPort: 8080
---
# ✅ Stable Ingress (메인 라우팅)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-stable
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-stable
            port:
              number: 80
---
# ✅ Canary Ingress (가중치 기반 라우팅)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"           # 이 Ingress는 canary다
    nginx.ingress.kubernetes.io/canary-weight: "5"       # stable의 5%만 이 서비스로
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"  # Header 기반 라우팅도 가능
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

**배포 프로세스:**
```
1단계: Stable 운영 (100% 트래픽)
       Canary Pod: 아직 배포 안 함

2단계: Canary Ingress 생성 (canary-weight: 5)
       이제 5% 트래픽이 canary로 흐름
       
3단계: 모니터링 (에러율, 응답시간, 로그)
       - 정상 → 다음 단계로
       - 에러 → canary-weight: 0으로 변경 (즉시 롤백)
       
4단계: canary-weight: 5 → 20 로 증가
       5분 모니터링
       
5단계: canary-weight: 20 → 50 로 증가
       5분 모니터링
       
6단계: canary-weight: 50 → 100 으로 변경
       Canary가 메인 버전이 됨
       
7단계: Stable Deployment 종료 (replicas=0)
       Canary Ingress만 유지 (또는 삭제)
```

## 🔬 내부 동작 원리

### Nginx Ingress Canary 메커니즘

```
사용자 요청 (HTTP)
  ↓
Nginx Ingress Controller
  ↓
어느 Ingress 규칙을 쓸 것인가? 결정
  ├─ myapp-stable Ingress
  └─ myapp-canary Ingress (canary: "true")
  
canary Ingress 선택 로직:
  1. canary-by-header가 설정되어 있는가?
     → 해당 header 값으로 라우팅 결정
     예: X-Canary: true → canary로 라우팅
     
  2. canary-by-cookie가 설정되어 있는가?
     → 해당 cookie 값으로 결정
     
  3. canary-weight가 설정되어 있는가?
     → 확률적으로 트래픽 분산
     예: canary-weight: 10 → 10/100 확률로 canary로 라우팅
     
  4. 위 모두 아니면? → stable로 라우팅

결과:
  ├─ canary로 라우팅된 요청 → myapp-canary Service
  │                        → canary Deployment의 Pod
  │
  └─ stable로 라우팅된 요청 → myapp-stable Service
                           → stable Deployment의 Pod
```

### canary-weight의 정확한 작동

```
canary-weight: 5 설정 시

Nginx 내부 동작:
  요청 1 → hash(X-Real-IP) % 100 = 15 → 5보다 크다 → stable로 라우팅
  요청 2 → hash(X-Real-IP) % 100 = 3  → 5보다 작다 → canary로 라우팅
  요청 3 → hash(X-Real-IP) % 100 = 45 → 5보다 크다 → stable로 라우팅
  ...

결과:
  - 각 사용자별로 일관된 라우팅 (같은 사용자는 같은 버전)
  - 전체 트래픽의 약 5%가 canary로 분산
  - "약"인 이유: hash 기반이라 정확히 5%가 아닐 수도 있음
```

### Header 기반 Canary (특정 사용자만 테스트)

```yaml
# VIP 사용자만 canary 버전 테스트
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary-vip
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
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

```
사용자 요청:
  curl -H "X-Canary: true" https://myapp.example.com
  → canary 버전으로 라우팅 (내부 팀/QA 테스트용)
  
일반 사용자 요청:
  curl https://myapp.example.com
  → stable 버전으로 라우팅
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 사전 준비: Nginx Ingress Controller 설치

```bash
# Nginx Ingress Controller 설치 (없으면 설치)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace

# 설치 확인
kubectl get pods -n ingress-nginx
```

### 실험 1: 기본 Canary 배포

```bash
# 1. Stable 버전 배포
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
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
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-stable
spec:
  selector:
    app: myapp
    version: stable
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-stable
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-stable
            port:
              number: 80
EOF

# 2. Pod Ready 대기
kubectl wait --for=condition=ready pod -l version=stable --timeout=60s

# 3. Stable Ingress IP 확인
kubectl get ingress myapp-stable
```

### 실험 2: Canary 배포 시작 (5% 트래픽)

```bash
# 1. Canary Deployment 배포
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: nginx:1.20
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary
spec:
  selector:
    app: myapp
    version: canary
  ports:
  - port: 80
    targetPort: 80
EOF

# 2. Pod Ready 대기
kubectl wait --for=condition=ready pod -l version=canary --timeout=60s

# 3. Canary Ingress 생성 (5% 트래픽)
cat <<'EOF' | kubectl apply -f -
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
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
EOF

# 4. 트래픽 확인 (Pod 로그로)
kubectl logs -f -l version=stable --max-log-requests=5 &
kubectl logs -f -l version=canary --max-log-requests=5 &

# 5. 테스트 트래픽 발생
for i in {1..100}; do
  curl -H "Host: myapp.local" http://localhost -v 2>&1 | grep "< HTTP"
done

# 결과: stable에 약 95개, canary에 약 5개 요청
```

### 실험 3: Canary Weight 단계적 증가

```bash
# 1단계: 5% → 20%
kubectl patch ingress myapp-canary -p \
  '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"20"}}}'

# 트래픽이 즉시 변경됨을 확인
for i in {1..100}; do
  curl -H "Host: myapp.local" http://localhost -v 2>&1 | grep "< HTTP"
done

# 2단계: 20% → 50%
kubectl patch ingress myapp-canary -p \
  '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"50"}}}'

# 3단계: 50% → 100% (Canary가 메인이 됨)
kubectl patch ingress myapp-canary -p \
  '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"100"}}}'

# 4. Stable Deployment 종료
kubectl scale deployment myapp-stable --replicas=0

# 확인
kubectl get pods -l app=myapp
# canary Pod만 3개로 스케일 업된 것 보임
```

### 실험 4: Header 기반 Canary (특정 사용자만 테스트)

```bash
# 1. Header 기반 Canary Ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary-header
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary-Test"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
EOF

# 2. 일반 요청 (Stable으로 라우팅)
curl -H "Host: myapp.local" http://localhost

# 3. Header 있는 요청 (Canary로 라우팅)
curl -H "Host: myapp.local" -H "X-Canary-Test: true" http://localhost

# 4. 로그 확인
kubectl logs -l version=canary -c app | grep "X-Canary-Test"
# 이 header를 받은 요청만 canary에서 처리됨
```

### 실험 5: 에러 감지 및 자동 롤백 스크립트

```bash
#!/bin/bash
# canary-deploy-with-monitoring.sh

NAMESPACE="default"
STABLE_DEPLOYMENT="myapp-stable"
CANARY_DEPLOYMENT="myapp-canary"
CANARY_INGRESS="myapp-canary"
SERVICE_IP="$(kubectl get svc nginx-ingress -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"

echo "Service IP: $SERVICE_IP"

# Canary 배포 및 단계적 트래픽 전환
CANARY_WEIGHTS=(5 20 50 100)
MONITORING_INTERVAL=30  # 각 단계마다 30초 모니터링

for weight in "${CANARY_WEIGHTS[@]}"; do
  echo "=== Increasing canary weight to: $weight% ==="
  
  # Weight 업데이트
  kubectl patch ingress $CANARY_INGRESS \
    -p "{\"metadata\":{\"annotations\":{\"nginx.ingress.kubernetes.io/canary-weight\":\"$weight\"}}}"
  
  # 모니터링
  echo "Monitoring for $MONITORING_INTERVAL seconds..."
  ERROR_COUNT=0
  SUCCESS_COUNT=0
  
  for i in $(seq 1 $MONITORING_INTERVAL); do
    # 테스트 요청
    HTTP_CODE=$(curl -s -w "%{http_code}" -o /dev/null \
      -H "Host: myapp.local" http://$SERVICE_IP)
    
    if [ "$HTTP_CODE" -lt 400 ]; then
      SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    else
      ERROR_COUNT=$((ERROR_COUNT + 1))
      echo "❌ Error detected: HTTP $HTTP_CODE"
    fi
    
    sleep 1
  done
  
  ERROR_RATE=$((ERROR_COUNT * 100 / (SUCCESS_COUNT + ERROR_COUNT)))
  echo "✓ Success: $SUCCESS_COUNT, ✗ Errors: $ERROR_COUNT, Error Rate: $ERROR_RATE%"
  
  # 에러율이 10%를 초과하면 롤백
  if [ $ERROR_RATE -gt 10 ]; then
    echo "❌ Error rate exceeded 10%! Rolling back..."
    kubectl patch ingress $CANARY_INGRESS \
      -p "{\"metadata\":{\"annotations\":{\"nginx.ingress.kubernetes.io/canary-weight\":\"0\"}}}"
    echo "✓ Rolled back to stable version"
    exit 1
  fi
done

echo "=== Canary deployment complete ==="
```

```bash
# 실행
chmod +x canary-deploy-with-monitoring.sh
./canary-deploy-with-monitoring.sh
```

### 실험 6: Cookie 기반 Canary (특정 사용자 세션)

```yaml
# Cookie 값에 따라 canary 버전으로 라우팅
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary-cookie
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "version"
    nginx.ingress.kubernetes.io/canary-by-cookie-value: "v2"
spec:
  rules:
  - host: myapp.local
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

```bash
# 일반 요청
curl -H "Host: myapp.local" http://localhost
# → stable

# Cookie 있는 요청
curl -H "Host: myapp.local" -H "Cookie: version=v2" http://localhost
# → canary (같은 사용자는 계속 canary로 라우팅)
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 항목 | Rolling Update | Blue-Green | Canary |
|------|---|---|---|
| **총 배포 시간** | 25-40초 | 30-60초 | 5-10분 (단계별) |
| **트래픽 전환** | 순차적 | 즉시 | 단계적 |
| **롤백 시간** | 25-40초 | <1초 | <1초 (weight=0) |
| **위험도** | 높음 (혼재) | 낮음 (무중단) | 낮음 (점진적) |
| **추가 비용** | 낮음 | 높음 (2배) | 중간 (5-10% 추가) |
| **에러율 감지** | 불가능 | 가능 (수동) | 가능 (자동) |
| **특정 사용자 테스트** | 불가능 | 가능 (VIP) | 가능 (Header/Cookie) |

### 실제 비용 계산

```
Canary weight: 5% 설정 시

필요 Pod 수:
- Stable: 3개 (95% 트래픽 처리)
- Canary: 1개 (5% 트래픽 처리, 실제로는 부하가 적음)
- 총 4개 Pod

추가 비용: 1개 Pod × 배포 시간(5-10분) ÷ 월 시간(720시간)
- 월 추가 비용 = $50 × (5-10분) ÷ 720시간 × 30일
- = $50 × (0.1-0.2시간) ÷ 720 = $0.007-$0.014/배포
- 일일 배포 10회: $0.07-$0.14/월 (무시할 수 있는 수준)
```

## ⚖️ 트레이드오프

### 단계적 배포 vs 빠른 배포

```
Canary (5% → 100%):
  ✅ 위험 최소화
  ✅ 에러 조기 발견
  ❌ 배포 시간 길음 (5-10분)
  ❌ 복잡한 모니터링 필요

Blue-Green (즉시 100%):
  ✅ 빠른 배포
  ❌ 위험이 큼 (즉시 전체 트래픽)
  ✅ 롤백 빠름
```

### 특정 사용자 테스트

```
Canary (Header/Cookie 기반):
  ✅ VIP 사용자만 먼저 테스트
  ✅ 내부 팀 먼저 검증
  ✅ 트래픽 비율과 상관없이 테스트
  
  ❌ 구현 복잡 (애플리케이션에서 header 전달 필요)

일반 가중치 기반:
  ✅ 구현 간단
  ❌ 임의의 사용자가 테스트받음
```

## 📌 핵심 정리

1. **Canary 배포의 원리:**
   - Stable과 Canary 2개 Deployment 유지
   - Nginx Ingress의 canary annotation으로 트래픽 분산
   - canary-weight: 0 → 5 → 20 → 50 → 100으로 단계적 증가

2. **트래픽 분산 방식:**
   - `canary-weight: 5`: 5% 확률적 라우팅
   - `canary-by-header`: 특정 header 값 기반
   - `canary-by-cookie`: 특정 cookie 값 기반
   - 혼합 가능 (weight + header 동시 사용)

3. **모니터링 지표:**
   - 에러율 (5xx, 4xx)
   - P99 응답시간
   - 데이터베이스 쿼리 시간
   - 외부 API 응답시간

4. **자동 롤백 조건:**
   ```
   에러율 > 5% → weight: 0으로 변경
   P99 응답시간 > 1초 → weight: 0으로 변경
   ```

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: Canary weight: 5로 설정했을 때, 정확히 5%의 트래픽이 canary로 가는가?

<details>
<summary>정답 및 해설</summary>

**답:** 약 5%, 정확히 5%는 아님

**이유:**
```
Nginx의 canary-weight 구현:
  hash(request) % 100 < canary_weight

예시: canary_weight=5
  요청의 hash 값에 따라
  0-4: canary로 라우팅 (5%)
  5-99: stable로 라우팅 (95%)

하지만:
1. Hash 함수는 완벽한 분산을 보장하지 않음
2. 실제 트래픽은 균등하지 않을 수 있음
3. 작은 샘플 크기에서는 통계 편차 발생

결과:
- 요청 100개: 약 3-7개 canary로 가능
- 요청 10000개: 약 4.8-5.2% canary로 수렴
```

**정확한 비율 필요 시:**
- 더 정교한 로드밸런서 사용 (예: Istio)
- 또는 애플리케이션 레벨에서 제어

</details>

### 문제 2: Header 기반 Canary 테스트 중, 일부 요청이 header를 잃어버렸다면?

<details>
<summary>정답 및 해설</summary>

**시나리오:**
```
1. 사용자 브라우저에서 curl 테스트:
   curl -H "X-Canary: true" http://myapp.example.com
   
2. Nginx Ingress에서 header 확인:
   X-Canary: true → canary로 라우팅

3. 애플리케이션 로그:
   X-Canary header가 없음!

왜?
```

**원인:**
```
1. 리다이렉트 (301/302)
   - 원래 요청: X-Canary header 포함
   - 리다이렉트 응답
   - 클라이언트가 따라가는 요청: header 손실 가능
   
2. 프록시/CDN
   - CloudFlare, AWS CloudFront 등이 header 제거
   
3. 브라우저 정책
   - 교차 도메인 요청 시 header 자동 제거
   
4. 애플리케이션 미지원
   - 애플리케이션이 header를 전파하지 않음
```

**해결책:**
```
1. Header 검증
   kubectl logs pod-name | grep "X-Canary"
   → 실제로 전달되는지 확인

2. Cookie 사용 (더 안정적)
   - 브라우저가 자동으로 유지
   - 리다이렉트에도 포함됨
   
3. URL Parameter 사용
   ?canary=true 형태
   - 리다이렉트에도 유지
```

</details>

### 문제 3: Canary weight를 5 → 50으로 한 번에 변경했을 때, 즉시 50%의 트래픽이 전환되는가? 아니면 시간이 걸리는가?

<details>
<summary>정답 및 해설</summary>

**답:** 즉시 전환됨 (밀리초 단위)

**이유:**
```
kubectl patch ingress로 annotation 변경
  ↓
Kubernetes API Server 업데이트
  ↓
Nginx Ingress Controller 감지 (1-2초)
  ↓
Nginx 설정 파일 재로드
  ↓
기존 TCP 연결: 영향 없음
새 요청: 즉시 새로운 weight로 라우팅
```

**결과:**
- 진행 중인 요청: 계속 기존 버전 처리
- 새로운 요청: 즉시 새로운 비율로 분산
- 전환 시간: 1-2초

**주의:**
```
하지만 트래픽 급변으로 인한 부작용:
- Stable Pod 부하 급감 → 메모리 여유
- Canary Pod 부하 급증 → CPU 스파이크
- 타이밍이 좋지 않으면 Canary Pod crash 가능

해결책:
1. 천천히 증가 (10% → 20% → 30% 단위)
2. 각 단계마다 2-3분 모니터링
3. Pod autoscaling 설정 (HPA)
```

</details>

---

<div align="center">

**[⬅️ 이전: Blue-Green 배포 — 즉각 롤백의 비용](./02-blue-green-deployment.md)** | **[홈으로 🏠](../README.md)** | **[다음: Argo Rollouts — 메트릭 기반 자동 승급 ➡️](./04-argo-rollouts.md)**

</div>
