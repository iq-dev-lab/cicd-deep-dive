# 장애 시나리오와 복구 — 자동 롤백 트리거와 Postmortem

## 🎯 핵심 질문

- "배포 후 에러율이 급증했어요. 어떻게 빠르게 대응해야 하나요?"
- "자동으로 이전 버전으로 롤백할 수 있을까요?"
- "데이터베이스는 어떻게 복구하나요?" — 롤백이 항상 가능한가?
- "장애가 발생한 후 같은 실수를 반복하지 않으려면?"
- "우리 팀이 진짜 대응할 수 있는지 테스트해 볼 수 있을까요?"

## 🔍 왜 이 개념이 실무에서 중요한가

**배포 후 장애는 필연이다.** 100% 안전한 배포는 없다. 중요한 것은 **얼마나 빨리 감지하고 복구하는가**다.

실제 사례:
- **대응 없이 1시간 다운**: 에러율 감지는 빨랐지만, 롤백 절차를 몰라서 수동 복구에만 1시간 소요.
- **자동 롤백 실패**: Prometheus Alert 발동 → ArgoCD 롤백 시도 → 실패 (이유: 이미지 없음). 결국 수동으로 kubectl 사용.
- **데이터베이스 손상**: 배포 직후 데이터베이스 마이그레이션이 실패했지만 감지 안 됨. 여러 시간 후 발견.
- **Postmortem 작성 안 함**: 같은 문제로 3번 장애 발생.

**현실:**
```
배포 실패 가능성: ~5-10%
대응 시간 (없는 경우): 30-60분
대응 시간 (자동 롤백): 1-2분
```

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# Before: 대응 절차가 없음
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        run: |
          kubectl set image deployment/api api=myapp:${{ github.sha }} \
            --namespace=production
      
      # 배포 완료. 이제 무엇? 누군가 모니터링하나?
      # 에러율 증가했는데 아무도 모른다?
      # 대응 절차는?
      # 롤백 방법은?
```

**문제점:**
1. 에러율 급증을 감지하는 메커니즘 없음
2. 자동 롤백 트리거 없음
3. 롤백 방법을 모름 (kubectl? ArgoCD?)
4. 장애 대응 체크리스트 없음
5. 사후 분석(Postmortem) 문화 없음

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# After: 자동 감지, 롤백, 알림, 사후 분석
name: Deploy with Incident Recovery

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    outputs:
      deployment-id: ${{ steps.deploy.outputs.id }}
      previous-image: ${{ steps.get-prev-image.outputs.image }}
    steps:
      - uses: actions/checkout@v4
      
      # 이전 이미지 저장 (롤백용)
      - name: Get previous deployment image
        id: get-prev-image
        run: |
          PREVIOUS_IMAGE=$(kubectl get deployment api -n production \
            -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "image=$PREVIOUS_IMAGE" >> $GITHUB_OUTPUT
          echo "Previous image: $PREVIOUS_IMAGE"
      
      # 배포 수행
      - name: Deploy to Production
        id: deploy
        run: |
          DEPLOYMENT_ID=$(kubectl set image deployment/api api=myapp:${{ github.sha }} \
            --namespace=production \
            --record \
            --output=jsonpath='{.metadata.uid}')
          
          # 배포 롤아웃 완료 대기
          kubectl rollout status deployment/api -n production --timeout=5m
          
          echo "id=$DEPLOYMENT_ID" >> $GITHUB_OUTPUT
          echo "Deployment completed: $DEPLOYMENT_ID"
      
      # 배포 직후 헬스 체크 (조기 감지)
      - name: Immediate health check
        id: health
        run: |
          RETRIES=3
          for i in $(seq 1 $RETRIES); do
            if curl -f https://api.example.com/health; then
              echo "Health check passed"
              exit 0
            fi
            
            if [ $i -lt $RETRIES ]; then
              echo "Health check attempt $i failed, retrying..."
              sleep 5
            fi
          done
          
          echo "Health check failed, marking deployment as unhealthy"
          exit 1
      
      - name: Notify deployment success
        if: success()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-type: application/json' \
            -d '{
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": ":white_check_mark: Production deployment successful\n*Commit:* ${{ github.sha }}"
                }
              }]
            }'
      
      - name: Initiate rollback on failure
        if: failure()
        run: |
          echo "Deployment failed, initiating rollback..."
          
          PREVIOUS_IMAGE="${{ steps.get-prev-image.outputs.image }}"
          
          # 1. 즉시 이전 이미지로 롤백
          kubectl set image deployment/api api=$PREVIOUS_IMAGE \
            --namespace=production
          
          # 2. 롤백 완료 대기
          kubectl rollout status deployment/api -n production --timeout=5m
          
          # 3. 긴급 알림
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-type: application/json' \
            -d '{
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": ":warning: AUTOMATIC ROLLBACK TRIGGERED\n*Reason:* Post-deployment health check failed\n*Rolled back to:* '$PREVIOUS_IMAGE'"
                }
              }]
            }'
          
          exit 1

  # 실시간 모니터링 (Prometheus Alert)
  monitor-deployment:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: Monitor for 10 minutes post-deployment
        run: |
          echo "Starting post-deployment monitoring..."
          
          # 10분간 메트릭 확인
          for minute in {1..10}; do
            echo "Minute $minute: Checking metrics..."
            
            # Prometheus API 호출
            ERROR_RATE=$(curl -s 'http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~"5.."}[5m])' \
              | jq '.data.result[0].value[1]' 2>/dev/null || echo "0")
            
            # 에러율 5% 이상이면 즉시 알림
            if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
              echo "⚠️  Error rate exceeded 5%: $ERROR_RATE"
              
              # Prometheus Alert 발동 (자동 롤백 시작)
              curl -X POST "http://alertmanager:9093/api/v1/alerts" \
                -H 'Content-Type: application/json' \
                -d @- <<EOF
              [{
                "status": "firing",
                "labels": {
                  "alertname": "HighErrorRatePostDeployment",
                  "severity": "critical",
                  "service": "api"
                },
                "annotations": {
                  "summary": "High error rate after deployment",
                  "description": "Error rate: $ERROR_RATE (threshold: 5%)"
                }
              }]
              EOF
            fi
            
            sleep 60
          done

  # 자동 롤백 Rule (Prometheus + ArgoCD)
  auto-rollback:
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Trigger ArgoCD rollback
        run: |
          # ArgoCD API로 이전 sync 상태로 롤백
          curl -X POST "http://argocd.example.com/api/v1/applications/api/sync" \
            -H "Authorization: Bearer ${{ secrets.ARGOCD_TOKEN }}" \
            -H 'Content-Type: application/json' \
            -d '{
              "revision": "HEAD~1",
              "syncPolicy": {
                "automated": {
                  "prune": true,
                  "selfHeal": true
                }
              }
            }'
          
          echo "ArgoCD rollback initiated"
      
      - name: Verify rollback
        run: |
          # 롤백 완료 대기
          sleep 30
          
          # 헬스 체크
          if curl -f https://api.example.com/health; then
            echo "✅ Service recovered after rollback"
          else
            echo "❌ Service still unhealthy after rollback - escalate"
            exit 1
          fi

  # Incident 생성 및 대응
  incident-response:
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Create incident
        id: incident
        run: |
          # PagerDuty 인시던트 생성
          INCIDENT_ID=$(curl -X POST "https://api.pagerduty.com/incidents" \
            -H "Authorization: Token token=${{ secrets.PAGERDUTY_TOKEN }}" \
            -H 'Content-Type: application/json' \
            -d @- <<EOF | jq -r '.incident.incident_number'
          {
            "incident": {
              "type": "incident",
              "title": "Production deployment failed - Auto-rollback triggered",
              "service": {
                "id": "api-service-id",
                "type": "service_reference"
              },
              "urgency": "high",
              "body": {
                "type": "incident_body",
                "details": "Commit: ${{ github.sha }}, Branch: ${{ github.ref }}"
              }
            }
          }
          EOF
          )
          
          echo "incident-id=$INCIDENT_ID" >> $GITHUB_OUTPUT
      
      - name: Post incident to war room
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_WAR_ROOM }}" \
            -H 'Content-type: application/json' \
            -d '{
              "text": "🚨 INCIDENT #${{ steps.incident.outputs.incident-id }}",
              "blocks": [
                {
                  "type": "header",
                  "text": {"type": "plain_text", "text": "🚨 Production Incident"}
                },
                {
                  "type": "section",
                  "fields": [
                    {"type": "mrkdwn", "text": "*Status*: Active"},
                    {"type": "mrkdwn", "text": "*Service*: API"},
                    {"type": "mrkdwn", "text": "*Trigger*: Post-deployment health check failure"},
                    {"type": "mrkdwn", "text": "*Action*: Automatic rollback initiated"}
                  ]
                }
              ]
            }'
      
      - name: Create Postmortem template
        run: |
          # 사후 분석 문서 자동 생성
          cat > postmortem-template.md <<EOF
          # Postmortem: Production Incident ${{ steps.incident.outputs.incident-id }}
          
          **Date:** $(date)
          **Severity:** High
          **Duration:** TBD (Rollback initiated automatically)
          
          ## Timeline
          - **Deployment started:** ${{ github.event.head_commit.timestamp }}
          - **Health check failed:** (time)
          - **Rollback initiated:** (time)
          - **Service recovered:** (time)
          
          ## What happened?
          - TODO: Fill in after incident is resolved
          
          ## Root cause
          - TODO: Investigation required
          
          ## Impact
          - TODO: Error rate, affected users, etc.
          
          ## Action items
          - [ ] Investigate root cause
          - [ ] Implement fix
          - [ ] Add monitoring/alerting
          - [ ] Update deployment runbook
          
          ## Lessons learned
          - TODO
          
          ---
          **Investigation deadline:** $(date -d '+24 hours')
          EOF
          
          # GitHub Issue로 생성
          gh issue create \
            --title "Postmortem: Incident #${{ steps.incident.outputs.incident-id }}" \
            --body "$(cat postmortem-template.md)" \
            --label incident,postmortem
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # 정기 재해 복구 테스트 (Game Day)
  disaster-recovery-test:
    if: github.event_name == 'schedule'  # 매주 목요일 실행
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Start simulated deployment failure
        run: |
          echo "Starting Game Day: Simulated deployment failure"
          
          # 카오스 엔지니어링: 일부러 실패 유도
          kubectl set image deployment/api api=myapp:broken-image \
            --namespace=production || true
          
          # 에러율 모니터링
          sleep 30
      
      - name: Team validates incident response
        run: |
          echo "Testing incident response procedures..."
          
          # 체크리스트
          echo "☐ Slack 알림 수신했나?"
          echo "☐ PagerDuty 인시던트 생성됐나?"
          echo "☐ War room 채널에 알림 왔나?"
          echo "☐ 롤백 프로세스 시작됐나?"
          echo "☐ Postmortem 템플릿 생성됐나?"
      
      - name: Verify automatic recovery
        run: |
          # 자동 롤백이 작동했는지 확인
          for i in {1..30}; do
            if curl -f https://api.example.com/health; then
              echo "✅ Service recovered automatically in $((i*10)) seconds"
              exit 0
            fi
            sleep 10
          done
          
          echo "❌ Service did not recover automatically"
          exit 1
      
      - name: Cleanup and report
        run: |
          # Game Day 정리
          kubectl set image deployment/api api=myapp:latest \
            --namespace=production
          
          # 결과 리포트
          echo "Game Day completed successfully!"
```

## 🔬 내부 동작 원리

### 자동 롤백 메커니즘

```
배포 수행
  ↓
헬스 체크 (1-2분)
  ↓
  PASS: 계속 모니터링
  FAIL: 즉시 롤백 트리거
  
롤백 수행 (kubectl 또는 ArgoCD)
  ↓
이전 이미지로 교체
  ↓
Pod 재시작
  ↓
헬스 체크 (서비스 정상 여부 확인)
```

**두 가지 롤백 방식:**

1. **kubectl 롤백** (즉각적):
```bash
kubectl rollout undo deployment/api -n production
# 또는
kubectl set image deployment/api api=previous-image
```

2. **ArgoCD 롤백** (선언형):
```bash
argocd app rollback api-app HEAD~1
# Git에 저장된 이전 버전으로 자동 동기화
```

### Prometheus Alert → Incident 흐름

```
Prometheus Alert Rule
  ↓
Alert Manager (email/Slack/PagerDuty)
  ↓
Webhook (자동 롤백 트리거)
  ↓
ArgoCD/kubectl (실행)
  ↓
서비스 복구
```

**Alert Rule 예시:**
```yaml
# prometheus/rules.yaml
- alert: HighErrorRatePostDeployment
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
  for: 1m
  annotations:
    summary: "High error rate detected"
    runbook: "https://wiki.example.com/runbooks/high-error-rate"
```

### Postmortem 템플릿과 사후 분석 문화

```
Incident 발생
  ↓ (해결되자마자)
Postmortem 시작
  ↓
Timeline 작성 (정확한 시각 기록)
  ↓
근본 원인 분석 (Why x5)
  ↓
Action items 정의 (개선사항)
  ↓
팀 전체 공유 및 학습
```

## 💻 실전 실험 (GitHub Actions YAML, CLI 명령어로 재현 가능)

### 실험 1: 자동 롤백 테스트

```bash
# 1. 현재 배포 상태 확인
kubectl get deployment api -n production -o wide

# 2. 새로운 "broken" 이미지로 배포
kubectl set image deployment/api api=myapp:broken \
  --namespace=production

# 3. 자동 롤백이 작동해야 함 (구현되었다면)
# 또는 수동으로 롤백
kubectl rollout undo deployment/api -n production

# 4. 상태 확인
kubectl rollout history deployment/api -n production
```

### 실험 2: Prometheus Alert 테스트

```bash
# AlertManager 데이터베이스에 직접 alert 주입
curl -X POST "http://alertmanager:9093/api/v1/alerts" \
  -H 'Content-Type: application/json' \
  -d @- <<EOF
[{
  "status": "firing",
  "labels": {
    "alertname": "HighErrorRatePostDeployment",
    "severity": "critical"
  },
  "annotations": {
    "summary": "Test alert for rollback"
  }
}]
EOF

# AlertManager가 webhook을 호출하여 롤백을 시작해야 함
```

### 실험 3: Postmortem 자동 생성

```bash
# GitHub Issue로 Postmortem 템플릿 생성
gh issue create \
  --title "Postmortem: High error rate incident" \
  --body "
# Incident Postmortem

## Timeline
- 10:30: Deployment started
- 10:35: Error rate increased to 8%
- 10:36: Automatic rollback triggered
- 10:37: Service recovered

## Root cause
- Database connection pool exhausted
- New code opened connections without closing them

## Action items
- [ ] Fix database connection leak
- [ ] Add connection pool monitoring
- [ ] Review database tuning
" \
  --label incident,postmortem
```

### 실험 4: 카오스 엔지니어링 (Game Day)

```yaml
# chaos-test.yml
name: Chaos Engineering - Game Day

on:
  schedule:
    - cron: '0 14 ? * THU'  # 매주 목요일 2PM

jobs:
  chaos-test:
    runs-on: ubuntu-latest
    steps:
      - name: Scenario 1 - Pod crash
        run: |
          # 임의로 Pod 제거
          kubectl delete pod -l app=api -n production --all
          
          # 자동 복구되는지 확인
          sleep 10
          kubectl get pods -l app=api -n production
      
      - name: Scenario 2 - Network partition
        run: |
          # 네트워크 지연 주입
          kubectl exec -it -n production \
            $(kubectl get pod -l app=api -n production -o jsonpath='{.items[0].metadata.name}') \
            -- bash -c 'tc qdisc add dev eth0 root netem delay 500ms'
          
          # 서비스가 여전히 작동하는지 확인
          curl --connect-timeout 5 https://api.example.com/health || echo "Degraded"
      
      - name: Scenario 3 - Resource exhaustion
        run: |
          # 메모리 사용량 증가 시뮬레이션
          kubectl set resources deployment/api \
            --limits=memory=256Mi \
            -n production
          
          # OOM killer가 작동하는지, HPA가 스케일링하는지 확인
          sleep 20
          kubectl get deployment api -n production

      - name: Report results
        if: always()
        run: |
          echo "Game Day Results:"
          echo "1. Pod recovery: PASS/FAIL"
          echo "2. Network resilience: PASS/FAIL"
          echo "3. Auto-scaling: PASS/FAIL"
```

### 실험 5: 사후 분석 자동화

```bash
# GitHub 이슈 자동 분석
gh issue view <issue-number> \
  --json title,body,assignees,labels \
  --template '{{.title}}: {{.body}}'

# 모든 postmortem 이슈 조회
gh issue list \
  --label postmortem \
  --json title,createdAt,number \
  --template '{{range .}}#{{.number}}: {{.title}} ({{.createdAt}}){{printf "\n"}}{{end}}'
```

## 📊 성능/비용 비교

| 방식 | 대응 시간 | 준비도 | 비용 | 리스크 |
|-----|---------|-------|------|--------|
| **수동 대응** | 15-30분 | 낮음 | 없음 | 매우 높음 |
| **Alert + 수동** | 5-10분 | 중간 | 무료 | 높음 |
| **자동 롤백** | 1-2분 | 높음 | 낮음 | 낮음 |
| **카오스 테스트** | - | 높음 | 중간 | 제어됨 |

**누적 비용 계산:**
```
수동 대응으로 인한 손실:
- 20분 다운타임 × $1000/분 (기회비용) = $20,000
- 개발자 시간 4명 × 2시간 = $1,000

자동 롤백:
- 설정 비용: 2시간 × $100/시간 = $200
- 장애 시간: 2분 × $1000/분 = $2,000
= 총 $2,200 (손실 대비 90% 절감)
```

## ⚖️ 트레이드오프

### 자동 롤백 vs 인간 판단

**자동 롤백:**
- 장점: 빠른 대응
- 단점: 잘못된 롤백 가능 (이미지가 없다면?)

**알림 + 수동:**
- 장점: 인간이 판단
- 단점: 느린 대응, 휴일/야간 문제

**권장:** 자동 롤백은 항상 켜되, Pre-flight check를 추가.

### 데이터베이스 마이그레이션의 롤백 불가능성

**마이그레이션 사항:**
```sql
-- forward (배포 시)
ALTER TABLE users ADD COLUMN age INT;

-- backward (롤백 시)?
ALTER TABLE users DROP COLUMN age;
-- 하지만 데이터가 사라짐!
```

**해결책:**
1. **Blue-Green 배포**: 마이그레이션 전에 새 DB 준비
2. **Forward-only 마이그레이션**: 롤백 대신 다음 버전에서 처리
3. **Canary 배포**: 5% 트래픽으로 테스트 후 100% 배포

**권장:** 마이그레이션은 배포와 분리. 먼저 배포 성공 확인 후, 별도로 마이그레이션 수행.

## 📌 핵심 정리

1. **자동 롤백은 필수**: 수동으로 기다리는 순간 피해 증가.

2. **에러율 모니터링**: 배포 후 최소 10분은 적극적 모니터링.

3. **Postmortem 문화**: 장애 후 반드시 사후 분석. "왜?" 5번 반복.

4. **정기적 Game Day**: 3개월마다 카오스 엔지니어링으로 대응 능력 검증.

5. **데이터베이스는 Forward-only**: 마이그레이션은 배포와 분리하고 순방향만 지원.

## 🤔 생각해볼 문제

### Q1: 자동 롤백이 작동했는데 여전히 서비스가 안 돼요. 뭐가 문제일까요?

<details>
<summary>해설</summary>

다음을 확인하세요:

```bash
# 1. 이미지가 레지스트리에 존재하는가?
docker pull myregistry/myapp:previous-version

# 2. Pod이 정상 시작했는가?
kubectl describe pod -l app=api -n production
# 이미지 pull 실패? 이미지 없음? 크래시?

# 3. 헬스 체크 엔드포인트가 작동하는가?
kubectl exec -it <pod> -- curl localhost:8080/health

# 4. 의존성은 정상인가? (DB, Cache 등)
kubectl logs -l app=api -n production --tail=100

# 5. 네트워크 정책이 롤백을 막고 있지 않은가?
kubectl get networkpolicies -n production
```

**가장 흔한 원인:**
- 이전 이미지가 레지스트리에서 삭제됨
- 데이터베이스 마이그레이션이 호환되지 않음
- Pod가 시작되지만 health check 실패

**해결책:**
```bash
# 1. 마지막으로 작동했던 이미지 파악
kubectl rollout history deployment/api -n production

# 2. 그 이미지로 명시적 롤백
kubectl set image deployment/api \
  api=myregistry/myapp:v1.2.3 \
  -n production
```
</details>

### Q2: Postmortem을 써도 같은 실수가 반복돼요.

<details>
<summary>해설</summary>

Action items을 **책임자와 마감일**과 함께 작성하세요:

```markdown
## Action Items

- [ ] Fix database connection leak
  - Owner: @alice
  - Deadline: 2024-01-20
  - Tracking: #1234

- [ ] Add connection pool monitoring
  - Owner: @bob
  - Deadline: 2024-01-22
  - Tracking: #1235

- [ ] Code review process improvement
  - Owner: @charlie
  - Deadline: 2024-01-25
  - Tracking: #1236
```

그리고 **정기 Follow-up:**
```bash
# 매주 월요일 Postmortem 체크리스트 자동 생성
gh issue create \
  --title "Weekly Postmortem Follow-up" \
  --body "$(
    gh issue list --label postmortem --json number,title,assignees \
    --template '{{range .}}#{{.number}}: {{.title}}{{printf "\n"}}{{end}}'
  )"
```

또한 **같은 문제의 반복을 자동 감지:**
```bash
# 최근 3개월 Postmortem 분석
gh issue list \
  --label postmortem \
  --created=">=2024-01-01" \
  --json body | jq '.[] | .body' | \
  grep -oE "Root cause: (.+)" | sort | uniq -c
```

같은 근본 원인이 반복되면 시스템적 대책이 필요하다.
</details>

### Q3: 우리 팀이 실제로 대응할 수 있는지 테스트하려면 어떻게 할까요?

<details>
<summary>해설</summary>

**Game Day (장애 시뮬레이션) 실행:**

```yaml
# game-day.yml
name: Game Day - Incident Response Drill

on:
  workflow_dispatch:
  schedule:
    - cron: '0 14 ? * THU'  # 매주 목요일

jobs:
  game-day:
    runs-on: ubuntu-latest
    steps:
      - name: Announce Game Day
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_WAR_ROOM }}" \
            -H 'Content-type: application/json' \
            -d '{
              "text": ":fire: GAME DAY STARTED - Production deployment failure simulation"
            }'
      
      - name: Scenario - Deployment failure
        run: |
          # 일부러 배포 실패 유도
          kubectl set image deployment/api api=myapp:broken \
            --namespace=production
          
          echo "Production deployment failed - respond to incident"
      
      - name: Wait for team response (30 minutes)
        run: sleep 1800
      
      - name: Evaluate team response
        run: |
          # 체크리스트
          CHECKLIST=(
            "Incident #X created in PagerDuty"
            "War room Slack channel activated"
            "Postmortem document created"
            "Rollback initiated"
            "Service recovered"
            "Root cause identified"
          )
          
          echo "Game Day Evaluation:"
          for item in "${CHECKLIST[@]}"; do
            echo "- [ ] $item"
          done
      
      - name: Auto-recover production
        run: |
          # 게임 데이 종료 후 자동 복구
          kubectl rollout undo deployment/api -n production
```

**중요:** Game Day는 **실제 Production에서**해야 효과적하다.
Dev 환경에서는 실제와 다를 수 있는 변수가 많다.
</details>

---

[⬅️ 이전](./04-environment-management.md) | **[홈으로 🏠](../README.md)**
