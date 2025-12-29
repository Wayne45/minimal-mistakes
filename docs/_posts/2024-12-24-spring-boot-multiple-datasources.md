---
title: "多資料來源（Oracle + PostgreSQL）的 Spring Boot 設計要點"
excerpt: "當系統同時連接多種資料庫時，資料來源切換、交易管理與 Flyway Migration 都需要特別處理。"
categories:
  - Backend
  - Database
tags:
  - Spring Boot
  - Oracle
  - PostgreSQL
  - Multiple Datasources
toc: true
---

## 前言

當系統同時連接多種資料庫時，資料來源切換、交易管理與 Flyway Migration 都需要特別處理。本文分享實際踩雷與解法。

## 常見問題

1. Bean 衝突：多個 DataSource 無法自動注入
2. Transaction 管理混亂
3. Flyway Migration 只跑一個資料庫
4. JPA Repository 無法區分資料來源

## 解決方案

### 1. 資料來源配置

```yaml
spring:
  datasource:
    oracle:
      url: jdbc:oracle:thin:@localhost:1521:orcl
      username: user
      password: pass
      driver-class-name: oracle.jdbc.OracleDriver
    postgres:
      url: jdbc:postgresql://localhost:5432/mydb
      username: user
      password: pass
      driver-class-name: org.postgresql.Driver
```

### 2. DataSource Bean 配置

```java
@Configuration
public class OracleDataSourceConfig {

    @Primary
    @Bean(name = "oracleDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.oracle")
    public DataSource oracleDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "oracleEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean oracleEntityManagerFactory(
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
    public PlatformTransactionManager oracleTransactionManager(
            @Qualifier("oracleEntityManagerFactory") EntityManagerFactory factory) {
        return new JpaTransactionManager(factory);
    }
}
```

### 3. Flyway 多資料庫 Migration

```java
@Configuration
public class FlywayConfig {

    @Bean
    public Flyway oracleFlyway(@Qualifier("oracleDataSource") DataSource dataSource) {
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/oracle")
            .baselineOnMigrate(true)
            .load();
        flyway.migrate();
        return flyway;
    }

    @Bean
    public Flyway postgresFlyway(@Qualifier("postgresDataSource") DataSource dataSource) {
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/postgres")
            .baselineOnMigrate(true)
            .load();
        flyway.migrate();
        return flyway;
    }
}
```

### 4. Repository 區分

```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.oracle.repository",
    entityManagerFactoryRef = "oracleEntityManagerFactory",
    transactionManagerRef = "oracleTransactionManager"
)
public class OracleRepositoryConfig {}

@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.postgres.repository",
    entityManagerFactoryRef = "postgresEntityManagerFactory",
    transactionManagerRef = "postgresTransactionManager"
)
public class PostgresRepositoryConfig {}
```

## 踩雷經驗

1. **忘記 @Primary**：Spring 無法決定預設 DataSource
2. **Package 路徑錯誤**：Entity 掃描不到
3. **Transaction 跨資料庫**：需要 JTA 或 Saga 模式

## 結論

多資料來源配置需要清楚區分每個元件的職責，善用 @Qualifier 和 @Primary 註解可以有效避免衝突。
