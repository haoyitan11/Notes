# JdbcTemplate Dependencies Handle

## Overview

This document provides a complete guide for database connectivity using JdbcTemplate in Spring applications. It covers the dependency chain from properties configuration to database operations.

---

## Component Dependency Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         1. PROPERTIES FILE                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                    batch.properties                                      │    │
│  │                                                                          │    │
│  │  spring.datasource.jdbcUrl=jdbc:mysql://host:3306/db                    │    │
│  │  spring.datasource.username=user                                         │    │
│  │  spring.datasource.password=${PASSWORD}                                  │    │
│  │  spring.datasource.type=com.zaxxer.hikari.HikariDataSource              │    │
│  │                                                                          │    │
│  │  batch.datasource.jdbcUrl=jdbc:mysql://host:3306/db                     │    │
│  │  batch.datasource.username=user                                          │    │
│  │  batch.datasource.password=${PASSWORD}                                   │    │
│  │  batch.datasource.type=com.zaxxer.hikari.HikariDataSource               │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────┬─────────────────────┘
                                                            │
                              @ConfigurationProperties binds properties
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         2. DATASOURCE CONFIGURATION                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │              DatasourceConfiguration.java                                │    │
│  │                                                                          │    │
│  │  @ConfigurationProperties(prefix="spring.datasource")                   │    │
│  │  public DataSource springDataSource() ──────────┐                       │    │
│  │                                                  │                       │    │
│  │  @ConfigurationProperties(prefix="batch.datasource")                    │    │
│  │  public DataSource batchDatasource() ───────────┼───┐                   │    │
│  │                                                  │   │                   │    │
│  └──────────────────────────────────────────────────┼───┼───────────────────┘    │
└─────────────────────────────────────────────────────┼───┼───────────────────────┘
                                                      │   │
                         DataSourceBuilder.create().build()
                                  creates HikariDataSource
                                                      │   │
                                                      ▼   ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         3. DATASOURCE BEANS                                      │
│  ┌────────────────────────────┐        ┌────────────────────────────┐           │
│  │    springDatasource        │        │     batchDatasource        │           │
│  │       (@Primary)           │        │                            │           │
│  │                            │        │                            │           │
│  │  ┌──────────────────────┐  │        │  ┌──────────────────────┐  │           │
│  │  │    HikariDataSource  │  │        │  │    HikariDataSource  │  │           │
│  │  │   (Connection Pool)  │  │        │  │   (Connection Pool)  │  │           │
│  │  └──────────────────────┘  │        │  └──────────────────────┘  │           │
│  └────────────────────────────┘        └────────────────────────────┘           │
└───────────────────────────────────────────────────────┬─────────────────────────┘
                                                        │
                              @Autowired @Qualifier injects DataSource
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         4. REPOSITORY CLASS                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  @Repository                                                             │    │
│  │  public class BatchRepository {                                          │    │
│  │                                                                          │    │
│  │      @Autowired @Qualifier("batchDatasource")                           │    │
│  │      DataSource readOnlySource;                                          │    │
│  │                                                                          │    │
│  │      @Autowired @Qualifier("springDatasource")                          │    │
│  │      DataSource writeDataSource;                                         │    │
│  │  }                                                                       │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────┬─────────────────────────┘
                                                        │
                         new NamedParameterJdbcTemplate(dataSource)
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         5. JDBCTEMPLATE USAGE                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  NamedParameterJdbcTemplate jdbcTemplate =                               │    │
│  │      new NamedParameterJdbcTemplate(readOnlySource);                     │    │
│  │                                                                          │    │
│  │  jdbcTemplate.query(sql, params, rowMapper);   // SELECT                 │    │
│  │  jdbcTemplate.update(sql, params);             // INSERT/UPDATE          │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────┬─────────────────────────┘
                                                        │
                              JDBC Connection from HikariCP Pool
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         6. DATABASE                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                         MySQL Database                                   │    │
│  │                    jdbc:mysql://host:3306/database                       │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 1: Properties Configuration

## Purpose

Defines database connection parameters that will be bound to DataSource beans.

## Source Reference

**File:** `conf/batch.properties`

## Configuration

```properties
# ═══════════════════════════════════════════════════════════════════════════════
# SPRING DATASOURCE (for write operations)
# ═══════════════════════════════════════════════════════════════════════════════

# JDBC Driver
spring.datasource.driver=com.mysql.jdbc.Driver

# Connection URL
spring.datasource.jdbcUrl=jdbc:mysql://db-host:3306/database_name?sslMode=REQUIRED

# Credentials
spring.datasource.username=svc_user
spring.datasource.password=${DATABASE_PASSWORD}

# Connection Pool Type
spring.datasource.type=com.zaxxer.hikari.HikariDataSource

# HikariCP Settings
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=600000
spring.datasource.hikari.connection-test-query=SELECT 1

# ═══════════════════════════════════════════════════════════════════════════════
# BATCH DATASOURCE (for read operations)
# ═══════════════════════════════════════════════════════════════════════════════

# JDBC Driver
batch.datasource.driver=com.mysql.jdbc.Driver

# Connection URL
batch.datasource.jdbcUrl=jdbc:mysql://db-host:3306/database_name?sslMode=REQUIRED

# Credentials
batch.datasource.username=svc_user
batch.datasource.password=${DATABASE_PASSWORD}

# Connection Pool Type
batch.datasource.type=com.zaxxer.hikari.HikariDataSource

# HikariCP Settings
batch.datasource.hikari.minimum-idle=5
batch.datasource.hikari.maximum-pool-size=15
batch.datasource.hikari.auto-commit=true
batch.datasource.hikari.idle-timeout=600000
batch.datasource.hikari.max-lifetime=1800000
batch.datasource.hikari.connection-timeout=600000
batch.datasource.hikari.connection-test-query=SELECT 1
```

## HikariCP Property Reference

| Property | Description | Value |
|----------|-------------|-------|
| minimum-idle | Minimum idle connections in pool | 5 |
| maximum-pool-size | Maximum connections in pool | 15 |
| auto-commit | Auto-commit mode | true |
| idle-timeout | Max time connection can be idle (ms) | 600000 (10 min) |
| max-lifetime | Max lifetime of connection (ms) | 1800000 (30 min) |
| connection-timeout | Max time to wait for connection (ms) | 600000 (10 min) |
| connection-test-query | Query to validate connections | SELECT 1 |

---

# Part 2: DataSource Configuration

## Purpose

Creates DataSource beans from properties using **@ConfigurationProperties** binding.

## Dependencies

```java
javax.sql.DataSource
org.springframework.boot.jdbc.DataSourceBuilder
org.springframework.boot.context.properties.ConfigurationProperties
org.springframework.context.annotation.Bean
org.springframework.context.annotation.Primary
org.springframework.context.annotation.Configuration
```

## Source Reference

**File:** `com.example.batch.config.DatasourceConfiguration`

## Actual Usage

```java
@Configuration
public class DatasourceConfiguration {
    
    @Primary
    @Bean(name={"springDatasource", "dataSource"})
    @ConfigurationProperties(prefix="spring.datasource")
    public DataSource springDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean(name="batchDatasource")
    @ConfigurationProperties(prefix="batch.datasource")
    public DataSource batchDatasource() {
        return DataSourceBuilder.create().build();
    }
}
```

## How the Linkage Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  @ConfigurationProperties(prefix="spring.datasource")                        │
│                                                                              │
│  This annotation tells Spring to:                                            │
│                                                                              │
│  1. Find all properties starting with "spring.datasource."                   │
│  2. Map them to the DataSource being built:                                  │
│                                                                              │
│     spring.datasource.jdbcUrl    ───────▶  DataSource.setJdbcUrl()          │
│     spring.datasource.username   ───────▶  DataSource.setUsername()         │
│     spring.datasource.password   ───────▶  DataSource.setPassword()         │
│     spring.datasource.type       ───────▶  Creates HikariDataSource         │
│     spring.datasource.hikari.*   ───────▶  HikariConfig properties          │
│                                                                              │
│  3. DataSourceBuilder.create().build() creates the configured DataSource    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Bean Naming

| Bean Name | Qualifier | Purpose |
|-----------|-----------|---------|
| springDatasource | @Primary (default) | Write operations (INSERT, UPDATE, DELETE) |
| batchDatasource | "batchDatasource" | Read operations (SELECT) |

---

# Part 3: Repository Layer

## Purpose

Injects DataSource beans and creates JdbcTemplate for database operations.

## Dependencies

```java
javax.sql.DataSource
org.springframework.jdbc.core.JdbcTemplate
org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
org.springframework.beans.factory.annotation.Autowired
org.springframework.beans.factory.annotation.Qualifier
org.springframework.stereotype.Repository
```

## Source Reference

**File:** `com.example.batch.repository.BatchRepository`

## DataSource Injection

```java
@Repository
public class BatchRepository {

    @Autowired
    @Qualifier("batchDatasource")
    DataSource readOnlySource;      // Injected from batchDatasource bean
    
    @Autowired
    @Qualifier("springDatasource")
    DataSource writeDataSource;     // Injected from springDatasource bean (@Primary)
}
```

## How the Linkage Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  @Autowired @Qualifier("batchDatasource")                                    │
│  DataSource readOnlySource;                                                  │
│                                                                              │
│  This tells Spring to:                                                       │
│                                                                              │
│  1. Look for a bean named "batchDatasource" in the ApplicationContext       │
│  2. That bean was created by DatasourceConfiguration.batchDatasource()      │
│  3. Inject it into the readOnlySource field                                 │
│                                                                              │
│  ┌──────────────────────┐         ┌──────────────────────┐                  │
│  │  DatasourceConfig    │         │    Repository        │                  │
│  │                      │         │                      │                  │
│  │  @Bean("batchData-   │ ──────▶ │  @Qualifier("batch-  │                  │
│  │   source")           │  inject │   Datasource")       │                  │
│  │  DataSource bean     │         │  DataSource field    │                  │
│  └──────────────────────┘         └──────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 4: JdbcTemplate Usage

## Purpose

Creates JdbcTemplate from DataSource to execute SQL queries and updates.

## Dependencies

```java
org.springframework.jdbc.core.JdbcTemplate
org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
org.springframework.jdbc.core.namedparam.MapSqlParameterSource
org.springframework.jdbc.core.RowMapper
```

## How the Linkage Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  NamedParameterJdbcTemplate jdbcTemplate =                                   │
│      new NamedParameterJdbcTemplate(readOnlySource);                         │
│                                                                              │
│  This creates a JdbcTemplate that:                                           │
│                                                                              │
│  1. Wraps the DataSource (readOnlySource)                                   │
│  2. Obtains JDBC connections from the HikariCP pool                         │
│  3. Executes SQL using those connections                                    │
│  4. Returns connections to the pool after use                               │
│                                                                              │
│  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     │
│  │ NamedParameter   │     │    DataSource    │     │    HikariCP      │     │
│  │   JdbcTemplate   │────▶│  (readOnlySource)│────▶│  Connection Pool │     │
│  │                  │     │                  │     │                  │     │
│  │  .query()        │     │  .getConnection()│     │  Pool of JDBC    │     │
│  │  .update()       │     │                  │     │  Connections     │     │
│  └──────────────────┘     └──────────────────┘     └──────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4.1 SELECT Query with Lambda RowMapper

### Actual Usage

```java
public List<DetailRecord> getRecords() {

    // 1. Create JdbcTemplate from DataSource
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);

    // 2. Prepare named parameters
    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("status", "ACTIVE");

    // 3. Define SQL query with named parameters
    String query = "SELECT id, name, amount FROM records WHERE status = :status";
    
    // 4. Execute query with RowMapper
    List<DetailRecord> records = jdbcTemplate.query(query, namedParameters, (rs, rowNum) -> {
        DetailRecord record = new DetailRecord();
        record.setId(rs.getLong("id"));
        record.setName(rs.getString("name"));
        record.setAmount(rs.getBigDecimal("amount"));
        return record;
    });

    return records;
}
```

### Execution Flow

```
jdbcTemplate.query(sql, params, rowMapper)
        │
        ▼
┌───────────────────┐
│ Get Connection    │◄─── From HikariCP pool via DataSource
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Create Prepared   │◄─── Named params (:status) converted to (?)
│ Statement         │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Execute Query     │◄─── SQL sent to MySQL
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Process ResultSet │◄─── RowMapper called for each row
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Return Connection │◄─── Connection returned to pool
│ to Pool           │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Return List<T>    │◄─── Mapped objects returned to caller
└───────────────────┘
```

---

## 4.2 SELECT Query with RowMapper Class

### Actual Usage

```java
public List<DetailRecord> getRecords(String date, Set<Integer> ids) {

    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("date", date);
    namedParameters.put("ids", ids);  // IN clause support

    String query = "SELECT id, name, amount FROM records "
            + "WHERE created_date = :date AND category_id IN (:ids)";
    
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    return jdbcTemplate.query(query, namedParameters, new RecordRowMapper());
}

private class RecordRowMapper implements RowMapper<DetailRecord> {
    @Override
    public DetailRecord mapRow(ResultSet rs, int rowNum) throws SQLException {
        DetailRecord record = new DetailRecord();
        record.setId(rs.getLong("id"));
        record.setName(rs.getString("name"));
        record.setAmount(rs.getBigDecimal("amount"));
        return record;
    }
}
```

---

## 4.3 SELECT Query with BeanPropertyRowMapper

### Actual Usage

```java
public List<RefundRecord> getRefunds(String startDate, String endDate) {
    
    NamedParameterJdbcTemplate template = new NamedParameterJdbcTemplate(readOnlySource);
    
    // Column aliases must match bean property names
    String query = "SELECT "
            + "authorize_date as txnDate, "      // Maps to RefundRecord.txnDate
            + "stan as txnRRN, "                  // Maps to RefundRecord.txnRRN
            + "target_amount as refundAmt "       // Maps to RefundRecord.refundAmt
            + "FROM transactions WHERE date BETWEEN :start AND :end";
    
    Map<String, Object> params = new HashMap<>();
    params.put("start", startDate);
    params.put("end", endDate);
    
    // BeanPropertyRowMapper auto-maps columns to bean properties by name
    return template.query(query, params, new BeanPropertyRowMapper<>(RefundRecord.class));
}
```

---

## 4.4 INSERT with JdbcTemplate

### Actual Usage

```java
public void createRecord(String name, Integer status, Timestamp createdDate) {
    
    // Create JdbcTemplate from write DataSource
    JdbcTemplate template = new JdbcTemplate(writeDataSource);
    
    // SQL with positional parameters (?)
    String sql = "INSERT INTO records (NAME, STATUS, CREATED_DATE) VALUES (?, ?, ?)";
    
    // Execute with varargs parameters (in order)
    template.update(sql, name, status, createdDate);
}
```

---

## 4.5 UPDATE with NamedParameterJdbcTemplate

### Actual Usage

```java
public void updateStatus(String newStatus, int recordId) {

    // Create JdbcTemplate from write DataSource
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(writeDataSource);
    
    // Build parameters using MapSqlParameterSource
    SqlParameterSource namedParameters = new MapSqlParameterSource()
            .addValue("status", newStatus)
            .addValue("id", recordId);
    
    // SQL with named parameters
    String updateSql = "UPDATE records SET STATUS = :status WHERE ID = :id";
    
    // Execute update
    int affectedRows = jdbcTemplate.update(updateSql, namedParameters);
}
```

---

# Part 5: Transaction Management

## Purpose

Controls transaction boundaries for database operations.

## Dependencies

```java
org.springframework.transaction.annotation.Transactional
```

## @Transactional(readOnly = true)

```java
@Transactional(readOnly = true)
public List<DetailRecord> getRecords(String date) {
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    // ... SELECT query
}
```

### What readOnly = true Does

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  @Transactional(readOnly = true)                                             │
│                                                                              │
│  1. Hints to the database that no writes will occur                         │
│  2. Database may optimize (no write locks, use read replicas)               │
│  3. Spring will throw exception if UPDATE/INSERT attempted                  │
│  4. Connection returned to pool in read-only state                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Summary

## Complete Linkage Chain

```
batch.properties
    │
    │ spring.datasource.* properties
    │ batch.datasource.* properties
    │
    ▼
@ConfigurationProperties ─────────────────────────────────────────────────────┐
    │                                                                          │
    │ Binds properties to DataSource configuration                            │
    │                                                                          │
    ▼                                                                          │
DataSourceBuilder.create().build()                                            │
    │                                                                          │
    │ Creates HikariDataSource with connection pool                           │
    │                                                                          │
    ▼                                                                          │
@Bean DataSource                                                               │
    │                                                                          │
    │ Registers DataSource bean in Spring ApplicationContext                  │
    │                                                                          │
    ▼                                                                          │
@Autowired @Qualifier ────────────────────────────────────────────────────────┤
    │                                                                          │
    │ Injects DataSource bean into Repository field                           │
    │                                                                          │
    ▼                                                                          │
new NamedParameterJdbcTemplate(dataSource)                                    │
    │                                                                          │
    │ Creates JdbcTemplate wrapper around DataSource                          │
    │                                                                          │
    ▼                                                                          │
jdbcTemplate.query() / jdbcTemplate.update()                                  │
    │                                                                          │
    │ Gets connection from pool, executes SQL, returns connection             │
    │                                                                          │
    ▼                                                                          │
MySQL Database                                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Quick Reference Tables

### DataSource Configuration

| Bean Name | Property Prefix | Purpose |
|-----------|-----------------|---------|
| springDatasource | spring.datasource.* | Write operations |
| batchDatasource | batch.datasource.* | Read operations |

### JdbcTemplate Methods

| Method | Purpose | DataSource |
|--------|---------|------------|
| `query(sql, params, rowMapper)` | SELECT queries | readOnlySource |
| `update(sql, params)` | INSERT/UPDATE/DELETE | writeDataSource |

### RowMapper Patterns

| Pattern | Use Case |
|---------|----------|
| Lambda `(rs, rowNum) -> {}` | Simple inline mapping |
| RowMapper Class | Reusable complex mapping |
| BeanPropertyRowMapper | Auto-map by column name |

## Import Reference

```java
// DataSource
import javax.sql.DataSource;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.boot.context.properties.ConfigurationProperties;

// JdbcTemplate
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.namedparam.SqlParameterSource;

// RowMapper
import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.core.BeanPropertyRowMapper;

// Spring Annotations
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Repository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.transaction.annotation.Transactional;
```
