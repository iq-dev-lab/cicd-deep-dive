# Secrets와 환경 변수 — OIDC로 자격증명 없이 인증

## 🎯 핵심 질문

Repository Secret과 Environment Secret의 범위 차이는? Secrets가 로그에 마스킹되는 원리는 정말 안전할까? OIDC란 무엇이고, AWS/GCP에 장기 자격증명을 저장하지 않으면서 인증할 수 있는 이유는? Environment의 보호 규칙(Required reviewers, Wait timer)은 어떻게 작동할까?

## 🔍 왜 이 개념이 실무에서 중요한가

- **보안**: 자격증명 노출은 가장 심각한 보안 사건이다. Secret 마스킹이 어떻게 작동하는지 알아야 우회할 수 없다.
- **규정 준수**: 금융, 의료, 퍼블릭 클라우드 산업에서는 OIDC 없이 배포가 거의 불가능하다.
- **비용**: 장기 자격증명은 로테이션이 필수지만 OIDC는 자동 로테이션된다.
- **감사 추적**: OIDC를 사용하면 누가, 어떤 Action에서 배포했는지 추적 가능하다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 예 1: Secret을 직접 로그 출력
name: Unsafe Secret Usage
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "API_KEY: $API_KEY"  # 로그에 * 마스킹되나?
          curl -H "Authorization: $API_KEY" https://api.example.com

      - name: Even worse
        run: echo "${{ secrets.API_KEY }}"
        # 직접 표현식으로 전달하면 마스킹 보장 안 됨

# ❌ 잘못된 예 2: Repository Secret을 모든 Branch에서 사용
name: Repository Secret Scope Issue
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Push to production
        run: |
          # Repository Secret은 fork에서도 접근 가능 (보안 위험)
          curl -X POST \
            -H "Authorization: token ${{ secrets.PROD_TOKEN }}" \
            https://api.example.com/deploy

# ❌ 잘못된 예 3: 장기 자격증명 저장
name: Long-lived Credentials
on: [push]
jobs:
  deploy-to-aws:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_KEY }}
        # 수동으로 로테이션해야 함
        # 유출되면 유효 기간 동안 위험
        run: aws s3 cp file.txt s3://bucket/

# ❌ 잘못된 예 4: Environment 보호 규칙 무시
name: No Environment Protection
on: [push]
jobs:
  deploy-to-prod:
    environment: production
    # 보호 규칙이 없으면 누구든 배포 가능
    # (branch protection과 다름)
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 예 1: Secret을 환경변수로만 전달
name: Safe Secret Usage
on: [push]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          # 환경변수 이름을 echo하는 것은 안전
          echo "Using API_KEY environment variable"
          # 값을 echo하지 않으면 마스킹 보장
          curl -H "Authorization: $API_KEY" https://api.example.com

      - name: Verify secret handling
        env:
          TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # GitHub Token으로 API 호출 (로그에는 *** 마스킹)
          curl -H "Authorization: Bearer $TOKEN" \
               -H "Accept: application/vnd.github.v3+json" \
               https://api.github.com/user

# ✅ 올바른 예 2: Environment를 사용해 배포 환경 분리
name: Environment-based Deployment
on: [push]
jobs:
  deploy-to-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          STAGING_TOKEN: ${{ secrets.STAGING_TOKEN }}
        run: ./deploy-to-staging.sh
        # Environment Secret: staging에만 존재

  deploy-to-prod:
    environment: 
      name: production
      url: https://prod.example.com
    needs: [deploy-to-staging]  # 스테이징 후 프로덕션
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        env:
          PROD_TOKEN: ${{ secrets.PROD_TOKEN }}
        run: ./deploy-to-prod.sh
        # Environment Secret: production에만 존재
        # 보호 규칙: 1명 이상 승인 필수 (자동 요청)

# ✅ 올바른 예 3: OIDC로 AWS 자격증명 없이 인증
name: Deploy with OIDC
on: [push]
jobs:
  deploy-to-aws:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # OIDC 토큰 발급 권한
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials with OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: us-east-1
          # 장기 자격증명 없음 (Secret에 저장할 것 없음)
          # OIDC 토큰 자동 발급 및 임시 자격증명 취득

      - name: Deploy to S3
        run: aws s3 sync . s3://my-bucket/ --delete
        # 임시 자격증명으로 실행 (유효기간 1시간)

# ✅ 올바른 예 4: Environment 보호 규칙 설정
name: Protected Deployment
on: [push]
jobs:
  deploy:
    environment:
      name: production
      url: https://prod.example.com
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 이 step 실행 전 GitHub UI에서 배포 승인 대기
      # (자동으로 Required reviewers 체크, Wait timer 적용)
      
      - name: Deploy
        run: ./deploy.sh
```

## 🔬 내부 동작 원리

### 1. Secret의 범위와 접근 제어

**Repository Secret:**
```
저장 위치: 리포지토리 설정 → Secrets
접근 범위: 해당 리포지토리의 모든 Workflow
Fork: ❌ Fork에서는 접근 불가 (Pull Request에서도 불가)
보안 위험: 
  - 모든 Branch에서 접근 가능
  - 악성 코드가 PR에 포함되어도 Secret은 숨겨짐
  - 하지만 Fork PR에는 Secret이 전달되지 않음
```

**Organization Secret:**
```
저장 위치: Organization 설정 → Secrets
접근 범위: 선택된 리포지토리들
권한: Organization Owner만 관리
사용 사례: 모든 서비스가 공통으로 사용하는 자격증명
```

**Environment Secret:**
```
저장 위치: 리포지토리 → Environments → [env-name] → Secrets
접근 범위: 특정 Environment를 지정한 Job에서만
보호 규칙: Required reviewers, Wait timer, Deployment branches
사용 사례: 프로덕션과 스테이징 자격증명 분리
```

**예시:**
```yaml
# Repository Secret: COMMON_TOKEN (모든 Job 접근 가능)
jobs:
  any-job:
    steps:
      - run: echo ${{ secrets.COMMON_TOKEN }}  # ✅ 접근 가능

# Environment Secret: PROD_TOKEN (production Environment만)
jobs:
  prod-job:
    environment: production
    steps:
      - run: echo ${{ secrets.PROD_TOKEN }}  # ✅ 접근 가능

  staging-job:
    environment: staging
    steps:
      - run: echo ${{ secrets.PROD_TOKEN }}  # ❌ 접근 불가 (undefined)
```

### 2. Secret 마스킹 메커니즘

**마스킹 작동 원리:**
```
1. GitHub 서버: Secret 값을 Runner로 전송
2. Runner 에이전트: 모든 로그 라인을 메모리에 버퍼링
3. Secret 값 등록: Runner는 Secret 이름과 실제 값을 메모리에 기록
   - 환경변수 전달 시: API_KEY=${{ secrets.API_KEY }}
   - Runner가 실제 값(예: "abc123xyz")을 마스킹 리스트에 추가
4. 로그 마스킹: 모든 출력에서 마스킹 리스트의 값을 "***"로 교체
5. 로그 업로드: GitHub으로 전송
```

**마스킹이 작동하는 경우:**
```yaml
jobs:
  safe:
    steps:
      - env:
          TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          curl -H "Auth: $TOKEN" https://api.example.com
          # 로그에: curl -H "Auth: ***" https://api.example.com

      - run: echo "Using secrets safely"
        # 실제 값을 echo하지 않으므로 마스킹 안 필요
```

**마스킹이 작동하지 않는 경우:**
```yaml
jobs:
  unsafe:
    steps:
      # ❌ 방법 1: 직접 표현식으로 echo
      - run: echo "${{ secrets.API_TOKEN }}"
        # 표현식이 run 명령에 전달되기 전에 평가됨
        # Runner가 실제 값을 미리 알 수 없어 마스킹 불가능

      # ❌ 방법 2: 파라미터로 전달
      - run: npm test --api-key "${{ secrets.API_TOKEN }}"
        # 커맨드 라인 인자로 전달되므로 프로세스 환경에서 노출 가능

      # ❌ 방법 3: Base64 인코딩했으나 결과가 로그에
      - run: |
          ENCODED=$(echo "${{ secrets.API_TOKEN }}" | base64)
          echo "Token: $ENCODED"
        # echo로 출력하면 마스킹 안 됨
```

**마스킹 우회의 가능성:**
```
이론적 우회:
  - Secret이 조건부로 처리될 경우 (정규표현식 매칭 실패)
  - Secret이 문자열 변환되어 다른 형태로 변환될 경우
  - 로그 스트리밍 중 경합(race condition) 발생

실제로는:
  - GitHub이 Runner 통신을 완전히 제어
  - 마스킹은 매우 신뢰할 수 있지만, echo 피하기가 기본
```

### 3. OIDC(OpenID Connect) 인증 흐름

**OIDC 없이 (장기 자격증명):**
```
┌─────────────────┐
│ GitHub Actions  │
└────────┬────────┘
         │ 1. Secret 읽기
         ├─► AWS_ACCESS_KEY_ID=AKIA...
         └─► AWS_SECRET_ACCESS_KEY=+K4...
                 │
                 │ 2. AWS SDK로 로그인
                 ▼
         ┌──────────────────┐
         │ AWS (API calls)  │
         │ - S3 업로드      │
         │ - EC2 제어       │
         └──────────────────┘

문제:
  - 자격증명은 영구적 (로테이션 어려움)
  - 유출되면 유효기간 동안 악용 가능
  - 누가 배포했는지 감사 추적 어려움
```

**OIDC 사용 (임시 자격증명):**
```
┌──────────────────────────┐
│ GitHub Actions Runner    │
└────────┬─────────────────┘
         │ 1. OIDC 요청
         │    scope: id-token
         ▼
┌──────────────────────────┐
│ GitHub (OIDC 서버)        │
│ 토큰 생성                  │
│ {                         │
│   "iss": "https://...",   │
│   "sub": "repo:owner/...",│
│   "aud": "sts.amazonaws...",
│   "actor": "octocat",     │
│   "ref": "refs/heads/main"│
│ }                         │
└────────┬─────────────────┘
         │ 2. OIDC 토큰 반환
         ▼
┌──────────────────────────┐
│ GitHub Actions Runner    │
│ - OIDC 토큰 받음          │
└────────┬─────────────────┘
         │ 3. AWS STS AssumeRoleWithWebIdentity
         │    - OIDC 토큰 전송
         │    - 역할(Role) 지정
         ▼
┌──────────────────────────┐
│ AWS STS                  │
│ - OIDC 토큰 검증         │
│ - 신뢰 관계 확인         │
│ - 임시 자격증명 발급     │
│ {                        │
│   "AccessKeyId": "...",  │
│   "SecretAccessKey": "", │
│   "SessionToken": "..."  │
│   "Expiration": "1h"     │
│ }                        │
└────────┬─────────────────┘
         │ 4. 임시 자격증명 반환
         ▼
┌──────────────────────────┐
│ GitHub Actions Runner    │
│ - 환경변수로 임시 자격증명│
└────────┬─────────────────┘
         │ 5. AWS API 호출
         │    (임시 자격증명 사용)
         ▼
┌──────────────────────────┐
│ AWS (API calls)          │
│ - 이 토큰의 Action 로깅  │
│ - "assum-role-session"   │
└──────────────────────────┘
```

**OIDC 토큰 내용 (Claims):**
```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:owner/repo:ref:refs/heads/main",
  "aud": "sts.amazonaws.com",
  "actor": "octocat",
  "repository": "owner/repo",
  "repository_owner": "owner",
  "run_id": "123456789",
  "run_number": "42",
  "run_attempt": "1",
  "workflow": "CI",
  "ref": "refs/heads/main",
  "ref_type": "branch",
  "job_workflow_label": "deploy",
  "exp": 1234567890,
  "iat": 1234567800
}
```

### 4. Environment 보호 규칙

**Required Reviewers:**
```
설정: Environment → Deployment branches and reviewers → Required reviewers
동작 흐름:
  1. Job이 Environment를 사용하는 step 도달
  2. GitHub이 Job 일시 중지
  3. 지정된 Reviewers에게 알림
  4. Reviewer가 GitHub UI에서 승인 클릭
  5. Job 재개

권한: Repository Owner, Collaborator
```

**Wait Timer:**
```
설정: Environment → Wait timer (분 단위)
동작:
  1. Job이 Environment 사용하는 step 도달
  2. GitHub이 Job 일시 중지
  3. 설정된 시간만큼 대기 (예: 5분)
  4. 자동으로 Job 재개

용도: 배포 전 최소 대기 시간 (운영팀이 준비할 수 있도록)
```

**Deployment Branches:**
```
설정: Environment → Deployment branches
옵션:
  - All branches
  - Protected branches only
  - Custom deployment branches (정규표현식)

동작: main, release/* 브랜치에서만 배포 허용
```

## 💻 실전 실험 (GitHub Actions YAML 재현 가능한 예시)

### 실험 1: Secret 마스킹 테스트

```yaml
name: Secret Masking Test
on: [push]

jobs:
  masking-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Safe approach (마스킹 보장)
        env:
          API_TOKEN: ${{ secrets.DEMO_TOKEN }}
        run: |
          echo "Contacting API with token..."
          echo "Token length: ${#API_TOKEN}"
          # 로그: "Token length: 15" (값 안 나옴)

      - name: Unsafe approach (마스킹 미보장)
        run: echo "Token: ${{ secrets.DEMO_TOKEN }}"
        # 이론상 "Token: ***" 출력되나
        # 실제로는 마스킹 우회 가능성

      - name: Check masking in conditionals
        env:
          TOKEN: ${{ secrets.DEMO_TOKEN }}
        run: |
          if [[ "$TOKEN" == "expected-value" ]]; then
            echo "Match"  # 매칭되면 마스킹될 가능성
          fi

      - name: Check process list
        env:
          SECRET: ${{ secrets.DEMO_TOKEN }}
        run: |
          # ps 명령으로 환경변수 확인 (마스킹 안 됨)
          ps aux | grep -v grep
```

### 실험 2: Environment Secret 범위

**Repository 설정에서 생성:**
- Repository Secret: `COMMON_KEY`
- Environment: `staging` → Secret: `STAGING_KEY`
- Environment: `production` → Secret: `PROD_KEY`

```yaml
name: Environment Secret Scope
on: [push]

jobs:
  test-common:
    runs-on: ubuntu-latest
    steps:
      - run: echo "COMMON_KEY available: ${{ secrets.COMMON_KEY != '' }}"

  test-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "COMMON_KEY available: ${{ secrets.COMMON_KEY != '' }}"
          echo "STAGING_KEY available: ${{ secrets.STAGING_KEY != '' }}"
          echo "PROD_KEY available: ${{ secrets.PROD_KEY != '' }}"

  test-prod:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "COMMON_KEY available: ${{ secrets.COMMON_KEY != '' }}"
          echo "STAGING_KEY available: ${{ secrets.STAGING_KEY != '' }}"
          echo "PROD_KEY available: ${{ secrets.PROD_KEY != '' }}"

# 결과:
# test-common: COMMON_KEY=true
# test-staging: COMMON_KEY=true, STAGING_KEY=true, PROD_KEY=false
# test-prod: COMMON_KEY=true, STAGING_KEY=false, PROD_KEY=true
```

### 실험 3: OIDC로 AWS 배포

**AWS 설정:**
```bash
# 1. GitHub OIDC 제공자 추가
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 1b511abead59c6cea8b076bac41f1db4d6d8e136

# 2. Role 생성
aws iam create-role \
  --role-name github-actions-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:owner/repo:ref:refs/heads/main"
        }
      }
    }]
  }'

# 3. Policy 추가
aws iam put-role-policy \
  --role-name github-actions-role \
  --policy-name s3-deploy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }]
  }'
```

**.github/workflows/deploy.yml:**
```yaml
name: Deploy with OIDC
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # OIDC 토큰 요청 권한
      contents: read
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials with OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: us-east-1

      - name: Deploy to S3
        run: |
          aws s3 cp index.html s3://my-bucket/
          # AWS CLI가 자동으로 OIDC 토큰 기반 임시 자격증명 사용

      - name: Verify credentials are temporary
        run: aws sts get-caller-identity
        # 출력: AssumedRoleUser ARN에 임시 세션 ID 포함
```

### 실험 4: Environment 보호 규칙

**.github/workflows/protected-deploy.yml:**
```yaml
name: Protected Deployment
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-url: s3://bucket/app-latest.tar.gz
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - run: npm test

  deploy-staging:
    needs: [build]
    environment:
      name: staging
      url: https://staging.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: echo "Deploying to staging"

  deploy-production:
    needs: [build, deploy-staging]
    environment:
      name: production
      url: https://prod.example.com
      # Environment 설정에서:
      # - Required reviewers: 1명 (배포 시 승인 필요)
      # - Wait timer: 5분 (스테이징 배포 후 5분 대기)
      # - Deployment branches: main만 (다른 브랜치에서는 배포 불가)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Production deployment
        env:
          PROD_TOKEN: ${{ secrets.PROD_TOKEN }}  # production Environment Secret
        run: |
          # 이 step 도달 전에 GitHub UI에서 승인 대기
          # 승인되고 5분 경과 후 실행
          echo "Deploying to production"
          ./deploy-prod.sh

      - name: Notify deployment
        if: success()
        run: echo "Production deployment completed"
```

## 📊 성능/비용 비교

| 방식 | 보안 | 편의성 | 비용 | 감사 추적 |
|------|------|--------|------|---------|
| **장기 자격증명** | ❌ 낮음 | ✅ 높음 | 낮음 | ❌ 어려움 |
| **OIDC** | ✅ 높음 | ⚠️ 중간 | 낮음 | ✅ 우수 |
| **Environment Secret** | ⚠️ 중간 | ✅ 높음 | 낮음 | ⚠️ 중간 |
| **Environment + OIDC** | ✅ 높음 | ⚠️ 중간 | 낮음 | ✅ 우수 |

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 |
|------|------|------|
| **Repository Secret** | 간단함, 빠름 | 모든 Branch에서 접근 가능 |
| **Environment Secret** | 환경별 분리, 보호 규칙 | 각 Environment마다 설정 필요 |
| **OIDC** | 안전, 자동 로테이션, 감사 추적 | 클라우드 설정 복잡 |
| **Required Reviewers** | 이중 검증 | 배포 지연, 인력 필요 |
| **Wait Timer** | 변경 파악 시간 제공 | 배포 지연 |

## 📌 핵심 정리

1. **Secret 범위**: Repository (전체) > Environment (특정 환경) > Organization (선택 리포지토리)
2. **마스킹**: 환경변수로 전달 시 보장 → 직접 echo는 피하기
3. **OIDC**: 장기 자격증명 불필요 → 임시 자격증명 + 자동 로테이션
4. **Environment**: 보호 규칙(승인, 대기시간)으로 안전한 배포 보장
5. **감사 추적**: OIDC 사용 시 누가, 어디서 배포했는지 추적 가능

## 🤔 생각해볼 문제

### 문제 1: 이 워크플로우에서 문제점은?

```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        env:
          SECRET: ${{ secrets.PROD_SECRET }}
        run: ./deploy.sh
```

GitHub이 Job 일시 중지했는데, 10분 후 승인되지 않았다. 그러면?

<details>
<summary>해설</summary>

**문제:** Environment 보호 규칙에서 "Wait timer" 또는 "Required reviewers"가 설정된 경우:

1. Job이 환경변수 할당 step에 도달
2. GitHub이 Job 일시 중지 (대기 시작)
3. 설정된 조건 대기 (승인 또는 시간)

**10분 후:**
- **Timeout 없음** → 무한정 대기 (실수)
- GitHub이 자동 취소하지 않음
- 사용자가 수동으로 취소하거나 승인해야 함

**문제 해결:**
```yaml
jobs:
  deploy:
    environment: production
    timeout-minutes: 30  # 전체 Job 타임아웃
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        timeout-minutes: 10  # 이 step만 타임아웃
        env:
          SECRET: ${{ secrets.PROD_SECRET }}
        run: ./deploy.sh
```

**주의:** timeout-minutes는 실제 실행 시간이지, "승인 대기 시간"이 아님
</details>

### 문제 2: OIDC로 AWS 역할을 정의할 때, Subject (sub) 조건이 없으면?

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/..."
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        // sub 조건 없음!
      }
    }
  }]
}
```

<details>
<summary>해설</summary>

**문제:** `sub` 조건이 없으면 **모든 GitHub repository의 모든 workflow가 이 역할을 Assume 가능**

**공격 시나리오:**
1. 악의적 행위자가 어떤 퍼블릭 GitHub 계정에서 workflow 생성
2. 자신의 `OIDC` 토큰으로 AWS STS AssumeRoleWithWebIdentity 호출
3. sub 조건이 없으므로 토큰 검증 통과
4. 임시 자격증명 획득
5. 프로덕션 환경 제어 가능

**권장 설정:**
```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:owner/repo:*"
    }
  }
}
```

또는 더 제한적으로:
```json
{
  "StringEquals": {
    "token.actions.githubusercontent.com:sub": "repo:owner/repo:ref:refs/heads/main"
  }
}
```

**결론:** OIDC는 안전하지만 **Trust Policy 설정이 생명**
</details>

### 문제 3: Secret이 파이프된 데이터에 포함되면 마스킹될까?

```yaml
steps:
  - name: Secret in piped data
    env:
      TOKEN: ${{ secrets.API_TOKEN }}
    run: |
      echo "Processing request"
      echo "Bearer $TOKEN" | some-command
      # TOKEN이 파이프 입력으로 전달되면?

  - name: Secret in command substitution
    env:
      TOKEN: ${{ secrets.API_TOKEN }}
    run: |
      RESPONSE=$(curl -H "Auth: $TOKEN" https://api.example.com)
      echo "Status: $RESPONSE"
      # RESPONSE에 TOKEN이 포함되면?
```

<details>
<summary>해설</summary>

**마스킹 동작:**

1. **파이프 입력:**
   ```bash
   echo "Bearer $TOKEN" | some-command
   ```
   - `$TOKEN` 값이 some-command의 stdin으로 전달
   - 로그에는 표시되지 않음 (파이프는 로그 대상 아님)
   - ✅ 안전

2. **Command Substitution:**
   ```bash
   RESPONSE=$(curl -H "Auth: $TOKEN" https://api.example.com)
   echo "Status: $RESPONSE"
   ```
   - curl이 전송할 때: `$TOKEN` 값 사용 (로그 안 남음)
   - RESPONSE 변수: API 응답 포함 (TOKEN 없음, 보통은)
   - echo의 로그: RESPONSE 출력 (TOKEN 불포함, 일반적)
   - ✅ 대부분 안전

**위험한 경우:**
```bash
curl -H "Auth: $TOKEN" https://api.example.com | tee response.txt
```
- `tee`가 curl의 출력을 파일과 stdout으로 복사
- curl 에러 응답에 TOKEN 포함되면 위험

**권장:**
```bash
env:
  TOKEN: ${{ secrets.API_TOKEN }}
run: |
  curl -s -H "Authorization: $TOKEN" https://api.example.com \
    -o response.json
  # response.json에만 결과, 로그는 깨끗
```
</details>

---

<div align="center">
**[⬅️ 이전: Job 의존성과 병렬화](./03-job-dependency-parallelism.md)** | **[홈으로 🏠](../README.md)** | **[다음: 캐시 전략 ➡️](./05-cache-strategy.md)**
</div>
