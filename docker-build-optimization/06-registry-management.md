# 레지스트리 관리 — 태그 전략과 이미지 정리

## 🎯 핵심 질문

- `latest` 태그가 왜 위험할까? 같은 태그가 다른 이미지를 가리키는 상황은?
- Docker Hub, GHCR, ECR의 차이점은? 어떤 기준으로 선택해야 할까?
- 수백 개의 이미지 버전을 자동으로 관리할 수 있을까?

## 🔍 왜 이 개념이 실무에서 중요한가

**이미지 태그 관리의 심각한 영향:**

1. **`latest` 태그의 위험성**
   - 개발자 A: `docker pull myapp:latest` → v1.2.3 받음
   - 개발자 B: `docker pull myapp:latest` → v1.2.4 받음 (같은 시간)
   - CI: `docker pull myapp:latest` → v1.2.5 받음 (다른 환경)
   - **결과: 재현 불가능 (다른 버전들이 섞여 있음)**

2. **레지스트리 스토리지 폭증**
   - 매일 빌드 50회 × 30일 = 1,500개 이미지
   - 각 200MB = 300GB 저장소 (비용 폭증)
   - 자동 정리 없으면 월간 $300+ 비용

3. **컨플라이언스 위반**
   - GDPR: 이미지 메타데이터 추적 필수
   - SOC 2: 모든 배포 이력 기록 필수
   - PCI-DSS: 특정 이미지 버전만 허용

4. **배포 속도 저하**
   - 메타데이터 검색 (수천 태그) 필요
   - 불필요한 이미지 다운로드
   - 레지스트리 API 요청 초과로 rate limit 걸림

**실무 사례:**
- Google: 100만 개 이미지 관리, 자동 정리로 월간 $500K 절감
- Uber: Semantic Versioning + SHA 태그로 배포 안정성 99.99%

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

### 문제 1: `latest` 태그만 사용

```yaml
# ❌ 나쁜 GitHub Actions 설정
name: Build and Push
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: myapp:latest  # ❌ latest만!
          # 위험: 매번 같은 태그를 덮어씀
```

**문제점:**

```
Run 1 (2024-01-01 10:00):
docker pull myapp:latest
→ Image ID: sha256:abc123 (v1.2.3)

Run 2 (2024-01-01 10:05):
docker pull myapp:latest
→ Image ID: sha256:def456 (v1.2.4) ❌ 다른 이미지!

Run 3 (2024-01-01 10:10):
docker pull myapp:latest
→ Image ID: sha256:ghi789 (v1.2.5) ❌ 또 다른 이미지!

문제:
- 어떤 코드가 실제로 배포된 걸까?
- 버그가 발생했을 때 원인 찾을 수 없음
- 롤백해도 이전 버전 찾기 어려움
```

### 문제 2: 자동 정리 정책 없음

```bash
# 레지스트리 저장소 모니터링
$ docker-stats $(hostname) registry
STORAGE: 15TB
├─ 2024-01: 5TB (100일 전)
├─ 2024-02: 4TB (60일 전)
├─ 2024-03: 3TB (30일 전)
└─ 2024-04: 3TB (현재)

월간 비용 (AWS ECR): 15TB × $0.10/GB = $1,500/월!

관리되는 이미지:
$ aws ecr describe-repositories --query 'repositories[0].imageScanningConfiguration'

실제 사용 이미지: 50개
미사용 이미지: 4,950개 (99% 미사용!)

정리되지 않는 이미지들:
- 1년 전 버전 (시스템 업그레이드 후 미사용)
- 실패한 빌드 (테스트 실패했는데 푸시됨)
- 임시 태그 (PR 빌드, 개발용)
```

### 문제 3: Semantic Versioning 미흡

```dockerfile
# ❌ 나쁜 태그 전략
docker tag myapp:abc123 myapp:latest
docker tag myapp:abc123 myapp:1.2  # 부분 버전?
docker tag myapp:abc123 myapp:1    # 메이저 버전?

# 문제:
# - 개발자는 어떤 버전이 최신인지 알 수 없음
# - docker pull myapp:1.2 할 때 1.2.0? 1.2.1? 1.2.10?
# - 시맨틱 버저닝 규칙 미정의
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 방식 1: Git SHA 기반 태그 (재현성)

```yaml
# ✅ 좋은 GitHub Actions 설정: Git SHA 태그
name: Build and Push
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Git commit SHA 가져오기
      - id: meta
        run: |
          echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
      
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            myapp:${{ steps.meta.outputs.sha_short }}
            myapp:${{ github.ref_name }}-${{ steps.meta.outputs.sha_short }}
          # 결과:
          # myapp:abc1234 (Git commit SHA)
          # myapp:main-abc1234 (브랜치 + SHA)
```

**장점:**

```
배포:
$ docker pull myapp:abc1234
→ 정확히 이 commit의 코드로 빌드된 이미지

버전 추적:
$ docker images myapp
myapp   abc1234   sha256:abc...
myapp   def5678   sha256:def...
myapp   ghi9012   sha256:ghi...

특정 버전 롤백:
$ docker pull myapp:abc1234  # 정확히 그 버전!

CI 재현:
$ git log --oneline
abc1234 Fix bug in auth
def5678 Add feature X

$ docker run myapp:abc1234  # 그 당시 정확한 이미지
```

### 방식 2: Semantic Versioning (v1.2.3)

```yaml
# ✅ Semantic Versioning 태그 전략
name: Release

on:
  push:
    tags:
      - 'v*'  # v1.0.0, v1.2.3 등의 태그

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # 버전 추출: v1.2.3 → 1.2.3
      - id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - uses: docker/metadata-action@v4
        id: meta
        with:
          images: myapp
          tags: |
            type=semver,pattern={{version}},value=${{ steps.version.outputs.version }}
            type=semver,pattern={{major}}.{{minor}},value=${{ steps.version.outputs.version }}
            type=semver,pattern={{major}},value=${{ steps.version.outputs.version }}
            type=raw,value=latest
          # 결과 태그:
          # myapp:1.2.3  (정확한 버전)
          # myapp:1.2    (마이너 버전)
          # myapp:1      (메이저 버전)
          # myapp:latest (최신)
      
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

**태그 자동 생성 예:**

```
Git에 태그 생성: git tag v1.2.3 && git push origin v1.2.3

자동으로 생성되는 태그:
├─ myapp:1.2.3 (정확한 버전)
├─ myapp:1.2 (마이너 버전, 최신 1.2.x 가리킴)
├─ myapp:1 (메이저 버전, 최신 1.x.x 가리킴)
└─ myapp:latest (최신 전체)

사용 사례:
$ docker pull myapp:1.2.3  # 정확히 이 버전
$ docker pull myapp:1.2    # 최신 1.2.x (자동 업그레이드)
$ docker pull myapp:1      # 최신 1.x.x (자동 업그레이드)
$ docker pull myapp:latest # 최신 (권장하지 않음)
```

### 방식 3: 다중 태그 전략 (프로덕션)

```yaml
# ✅ 프로덕션 권장: Git SHA + Semantic Version + Branch
name: Build and Push
on: 
  push:
    branches: [main, develop]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: docker/metadata-action@v4
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            # Git SHA (항상)
            type=sha,prefix={{branch}}-
            
            # 브랜치 태그 (main, develop)
            type=ref,event=branch
            
            # Semantic Version (태그일 때)
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=semver,value=latest
      
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

**실제 생성되는 태그:**

```
Scenario 1: Push to main (commit abc1234)
ghcr.io/myuser/myapp:main-abc1234  (Git SHA)
ghcr.io/myuser/myapp:main          (브랜치)

Scenario 2: Push to develop
ghcr.io/myuser/myapp:develop-def5678  (Git SHA)
ghcr.io/myuser/myapp:develop          (브랜치)

Scenario 3: Push tag v1.2.3
ghcr.io/myuser/myapp:1.2.3         (정확한 버전)
ghcr.io/myuser/myapp:1.2           (마이너)
ghcr.io/myuser/myapp:1             (메이저)
ghcr.io/myuser/myapp:latest        (최신)
```

### 방식 4: 자동 이미지 정리 정책

**GitHub Container Registry (GHCR):**

```yaml
# .github/workflows/cleanup.yml
name: Cleanup Old Images
on:
  schedule:
    - cron: '0 2 * * *'  # 매일 02:00 UTC

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # GHCR 패키지 정리
      - uses: snok/container-retention-policy@v2
        with:
          image-names: |
            myapp
            myservice
          cut-off: A week ago UTC
          keep-latest: 5  # 최근 5개 버전 유지
          skip-tags: |
            latest
            v*      # semantic version 유지
            main    # main 브랜치 유지
          untagged-only: true  # 태그 없는 이미지만 삭제
```

**AWS ECR:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Delete untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["release"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

## 🔬 내부 동작 원리

### 1. **레지스트리 태그의 실제 구조**

```
Docker Registry (모든 레지스트리는 동일):

┌─────────────────────────────────────────────┐
│ Registry (docker.io, ghcr.io, ecr.aws.com)  │
├─────────────────────────────────────────────┤
│ Repository: myapp                           │
│  ├─ Tag: latest                             │
│  │   └─ Manifest: sha256:abc123...          │
│  │       └─ Image ID: sha256:abc123...      │
│  │                                          │
│  ├─ Tag: v1.2.3                             │
│  │   └─ Manifest: sha256:def456...          │
│  │       └─ Image ID: sha256:def456...      │
│  │                                          │
│  └─ Tag: main-xyz789                        │
│      └─ Manifest: sha256:xyz789...          │
│          └─ Image ID: sha256:xyz789...      │
└─────────────────────────────────────────────┘

Tag = 이름 (문자열)
Manifest = 이미지 메타데이터 (JSON)
Image ID = 실제 이미지 데이터
```

### 2. **Tag가 가리키는 이미지 변경**

```
시간대별 이미지 변화:

10:00 - docker push myapp:latest (commit abc1234)
Registry:
  latest → Image ID: sha256:abc123...

10:05 - docker push myapp:latest (commit def5678)
Registry:
  latest → Image ID: sha256:def456... (변경됨!)
  이전 이미지(abc123)는 태그 없이 떠있음

문제: 다른 개발자들의 로컬 이미지는?
$ docker images myapp
myapp   latest   sha256:abc123...  (1시간 전 pull, 여전히 old)

실제 레지스트리의 latest:
myapp   latest   sha256:def456...  (새로운 버전)

재현 불가능!
```

### 3. **Image Manifest와 Layer**

```
Docker Image 구조:

┌─────────────────────────────────────────┐
│ Image Manifest (v2)                     │
├─────────────────────────────────────────┤
│ {                                       │
│   "schemaVersion": 2,                   │
│   "mediaType": "...",                   │
│   "config": {                           │
│     "digest": "sha256:abc123..."        │
│   },                                    │
│   "layers": [                           │
│     {                                   │
│       "digest": "sha256:layer1..."      │
│     },                                  │
│     {                                   │
│       "digest": "sha256:layer2..."      │
│     }                                   │
│   ]                                     │
│ }                                       │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│ Image Config (JSON)                     │
│ └─ Entrypoint, Env, WorkingDir, etc.   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Layer 1 (sha256:layer1...)              │
│ └─ Base OS (Ubuntu 22.04)               │
├─────────────────────────────────────────┤
│ Layer 2 (sha256:layer2...)              │
│ └─ Application + Libraries              │
└─────────────────────────────────────────┘

Tag (latest):
  └─ Manifest Hash (고유한 ID)
     └─ Config + Layer 구성
```

### 4. **레지스트리별 특성**

```
Docker Hub:
- 저장소: hub.docker.com
- 인증: 사용자명/리포지토리
- 무료: public 이미지 무제한
- 유료: private 이미지 $5/월
- 다운로드: rate limit (IP당 6시간에 100회)

GitHub Container Registry (GHCR):
- 저장소: ghcr.io
- 인증: GitHub token
- 무료: public/private 무제한 (리포지토리 당 5GB)
- 다운로드: rate limit 없음 (로그인 필수)
- 장점: GitHub Actions와 통합 좋음

AWS ECR:
- 저장소: {account-id}.dkr.ecr.{region}.amazonaws.com
- 인증: IAM role (매우 안전)
- 가격: $0.10/GB/월 저장소 + 대역폭
- 다운로드: rate limit 없음
- 장점: VPC 내부 배포 빠름, KMS 암호화
```

## 💻 실전 실험

### 실험 1: `latest` 태그의 위험성

**Step 1: 여러 버전의 이미지 푸시**

```bash
# Version 1: ubuntu:22.04 기반
$ docker build -t myapp:latest -f Dockerfile.v1 .
$ docker push myapp:latest

# Version 2로 변경
$ docker build -t myapp:latest -f Dockerfile.v2 .
$ docker push myapp:latest

# Version 3로 변경
$ docker build -t myapp:latest -f Dockerfile.v3 .
$ docker push myapp:latest

# 실제 레지스트리:
# myapp:latest → Dockerfile.v3 기반 (현재)
```

**Step 2: 다른 머신에서 pull 시점의 차이**

```bash
# 머신 A (10:00에 pull)
$ docker pull myapp:latest
v1: Pulling from library/myapp
sha256:abc123: Pull complete

# 머신 B (10:05에 pull)
$ docker pull myapp:latest
v2: Pulling from library/myapp
sha256:def456: Pull complete  # ❌ 다른 이미지!

# 머신 C (10:10에 pull)
$ docker pull myapp:latest
v3: Pulling from library/myapp
sha256:ghi789: Pull complete  # ❌ 또 다른 이미지!

# 로컬에는 세 버전이 모두 남아있음
$ docker images myapp
myapp    latest   sha256:abc123...  (v1)
myapp    latest   sha256:def456...  (v2)
myapp    latest   sha256:ghi789...  (v3)

❌ latest 태그가 계속 덮어써짐
```

**Step 3: Dockerfile로 재현 불가능**

```dockerfile
# Dockerfile (latest 사용)
FROM myapp:latest  # 어느 버전?

RUN ./build.sh
```

```bash
$ docker build -t myapp .
# 머신 A: v1 기반 (이미지 ID: abc123)
# 머신 B: v2 기반 (이미지 ID: def456)
# 머신 C: v3 기반 (이미지 ID: ghi789)

# 빌드 결과가 머신마다 다름!
```

### 실험 2: Git SHA 태그 사용

```bash
# Commit SHA를 태그로 사용
$ git rev-parse --short HEAD
abc1234

$ docker build -t myapp:abc1234 .
$ docker push myapp:abc1234

# 레지스트리:
# myapp:abc1234 → 정확히 이 commit의 코드

# 재현 가능:
$ git checkout abc1234
$ docker pull myapp:abc1234  # 정확히 이 버전

# 자동으로 동일한 환경 재현!
```

### 실험 3: 자동 태그 생성

```bash
# docker/metadata-action 사용
$ cat .github/workflows/release.yml

# Tag 생성: git tag v1.2.3
# 자동으로 생성되는 태그 확인:

$ docker/metadata-action (시뮬레이션)
Input: version=1.2.3

Output Tags:
- myapp:1.2.3       (MAJOR.MINOR.PATCH)
- myapp:1.2         (MAJOR.MINOR)
- myapp:1           (MAJOR)
- myapp:latest      (현재 최신)

생성 검증:
$ docker images myapp
myapp   1.2.3   sha256:abc123
myapp   1.2     sha256:abc123  (같은 이미지)
myapp   1       sha256:abc123  (같은 이미지)
myapp   latest  sha256:abc123  (같은 이미지)
```

### 실험 4: 이미지 자동 정리

**Step 1: 정리 전 상태**

```bash
# GHCR에서 모든 이미지 확인
$ gh api repos/myuser/myapp/packages

name:     myapp
versions: 1500개 (6개월간 50회/일 빌드)
size:     200GB

태그 분포:
- main-abc1234, main-def5678, ... (PR/commit 별)
- untagged (태그 없는 이미지)
- v1.0.0, v1.1.0, v1.2.3, ... (릴리스)

무용지물 이미지: 1,450개 (97%)
```

**Step 2: cleanup policy 적용**

```yaml
keep-latest: 5
skip-tags: |
  latest
  v*
  main
untagged-only: true
```

```bash
# 정리 후
# 유지되는 이미지:
# - main-abc1234 ~ main-abc1238 (최근 5개)
# - latest, v1.0.0, v1.1.0, v1.2.3 (semantic)
# - develop, develop-xyz789, ... (main 브랜치)

# 제거된 이미지:
# - 태그 없는 이미지 1,400개 (자동 정리)
# - main-abc1000, main-abc1001, ... (5개 이상 old)

# 결과:
size: 200GB → 5GB (97.5% 감소!)
```

## 📊 성능/비용 비교

### 레지스트리 비용 비교 (월간)

```
프로젝트: 1일 50회 빌드, 6개월 운영

Docker Hub (Public):
- 저장소: 무료
- 대역폭: 일부 제한 (rate limit)
- 총 비용: 무료 (단, 신뢰성 낮음)

GitHub Container Registry (GHCR):
- 저장소: 무료 (5GB/리포지토리)
- 실제 사용: 200GB (5개 리포지토리)
- 초과분: 200GB = 자동 정리로 5GB 이내 유지
- 총 비용: 무료

AWS ECR:
- 저장소: 200GB × $0.10 = $20/월
- 자동 정리 후: 5GB × $0.10 = $0.50/월
- 대역폭: 약 $10/월 (데이터 다운로드)
- 총 비용: $10.50/월 (정리 전 $20+)

Private Docker Hub:
- 구독료: $5 × 5개 리포지토리 = $25/월
- 스토리지: 200GB × $0.10 = $20/월
- 총 비용: $45/월
```

### 배포 안정성 비교

```
이미지 태그 전략별 상황 분석:

❌ latest만 사용:
- 재현성: 0% (다른 환경에서 다른 버전)
- 추적성: 0% (어떤 버전인지 알 수 없음)
- 롤백: 불가능 (이전 버전 특정 불가)

✅ Git SHA + Semantic Version:
- 재현성: 100% (정확한 commit 추적)
- 추적성: 100% (언제 어떤 버전 배포된지 기록)
- 롤백: 가능 (정확한 버전 선택)
```

### 저장소 비용 절감

```
자동 정리 정책의 효과:

정리 전 (6개월):
- 일일 50회 빌드 × 180일 = 9,000개 이미지
- 평균 200MB = 1.8TB
- AWS ECR: 1.8TB × $0.10 = $180/월 × 6 = $1,080

정리 정책:
- 최근 5개 버전만 유지
- Semantic Version 유지 (v1.0.0, v1.1.0, ...)
- 브랜치 최신만 유지

정리 후 (6개월):
- 유지 이미지: 50개 (semantic) + 5개(각 브랜치) = 55개
- 평균 200MB = 11GB
- AWS ECR: 11GB × $0.10 = $1.10/월 × 6 = $6.60

절감: $1,080 - $6.60 = $1,073.40 (99.4% 감소!)
```

## ⚖️ 트레이드오프

### 1. **Git SHA vs Semantic Version**

```
Git SHA (abc1234):
장점:
- 정확한 commit 추적
- 완벽한 재현성
- 모든 빌드에 적용 가능

단점:
- 사용자 친화적이 아님 (정확한 버전 모름)
- 버전 의존성 명시 어려움

Semantic Version (v1.2.3):
장점:
- 사용자 친화적 (명확한 버전)
- 버전 의존성 명시 가능
- 호환성 예측 가능

단점:
- 수동 버전 관리 필요
- 태그 누락 가능성

권장: 둘 다 사용!
- Git SHA: 모든 빌드에 자동 적용
- Semantic Version: 릴리스 버전에만 수동 적용
```

### 2. **Public vs Private 레지스트리**

```
Public (Docker Hub):
장점:
- 무료
- 광범위한 네트워크 (CDN)
- 사용자 편의

단점:
- 보안 취약 (이미지 공개)
- Rate limit (제한적)
- 신뢰성 낮음

Private (ECR/GHCR):
장점:
- 보안 강함 (접근 제어)
- Rate limit 없음
- 신뢰성 높음

단점:
- 비용 ($5-50/월)
- 네트워크 설정 필요

권장: 프로덕션은 Private, 라이브러리는 Public
```

### 3. **자동 정리의 공격성**

```
보수적 정리:
- 최근 30개 버전 유지
- 6개월 이상 된 이미지 유지
- 저장소 크기: 크다
- 비용: 높음
- 장점: 예전 이미지 접근 가능

공격적 정리:
- 최근 5개 버전만 유지
- 7일 이상 된 untagged 이미지 삭제
- 저장소 크기: 작음
- 비용: 낮음
- 단점: 예전 버전 복구 어려움

권장: 보수적 + 자동 백업
- 자동 정리는 공격적
- 월간 스냅샷을 다른 저장소에 보관
- 긴급 복구 필요시 스냅샷에서 복원
```

## 📌 핵심 정리

1. **`latest` 태그는 재현성 파괴 (절대 프로덕션 사용 금지)**
   - 같은 태그가 시간에 따라 다른 이미지를 가리킴
   - 배포 재현 불가능

2. **Git SHA + Semantic Version 조합이 최고 (권장)**
   - Git SHA: 모든 빌드에 자동 적용 (재현성)
   - Semantic Version: 릴리스에만 적용 (사용 편의성)

3. **`docker/metadata-action` 자동 태그 생성 필수**
   - v1.2.3 태그 1개로 v1.2.3, v1.2, v1, latest 자동 생성
   - 휴먼 에러 방지

4. **자동 정리 정책으로 저장소 비용 99% 절감**
   - 최근 5-10개 버전만 유지
   - untagged 이미지 자동 삭제
   - 월간 $1,000+ 절감 가능

5. **GitHub Container Registry (GHCR) 권장**
   - GitHub Actions와 완벽 통합
   - Rate limit 없음
   - 저장소 5GB 무료 (자동 정리로 충분)

## 🤔 생각해볼 문제

### 문제 1: 다음 태그 전략의 문제점을 찾으세요.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            myapp:${{ github.sha }}
            myapp:latest
```

<details>
<summary>해설</summary>

**문제점:**
1. `github.sha` = 전체 40자리 SHA (너무 길다)
   - `myapp:3f48d1c7c4d3e2a1b9f8e7d6c5b4a39281f0e0d`
   - 사용자가 기억할 수 없음

2. `latest` 태그는 여전히 위험
   - 매번 덮어씀
   - 재현성 없음

**올바른 방식:**
```yaml
tags: |
  myapp:${{ github.ref_name }}-${{ github.sha.substring(0, 7) }}
  myapp:${{ github.ref_name }}
  
# 결과:
# myapp:main-3f48d1c
# myapp:main
```

또는 릴리스 태그일 때만:
```yaml
tags: |
  type=sha,prefix={{branch}}-,format=short
  type=semver,pattern={{version}}
  type=semver,pattern={{major}}.{{minor}}
```
</details>

### 문제 2: ECR Lifecycle Policy에서 언제 이미지가 삭제될까?

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Delete untagged images older than 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

<details>
<summary>해설</summary>

**이미지가 삭제되는 경우:**

1. **Untagged 이미지 (태그 없음)**
   - 새 이미지 푸시 후 기존 이미지에서 태그 제거
   - 예: myapp:latest를 새 이미지로 재태그
   ```
   Before: myapp:latest → Image A
   After: myapp:latest → Image B
   
   Image A는 untagged 상태 (7일 후 삭제)
   ```

2. **특정 기간 이상 지난 이미지**
   - sinceImagePushed: 이미지 푸시 시점부터 카운트
   - countNumber: 7 (7일)
   - 7일 이상 된 untagged 이미지 삭제

**삭제되지 않는 경우:**
- 태그가 있는 이미지 (myapp:v1.0.0)
- 최근 7일 내에 푸시된 이미지

**주의사항:**
```json
{
  "selection": {
    "tagStatus": "tagged",  // 주의: 이것도 삭제 가능
    "tagPrefixList": ["release"],
    "countType": "imageCountMoreThan",
    "countNumber": 10
  }
}
```
- 이 설정: release-로 시작하는 tagged 이미지 중 10개 이상만 유지
- release-v1.0.0 ~ release-v1.0.10이 있으면 v1.0.0만 삭제!

</details>

### 문제 3: 브랜치별로 다른 태그 정책을 적용할 수 있을까?

<details>
<summary>해설</summary>

**브랜치별 태그 정책:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: docker/metadata-action@v4
        id: meta
        with:
          images: myapp
          tags: |
            # Main 브랜치: Semantic Version + 짧은 SHA
            type=ref,event=branch,enable=${{ github.ref == 'refs/heads/main' }}
            type=sha,prefix={{branch}}-,format=short,enable=${{ github.ref == 'refs/heads/main' }}
            
            # Develop 브랜치: 브랜치 이름 + 짧은 SHA
            type=ref,event=branch,enable=${{ github.ref == 'refs/heads/develop' }}
            type=sha,prefix={{branch}}-,format=short,enable=${{ github.ref == 'refs/heads/develop' }}
            
            # Tag 푸시: Semantic Version만
            type=semver,pattern={{version}},enable=${{ startsWith(github.ref, 'refs/tags/v') }}
```

**결과:**

Main 브랜치 push:
```
myapp:main
myapp:main-abc1234
```

Develop 브랜치 push:
```
myapp:develop
myapp:develop-def5678
```

Tag 푸시 (v1.2.3):
```
myapp:1.2.3
myapp:1.2
myapp:1
myapp:latest
```

**자동 정리 정책 조정:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep main-* tags (최근 5개)",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["main-"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Keep develop-* tags (최근 3개)",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["develop-"],
        "countType": "imageCountMoreThan",
        "countNumber": 3
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 3,
      "description": "Keep semantic version tags",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 100  # 모든 v* 태그 유지
      },
      "action": {"type": "expire"}
    }
  ]
}
```
</details>

---

<div align="center">

**[⬅️ 이전: Spring Boot 최적화 이미지](./05-spring-boot-image.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Rolling Update 완전 분해 ➡️](../deployment-strategies/01-rolling-update-internals.md)**

</div>
