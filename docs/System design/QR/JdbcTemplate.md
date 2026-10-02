# JdbcTemplate Dependencies Handle

## Overview

This document provides a complete guide for database connectivity using JdbcTemplate in Spring applications. It covers:

- **Common Foundation:** Properties → DataSource → Repository (shared infrastructure)

- **Scenario 1:** JdbcTemplate with positional parameters `?` (for INSERT operations)

- **Scenario 2:** NamedParameterJdbcTemplate with named parameters `:param` (for SELECT/UPDATE operations)

- **Scenario 3:** JdbcTemplate queryForRowSet + NamedParameterJdbcTemplate (for DELETE operations with SqlRowSet)

---

# Common Foundation: Database Infrastructure

This section covers the shared infrastructure used by all JdbcTemplate scenarios.

---

## Part 1: Properties Configuration

### Purpose

Defines database connection parameters that will be bound to DataSource beans.

### Source Reference

**File:** `conf/batch.properties`

### Configuration

```properties
# ═══════════════════════════════════════════════════════════════════════════════
# SPRING DATASOURCE (for write operations)
# ═══════════════════════════════════════════════════════════════════════════════
spring.datasource.driver=com.mysql.jdbc.Driver
spring.datasource.jdbcUrl=jdbc:mysql://db-host:3306/database_name?sslMode=REQUIRED
spring.datasource.username=svc_user
spring.datasource.password=${SPRINGBATCH_DATABASE_PASSWORD}
spring.datasource.type=com.zaxxer.hikari.HikariDataSource

# HikariCP Settings
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.pool-name=BatchHikariCP
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=600000
spring.datasource.hikari.connection-test-query=SELECT 1

# ═══════════════════════════════════════════════════════════════════════════════
# BATCH DATASOURCE (for read operations)
# ═══════════════════════════════════════════════════════════════════════════════
batch.datasource.driver=com.mysql.jdbc.Driver
batch.datasource.jdbcUrl=jdbc:mysql://db-host:3306/database_name?sslMode=REQUIRED
batch.datasource.username=svc_user
batch.datasource.password=${BATCH_DATABASE_PASSWORD}
batch.datasource.type=com.zaxxer.hikari.HikariDataSource

# HikariCP Settings
batch.datasource.hikari.minimum-idle=5
batch.datasource.hikari.maximum-pool-size=15
batch.datasource.hikari.auto-commit=true
batch.datasource.hikari.idle-timeout=600000
batch.datasource.hikari.pool-name=BatchHikariCP
batch.datasource.hikari.max-lifetime=1800000
batch.datasource.hikari.connection-timeout=600000
batch.datasource.hikari.connection-test-query=SELECT 1
```

### HikariCP Property Reference

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

## Part 2: DataSource Configuration

### Purpose

Creates DataSource beans from properties using **@ConfigurationProperties** binding.

### Source Reference

**File:** `com.example.batch.config.DatasourceConfiguration`

### Actual Usage

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

### How It Works

- `@ConfigurationProperties(prefix="spring.datasource")` binds all properties starting with `spring.datasource.*` to the DataSource
- `DataSourceBuilder.create().build()` creates a HikariDataSource based on the `spring.datasource.type` property
- `@Primary` makes `springDatasource` the default when no qualifier is specified

### Bean Summary

| Bean Name | Property Prefix | Purpose |
|-----------|-----------------|---------|
| springDatasource | spring.datasource.* | Write operations (INSERT, UPDATE, DELETE) |
| batchDatasource | batch.datasource.* | Read operations (SELECT) |

---

## Part 3: Repository Layer

### Purpose

Injects DataSource beans for creating JdbcTemplate instances.

### Source Reference

**File:** `com.example.batch.repository.BatchRepository`

### DataSource Injection

```java
@Repository
public class BatchRepository {

    @Autowired
    @Qualifier("batchDatasource")
    DataSource readOnlySource;      // For SELECT queries
    
    @Autowired
    @Qualifier("springDatasource")
    DataSource writeDataSource;     // For INSERT/UPDATE/DELETE
}
```

### How It Works

- `@Qualifier("batchDatasource")` tells Spring to inject the specific bean named "batchDatasource"
- Without `@Qualifier`, Spring would inject the `@Primary` bean (springDatasource)

---

## Part 4: Transaction Management

### @Transactional(readOnly = true)

```java
@Transactional(readOnly = true)
public List<DetailRecord> getTransactions(String date) {
    // ... SELECT query
}
```

**What readOnly = true Does:**

- Hints to the database that no writes will occur
- Database may optimize (no write locks, can use read replicas)
- Spring will throw exception if UPDATE/INSERT attempted
- Connection returned to pool in read-only state

### @Transactional(propagation = Propagation.REQUIRES_NEW)

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void deleteRecords(String tableName, String primaryKey, List<Long> idList) {
    // ... DELETE query - runs in its own transaction
}
```

**What REQUIRES_NEW Does:**

- Always creates a new transaction
- Suspends any existing transaction
- Each call commits independently
- Useful for batch deletes where partial success is acceptable

---

# Scenario 1: JdbcTemplate (Positional Parameters)

This scenario covers `JdbcTemplate` usage with positional `?` parameters. Best suited for simple INSERT operations.

---

## Overview

**Class:** `org.springframework.jdbc.core.JdbcTemplate`

**Parameter Style:** Positional `?` placeholders

**Use Case:** Simple INSERT/UPDATE with few parameters

---

## INSERT with Positional Parameters

### Source Reference

**File:** `com.example.batch.repository.BatchRepository`

### Actual Usage

```java
public void createFileUploadEntry(String createdDate, String directory, String fileName, 
        String fileId, Integer status, Integer lastUpdatedBy, Timestamp lastUpdatedDate) {
    
    // Create JdbcTemplate from write DataSource
    JdbcTemplate template = new JdbcTemplate(writeDataSource);
    
    // SQL with positional parameters (?)
    StringBuilder query = new StringBuilder();
    query.append("INSERT INTO file_upload ");
    query.append("(CREATED_DATE, DIRECTORY, FILENAME, STATUS, LAST_UPDATED_BY, LAST_UPDATED_DATE, FILE_ID) ");
    query.append("VALUES (?, ?, ?, ?, ?, ?, ?)");
    
    // Execute with parameters in order matching ? positions
    template.update(query.toString(), 
            createdDate,        // ? position 1
            directory,          // ? position 2
            fileName,           // ? position 3
            status,             // ? position 4
            lastUpdatedBy,      // ? position 5
            lastUpdatedDate,    // ? position 6
            fileId);            // ? position 7
}
```

### Key Points

| Aspect | Description |
|--------|-------------|
| Parameter Binding | Parameters bound by position (order matters) |
| DataSource | Uses `writeDataSource` for INSERT |
| Return Value | `update()` returns number of affected rows |
| Best For | Simple INSERT with sequential parameters |

---

## Scenario 1 Summary

### When to Use JdbcTemplate

- Simple INSERT operations
- Few parameters in fixed order
- No need for parameter reuse in query

### Limitations

- Parameter order must match `?` positions exactly
- Hard to read with many parameters
- Cannot reuse same parameter value in multiple places

---

# Scenario 2: NamedParameterJdbcTemplate (Named Parameters)

This scenario covers `NamedParameterJdbcTemplate` usage with named `:param` parameters. Best suited for complex SELECT/UPDATE operations.

---

## Overview

**Class:** `org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate`

**Parameter Style:** Named `:paramName` placeholders

**Use Case:** Complex queries with multiple/reusable parameters

---

## Part 1: SELECT Operations

### 1.1 SELECT with Lambda RowMapper

Simple inline mapping for straightforward result sets.

```java
@Transactional(readOnly = true)
public List<DetailRecord> getTransactions(String settlementDate, Set<Integer> acquirerIds, Set<String> states) {

    // Prepare named parameters
    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("settlementDate", settlementDate);
    namedParameters.put("acquirerIds", acquirerIds);   // Supports IN clause with Set
    namedParameters.put("states", states);

    // SQL with named parameters
    String query = "SELECT ttr01.state, ttr01.tran_type, ttr01.target_amount, "
            + "ttr30.transaction_date, ttr30.transaction_time "
            + "FROM ttr01_transaction ttr01 "
            + "JOIN ttr30_aps_request ttr30 ON ttr30.transaction_id = ttr01.transaction_id "
            + "WHERE ttr01.bank_settle_date = :settlementDate "
            + "AND ttr01.acquirer_id IN (:acquirerIds) "
            + "AND ttr01.state IN (:states)";

    // Create NamedParameterJdbcTemplate from read DataSource
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    
    // Execute with Lambda RowMapper
    return jdbcTemplate.query(query, namedParameters, (rs, rowNum) -> {
        DetailRecord record = new DetailRecord();
        record.setState(rs.getString("state"));
        record.setTranType(rs.getString("tran_type"));
        record.setTargetAmount(rs.getBigDecimal("target_amount"));
        record.setTransactionDate(rs.getString("transaction_date"));
        record.setTransactionTime(rs.getString("transaction_time"));
        return record;
    });
}
```

---

### 1.2 SELECT with RowMapper Class

Reusable mapper for complex mapping logic.

```java
@Transactional(readOnly = true)
public List<DetailRecord> getTransactions(String settlementDate, Set<Integer> acquirersId, Set<String> states) {

    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("settlementDate", settlementDate);
    namedParameters.put("acquirerIds", acquirersId);
    namedParameters.put("states", states);

    String query = "SELECT ttr01.state, ttr01.tran_type, ttr01.target_amount, "
            + "ttr30.transaction_date, ttr30.transaction_time, ttr30.stan, "
            + "ttr30.tid, ttr30.mid, ttr30.generated_pan, ttr30.payment_type "
            + "FROM ttr01_transaction ttr01 "
            + "JOIN ttr30_aps_request ttr30 ON ttr30.transaction_id = ttr01.transaction_id "
            + "WHERE ttr01.bank_settle_date = :settlementDate "
            + "AND ttr01.acquirer_id IN (:acquirerIds) "
            + "AND ttr01.state IN (:states)";

    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    
    // Execute with RowMapper class
    return jdbcTemplate.query(query, namedParameters, new TransactionRowMapper());
}

// Reusable RowMapper class with complex logic
private class TransactionRowMapper implements RowMapper<DetailRecord> {
    @Override
    public DetailRecord mapRow(ResultSet rs, int rowNum) throws SQLException {
        DetailRecord record = new DetailRecord();
        record.setState(rs.getString("state"));
        record.setTranType(rs.getString("tran_type"));
        record.setTargetAmount(rs.getBigDecimal("target_amount"));
        record.setTransactionDate(rs.getString("transaction_date"));
        record.setTransactionTime(rs.getString("transaction_time"));
        record.setStan(rs.getString("stan"));
        record.setTerminalTid(rs.getString("tid"));
        record.setTerminalMid(rs.getString("mid"));
        
        // Conditional logic in mapper
        if (StringUtils.isEmpty(rs.getString("generated_pan"))) {
            record.setTerminalPan("9009901000000001");  // Default value
        } else {
            record.setTerminalPan(rs.getString("generated_pan"));
        }
        record.setPaymentType(rs.getString("payment_type"));
        return record;
    }
}
```

---

### 1.3 SELECT with BeanPropertyRowMapper

Auto-mapping by column alias to bean property name.

```java
@Transactional(readOnly = true)
public List<ExceptionRecord> getExceptions(String startDate, String endDate) {
    
    NamedParameterJdbcTemplate template = new NamedParameterJdbcTemplate(readOnlySource);
    
    // Column aliases MUST match bean property names exactly
    String query = "SELECT tbo.authorize_date as txnDate, "       // Maps to ExceptionRecord.txnDate
            + "tbo.stan as txnRRN, "                               // Maps to ExceptionRecord.txnRRN
            + "tbr.stan as refundRRN, "                            // Maps to ExceptionRecord.refundRRN
            + "tbr.target_amount as refundAmt, "                   // Maps to ExceptionRecord.refundAmt
            + "tbr.authorize_date as refundDate, "                 // Maps to ExceptionRecord.refundDate
            + "tbo.target_amount as txnAmount, "                   // Maps to ExceptionRecord.txnAmount
            + "tc02.currency_iso_name as currency "                // Maps to ExceptionRecord.currency
            + "FROM transaction_basics tbr "
            + "JOIN foreign_refund_request fre ON tbr.id = fre.refund_transaction_id "
            + "JOIN transaction_basics tbo ON tbo.id = fre.original_transaction_id "
            + "JOIN tc02_currency tc02 ON tbr.source_currency_id = tc02.currency_id "
            + "WHERE tbr.authorize_date >= :startDate AND tbr.authorize_date < :endDate";

    Map<String, Object> params = new HashMap<>();
    params.put("startDate", startDate);
    params.put("endDate", endDate);
    
    // BeanPropertyRowMapper auto-maps columns to bean properties by name
    return template.query(query, params, new BeanPropertyRowMapper<>(ExceptionRecord.class));
}
```

---

### 1.4 SELECT Single Value with Anonymous RowMapper

Extracting single column value.

```java
@Transactional(readOnly = true)
public String getNextFileId(String createdDate) {
    
    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("createdDate", createdDate);

    String query = "SELECT fue.FILE_ID FROM file_upload fue "
            + "WHERE fue.CREATED_DATE = :createdDate "
            + "ORDER BY fue.FILE_UPLOAD_ID DESC LIMIT 1";

    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    
    // Anonymous RowMapper for single column
    List<String> results = jdbcTemplate.query(query, namedParameters, new RowMapper<String>() {
        public String mapRow(ResultSet rs, int rowNum) throws SQLException {
            return rs.getString(1);  // Get first column
        }
    });
    
    if (results != null && results.size() > 0) {
        return results.get(0);
    }
    return "";
}
```

---

## Part 2: UPDATE Operations

### 2.1 UPDATE with MapSqlParameterSource

Using builder pattern for parameters.

```java
public void updateStatus(String state, int posId, String scheme) {

    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(writeDataSource);
    
    // Builder pattern for parameters
    SqlParameterSource namedParameters = new MapSqlParameterSource()
            .addValue("state", state)
            .addValue("posID", posId)
            .addValue("scheme", scheme);

    String updateSql = "UPDATE pos_qr SET STATE = :state WHERE POS_ID = :posID AND SCHEME = :scheme";
    
    jdbcTemplate.update(updateSql, namedParameters);
}
```

---

### 2.2 UPDATE with Multiple Parameters

Complex update with many parameters.

```java
public void updateAdjustmentStatus(LocalDateTime startDate, LocalDateTime endDate) {
    
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(writeDataSource);
    
    // Multiple parameters with MapSqlParameterSource
    SqlParameterSource namedParameters = new MapSqlParameterSource()
            .addValue("startDate", startDate)
            .addValue("endDate", endDate)
            .addValue("approvedStatus", OutboundCreditAdjustmentStatus.APPROVED.getCode())
            .addValue("tranType", "40")
            .addValue("submittedStatus", OutboundCreditAdjustmentStatus.SUBMITTED.getCode())
            .addValue("updateBy", 2)
            .addValue("currentDatetime", new Timestamp(Calendar.getInstance().getTime().getTime()));

    // Same parameter can be used multiple times (:currentDatetime)
    String updateSql = "UPDATE outbound_dispute "
            + "SET status = :submittedStatus, "
            + "updated_by = :updateBy, "
            + "updated_date_time = :currentDatetime, "
            + "closed_date = :currentDatetime "                    // Reused parameter
            + "WHERE (approval_date_time BETWEEN :startDate AND :endDate) "
            + "AND status = :approvedStatus "
            + "AND tran_type = :tranType";

    jdbcTemplate.update(updateSql, namedParameters);
}
```

---

## Scenario 2 Summary

### RowMapper Patterns

| Pattern | Use Case |
|---------|----------|
| Lambda `(rs, rowNum) -> {}` | Simple inline mapping |
| RowMapper Class | Reusable complex mapping with logic |
| BeanPropertyRowMapper | Auto-map by column alias to bean property |
| Anonymous RowMapper | Single value extraction |

### Parameter Source Options

| Class | Use Case |
|-------|----------|
| `Map<String, Object>` | Simple parameter map |
| `MapSqlParameterSource` | Builder pattern with chained `.addValue()` |

### When to Use NamedParameterJdbcTemplate

- Complex queries with multiple parameters
- Need to reuse same parameter in multiple places
- IN clause with collections (Set, List)
- Better readability for queries with many parameters

---

# Scenario 3: JdbcTemplate queryForRowSet + NamedParameterJdbcTemplate (DELETE Operations)

This scenario covers batch DELETE operations that require:
1. **JdbcTemplate.queryForRowSet()** - SELECT records to delete into disconnected SqlRowSet
2. **NamedParameterJdbcTemplate.update()** - DELETE records using IN clause with chunking

---

## Overview

**Classes Used:**
- `org.springframework.jdbc.core.JdbcTemplate` - For queryForRowSet()
- `org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate` - For DELETE with IN clause
- `org.springframework.jdbc.support.rowset.SqlRowSet` - Disconnected result set

**Use Case:** Batch deletion with:
- Pre-deletion data export (audit trail)
- Chunked deletes to avoid lock contention
- Dynamic table/column handling

---

## Part 1: SELECT with queryForRowSet()

### Purpose

Returns a disconnected `SqlRowSet` that can be processed after the database connection is released.

### 1.1 Single Table Query

```java
public SqlRowSet getSingleTableRecords(LocalDate startDate, LocalDate endDate, 
        String tableName, String primaryKey) {
    
    SqlRowSet rs = null;
    StringBuffer query = new StringBuffer();
    query.append("SELECT * FROM " + tableName + " WHERE CREATION_DATE <= '" + endDate);
    query.append("' AND CREATION_DATE > '" + startDate + "' ORDER BY " + primaryKey + " ASC");
    
    log.info("getSingleTableRecords query " + query.toString());
    rs = jdbcTemplate.queryForRowSet(query.toString());
    return rs;
}
```

### 1.2 JOIN Table Query

```java
public SqlRowSet getJoinRecords(LocalDate startDate, LocalDate endDate, String tableName, 
        String primaryKey, String joinMainColumn, String joinTable, String joinColumn) {
    
    SqlRowSet rs = null;
    String alias = (tableName.split("[.]")[1]).split("_")[0];
    String joinAlias = (joinTable.split("[.]")[1]).split("_")[0];
    
    StringBuffer query = new StringBuffer();
    query.append("SELECT " + alias + ".* FROM " + tableName + " " + alias);
    query.append(" JOIN " + joinTable + " " + joinAlias);
    query.append(" ON " + alias + "." + joinMainColumn + " = " + joinAlias + "." + joinColumn);
    query.append(" WHERE " + joinAlias + ".CREATION_DATE <= '" + endDate);
    query.append("' AND " + joinAlias + ".CREATION_DATE > '" + startDate);
    query.append("' ORDER BY " + alias + "." + primaryKey + " ASC");
    
    log.info("getJoinRecords query " + query.toString());
    rs = jdbcTemplate.queryForRowSet(query.toString());
    return rs;
}
```

### 1.3 Single Record Query

```java
public SqlRowSet getSingleRecord(String tableName, String columnName, Long id) {
    
    SqlRowSet rs = null;
    StringBuffer query = new StringBuffer();
    query.append("SELECT * FROM " + tableName + " WHERE " + columnName + " = " + id);
    
    log.info("getSingleRecord query " + query.toString());
    rs = jdbcTemplate.queryForRowSet(query.toString());
    return rs;
}
```

---

## Part 2: SqlRowSet Processing

### Purpose

Extract data from SqlRowSet for export and collect IDs for deletion.

### 2.1 SqlRowSetMetaData - Dynamic Column Discovery

```java
SqlRowSet rs = jdbcTemplate.queryForRowSet(query.toString());
int columnNo = rs.getMetaData().getColumnCount();

// Find primary key column index
int primaryKeyNo = 1;
for (int i = 1; i <= columnNo; i++) {
    String columnName = rs.getMetaData().getColumnName(i);
    if (columnName.equalsIgnoreCase(tableKey)) {
        primaryKeyNo = i;
        log.debug("primary key index is " + primaryKeyNo);
    }
}
```

### SqlRowSetMetaData Methods

| Method | Description |
|--------|-------------|
| getColumnCount() | Returns number of columns |
| getColumnName(int) | Returns column name by index (1-based) |
| getColumnLabel(int) | Returns column label/alias |
| getColumnType(int) | Returns SQL type (java.sql.Types) |

### 2.2 Build CSV Header from Metadata

```java
StringBuilder recordsDump = new StringBuilder();

// Build header row from column names
for (int i = 1; i <= columnNo; i++) {
    String columnName = rs.getMetaData().getColumnName(i);
    recordsDump.append(columnName + ",");
    if (columnName.equalsIgnoreCase(tableKey)) {
        primaryKeyNo = i;
    }
}
recordsDump.append("\n");
```

### 2.3 Extract Data and Collect IDs

```java
List<Long> deletionIDs = new ArrayList<>();
List<String> deletionIDsString = new ArrayList<>();

while (rs.next()) {
    // Export each row to CSV
    for (int x = 1; x <= columnNo; x++) {
        String value = rs.getString(x);
        
        // Collect primary key for deletion
        if (x == primaryKeyNo) {
            try {
                deletionIDs.add(Long.valueOf(value));
            } catch (NumberFormatException ex) {
                deletionIDsString.add(value);  // Handle String PKs
            }
        }
        recordsDump.append(value + ", ");
    }
    recordsDump.append("\n");
}
```

### SqlRowSet Navigation Methods

| Method | Description |
|--------|-------------|
| next() | Moves cursor to next row, returns false if no more rows |
| previous() | Moves cursor to previous row |
| first() | Moves cursor to first row |
| last() | Moves cursor to last row |
| getString(int/String) | Gets column value as String |
| getLong(int/String) | Gets column value as Long |
| getDate(int/String) | Gets column value as Date |

---

## Part 3: DELETE Operations

### 3.1 Batch DELETE with IN Clause (Long IDs)

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void deleteRecords(String tableName, String primaryKey, List<Long> idList) {
    
    StringBuffer query = new StringBuffer();
    query.append("DELETE FROM " + tableName + " WHERE " + primaryKey + " IN (:ids)");
    
    NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
    Map<String, List<Long>> params = new HashMap<String, List<Long>>();
    params.put("ids", idList);
    
    int records = namedParameterJdbcTemplate.update(query.toString(), params);
    log.info("deleted " + records + " in " + tableName);
}
```

### 3.2 Batch DELETE with IN Clause (String IDs)

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void deleteRecordsWithStringId(String tableName, String primaryKey, List<String> idList) {
    
    StringBuffer query = new StringBuffer();
    query.append("DELETE FROM " + tableName + " WHERE " + primaryKey + " IN (:ids)");
    
    NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
    Map<String, List<String>> params = new HashMap<String, List<String>>();
    params.put("ids", idList);
    
    int records = namedParameterJdbcTemplate.update(query.toString(), params);
    log.info("deleted " + records + " in " + tableName);
}
```

### 3.3 Single Record DELETE

```java
public void deleteRecord(String tableName, String primaryKey, Long id) {
    
    StringBuffer query = new StringBuffer();
    query.append("DELETE FROM " + tableName + " WHERE " + primaryKey + " = :id");
    
    NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
    Map<String, Long> params = new HashMap<String, Long>();
    params.put("id", id);
    
    namedParameterJdbcTemplate.update(query.toString(), params);
}
```

### 3.4 JOIN DELETE

```java
public void deleteJoinRecord(String tableName, String joinTable, String mainColumn, 
        String joinColumn, String primaryKey, Long id) {
    
    StringBuffer query = new StringBuffer();
    String alias = tableName.split("_")[0];
    String joinAlias = joinTable.split("_")[0];
    
    query.append("DELETE " + joinAlias + " FROM " + tableName + " " + alias);
    query.append(" JOIN " + joinTable + " " + joinAlias);
    query.append(" ON " + alias + "." + mainColumn + " = " + joinAlias + "." + joinColumn);
    query.append(" WHERE " + alias + "." + primaryKey + " = :id");
    
    NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
    Map<String, Long> params = new HashMap<String, Long>();
    params.put("id", id);
    
    namedParameterJdbcTemplate.update(query.toString(), params);
}
```

---

## Part 4: Chunked Delete Execution Flow

### Purpose

Delete large datasets in chunks to:
- Avoid long-running transactions
- Reduce lock contention
- Allow DB replication to catch up
- Enable partial success on failure

### Actual Implementation

```java
// Step 1: Query records to delete
SqlRowSet rs = repository.getSingleTableRecords(deletionDate, intervalDate, tableName, tableKey);

// Step 2-3: Extract IDs from SqlRowSet
List<Long> deletionIDs = new ArrayList<>();
while (rs.next()) {
    deletionIDs.add(Long.valueOf(rs.getString(primaryKeyNo)));
}

// Step 4: Partition into chunks
List<List<Long>> partitions = new ArrayList<>();
for (int i = 0; i < deletionIDs.size(); i += chunkSize) {
    partitions.add(deletionIDs.subList(i, Math.min(i + chunkSize, deletionIDs.size())));
}

// Step 5: Delete each chunk with throttling
int deletedRecords = 0;
for (List<Long> chunk : partitions) {
    repository.deleteRecords(tableName, tableKey, chunk);
    deletedRecords += chunk.size();
    
    // Throttle after reaching threshold
    if (deletedRecords >= deletionThreshold) {
        Thread.sleep(TimeUnit.SECONDS.toMillis(intervalTime));
        deletedRecords = 0;
    }
}
```

---

## Scenario 3 Summary

### Components Used

| Component | Purpose |
|-----------|---------|
| JdbcTemplate.queryForRowSet() | SELECT records into disconnected SqlRowSet |
| SqlRowSet | Process results without holding connection |
| SqlRowSetMetaData | Dynamic column discovery |
| NamedParameterJdbcTemplate.update() | DELETE with IN clause |
| @Transactional(REQUIRES_NEW) | Independent transaction per chunk |

### When to Use This Pattern

- Batch deletion of large datasets
- Need audit trail (export before delete)
- Dynamic table/column names
- Want to avoid long-running transactions
- Need throttling for DB replication

### Key Benefits

| Benefit | How Achieved |
|---------|--------------|
| Audit Trail | Export SqlRowSet to CSV before delete |
| No Lock Contention | Chunked deletes with sleep intervals |
| Partial Success | REQUIRES_NEW commits each chunk independently |
| Memory Efficient | SqlRowSet is disconnected, connection released early |
| Replication Safe | Sleep between chunks allows replicas to catch up |

---

# Overall Summary

## Scenario Comparison

| Aspect | Scenario 1: JdbcTemplate | Scenario 2: NamedParameter | Scenario 3: queryForRowSet + NamedParameter |
|--------|--------------------------|----------------------------|---------------------------------------------|
| Parameter Style | Positional `?` | Named `:param` | Named `:param` |
| Primary Operation | INSERT | SELECT/UPDATE | DELETE (batch) |
| Result Handling | N/A | RowMapper | SqlRowSet |
| Transaction | Single | Single | Chunked (REQUIRES_NEW) |
| Best For | Simple INSERT | Complex SELECT/UPDATE | Large batch DELETE |

## DataSource Usage

| Operation | DataSource | JdbcTemplate Type |
|-----------|------------|-------------------|
| SELECT | readOnlySource (batchDatasource) | NamedParameterJdbcTemplate or JdbcTemplate |
| INSERT | writeDataSource (springDatasource) | JdbcTemplate |
| UPDATE | writeDataSource (springDatasource) | NamedParameterJdbcTemplate |
| DELETE | writeDataSource (via JdbcTemplate) | NamedParameterJdbcTemplate |

## Quick Reference

### Scenario 1: JdbcTemplate (Positional)

```java
JdbcTemplate template = new JdbcTemplate(writeDataSource);
template.update("INSERT INTO table (col1, col2) VALUES (?, ?)", value1, value2);
```

### Scenario 2: NamedParameterJdbcTemplate (Named)

```java
NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(dataSource);
Map<String, Object> params = new HashMap<>();
params.put("param1", value1);
jdbcTemplate.query("SELECT * FROM table WHERE col = :param1", params, rowMapper);
```

### Scenario 3: queryForRowSet + NamedParameterJdbcTemplate (DELETE)

```java
// Step 1: Query with queryForRowSet
SqlRowSet rs = jdbcTemplate.queryForRowSet("SELECT id FROM table WHERE date < ?", date);

// Step 2: Collect IDs
List<Long> ids = new ArrayList<>();
while (rs.next()) {
    ids.add(rs.getLong("id"));
}

// Step 3: Delete with NamedParameterJdbcTemplate
NamedParameterJdbcTemplate npTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
Map<String, List<Long>> params = new HashMap<>();
params.put("ids", ids);
npTemplate.update("DELETE FROM table WHERE id IN (:ids)", params);
```
