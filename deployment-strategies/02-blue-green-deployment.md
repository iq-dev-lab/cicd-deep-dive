# Blue-Green 배포 — 즉각 롤백의 비용

## 🎯 핵심 질문
- Blue-Green 배포의 핵심 원리는 무엇인가?
- Kubernetes Service selector를 변경하는 것만으로 트래픽을 전환할 수 있는가?
- Blue-Green 배포의 비용은 정확히 얼마나 드는가?
- DB 스키마 변경이 있을 때 왜 Blue-Green이 복잡해지는가?

## 🔍 왜 이 개념이 실무에서 중요한가

많은 팀이 "무중단 배포"를 원합니다. 그런데:
- **Rolling Update는 배포 중 혼재 상태 (구 버전 + 새 버전)** → 복잡한 버전 호환성 관리 필요
- **Canary 배포는 단계적이라 시간 걸림** → 긴급 hotfix 배포에 부적합
- **Blue-Green은 즉각 전환 + 즉각 롤백** → 높은 신뢰도, 대신 비용 높음

Blue-Green의 메커니즘을 정확히 이해하면:
1. **언제 Blue-Green을 써야 하는지** 판단 가능
2. **비용 대비 효과를 계산** 가능
3. **DB 마이그레이션 전략**을 세울 수 있음

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 이해: Deployment 2개가 있는 걸로 끝
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: app
        image: myapp:1.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 0  # 처음엔 0개
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
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
    # ❌ 문제: version label이 없다!
    # v1에도 매치되고 v2에도 매치됨 (혼재)
  ports:
  - port: 80
    targetPort: 8080
```

**이 방식의 문제점:**
1. Service selector가 모든 Pod과 매치 → 혼재 상태
2. 배포 중 어느 version으로 트래픽이 가는지 불명확
3. 롤백 시 수동으로 replicas 수를 변경해야 함
4. "Blue는 몇 개 유지? Green은 몇 개?"의 의사결정 없음

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 구조: Blue/Green을 명시적으로 관리
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: blue
  template:
    metadata:
      labels:
        app: myapp
        slot: blue
    spec:
      containers:
      - name: app
        image: myapp:1.0
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3  # 새 버전도 동일 replicas
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
    spec:
      containers:
      - name: app
        image: myapp:2.0
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    slot: blue  # ✅ 현재는 Blue로 모든 트래픽 라우팅
  ports:
  - port: 80
    targetPort: 8080
```

**배포 프로세스:**
```
1단계: Blue 환경 현재 운영 중 (트래픽 수신)
       Green 환경 대기 중 (트래픽 미수신, replicas=3)

2단계: Green 환경에서 모든 Pod이 Ready 될 때까지 테스트
       kubectl get pods -l slot=green
       → 3개 모두 Running & Ready

3단계: Service selector 변경 (Blue → Green)
       kubectl patch service myapp -p '{"spec":{"selector":{"slot":"green"}}}'

4단계: 즉시 트래픽 전환 (Rolling Update 없음!)
       
5단계: 모니터링 (에러율, 응답시간 등)
       - 정상: 배포 완료
       - 문제: 즉시 selector 변경 (Green → Blue로 롤백)
       
6단계: 안정화 확인 후 구 Blue Deployment 종료
       kubectl scale deployment myapp-blue --replicas=0
```

## 🔬 내부 동작 원리

### Service Selector 변경의 즉각 효과

```
Service 객체:
  selector:
    app: myapp
    slot: blue

Kubernetes의 동작:
1. Service의 selector와 매치되는 Pod 검색
2. 매치되는 Pod의 IP를 모두 Endpoints 객체에 추가
3. 클라이언트가 Service IP로 접속하면
   → Endpoints에 등록된 IP 중 하나로 라우팅 (round-robin)

Service selector 변경 시점:
  kubectl patch service myapp -p '{"spec":{"selector":{"slot":"green"}}}'
  
  실행 즉시:
  - Blue Pod들은 Endpoints에서 제거
  - Green Pod들은 Endpoints에 추가
  - 기존 TCP 연결: 즉시 끊김 (또는 timeout 대기)
  - 새 연결: Green으로만 라우팅

결과: ⚡ 밀리초 단위의 트래픽 전환
```

### Blue-Green과 DB 마이그레이션의 복잡성

```
시나리오 1: 애플리케이션만 변경 (DB 스키마 변경 없음)
  Blue (v1) ──→ DB (스키마 v1)
  Green (v2) ──→ DB (스키마 v1)  # 같은 스키마 사용 가능!
  
  Blue-Green 배포 가능:
  1. Green v2를 DB v1 스키마로 테스트
  2. 문제 없으면 selector 변경 → Green으로 트래픽 전환
  3. 구 Blue는 대기 (즉시 롤백 가능)

시나리오 2: DB 스키마도 변경 필요 (예: 새 column 추가)
  Blue (v1) ──→ DB (스키마 v1)
  Green (v2) ──→ DB (스키마 v2)  # 다른 스키마 필요!
  
  ❌ 불가능: DB 스키마는 공유 리소스
     v1과 v2가 동시에 같은 DB에 접근할 수 없음
     
해결책:
  1. DB 스키마 마이그레이션 (v1 → v2)
  2. v2 코드: 이전 column도 읽고 새 column도 읽음 (호환성)
  3. Green 배포 + selector 변경
  4. Blue 운영 (필요시 즉시 롤백 가능)
  5. Blue 종료 후 이전 column 제거
```

### 비용 계산: 2배 인프라 유지

```
Rolling Update (기본):
  배포 중: 최대 Pod 수 = replicas + maxSurge
  예: replicas=3, maxSurge=1 → 최대 4개
  
  배포 후: Pod 수 = replicas = 3개
  
  추가 비용: 배포 시간만 발생

Blue-Green 배포:
  배포 전: Blue 3개
  배포 중: Blue 3개 + Green 3개 = 6개 (100% 추가)
  
  트래픽 전환 후: Green 3개 (Blue 대기)
  
  추가 비용: 배포 완료 후에도 지속
           (즉시 롤백을 위해 Blue 유지)
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 실험 1: Blue-Green 배포 기본 구조

```bash
# 1. Blue Deployment 생성 (현재 운영 버전)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: blue
  template:
    metadata:
      labels:
        app: myapp
        slot: blue
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

# 2. Green Deployment 생성 (새 버전, 대기 중)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 0  # 아직 배포 안 함
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
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
          periodSeconds: 2
EOF

# 3. Service 생성 (Blue로 라우팅)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    slot: blue
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# 4. 초기 상태 확인
kubectl get pods -l app=myapp --show-labels
# 출력:
# NAME                    READY   STATUS    RESTARTS   LABELS
# myapp-blue-abc123       1/1     Running   0          app=myapp,slot=blue
# myapp-blue-def456       1/1     Running   0          app=myapp,slot=blue
# myapp-blue-ghi789       1/1     Running   0          app=myapp,slot=blue

# 5. Service endpoints 확인
kubectl get endpoints myapp
# 출력:
# NAME    ENDPOINTS                                   AGE
# myapp   10.244.0.10:80,10.244.0.11:80,10.244.0.12:80   1m
# (모두 Blue Pod)
```

### 실험 2: Green 배포 및 전환

```bash
# 1. Green Deployment 스케일업 (새 버전 배포)
kubectl scale deployment myapp-green --replicas=3

# 2. Green Pod이 모두 Ready될 때까지 대기
kubectl rollout status deployment myapp-green

# 3. Green Pod 확인
kubectl get pods -l slot=green
# 출력:
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-green-xyz123       1/1     Running   0          10s
# myapp-green-xyz456       1/1     Running   0          10s
# myapp-green-xyz789       1/1     Running   0          10s

# 4. 테스트 트래픽 전송 (Green으로 수동 라우팅)
# Green Pod 중 하나의 IP 직접 접근
kubectl exec -it deployment/myapp-blue -- \
  curl http://10.244.0.20:80   # Green Pod IP

# 5. 현재 상태: 두 환경 모두 Ready
kubectl get pods -l app=myapp
# 출력:
# Blue:  3개 (트래픽 수신)
# Green: 3개 (트래픽 미수신)
# 총 6개 Pod 실행 중

# 6. ⚡️ Service selector 변경 (즉시 트래픽 전환)
kubectl patch service myapp -p '{"spec":{"selector":{"slot":"green"}}}'

# 7. 전환 즉시 확인
kubectl get endpoints myapp
# 출력:
# NAME    ENDPOINTS                                   AGE
# myapp   10.244.0.20:80,10.244.0.21:80,10.244.0.22:80   2m
# (모두 Green Pod으로 변경됨!)

# 8. Blue는 여전히 Running 상태 (롤백용)
kubectl get pods -l slot=blue
# 출력:
# 3개 모두 Running, 하지만 Service의 Endpoints에는 없음
```

### 실험 3: 즉시 롤백

```bash
# 모니터링 중 문제 발견! → 즉시 롤백

# 1. Service selector를 다시 Blue로 변경
kubectl patch service myapp -p '{"spec":{"selector":{"slot":"blue"}}}'

# 2. 트래픽이 즉시 Blue로 복구됨 (밀리초 단위!)
kubectl get endpoints myapp
# 출력:
# NAME    ENDPOINTS                                   AGE
# myapp   10.244.0.10:80,10.244.0.11:80,10.244.0.12:80   3m
# (다시 Blue로 복구)

# 3. Green은 계속 실행 중 (로그 분석용)
kubectl logs -l slot=green -c app --tail=50

# 4. 원인 파악 후, Green 종료
kubectl scale deployment myapp-green --replicas=0

# 5. 롤백 완료
```

### 실험 4: 자동화된 Blue-Green 배포 스크립트

```bash
#!/bin/bash
# blue-green-deploy.sh

DEPLOYMENT_BLUE="myapp-blue"
DEPLOYMENT_GREEN="myapp-green"
SERVICE="myapp"
NEW_IMAGE="$1"  # 예: nginx:1.21

if [ -z "$NEW_IMAGE" ]; then
  echo "Usage: $0 <new-image>"
  exit 1
fi

echo "=== Blue-Green Deployment Started ==="

# 1. 현재 상태 확인
echo "1. Checking current state..."
CURRENT_SLOT=$(kubectl get service $SERVICE -o jsonpath='{.spec.selector.slot}')
echo "   Current traffic: $CURRENT_SLOT"

# Blue가 현재 트래픽을 받으면 Green에 배포, 반대면 Blue에 배포
if [ "$CURRENT_SLOT" == "blue" ]; then
  TARGET_SLOT="green"
  TARGET_DEPLOYMENT=$DEPLOYMENT_GREEN
  STANDBY_DEPLOYMENT=$DEPLOYMENT_BLUE
else
  TARGET_SLOT="blue"
  TARGET_DEPLOYMENT=$DEPLOYMENT_BLUE
  STANDBY_DEPLOYMENT=$DEPLOYMENT_GREEN
fi

echo "   Deploying to: $TARGET_SLOT"

# 2. 대기 환경에 새 이미지로 배포
echo "2. Updating $TARGET_DEPLOYMENT with image: $NEW_IMAGE"
kubectl set image deployment/$TARGET_DEPLOYMENT \
  app=$NEW_IMAGE

# 3. 모든 Pod이 Ready될 때까지 대기
echo "3. Waiting for deployment to complete..."
kubectl rollout status deployment/$TARGET_DEPLOYMENT --timeout=5m

if [ $? -ne 0 ]; then
  echo "❌ Deployment failed!"
  exit 1
fi

# 4. 수동 테스트 (선택사항)
echo "4. Manual testing (Ctrl+C to skip)..."
echo "   Try: kubectl exec -it deployment/$CURRENT_SLOT -- curl http://127.0.0.1:8080"
sleep 10

# 5. Service selector 변경 (트래픽 전환)
echo "5. Switching traffic to $TARGET_SLOT"
kubectl patch service $SERVICE -p "{\"spec\":{\"selector\":{\"slot\":\"$TARGET_SLOT\"}}}"

# 6. 헬스체크 (5회, 각 2초)
echo "6. Health checks:"
for i in {1..5}; do
  READY=$(kubectl get pods -l slot=$TARGET_SLOT -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -o True | wc -l)
  TOTAL=$(kubectl get pods -l slot=$TARGET_SLOT --no-headers | wc -l)
  echo "   Check $i: $READY/$TOTAL pods ready"
  sleep 2
done

echo "=== Deployment Complete ==="
echo "✅ Traffic switched to: $TARGET_SLOT"
echo "⏮️  Rollback available (current standby: $CURRENT_SLOT)"
```

```bash
# 사용 방법
./blue-green-deploy.sh nginx:1.21

# 롤백 (기본 명령으로도 가능)
kubectl patch service myapp -p '{"spec":{"selector":{"slot":"blue"}}}'
```

### 실험 5: 부하 테스트로 성능 비교

```bash
# Blue-Green 배포 전후 응답시간 비교

# 1. Load 발생 (기존 Blue에서 처리 중)
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://myapp > /dev/null; done'

# 2. 모니터링 (다른 터미널)
while true; do
  echo "=== $(date +%H:%M:%S) ==="
  kubectl top pods -l app=myapp
  sleep 2
done

# 3. 배포 시작 (다른 터미널)
# 현재 트래픽은 계속 Blue로 흘러감 (Service selector는 아직 Blue)
kubectl set image deployment/myapp-green app=nginx:1.21 &

# 4. Green Ready 후 selector 변경
# → 트래픽 즉시 전환 (지연 거의 없음)

# 5. Blue에서 Green으로 전환 시점 확인
# Green Pod의 CPU/Memory 급증 (트래픽 수신)
# Blue Pod의 CPU/Memory 급감 (트래픽 중단)
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 항목 | Rolling Update | Blue-Green | Canary |
|------|---|---|---|
| **배포 시간** | 25-40초 | 30-60초 (테스트 포함) | 2-5분 (단계별) |
| **트래픽 전환** | 순차적 (Pod 교체) | 즉시 (selector 변경) | 단계적 (5% → 100%) |
| **롤백 시간** | 25-40초 (ReplicaSet 재활성) | <1초 (selector 변경) | 수동 개입 |
| **메모리 피크** | +20% (maxSurge=1) | +100% (Blue+Green 동시) | +10-20% |
| **동시 버전** | 2개 (구 + 신) | 2개 (구 + 신, 완전 분리) | 2개 (버전 섞임) |
| **DB 호환성** | 필수 | 필수 (스키마 변경 시 복잡) | 필수 |

### 실제 비용 계산

```
월 기준 (30일 × 24시간)

1개 Pod 월 비용: $50 (t3.medium)
3개 Pod 기본 비용: $150

Rolling Update:
- 배포 시간: 30초
- 추가 Pod: 1개 × (30초 / 86400초/일 × 30일)
- 월 추가 비용: $50 × (1/2880) × 30 = $0.52
- 일일 배포 10회: $5.2/월

Blue-Green (배포 완료 후 1시간 대기):
- 추가 유지 시간: 1시간
- 추가 Pod: 3개 × (1시간 / 24시간 × 30일)
- 월 추가 비용: $150 × (30/24) = $187.5
- 일일 배포 1회: $187.5/월

결론:
- Rolling Update: 저비용, 느린 롤백
- Blue-Green: 고비용, 빠른 롤백
```

## ⚖️ 트레이드오프

### 비용 vs 롤백 속도

```
Rolling Update:
  ✅ 비용 저렴 (추가 비용 무시할 수준)
  ✅ 메모리 효율적
  ❌ 롤백 느림 (25-40초, ReplicaSet 재활성 필요)
  ❌ 배포 중 두 버전 혼재

Blue-Green:
  ❌ 비용 높음 (2배 인프라 비용)
  ❌ 메모리 사용량 2배
  ✅ 롤백 빠름 (<1초)
  ✅ 배포 중에도 순환 무중단
```

### DB 마이그레이션의 복잡성

```
시나리오 1: 코드만 변경
  Rolling Update: 간단 (기존 DB 스키마 사용)
  Blue-Green: 간단 (기존 DB 스키마 사용)
  
시나리오 2: DB 스키마 추가 (새 column)
  Rolling Update: 
    1. 먼저 DB 스키마 추가
    2. 구 코드도 새 column 읽게 수정
    3. Rolling Update 실행
    → 배포 중에도 양쪽 모두 새 스키마 접근
    
  Blue-Green:
    1. DB 스키마 추가
    2. Blue는 구 column 읽음
    3. Green은 새 column 읽음 (호환성 필요)
    4. selector 변경
    5. 안정화 후 구 column 제거
    → 배포 중 두 버전이 다른 스키마 접근 위험

결론: 스키마 변경이 있으면 Rolling Update가 더 간단함
```

## 📌 핵심 정리

1. **Blue-Green의 핵심:**
   - 2개의 동일한 환경 유지 (Blue=현재, Green=새 버전)
   - Service selector 변경으로 트래픽 즉시 전환
   - 기존 환경 유지 → 즉시 롤백 가능

2. **배포 프로세스:**
   ```
   1. Green 배포 (Blue는 현재 트래픽 수신)
   2. Green 모든 Pod Ready 대기
   3. selector: blue → green 변경
   4. 트래픽 즉시 Green으로 전환
   5. 모니터링 (에러율, 응답시간)
   6. 문제 시 즉시 selector 변경 (green → blue)
   ```

3. **언제 Blue-Green을 쓸 것인가:**
   - ✅ 긴급 배포 (빠른 롤백 필수)
   - ✅ 검증되지 않은 버전 (테스트 후 배포)
   - ✅ 메모리 여유 있을 때 (2배 비용 감당 가능)
   - ❌ 빈번한 배포 (비용 높음)
   - ❌ 메모리 부족 환경

4. **DB 마이그레이션:**
   - 스키마 변경 필요 시 매우 복잡
   - 호환성 계층 필수 (두 스키마 모두 지원)
   - Rolling Update가 더 안전한 경우 多

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: Blue-Green 배포 중 다음 상황에서 문제는?

```yaml
Blue Deployment (v1): replicas=3
Green Deployment (v2): replicas=3
Service selector: 현재 blue

배포 시작:
1. Green의 새 Pod 3개가 Ready됨
2. Service selector를 blue → green으로 변경
3. 트래픽이 Green으로 전환됨
4. 2분 후 에러 발생
5. Green Pods의 에러율이 10%
```

이 상황에서:
- (1) Blue를 이미 종료했다면?
- (2) Blue는 여전히 running 상태라면?

<details>
<summary>정답 및 해설</summary>

**(1) Blue를 이미 종료했다면?**
```
상황: Green에서 에러 발생, Blue는 이미 0개

문제:
- 즉시 롤백 불가능
- 다시 Blue를 배포해야 함 (2-3분)
- 그 사이 사용자는 계속 에러 수신

해결책: 원본 Blue Pod을 즉시 재시작
  kubectl scale deployment myapp-blue --replicas=3
  → 기존 image:v1 이미지로 3개 Pod 생성 (약 20초)
  
  그 사이 에러 지속. Blue가 Ready될 때까지:
  selector: green → blue 변경까지 총 20-30초 손상 시간

교훈: Blue를 즉시 종료하지 말 것! 
      안정화 확인 후 (최소 1시간) 종료
```

**(2) Blue는 여전히 running 상태라면?**
```
상황: Green에서 에러 발생, Blue는 3개 Pod running & Ready

완벽한 해결:
1. Service selector 변경 (green → blue)
2. 트래픽이 즉시 Blue로 복구됨
3. 에러 즉시 중단 (밀리초 단위)
4. 사용자 영향 최소화

총 RTO (Recovery Time Objective): <1초
총 데이터 손실: 거의 없음 (기존 연결만 끊김)

교훈: Blue-Green의 최대 장점!
```

</details>

### 문제 2: 다음 시나리오에서 Blue-Green 배포가 가능한가?

```
현재: DB 스키마 v1 (users 테이블: id, name, email)

배포 계획:
- 새 feature: phone_number column 추가
- Blue (v1 코드): users.id, name, email만 읽음
- Green (v2 코드): users.id, name, email, phone_number 읽음

배포 순서:
1. 기존 Blue는 계속 운영 (v1 코드)
2. Green에 v2 코드 배포
3. DB 스키마에 phone_number 추가
4. Service selector: blue → green
5. Green 운영 시작
```

이 방식이 작동하는가? 문제점은?

<details>
<summary>정답 및 해설</summary>

**작동 여부: 기술적으로 작동하지만 위험**

**시나리오 분석:**
```
배포 단계별 상황:

1단계: Blue 운영, Green v2 배포 중 (아직 DB 스키마 미변경)
   → Green v2가 phone_number 읽으려 하면?
   → 컬럼 없음 → NULL 또는 에러
   → Green Pod: CrashLoopBackOff

2단계: DB 스키마 수정 (phone_number 추가)
   → 그제야 Green v2가 정상 작동

3단계: selector 변경 (blue → green)
   → Green이 phone_number를 정상 읽음
   
롤백 시나리오 (Green에서 에러 발생):
   selector: green → blue로 변경
   → Blue (v1)는 phone_number column을 무시
   → 정상 작동
   → 하지만 새 phone_number 데이터는 DB에만 저장된 상태
```

**위험한 부분:**
1. Green 배포 중 DB 스키마 없음 → Pod 시작 실패
2. 배포 순서: Green 먼저 배포? 스키마 먼저 추가?
3. 동시성 문제: Blue와 Green이 동시에 DB 접근 중에 스키마 변경?

**안전한 순서:**
```
1. DB 마이그레이션: phone_number column 추가 (nullable)
   → Blue (v1)는 무시하고 진행
   
2. Blue 코드 업데이트: phone_number도 읽도록 수정
   → Blue를 먼저 v1.5 버전으로 배포
   
3. Green 배포: v1.5 코드로 Green 배포
   → phone_number 읽기 가능
   
4. selector 변경: blue → green

아니면 Rolling Update를 쓸 것:
1. DB 마이그레이션
2. 코드 v2 배포 (Rolling Update)
   → 모든 Pod이 새 코드로 자동 교체
   → 혼재 기간 최소화
```

**결론: 스키마 변경이 있으면 Blue-Green이 복잡함**

</details>

### 문제 3: Blue-Green 배포 중 미리 계획해야 할 항목은?

<details>
<summary>정답 및 해설</summary>

**필수 사전 계획:**

1. **리소스 검증**
   ```bash
   # Blue + Green 동시 실행 시 필요 리소스
   필요 메모리 = replicas × 2 × memory_per_pod
   필요 CPU = replicas × 2 × cpu_per_pod
   
   현재 사용 가능한 리소스 < 필요 리소스?
   → Blue-Green 불가능 (이전 버전까지 대기해야 함)
   ```

2. **헬스체크 엔드포인트**
   ```yaml
   Green 배포 후 판단 기준:
   - /health/ready: 모든 Pod이 Ready 상태인가?
   - /metrics: 에러율, 응답시간은 정상인가?
   - /logs: 경고/에러 메시지는 없는가?
   ```

3. **롤백 계획**
   ```
   - Green에서 에러 발생 시:
     selector: green → blue (밀리초)
   - 그 후 원인 분석:
     kubectl logs -l slot=green
   - 문제 해결 후 다시 배포:
     Green Deployment 재배포
   ```

4. **데이터 일관성**
   ```
   Blue와 Green이 동시에 DB 접근할 때:
   - 트랜잭션 격리 수준은?
   - race condition 가능성은?
   - 각 버전이 같은 데이터 형식을 가정하는가?
   ```

5. **모니터링 설정**
   ```
   배포 후 모니터링할 지표:
   - 에러율 (5xx, 4xx 에러 수)
   - P99 응답시간
   - 데이터베이스 연결 수
   - 메모리/CPU 사용량
   - 외부 API 응답 시간
   
   임계값 설정:
   - 에러율 > 5% → 자동 롤백?
   - P99 응답시간 > 2초 → 알림?
   ```

</details>

---

<div align="center">

**[⬅️ 이전: Rolling Update 완전 분해](./01-rolling-update-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Canary 배포 — 트래픽을 단계적으로 ➡️](./03-canary-deployment.md)**

</div>
