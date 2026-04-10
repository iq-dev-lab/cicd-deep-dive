<div align="center">

# 🚀 CI/CD Pipeline Deep Dive

**"배포 버튼을 누르는 것과, 파이프라인이 코드를 신뢰할 수 있는 결과물로 만드는 원리를 아는 것은 다르다"**

<br/>

> *"`main`에 push하면 배포되겠지 — 와 — GitHub Actions Runner가 Job을 어떻게 격리하고, Docker 레이어 캐시가 빌드를 어떻게 가속하며, ArgoCD가 Kubernetes 상태를 어떻게 수렴시키는지 아는 것의 차이를 만드는 레포"*

파이프라인 단계별 실패 원인을 즉시 파악하고, 롤백 시점과 전략을 설계하며,  
카나리 배포로 위험을 최소화하는 것 — **왜 이렇게 설계됐는가**라는 질문으로 CI/CD 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)](https://docs.github.com/en/actions)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat-square&logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![Docker](https://img.shields.io/badge/Docker_BuildKit-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com/build/buildkit/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io/docs/)
[![Docs](https://img.shields.io/badge/Docs-40개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

CI/CD에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "GitHub Actions에 YAML을 작성하면 됩니다" | `on:` Webhook 수신 → Runner 선택 → Job 스케줄링 → 컨테이너 프로비저닝 전 과정, `uses:` Action이 Docker 또는 JavaScript로 실행되는 방식 |
| "Docker 이미지를 빌드하세요" | 레이어 불변성과 명령어 순서가 캐시 히트율에 미치는 영향, 멀티 스테이지 빌드로 이미지를 800MB → 80MB로 줄이는 원리 |
| "ArgoCD를 쓰면 GitOps를 할 수 있습니다" | Reconciliation Loop가 Git과 Cluster 상태를 비교하는 주기, Self-Heal이 클러스터 직접 수정을 자동으로 되돌리는 원리 |
| "Rolling Update로 배포하세요" | `maxSurge`/`maxUnavailable`이 배포 속도와 가용성에 미치는 영향, Readiness Probe 실패 시 롤아웃이 중단되는 메커니즘 |
| "Canary 배포를 권장합니다" | Kubernetes Ingress 가중치 기반 라우팅, Argo Rollouts AnalysisTemplate으로 메트릭 기반 자동 승급/롤백 |
| "배포 실패 시 롤백하세요" | `kubectl rollout undo`의 내부 동작, 이전 ReplicaSet이 보존되는 방식, DB 마이그레이션이 있을 때 롤백이 불가능한 이유 |
| 이론 나열 | 실행 가능한 GitHub Actions YAML + `kubectl` + `argocd` CLI + Docker Compose 로컬 GitOps 환경 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Ch1-CI/CD_철학과_Pipeline_아키텍처-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)](./pipeline-fundamentals/01-cicd-philosophy.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-GitHub_Actions_완전_분해-181717?style=for-the-badge&logo=github&logoColor=white)](./github-actions/01-workflow-yaml-internals.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-Docker_이미지_빌드_최적화-2496ED?style=for-the-badge&logo=docker&logoColor=white)](./docker-build-optimization/01-layer-cache-principles.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-배포_전략과_Kubernetes_통합-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./deployment-strategies/01-rolling-update-internals.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-GitOps와_ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](./gitops-argocd/01-gitops-principles.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-테스트_자동화와_품질_게이트-4CAF50?style=for-the-badge&logo=testing-library&logoColor=white)](./test-automation/01-test-pyramid-pipeline.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-모니터링과_운영-FF6B35?style=for-the-badge&logo=grafana&logoColor=white)](./monitoring-operations/01-deployment-tracking.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: CI/CD 철학과 Pipeline 아키텍처

> **핵심 질문:** CI/CD가 해결하는 실제 문제는 무엇인가? Pipeline의 각 구성 요소는 어떻게 연결되고, GitOps는 명령형 배포와 무엇이 다른가?

<details>
<summary><b>수동 배포의 위험부터 GitOps 원칙까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. CI/CD 철학 — 수동 배포가 실패하는 이유](./pipeline-fundamentals/01-cicd-philosophy.md) | 휴먼 에러, 환경 불일치, 롤백 불가능이라는 수동 배포의 3대 위험, Continuous Integration이 "항상 배포 가능한 상태"를 만드는 원리, CI가 없을 때 머지 지옥이 발생하는 이유 |
| [02. Pipeline 구성 요소 — Trigger/Job/Step/Artifact](./pipeline-fundamentals/02-pipeline-components.md) | Webhook Trigger부터 Artifact 보존까지 각 구성 요소의 역할과 관계, GitHub-hosted Runner vs Self-hosted Runner의 비용/격리/보안 트레이드오프, 작업 격리가 왜 필수인가 |
| [03. GitHub Actions 내부 동작 — Webhook에서 Job 실행까지](./pipeline-fundamentals/03-github-actions-internals.md) | `git push` → Webhook 수신 → Runner 선택 → 컨테이너/VM 프로비저닝 전 과정, `GITHUB_TOKEN`의 권한 범위와 최소 권한 원칙, Job이 격리된 환경에서 실행되는 이유 |
| [04. Pipeline 설계 원칙 — Fast Fail과 병렬 실행](./pipeline-fundamentals/04-pipeline-design-principles.md) | Fast Fail이 피드백 루프를 단축시키는 원리, 단계별 게이트(테스트 통과 없이 배포 차단)를 설계하는 방법, 병렬 Job으로 전체 파이프라인 시간을 줄이는 전략 |
| [05. GitOps 원칙 — Git이 시스템 상태의 Single Source of Truth](./pipeline-fundamentals/05-gitops-principles.md) | Declarative / Versioned / Automated / Continuously Reconciled 4원칙, 명령형(`kubectl apply`) vs 선언형(Git 상태 수렴) 배포의 감사 추적성 차이, Pull 모델이 Push 모델보다 안전한 이유 |

</details>

<br/>

### 🔹 Chapter 2: GitHub Actions 완전 분해

> **핵심 질문:** `uses:` 뒤의 Action은 어떻게 실행되는가? Job 의존성은 내부적으로 어떻게 처리되고, Secrets는 어떻게 로그에서 가려지는가?

<details>
<summary><b>Workflow YAML 구조부터 OIDC 인증까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Workflow YAML 구조 분해 — 키워드 처리 방식](./github-actions/01-workflow-yaml-internals.md) | `on:` / `jobs:` / `steps:` / `env:` / `secrets:` 각 키워드의 실제 처리 방식, 표현식 `${{ }}` 의 평가 시점과 컨텍스트 범위, `if:` 조건으로 단계를 동적으로 제어하는 방법 |
| [02. Actions 내부 동작 — Docker Action vs JavaScript Action](./github-actions/02-actions-internals.md) | `uses:` Action이 Docker 컨테이너 또는 JavaScript로 실행되는 방식, Composite Action vs Reusable Workflow의 차이와 선택 기준, Action이 Runner 환경에 격리되는 이유 |
| [03. Job 의존성과 병렬화 — DAG와 Matrix Strategy](./github-actions/03-job-dependency-parallelism.md) | `needs:` 키워드로 DAG(방향 비순환 그래프)를 구성하는 방식, 병렬 Job의 Runner 할당 메커니즘, Matrix Strategy로 다중 OS/버전 환경을 동시에 테스트하는 방법 |
| [04. Secrets와 환경 변수 — OIDC로 자격증명 없이 인증](./github-actions/04-secrets-env-oidc.md) | Repository / Environment / Organization Secret 범위 차이, Secrets가 로그에 마스킹되는 내부 원리, OIDC로 AWS/GCP 장기 자격증명 없이 인증하는 방법과 보안 이점 |
| [05. 캐시 전략 — actions/cache 계층 구조](./github-actions/05-cache-strategy.md) | `actions/cache`의 `key` / `restore-keys` 계층 구조, Gradle/Maven/npm 의존성 캐시 적중률 최적화 방법, 캐시 충돌 방지와 캐시 무효화 전략 |
| [06. 아티팩트와 리포트 — Job 간 결과물 공유](./github-actions/06-artifacts-reports.md) | 빌드 결과물을 Job 간 공유하는 `upload-artifact` / `download-artifact` 패턴, 테스트 결과 리포트 업로드와 GitHub PR 코멘트 자동화, 바이너리 아티팩트 보존 정책과 비용 |

</details>

<br/>

### 🔹 Chapter 3: Docker 이미지 빌드 최적화

> **핵심 질문:** 레이어 순서 하나가 왜 빌드 시간을 10배 차이나게 만드는가? 멀티 스테이지 빌드는 내부에서 어떻게 동작하고, BuildKit은 무엇을 병렬화하는가?

<details>
<summary><b>레이어 캐시 원리부터 Spring Boot 최적화 이미지까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Dockerfile 레이어 캐시 원리 — 순서가 전부다](./docker-build-optimization/01-layer-cache-principles.md) | 레이어 불변성과 캐시 무효화 연쇄 반응, `COPY . .` 한 줄이 의존성 캐시를 날리는 이유, 의존성 먼저 복사 → 소스 복사 순서가 만드는 극적인 빌드 시간 차이 |
| [02. 멀티 스테이지 빌드 — 800MB를 80MB로](./docker-build-optimization/02-multi-stage-build.md) | Builder 스테이지에서 컴파일 후 Runtime 스테이지에만 결과물 복사하는 원리, Java 애플리케이션 이미지를 800MB → 80MB로 줄이는 실전 Before/After, `COPY --from=` 의 동작 방식 |
| [03. BuildKit 내부 동작 — 병렬 스테이지와 원격 캐시](./docker-build-optimization/03-buildkit-internals.md) | BuildKit이 의존성 없는 스테이지를 병렬로 빌드하는 방식, `--cache-to` / `--cache-from`으로 CI 간 캐시를 공유하는 원격 캐시 레지스트리 활용, `DOCKER_BUILDKIT=1`이 기존 빌더와 다른 점 |
| [04. 이미지 보안 — 루트 탈출과 공격 면적 최소화](./docker-build-optimization/04-image-security.md) | 루트가 아닌 사용자 실행이 컨테이너 탈출 시 피해를 제한하는 원리, 불필요한 패키지 제거와 `distroless` 베이스 이미지, Trivy로 CI 파이프라인에서 취약점을 자동 스캐닝하는 방법 |
| [05. Spring Boot 최적화 이미지 — Layered JAR와 Buildpacks](./docker-build-optimization/05-spring-boot-image.md) | Spring Boot Layered JAR 구조(dependencies / spring-boot-loader / snapshot-dependencies / application), `./gradlew bootBuildImage`와 Buildpacks의 동작 원리, JVM 컨테이너 메모리 설정(`-XX:MaxRAMPercentage`) |
| [06. 레지스트리 관리 — 태그 전략과 이미지 정리](./docker-build-optimization/06-registry-management.md) | Docker Hub vs GHCR vs ECR 비교, `latest` 태그가 프로덕션에서 위험한 이유, Git SHA 태그 전략, 오래된 이미지 자동 정리 정책과 저장소 비용 관리 |

</details>

<br/>

### 🔹 Chapter 4: 배포 전략과 Kubernetes 통합

> **핵심 질문:** Rolling Update는 언제 자동으로 중단되는가? Blue-Green과 Canary 중 무엇을 선택해야 하고, DB 마이그레이션이 있을 때 롤백이 왜 불가능한가?

<details>
<summary><b>Rolling Update부터 Zero-Downtime 배포 조건까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Rolling Update 완전 분해 — maxSurge와 Readiness Probe](./deployment-strategies/01-rolling-update-internals.md) | `maxSurge` / `maxUnavailable` 설정이 배포 속도와 가용성에 미치는 영향, 새 Pod의 Readiness Probe 실패 시 롤아웃이 자동 중단되는 메커니즘, `kubectl rollout status`로 진행 상황 추적 |
| [02. Blue-Green 배포 — 즉각 롤백의 비용](./deployment-strategies/02-blue-green-deployment.md) | 두 환경을 동시에 유지하는 비용 구조, Kubernetes Service 트래픽 전환 방식, 롤백이 Service selector 변경 하나로 즉각 처리되는 이유, DB 스키마 변경과의 충돌 문제 |
| [03. Canary 배포 — 트래픽을 단계적으로 이동하기](./deployment-strategies/03-canary-deployment.md) | 트래픽 비율을 단계적으로 증가시키는 방법, Kubernetes Ingress 가중치 기반 라우팅(`nginx.ingress.kubernetes.io/canary-weight`), 에러율 임계값 초과 시 자동 롤백 트리거 |
| [04. Argo Rollouts — 메트릭 기반 자동 승급과 롤백](./deployment-strategies/04-argo-rollouts.md) | `AnalysisTemplate`으로 Prometheus 쿼리 기반 에러율 임계값을 설정하는 방법, 자동 승급/롤백 조건, 수동 개입이 필요한 시점과 `kubectl argo rollouts promote` |
| [05. 롤백 전략 — 언제 되돌릴 수 없는가](./deployment-strategies/05-rollback-strategy.md) | `kubectl rollout undo`가 이전 ReplicaSet을 활성화하는 내부 동작, DB 마이그레이션이 있을 때 롤백이 불가능한 이유, Forward-Only 전략으로 전진 수정하는 접근법 |
| [06. Zero-Downtime 배포 조건 — Graceful Shutdown과 preStop](./deployment-strategies/06-zero-downtime.md) | SIGTERM을 받아 진행 중인 요청을 완료하는 Graceful Shutdown 구현, Connection Draining이 없을 때 발생하는 503 에러, `preStop` Hook으로 종료 전 대기 시간을 확보하는 방법 |
| [07. 배포 파이프라인 전체 흐름 — 코드 Push부터 헬스체크까지](./deployment-strategies/07-deployment-pipeline-flow.md) | 코드 Push → 테스트 → 이미지 빌드 → 레지스트리 Push → Kubernetes 배포 → 헬스체크 → 알림 전 과정, 각 단계에서 실패했을 때 파이프라인이 멈추는 조건, 운영 관점의 체크리스트 |

</details>

<br/>

### 🔹 Chapter 5: GitOps와 ArgoCD

> **핵심 질문:** ArgoCD의 Reconciliation Loop는 얼마나 자주 실행되고, Self-Heal은 어떻게 직접 수정을 감지하는가? Git에 Secret을 저장하면 왜 안 되는가?

<details>
<summary><b>GitOps 4원칙부터 멀티 클러스터 관리까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. GitOps 원칙 완전 분해 — 4원칙과 감사 추적성](./gitops-argocd/01-gitops-principles.md) | Declarative / Versioned / Automated / Continuously Reconciled 4원칙의 실제 의미, 명령형 배포와 비교했을 때 Git이 Audit Log를 대체하는 방식, Pull 모델이 보안상 유리한 이유 |
| [02. ArgoCD 아키텍처 — Reconciliation Loop 내부](./gitops-argocd/02-argocd-architecture.md) | Application Controller / Repo Server / API Server의 역할 분리, Reconciliation Loop가 Git과 Cluster 상태를 비교하는 주기(3분 기본 + Webhook 즉시 트리거), `OutOfSync` 상태 감지 원리 |
| [03. Sync 전략 — Self-Heal과 Pruning](./gitops-argocd/03-sync-strategy.md) | Manual vs Automated Sync 차이, Self-Heal이 `kubectl edit`으로 직접 수정한 내용을 자동으로 되돌리는 원리, Pruning이 Git에서 삭제된 리소스를 클러스터에서 자동 제거하는 방식 |
| [04. Kustomize와 Helm 통합 — 환경별 설정 분리](./gitops-argocd/04-kustomize-helm-integration.md) | ArgoCD에서 Kustomize overlay로 dev/staging/prod 환경별 설정을 분리하는 방법, Helm `values.yaml`을 Git으로 관리하는 패턴, 템플릿 렌더링이 ArgoCD 서버 측에서 일어나는 이유 |
| [05. 멀티 클러스터 관리 — ApplicationSet과 환경별 프로모션](./gitops-argocd/05-multi-cluster-management.md) | `ApplicationSet`으로 여러 클러스터에 동일 앱을 배포하는 방법, 클러스터별 설정 오버라이드, dev → staging → prod 환경별 프로모션 파이프라인 설계 |
| [06. Secret 관리 — Git에 저장하면 안 되는 이유](./gitops-argocd/06-secret-management.md) | Git에 평문 Secret을 커밋하는 위험(히스토리에 영구 기록), Sealed Secrets / External Secrets Operator / Vault Agent 비교, SOPS(Mozilla Secret OPerationS)로 암호화해서 Git에 저장하는 방법 |

</details>

<br/>

### 🔹 Chapter 6: 테스트 자동화와 품질 게이트

> **핵심 질문:** 단위/통합/E2E 테스트를 Pipeline의 어느 단계에 배치해야 하는가? SonarQube Quality Gate는 어떻게 파이프라인을 멈추는가?

<details>
<summary><b>테스트 피라미드 배치 전략부터 보안 스캐닝 자동화까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 테스트 피라미드와 Pipeline — 어디에 배치할 것인가](./test-automation/01-test-pyramid-pipeline.md) | 단위/통합/E2E 테스트의 Pipeline 배치 전략과 피드백 속도 트레이드오프, 빠른 피드백을 위한 테스트 병렬화, Docker 컨테이너로 테스트 환경을 격리하는 방법 |
| [02. Spring Boot 테스트 최적화 — Testcontainers와 컨텍스트 재사용](./test-automation/02-spring-boot-test-optimization.md) | `@SpringBootTest` 컨텍스트 재사용으로 테스트 시간을 줄이는 방법, Testcontainers로 실제 DB/Redis를 사용하는 통합 테스트, `@MockBean` 남용이 컨텍스트를 재생성하게 만드는 이유 |
| [03. 코드 품질 게이트 — SonarQube와 커버리지 임계값](./test-automation/03-code-quality-gate.md) | SonarQube Quality Gate가 파이프라인을 중단시키는 조건과 설정 방법, JaCoCo 코드 커버리지 임계값을 파이프라인에 통합하는 방법, SpotBugs/Checkstyle 정적 분석 자동화 |
| [04. 성능 회귀 감지 — k6 Smoke Test와 Baseline 비교](./test-automation/04-performance-regression.md) | k6 smoke test를 Pipeline에 통합하는 방법, 응답시간 임계값 초과 시 배포 차단 조건 설정, Baseline 비교로 성능 저하를 자동 감지하는 전략 |
| [05. 보안 스캐닝 자동화 — CVE 감지와 Secret 노출 방지](./test-automation/05-security-scanning.md) | Trivy 이미지 취약점 스캔을 CI에 통합하는 방법, OWASP Dependency-Check로 라이브러리 CVE를 감지하는 방식, TruffleHog/git-secrets으로 커밋에 포함된 Secret 노출을 사전 차단 |

</details>

<br/>

### 🔹 Chapter 7: 모니터링, 알림과 운영

> **핵심 질문:** 배포 실패 원인을 30분 안에 찾으려면 어떻게 해야 하는가? 파이프라인 병목은 어디서 발생하고, 자동 롤백 트리거는 어떻게 설계하는가?

<details>
<summary><b>배포 추적부터 장애 시나리오 복구까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 배포 추적과 알림 — Slack 연동과 GitHub Deployment API](./monitoring-operations/01-deployment-tracking.md) | Slack/Discord 배포 성공/실패 알림 구성, GitHub Deployment API로 배포 이력을 추적하는 방법, Datadog/Grafana 배포 마커로 배포와 메트릭 변화를 연결하는 방식 |
| [02. 파이프라인 장애 진단 — 간헐적 실패의 원인 분석](./monitoring-operations/02-pipeline-failure-diagnosis.md) | `ACTIONS_STEP_DEBUG` 디버그 로깅 활성화, Runner 환경 상태 확인 방법, 간헐적 실패(Flaky Test, 네트워크 타임아웃, 캐시 충돌)의 원인 분석 방법론 |
| [03. Pipeline 성능 최적화 — 병목 단계 식별](./monitoring-operations/03-pipeline-performance-optimization.md) | Job 실행 시간 분석과 병목 단계 식별 방법, 의존성 캐시 적중률 향상 전략, 불필요한 재빌드 제거로 파이프라인 전체 시간을 단축하는 접근법 |
| [04. 환경 관리 전략 — Dev/Staging/Prod 분리](./monitoring-operations/04-environment-management.md) | Dev/Staging/Prod 환경 분리 전략, 환경별 변수와 Secret 관리, Production-like Staging 환경을 구성해야 하는 이유와 비용 트레이드오프 |
| [05. 장애 시나리오와 복구 — 자동 롤백 트리거와 Postmortem](./monitoring-operations/05-incident-recovery.md) | 배포 중 에러율 급증 시 자동 롤백 트리거 설계, 데이터베이스 롤백이 없는 이유와 Forward-Only 복구 전략, 포스트모텀(Postmortem) 작성으로 반복 장애를 예방하는 방법 |

</details>

---

## 🔗 연결 학습 지도

이 레포는 단독으로 보아도 완결되지만, 아래 레포와 함께 보면 **내부 원리의 연결 고리**가 보입니다.

```
선행 학습 (권장):
  docker-deep-dive         →  컨테이너 이미지 레이어, Namespace/Cgroup 이해 후
  kubernetes-deep-dive     →  Pod, Deployment, Service, Reconciliation Loop 이해 후
  linux-for-backend        →  프로세스 격리, 환경변수, 파일시스템 이해 후

연계 학습:
  observability-deep-dive  →  배포 마커와 메트릭 연동, 배포 후 이상 감지
```

| 레포 | 이 레포와 연결되는 지점 |
|------|----------------------|
| `docker-deep-dive` | Docker 레이어 캐시 원리(Ch3), 멀티 스테이지 빌드 내부, 컨테이너 이미지 보안 |
| `kubernetes-deep-dive` | Rolling Update의 Deployment 내부(Ch4), Readiness/Liveness Probe 설정, Service 트래픽 전환 |
| `linux-for-backend` | GitHub Actions Runner의 프로세스 격리, SIGTERM Graceful Shutdown, 파일시스템 레이어 |
| `observability-deep-dive` | 배포 추적 및 Grafana 마커(Ch7), Prometheus 메트릭 기반 Canary 분석, 장애 진단 |

---

## 🧪 실험 환경

모든 실험은 로컬에서 재현할 수 있습니다.

```yaml
# docker-compose.yml — 로컬 GitOps 환경
services:
  gitea:
    image: gitea/gitea:latest
    ports:
      - "3000:3000"
    volumes:
      - gitea-data:/data

  act-runner:
    image: gitea/act_runner:latest
    environment:
      GITEA_INSTANCE_URL: http://gitea:3000
      GITEA_RUNNER_REGISTRATION_TOKEN: ${RUNNER_TOKEN}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  argocd:
    image: quay.io/argoproj/argocd:latest
    ports:
      - "8080:8080"

volumes:
  gitea-data:
```

```bash
# 핵심 진단 명령어 세트
kubectl rollout status deployment/myapp -n production   # Rolling Update 진행 상황
kubectl rollout undo deployment/myapp -n production     # 롤백 실행
argocd app get myapp                                    # ArgoCD Sync 상태 확인
argocd app sync myapp                                   # 수동 Sync 트리거
kubectl argo rollouts set weight myapp 20               # 카나리 트래픽 20%로 조정
kubectl argo rollouts promote myapp                     # 카나리 승급
docker history myapp:latest --no-trunc                  # Docker 레이어 캐시 분석
```

---

## 💡 이 레포가 만드는 차이

```
Before (원리 없이):
  배포 실패 → 로그 전체 검색 → 30분 소요
  이미지 빌드 느림 → 서버 사양 업그레이드
  ArgoCD OutOfSync → 무작정 Force Sync
  롤백 → kubectl rollout undo (DB 변경 있었는데...)

After (원리를 알고):
  배포 실패 → Readiness Probe 실패 → 헬스체크 엔드포인트 확인 → 5분 내 진단
  이미지 빌드 느림 → 레이어 순서 수정 → 의존성 캐시 복구 → 빌드 시간 10배 단축
  ArgoCD OutOfSync → Self-Heal 활성화 → 드리프트 자동 수렴
  롤백 → DB Forward-Only 전략 → 새 버전으로 핫픽스 전진
```

---

<div align="center">

**"파이프라인은 코드가 신뢰로 바뀌는 과정이다"**

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)

</div>
