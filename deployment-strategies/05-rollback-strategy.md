# 롤백 전략 — 언제 되돌릴 수 없는가

## 🎯 핵심 질문
- `kubectl rollout undo`는 정확히 무엇을 되돌리는가?
- `revisionHistoryLimit`을 설정하지 않으면 어떻게 되는가?
- DB 마이그레이션이 있으면 왜 롤백이 불가능한가?
- Rolling Update 롤백이 Blue-Green 롤백보다 느린 이유는?

## 🔍 왜 이 개념이 실무에서 중요한가

배포 전략을 배웠다면, 이제 실패에 대비해야 합니다:
- **"배포 직후 에러가 나면 언제나 롤백 가능한가?"** → 데이터 마이그레이션 때문에 불가능할 수 있음
- **"언제까지 이전 버전으로 롤백할 수 있는가?"** → `revisionHistoryLimit` 설정에 따라 다름
- **"롤백 중에 무중단을 보장할 수 있는가?"** → 전략과 설정에 따라 다름

실무에서 흔한 실수:
1. Rolling Update 배포 후 2시간 뒤 에러 발견 → 롤백 시도 → 15초 롤링 진행 중 에러 (왜지?)
2. DB 스키마 변경 후 롤백 시도 → 코드는 구 버전이어도 스키마는 새 버전 (불일치!)
3. 과거 버전으로 롤백하려고 했는데 revision history가 없음

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```bash
# ❌ 배포 후 문제 발생 → 무작정 롤백 시도
kubectl rollout undo deployment/myapp

# 하지만...
# 1. 이전 ReplicaSet이 더 이상 존재하지 않음 (revisionHistoryLimit 초과)
# 2. 롤백 중 새 버전과 구 버전이 혼재됨 (5-20초)
# 3. 진행 중인 요청들이 갑자기 구 코드로 처리됨

# 결과: 여전히 에러 상태
```

**회사에서 일어나는 실제 상황:**
```
09:00 배포 (v2.0 → v3.0)
      Rolling Update로 순차 진행
      
09:05 배포 완료 (모든 Pod이 v3.0)

09:30 치명적 에러 발견
      "사용자 데이터가 사라졌다!"
      
09:31 롤백 시도
      kubectl rollout undo deployment/myapp
      
09:32 롤백 완료 (모든 Pod이 v2.0)
      하지만... 사라진 데이터는?

문제:
v3.0 코드가 이미 DB에 데이터를 변경했음
v2.0 코드가 조회해도 데이터 구조가 달라짐
→ 롤백해도 데이터 손실 되돌릴 수 없음
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 1단계: 롤백 가능성 미리 계획

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  
  # ✅ 여러 revision 보존 (기본값: 10)
  # 이전 버전으로 롤백할 수 있는 기간 확보
  revisionHistoryLimit: 10
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
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
        image: myapp:3.0
        # ... 나머지 설정
```

### 2단계: DB 마이그레이션 시 Backward Compatibility 확보

```yaml
# ❌ 위험한 DB 마이그레이션 (v2에서 v3으로)
# v3: users 테이블에 new_column 추가, 그리고 old_column 제거

# ✅ 안전한 DB 마이그레이션 (3단계 필요)
# 단계 1: new_column 추가 (v2도 읽을 수 있는 상태)
# 단계 2: 코드 v3 배포 (v3이 new_column 사용, old_column도 읽음)
# 단계 3: 안정화 후 old_column 제거 (v3만 남음)

apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-add-column
spec:
  template:
    spec:
      containers:
      - name: migration
        image: liquibase:latest
        env:
        - name: SQL
          value: |
            ALTER TABLE users ADD COLUMN new_column VARCHAR(255) NULL;
            -- nullable로 설정 (v2도 insert 가능)
      restartPolicy: Never
---
# 이 migration 완료 후 코드 배포
# v3 코드는 new_column을 읽지만, old_column도 읽을 수 있음
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:3.0  # 이중 읽기 지원
---
# 며칠 뒤 old_column 제거 (v3가 완전히 안정화된 후)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-remove-column
spec:
  template:
    spec:
      containers:
      - name: migration
        image: liquibase:latest
        env:
        - name: SQL
          value: |
            ALTER TABLE users DROP COLUMN old_column;
      restartPolicy: Never
```

### 3단계: 롤백 전 데이터 일관성 확인

```bash
# 롤백 전 수동 점검 체크리스트

# 1. 배포 히스토리 확인
kubectl rollout history deployment/myapp

# 2. 롤백할 revision의 상세정보 확인
kubectl rollout history deployment/myapp --revision=2

# 3. DB 스키마 상태 확인
# v3 코드에서만 사용하는 column이 있는가?
mysql> DESC users;

# 4. 데이터 일관성 확인
# 이미 v3에서 변경된 데이터가 v2 코드로 처리 가능한가?
mysql> SELECT * FROM users WHERE created_at > '2024-04-10 09:00';

# 5. 진행 중인 요청 확인
kubectl get pods -l app=myapp -o wide | grep "NotReady"

# 6. 안전 확인 후 롤백
kubectl rollout undo deployment/myapp --to-revision=2
```

## 🔬 내부 동작 원리

### ReplicaSet의 Revision 관리

```
Deployment을 생성하면:
  Deployment
    ├─ ReplicaSet v1 (revision 1)
    │   └─ Pod 3개 (image: myapp:1.0)
    
새 버전 배포 (image: myapp:2.0):
  Deployment 업데이트
    ├─ ReplicaSet v1 (revision 1) → 스케일 다운 (0개)
    └─ ReplicaSet v2 (revision 2) ← 스케일 업 (3개)
    
또 다시 배포 (image: myapp:3.0):
  Deployment 업데이트
    ├─ ReplicaSet v1 (revision 1) → 여전히 0개 (보존됨!)
    ├─ ReplicaSet v2 (revision 2) → 스케일 다운 (0개)
    └─ ReplicaSet v3 (revision 3) ← 스케일 업 (3개)

revisionHistoryLimit: 10 설정 시:
  최대 10개의 ReplicaSet 보존
  11번째 배포 시 가장 오래된 ReplicaSet 삭제

결과:
  언제든 이전 버전으로 "즉시" 롤백 가능
  (새 Pod 생성이 아니라 기존 ReplicaSet만 활성화)
```

### kubectl rollout undo의 실제 동작

```
명령어: kubectl rollout undo deployment/myapp --to-revision=2

1단계: 이전 ReplicaSet 찾기
       revision 2의 ReplicaSet 객체 검색
       
2단계: 현재 버전 (v3)의 replicas를 0으로 설정
       ReplicaSet v3: replicas: 0 → Pod 0개
       
3단계: 이전 버전 (v2)의 replicas를 3으로 설정
       ReplicaSet v2: replicas: 3 → Pod 3개 생성 시작
       
4단계: Pod 생성 및 Ready 대기
       v2 이미지로 3개 Pod 생성
       readinessProbe 체크
       
5단계: Service endpoints 업데이트
       새로 생성된 Pod들이 Ready되면 endpoints에 추가
       
결과:
  ✅ 빠른 복구 (새 이미지 다운로드 안 함, 캐시된 이미지 사용)
  ✅ 이전 설정 그대로 적용 (환경변수, 리소스 제한 등)
```

### Rolling Update 롤백 vs Blue-Green 롤백

```
Rolling Update 배포 후 롤백:
  배포 단계: v2 Pod 1 → v2 Pod 2 → v2 Pod 3
  
  롤백: v2 Pod 3 → v1 Pod 3 (다시 Rolling Update!)
        약 15-25초 소요
        
        이유: ReplicaSet이 새로 Pod을 생성해야 함
        
Blue-Green 롤백:
  배포 단계: Blue(v1) 3개 유지 + Green(v2) 3개 생성
  
  롤백: Service selector 변경 (green → blue)
        밀리초 단위 (기존 Blue Pod 활성화)
        
결론: 롤백 속도 차이 큼!
```

## 💻 실전 실험 (kubectl, YAML로 재현 가능한 예시)

### 실험 1: revisionHistoryLimit 확인

```bash
# 1. Deployment 생성
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 5  # 5개 revision만 보존
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
EOF

# 2. 여러 버전 배포 (6번)
for i in {1..6}; do
  echo "Deployment $i: nginx:1.$((18+i))"
  kubectl set image deployment/myapp app=nginx:1.$((18+i))
  sleep 5
done

# 3. 배포 히스토리 확인
kubectl rollout history deployment/myapp

# 출력:
# REVISION  CHANGE-CAUSE
# 2         <none>
# 3         <none>
# 4         <none>
# 5         <none>
# 6         <none>  ← revision 1은 삭제됨 (revisionHistoryLimit=5)

# 4. ReplicaSet 확인
kubectl get rs -l app=myapp
# 출력:
# NAME                     DESIRED   CURRENT   READY   AGE
# myapp-abc123 (rev 2)     0         0         0       25m
# myapp-def456 (rev 3)     0         0         0       20m
# myapp-ghi789 (rev 4)     0         0         0       15m
# myapp-jkl012 (rev 5)     0         0         0       10m
# myapp-mno345 (rev 6)     3         3         3       2m   ← 현재 활성

# 5. revision 1로 롤백 시도 (삭제됨)
kubectl rollout undo deployment/myapp --to-revision=1

# 에러:
# error: unable to find specified revision 1
```

### 실험 2: 안전한 롤백 프로세스

```bash
# 1. 현재 상태 확인
kubectl get deployment myapp -o wide

# 2. 배포 히스토리
kubectl rollout history deployment/myapp

# 3. 롤백할 revision의 상세정보
kubectl rollout history deployment/myapp --revision=4
# 출력:
# myapp
# Pod Template:
#   Containers:
#    app
#     Image:      nginx:1.22

# 4. 테스트 (현재 버전 먼저 확인)
POD=$(kubectl get pod -l app=myapp -o name | head -1)
kubectl exec $POD -- nginx -v
# output: nginx version: nginx/1.23

# 5. 롤백 (dry-run으로 먼저 확인)
kubectl rollout undo deployment/myapp --to-revision=4 --dry-run=client

# 6. 실제 롤백
kubectl rollout undo deployment/myapp --to-revision=4

# 7. 롤백 진행 상황 확인
kubectl rollout status deployment/myapp

# 8. 롤백 후 버전 확인
POD=$(kubectl get pod -l app=myapp -o name | head -1)
kubectl exec $POD -- nginx -v
# output: nginx version: nginx/1.22 (이전 버전)
```

### 실험 3: DB 마이그레이션 시나리오

```bash
# ❌ 안전하지 않은 DB 마이그레이션
cat <<'EOF' | kubectl apply -f -
# 1. DB 마이그레이션
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v3
spec:
  template:
    spec:
      containers:
      - name: migration
        image: busybox
        command: 
        - sh
        - -c
        - |
          # "users" 테이블에서 "old_column"을 "new_column"으로 이름 변경
          # 이 작업은 v2와 v3 사이에 호환성이 없음!
          echo "Renaming old_column to new_column"
          # ALTER TABLE users CHANGE COLUMN old_column new_column VARCHAR(255);
      restartPolicy: Never
---
# 2. 코드 배포 (v3: new_column 사용)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
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
        image: myapp:3.0  # v3: new_column만 읽음
EOF

# ❌ 몇 시간 후 롤백 시도
kubectl rollout undo deployment/myapp --to-revision=1

# 결과:
# - 코드: v2 (old_column 사용)
# - DB: v3 (new_column만 존재)
# 불일치! → 에러

# ✅ 올바른 방법 (backward compatible)
cat <<'EOF' | kubectl apply -f -
# 1. DB 마이그레이션 (column 추가)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-add-column
spec:
  template:
    spec:
      containers:
      - name: migration
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Adding new_column (nullable)"
          # ALTER TABLE users ADD COLUMN new_column VARCHAR(255) NULL;
      restartPolicy: Never
---
# 2. 코드 배포 (v3: 둘 다 읽기 지원)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
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
        image: myapp:3.0  # v3: old_column, new_column 모두 읽음
---
# 3. 안정화 확인 후 (며칠 뒤)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-remove-column
spec:
  template:
    spec:
      containers:
      - name: migration
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Removing old_column (v3만 운영 중이므로 안전)"
          # ALTER TABLE users DROP COLUMN old_column;
      restartPolicy: Never
EOF

# 이제 안전하게 롤백 가능!
kubectl rollout undo deployment/myapp --to-revision=1
# v2 코드도 여전히 작동 (new_column은 읽지 않음)
```

### 실험 4: 롤백 불가능한 상황 감지

```bash
#!/bin/bash
# pre-rollback-check.sh - 롤백 전 안전성 검사

DEPLOYMENT="myapp"
TARGET_REVISION=$1

if [ -z "$TARGET_REVISION" ]; then
  echo "Usage: $0 <revision>"
  exit 1
fi

echo "=== Pre-Rollback Safety Check ==="

# 1. Revision 존재 여부 확인
echo "1. Checking revision existence..."
kubectl rollout history deployment/$DEPLOYMENT --revision=$TARGET_REVISION > /dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "❌ Revision $TARGET_REVISION not found!"
  exit 1
fi
echo "✓ Revision $TARGET_REVISION exists"

# 2. 현재 버전과 대상 버전 비교
echo "2. Comparing current vs target version..."
CURRENT_IMAGE=$(kubectl get deployment $DEPLOYMENT \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
TARGET_IMAGE=$(kubectl rollout history deployment/$DEPLOYMENT \
  --revision=$TARGET_REVISION 2>/dev/null | grep "Image:" | awk '{print $2}')

echo "  Current: $CURRENT_IMAGE"
echo "  Target:  $TARGET_IMAGE"

# 3. DB 스키마 호환성 검사 (수동)
echo "3. DB schema compatibility (manual check needed):"
echo "   - Has schema changed since target revision?"
echo "   - Are there backward-incompatible migrations?"
read -p "  Are you sure? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "❌ Rollback cancelled"
  exit 1
fi

# 4. 현재 Pod 상태 확인
echo "4. Checking current Pod status..."
NOT_READY=$(kubectl get pods -l app=$DEPLOYMENT \
  -o jsonpath='{range .items[?(@.status.conditions[?(@.type=="Ready")].status=="False")]}1{end}' | wc -c)

if [ $NOT_READY -gt 1 ]; then
  echo "⚠️  Warning: $NOT_READY pod(s) are not ready"
fi

# 5. 최종 확인
echo "5. Final confirmation before rollback..."
read -p "  Execute rollback to revision $TARGET_REVISION? (yes/no): " FINAL

if [ "$FINAL" = "yes" ]; then
  echo "✓ Rolling back to revision $TARGET_REVISION..."
  kubectl rollout undo deployment/$DEPLOYMENT --to-revision=$TARGET_REVISION
  kubectl rollout status deployment/$DEPLOYMENT
else
  echo "❌ Rollback cancelled"
fi
```

```bash
# 실행
chmod +x pre-rollback-check.sh
./pre-rollback-check.sh 3
```

### 실험 5: 동시 롤백 불가능 상황

```bash
# 1. 구성: 3개 Pod이 각각 다른 이미지 버전 실행
# (이상한 상황이지만, kubectl patch로 강제할 수 있음)

# Pod1: nginx:1.19
kubectl patch pods pod1 -p '{"spec":{"containers":[{"name":"app","image":"nginx:1.19"}]}}'

# Pod2: nginx:1.20
kubectl patch pods pod2 -p '{"spec":{"containers":[{"name":"app","image":"nginx:1.20"}]}}'

# Pod3: nginx:1.21
kubectl patch pods pod3 -p '{"spec":{"containers":[{"name":"app","image":"nginx:1.21"}]}}'

# 2. 문제: 어느 버전으로 롤백할 것인가?

# 3. ReplicaSet도 혼란 상태
kubectl get rs -o wide
# 각 Pod이 다른 ReplicaSet에 속함

# 4. Deployment를 통한 일관성 복구
kubectl rollout undo deployment/myapp

# 결과: 모든 Pod이 동일한 이미지로 통일됨
# (마지막 정상 상태인 이전 revision으로)
```

## 📊 성능/비용 비교 (가용성, 배포 시간, 비용)

| 항목 | Rolling Update | Blue-Green | 데이터 마이그레이션 |
|------|---|---|---|
| **롤백 시간** | 15-40초 | <1초 | 불가능 |
| **롤백 데이터 손실** | 없음 | 없음 | 있을 수 있음 |
| **과거 버전 복구** | 최대 10개 | 최대 10개 | 스키마 변경 시 불가 |
| **운영 복잡도** | 낮음 | 중간 | 매우 높음 |
| **준비 시간** | 없음 | 2배 인프라 | 마이그레이션 계획 필수 |

## ⚖️ 트레이드오프

### 빠른 롤백 vs 데이터 안정성

```
Blue-Green (빠른 롤백):
  ✅ <1초 롤백
  ❌ DB 마이그레이션 불가능
  ❌ 혼재 기간 없음 (좋은 점)

Rolling Update (느린 롤백):
  ❌ 15-40초 롤백
  ✅ DB 마이그레이션 가능 (단계적)
  ✅ 호환성 계층 추가 가능

결론: 데이터 일관성 > 롤백 속도
```

### revisionHistoryLimit 값 선택

```
revisionHistoryLimit: 10 (기본값)
  ✅ 과거 10개 버전 보존
  ✅ 저장소 사용량 적음
  ❌ 10개 이상 역사 보존 불가
  
  사용: 일일 배포 2회 이상 (약 5일 역사)

revisionHistoryLimit: 100
  ✅ 과거 100개 버전 보존
  ✅ 매우 오래 전 버전으로도 롤백 가능
  ❌ etcd 저장소 사용량 증가
  
  사용: 빈번한 배포 + 장기 호환성 필요

revisionHistoryLimit: 0 (무제한 보존)
  ❌ etcd 저장소 계속 증가
  ❌ 쿠버네티스 성능 저하
  
  권장하지 않음
```

## 📌 핵심 정리

1. **롤백의 핵심:**
   - ReplicaSet은 revision별로 보존됨
   - `kubectl rollout undo`는 이전 ReplicaSet을 활성화
   - Pod 생성이 아니라 기존 ReplicaSet 재활용 → 빠른 복구

2. **롤백 불가능한 경우:**
   - DB 스키마 변경 (호환성 계층 없음)
   - revisionHistoryLimit 초과로 삭제된 revision
   - 데이터 손실이 이미 발생 (코드 롤백으로는 복구 불가)

3. **안전한 DB 마이그레이션:**
   ```
   1단계: 새 column 추가 (nullable)
   2단계: 코드 배포 (이중 읽기)
   3단계: 안정화 확인 (며칠)
   4단계: 구 column 제거
   ```

4. **롤백 전 체크리스트:**
   - Revision 존재 여부
   - 현재/대상 이미지 버전 확인
   - DB 스키마 호환성 검사
   - Pod 상태 정상 여부

## 🤔 생각해볼 문제 (+ 해설)

### 문제 1: 두 가지 배포 방식의 롤백 속도 차이는?

```
Rolling Update 롤백:
  v3 (3개) → v2 (3개)
  - maxUnavailable=0이면
  - 새 Pod 생성 + Ready 대기 필요
  - 약 20-40초

Blue-Green 롤백:
  selector: green → blue
  - 기존 Pod만 활성화
  - <1초
```

<details>
<summary>정답 및 해설</summary>

**Rolling Update 롤백 상세 프로세스:**
```
1. Deployment가 이전 ReplicaSet v2 활성화
2. ReplicaSet v2: replicas 3으로 설정
3. kubelet이 Pod 생성 요청
4. 컨테이너 이미지 pull (캐시에서 빠름, 약 5초)
5. 컨테이너 시작, readinessProbe 시작
6. Pod가 Ready 상태 되기까지 (initialDelaySeconds + 체크 시간)
   약 10-15초
7. ReplicaSet v3이 순차적으로 스케일 다운
   maxUnavailable=0이므로 v2가 Ready된 후
   약 5-10초 추가

총 소요 시간: 20-40초

Blue-Green 롤백:
1. Service selector 변경 (green → blue)
2. API Server 업데이트 (밀리초)
3. Endpoints 컨트롤러 감지 (1-2초)
4. iptables 규칙 변경 (밀리초)
5. 새로운 연결이 blue로 라우팅됨

총 소요 시간: <1초
```

**왜 Blue-Green이 빠른가?**
```
기존 Pod을 재활용 vs 새 Pod 생성
- 재활용: 메모리, 캐시 상태 유지 → 즉시 사용 가능
- 새 생성: 이미지 pull, 초기화, readiness 대기 필요
```

</details>

### 문제 2: revisionHistoryLimit=3이고 5번 배포했다면, 몇 개의 revision으로 롤백 가능한가?

<details>
<summary>정답 및 해설</summary>

**답:** 3개 revision

**상황:**
```
배포 1: revision 1
배포 2: revision 2
배포 3: revision 3
배포 4: revision 4
배포 5: revision 5 ← 현재

revisionHistoryLimit=3이므로:
가장 오래된 revision 2개(1, 2)는 삭제됨

보존되는 revision: 3, 4, 5

현재: revision 5
롤백 가능: revision 4, 3
```

**커맨드:**
```bash
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 3         <none>
# 4         <none>
# 5         <none>  ← 현재

kubectl rollout undo deployment/myapp --to-revision=4
# OK

kubectl rollout undo deployment/myapp --to-revision=2
# Error: unable to find specified revision 2
```

**영향:**
```
일일 배포 5회 기준:
- revisionHistoryLimit=10 → 2일 역사 유지
- revisionHistoryLimit=3 → 12시간 역사만 유지
```

</details>

### 문제 3: 롤백 도중 새 버전으로 다시 배포하면 어떻게 되는가?

<details>
<summary>정답 및 해설</summary>

**시나리오:**
```
배포 히스토리:
revision 1: v1
revision 2: v2
revision 3: v3 (현재)

롤백 시작: v3 → v1
  v1 Pod 생성 중 (2개 생성, 1개 남음)

롤백 진행 중 갑자기:
  새 버전 배포: kubectl set image deployment/myapp app=v4:latest
```

**Deployment의 동작:**
```
1. 기존 롤백 작업 취소
2. 새 버전 배포 우선
3. v3 ReplicaSet 스케일 다운 → 0개 (v1으로의 롤백 취소)
4. v4 ReplicaSet 생성 및 스케일 업

결과:
- revision 1로의 롤백이 완료되지 않음
- 대신 새로운 revision 4가 생성됨
- 결국 v4가 실행됨 (v1 아님)
```

**커맨드 log:**
```bash
# 롤백 진행 중
kubectl rollout undo deployment/myapp --to-revision=1

# (롤백 진행 중)
kubectl set image deployment/myapp app=v4:latest

# 결과 확인
kubectl rollout status deployment/myapp

# 현재 running version
kubectl get pods -l app=myapp -o jsonpath='{.items[0].spec.containers[0].image}'
# v4
```

**결론:**
```
마지막 kubectl 명령이 우선됨
Deployment는 최신 명령을 따름
롤백 중 새 배포 명령은 롤백을 취소
```

</details>

---

<div align="center">

**[⬅️ 이전: Argo Rollouts — 메트릭 기반 자동 승급](./04-argo-rollouts.md)** | **[홈으로 🏠](../README.md)** | **[다음: Zero-Downtime 배포 조건 ➡️](./06-zero-downtime.md)**

</div>
