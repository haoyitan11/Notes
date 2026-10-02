# Hibernate / Spring Data JPA Dependencies Handle

## Overview

This document provides a complete guide for the Hibernate ORM integration in the application using **Spring Data JPA with Hibernate as the underlying JPA provider**.

It covers four layers:

1. **Core Infrastructure** - Properties → Auto-Configuration (shared infrastructure)
2. **Entity Layer** - JPA entities with `javax.persistence` annotations
3. **Repository Layer** - Spring Data JPA repositories (CRUD operations)
4. **Service Layer** - Business-level data access with repository delegation

---

# Common Foundation: Database Infrastructure

This section covers the shared infrastructure used by all JPA/Hibernate operations.

---

## Part 1: Properties Configuration

### Purpose

Defines database connection and Hibernate JPA settings that Spring Boot auto-configuration will use.

### Source Reference

**File:** `config/application.properties`

### Configuration

```properties
# ═══════════════════════════════════════════════════════════════════════════════
# DATABASE CONNECTION
# ═══════════════════════════════════════════════════════════════════════════════
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/pwap_core?autoReconnect=true&serverTimezone=UTC&useSSL=false
spring.datasource.username=root
spring.datasource.password=test

# ═══════════════════════════════════════════════════════════════════════════════
# HIBERNATE / JPA CONFIGURATION
# ═══════════════════════════════════════════════════════════════════════════════
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect

# ═══════════════════════════════════════════════════════════════════════════════
# CONNECTION POOL VALIDATION (Tomcat JDBC Pool)
# ═══════════════════════════════════════════════════════════════════════════════
spring.datasource.tomcat.test-while-idle=true
spring.datasource.tomcat.test-on-borrow=true
spring.datasource.tomcat.validation-query=SELECT 1
```

### Hibernate Property Reference

| Property | Description | Value |
|----------|-------------|-------|
| database-platform | SQL dialect for query generation | org.hibernate.dialect.MySQLDialect |
| show_sql | Print SQL to console | true (UAT) / false (PROD) |
| format_sql | Pretty-print SQL | true (UAT) |
| use_sql_comments | Add comments to SQL | true (UAT) |

### Environment-Specific Overrides

| Environment | File | Key Settings |
|-------------|------|--------------|
| DEV | `config/application.properties` | MySQLDialect, plain password |
| SIT | `AppConfigs/SIT/application.properties` | MySQLDialect, ENC() password |
| UAT | `AppConfigs/UAT/application.properties` | show_sql=true, format_sql=true, SQL DEBUG logging |
| PROD | `AppConfigs/PROD/DC1/config/application.properties` | show_sql=false, SQL logging disabled |
| Test | `src/test/resources/application.properties` | H2Dialect (in-memory) |

### SQL Debugging Configuration (UAT)

```properties
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.type=trace
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

---

## Part 2: JPA Auto-Configuration

### Purpose

Enables entity scanning and Spring Data JPA repository scanning via annotations.

### Source Reference

**File:** `com.nets.upi.config.ProcessingConfig`

### Actual Usage

```java
@Configuration
@EnableAutoConfiguration
@EnableCaching
@ComponentScan(basePackages = { "com.nets.nps.qr", "com.nets.upi" })
@EntityScan(basePackages = { "com.nets.nps.qr.core.entity", "com.nets.upi.core.entity" })
@EnableJpaRepositories(basePackages = { "com.nets.nps.qr.core.repository", "com.nets.upi.repository" })
//@EnableTransactionManagement
public class ProcessingConfig {

    @Bean
    public CommonsRequestLoggingFilter logFilter() {
        CommonsRequestLoggingFilter filter = new CommonsRequestLoggingFilter();
        filter.setIncludeQueryString(true);
        filter.setIncludePayload(true);
        filter.setMaxPayloadLength(10000);
        filter.setIncludeHeaders(true);
        filter.setAfterMessagePrefix("Request: ");
        return filter;
    }
}
```

### How It Works

| Annotation | Purpose |
|------------|---------|
| `@EnableAutoConfiguration` | Triggers Spring Boot auto-config for DataSource, EntityManagerFactory, JpaTransactionManager |
| `@EntityScan(basePackages)` | Tells Hibernate which packages to scan for `@Entity` classes |
| `@EnableJpaRepositories(basePackages)` | Scans packages for `JpaRepository` interfaces, creates runtime proxy implementations |
| `@EnableCaching` | Enables caching abstraction (ehcache provider) |

### Package Scanning Configuration

| Scan Type | Packages |
|-----------|----------|
| Entity Scan | `com.nets.nps.qr.core.entity`, `com.nets.upi.core.entity` |
| Repository Scan | `com.nets.nps.qr.core.repository`, `com.nets.upi.repository` |

### Auto-Configured Beans

| Bean | Class | Purpose |
|------|-------|---------|
| dataSource | Tomcat JDBC DataSource | Database connection pool |
| entityManagerFactory | LocalContainerEntityManagerFactoryBean | Creates Hibernate EntityManager instances |
| transactionManager | JpaTransactionManager | Declarative transaction management |

---

# Part 3: Entity Layer

## Purpose

Defines JPA entities that map database tables to Java objects using `javax.persistence` annotations.

---

## 3.1 Entity Class

### Source Reference

**File:** `com.nets.upi.core.entity.KeyInfo`

### Actual Usage

```java
package com.nets.upi.core.entity;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Table(name = "tbl_key_info")
public class KeyInfo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ins_code")
    private String insCode;

    @Column(name = "sign_certid")
    private String signCertid;

    @Column(name = "sign_publickey")
    private String signPublickey;

    @Column(name = "sign_privatekey")
    private String signPrivatekey;

    @Column(name = "enc_certid")
    private String encCertid;

    @Column(name = "enc_publickey")
    private String encPublickey;

    @Column(name = "enc_privatekey")
    private String encPrivatekey;

    @Column(name = "uais_sign_certid")
    private String uaisSignCertid;

    @Column(name = "uais_sign_publickey")
    private String uaisSignPublickey;

    @Column(name = "uais_enc_certid")
    private String uaisEncCertid;

    @Column(name = "uais_enc_publickey")
    private String uaisEncPublickey;

    @Column(name = "signcert_update_time")
    private String signcertUpdateTime;

    @Column(name = "enccert_update_time")
    private String enccertUpdateTime;

    // Getters and Setters
    public String getInsCode() {
        return insCode;
    }

    public void setInsCode(String insCode) {
        this.insCode = insCode;
    }

    public String getSignCertid() {
        return signCertid;
    }

    public void setSignCertid(String signCertid) {
        this.signCertid = signCertid;
    }

    // ... additional getters/setters for all fields
}
```

### JPA Annotation Reference

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@Entity` | Marks class as JPA-managed entity | Class-level |
| `@Table(name = "...")` | Maps entity to database table | `@Table(name = "tbl_key_info")` |
| `@Id` | Marks primary key field | Field-level |
| `@GeneratedValue(strategy)` | PK generation strategy | `GenerationType.IDENTITY` (DB auto-increment) |
| `@Column(name = "...")` | Maps field to column | `@Column(name = "ins_code")` |

### Column Mapping Reference

| Column Name | Java Property | JDBC Type |
|-------------|---------------|-----------|
| ins_code | insCode | VARCHAR (PK) |
| sign_certid | signCertid | VARCHAR |
| sign_publickey | signPublickey | VARCHAR |
| sign_privatekey | signPrivatekey | VARCHAR |
| enc_certid | encCertid | VARCHAR |
| enc_publickey | encPublickey | VARCHAR |
| enc_privatekey | encPrivatekey | VARCHAR |
| uais_sign_certid | uaisSignCertid | VARCHAR |
| uais_sign_publickey | uaisSignPublickey | VARCHAR |
| uais_enc_certid | uaisEncCertid | VARCHAR |
| uais_enc_publickey | uaisEncPublickey | VARCHAR |
| signcert_update_time | signcertUpdateTime | VARCHAR |
| enccert_update_time | enccertUpdateTime | VARCHAR |

---

## 3.2 External Entity (from nps-qr-core library)

### Source Reference

**Package:** `com.nets.nps.qr.core.entity.UpiH5Authorization` (external dependency)

### Actual Usage in Controller

```java
import com.nets.nps.qr.core.entity.UpiH5Authorization;
import com.nets.nps.qr.core.repository.UpiH5AuthorizationRepository;

@RestController
public class GetUserIdController extends UpiCommonController {

    @Autowired
    private UpiH5AuthorizationRepository authRepository;

    @PostMapping("/post/crossborder/upi/qr/user")
    public ResponseEntity<String> getUserID(@RequestBody GetUserIdRequest request) {
        
        // Query using custom finder method
        UpiH5Authorization auth = authRepository.findByAuthCodeAndStatus(
            request.getTrxInfo().getUserAuthCode(), RequestState.WAITING.getState());
        
        if (Objects.nonNull(auth)) {
            // Update entity state
            auth.setStatus(RequestState.COMPLETED.getState());
            // Persist changes
            authRepository.save(auth);
        }
    }
}
```

---

# Part 4: Repository Layer

## Purpose

Provides data access through Spring Data JPA repository interfaces. Hibernate generates SQL at runtime; no implementation class is hand-written.

---

## 4.1 JpaRepository Interface

### Source Reference

**File:** `com.nets.upi.repository.KeyInfoRepository`

### Actual Usage

```java
package com.nets.upi.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import com.nets.upi.core.entity.KeyInfo;

@Repository
public interface KeyInfoRepository extends JpaRepository<KeyInfo, String> {
}
```

### How It Works

- Extends `JpaRepository<KeyInfo, String>` — entity type `KeyInfo`, primary key type `String`
- Spring Data JPA creates a runtime proxy implementation
- Inherits CRUD operations without any hand-written SQL
- `@Repository` enables Spring exception translation

### Inherited Operations (from JpaRepository)

| Method | Description | SQL Generated |
|--------|-------------|---------------|
| `findById(ID)` | Returns `Optional<T>` by primary key | `SELECT ... WHERE ins_code = ?` |
| `findAll()` | Returns all records | `SELECT ... FROM tbl_key_info` |
| `save(T)` | Insert or update (upsert by PK) | `INSERT ...` or `UPDATE ...` |
| `saveAndFlush(T)` | Save and immediately flush to DB | Same as save, forces flush |
| `deleteById(ID)` | Delete by primary key | `DELETE ... WHERE ins_code = ?` |
| `count()` | Row count | `SELECT COUNT(*) ...` |
| `existsById(ID)` | Existence check | `SELECT COUNT(*) ... WHERE ins_code = ?` |

---

## 4.2 External Repository with Custom Query (from nps-qr-core library)

### Source Reference

**Package:** `com.nets.nps.qr.core.repository.UpiH5AuthorizationRepository` (external dependency)

### Actual Usage

```java
// Custom finder method (query derived from method name)
UpiH5Authorization auth = authRepository.findByAuthCodeAndStatus(authCode, status);

// Standard save operation
authRepository.save(auth);
```

### Query Derivation

| Method Name | Generated Query |
|-------------|-----------------|
| `findByAuthCodeAndStatus(code, status)` | `SELECT ... WHERE auth_code = ? AND status = ?` |

---

# Part 5: Service Layer

## Purpose

Provides business-level data access with repository delegation. Abstracts JPA repository from controllers.

---

## 5.1 Service Interface

### Source Reference

**File:** `com.nets.upi.core.service.KeyInfoService`

### Actual Usage

```java
package com.nets.upi.core.service;

import com.nets.upi.core.entity.KeyInfo;

public interface KeyInfoService {

    public KeyInfo save(KeyInfo queryDetails);
    
    public KeyInfo findById(String id);
}
```

---

## 5.2 Service Implementation

### Source Reference

**File:** `com.nets.upi.core.service.impl.KeyInfoServiceImpl`

### Actual Usage

```java
package com.nets.upi.core.service.impl;

import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.nets.upi.core.entity.KeyInfo;
import com.nets.upi.core.service.KeyInfoService;
import com.nets.upi.repository.KeyInfoRepository;

@Service
public class KeyInfoServiceImpl implements KeyInfoService {

    @Autowired
    private KeyInfoRepository keyInfoRepository;

    public KeyInfo save(KeyInfo queryDetails) {
        return keyInfoRepository.saveAndFlush(queryDetails);
    }

    public KeyInfo findById(String id) {
        Optional<KeyInfo> optional = keyInfoRepository.findById(id);
        if (optional.isPresent())
            return optional.get();
        else
            return null;
    }
}
```

### How It Works

| Aspect | Description |
|--------|-------------|
| `@Service` | Registers as Spring bean |
| `@Autowired` | Injects JPA repository |
| `saveAndFlush()` | Persist and immediately synchronise with DB |
| `findById()` | Unwraps `Optional<KeyInfo>`, returns null when absent |

---

## 5.3 Service Consumers

### Usage Pattern in Controllers

```java
@RestController
public class KeyExchangeController {

    @Autowired
    private KeyInfoService keyInfoService;

    // Load keys by institution ID
    private KeyInfo getKeyInfo(String insID) {
        KeyInfo keyInfo = new KeyInfo();
        keyInfo = keyInfoService.findById(insID);
        return keyInfo;
    }

    @PostMapping("/post/crossborder/upi/keyexchange")
    ResponseEntity<KeyExchangeMessage> processDebit(@RequestBody KeyExchangeMessage request) {
        
        String instId = getInstId(request.getMsgID());
        KeyInfo keyInfo = getKeyInfo(instId);

        // Update keys after exchange
        keyInfo.setUaisEncPublickey(encPublicKey);
        keyInfo.setEnccertUpdateTime(UtillComponents.getNowUPIFormatDate());
        
        // Persist updated keys
        keyInfoService.save(keyInfo);
    }
}
```

### Consumer Summary

| Component | Location | Operations Used |
|-----------|----------|-----------------|
| KeyExchangeController | controller | `findById`, `save` |
| DebitTransactionController | controller | `findById` |
| RefundTransactionController | controller | `findById` |
| ReversalTransactionController | controller | `findById` |
| TransactionResultController | controller | `findById` |
| AddProcessingController | controller | `findById` |
| UpiCommonController | controller | `findById` |
| GetUserIdController | controller | External repo: `findByAuthCodeAndStatus`, `save` |
| ExchangeService | processing | `findById` |
| KeyExchangeService | processing | `findById`, `save` |
| TspUpiProxyProcessingService | processing | `findById` |

---

# Execution Flow

## SELECT Flow (findById)

```
Controller
    │  keyInfoService.findById(insID)
    ▼
┌──────────────────────────────────────┐
│  KeyInfoServiceImpl                   │
│  - keyInfoRepository.findById(id)     │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  KeyInfoRepository (Spring Data Proxy)│
│  - Creates query from method name     │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  Hibernate EntityManager              │
│  - Generates: SELECT * FROM           │
│    tbl_key_info WHERE ins_code = ?    │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  JDBC / DataSource                    │
│  - Executes SQL on MySQL              │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  ResultSet → Entity Mapping           │
│  - Hibernate hydrates KeyInfo         │
└──────────────────────────────────────┘
    │
    ▼
Returns Optional<KeyInfo> → unwrap → KeyInfo
```

---

## INSERT/UPDATE Flow (save)

```
Controller
    │  keyInfoService.save(keyInfo)
    ▼
┌──────────────────────────────────────┐
│  KeyInfoServiceImpl                   │
│  - keyInfoRepository.saveAndFlush()   │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  KeyInfoRepository (Spring Data Proxy)│
│  - Delegates to EntityManager         │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  JpaTransactionManager                │
│  - Begin transaction (if none)        │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  Hibernate EntityManager              │
│  - Check persistence context          │
│  - New entity → INSERT                │
│  - Existing entity → UPDATE           │
│  - saveAndFlush forces immediate SQL  │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  JDBC / DataSource                    │
│  - Executes INSERT/UPDATE on MySQL    │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  Transaction Commit                   │
│  - saveAndFlush commits immediately   │
└──────────────────────────────────────┘
    │
    ▼
Returns persisted KeyInfo
```

---

# Configuration Summary

## Configuration Files

| File | Type | Layer | Purpose |
|------|------|-------|---------|
| config/application.properties | Properties | Core | DataSource + Hibernate settings |
| AppConfigs/<ENV>/application.properties | Properties | Core | Environment-specific JPA/SQL logging |
| ProcessingConfig.java | Java | Core | @EntityScan, @EnableJpaRepositories |
| KeyInfo.java | Java | Entity | tbl_key_info mapping |
| KeyInfoRepository.java | Java | Repository | Spring Data JPA interface |
| KeyInfoService.java | Java | Service | Business-level interface |
| KeyInfoServiceImpl.java | Java | Service | Repository delegation |

---

## Bean Summary

| Bean Name | Class | Layer | Purpose |
|-----------|-------|-------|---------|
| dataSource | Tomcat JDBC DataSource | Core | Database connection pool |
| entityManagerFactory | LocalContainerEntityManagerFactoryBean | Core | Creates Hibernate EntityManager |
| transactionManager | JpaTransactionManager | Core | Transaction management |
| keyInfoRepository | KeyInfoRepository (proxy) | Repository | CRUD on tbl_key_info |
| keyInfoServiceImpl | KeyInfoServiceImpl | Service | Key info business logic |

---

## Entity / Repository / Table Summary

| Entity | Table | Repository | Primary Key | Package |
|--------|-------|------------|-------------|---------|
| KeyInfo | tbl_key_info | KeyInfoRepository | ins_code (String) | com.nets.upi |
| UpiH5Authorization | (external) | UpiH5AuthorizationRepository | (external) | com.nets.nps.qr.core |

---

## Properties Reference

### DataSource Properties

```properties
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://<host>:<port>/<database>?autoReconnect=true&serverTimezone=UTC&useSSL=false
spring.datasource.username=<username>
spring.datasource.password=<password>   # ENC(...) in SIT/UAT/PROD via Jasypt
```

### Hibernate / JPA Properties

```properties
# SQL dialect
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect   # H2Dialect for tests

# SQL logging (UAT only)
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Connection Pool Validation Properties

```properties
spring.datasource.tomcat.test-while-idle=true
spring.datasource.tomcat.test-on-borrow=true
spring.datasource.tomcat.validation-query=SELECT 1
```

---

# Quick Reference

## Hibernate / Spring Data JPA Pattern

```java
// 1. Entity - maps table to Java object
@Entity
@Table(name = "tbl_key_info")
public class KeyInfo {
    @Id
    @Column(name = "ins_code")
    private String insCode;
    
    @Column(name = "sign_publickey")
    private String signPublickey;
    // ...
}

// 2. Repository - CRUD operations (no implementation needed)
@Repository
public interface KeyInfoRepository extends JpaRepository<KeyInfo, String> {
}

// 3. Service - business logic delegation
@Service
public class KeyInfoServiceImpl implements KeyInfoService {
    @Autowired
    private KeyInfoRepository keyInfoRepository;

    public KeyInfo save(KeyInfo entity) {
        return keyInfoRepository.saveAndFlush(entity);
    }

    public KeyInfo findById(String id) {
        return keyInfoRepository.findById(id).orElse(null);
    }
}

// 4. Controller - consumes service
@RestController
public class MyController {
    @Autowired
    private KeyInfoService keyInfoService;

    @PostMapping("/api")
    public void process() {
        KeyInfo keyInfo = keyInfoService.findById("12345");
        keyInfo.setSignPublickey("new-key");
        keyInfoService.save(keyInfo);
    }
}
```

---

# Comparison: Hibernate (JPA) vs MyBatis vs JdbcTemplate

| Aspect | Hibernate / Spring Data JPA | MyBatis | JdbcTemplate |
|--------|-----------------------------|---------|--------------|
| Mapping | Annotations on `@Entity` | XML `<resultMap>` | Manual RowMapper |
| SQL | Generated by Hibernate | Hand-written in XML | Hand-written in code |
| Repository | Interface extends `JpaRepository` | Mapper interface + XML | Repository with template calls |
| Boilerplate | Lowest (CRUD inherited) | Medium | Highest |
| Control over SQL | Lowest (abstracted) | High | Full |
| Best for | Simple CRUD, rapid development | Complex queries, full SQL control | Batch operations, legacy |
