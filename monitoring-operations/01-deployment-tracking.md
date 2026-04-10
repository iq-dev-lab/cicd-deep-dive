# 배포 추적과 알림 — Slack 연동과 GitHub Deployment API

## 🎯 핵심 질문

- 배포가 완료되었는데 팀원들이 모르면 어떻게 하나요?
- 배포 실패 후 어떻게 빠르게 알리고 조사할 수 있나요?
- 배포 이력을 GitHub에 기록해서 나중에 언제 뭐가 배포됐는지 확인하려면?
- 메트릭 그래프에서 배포 시점을 표시해서 "이때 뭔가 바뀌었구나"를 한눈에 알 수 있을까요?

## 🔍 왜 이 개념이 실무에서 중요한가

배포는 소프트웨어의 생사를 갈린다. 배포 실패는 서비스 장애로 직결되고, 배포 성공도 영향도를 제대로 파악해야 다음 대응을 한다. **배포 사실 자체를 팀 전체가 알아야** 조직 수준의 의사결정과 장애 대응이 빨라진다.

실제 사례:
- **배포 알림이 없었다**: 성능 저하가 발생했는데 배포 팀도, 모니터링 팀도 배포 사실을 몰랐다. 원인 파악에 2시간 소요.
- **배포 이력이 기록되지 않았다**: "이 버그 언제부터 있었지?" 하는 질문에 답할 수 없었다. PR 머지 날짜로 대충 짚어야 했다.
- **배포 마커가 없었다**: Datadog 메트릭 변화와 배포의 인과관계를 증명할 수 없었다. "배포 때문일까, 아니면 트래픽 변화 때문일까?"

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# Before: 배포 완료를 아무도 모른다
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
      
      - name: Deploy to Production
        run: |
          kubectl set image deployment/api api=myapp:${{ github.sha }} \
            --namespace=production
      
      # 배포 완료! 하지만 누가 알지?
      # Slack 메시지도 없고, 배포 이력도 기록 안 됨
```

**문제점:**
1. 배포 완료가 Slack에 공지되지 않음 → 팀원들이 모름
2. 배포 실패해도 알림이 없음 → 서비스 다운타임이 길어짐
3. 배포 이력이 GitHub에 기록되지 않음 → 롤백 판단 어려움
4. 메트릭과 배포의 연관성을 증명할 수 없음

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# After: 배포를 Slack/Discord에 알리고, GitHub에 기록하고, 메트릭에 표시
name: Deploy with Notifications

on:
  push:
    branches:
      - main

env:
  SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
  DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK_URL }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get deployment info
        id: deploy_info
        run: |
          echo "short_sha=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT
          echo "commit_author=${{ github.actor }}" >> $GITHUB_OUTPUT
          echo "commit_msg=$(git log -1 --pretty=%B)" >> $GITHUB_OUTPUT
      
      # 배포 시작 알림
      - name: Notify deployment start (Slack)
        uses: slackapi/slack-github-action@v1.26
        with:
          payload: |
            {
              "text": "배포 시작",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": ":rocket: 배포 시작"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*커밋*: <${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}|${{ steps.deploy_info.outputs.short_sha }}>"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*담당자*: ${{ github.actor }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*메시지*: ${{ steps.deploy_info.outputs.commit_msg }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*환경*: Production"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Deploy to Kubernetes
        id: deploy
        run: |
          kubectl set image deployment/api api=myapp:${{ github.sha }} \
            --namespace=production
          kubectl rollout status deployment/api -n production --timeout=5m
      
      # 배포 성공 알림
      - name: Notify deployment success (Slack)
        if: success()
        uses: slackapi/slack-github-action@v1.26
        with:
          payload: |
            {
              "text": "배포 성공",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": ":white_check_mark: 배포 성공"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*커밋*: <${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}|${{ steps.deploy_info.outputs.short_sha }}>"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*환경*: Production"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*완료 시간*: <!date^${{ github.event.repository.pushed_at }}^{date_short_pretty} {time_secs}|completed>"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "배포 로그 보기"
                      },
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      # 배포 실패 알림
      - name: Notify deployment failure (Slack)
        if: failure()
        uses: slackapi/slack-github-action@v1.26
        with:
          payload: |
            {
              "text": "배포 실패",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": ":warning: 배포 실패",
                    "emoji": true
                  }
                },
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*긴급*: 배포 실패. 즉시 조사 필요"
                  },
                  "accessory": {
                    "type": "button",
                    "text": {
                      "type": "plain_text",
                      "text": "배포 로그 확인"
                    },
                    "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
                    "style": "danger"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      # GitHub Deployment API로 배포 이력 기록
      - name: Record deployment to GitHub
        if: always()
        run: |
          # 배포 상태 결정
          if [ "${{ job.status }}" == "success" ]; then
            DEPLOYMENT_STATE="success"
          else
            DEPLOYMENT_STATE="failure"
          fi
          
          # GitHub API로 배포 기록
          gh api repos/${{ github.repository }}/deployments/${{ github.run_id }} \
            -X POST \
            -f auto_merge=false \
            -f required_contexts='[]' \
            -f description="Deployment from commit ${{ github.sha }}" \
            -f environment="production" \
            2>/dev/null || echo "Deployment record exists"
          
          # 배포 상태 업데이트
          gh api repos/${{ github.repository }}/deployments/${{ github.run_id }}/statuses \
            -X POST \
            -f state="$DEPLOYMENT_STATE" \
            -f description="Deployment $DEPLOYMENT_STATE" \
            -f environment_url="https://api.example.com" || echo "Status recorded"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      # Discord webhook으로도 알림 (선택사항)
      - name: Notify Discord
        if: always()
        run: |
          DEPLOY_STATUS="성공"
          COLOR="65280"  # 초록색
          
          if [ "${{ job.status }}" != "success" ]; then
            DEPLOY_STATUS="실패"
            COLOR="16711680"  # 빨간색
          fi
          
          curl -X POST "${{ secrets.DISCORD_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d @- <<EOF
          {
            "embeds": [
              {
                "title": "배포 $DEPLOY_STATUS",
                "description": "Commit: ${{ steps.deploy_info.outputs.short_sha }}",
                "color": $COLOR,
                "fields": [
                  {
                    "name": "담당자",
                    "value": "${{ github.actor }}",
                    "inline": true
                  },
                  {
                    "name": "환경",
                    "value": "Production",
                    "inline": true
                  }
                ],
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            ]
          }
          EOF
```

## 🔬 내부 동작 원리

### Slack Incoming Webhook 작동 방식

```
GitHub Actions → HTTP POST → Slack Incoming Webhook → Slack 채널
        ↓
    JSON 페이로드
    (메시지, 색상, 링크)
```

**Slack Webhook 등록:**
1. Slack 워크스페이스 관리자 권한으로 로그인
2. https://api.slack.com/apps → "Create New App" → "From scratch"
3. App name: "GitHub Deployment Notifications"
4. 채널 선택: #deployments
5. "Incoming Webhooks" → "Add New Webhook to Workspace"
6. 승인 후 Webhook URL 복사: `https://hooks.slack.com/services/T0000/B0000/XXXX`
7. GitHub Repository Settings → Secrets → `SLACK_WEBHOOK_URL` 저장

### GitHub Deployment API 구조

GitHub Deployment API는 배포 이력을 GitHub 상에서 관리하는 표준 인터페이스다.

```bash
# 배포 생성 (배포 이벤트를 등록)
gh api repos/owner/repo/deployments \
  -f ref="main" \
  -f environment="production" \
  -f description="Production deployment" \
  -f required_contexts='[]' \
  -f auto_merge=false

# 배포 상태 업데이트 (배포 성공/실패 기록)
gh api repos/owner/repo/deployments/{deployment_id}/statuses \
  -f state="success" \
  -f description="Deployment successful" \
  -f environment_url="https://api.example.com"

# 배포 이력 조회
gh api repos/owner/repo/deployments \
  --limit 10
```

**상태 값:**
- `pending`: 배포 대기 중
- `in_progress`: 배포 진행 중
- `success`: 배포 성공
- `failure`: 배포 실패
- `inactive`: 이전 배포 (현재 활성 아님)

### GitHub Environment과 배포 이력의 연결

```yaml
jobs:
  deploy:
    environment: production  # 이 job이 production 환경을 사용
    steps:
      - run: kubectl apply -f app.yaml
```

GitHub Repository → Settings → Environments → production에서 다음을 설정 가능:
- **배포 보호 규칙**: 특정 브랜치에서만 배포 허용
- **필수 검수자**: 배포 전 승인자 지정 (반자동 배포)
- **환경 Secret**: 이 환경 전용 Slack Webhook, 데이터베이스 비밀번호 등

배포 후 Repository → "Deployments" 탭에서 배포 이력이 자동으로 표시된다.

### Datadog/Grafana 배포 마커(Annotation) 추가

배포 후 메트릭 그래프에 수직선(마커)을 추가해 배포 시점을 시각화한다.

**Datadog Annotation 추가:**
```bash
# 배포 완료 후
curl -X POST "https://api.datadoghq.com/api/v1/events" \
  -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "title": "Deployment to production",
  "text": "Commit: ${{ github.sha }}",
  "priority": "normal",
  "tags": [
    "environment:production",
    "service:api"
  ],
  "alert_type": "info"
}
EOF
```

**Grafana Annotation 추가:**
```bash
curl -X POST "http://grafana.example.com/api/annotations" \
  -H "Authorization: Bearer ${{ secrets.GRAFANA_API_TOKEN }}" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "text": "Deployed commit ${{ github.sha }}",
  "tags": ["deployment", "production"],
  "time": $(date +%s)000
}
EOF
```

## 💻 실전 실험 (GitHub Actions YAML, CLI 명령어로 재현 가능)

### 실험 1: Slack Webhook 테스트

```bash
# Slack Webhook URL 로컬 테스트
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# 간단한 메시지 전송
curl -X POST "$SLACK_WEBHOOK" \
  -H 'Content-type: application/json' \
  -d '{
    "text": "Test message from CLI"
  }'

# 블록 형식의 구조화된 메시지
curl -X POST "$SLACK_WEBHOOK" \
  -H 'Content-type: application/json' \
  -d '{
    "blocks": [
      {
        "type": "header",
        "text": {
          "type": "plain_text",
          "text": ":rocket: 배포 시작"
        }
      },
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": "*Status:* In Progress\n*Commit:* abc123def456"
        }
      }
    ]
  }'
```

### 실험 2: GitHub Deployment API 호출

```bash
# 배포 생성
DEPLOYMENT_ID=$(gh api repos/owner/repo/deployments \
  -f ref="main" \
  -f environment="production" \
  -f description="Test deployment" \
  --jq '.id')

echo "Created deployment: $DEPLOYMENT_ID"

# 배포 성공 상태로 업데이트
gh api repos/owner/repo/deployments/$DEPLOYMENT_ID/statuses \
  -f state="success" \
  -f description="Deployment successful"

# 배포 이력 조회
gh api repos/owner/repo/deployments \
  --jq '.[] | "\(.id): \(.environment) - \(.created_at)"'
```

### 실험 3: 배포 실패 시뮬레이션

```yaml
# .github/workflows/test-failure-notification.yml
name: Test Deployment Failure

on: workflow_dispatch

jobs:
  deploy_and_fail:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Simulate deployment failure
        id: deploy
        run: |
          echo "Starting deployment..."
          sleep 2
          echo "Simulating error..."
          exit 1
      
      - name: Send failure notification
        if: failure()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-type: application/json' \
            -d @- <<'EOF'
          {
            "blocks": [
              {
                "type": "header",
                "text": {
                  "type": "plain_text",
                  "text": ":fire: 배포 실패!"
                }
              },
              {
                "type": "section",
                "fields": [
                  {
                    "type": "mrkdwn",
                    "text": "*Commit*: ${{ github.sha }}"
                  },
                  {
                    "type": "mrkdwn",
                    "text": "*Branch*: ${{ github.ref }}"
                  },
                  {
                    "type": "mrkdwn",
                    "text": "*Run*: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"
                  }
                ]
              }
            ]
          }
          EOF
```

### 실험 4: 환경별 배포 알림

```yaml
# .github/workflows/multi-env-deploy.yml
name: Deploy to Multiple Environments

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [staging, production]
      max-parallel: 1
    environment: ${{ matrix.environment }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        run: |
          echo "Deploying to ${{ matrix.environment }}"
          # 실제 배포 명령어
      
      - name: Post deployment notification
        if: always()
        run: |
          STATUS_ICON="✅"
          COLOR="65280"
          
          if [ "${{ job.status }}" != "success" ]; then
            STATUS_ICON="❌"
            COLOR="16711680"
          fi
          
          curl -X POST "${{ secrets.DISCORD_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d "{
              \"embeds\": [{
                \"title\": \"$STATUS_ICON ${{ matrix.environment }} 배포\",
                \"color\": $COLOR,
                \"fields\": [
                  {\"name\": \"Commit\", \"value\": \"${{ github.sha }}\"},
                  {\"name\": \"Author\", \"value\": \"${{ github.actor }}\"}
                ]
              }]
            }"
```

## 📊 성능/비용 비교

| 항목 | Slack Webhook | Discord Webhook | Datadog Events |
|------|---------------|-----------------|-----------------|
| **설정 난도** | 쉬움 | 매우 쉬움 | 중간 |
| **메시지 포맷** | 풍부한 형식 지원 | 풍부한 형식 지원 | 간단함 |
| **비용** | 무료 | 무료 | 유료 (이벤트당) |
| **메트릭 연동** | 불가 | 불가 | 가능 |
| **팀 알림** | 우수 | 우수 | 보통 |
| **이력 관리** | 불가 | 불가 | 우수 |
| **응답 시간** | <1초 | <1초 | <2초 |

**GitHub Deployment API의 이점:**
- 추가 비용 없음
- GitHub UI에서 배포 이력을 체계적으로 관리
- 배포 보호 규칙과 승인 워크플로우 연동
- REST/GraphQL API로 프로그래밍 가능

## ⚖️ 트레이드오프

### Slack vs Discord

**Slack이 나은 이유:**
- 기업 환경에서 이미 배포된 경우 많음
- 스레드 기능으로 배포 관련 논의 중앙화
- 더 많은 통합 앱 생태계

**Discord가 나은 이유:**
- 완전 무료
- 서버 구조로 채널 관리가 직관적
- 개발팀 문화에 어울림

### Webhook vs GitHub Actions App

**Webhook (현재 방식):**
- 장점: 간단하고 빠름, 외부 서비스 의존성 낮음
- 단점: 봇이 느껴짐, 커스터마이징 어려움

**GitHub Actions App:**
- 장점: 공식 인증, 더 안전함
- 단점: 설정 복잡, 권한 관리 번거로움

### Annotation 기록 전략

**매번 기록 (더 정확):**
```bash
# 모든 배포마다 annotation 추가
gh api repos/owner/repo/deployments/$ID/statuses -f state=success
curl -X POST "datadog api" -d annotation
```
- 장점: 정확한 배포 시점 추적
- 비용: 이벤트 API 호출 증가

**주기적 기록 (비용 절감):**
```bash
# 1시간마다 배포된 커밋을 annotation에 추가
# 스케줄된 job에서 처리
```
- 장점: API 호출 최소화
- 단점: 정확도 감소

## 📌 핵심 정리

1. **배포 알림은 의무**: Slack, Discord, 이메일 중 하나는 반드시 구성해서 팀이 배포 사실을 알아야 한다.

2. **GitHub Deployment API로 이력 관리**: 배포 이력을 GitHub에 기록하면 롤백 판단과 사후 분석에 도움된다.

3. **메트릭 변화와 배포를 연결**: Datadog/Grafana Annotation으로 "언제 뭐가 바뀌었는가"를 증명할 수 있다.

4. **환경별로 다른 설정**: Production은 승인이 필요하지만, Staging은 자동배포할 수 있다.

5. **실패 알림이 가장 중요**: 배포 성공 메시지는 노이즈일 수 있지만, 실패 알림은 긴급하다.

## 🤔 생각해볼 문제

### Q1: 배포가 성공했다고 Slack 메시지가 왔는데, 실제로 요청을 받아들이고 있지 않아요. 어떤 검사를 추가해야 할까요?

<details>
<summary>해설</summary>

배포 성공 ≠ 서비스 정상 작동. 다음 검사를 추가하세요:

```yaml
- name: Health check after deployment
  run: |
    RETRIES=5
    for i in $(seq 1 $RETRIES); do
      if curl -f http://api.example.com/health; then
        echo "Health check passed"
        exit 0
      fi
      echo "Attempt $i failed, retrying..."
      sleep 5
    done
    exit 1

- name: Notify with health status
  if: always()
  run: |
    HEALTH_STATUS="건강"
    if [ $? -ne 0 ]; then
      HEALTH_STATUS="문제 있음"
    fi
    
    # Slack으로 health check 결과도 함께 알림
```

배포 후 health check가 실패하면 자동 롤백을 트리거할 수도 있다.
</details>

### Q2: 배포를 여러 단계로 나눠서 알림을 보내려면?

<details>
<summary>해설</summary>

배포 파이프라인을 단계별로 나누고, 각 단계마다 알림을 보내세요:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Stage 1 - Build and test
        run: npm run build && npm test
      
      - name: Notify stage 1 complete
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -d '{"text": ":building_construction: Stage 1: Build & Test 완료"}'
      
      - name: Stage 2 - Push to registry
        run: docker push myapp:latest
      
      - name: Notify stage 2 complete
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -d '{"text": ":whale: Stage 2: Docker Push 완료"}'
      
      - name: Stage 3 - Deploy to K8s
        run: kubectl apply -f deploy.yaml
      
      - name: Notify stage 3 complete
        if: success()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -d '{"text": ":rocket: Stage 3: K8s 배포 완료"}'
```

이렇게 하면 배포 진행 상황을 실시간으로 추적할 수 있다.
</details>

### Q3: 배포 알림에 "롤백" 버튼을 추가해서 한 번의 클릭으로 롤백하려면?

<details>
<summary>해설</summary>

Slack의 "interactive message" 기능을 사용해 버튼을 추가할 수 있지만, 실제 롤백 로직은 별도 엔드포인트가 필요합니다:

```yaml
# .github/workflows/slack-rollback-handler.yml
name: Handle Rollback Request

on:
  repository_dispatch:
    types: [rollback-request]

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Rollback to previous version
        run: |
          # 이전 배포 버전 가져오기
          PREVIOUS_IMAGE=$(gh api repos/${{ github.repository }}/deployments \
            --jq '.[1].creator.name' | head -1)
          
          kubectl set image deployment/api api=$PREVIOUS_IMAGE \
            -n production
      
      - name: Notify rollback success
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -d '{"text": ":rewind: 롤백 완료"}'
```

Slack 메시지에는:
```json
{
  "type": "button",
  "text": {"type": "plain_text", "text": "Rollback"},
  "action_id": "rollback_action"
}
```

하지만 Slack 버튼은 서명 검증이 필요하므로, 실제로는 외부 웹훅 서버가 필요하다. 더 간단하게는 수동으로 CLI 명령어를 실행하는 것이 낫다.
</details>

---

[⬅️ 이전: Chapter 6 — 보안 스캐닝 자동화](../test-automation/05-security-scanning.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./02-pipeline-failure-diagnosis.md)
