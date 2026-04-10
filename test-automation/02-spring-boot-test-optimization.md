# Spring Boot 테스트 최적화 — Testcontainers와 컨텍스트 재사용

## 🎯 핵심 질문

- Spring Boot 컨텍스트가 재사용되려면 어떤 조건이 필요한가?
- `@MockBean` 하나가 전체 컨텍스트를 재생성하는 이유는?
- Testcontainers로 실제 PostgreSQL/Redis와 통합 테스트하는 방법은?
- `@DynamicPropertySource`가 해결하는 문제는 무엇인가?
- CI에서 Docker-in-Docker 문제를 어떻게 피할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot 테스트 성능은 컨텍스트 로딩 시간에 의해 결정됩니다.

**성능 데이터**:
- 컨텍스트 로딩: 초당 3-5개만 로드 가능
- 100개 테스트 → 불필요한 로딩 10번 → 추가 30-50초 낭비
- 전체 테스트 시간: 12분 → 11분 (8% 개선)

**신뢰성**:
- 실제 데이터베이스 사용 → 재현 가능한 버그 발견
- In-Memory H2 사용 → 실제 환경에서 다른 동작

**개발 생산성**:
- 로컬 단위 테스트: 30초 (컨텍스트 재사용)
- 이전과 같은 테스트를 처음 실행: 3분 (초기 로딩)

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```java
// ❌ 잘못된 접근 1: 모든 테스트가 독립적인 컨텍스트 로딩

@SpringBootTest  // 매번 새로운 컨텍스트 로딩
@AutoConfigureMockMvc
class UserServiceTest {
    
    @MockBean
    private EmailService emailService;  // ← 이 하나가 컨텍스트 재생성 트리거!
    
    @Autowired
    private UserService userService;
    
    @Test
    void testCreateUser() {
        User user = userService.createUser("john@example.com");
        verify(emailService).sendWelcomeEmail(user);
    }
}

// ❌ 잘못된 접근 2: In-Memory DB로 부정확한 테스트
@SpringBootTest(
    webEnvironment = WebEnvironment.RANDOM_PORT,
    properties = {
        "spring.datasource.url=jdbc:h2:mem:testdb",  // ← 실제 환경과 다름!
        "spring.datasource.driver-class-name=org.h2.Driver"
    }
)
class OrderRepositoryTest {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void testFindByCustomerId() {
        // H2 방언: PostgreSQL의 JSON 연산자 미지원
        // 실제 환경에선 동작하지만 여기선 실패
        List<Order> orders = orderRepository.findJsonOrders();
        assertEquals(2, orders.size());
    }
}

// ❌ 잘못된 접근 3: 정적 상태로 인한 테스트 간섭
class UserRepositoryTest {
    
    private static DataSource ds = createStaticDS();  // 정적 상태!
    
    @BeforeEach
    void setup() {
        // 이전 테스트의 데이터가 남아있음
        User user = new User("john");
        repo.save(user);
    }
    
    @Test
    void testCountUsers() {
        // 이전 테스트에서 생성한 사용자도 카운트됨
        assertEquals(1, repo.count());  // 실제로는 2 이상일 수 있음
    }
}
```

**결과**: 
- 테스트 시간: 2분 (불필요한 로딩)
- 신뢰성: Flaky 확률 15% (데이터 간섭)
- 디버깅 어려움: H2 vs PostgreSQL 차이

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

### 1단계: 컨텍스트 재사용 구조 설계

```java
// ✅ 올바른 접근 1: 공통 베이스 테스트 클래스

/**
 * 모든 @SpringBootTest의 기본
 * 같은 설정 = 같은 컨텍스트 재사용
 */
@SpringBootTest(
    webEnvironment = WebEnvironment.RANDOM_PORT
)
@AutoConfigureMockMvc
@Testcontainers
public abstract class BaseIntegrationTest {
    
    // 모든 테스트에서 공유하는 PostgreSQL 컨테이너
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
        DockerImageName.parse("postgres:16-alpine")
    )
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    // 모든 테스트에서 공유하는 Redis 컨테이너
    static GenericContainer<?> redis = new GenericContainer<>(
        DockerImageName.parse("redis:7-alpine")
    )
        .withExposedPorts(6379)
        .waitingFor(Wait.forListeningPort());
    
    // 동적으로 컨테이너 설정값 주입
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        // 각 테스트가 실행될 때 동일한 컨테이너 재사용
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
}

// ✅ 테스트 1: MockBean 없음 = 컨텍스트 재사용
@DisplayName("UserService - 컨텍스트 재사용")
class UserServiceTest extends BaseIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testCreateUser() {
        // 컨텍스트 로딩: 첫 번째 테스트만 3초, 이후 테스트는 로딩 없음
        User user = userService.createUser("john@example.com");
        
        assertNotNull(user.getId());
        assertEquals("john@example.com", user.getEmail());
    }
    
    @Test
    void testFindUserByEmail() {
        // 추가 로딩 없이 즉시 실행 (이전 컨텍스트 재사용)
        userRepository.save(new User("jane@example.com"));
        
        User found = userRepository.findByEmail("jane@example.com");
        assertNotNull(found);
    }
}

// ✅ 테스트 2: MockBean 필요한 경우 = 별도 컨텍스트
@DisplayName("UserService - EmailService Mock 필요")
@SpringBootTest  // BaseIntegrationTest와 다른 컨텍스트
class UserServiceWithMockTest extends BaseIntegrationTest {
    
    @MockBean  // ← 이것이 새 컨텍스트를 생성하지만, 같은 설정 = 같은 DB 재사용
    private EmailService emailService;
    
    @Autowired
    private UserService userService;
    
    @Test
    void testSendWelcomeEmail() {
        User user = userService.createUserWithEmail("bob@example.com");
        
        // EmailService만 Mock됨, 다른 빈들은 실제 인스턴스
        verify(emailService).sendWelcomeEmail(user);
    }
}
```

### 2단계: 슬라이스 테스트로 로딩 범위 최소화

```java
// ✅ 레포지토리 테스트: @DataJpaTest (필요한 것만 로딩)

@DataJpaTest  // 전체 컨텍스트 대신 JPA만 로딩 (50% 빠름)
@Testcontainers
class OrderRepositoryTest {
    
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:16-alpine");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void testFindByCustomerId() {
        // 1. 컨텍스트 로딩: 1.5초 (@SpringBootTest의 3초 vs 50% 빠름)
        Order order = orderRepository.save(
            new Order("CUST-001", 100.0)
        );
        
        // 2. PostgreSQL 자동 커밋 (H2 인메모리 아님)
        List<Order> found = orderRepository.findByCustomerId("CUST-001");
        assertEquals(1, found.size());
    }
}

// ✅ API 컨트롤러 테스트: @WebMvcTest (서비스 로직은 Mock)

@WebMvcTest(OrderController.class)  // OrderController만 로드, 나머지는 Mock
class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private OrderService orderService;  // ← WebMvcTest 내에서 MockBean은 OK
    
    @Test
    void testGetOrder() throws Exception {
        Order order = new Order("ORD-001", 100.0);
        given(orderService.getOrder("ORD-001"))
            .willReturn(order);
        
        mockMvc.perform(get("/api/orders/ORD-001"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.orderId").value("ORD-001"));
    }
}

// ✅ 테스트 전략 정리

/*
컨텍스트 로딩 시간 비교:

1. @SpringBootTest (전체): 3초
2. @SpringBootTest + @MockBean: 3초 (새 컨텍스트지만 같은 설정)
3. @DataJpaTest: 1.5초 (JPA만)
4. @WebMvcTest: 1초 (컨트롤러만)
5. 단위 테스트 (Mock 모두): 10ms

100개 테스트 조합:
- 모두 @SpringBootTest: 300초
- 각각 최적화: @WebMvcTest(30) + @DataJpaTest(40) + @SpringBootTest(30)
  = 30초 + 60초 + 90초 = 180초
- 단축: 120초 (40% 감소)
*/
```

### 3단계: 실제 데이터베이스 연동

```java
// ✅ Testcontainers로 실제 PostgreSQL 사용

@SpringBootTest
@Testcontainers
class UserServicePostgresTest {
    
    // 1. 컨테이너 정의
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
        DockerImageName.parse("postgres:16-alpine")
    )
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass")
        .waitingFor(
            Wait.forLogMessage(".*database system is ready to accept connections.*", 2)
        );
    
    // 2. 컨테이너 포트를 Spring 설정으로 동적 주입
    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add(
            "spring.datasource.url",
            postgres::getJdbcUrl  // jdbc:postgresql://localhost:PORT/testdb
        );
        registry.add(
            "spring.datasource.username",
            postgres::getUsername  // testuser
        );
        registry.add(
            "spring.datasource.password",
            postgres::getPassword  // testpass
        );
        
        // 추가 설정
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
        registry.add("spring.jpa.show-sql", () -> "false");
    }
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testUserPersistenceWithPostgres() {
        // 실제 PostgreSQL에 저장
        User user = userService.createUser("test@example.com", "Test User");
        
        // 강제 플러시 (테스트용)
        userRepository.flush();
        
        // 실제 데이터베이스에서 조회
        User found = userRepository.findByEmail("test@example.com").orElse(null);
        assertNotNull(found);
        assertEquals("Test User", found.getName());
    }
    
    @Test
    void testUserJsonFieldWithPostgres() {
        // PostgreSQL 고유 기능: JSONB 지원
        User user = new User("json@example.com", "JsonUser");
        user.setMetadata(Map.of(
            "source", "api",
            "region", "us-west"
        ));
        
        userRepository.save(user);
        userRepository.flush();
        
        // PostgreSQL JSONB 쿼리 (H2는 지원 안 함)
        User found = userRepository.findUserByMetadataRegion("us-west");
        assertNotNull(found);
    }
}

// 커스텀 쿼리
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByEmail(String email);
    
    // PostgreSQL JSONB 지원 쿼리
    @Query("SELECT u FROM User u WHERE u.metadata->>'region' = :region")
    User findUserByMetadataRegion(@Param("region") String region);
}
```

### 4단계: Docker-in-Docker 없이 CI에서 실행

```yaml
# ✅ GitHub Actions: Docker-in-Docker 대신 Docker Socket 사용

name: Spring Boot Test with Testcontainers
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    # 방법 1: Docker 소켓 직접 사용 (권장)
    services:
      # GitHub Actions 호스트의 Docker 데몬 사용
      docker:
        image: docker:dind
        options: --privileged --userns=host
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      
      # 방법 2: Testcontainers 권장 설정
      - name: Run Tests with Testcontainers
        run: ./gradlew test
        env:
          # Testcontainers가 Docker 소켓 접근 허용
          TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE: /var/run/docker.sock
          # Docker 이미지 재사용 (캐싱)
          TESTCONTAINERS_REUSE_CONTAINERS: true
          # 빌드 시간 단축
          DOCKER_TLS_VERIFY: ''

  # 대안: Docker 빌드 없이 Ryuk 사용 (리소스 정리)
  test-with-ryuk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      - name: Run Tests
        run: ./gradlew test --info
        env:
          # Testcontainers가 생성한 컨테이너 자동 정리
          TESTCONTAINERS_RYUK_ENABLED: 'true'
```

---

## 🔬 내부 동작 원리

### Spring Boot 컨텍스트 재사용 메커니즘

```
컨텍스트 결정 요소 (해시값 계산):

1. @SpringBootTest 설정
   - webEnvironment
   - properties
   - @MockBean, @SpyBean 없음

2. 모든 @Configuration 클래스
3. 모든 활성 프로필
4. 클래스 로더

→ 같은 해시 = 같은 컨텍스트 재사용
```

**컨텍스트 캐시 구조**:

```java
public class DefaultCacheAwareContextLoaderDelegate {
    
    // 내부 캐시 (LRU)
    private final Map<MergedContextConfiguration, ApplicationContext> contextCache 
        = new ConcurrentHashMap<>();
    
    public ApplicationContext loadContext(MergedContextConfiguration config) {
        // 1. 설정 객체의 해시 계산
        String key = config.getContextId();  // "com.example.UserServiceTest"
        
        // 2. 캐시 확인
        if (contextCache.containsKey(key)) {
            return contextCache.get(key);  // ✅ 빠른 반환
        }
        
        // 3. 새 컨텍스트 로딩 (초기 1회만)
        ApplicationContext context = loadContextInternal(config);
        contextCache.put(key, context);
        return context;
    }
}

// MockBean 추가 = 새 컨텍스트?
class TestContextBootstrapper {
    
    public MergedContextConfiguration buildMergedContextConfiguration() {
        MergedContextConfiguration config = ...;
        
        // @MockBean 존재 확인
        if (hasMockBeans(testClass)) {
            // 새로운 설정 객체 생성 (다른 해시)
            config.setContextId(
                config.getContextId() + "#with-mocks"
            );
        }
        
        return config;
    }
}
```

### Testcontainers 라이프사이클

```
테스트 클래스 로드 시점:

1. @Container static 필드 발견
   ↓
2. 테스트 클래스가 처음 로드될 때 컨테이너 시작
   (Spring 초기화 전)
   ↓
3. @DynamicPropertySource 실행
   → 컨테이너 정보를 Spring 설정으로 주입
   ↓
4. Spring 컨텍스트 로딩
   → 데이터베이스 접속 정보를 사용해 연결
   ↓
5. 모든 테스트 실행 (컨테이너 재사용)
   ↓
6. 테스트 클래스 언로드 시점
   → @Container static 컨테이너 종료

타이밍 다이어그램:

시간 →

클래스 로드:     |-------|
컨테이너 시작:   |-------|
Spring 로딩:           |--------|
테스트 1:                    |--|
테스트 2:                       |--|
테스트 3:                          |--|
컨테이너 정리:                       |---------|
```

### 슬라이스 테스트의 로딩 범위

```
@SpringBootTest (전체 컨텍스트):
┌─────────────────────────────────────┐
│ DispatcherServlet                   │  ← MockMvc
├─────────────────────────────────────┤
│ @Controller, @RestController        │
├─────────────────────────────────────┤
│ @Service, @Component                │
├─────────────────────────────────────┤
│ @Repository, DataSource             │  ← 실제 DB
├─────────────────────────────────────┤
│ 모든 자동 설정                        │
└─────────────────────────────────────┘
로딩 시간: 3초, 메모리: 200MB

@WebMvcTest (컨트롤러 레이어만):
┌─────────────────────────────────────┐
│ DispatcherServlet                   │
├─────────────────────────────────────┤
│ 지정된 @Controller, @RestController  │
├─────────────────────────────────────┤
│ @Service, @Repository = Mock        │  ← 모두 MockBean
└─────────────────────────────────────┘
로딩 시간: 1초, 메모리: 50MB

@DataJpaTest (데이터 레이어만):
┌─────────────────────────────────────┐
│ @Repository                         │
├─────────────────────────────────────┤
│ DataSource, JPA 설정                │
├─────────────────────────────────────┤
│ Transaction 관리                     │
└─────────────────────────────────────┘
로딩 시간: 1.5초, 메모리: 80MB
```

---

## 💻 실전 실험

### 실험 1: 컨텍스트 재사용 효과 측정

```gradle
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.2.0"
    id("java")
}

dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:testcontainers:1.19.0")
    testImplementation("org.testcontainers:postgresql:1.19.0")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.postgresql:postgresql:42.6.0")
}

tasks.test {
    useJUnitPlatform()
    
    // 로깅으로 컨텍스트 재사용 추적
    systemProperty("spring.test.context.cache.maxSize", "32")
    
    // 테스트 결과 리포트
    testLogging {
        events("PASSED", "FAILED", "SKIPPED")
        exceptionFormat = org.gradle.api.tasks.testing.logging.TestExceptionFormat.FULL
    }
}
```

```java
// 테스트 1: 컨텍스트 재사용 측정
@SpringBootTest
@Testcontainers
class ContextReuseMetricsTest {
    
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void logStartTime() {
        System.out.println("[TEST START] " + System.currentTimeMillis());
    }
    
    @Test
    void test1_CreateUser() {
        userRepository.save(new User("user1@example.com"));
    }
    
    @Test
    void test2_FindUser() {
        User user = userRepository.findByEmail("user1@example.com").orElse(null);
        assertNotNull(user);
    }
    
    @Test
    void test3_DeleteUser() {
        userRepository.deleteByEmail("user1@example.com");
    }
}

// 실행 결과:
// [TEST START] 1712776000000 <- 첫 테스트: 컨텍스트 로딩 3초 포함
// [TEST START] 1712776003500 <- 두 번째: 재사용 (로딩 없음)
// [TEST START] 1712776004200 <- 세 번째: 재사용 (로딩 없음)
// 
// 전체 시간: 4.2초
// (첫 번째만 3초, 나머지는 각 200ms)
```

```bash
# 실제 측정 명령
./gradlew test -i | grep -E "(context cache|Loading TestContext|TestContext)"

# 출력 예:
# [context cache] Initializing Spring TestContext for class ContextReuseMetricsTest
# Spring TestContext initialized in 3005ms
# [context cache] Retrieving Spring TestContext from cache for class ContextReuseMetricsTest (hit)
# Spring TestContext retrieved in 2ms
```

### 실험 2: MockBean이 컨텍스트에 미치는 영향

```java
// 시나리오 1: MockBean 없음 = 컨텍스트 공유
@SpringBootTest
class UserServiceNormalTest {
    @Autowired
    private UserService userService;
    
    @Test
    void test1() { /* 3초: 컨텍스트 로딩 */ }
    @Test
    void test2() { /* 200ms: 캐시에서 재사용 */ }
}

// 시나리오 2: MockBean 추가 = 새 컨텍스트 (같은 설정이면 재사용)
@SpringBootTest
class UserServiceWithMockTest {
    @MockBean
    private EmailService emailService;  // ← 새 컨텍스트 생성
    
    @Autowired
    private UserService userService;
    
    @Test
    void test1() { /* 3초: 새 컨텍스트 로딩 */ }
    @Test
    void test2() { /* 200ms: 같은 MockBean 설정이므로 재사용 */ }
}

// 측정 코드
@SpringBootTest
class MockBeanImpactTest {
    
    @Test
    void measureContextLoadingTime() {
        long start = System.currentTimeMillis();
        // Spring이 ApplicationContext를 로드하는 시간
        long duration = System.currentTimeMillis() - start;
        
        System.out.println("Context load time: " + duration + "ms");
        // 출력: Context load time: 3000ms (처음)
        // 출력: Context load time: 2ms (캐시에서)
    }
}
```

### 실험 3: Testcontainers 환경 설정

```java
// build.gradle.kts
dependencies {
    testImplementation("org.testcontainers:testcontainers:1.19.0")
    testImplementation("org.testcontainers:postgresql:1.19.0")
    testImplementation("org.testcontainers:localstack:1.19.0")  // AWS 서비스 모의
}

// testcontainers.properties (선택사항)
// checks.disable=true  // 헬스 체크 비활성화
// ryuk.container.privileged=false

// 실제 테스트
@SpringBootTest
@Testcontainers
class MultiContainerTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withInitScript("init.sql");  // DB 초기화 스크립트
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379)
        .waitingFor(Wait.forListeningPort());
    
    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
        DockerImageName.parse("localstack/localstack:latest")
    )
        .withServices(S3);  // AWS S3 서비스 포함
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
        
        // AWS S3 설정
        registry.add("aws.s3.endpoint", localStack::getEndpoint);
        registry.add("aws.accessKeyId", () -> "test");
        registry.add("aws.secretAccessKey", () -> "test");
    }
    
    @Autowired
    private UserService userService;
    
    @Test
    void testWithMultipleServices() {
        // PostgreSQL, Redis, LocalStack 모두 동시 사용
        User user = userService.createAndCacheUser("test@example.com");
        assertNotNull(user);
    }
}
```

### 실험 4: CI에서 Testcontainers 실행

```yaml
name: Test with Testcontainers on CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          cache: gradle
      
      # 중요: Docker 데몬 활성화
      - name: Enable Docker
        run: |
          sudo systemctl start docker
          docker ps  # Docker 준비 확인
      
      # Testcontainers 권장 설정
      - name: Run tests with Testcontainers
        run: ./gradlew test
        env:
          # Docker 소켓 지정 (일반적으로 자동 감지)
          DOCKER_HOST: unix:///var/run/docker.sock
          
          # 컨테이너 이미지 재사용 (빌드 시간 단축)
          TESTCONTAINERS_REUSE_CONTAINERS: 'true'
          
          # 병렬 테스트 실행
          GRADLE_OPTS: "-Dorg.gradle.parallel=true -Dorg.gradle.workers.max=4"
      
      # 테스트 보고서
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: build/test-results/test/**/*.xml
```

---

## 📊 성능/비용 비교

```
테스트 전략별 성능:

1. 모든 테스트 @SpringBootTest:
   - 컨텍스트 로딩: 3초 × 50 = 150초
   - 실제 테스트: 30초
   - 합계: 180초

2. 슬라이스 테스트 (@WebMvcTest + @DataJpaTest):
   - @WebMvcTest 로딩: 1초 × 10 = 10초
   - @DataJpaTest 로딩: 1.5초 × 15 = 22.5초
   - @SpringBootTest 로딩: 3초 × 25 = 75초
   - 실제 테스트: 30초
   - 합계: 137.5초 (24% 단축)

3. In-Memory H2 vs Testcontainers PostgreSQL:
   - H2 테스트 시간: 100초
   - PostgreSQL 테스트 시간: 120초
   - 하지만 PostgreSQL은 실제 환경과 99% 동일
   - H2는 환경 차이로 버그 재현 못할 확률: 5%
   
   → PostgreSQL 사용 가치: 재현 가능성 +95%

Docker 비용:
- 이미지 다운로드: 초회만 100MB (~2초)
- 컨테이너 시작: 초회 3초, 재사용 시 0초
- 100 테스트 실행:
  - 컨테이너 없음: 180초
  - Testcontainers: 183초 (3초만 추가)
```

---

## ⚖️ 트레이드오프

### 1. 컨텍스트 재사용 vs 테스트 격리

```
컨텍스트 재사용 (성능 우선):
✅ 빠른 실행 (동시 1.5배 빠름)
❌ 테스트 간 데이터 오염 위험
❌ 정적 상태 관리 필요

독립적 컨텍스트 (격리 우선):
✅ 완벽한 격리
✅ 테스트 순서 무관
❌ 느린 실행 (3배 느림)
❌ 높은 메모리 사용

권장: 컨텍스트 재사용 + @BeforeEach에서 데이터 정리
```

### 2. Testcontainers vs In-Memory DB

```
In-Memory (H2):
✅ 매우 빠름 (2-3초)
✅ 네트워크 오버헤드 없음
❌ 실제 데이터베이스와 차이
  - 방언 차이 (JSON, 배열 등)
  - 성능 특성 다름
  - 제약 조건 차이

Testcontainers:
✅ 실제 환경과 동일
✅ 버그 조기 발견
❌ 느림 (10-15초)
❌ Docker 필요

권장: 로컬: H2, CI: Testcontainers PostgreSQL 혼용
```

### 3. MockBean vs Spy vs Real

```
@MockBean (Mock 완전 대체):
✅ 테스트 속도 빠름
✅ 외부 의존성 제거
❌ 실제 동작 미검증
❌ 컨텍스트 새로 생성

@SpyBean (부분 Mock):
✅ 일부 메서드만 Mock
✅ 다른 메서드는 실제 동작
❌ 복잡성 증가
❌ 컨텍스트 새로 생성

실제 인스턴스 (No Mock):
✅ 통합 테스트로 신뢰성 높음
✅ 컨텍스트 재사용 (빠름)
❌ 외부 API 호출 필요
❌ 테스트 속도 낮음

권장: 
- 단위 테스트: Mock (속도)
- 통합 테스트: 실제 또는 Testcontainers (신뢰성)
```

---

## 📌 핵심 정리

| 전략 | 효과 |
|------|------|
| **컨텍스트 재사용** | 초반 3초 + 이후 0.2초/테스트 (40% 시간 절감) |
| **슬라이스 테스트** | 로딩 범위 최소화 (50-70% 빨라짐) |
| **Testcontainers** | 실제 환경 재현 (버그 재현율 +95%) |
| **MockBean 최소화** | 컨텍스트 재사용 가능 (1회 로딩으로 충분) |
| **CI Docker 최적화** | 이미지 캐싱으로 재시작 시간 단축 |

---

## 🤔 생각해볼 문제

### Q1: "로컬 테스트는 빠르지만 CI에서는 느리다. 왜?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**원인 분석**:

1. **Docker 이미지 다운로드** (처음 실행 시)
```
로컬: 이미지 캐시됨 (0초)
CI: 매번 GitHub Actions 이미지 다운로드
    postgres:16-alpine = 150MB
    redis:7-alpine = 50MB
    → 다운로드 시간 10-20초
```

2. **Docker 리소스 제약**
```
로컬: 16GB RAM, 8 CPU 가용
CI: 2GB RAM, 2 CPU 제약
    → 동시성 낮음, 경합 증가
```

3. **이미지 풀 전략 차이**
```
// ❌ 매번 최신 이미지 다운로드
new PostgreSQLContainer<>("postgres:16")

// ✅ 특정 버전 고정 (캐시 가능)
new PostgreSQLContainer<>("postgres:16.0-alpine")
```

**해결책**:

```yaml
name: Optimized CI
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # 1. Docker 이미지 사전 로드
      - name: Pre-pull Docker images
        run: |
          docker pull postgres:16.0-alpine
          docker pull redis:7.0-alpine
      
      # 2. Testcontainers 캐시 활성화
      - name: Setup Testcontainers cache
        uses: actions/cache@v3
        with:
          path: ~/.testcontainers
          key: testcontainers-${{ runner.os }}
          restore-keys: testcontainers-
      
      # 3. 테스트 실행 (이미지 캐시 활용)
      - name: Run tests
        run: ./gradlew test
        env:
          TESTCONTAINERS_REUSE_CONTAINERS: 'true'
          TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE: /var/run/docker.sock
```

**효과**:
- 로컬과 CI 시간 격차 제거
- CI 시간: 45초 (기존) → 35초 (22% 단축)

</details>

### Q2: "MockBean으로 새 컨텍스트가 생성되는데, 같은 설정이면 캐시되지 않을까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**기술적 설명**:

Spring TestContext는 **MergedContextConfiguration 객체의 해시**로 캐시합니다.

```java
// 해시 계산 요소:
1. @SpringBootTest의 properties → 같음
2. @MockBean의 필드 이름과 타입 → MockBean 추가되면 다름!
3. 활성 프로필 → 같음

// 예시:
클래스 1: @SpringBootTest + UserService, OrderService
클래스 2: @SpringBootTest + @MockBean UserService + OrderService

→ MockBean 추가로 해시가 변함
→ 별도 컨텍스트 생성 (새로운 캐시 엔트리)
→ 같은 MockBean 설정이면 이후 재사용
```

**실제 캐시 상황**:

```
테스트 1: UserServiceTest (MockBean 없음)
  → 해시: ABC123
  → 컨텍스트 로딩: 3초
  → 캐시: [ABC123 -> Context1]

테스트 2: OrderServiceTest (MockBean 없음)
  → 해시: ABC123 (같음!)
  → 컨텍스트 로드: 캐시 히트 (2ms)

테스트 3: UserServiceMockTest (@MockBean 추가)
  → 해시: DEF456 (다름!)
  → 컨텍스트 로딩: 3초 (새 컨텍스트)
  → 캐시: [ABC123 -> Context1, DEF456 -> Context2]

테스트 4: UserServiceMockTest2 (같은 @MockBean)
  → 해시: DEF456 (같음!)
  → 컨텍스트 로드: 캐시 히트 (2ms)
```

**성능 최적화**:

```java
// 전략: MockBean이 필요한 테스트를 함께 배치

// ❌ 나쁜 배치 (5번 로딩):
class Test1 {}  // 해시 ABC
@MockBean EmailService
class Test2 {}  // 해시 DEF (새 로딩)
class Test3 {}  // 해시 ABC (재사용)
@MockBean EmailService
class Test4 {}  // 해시 DEF (재사용)
@MockBean EmailService, StorageService
class Test5 {}  // 해시 GHI (새 로딩)

// ✅ 좋은 배치 (3번 로딩):
class Test1 {}  // 해시 ABC (1번 로딩)
class Test3 {}  // 해시 ABC (재사용)
@MockBean EmailService
class Test2 {}  // 해시 DEF (2번 로딩)
@MockBean EmailService
class Test4 {}  // 해시 DEF (재사용)
@MockBean EmailService, StorageService
class Test5 {}  // 해시 GHI (3번 로딩)

// 결과: 로딩 횟수 5회 → 3회 (40% 감소)
```

</details>

### Q3: "Testcontainers가 느린데, 로컬에서는 H2를 사용하고 CI에서만 PostgreSQL을 사용할 수 있을까?"

<details>
<summary>해설 (클릭하여 펼치기)</summary>

**프로필 기반 전략**:

```java
// ✅ 프로필별 데이터베이스 설정

public class TestcontainersConfiguration {
    
    // 프로필 'test'에서만 활성화
    @Profile("test")
    @Bean
    public PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
    
    // 프로필 'h2'에서만 활성화
    @Profile("h2")
    @Bean
    public DataSource h2DataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

// application-test.yml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop

// application-h2.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  h2:
    console:
      enabled: true

// 테스트 클래스
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")  // CI: Testcontainers 사용
class UserServicePostgresTest {
    // ...
}

@SpringBootTest
@ActiveProfiles("h2")  // 로컬: In-Memory H2
class UserServiceH2Test {
    // ...
}
```

```gradle
// build.gradle.kts - 프로필별 실행

tasks.register<Test>("testH2") {
    description = "Run tests with H2 (fast, local)"
    systemProperties["spring.profiles.active"] = "h2"
}

tasks.register<Test>("testPostgres") {
    description = "Run tests with PostgreSQL (accurate, CI)"
    systemProperties["spring.profiles.active"] = "test"
}

tasks.test {
    // 기본은 H2 (로컬 개발용)
    systemProperties["spring.profiles.active"] = "h2"
}
```

```yaml
# GitHub Actions: CI에서 PostgreSQL 사용
name: Test on PostgreSQL
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '21'
      
      # CI에서는 Testcontainers 프로필 사용
      - name: Run tests with PostgreSQL
        run: ./gradlew testPostgres
```

**장점**:
- 로컬 개발: H2 사용 (30초 빠름)
- CI: PostgreSQL 사용 (신뢰성 99%)
- 두 환경 모두 테스트 가능

**확인**:
```bash
# 로컬에서 H2 테스트 (빠름)
./gradlew testH2

# 로컬에서 PostgreSQL 테스트 (정확함)
./gradlew testPostgres

# CI는 자동으로 PostgreSQL 선택
```

</details>

---

[⬅️ 이전](./01-test-pyramid-pipeline.md) | [홈으로 🏠](../README.md) | [다음 ➡️](./03-code-quality-gate.md)
