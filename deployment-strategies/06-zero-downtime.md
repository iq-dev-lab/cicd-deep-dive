# Zero-Downtime 배포 조건 — Graceful Shutdown과 preStop

## 🎯 핵심 질문
- Zero-Downtime 배포의 필요조건은 정확히 무엇인가?
- Pod이 종료되기 전 진행 중인 요청을 어떻게 처리하는가?
- `preStop` Hook은 정확히 언제 실행되는가?
- Spring Boot의 `server.shutdown=graceful` 설정이 의미하는 것은?

## 🔍 왜 이 개념이 실무에서 중요한가

Rolling Update, Blue-Green, Canary 배포를 배웠지만, 하나가 빠졌습니다: **애플리케이션 자체가 무중단을 지원해야 한다**는 것입니다.

실무에서 흔한 시나리오:
```
09:00 배포 시작 (Rolling Update)
      새 Pod 생성 시작
      
09:05 구 Pod 종료 (SIGTERM)
      
      하지만 이 시점에 사용자가 요청 중:
      POST /api/payment (결제 API)
      
      응답 시간: 5초 필요
      
결과:
09:05 SIGTERM 받자마자 강제 종료 (killGracePeriodSeconds=30)
      
      요청이 완료되지 않음 → 사용자: 연결 끊김 → 에러!
      
      "배포 중 결제가 실패했습니다!"
```

이것을 해결하려면:
1. **Pod이 새 요청은 받지 않고** (Readiness Probe)
2. **진행 중인 요청은 완료하고** (Graceful Shutdown)
3. **그 다음 종료** (preStop, SIGTERM)

이 3가지 조건을 모두 만족해야 진정한 **Zero-Downtime 배포**입니다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 무신경한 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
      - name: app
        image: payment-api:2.0
        ports:
        - containerPort: 8080
        
        # ❌ Readiness Probe 없음
        # → 준비 안 된 Pod에도 트래픽 전달
        
        # ❌ preStop Hook 없음
        # → 진행 중인 요청 무시하고 즉시 종료
```

**실제 발생하는 일:**
```
Pod 종료 순서 (maxUnavailable=0이어도):
1. Endpoint Controller가 Pod을 endpoints에서 제거 (1-2초)
2. 기존 클라이언트 연결은 여전히 존재
3. Kubelet이 SIGTERM 신호 전송
4. Pod이 즉시 종료됨 (killGracePeriodSeconds=30은 있지만 무시)
5. 진행 중인 요청들: 503 Service Unavailable (연결 끊김)

사용자 경험:
POST /api/payment 진행 중
  → 갑자기 연결 끊김
  → 사용자: "결제가 안 됐는데 돈은 빠져나갔나?"
  → Support ticket 급증
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ Zero-Downtime 배포 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # 항상 최소 1개는 Ready
  
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      # ✅ 종료 대기 시간 (기본 30초)
      terminationGracePeriodSeconds: 45
      
      containers:
      - name: app
        image: payment-api:2.0
        ports:
        - containerPort: 8080
        
        # ✅ 조건 1: Readiness Probe
        # Pod이 정말 준비됐을 때만 트래픽 전달
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        
        # ✅ 조건 2: preStop Hook
        # SIGTERM 보내기 전에 실행
        # 트래픽이 천천히 빠져나갈 시간 확보
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15  # 15초 대기 (트래픽 정리)
        
        # 환경 변수: Spring Boot 설정
        env:
        - name: SERVER_SHUTDOWN
          value: "graceful"  # Graceful shutdown 활성화
        - name: SPRING_LIFECYCLE_TIMEOUT_PER_SHUTDOWN_PHASE
          value: "30s"  # 각 phase마다 최대 30초 대기
```

**Zero-Downtime 배포 프로세스:**
```
Rolling Update 시작 (새 Pod 3개 생성 예정)

단계 1: 새 Pod 생성
  새 Pod: 시작 중
  readinessProbe 실패 (앱이 아직 시작 중)
  → 트래픽 전달 안 함
  
단계 2: 새 Pod Ready
  readinessProbe 성공
  → 이제 트래픽 전달 시작
  → Service endpoints에 추가
  
단계 3: 구 Pod 하나 선택해서 종료
  Endpoint Controller가 endpoints에서 제거 (1-2초)
  
  하지만:
  - 기존 연결은 살아있음 (TCP 연결은 끊어지지 않음)
  - 기존 클라이언트는 계속 요청 가능
  
단계 4: kubelet이 SIGTERM 신호 전송 (30초 전)
  하지만 먼저 preStop Hook 실행
  sleep 15 → 15초 대기
  
  이 15초 동안:
  - 새 요청은 올 수 없음 (endpoints에서 제거됨)
  - 기존 요청은 계속 처리 중
  
단계 5: sleep 15 완료, SIGTERM 신호 전송
  Spring Boot이 graceful shutdown 시작
  
  요청 처리 상태:
  - 진행 중인 요청: 계속 처리 (최대 30초)
  - 새로운 요청: 거절 (503 Service Unavailable)
  
단계 6: 모든 진행 중인 요청 완료
  → Pod 종료
  
결과: Zero-Downtime!
```

## 🔬 내부 동작 원리

### Pod 종료의 4단계

```
Rolling Update 중 Pod 종료:

1단계: Endpoint Controller
   └─ Service endpoints에서 Pod IP 제거 (1-2초 소요)
   └─ 새로운 요청: 이 Pod으로 라우팅되지 않음
   └─ 기존 연결: 여전히 존재함
   
2단계: preStop Hook 실행 (설정된 경우)
   └─ exec: 지정된 명령어 실행 (예: sleep 15)
   └─ 이 시간 동안 Pod은 "Terminating" 상태
   └─ 목적: 트래픽이 다른 Pod으로 리밸런싱될 시간 확보
   
3단계: SIGTERM 신호 전송
   └─ 애플리케이션이 graceful shutdown 시작
   └─ 진행 중인 요청 처리 계속
   └─ 새로운 요청 거절 (이미 endpoints에서 제거됨)
   
4단계: killGracePeriodSeconds 대기 (기본 30초)
   └─ 애플리케이션이 자발적으로 종료될 때까지 대기
   └─ killGracePeriodSeconds 초과 시 강제 종료 (SIGKILL)

전체 소요 시간:
  preStop(15초) + SIGTERM에서 자발적 종료(~5초) = ~20초 이내
```

### readinessProbe와 Pod 종료의 타이밍

```
Pod 생성:
  1. Container 시작
  2. initialDelaySeconds 경과 (10초)
  3. readinessProbe 체크 시작
  4. 성공 → Pod을 Ready로 판정
  5. Service endpoints에 Pod IP 추가
  6. 트래픽 전달 시작

Pod 종료:
  1. Deployment이 Pod 삭제 결정 (Rolling Update)
  2. Endpoint Controller가 endpoints에서 제거
     (readinessProbe와는 별개, 즉시 처리)
  3. preStop Hook 실행
  4. SIGTERM 신호
  5. graceful shutdown
  6. Pod 종료

중요: readinessProbe는 Pod 생성 시에만 체크, 
     종료 시에는 관관없음!
     
     따라서 SIGTERM → graceful shutdown이 중요
```

### Spring Boot Graceful Shutdown

```yaml
server:
  shutdown: graceful  # Enable graceful shutdown
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 각 phase마다 최대 30초

# 동작 원리:
#
# 1. SIGTERM 신호 수신
# 2. Spring Boot이 다음 단계 실행:
#
#    Phase 1: Server stop (5초 이내)
#    - 새로운 HTTP 요청 거절
#    - 기존 요청은 계속 처리
#
#    Phase 2: ApplicationContext shutdown (timeout-per-shutdown-phase)
#    - Bean lifecycle 정리
#    - Database connection 정리
#    - graceful하게 자원 해제
#
# 3. 모든 단계 완료 후 프로세스 종료
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 실험 1: Zero-Downtime 배포 설정

```bash
# 1. Zero-Downtime 배포 Deployment 생성
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 항상 최소 1개 Ready
  
  selector:
    matchLabels:
      app: zero-downtime-app
  
  template:
    metadata:
      labels:
        app: zero-downtime-app
    spec:
      terminationGracePeriodSeconds: 45  # 45초 종료 대기
      
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        
        # preStop Hook
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15  # 15초 대기
---
apiVersion: v1
kind: Service
metadata:
  name: zero-downtime-app
spec:
  selector:
    app: zero-downtime-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# 2. 모든 Pod이 Ready될 때까지 대기
kubectl wait --for=condition=ready pod -l app=zero-downtime-app --timeout=60s

# 3. Service endpoints 확인 (모든 Pod이 등록됨)
kubectl get endpoints zero-downtime-app

# 출력:
# NAME                 ENDPOINTS                         AGE
# zero-downtime-app    10.244.0.10:80,10.244.0.11:80,10.244.0.12:80   1m
```

### 실험 2: Pod 종료 과정 관찰

```bash
# 1. 무한 요청 발생 (배경)
kubectl run -it --rm curl-client --image=curlimages/curl --restart=Never -- \
  sh -c 'while true; do curl -v http://zero-downtime-app; sleep 1; done'

# 다른 터미널에서...

# 2. Pod 종료 프로세스 모니터링
watch -n 1 kubectl describe pod pod-name

# 3. 배포 시작 (이미지 업데이트)
kubectl set image deployment/zero-downtime-app app=nginx:latest

# 4. 롤아웃 진행 상황 관찰
kubectl rollout status deployment/zero-downtime-app --watch

# 5. 이벤트 확인 (시간 기록)
kubectl get events --sort-by='.lastTimestamp' | grep zero-downtime-app

# 출력:
# 09:00:00 Pod spec.containers[0].image from nginx:1.19 to nginx:latest
# 09:00:05 Pod is being initialized
# 09:00:10 Pod's readiness probe failed
# 09:00:15 Pod is now ready
# 09:00:20 Pod is being terminated
# 09:00:20 PreStop hook: sleep 15
# 09:00:35 Pod terminated
```

### 실험 3: preStop 없을 때 vs 있을 때 비교

```bash
# 설정 A: preStop 없음
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: without-prestop
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
  selector:
    matchLabels:
      app: without-prestop
  template:
    metadata:
      labels:
        app: without-prestop
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: nginx:1.19
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# 설정 B: preStop 있음
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: with-prestop
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
  selector:
    matchLabels:
      app: with-prestop
  template:
    metadata:
      labels:
        app: with-prestop
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: nginx:1.19
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15
EOF

# 두 Deployment 동시에 배포
time kubectl set image deployment/without-prestop app=nginx:1.20 &
time kubectl set image deployment/with-prestop app=nginx:1.20 &

wait

# 결과:
# without-prestop: 더 빠른 종료 (하지만 요청 손실 위험)
# with-prestop: 약간 더 느린 종료 (하지만 요청 안전)
```

### 실험 4: 장시간 실행 요청 처리

```bash
# 1. 장시간 요청을 처리하는 애플리케이션
cat <<'EOF' > app.py
from flask import Flask
import time
import sys

app = Flask(__name__)

@app.route('/health/ready')
def health():
    return 'OK', 200

@app.route('/api/long-task')
def long_task():
    """10초 걸리는 작업"""
    time.sleep(10)
    return 'Task completed', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 2. Docker 이미지 빌드
docker build -t long-task-app:1.0 .

# 3. Kubernetes Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: long-task-app
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
  selector:
    matchLabels:
      app: long-task-app
  template:
    metadata:
      labels:
        app: long-task-app
    spec:
      terminationGracePeriodSeconds: 20  # 20초 (10초 요청 + 10초 여유)
      containers:
      - name: app
        image: long-task-app:1.0
        ports:
        - containerPort: 5000
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 5  # endpoints 제거 후 5초 대기
---
apiVersion: v1
kind: Service
metadata:
  name: long-task-app
spec:
  selector:
    app: long-task-app
  ports:
  - port: 5000
    targetPort: 5000
EOF

# 4. 장시간 요청 시작
POD=$(kubectl get pod -l app=long-task-app -o name | head -1)
kubectl exec -it $POD -- curl http://localhost:5000/api/long-task &

# 5. 배포 시작 (요청 진행 중)
sleep 2
kubectl set image deployment/long-task-app app=long-task-app:1.1

# 6. 결과 확인
# - terminationGracePeriodSeconds=20이므로
# - 10초 요청 처리가 완료되어야 Pod 종료
# - preStop(5초) + 요청(10초) = 15초 < 20초 → 성공!

# 로그 확인
kubectl logs $POD
# Task completed (요청이 완료됨!)
```

### 실험 5: Spring Boot Graceful Shutdown

```bash
# 1. Spring Boot 애플리케이션 (application.yml)
cat <<'EOF' > application.yml
server:
  shutdown: graceful
  servlet:
    context-path: /api

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 각 단계마다 최대 30초

logging:
  level:
    org.springframework.boot: INFO
EOF

# 2. Deployment 설정
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      terminationGracePeriodSeconds: 60  # Spring Boot graceful 대기 시간
      containers:
      - name: app
        image: spring-boot-app:2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /api/health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 10  # endpoints 제거 후 10초 대기
EOF

# 3. Pod 로그로 graceful shutdown 확인
kubectl logs -f pod-name | grep -i shutdown

# 출력 예:
# Application run successful
# [main] Shutting down ...
# [main] Wait for ongoing requests to complete
# [main] Completed shutdown in 5 seconds
# [main] Application stopped
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 설정 | Readiness Probe | preStop | graceful shutdown | 배포 중 에러율 | 배포 시간 |
|------|---|---|---|---|---|
| 기본 설정 | ❌ | ❌ | ❌ | 높음 (1-5%) | 25초 |
| Readiness만 | ✅ | ❌ | ❌ | 중간 (0.1-1%) | 30초 |
| + preStop | ✅ | ✅ | ❌ | 낮음 (<0.01%) | 40초 |
| + graceful | ✅ | ✅ | ✅ | 거의 0% | 45초 |

### 비용 분석

```
배포 중 에러로 인한 손실 계산:

기본 설정 (에러율 1%):
  일일 요청: 100,000개
  배포: 4회 (각 30초)
  배포 중 요청: 100,000 × (4 × 30 / 86400) ≈ 139개
  에러: 139 × 1% ≈ 1.4개 요청 실패
  
  월간 손실: 1.4 × 20 = 28개 실패 요청
  평균 결제: $100
  월간 손실액: 28 × $100 = $2,800

Zero-Downtime 설정 (에러율 0%):
  배포 시간 추가: 15초
  월간 추가 비용: 무시할 수 있는 수준 (<$1)

결론: Zero-Downtime 설정의 ROI가 매우 높음
```

## ⚖️ 트레이드오프

### 배포 시간 vs 안정성

```
배포 시간을 줄이려면:
  ❌ preStop 제거 → 더 빠른 종료
  ❌ terminationGracePeriodSeconds 감소
  ❌ graceful shutdown 비활성화
  
  결과: 요청 손실 위험 증가

안정성을 위해:
  ✅ preStop 15초 추가
  ✅ graceful shutdown 활성화
  ✅ terminationGracePeriodSeconds 60초
  
  결과: 배포 시간 +15-20초 (무시할 수 있는 수준)
```

### 요청 처리 시간 vs graceful period

```
애플리케이션의 최장 요청 시간: 5초

설정 선택:
  ❌ terminationGracePeriodSeconds: 10
     5초 요청 진행 중 종료 신호 받으면 5초만 대기
     그 사이 완료 못할 수 있음
  
  ✅ terminationGracePeriodSeconds: 30
     충분한 여유 시간 확보
     모든 요청 완료 후 우아하게 종료

권장: 최장 요청 시간 × 2 + 여유 10초
```

## 📌 핵심 정리

1. **Zero-Downtime의 4가지 조건:**
   - ✅ Readiness Probe (준비된 Pod만 트래픽)
   - ✅ maxUnavailable=0 (항상 최소 1개 Ready)
   - ✅ preStop Hook (endpoints 제거 후 요청 정리)
   - ✅ Graceful Shutdown (진행 중인 요청 완료)

2. **Pod 종료의 4단계:**
   ```
   Endpoint 제거 → preStop → SIGTERM → graceful shutdown
   (1-2초)   (15초)    (~5초)    (10초)
   ```

3. **각 기술의 역할:**
   ```
   Readiness Probe: 새 요청을 줄임 (준비 안 된 Pod 제외)
   maxUnavailable=0: 항상 대체 Pod 유지
   preStop: 트래픽이 다른 Pod으로 옮겨질 시간 확보
   Graceful Shutdown: 진행 중인 요청 완료 후 종료
   ```

4. **Spring Boot 설정:**
   ```yaml
   server.shutdown: graceful
   spring.lifecycle.timeout-per-shutdown-phase: 30s
   ```

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: Pod 종료 중 새로운 요청이 들어왔다면?

```
Pod 상태: Terminating
Endpoint: 이미 제거됨
새로운 HTTP 요청 도착

어디로 라우팅되는가?
```

<details>
<summary>정답 및 해설</summary>

**답:** 다른 Pod으로 라우팅됨 (Terminating Pod으로는 가지 않음)

**이유:**
```
1. Endpoint Controller가 endpoints에서 Pod IP 제거
2. Service는 endpoints에 등록된 Pod들로만 트래픽 라우팅
3. Terminating Pod은 endpoints에 없음
4. 새 요청은 다른 Ready Pod으로만 간다

타이밍:
  Endpoint 제거 (1-2초) → preStop(15초) → SIGTERM
  
  이 동안 새 요청은 모두 다른 Pod으로 라우팅됨
  Terminating Pod은 기존 연결만 처리 (요청 거절 안 함)
```

**예시:**
```
Pod A, B, C 중 A가 종료 중

요청 1: A의 기존 연결 → 처리 계속 (graceful shutdown)
요청 2: 새로운 연결 → B 또는 C로 라우팅 (A 제외)
요청 3: 또 다른 새 연결 → B 또는 C로 라우팅
```

</details>

### 문제 2: preStop(15초) 중에 Pod이 강제 종료되면?

```
Pod 상태: preStop Hook 실행 중 (sleep 15)

갑자기 누군가가:
  kubectl delete pod pod-name --grace-period=5
```

<details>
<summary>정답 및 해설</summary>

**답:** 5초 후 강제 종료 (SIGKILL)

**프로세스:**
```
1. kubectl delete pod pod-name --grace-period=5 실행
2. Kubernetes가 Pod에 SIGTERM 신호 (보통 수신)
3. preStop Hook 실행 중... (5초 남음)
4. 5초 후 강제 종료 (SIGKILL)
5. preStop Hook이 완료되지 않음

결과:
  - sleep 15가 완료되지 않음 (5초만 실행)
  - SIGTERM도 미처 처리 안 됨
  - 진행 중인 요청들이 갑자기 끊김
```

**해결책:**
```
--grace-period는 --force보다 높게 설정
또는 Deployment의 terminationGracePeriodSeconds와 일치

kubectl delete pod pod-name --grace-period=45
```

**실제 값:**
```
terminationGracePeriodSeconds: 45
  = preStop(15) + SIGTERM 처리(~5) + graceful(~20) + 여유(5)

이를 초과하면 강제 종료 발생
```

</details>

### 문제 3: Readiness Probe가 간헐적으로 실패하면?

```
readinessProbe:
  periodSeconds: 5
  failureThreshold: 3

의미: 3회 연속 실패하면 Pod을 NotReady로 판정

배포 중:
  Pod A: 진행 중인 요청 처리 중
  
  갑자기 Readiness Probe 실패 (하드웨어 일시적 문제)
  → 2회 더 실패 → NotReady로 판정
  → Endpoint 제거
  → 트래픽이 다른 Pod으로 이동

이게 정상인가?
```

<details>
<summary>정답 및 해설</summary>

**상황 분석:**
```
readinessProbe 실패의 원인:
1. Pod 내부 문제 (메모리 부족, DB 연결 끊김)
2. 일시적 네트워크 지연
3. 문제 있는 엔드포인트 설정

failureThreshold: 3 설정의 의미:
  - 1회 실패는 무시 (일시적 오류)
  - 2회 실패도 무시
  - 3회 연속 실패 → NotReady로 판정
```

**배포 중 동작:**
```
배포 진행 중이고 Pod A가 NotReady가 되면:

시나리오 1: Pod 내부 진짜 문제
  → 빨리 제거되는 게 맞음 (다른 Pod로 트래픽 전환)
  → Zero-Downtime 달성

시나리오 2: 일시적 문제 (또 다시 Ready됨)
  → 진행 중인 요청은 계속 처리 (endpoints 제거되어도)
  → 몇 초 후 다시 Ready → 새 요청 수신 재개

문제 가능성: 3회 연속 실패를 피했는데
배포 중에는 Pod이 스트레스 받을 수 있음
→ failureThreshold를 3이 아닌 5-10으로 증가 고려
```

**권장 설정:**
```yaml
readinessProbe:
  periodSeconds: 5
  failureThreshold: 5  # 25초 수용 (일시적 오류 허용)
  successThreshold: 1  # 1회 성공으로 Ready 판정
```

</details>

---

<div align="center">

**[⬅️ 이전: 롤백 전략 — 언제 되돌릴 수 없는가](./05-rollback-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 배포 파이프라인 전체 흐름 ➡️](./07-deployment-pipeline-flow.md)**

</div>
