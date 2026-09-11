pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/>
    </parent>

    <groupId>com.quizstakes</groupId>
    <artifactId>quizstakes-boot</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

application.yml
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:quizstakes;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
    hikari:
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    open-in-view: false
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
package com.quizstakes.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;

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

Client repository class: 
```java
public interface ClientRepository extends JpaRepository<Client, Long> {
}
```

Account repository class:
```java
public interface AccountRepository extends JpaRepository<Account, Long> {
}
```

Client onboarding service class: 
```java
@Service
public class ClientOnboardingService {

    private final ClientRepository clientRepository;

    public ClientOnboardingService(ClientRepository clientRepository) {
        this.clientRepository = clientRepository;
    }

    @Transactional
    public Client onboardClient(String name, String email) {
        Client client = new Client(name, email);
        client.addAccount(new Account(Account.AccountType.IGCFD, BigDecimal.ZERO));
        client.addAccount(new Account(Account.AccountType.IGSTK, BigDecimal.ZERO));
        return clientRepository.save(client); // cascades INSERT to both accounts
    }
}
```

Application class: 
```java
@SpringBootApplication
public class QuizStakesBootApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(QuizStakesBootApplication.class, args);

        ClientOnboardingService service = ctx.getBean(ClientOnboardingService.class);
        Client client = service.onboardClient("Rajat", "rajat@example.com");

        System.out.println("Saved client id=" + client.getId()
                + " with " + client.getAccounts().size() + " accounts");

        ctx.close();
    }
}
```
