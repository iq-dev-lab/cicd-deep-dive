# Rolling Update 완전 분해 — maxSurge와 Readiness Probe

## 🎯 핵심 질문
- Deployment의 Rolling Update는 정확히 어떤 순서로 Pod를 교체하는가?
- `maxSurge`와 `maxUnavailable`을 설정하지 않으면 어떤 일이 발생하는가?
- Readiness Probe가 실패하면 Rolling Update가 중단되는가?
- `progressDeadlineSeconds` 초과 시 어떤 상태가 되는가?

## 🔍 왜 이 개념이 실무에서 중요한가

Kubernetes Deployment의 Rolling Update는 가장 널리 사용되는 배포 전략입니다. 그런데 많은 엔지니어들이 `kubectl rollout status`만 보고 배포가 완료되었다고 생각하곤 합니다.

문제는 여기서 시작됩니다:
- **무중단 배포인 줄 알았는데 갑자기 5xx 에러가 떨어진다** → Readiness Probe 설정 부재
- **배포 중간에 갑자기 롤아웃이 중단된다** → `progressDeadlineSeconds` 초과
- **배포 속도가 너무 느리다** → `maxSurge`를 보수적으로 설정한 결과
- **메모리 부족으로 새 Pod이 스케줄되지 않는다** → `maxSurge`로 동시 Pod 수를 제어하지 않음

Rolling Update의 내부 동작을 정확히 이해하면, 이 모든 문제를 사전에 예방할 수 있습니다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 위험한 설정: 무신경한 Rolling Update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    # rollingUpdate 섹션이 생략됨 → 기본값 사용
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
        image: myapp:v1
        # ⚠️ Readiness Probe가 없다!
        # 새 Pod이 Ready 상태인지 확인하지 않고 트래픽 전달
```

**이 설정의 문제점:**
1. 기본값 `maxSurge=1`, `maxUnavailable=1` 사용 → 배포가 매우 느림 (3개 Pod 배포에 20분 이상)
2. Readiness Probe 부재 → 애플리케이션 시작 중인 Pod에 즉시 트래픽 전달 → **503 에러 폭주**
3. `progressDeadlineSeconds`의 기본값 600초(10분) 초과 가능 → 갑작스런 롤아웃 중단
4. 장애 감지 후 대응할 메커니즘 없음 → 반은 새 버전, 반은 구 버전 상태로 중단될 가능성

**실제로 발생하는 상황:**
```
배포 시작 (v1: 3개 Pod)
  ↓
Rolling Update 시작 (maxSurge=1이므로 최대 동시 4개 Pod)
  ↓
v2 Pod 1개 생성, 즉시 트래픽 전달 (Readiness Probe 없음)
  ↓
v2 Pod이 아직 DB 마이그레이션 중... 
  ↓
사용자: 503 Service Unavailable 에러 수신
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 동시에 1개 추가 Pod 생성 가능 (또는 "50%")
      maxUnavailable: 0  # 배포 중 사용 불가능한 Pod 수 0 → 무중단
  progressDeadlineSeconds: 600  # 10분 내 배포 완료, 안 되면 자동 중단
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
        image: myapp:v2
        ports:
        - containerPort: 8080
        
        # ✅ Readiness Probe: 애플리케이션이 정말 준비됐는지 확인
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10    # 컨테이너 시작 후 10초 대기
          periodSeconds: 5           # 5초마다 체크
          timeoutSeconds: 2          # 2초 내 응답 필요
          successThreshold: 1        # 1회 성공으로 Ready 판정
          failureThreshold: 3        # 3회 실패 시 Not Ready
        
        # ✅ Liveness Probe: Pod이 살아있는지 확인 (데드락 감지)
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

**올바른 배포 흐름:**
```
배포 시작 (v1: 3개 Pod, 모두 Ready)
  ↓
v2 Pod 1개 생성
  ↓
v2 Pod 내부: 애플리케이션 시작 중
  Readiness Probe 실패 (앱이 아직 시작 중)
  ↓
v2 Pod이 /health/ready 응답 시작 (약 15초 소요)
  ↓
Readiness Probe 성공 → Pod을 Ready로 판정
  ↓
이제 트래픽 전달 시작 (Service selector에 포함됨)
  ↓
v1 Pod 1개 종료 (graceful shutdown)
  ↓
계속 진행... v2 Pod 2개, 3개 순차 생성
```

## 🔬 내부 동작 원리

### Rolling Update의 3가지 주요 파라미터

#### 1. maxSurge: 추가로 생성할 수 있는 Pod 수

```
maxSurge: 1 또는 "50%"

의미: 동시에 존재할 수 있는 최대 Pod 수 = replicas + maxSurge

예시 (replicas=3, maxSurge=1):
  시작: v1 Pod 3개 (총 3개)
  
  배포 단계 1: v2 Pod 1개 추가 (총 4개)
             → v2가 Ready 될 때까지 대기
             → v2 Ready 됨
             
  배포 단계 2: v1 Pod 1개 제거 (총 3개)
             → 다시 여유 생김
             → v2 Pod 1개 추가 (총 4개)
             
  배포 단계 3: v2 Ready, v1 1개 제거 (총 3개)
  
  배포 단계 4: v2 Pod 1개 추가 (총 4개)
  배포 단계 5: v2 Ready, v1 1개 제거 (총 3개)
  
결과: v2 Pod 3개, v1 Pod 0개 (배포 완료)
소요 시간: readinessProbe 시간 × 3 = 약 45초
```

#### 2. maxUnavailable: 동시에 사용 불가능할 수 있는 Pod 수

```
maxUnavailable: 1 또는 "33%"

의미: 동시에 Not Ready인 Pod 수의 최대값

예시 1 (replicas=3, maxUnavailable=1):
  시작: v1 Pod 3개 (모두 Ready)
  
  배포 단계 1: v1 Pod 1개 제거 (v1 2개 Ready, 사용 불가능 1개)
             → 즉시 v2 Pod 1개 생성
             → v2 Ready될 때까지 대기
             
결과: 항상 최소 2개의 Pod이 Ready 상태 유지 → 5xx 에러 가능성 감소

예시 2 (replicas=3, maxUnavailable=0):
  시작: v1 Pod 3개 (모두 Ready)
  
  배포 단계 1: v2 Pod 1개 추가 생성 (v1 3개 + v2 1개 = 4개)
             → v2 Ready 될 때까지 대기
             
  배포 단계 2: v2 Ready, v1 Pod 1개 제거 (v1 2개 + v2 1개 = 3개)
  
  배포 단계 3: v2 Pod 1개 추가 (v1 2개 + v2 2개 = 4개)
  
  배포 단계 4: v2 Ready, v1 Pod 1개 제거 (v1 1개 + v2 2개 = 3개)
  
  배포 단계 5: v2 Pod 1개 추가 (v1 1개 + v2 3개 = 4개)
  
  배포 단계 6: v2 Ready, v1 Pod 1개 제거 (v2 3개 = 3개)

결과: 항상 정확히 3개의 Ready Pod 유지 → 무중단 배포 보장
```

### Readiness Probe 동작 메커니즘

```
Pod 생성 → Container 시작 → initialDelaySeconds 대기
  ↓
readinessProbe 체크 시작 (주기: periodSeconds)
  ↓
HTTP GET /health/ready → 응답 대기 (timeoutSeconds)
  ↓
성공 (200-399)? → successThreshold 카운터 ++
  ↓
카운터 == successThreshold?
  ├─ Yes → Pod을 Ready로 판정
  │         (Service의 Endpoints 객체에 Pod IP 추가)
  │
  └─ No → 계속 대기
  
실패 (기타 상태)?→ failureThreshold 카운터 ++
  ↓
카운터 == failureThreshold?
  ├─ Yes → Pod을 NotReady로 판정
  │         (Service의 Endpoints 객체에서 Pod IP 제거)
  │
  └─ No → 계속 대기
```

### progressDeadlineSeconds 동작

```
Rolling Update 시작 → 타이머 시작 (기본값: 600초)
  ↓
매 30초마다 체크:
  "지정된 replicas 수만큼 Updated & Ready 상태의 Pod이 있는가?"
  
  ├─ Yes → 타이머 리셋, 계속 진행
  ├─ No → 타이머 카운트 다운
  
progressDeadlineSeconds 초과?
  ├─ Yes → Deployment 상태 = "Progressing: False"
  │         새로운 ReplicaSet 생성 중단
  │         기존 ReplicaSet은 유지 (부분 배포 상태)
  │
  └─ No → 계속 진행
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 실험 1: 기본 Rolling Update 관찰

```bash
# 1. 초기 Deployment 생성
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-demo
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  progressDeadlineSeconds: 60
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
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
          periodSeconds: 2
EOF

# 2. 롤아웃 상태 실시간 관찰
kubectl rollout status deployment/rolling-update-demo --timeout=3m

# 3. 배포 진행 중 상태 관찰 (다른 터미널)
watch kubectl get pods -l app=demo

# 4. ReplicaSet 확인 (v1만 존재)
kubectl get rs -l app=demo
# 출력 예:
# NAME                           DESIRED   CURRENT   READY   AGE
# rolling-update-demo-abc123     3         3         3       2m
```

### 실험 2: 새 버전으로 업데이트

```bash
# 1. 이미지 업데이트 (v1.20으로)
kubectl set image deployment/rolling-update-demo app=nginx:1.20

# 2. 롤아웃 진행 상황 실시간 관찰
kubectl rollout status deployment/rolling-update-demo

# 3. 상세 이벤트 확인 (ReadinesProbe 체크 등)
kubectl describe deployment rolling-update-demo

# 4. 배포 히스토리 확인
kubectl rollout history deployment/rolling-update-demo
# 출력:
# revision 1
# revision 2   <- 현재 배포 버전

# 5. 특정 revision의 상세 정보
kubectl rollout history deployment/rolling-update-demo --revision=2
```

### 실험 3: Readiness Probe 실패 시나리오

```yaml
# ❌ Readiness Probe가 계속 실패하는 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-fail-demo
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  progressDeadlineSeconds: 60
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
      - name: app
        image: nginx:1.20
        readinessProbe:
          httpGet:
            path: /this-path-does-not-exist  # ❌ 404 에러 → Readiness 실패
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 2
          failureThreshold: 3
```

```bash
# 이 Deployment를 배포하면:
kubectl apply -f readiness-fail-demo.yaml

# 잠시 후 상태 확인:
kubectl get pods -l app=readiness-demo

# NAME                                  READY   STATUS    RESTARTS   AGE
# readiness-fail-demo-xyz789-a1b2c      0/1     Running   0          30s
# readiness-fail-demo-xyz789-d4e5f      0/1     Running   0          25s
# readiness-fail-demo-old-abc123-11111  1/1     Running   0          2m

# 롤아웃 상태 확인:
kubectl rollout status deployment/readiness-fail-demo

# 출력:
# Waiting for deployment "readiness-fail-demo" to roll out...
# error: deployment "readiness-fail-demo" exceeded its progress deadline

# progressDeadlineSeconds 60초 초과 → 롤아웃 자동 중단
# 상태: 부분 배포 (구 Pod + 새 Pod 혼재)
```

### 실험 4: maxSurge와 maxUnavailable 성능 비교

```yaml
# 설정 A: 보수적 (느린 배포)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-rollout
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 동시 6개 Pod
      maxUnavailable: 1    # 최소 4개 Ready
  selector:
    matchLabels:
      app: slow
  template:
    metadata:
      labels:
        app: slow
    spec:
      containers:
      - name: app
        image: nginx:1.20
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 2
---
# 설정 B: 공격적 (빠른 배포)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fast-rollout
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # 동시 7개 Pod
      maxUnavailable: 0    # 항상 5개 Ready
  selector:
    matchLabels:
      app: fast
  template:
    metadata:
      labels:
        app: fast
    spec:
      containers:
      - name: app
        image: nginx:1.20
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 2
```

```bash
# 성능 비교 실험
kubectl apply -f slow-rollout.yaml
kubectl apply -f fast-rollout.yaml

# 초기 상태 대기
sleep 30

# 두 Deployment 동시에 업데이트 시작
time kubectl set image deployment/slow-rollout app=nginx:1.21 &
time kubectl set image deployment/fast-rollout app=nginx:1.21 &

# 두 롤아웃이 완료될 때까지 관찰
watch kubectl rollout status deployment/slow-rollout
watch kubectl rollout status deployment/fast-rollout

# 결과:
# slow-rollout: ~40초 (1개씩 순차 교체, 총 5회 반복)
# fast-rollout: ~15초 (2개씩 병렬 교체, 총 3회 반복)
```

### 실험 5: Deployment 상세 상태 추적

```bash
# Pod 생성부터 배포 완료까지의 모든 이벤트 확인
kubectl get events -l app=demo --sort-by='.lastTimestamp'

# 롤아웃 상태 JSON 형식으로 확인 (프로그래밍)
kubectl get deployment rolling-update-demo -o json | \
  jq '.status'

# 출력:
# {
#   "observedGeneration": 2,
#   "replicas": 3,
#   "updatedReplicas": 3,        # 새 버전으로 업데이트된 Pod 수
#   "readyReplicas": 3,           # Ready 상태의 Pod 수
#   "availableReplicas": 3,       # Available 상태의 Pod 수
#   "conditions": [
#     {
#       "type": "Progressing",
#       "status": "True",
#       "reason": "NewReplicaSetAvailable",
#       "message": "ReplicaSet \"...\" has successfully progressed"
#     }
#   ]
# }

# 실시간으로 상태 변화 감시
kubectl rollout status deployment/rolling-update-demo --watch
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 설정 | maxSurge | maxUnavailable | 배포 시간 (5개 Pod) | 메모리 피크 | 가용성 | 비용 영향 |
|------|----------|---|---|---|---|---|
| 보수적 | 1 | 1 | ~40초 | +1개 (20%) | 중간 (최소 3개) | 낮음 |
| 균형 | 1 | 0 | ~25초 | +1개 (20%) | 높음 (항상 5개) | 낮음 |
| 공격적 | 2 | 0 | ~15초 | +2개 (40%) | 높음 (항상 5개) | 중간 |
| 초고속 | "50%" | 0 | ~8초 | +2-3개 (40-60%) | 높음 (항상 5개) | 높음 |

**실제 비용 분석 (AWS t3.medium 기준):**
```
1개 Pod 월 비용: $15

배포 1회마다 추가 비용:
- maxSurge=1: 추가 Pod 1개 × (배포 시간 40초) = $15 × (40/2592000) ≈ $0.0002
- maxSurge=2: 추가 Pod 2개 × (배포 시간 15초) = $15 × 2 × (15/2592000) ≈ $0.0002

결론: 배포 1회당 추가 비용은 무시할 수 있음 (수 센트 수준)
하지만 일일 배포 100회 기준: $0.02 × 100 = $2/일
```

## ⚖️ 트레이드오프

### maxSurge 증가 vs 배포 시간

```
maxSurge = 1 → 배포 시간 길음, 메모리 적게 사용, 점진적 배포
maxSurge = "50%" → 배포 시간 짧음, 메모리 많이 사용, 신속한 배포
```

**선택 기준:**
- CI/CD 자동화 환경: `maxSurge = 1` (안정적, 비용 절감)
- 긴급 hotfix 배포: `maxSurge = "50%"` (속도 우선)

### maxUnavailable = 0 vs 빠른 배포

```
maxUnavailable = 0 → 항상 가용 Pod 유지, 무중단, 메모리 추가 사용
maxUnavailable = 1 → 빠른 배포, 메모리 절약, 가용성 약간 저하
```

**선택 기준:**
- 미션 크리티컬 서비스: `maxUnavailable = 0` (무중단 필수)
- 배경 작업 서비스: `maxUnavailable = "33%"` (속도 우선)

### Readiness Probe 엄격함 vs 배포 속도

```
엄격한 Probe (failureThreshold=3) → 배포 느림, 안정성 높음
느슨한 Probe (failureThreshold=1) → 배포 빠름, 장애 가능성 있음
```

## 📌 핵심 정리

1. **Rolling Update의 3가지 핵심 파라미터:**
   - `maxSurge`: 동시 생성 Pod 수 제어 (배포 속도)
   - `maxUnavailable`: 동시 불가 Pod 수 제어 (가용성)
   - `progressDeadlineSeconds`: 배포 제한 시간 (무한 대기 방지)

2. **Readiness Probe의 역할:**
   - 애플리케이션이 진짜 준비됐는지 확인
   - Probe 실패 시 Pod에 트래픽 전달 안 함
   - Rolling Update를 자동 제어하는 안전장치

3. **무중단 배포 조건:**
   - `maxUnavailable = 0` (항상 Ready Pod 존재)
   - 정확한 `readinessProbe` 설정
   - `progressDeadlineSeconds` 적절한 값 (최소: maxReplicas × readiness 시간)

4. **모니터링 포인트:**
   - `kubectl rollout status` (완료 여부)
   - `kubectl get pods` (각 Pod의 Ready 상태)
   - `kubectl describe deployment` (이벤트, 조건)
   - `kubectl get events` (실시간 상태 변화)

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: maxSurge=1, maxUnavailable=1, replicas=10인 경우, 배포 중 최소 몇 개의 Ready Pod이 존재하는가?

<details>
<summary>정답 및 해설</summary>

**답:** 최소 9개

**이유:**
- maxUnavailable=1 → 동시 불가 Pod 수 ≤ 1
- replicas=10 → 항상 10개 관리
- 최소 Ready = 10 - 1 = 9개

하지만 주의: maxSurge=1이므로 동시 최대 Pod = 10 + 1 = 11개
실제 흐름:
1. v1 Pod 10개 (모두 Ready)
2. v2 Pod 1개 추가 (v1 10 + v2 1 = 11)
3. v2 Ready 될 때까지 대기
4. v2 Ready (v1 9 + v2 1 = 10, 모두 Ready)
5. v1 Pod 1개 제거 (v1 8 + v2 1 = 9)
6. ... 계속

따라서 최소 Ready는 9개다.

</details>

### 문제 2: Readiness Probe를 다음과 같이 설정했을 때, Pod이 Ready 상태가 되는 최소 시간은?

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  successThreshold: 2
  failureThreshold: 3
```

<details>
<summary>정답 및 해설</summary>

**답:** 최소 15초

**이유:**
1. Container 시작 → initialDelaySeconds(10초) 대기
2. 첫 번째 체크: 10초 시점 → 실패 (앱이 아직 시작 중)
3. 두 번째 체크: 15초 시점 → 실패
4. 세 번째 체크: 20초 시점 → 성공
5. 네 번째 체크: 25초 시점 → 성공 (successThreshold=2 만족)

따라서 최소 25초... 아니다!

**정정:**
- 10초 대기 후 첫 체크 시작
- initialDelaySeconds=10이므로 10초 경과
- 그 이후 periodSeconds=5마다 체크
- 만약 10초 시점부터 바로 성공하면: 10 + (5 × 1) = 15초

하지만 보통 앱 시작에 시간 걸림. 실제로는 20-30초 예상.

</details>

### 문제 3: Deployment에서 `progressDeadlineSeconds=60`으로 설정했는데, 60초 후에도 배포가 진행 중이라면?

<details>
<summary>정답 및 해설</summary>

**답:** 배포 자동 중단, Deployment 상태 = "Progressing: False"

**실제 동작:**
1. 배포 시작 → 타이머 시작
2. 30초: 진행 상황 체크 → 타이머 리셋
3. 60초: 진행 상황 체크 → 아직 replicas 수만큼 Ready 아님
4. 60초 초과 → Deployment 상태 변경
   - `status.conditions[type=Progressing].status = "False"`
   - `reason = "ProgressDeadlineExceeded"`
5. 새로운 Pod 생성 중단, 기존 Pod은 유지

**결과:**
- 일부 Pod: 새 버전 (아직 배포 완료 아님)
- 일부 Pod: 구 버전
- 혼재 상태로 중단

**복구 방법:**
```bash
# 1. 원인 파악
kubectl describe deployment myapp
# ProgressDeadlineExceeded 확인

# 2. Readiness Probe 실패 이유 조사
kubectl logs pod-name

# 3. 문제 수정 후 재배포
# (자동으로 새로운 rollout 시작)

# 또는 수동 롤백
kubectl rollout undo deployment/myapp
```

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 3 — 레지스트리 관리](../docker-build-optimization/06-registry-management.md)** | **[홈으로 🏠](../README.md)** | **[다음: Blue-Green 배포 — 즉각 롤백의 비용 ➡️](./02-blue-green-deployment.md)**

</div>
