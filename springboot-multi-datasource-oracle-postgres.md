# Integrating PostgreSQL in a Spring Boot Application with Existing Oracle DB

---

## Part 1: Integration Setup

### 1. Add PostgreSQL Dependency

In `pom.xml`:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

### 2. Configure Both Datasources in `application.yml`

```yaml
# Oracle (primary)
oracle:
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/ORCL
    username: oracle_user
    password: oracle_pass
    driver-class-name: oracle.jdbc.OracleDriver

# PostgreSQL (secondary)
postgres:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: pg_user
    password: pg_pass
    driver-class-name: org.postgresql.Driver
```

---

### 3. Create Separate DataSource Beans

**Oracle config (primary):**

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.example.oracle.repository",
    entityManagerFactoryRef = "oracleEntityManagerFactory",
    transactionManagerRef = "oracleTransactionManager"
)
public class OracleDataSourceConfig {

    @Primary
    @Bean(name = "oracleDataSource")
    @ConfigurationProperties(prefix = "oracle.datasource")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "oracleEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("oracleDataSource") DataSource dataSource) {
        return builder
            .dataSource(dataSource)
            .packages("com.example.oracle.entity")
            .persistenceUnit("oracle")
            .build();
    }

    @Primary
    @Bean(name = "oracleTransactionManager")
    public PlatformTransactionManager transactionManager(
            @Qualifier("oracleEntityManagerFactory") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```

**PostgreSQL config (secondary):**

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.example.postgres.repository",
    entityManagerFactoryRef = "postgresEntityManagerFactory",
    transactionManagerRef = "postgresTransactionManager"
)
public class PostgresDataSourceConfig {

    @Bean(name = "postgresDataSource")
    @ConfigurationProperties(prefix = "postgres.datasource")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "postgresEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("postgresDataSource") DataSource dataSource) {
        return builder
            .dataSource(dataSource)
            .packages("com.example.postgres.entity")
            .persistenceUnit("postgres")
            .build();
    }

    @Bean(name = "postgresTransactionManager")
    public PlatformTransactionManager transactionManager(
            @Qualifier("postgresEntityManagerFactory") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```

---

### 4. Organize Package Structure

```
com.example
├── oracle
│   ├── entity/       ← Oracle JPA entities
│   └── repository/   ← Oracle repositories
└── postgres
    ├── entity/       ← PostgreSQL JPA entities
    └── repository/   ← PostgreSQL repositories
```

Each entity class maps to its respective DB via the package scanning configured above.

---

### 5. Disable Spring Boot's Auto-Configuration

Since datasources are defined manually, exclude auto-config to avoid conflicts:

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application { ... }
```

Or in `application.yml`:

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
      - org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

---

### 6. Dialect Configuration (Optional but Recommended)

Add dialect hints per datasource in your `entityManagerFactory` builder:

```java
Map<String, Object> props = new HashMap<>();
props.put("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect");

return builder
    .dataSource(dataSource)
    .packages("com.example.postgres.entity")
    .persistenceUnit("postgres")
    .properties(props)
    .build();
```

---

### Key Rules to Remember

| Concern | Oracle | PostgreSQL |
|---|---|---|
| `@Primary` annotation | Yes | No |
| Separate packages | Required | Required |
| Transaction manager | `oracleTransactionManager` | `postgresTransactionManager` |
| Use `@Transactional` | Defaults to primary | Must specify `@Transactional("postgresTransactionManager")` |

---

## Part 2: Testing Strategy

### Overview

Both approaches are needed — **test each datasource individually first, then test together**.

---

### Phase 1: Test Each Datasource Individually

#### Test Oracle (existing — regression check)

```java
@SpringBootTest
@Transactional("oracleTransactionManager")
class OracleRepositoryTest {

    @Autowired
    OracleUserRepository oracleUserRepository;

    @Test
    void testOracleSave() {
        OracleUser user = new OracleUser("Alice");
        OracleUser saved = oracleUserRepository.save(user);
        assertNotNull(saved.getId());
    }

    @Test
    void testOracleFetch() {
        List<OracleUser> users = oracleUserRepository.findAll();
        assertFalse(users.isEmpty());
    }
}
```

#### Test PostgreSQL (new datasource)

```java
@SpringBootTest
@Transactional("postgresTransactionManager")
class PostgresRepositoryTest {

    @Autowired
    PostgresOrderRepository postgresOrderRepository;

    @Test
    void testPostgresSave() {
        PostgresOrder order = new PostgresOrder("ORDER-001");
        PostgresOrder saved = postgresOrderRepository.save(order);
        assertNotNull(saved.getId());
    }

    @Test
    void testPostgresFetch() {
        List<PostgresOrder> orders = postgresOrderRepository.findAll();
        assertFalse(orders.isEmpty());
    }
}
```

---

### Phase 2: Test Both Together (Cross-DB Integration)

Tests real-world scenarios where a single service reads from one DB and writes to another.

```java
@SpringBootTest
class CrossDatabaseServiceTest {

    @Autowired
    OrderSyncService orderSyncService; // service that spans both DBs

    @Test
    void testSyncFromOracleToPostgres() {
        // Fetch from Oracle, write to Postgres
        orderSyncService.syncOrders();

        // Verify in Postgres
        List<PostgresOrder> synced = postgresOrderRepository.findAll();
        assertFalse(synced.isEmpty());
    }
}
```

---

### Phase 3: Use Testcontainers for Isolated Tests

Avoid depending on real DB servers in CI. Use **Testcontainers**:

#### Dependencies

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>oracle-xe</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

#### Test with Testcontainers

```java
@SpringBootTest
@Testcontainers
class MultiDatasourceIntegrationTest {

    @Container
    static OracleContainer oracle = new OracleContainer("gvenzl/oracle-xe:21-slim");

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        // Override Oracle props
        registry.add("oracle.datasource.url", oracle::getJdbcUrl);
        registry.add("oracle.datasource.username", oracle::getUsername);
        registry.add("oracle.datasource.password", oracle::getPassword);

        // Override Postgres props
        registry.add("postgres.datasource.url", postgres::getJdbcUrl);
        registry.add("postgres.datasource.username", postgres::getUsername);
        registry.add("postgres.datasource.password", postgres::getPassword);
    }

    @Test
    void testBothDatasourcesUp() {
        assertTrue(oracle.isRunning());
        assertTrue(postgres.isRunning());
    }
}
```

---

### Phase 4: Test Transaction Isolation

Verify that a failure in one DB does **not** affect the other:

```java
@Test
void testOracleFailureDoesNotRollbackPostgres() {
    assertThrows(Exception.class, () -> {
        orderSyncService.syncWithForcedOracleFailure();
        // Oracle write fails, but Postgres write should still commit
    });

    // Postgres data should still be present
    List<PostgresOrder> orders = postgresOrderRepository.findAll();
    assertFalse(orders.isEmpty());
}
```

> **Note:** True distributed transactions (XA) require a JTA provider like Atomikos. Without it, each datasource manages its own transaction independently.

---

### Recommended Test Execution Order

```
┌─────────────────────────────────────────┐
│  1. Unit test Oracle repo (mocked)      │  ← Fast, no DB needed
│  2. Unit test Postgres repo (mocked)    │  ← Fast, no DB needed
│  3. Integration test Oracle alone       │  ← Real or Testcontainer DB
│  4. Integration test Postgres alone     │  ← Real or Testcontainer DB
│  5. Cross-DB integration test           │  ← Both DBs running
│  6. Transaction isolation test          │  ← Verify no cross-DB side effects
└─────────────────────────────────────────┘
```

---

### Quick Checklist Before Running Tests

| Check | What to verify |
|---|---|
| Connection pools | Both datasources connect successfully on startup |
| Schema migration | Liquibase/Flyway runs separately per DB |
| Entity scanning | No entity picked up by wrong datasource config |
| `@Transactional` | Correct transaction manager referenced per service |
| Rollback behavior | Each DB rolls back independently on failure |

---

## Summary

- Always test each datasource **individually first** to isolate issues.
- Then test **both together** to catch cross-DB interaction bugs.
- Use **Testcontainers** for clean, reproducible CI test environments.
- Without XA/JTA, each datasource handles its own transaction — design and test accordingly.
