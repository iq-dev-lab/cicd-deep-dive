# 캐시 전략 — actions/cache 계층 구조

## 🎯 핵심 질문

`actions/cache`의 `key`와 `restore-keys`는 정확히 어떻게 매칭되고, 캐시 계층 구조는 어떻게 구성될까? Gradle, Maven, npm, yarn의 캐시 경로와 키 설계는 어떻게 다를까? 캐시 키에 lock 파일 해시를 포함해야 하는 이유는? `actions/setup-java`의 `cache: 'gradle'` 옵션은 내부적으로 무엇을 하는가?

## 🔍 왜 이 개념이 실무에서 중요한가

- **CI/CD 속도**: 캐시가 제대로 작동하면 의존성 다운로드 시간을 5분에서 10초로 줄일 수 있다.
- **비용**: 가난한 팀에서는 GitHub Actions 분당 비용이 계산되므로 캐시 5분 절감 = 약 $0.04 절감/실행
- **캐시 미스(Cache Miss)**: 잘못된 키 설계로 매번 캐시를 다시 만들면 캐시가 없는 것보다 더 느릴 수 있다.
- **디스크 공간**: 캐시는 8GB 제한이 있고, 초과하면 먼저 저장된 캐시부터 삭제된다.

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```yaml
# ❌ 잘못된 예 1: 정적 캐시 키 (항상 캐시 미스)
name: Bad Cache Key
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache npm
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: npm-cache  # 의존성 버전 변경해도 같은 키
          # → 의존성 업그레이드 후에도 오래된 캐시 사용 (위험)

      - run: npm install

# ❌ 잘못된 예 2: 너무 자주 바뀌는 키 (항상 캐시 미스)
name: Overly Specific Cache Key
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache gradle
        uses: actions/cache@v3
        with:
          path: ~/.gradle/caches
          key: gradle-${{ hashFiles('build.gradle', 'build.gradle.kts', 'settings.gradle') }}-${{ runner.os }}-${{ github.run_id }}
          # github.run_id가 포함되면 매번 다른 키!
          # → 캐시가 절대 매칭되지 않음

# ❌ 잘못된 예 3: restore-keys 미사용 (캐시 계층 구조 없음)
name: No Restore Keys
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache npm
        uses: actions/cache@v3
        with:
          path: node_modules
          key: npm-${{ hashFiles('package-lock.json') }}
          # restore-keys 없음
          # package-lock.json이 약간 변경되면 완전 캐시 미스
          # → 모든 패키지 재다운로드

# ❌ 잘못된 예 4: 너무 큰 캐시 경로
name: Huge Cache
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache everything
        uses: actions/cache@v3
        with:
          path: /home/runner  # 전체 홈 디렉토리!
          key: everything-${{ github.run_id }}
          # 캐시 사이즈 > 8GB → 저장 실패
          # 매번 캐시 생성 시도하므로 로그가 가득 참
```

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```yaml
# ✅ 올바른 예 1: lock 파일 해시를 포함한 캐시 키
name: Proper Cache Key
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache npm dependencies
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            npm-${{ runner.os }}-
            # package-lock.json이 변경되면 새 캐시 생성
            # 변경되지 않으면 이전 캐시 재사용

      - run: npm install

# ✅ 올바른 예 2: Maven 계층 구조 캐시
name: Maven Cache with Layers
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache Maven dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: maven-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            maven-${{ runner.os }}-
            # pom.xml 변경 → 새 캐시 생성
            # 기존 캐시가 있으면 복구 후 업데이트 (부분 다운로드)

      - run: mvn clean install

# ✅ 올바른 예 3: actions/setup-java의 내장 캐시
name: Java with Built-in Cache
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK and cache
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'  # 또는 'maven'
          # 내부적으로 자동 캐시 설정:
          # key: gradle-${{ runner.os }}-${{ hashFiles('gradle/wrapper/gradle-wrapper.properties', '**/build.gradle*', '**/gradle.properties') }}
          # path: ~/.gradle/caches (Gradle의 경우)

      - run: ./gradlew build

# ✅ 올바른 예 4: 여러 의존성 관리자 캐시
name: Multi-Cache Strategy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'  # node_modules와 ~/.npm 자동 캐시

      - name: Cache Python
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            pip-${{ runner.os }}-

      - name: Cache Docker images
        uses: actions/cache@v3
        with:
          path: /tmp/docker-images
          key: docker-${{ runner.os }}-${{ hashFiles('Dockerfile') }}
          restore-keys: |
            docker-${{ runner.os }}-
```

## 🔬 내부 동작 원리

### 1. Cache Key 매칭 알고리즘

GitHub Actions의 캐시 매칭은 **정확한 일치(Exact Match) 먼저, 그 다음 부분 일치(Prefix Match)**입니다:

**알고리즘:**
```
1. 정확한 키 매칭 (Exact Match)
   - key가 정확히 일치하는 캐시 찾기
   - 찾으면 그 캐시 복구, 종료

2. 부분 매칭 (Restore Keys)
   - restore-keys 리스트를 순서대로 검사
   - 첫 번째로 매칭되는 프리픽스 찾기
   - 그 캐시 복구, 종료

3. 캐시 미스
   - 정확한 키도 부분 키도 매칭되지 않음
   - Job 진행 (의존성 재다운로드)
   - Job 완료 후 새 캐시 저장 (key로)
```

**예시:**

```yaml
cache:
  key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
  # 예: npm-Linux-abc123def456
  
  restore-keys: |
    npm-${{ runner.os }}-
    npm-
```

**매칭 순서:**
```
1. 정확한 키: npm-Linux-abc123def456 찾기
   ✅ 찾으면 복구

2. restore-keys 첫 번째: npm-Linux- (프리픽스)
   → npm-Linux-로 시작하는 모든 캐시 중 가장 최신 선택
   ✅ 찾으면 복구

3. restore-keys 두 번째: npm-
   → npm-으로 시작하는 모든 캐시 중 가장 최신 선택
   ✅ 찾으면 복구

4. 모두 실패 → 캐시 미스
```

### 2. 캐시 저장 메커니즘

**캐시 저장 과정:**
```
1. 지정된 경로의 모든 파일 수집
   path: ~/.npm
   → ~/.npm 디렉토리의 모든 파일 압축

2. 캐시 크기 확인
   - 파일 크기가 400MB 이상이면 경고
   - 2GB 이상이면 저장 실패 (단일 파일)
   - 전체 캐시 8GB 초과 시 가장 오래된 캐시 삭제

3. 캐시에 메타데이터 추가
   - key: npm-Linux-abc123def456
   - created_at: 현재 시간
   - expiration: 7일 (기본값)

4. GitHub 서버에 업로드
   - 같은 key로 이미 저장된 캐시 있으면 덮어씀
   - 다른 branch는 자체 캐시 구분

5. 캐시 만료
   - 7일간 접근 없으면 자동 삭제
   - 같은 key로 재저장하면 만료 리셋
```

### 3. Branch별 캐시 공유

캐시는 **같은 브랜치와 부모 브랜치에서만 공유**됩니다:

```
main 브랜치
  ├─ 캐시 1 (npm-Linux-abc123)
  └─ 캐시 2 (gradle-Linux-def456)

feature/new-dep 브랜치 (main에서 생성)
  1. npm-Linux-abc123 캐시 접근 가능 (main의 캐시)
  2. feature/new-dep에서 새 캐시 생성
     → feature/new-dep 전용
  3. main 병합 후 main의 캐시는 여전히 독립적

develop 브랜치 (main에서 분기)
  → main의 캐시 사용 불가 (다른 브랜치)
```

### 4. 의존성 관리자별 캐시 전략

#### npm/yarn

```yaml
# npm 캐시 위치
~/.npm            # npm 저장소 캐시
~/.cache/yarn     # yarn 저장소 캐시
node_modules      # 설치된 패키지 (보통 캐시 안 함)

# 추천 키 설계
key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

# 이유: package-lock.json이 정확한 버전 지정
# - 같은 파일 → 같은 의존성 → 같은 캐시
# - 파일 변경 → 새 캐시 생성
```

#### Gradle

```yaml
# gradle 캐시 위치
~/.gradle/caches      # gradle 컴파일 결과 캐시
~/.gradle/wrapper     # gradle wrapper 바이너리

# 추천 키 설계
key: gradle-${{ runner.os }}-${{ hashFiles('gradle/wrapper/gradle-wrapper.properties', '**/build.gradle*') }}

# 이유:
# - gradle-wrapper.properties: Gradle 버전 지정
# - build.gradle: 의존성 버전 지정
# - 이 둘이 변경되지 않으면 캐시 재사용 가능

# 주의: gradle.lockfile 사용하는 경우
key: gradle-${{ hashFiles('gradle.lockfile') }}
# lockfile이 정확한 버전 관리
```

#### Maven

```yaml
# maven 캐시 위치
~/.m2/repository      # 다운로드된 의존성 저장

# 추천 키 설계
key: maven-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}

# 이유:
# - pom.xml이 의존성 버전 지정
# - 여러 모듈의 pom.xml 모두 해시에 포함
# - 프로젝트 변경 시 캐시 갱신

# 주의: pom.xml은 명시적 버전만 포함
# - SNAPSHOT 버전은 매번 최신 다운로드 가능 (캐시 의미 없음)
```

### 5. `actions/setup-*`의 내장 캐시

`actions/setup-java`, `actions/setup-node` 등이 `cache:` 파라미터를 지원합니다:

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: 'gradle'  # 또는 'maven'

# 내부적으로 다음을 자동 실행:
# 1. 올바른 캐시 경로 결정 (gradle vs maven)
# 2. 최적의 키 생성
# 3. restore-keys 구성
# 4. actions/cache 호출
```

**내부 구현 (근사):**
```yaml
# setup-java cache: 'gradle'
- uses: actions/cache@v3
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ runner.os }}-${{ hashFiles('gradle/wrapper/gradle-wrapper.properties', '**/build.gradle*', '**/gradle.properties') }}
    restore-keys: |
      gradle-${{ runner.os }}-

# setup-java cache: 'maven'
- uses: actions/cache@v3
  with:
    path: ~/.m2/repository
    key: maven-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      maven-${{ runner.os }}-
```

## 💻 실전 실험 (GitHub Actions YAML 재현 가능한 예시)

### 실험 1: npm 캐시 기본 설정

```yaml
name: npm Cache Test
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node and cache
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm install

      - name: Build
        run: npm run build

# 첫 실행: npm install (5분)
# 두 번째 실행 (package-lock.json 변경 X): npm install (10초, 캐시에서)
# 세 번째 실행 (package-lock.json 변경 O): npm install (2분, 캐시 + 새 패키지)
```

### 실험 2: Gradle 캐시 옵션 비교

```yaml
name: Gradle Cache Comparison
on: [push]

jobs:
  # 방식 1: setup-java 내장 캐시
  with-setup-java:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK with cache
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Build with Gradle
        run: ./gradlew build

  # 방식 2: 수동 캐시 설정 (더 세밀한 제어)
  manual-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ runner.os }}-${{ hashFiles('gradle/wrapper/gradle-wrapper.properties', '**/build.gradle*', 'gradle.properties') }}
          restore-keys: |
            gradle-${{ runner.os }}-

      - name: Build with Gradle
        run: ./gradlew build

      - name: Print cache info
        run: |
          du -sh ~/.gradle/caches
          du -sh ~/.gradle/wrapper
```

### 실험 3: 계층화된 restore-keys

```yaml
name: Layered Cache Strategy
on: [push]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    steps:
      - uses: actions/checkout@v4

      - name: Cache dependencies (layered)
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.m2/repository
          # 3단계 매칭 전략
          key: |
            deps-${{ runner.os }}-${{ hashFiles('**/build.gradle*', '**/pom.xml', 'gradle.properties') }}
          
          restore-keys: |
            # 1순위: 정확한 의존성 파일 일치 (위의 key)
            # (자동)
            
            # 2순위: 같은 OS, 근처 의존성
            deps-${{ runner.os }}-
            
            # 3순위: 모든 OS의 최신 캐시
            deps-

      - name: Build
        run: |
          if [[ "${{ runner.os }}" == "Windows" ]]; then
            gradle.bat build
          else
            ./gradlew build
          fi

# 매칭 시뮬레이션:
# macos에서 실행, build.gradle 변경:
#   1. deps-macos-[new-hash] → 미스
#   2. deps-macos- → 이전 macos 캐시 (새 의존성으로 업데이트)
#   3. deps- → 다른 OS 캐시도 고려 (최후의 수단)
```

### 실험 4: 캐시 성능 측정

```yaml
name: Cache Performance Test
on: [push]

jobs:
  with-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Time npm install with cache
        run: |
          START=$(date +%s%N)
          npm install --prefer-offline
          END=$(date +%s%N)
          DURATION=$(( (END - START) / 1000000 ))
          echo "npm install with cache: ${DURATION}ms"
          echo "CACHE_TIME=$DURATION" >> $GITHUB_ENV

  without-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node (no cache)
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Clear npm cache
        run: npm cache clean --force

      - name: Time npm install without cache
        run: |
          START=$(date +%s%N)
          npm install
          END=$(date +%s%N)
          DURATION=$(( (END - START) / 1000000 ))
          echo "npm install without cache: ${DURATION}ms"
          echo "NO_CACHE_TIME=$DURATION" >> $GITHUB_ENV

  compare:
    needs: [with-cache, without-cache]
    runs-on: ubuntu-latest
    steps:
      - run: echo "비교 로그에서 시간 확인"

# 예상 결과:
# with-cache: ~10 seconds
# without-cache: ~50 seconds
# 캐시로 인한 속도 향상: 5배
```

### 실험 5: 캐시 적중률 모니터링

```yaml
name: Cache Hit Monitoring
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache npm
        id: cache
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          restore-keys: npm-${{ runner.os }}-

      - name: Cache metrics
        run: |
          echo "Cache hit: ${{ steps.cache.outputs.cache-hit }}"
          # true = 정확한 키 매칭 (캐시 적중)
          # false = restore-keys 매칭 또는 미스

      - name: Conditional install
        run: |
          if [[ "${{ steps.cache.outputs.cache-hit }}" != 'true' ]]; then
            echo "Cache miss detected, installing..."
            npm install
          else
            echo "Cache hit! Using cached dependencies..."
          fi

      - name: Build
        run: npm run build
```

## 📊 성능/비용 비교

| 캐시 전략 | 첫 실행 | 캐시 히트 | 캐시 미스 | 월 비용 절감 |
|---------|--------|---------|---------|-----------|
| **캐시 없음** | 5분 | 5분 | 5분 | $0 |
| **정적 키** | 5분 | 10초 (미스) | 10초 | $0 (미스 많음) |
| **lock 파일 해시** | 5분 | 10초 | 2분 | ~$0.10 (100회) |
| **계층화된 키** | 5분 | 10초 | 1분 30초 | ~$0.15 (100회) |

**계산:**
- Runner 분당 비용: $0.005 (GH-hosted 기본)
- 월 100회 실행
- 정적 키: 캐시 미스 → 미스 비용 포함되지 않음 (오버헤드만)
- 해시 키: 평균 (10초 × 50% + 2분 × 50%) = 1분 5초 절감 = $0.00875 / 회

## ⚖️ 트레이드오프

| 선택 | 장점 | 단점 |
|------|------|------|
| **정적 키** | 간단함 | 캐시 미스 자주 발생 |
| **lock 파일 해시** | 자동 업데이트, 정확함 | lock 파일 변경 감지 필요 |
| **계층화된 키** | 부분 캐시 재사용 | 복잡도 증가, 여러 파일 모니터링 |
| **setup-* 내장 캐시** | 자동 최적화, 간단함 | 커스터마이징 불가 |
| **수동 캐시** | 세밀한 제어 | 관리 부담 증가 |

## 📌 핵심 정리

1. **Key 설계**: lock 파일 해시 포함 → 의존성 변경 자동 감지
2. **restore-keys**: 계층화된 프리픽스 → 부분 캐시 재사용
3. **캐시 경로**: OS별 다름 (npm ~/.npm, gradle ~/.gradle/caches, maven ~/.m2/repository)
4. **내장 캐시**: actions/setup-java, setup-node 등의 `cache:` 파라미터 활용
5. **캐시 히트율**: 정적 키 (0%) < lock 해시 (80%) < 계층화 (95%)
6. **성능**: 캐시로 5배 속도 향상 가능 (평균 5분 → 1분)

## 🤔 생각해볼 문제

### 문제 1: 이 캐시 키 설정의 문제점은?

```yaml
cache:
  path: node_modules
  key: npm-v1-${{ hashFiles('package.json') }}
  restore-keys: |
    npm-v1-
    npm-
```

<details>
<summary>해설</summary>

**문제점:**

1. **hashFiles('package.json')만 사용**: package-lock.json 미포함
   - 버그 수정으로 같은 버전의 다른 부 버전이 lock에만 기록되면?
   - 예: package.json은 "react": "^18.0.0"이지만
   - lock 파일: 18.0.0 vs 18.0.1 (patch 버전 차이)
   - 캐시 키는 같지만 실제 의존성은 다를 수 있음

2. **node_modules 캐시**: OS별 바이너리 호환성 문제
   - Linux에서 캐시된 node_modules를 macOS에서 사용?
   - native 바인딩이 깨질 수 있음

3. **restore-keys의 npm- 프리픽스**: 모든 OS 혼용 위험
   - npm-windows-v1에서 캐시된 것을 npm-linux에서 사용?
   - 바이너리 호환성 문제

**올바른 버전:**
```yaml
cache:
  path: ~/.npm  # node_modules 대신 npm 캐시
  key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
  restore-keys: |
    npm-${{ runner.os }}-
```

또는 actions/setup-node 사용:
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'  # 자동 최적화
```
</details>

### 문제 2: 이 Gradle 캐시 설정에서 얼마나 자주 캐시 미스가 발생할까?

```yaml
cache:
  path: ~/.gradle/caches
  key: gradle-${{ hashFiles('build.gradle') }}
  restore-keys: gradle-
```

프로젝트 구조:
```
app/
├─ build.gradle     (dependency block)
├─ settings.gradle  (included modules)
└─ libs/
   ├─ lib1/build.gradle
   ├─ lib2/build.gradle
   └─ lib3/build.gradle
```

<details>
<summary>해설</summary>

**캐시 미스 빈도: 매우 높음 (50% 이상)**

**이유:**

1. **root build.gradle만 해시**: 서브프로젝트 build.gradle 미포함
   - app/libs/lib1/build.gradle의 의존성 추가 → 캐시 키 변경 없음
   - 의존성이 없는 캐시 사용 → 빌드 실패 가능

2. **settings.gradle 미포함**: 모듈 구조 변경 감지 안 됨
   - 새 모듈 추가 → 캐시 키 변경 없음

3. **gradle.properties 미포함**: Gradle 플러그인 버전 미포함
   - Gradle 플러그인 버전 변경 → 캐시 미스 감지 안 됨

**올바른 버전:**
```yaml
cache:
  path: ~/.gradle/caches
  key: gradle-${{ runner.os }}-${{ hashFiles('gradle/wrapper/gradle-wrapper.properties', '**/build.gradle*', 'settings.gradle', 'gradle.properties') }}
  restore-keys: gradle-${{ runner.os }}-
```

또는 setup-java 사용:
```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    cache: 'gradle'  # 자동 모든 파일 포함
```

**캐시 미스율 개선:**
- 원래: 50%
- 수동 설정: 10%
- setup-java: 5%
</details>

### 문제 3: CI/CD 중간에 캐시를 초기화하려면?

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cache
        uses: actions/cache@v3
        with:
          path: ~/.gradle/caches
          key: gradle-${{ runner.os }}-${{ hashFiles('build.gradle') }}

      - run: ./gradlew build
```

만약 캐시가 손상되었거나 새로 시작하고 싶다면?

<details>
<summary>해설</summary>

**문제:** GitHub Actions의 actions/cache는 삭제 기능이 없음

**해결책 1: 키 변경**
```yaml
cache:
  key: gradle-${{ runner.os }}-${{ hashFiles('build.gradle') }}-v2
  # v2 추가 → 새로운 캐시 생성
  # 이전 캐시는 7일 후 자동 삭제
```

**해결책 2: 타임스탬프 포함**
```yaml
cache:
  key: gradle-${{ runner.os }}-${{ hashFiles('build.gradle') }}-${{ github.run_id }}
  # 매번 새로운 캐시 생성 (캐시 의미 없음)
  # 복구만 하고 저장하지 않으려면:
  # → 실제로는 좋지 않은 방법
```

**해결책 3: 수동 캐시 삭제 (권장)**
```bash
# GitHub CLI로 캐시 조회
gh actions-cache list --repo owner/repo

# 특정 캐시 삭제
gh actions-cache delete --repo owner/repo --key gradle-Linux-abc123def456

# 모든 캐시 삭제
gh actions-cache list --repo owner/repo | \
  cut -f1 | \
  xargs -I {} gh actions-cache delete --repo owner/repo --key {}
```

**해결책 4: Workflow에서 조건부 캐시 초기화**
```yaml
cache:
  key: gradle-${{ runner.os }}-${{ hashFiles('build.gradle') }}
  # 특정 환경변수로 강제 초기화
  # → GitHub Secrets에 CACHE_RESET 플래그 추가
  # → key에 포함시켜 새 캐시 강제 생성

- run: |
    if [[ "${{ secrets.CACHE_RESET }}" == "true" ]]; then
      rm -rf ~/.gradle/caches
      echo "Cache cleared"
    fi
```
</details>

---

<div align="center">
**[⬅️ 이전: Secrets와 환경 변수](./04-secrets-env-oidc.md)** | **[홈으로 🏠](../README.md)** | **[다음: 아티팩트와 리포트 ➡️](./06-artifacts-reports.md)**
</div>
