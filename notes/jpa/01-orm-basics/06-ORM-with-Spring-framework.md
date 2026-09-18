Pom.xml:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.quizstakes</groupId>
    <artifactId>quizstakes-spring-plain</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <spring.version>6.1.13</spring.version>
        <hibernate.version>6.6.2.Final</hibernate.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-orm</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>${hibernate.version}</version>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <version>2.2.224</version>
        </dependency>
    </dependencies>
</project>
```

Client entity:
```java
@Entity
@Getter @Setter @NoArgsConstructor
@Table(name = "client")
public class Client {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @OneToMany(mappedBy = "client", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Account> accounts = new ArrayList<>();

    public void addAccount(Account account) {
        accounts.add(account);
        account.setClient(this);
    }
}
```

Account entity:
```java
@Entity
@Getter @Setter @NoArgsConstructor
@Table(name = "account")
public class Account {

    public enum AccountType { IGCFD, IGSTK }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(name = "account_type", nullable = false)
    private AccountType accountType;

    @Column(nullable = false)
    private BigDecimal balance;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "client_id", nullable = false)
    private Client client;
}
```

App config class: 
```java
@Configuration
@ComponentScan(basePackages = "com.quizstakes")
@EnableTransactionManagement
public class AppConfig {

    // --- step 2: the pool, wired by hand instead of via application.yml ---
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:quizstakes;DB_CLOSE_DELAY=-1");
        config.setUsername("sa");
        config.setPassword("");
        config.setMaximumPoolSize(10);
        return new HikariDataSource(config); // pool fills here, at bean creation
    }

    // --- Hibernate as a JPA provider, wired manually ---
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean emf = new LocalContainerEntityManagerFactoryBean();
        emf.setDataSource(dataSource);
        emf.setPackagesToScan("com.quizstakes.domain"); // holds client and account entity classes

        JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        emf.setJpaVendorAdapter(vendorAdapter);
        emf.setPersistenceProvider(new HibernatePersistenceProvider());

        Properties props = new Properties();
        props.put("hibernate.hbm2ddl.auto", "update");
        props.put("hibernate.show_sql", "true");
        emf.setJpaProperties(props);

        return emf;
    }

    // --- step 4/6: transaction manager backing @Transactional ---
    @Bean
    public PlatformTransactionManager transactionManager(
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }
}
```

Client onboarding service: 
```java
@Service
public class ClientOnboardingService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public Client onboardClient(String name, String email) {
        Client client = new Client(name, email);
        client.addAccount(new Account(Account.AccountType.IGCFD, BigDecimal.ZERO));
        client.addAccount(new Account(Account.AccountType.IGSTK, BigDecimal.ZERO));
        entityManager.persist(client); // transient -> managed; INSERT queued, flushed at commit
        return client;
    }
}
```

Main class:
```java
public class Main {

    public static void main(String[] args) {
        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {

            ClientOnboardingService service = ctx.getBean(ClientOnboardingService.class);
            Client client = service.onboardClient("Rajat", "rajat@example.com");

            System.out.println("Saved client id=" + client.getId()
                    + " with " + client.getAccounts().size() + " accounts");
        }
    }
}
```

Note:
- The HikariDataSource bean construction is the pool-fill moment; 
- @EnableTransactionManagement is the one line that makes @Transactional do anything at all (Boot enables this for you silently); 
- and the EntityManagerFactory/JpaTransactionManager pairing is exactly what a Spring Boot auto-configuration class builds behind the scenes.
