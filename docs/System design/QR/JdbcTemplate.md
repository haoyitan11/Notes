# JdbcTemplate Dependencies Handle

## Overview

This document provides a complete guide for the JdbcTemplate implementation in Java applications using Spring Framework. It covers three main patterns:

1. **Basic JdbcTemplate Operations** - Simple query and update operations using JdbcTemplate

2. **NamedParameterJdbcTemplate Operations** - Parameter-based queries with named placeholders

3. **SqlRowSet Processing** - Disconnected result set handling for data extraction

---

# Part 1: Core Infrastructure

## Purpose

Provides the foundational data access layer between the application and the relational database using Spring JDBC.

---

## Components

### 1.1 DataSource (Auto-Configuration)

#### Purpose

Provides connection pooling and database connectivity configuration.

#### Dependencies

```java
javax.sql.DataSource
```

#### Configuration (application.properties)

```properties
spring.datasource.driverClassName=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://db-host:3306/database_name?useSSL=true&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=username
spring.datasource.password=${DATABASE_PASSWORD}
```

#### Connection Pool Properties

```properties
spring.datasource.tomcat.test-while-idle=true
spring.datasource.tomcat.test-on-borrow=true
spring.datasource.tomcat.validation-query=SELECT 1
```

#### Responsibilities

- Manages database connection pooling.
- Provides connections to JdbcTemplate.
- Handles connection lifecycle and validation.
- Auto-configured by Spring Boot via **DataSourceAutoConfiguration**.

---

### 1.2 JdbcTemplate

#### Purpose

Simplifies JDBC operations by handling resource management, exception translation, and statement execution.

#### Dependencies

```java
org.springframework.jdbc.core.JdbcTemplate
javax.sql.DataSource
```

#### Actual Usage

```java
@Repository
public class OneoffRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
}
```

#### Responsibilities

- Integrates **DataSource** (obtains connections automatically).
- Executes SQL queries and updates.
- Handles resource cleanup (connections, statements, result sets).
- Translates SQL exceptions to Spring **DataAccessException** hierarchy.
- Provides template methods for common JDBC operations.

#### Configuration Source

Auto-configured by Spring Boot when spring-boot-starter-jdbc or spring-boot-starter-data-jpa is present:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

Or explicitly:

```xml
<dependency>
    <groupId>org.springframework.batch</groupId>
    <artifactId>spring-batch-infrastructure</artifactId>
</dependency>
```

---

### 1.3 NamedParameterJdbcTemplate

#### Purpose

Extends JdbcTemplate to support named parameters instead of traditional "?" placeholders.

#### Dependencies

```java
org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
org.springframework.jdbc.core.JdbcTemplate
java.util.Map
```

#### Actual Usage

```java
NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
Map<String, List<Long>> params = new HashMap<String, List<Long>>();
params.put("ids", idList);
int records = namedParameterJdbcTemplate.update(query.toString(), params);
```

#### Responsibilities

- Integrates **JdbcTemplate** (wraps for named parameter support).
- Supports named parameters using Map or SqlParameterSource.
- Translates named parameters to positional parameters internally.
- Provides IN clause support for collections.

---

### 1.4 SqlRowSet

#### Purpose

Provides a disconnected, scrollable result set that doesn't require active database connection.

#### Dependencies

```java
org.springframework.jdbc.support.rowset.SqlRowSet
org.springframework.jdbc.support.rowset.SqlRowSetMetaData
```

#### Actual Usage

```java
SqlRowSet rs = jdbcTemplate.queryForRowSet(query.toString());
while (rs.next()) {
    String value = rs.getString(columnIndex);
}
```

#### Responsibilities

- Provides disconnected result set (CachedRowSet wrapper).
- Supports scrollable cursor navigation.
- Provides metadata access via **SqlRowSetMetaData**.
- Allows result processing after connection is closed.

---

# Part 2: Query Operations

## Purpose

Provides various methods for executing SELECT queries and retrieving data.

---

## 2.1 queryForObject()

### Purpose

Executes a query that returns exactly one row and maps it to a single object.

### Dependencies

```java
org.springframework.jdbc.core.JdbcTemplate
java.lang.Class<T>
```

### Actual Usage

```java
String minDate = jdbcTemplate.queryForObject(query.toString(), String.class);
```

### Query Example

```java
public LocalDate getEarliestDate(String table) {
    LocalDate date = LocalDate.now();
    StringBuffer query = new StringBuffer();
    query.append("SELECT date(min(creation_date)) FROM " + table);
    log.info("getEarliestDate query " + query.toString());
    
    String minDate = jdbcTemplate.queryForObject(query.toString(), String.class);
    if(StringUtils.isNotEmpty(minDate)) {
        date = LocalDate.parse(minDate, DateTimeFormatter.ofPattern("yyyy-MM-dd"));
    }
    return date;
}
```

### Responsibilities

- Executes single-value queries (COUNT, MIN, MAX, etc.).
- Maps result to specified type using standard type converters.
- Throws **EmptyResultDataAccessException** if no rows found.
- Throws **IncorrectResultSizeDataAccessException** if multiple rows found.

### Supported Return Types

| Type | Usage |
|------|-------|
| String.class | Text values |
| Integer.class | Numeric values |
| Long.class | Large numeric values |
| Date.class | Date values |
| BigDecimal.class | Decimal values |

---

## 2.2 queryForList()

### Purpose

Executes a query that returns multiple rows, each mapped to a single column value.

### Dependencies

```java
org.springframework.jdbc.core.JdbcTemplate
java.util.List<T>
java.lang.Class<T>
```

### Actual Usage

```java
List<String> inactiveMerchants = jdbcTemplate.queryForList(query.toString(), String.class);
```

### Query Example

```java
public List<String> getInactiveMerchants() {
    List<String> inactiveMerchants = new ArrayList<>();
    StringBuffer query = new StringBuffer();
    query.append("SELECT TM01.MERCHANT_ID FROM TM01_MERCHANT TM01 ");
    query.append("WHERE TM01.STATUS = 3 AND TM01.LAST_UPDATE_DATE < ");
    query.append("DATE_SUB(NOW(), INTERVAL " + this.merchantRetentionRange + ") ");
    query.append("AND NOT EXISTS (SELECT 1 FROM TTR01_TRANSACTION TTR01 ");
    query.append("WHERE TTR01.MERCHANT_ID = TM01.MERCHANT_ID)");
    
    inactiveMerchants = jdbcTemplate.queryForList(query.toString(), String.class);
    return inactiveMerchants;
}
```

### Responsibilities

- Executes queries returning multiple single-column values.
- Maps each row to specified element type.
- Returns empty list if no rows found.
- Suitable for ID lists, code lookups, etc.

---

## 2.3 queryForRowSet()

### Purpose

Executes a query and returns results as a disconnected SqlRowSet.

### Dependencies

```java
org.springframework.jdbc.core.JdbcTemplate
org.springframework.jdbc.support.rowset.SqlRowSet
```

### Actual Usage

```java
SqlRowSet rs = jdbcTemplate.queryForRowSet(query.toString());
```

### Query Example (Single Table)

```java
public SqlRowSet getSingleTableRecords(LocalDate startDate, LocalDate endDate, 
        String tableName, String primaryKey) {
    SqlRowSet rs = null;
    StringBuffer query = new StringBuffer();
    query.append("SELECT * FROM " + tableName + " WHERE CREATION_DATE <= '" + endDate);
    query.append("' AND CREATION_DATE > '"+ startDate + "' ORDER BY " + primaryKey + " ASC");
    log.info("getSingleTableRecords query " + query.toString());
    
    rs = jdbcTemplate.queryForRowSet(query.toString());
    return rs;
}
```

### Query Example (JOIN)

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

### Responsibilities

- Returns disconnected result set.
- Supports dynamic schema inspection via metadata.
- Allows deferred processing without holding connection.
- Suitable for batch processing and data export.

---

# Part 3: SqlRowSet Processing

## Purpose

Processes disconnected result sets with metadata inspection and row iteration.

---

## 3.1 SqlRowSet Navigation

### Purpose

Iterates through result set rows.

### Dependencies

```java
org.springframework.jdbc.support.rowset.SqlRowSet
```

### Actual Usage

```java
while (rs.next()) {
    for (int x = 1; x <= columnNo; x++) {
        String value = rs.getString(x);
        // Process value
    }
}
```

### Available Navigation Methods

| Method | Description |
|--------|-------------|
| next() | Moves cursor to next row, returns false if no more rows |
| previous() | Moves cursor to previous row |
| first() | Moves cursor to first row |
| last() | Moves cursor to last row |
| beforeFirst() | Moves cursor before first row |
| afterLast() | Moves cursor after last row |
| absolute(int) | Moves cursor to absolute row position |
| relative(int) | Moves cursor relative to current position |

---

## 3.2 SqlRowSetMetaData

### Purpose

Provides schema information about the result set columns.

### Dependencies

```java
org.springframework.jdbc.support.rowset.SqlRowSet
org.springframework.jdbc.support.rowset.SqlRowSetMetaData
```

### Actual Usage

```java
SqlRowSet rs = jdbcTemplate.queryForRowSet(query.toString());
int columnNo = rs.getMetaData().getColumnCount();

for(int i = 1; i <= columnNo; i++) {
    String columnName = rs.getMetaData().getColumnName(i);
    if(columnName.equalsIgnoreCase(tableKey)) {
        primaryKeyNo = i;
        log.debug("primary key index is " + primaryKeyNo);
    }
}
```

### Header Generation Example

```java
StringBuilder recordsDump = new StringBuilder();
for(int i = 1; i <= columnNo; i++) {
    String columnName = rs.getMetaData().getColumnName(i);
    recordsDump.append(columnName + ",");
}
recordsDump.append("\n");
```

### Available Metadata Methods

| Method | Description |
|--------|-------------|
| getColumnCount() | Returns number of columns |
| getColumnName(int) | Returns column name by index (1-based) |
| getColumnLabel(int) | Returns column label/alias |
| getColumnType(int) | Returns SQL type (java.sql.Types) |
| getColumnTypeName(int) | Returns database-specific type name |
| getTableName(int) | Returns table name for column |
| getPrecision(int) | Returns precision for numeric columns |
| getScale(int) | Returns scale for decimal columns |

---

## 3.3 SqlRowSet Data Extraction

### Purpose

Extracts column values from current row.

### Dependencies

```java
org.springframework.jdbc.support.rowset.SqlRowSet
```

### Actual Usage (By Index)

```java
while (rs.next()) {
    for (int x = 1; x <= columnNo; x++) {
        String value = rs.getString(x);
        recordsDump.append(value + ", ");          
    }
    recordsDump.append("\n");
}
```

### Actual Usage (By Column Name)

```java
while (rs.next()) {
    String merchantId = rs.getString("MERCHANT_ID");
    Long transactionId = rs.getLong("TRANSACTION_ID");
    Date creationDate = rs.getDate("CREATION_DATE");
}
```

### Available Data Extraction Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| getString(int/String) | String | Gets column as String |
| getInt(int/String) | int | Gets column as int |
| getLong(int/String) | long | Gets column as long |
| getDouble(int/String) | double | Gets column as double |
| getBigDecimal(int/String) | BigDecimal | Gets column as BigDecimal |
| getDate(int/String) | Date | Gets column as java.sql.Date |
| getTimestamp(int/String) | Timestamp | Gets column as Timestamp |
| getBoolean(int/String) | boolean | Gets column as boolean |
| getObject(int/String) | Object | Gets column as Object |

---

# Part 4: Update Operations

## Purpose

Provides methods for executing INSERT, UPDATE, and DELETE statements.

---

## 4.1 NamedParameterJdbcTemplate.update() with IN Clause

### Purpose

Executes DELETE/UPDATE with collection parameters using IN clause.

### Dependencies

```java
org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
org.springframework.jdbc.core.JdbcTemplate
java.util.Map
java.util.List
```

### Actual Usage (Long IDs)

```java
@Transactional(propagation=Propagation.REQUIRES_NEW)
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

### Actual Usage (String IDs)

```java
@Transactional(propagation=Propagation.REQUIRES_NEW)
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

### Responsibilities

- Handles collection parameters for IN clauses.
- Automatically expands `:ids` to proper SQL syntax.
- Returns affected row count.
- Supports any Collection type (List, Set, etc.).

---

## 4.2 Single Record Delete

### Purpose

Deletes a single record by primary key.

### Dependencies

```java
org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
org.springframework.jdbc.core.JdbcTemplate
java.util.Map
```

### Actual Usage

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

---

## 4.3 JOIN Delete

### Purpose

Deletes records from a table based on a JOIN condition.

### Dependencies

```java
org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
org.springframework.jdbc.core.JdbcTemplate
java.util.Map
```

### Actual Usage

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

# Part 5: Transaction Management

## Purpose

Provides declarative transaction control for database operations.

---

## 5.1 @Transactional Annotation

### Purpose

Demarcates transactional boundaries with configurable propagation behavior.

### Dependencies

```java
org.springframework.transaction.annotation.Transactional
org.springframework.transaction.annotation.Propagation
```

### Actual Usage

```java
@Transactional(propagation=Propagation.REQUIRES_NEW)
public void deleteRecords(String tableName, String primaryKey, List<Long> idList) {
    // Delete operation
}
```

### Propagation Types

| Propagation | Description |
|-------------|-------------|
| REQUIRED | Uses current transaction, creates new if none exists (default) |
| REQUIRES_NEW | Always creates new transaction, suspends current if exists |
| SUPPORTS | Uses current transaction if exists, executes non-transactionally if not |
| NOT_SUPPORTED | Executes non-transactionally, suspends current transaction if exists |
| MANDATORY | Uses current transaction, throws exception if none exists |
| NEVER | Executes non-transactionally, throws exception if transaction exists |
| NESTED | Creates nested transaction if current exists, creates new if not |

### Responsibilities

- Manages transaction lifecycle automatically.
- Commits on successful method completion.
- Rolls back on unchecked exceptions.
- Integrates with **PlatformTransactionManager**.

---

# Part 6: Execution Flows

## 6.1 Query Execution Flow

1. **Repository Method** builds SQL query string:
   ```java
   StringBuffer query = new StringBuffer();
   query.append("SELECT date(min(creation_date)) FROM " + table);
   ```

2. **JdbcTemplate.queryForObject()** is called:
   ```java
   String minDate = jdbcTemplate.queryForObject(query.toString(), String.class);
   ```
   - JdbcTemplate obtains **Connection** from **DataSource**.
   - Creates **PreparedStatement** from query.
   - Executes query and retrieves **ResultSet**.
   - Maps single result to target type.
   - Closes resources (ResultSet, Statement, Connection).

3. **Result Processing**:
   ```java
   if(StringUtils.isNotEmpty(minDate)) {
       date = LocalDate.parse(minDate, DateTimeFormatter.ofPattern("yyyy-MM-dd"));
   }
   ```

---

## 6.2 Batch Delete Execution Flow

1. **Tasklet** retrieves records to delete:
   ```java
   SqlRowSet rs = repository.getSingleTableRecords(deletionDate, intervalDate, tableName, tableKey);
   ```

2. **SqlRowSet Processing** extracts IDs:
   ```java
   List<Long> deletionIDs = new ArrayList<>();
   while (rs.next()) {
       deletionIDs.add(Long.valueOf(rs.getString(primaryKeyNo)));
   }
   ```

3. **Chunking** partitions IDs for batch delete:
   ```java
   List<List<Long>> partitions = new ArrayList<>();
   for (int i=0; i < deletionIDs.size(); i += tableThresholdChunk) {
       partitions.add(deletionIDs.subList(i, Math.min(i + tableThresholdChunk, deletionIDs.size())));
   }
   ```

4. **Repository.deleteRecords()** executes batched deletes:
   ```java
   for(List<Long> list : partitions) {
       repository.deleteRecords(tableName, tableKey, list);
       deletedRecords += list.size();
       if(deletedRecords >= tableDeletionSize) {
           Thread.sleep(TimeUnit.SECONDS.toMillis(tableIntervalTime));
           deletedRecords = 0;
       }
   }
   ```

5. **NamedParameterJdbcTemplate.update()** executes each batch:
   ```java
   NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
   Map<String, List<Long>> params = new HashMap<>();
   params.put("ids", idList);
   int records = namedParameterJdbcTemplate.update(query.toString(), params);
   ```

---

## 6.3 SqlRowSet Data Export Flow

1. **Query Execution** returns SqlRowSet:
   ```java
   SqlRowSet rs = jdbcTemplate.queryForRowSet(query.toString());
   ```

2. **Metadata Extraction** builds header:
   ```java
   int columnNo = rs.getMetaData().getColumnCount();
   StringBuilder recordsDump = new StringBuilder();
   for(int i = 1; i <= columnNo; i++) {
       String columnName = rs.getMetaData().getColumnName(i);
       recordsDump.append(columnName + ",");
   }
   recordsDump.append("\n");
   ```

3. **Row Iteration** exports data:
   ```java
   while (rs.next()) {
       for (int x = 1; x <= columnNo; x++) {
           String value = rs.getString(x);
           recordsDump.append(value + ", ");          
       }
       recordsDump.append("\n");
   }
   ```

4. **File Writing** persists export:
   ```java
   createDeleteDump(tableName, recordsDump.toString(), startDate, endDate);
   ```

---

# Summary

## JdbcTemplate Method Reference

| Method | Purpose | Return Type |
|--------|---------|-------------|
| queryForObject(sql, type) | Single value query | T |
| queryForList(sql, type) | Multiple single-column values | List<T> |
| queryForRowSet(sql) | Disconnected result set | SqlRowSet |
| queryForMap(sql) | Single row as Map | Map<String, Object> |
| query(sql, rowMapper) | Multiple rows with mapping | List<T> |
| update(sql) | INSERT/UPDATE/DELETE | int (affected rows) |
| batchUpdate(sql, params) | Batch operations | int[] |
| execute(sql) | DDL statements | void |

## NamedParameterJdbcTemplate Method Reference

| Method | Purpose | Return Type |
|--------|---------|-------------|
| queryForObject(sql, params, type) | Single value query with named params | T |
| queryForList(sql, params, type) | Multiple values with named params | List<T> |
| queryForRowSet(sql, params) | Disconnected result set | SqlRowSet |
| query(sql, params, rowMapper) | Multiple rows with mapping | List<T> |
| update(sql, params) | INSERT/UPDATE/DELETE with named params | int |
| batchUpdate(sql, params[]) | Batch operations | int[] |

## SqlRowSet Method Reference

| Category | Method | Description |
|----------|--------|-------------|
| Navigation | next(), previous(), first(), last() | Cursor movement |
| Position | getRow(), isFirst(), isLast(), isBeforeFirst() | Position checking |
| Data | getString(), getInt(), getLong(), getDate() | Value extraction |
| Metadata | getMetaData() | Schema information |

## Configuration Properties Reference

### DataSource Properties

```properties
spring.datasource.driverClassName=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://host:3306/database
spring.datasource.username=user
spring.datasource.password=password
```

### Connection Pool Properties

```properties
spring.datasource.tomcat.test-while-idle=true
spring.datasource.tomcat.test-on-borrow=true
spring.datasource.tomcat.validation-query=SELECT 1
spring.datasource.tomcat.max-active=50
spring.datasource.tomcat.max-idle=10
spring.datasource.tomcat.min-idle=5
spring.datasource.tomcat.initial-size=5
```
