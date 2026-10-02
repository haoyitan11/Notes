# JdbcTemplate Dependencies Handle

## Overview

This document provides a complete guide for database connectivity using JdbcTemplate in Spring applications. It covers two scenarios:

- **Scenario 1:** JdbcTemplate without Spring Batch (standalone database operations)
- **Scenario 2:** JdbcTemplate with Spring Batch (batch job integration)

---

# Scenario 1: JdbcTemplate Without Batch

This scenario covers standalone JdbcTemplate usage for direct database operations without Spring Batch framework.

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

Injects DataSource beans and creates JdbcTemplate for database operations.

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

## Part 4: JdbcTemplate Usage Patterns

### 4.1 SELECT with Lambda RowMapper

```java
@Transactional(readOnly = true)
public List<DetailRecord> getTransactions(String settlementDate, Set<Integer> acquirerIds, Set<String> states) {

    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("settlementDate", settlementDate);
    namedParameters.put("acquirerIds", acquirerIds);
    namedParameters.put("states", states);

    String query = "SELECT ttr01.state, ttr01.tran_type, ttr01.target_amount, "
            + "ttr30.transaction_date, ttr30.transaction_time "
            + "FROM ttr01_transaction ttr01 "
            + "JOIN ttr30_aps_request ttr30 ON ttr30.transaction_id = ttr01.transaction_id "
            + "WHERE ttr01.bank_settle_date = :settlementDate "
            + "AND ttr01.acquirer_id IN (:acquirerIds) "
            + "AND ttr01.state IN (:states)";

    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    
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

### 4.2 SELECT with RowMapper Class

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
    return jdbcTemplate.query(query, namedParameters, new TransactionRowMapper());
}

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
        
        if (StringUtils.isEmpty(rs.getString("generated_pan"))) {
            record.setTerminalPan("9009901000000001");
        } else {
            record.setTerminalPan(rs.getString("generated_pan"));
        }
        record.setPaymentType(rs.getString("payment_type"));
        return record;
    }
}
```

---

### 4.3 SELECT with BeanPropertyRowMapper

```java
@Transactional(readOnly = true)
public List<ExceptionRecord> getExceptions(String startDate, String endDate) {
    
    NamedParameterJdbcTemplate template = new NamedParameterJdbcTemplate(readOnlySource);
    
    // Column aliases must match bean property names exactly
    String query = "SELECT tbo.authorize_date as txnDate, "
            + "tbo.stan as txnRRN, "
            + "tbr.stan as refundRRN, "
            + "tbr.target_amount as refundAmt, "
            + "tbr.authorize_date as refundDate, "
            + "tbo.target_amount as txnAmount, "
            + "tc02.currency_iso_name as currency "
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

### 4.4 SELECT Single Value with Anonymous RowMapper

```java
@Transactional(readOnly = true)
public String getNextFileId(String createdDate) {
    
    Map<String, Object> namedParameters = new HashMap<>();
    namedParameters.put("createdDate", createdDate);

    String query = "SELECT fue.FILE_ID FROM file_upload fue "
            + "WHERE fue.CREATED_DATE = :createdDate "
            + "ORDER BY fue.FILE_UPLOAD_ID DESC LIMIT 1";

    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    
    List<String> results = jdbcTemplate.query(query, namedParameters, new RowMapper<String>() {
        public String mapRow(ResultSet rs, int rowNum) throws SQLException {
            return rs.getString(1);
        }
    });
    
    if (results != null && results.size() > 0) {
        return results.get(0);
    }
    return "";
}
```

---

### 4.5 INSERT with JdbcTemplate

```java
public void createFileUploadEntry(String createdDate, String directory, String fileName, 
        String fileId, Integer status, Integer lastUpdatedBy, Timestamp lastUpdatedDate) {
    
    JdbcTemplate template = new JdbcTemplate(writeDataSource);
    
    String query = "INSERT INTO file_upload "
            + "(CREATED_DATE, DIRECTORY, FILENAME, STATUS, LAST_UPDATED_BY, LAST_UPDATED_DATE, FILE_ID) "
            + "VALUES (?, ?, ?, ?, ?, ?, ?)";
    
    template.update(query, createdDate, directory, fileName, status, 
            lastUpdatedBy, lastUpdatedDate, fileId);
}
```

---

### 4.6 UPDATE with NamedParameterJdbcTemplate

```java
public void updateStatus(String state, int posId, String scheme) {

    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(writeDataSource);
    
    SqlParameterSource namedParameters = new MapSqlParameterSource()
            .addValue("state", state)
            .addValue("posID", posId)
            .addValue("scheme", scheme);

    String updateSql = "UPDATE pos_qr SET STATE = :state WHERE POS_ID = :posID AND SCHEME = :scheme";
    
    jdbcTemplate.update(updateSql, namedParameters);
}
```

---

### 4.7 UPDATE with Multiple Parameters

```java
public void updateAdjustmentStatus(LocalDateTime startDate, LocalDateTime endDate) {
    
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(writeDataSource);
    
    SqlParameterSource namedParameters = new MapSqlParameterSource()
            .addValue("startDate", startDate)
            .addValue("endDate", endDate)
            .addValue("approvedStatus", "APPROVED")
            .addValue("tranType", "40")
            .addValue("submittedStatus", "SUBMITTED")
            .addValue("updateBy", 2)
            .addValue("currentDatetime", new Timestamp(Calendar.getInstance().getTime().getTime()));

    String updateSql = "UPDATE outbound_dispute "
            + "SET status = :submittedStatus, "
            + "updated_by = :updateBy, "
            + "updated_date_time = :currentDatetime, "
            + "closed_date = :currentDatetime "
            + "WHERE (approval_date_time BETWEEN :startDate AND :endDate) "
            + "AND status = :approvedStatus "
            + "AND tran_type = :tranType";

    jdbcTemplate.update(updateSql, namedParameters);
}
```

---

## Part 5: Transaction Management

### @Transactional(readOnly = true)

```java
@Transactional(readOnly = true)
public List<DetailRecord> getTransactions(String date) {
    NamedParameterJdbcTemplate jdbcTemplate = new NamedParameterJdbcTemplate(readOnlySource);
    // ... SELECT query
}
```

**What readOnly = true Does:**
- Hints to the database that no writes will occur
- Database may optimize (no write locks, can use read replicas)
- Spring will throw exception if UPDATE/INSERT attempted
- Connection returned to pool in read-only state

---

## Scenario 1 Summary

### Complete Linkage Chain

1. **Properties File** → Defines connection parameters (jdbcUrl, username, password, HikariCP settings)
2. **DatasourceConfiguration** → `@ConfigurationProperties` binds properties to DataSource beans
3. **Repository** → `@Autowired @Qualifier` injects specific DataSource beans
4. **JdbcTemplate** → Created from DataSource, executes SQL via HikariCP connection pool
5. **Database** → MySQL receives SQL queries via JDBC connections

### Quick Reference

| Component | Purpose |
|-----------|---------|
| Properties | Define database connection settings |
| DatasourceConfiguration | Create DataSource beans |
| Repository | Inject DataSource, create JdbcTemplate |
| JdbcTemplate | Execute SQL queries |

---

# Scenario 2: JdbcTemplate With Spring Batch

This scenario covers JdbcTemplate usage integrated with Spring Batch framework for batch job processing.

---

## Part 1: Spring Batch + Repository Integration

Spring Batch components (Tasklet and ItemReader) access the database through the Repository layer, which internally uses JdbcTemplate.

### Linkage Chain

1. **Job Configuration** → Defines Job and Step beans
2. **Tasklet/ItemReader** → Injects Repository via `@Autowired`
3. **Repository** → Uses JdbcTemplate to execute SQL
4. **Database** → Receives queries via HikariCP connection pool

---

## Part 2: Tasklet Pattern

Tasklet executes a single batch operation (not chunk-based). Used for file generation, bulk updates, or report generation.

### Source Reference

**File:** `com.example.batch.job.tasklet.ReportTasklet`

### Actual Usage

```java
@Component
public class ReportTasklet implements Tasklet {

    @Autowired
    BatchRepository repository;

    @Autowired
    BatchService service;

    @Value("${batch.output.dir.generated}")
    protected String generatedFilePath;

    @Value("${batch.output.dir.sent}")
    protected String sentFilePath;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
        
        // 1. Get job parameters from ChunkContext
        String inputDateStr = (String) chunkContext.getStepContext()
                .getJobParameters().get("inputDateStr");
        
        // 2. Parse and calculate dates
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern("yyyyMMdd");
        String processDate = LocalDate.parse(inputDateStr, dateFormatter)
                .minusDays(1).format(dateFormatter);
        
        LocalDateTime startDate = LocalDate.parse(processDate, dateFormatter).atStartOfDay();
        LocalDateTime endDate = LocalDate.parse(processDate, dateFormatter).atTime(LocalTime.MAX);

        // 3. Query database via Repository (Repository uses JdbcTemplate internally)
        List<DetailRecord> records = repository.getTransactions(startDate, endDate);
        
        // 4. Process records - write to file
        try {
            File outputFile = writeFile(records, inputDateStr);
            String sentFolderDir = sentFilePath + File.separator + inputDateStr;
            service.processFile(outputFile.getName(), sentFolderDir, records);
        } catch (IOException e) {
            logger.error("Error during write file " + e.getMessage(), e);
        }
        
        // 5. Update database status via Repository
        if (!records.isEmpty()) {
            repository.updateStatus(startDate, endDate);
        }

        return RepeatStatus.FINISHED;
    }
}
```

### How Tasklet Interacts with JdbcTemplate

| Step | Component | Action |
|------|-----------|--------|
| 1 | Tasklet | Gets job parameters from `ChunkContext` |
| 2 | Tasklet | Calls `repository.getTransactions()` |
| 3 | Repository | Creates `NamedParameterJdbcTemplate` from DataSource |
| 4 | JdbcTemplate | Executes SELECT query |
| 5 | Tasklet | Processes records, writes file |
| 6 | Tasklet | Calls `repository.updateStatus()` |
| 7 | Repository | Creates `NamedParameterJdbcTemplate`, executes UPDATE |

---

## Part 3: ItemReader Pattern (Chunk-Based)

ItemReader reads data one record at a time for chunk-based processing. Used for large dataset processing with memory efficiency.

### Source Reference

**File:** `com.example.batch.job.reader.ExceptionReader`

### Actual Usage

```java
public class ExceptionReader implements ItemReader<ExceptionRecord> {

    @Autowired
    BatchRepository repository;
    
    @Autowired
    EmailService email;
    
    private String inputDate;
    private Iterator<ExceptionRecord> transactionIterator;
    
    public ExceptionReader(String inputDate) {
        this.inputDate = inputDate;
    }
    
    @PostConstruct
    public void afterConstruct() throws BatchException {
        // Load all records once during initialization
        String startDate = CommonConstants.dateBatchInputFormat
                .parseDateTime(inputDate)
                .withTime(0, 0, 0, 0)
                .toString(CommonConstants.dateTimeBatchDbFormat);
        
        String endDate = CommonConstants.dateBatchInputFormat
                .parseDateTime(inputDate)
                .withTime(23, 59, 59, 59)
                .toString(CommonConstants.dateTimeBatchDbFormat);
        
        // Query via Repository (which uses JdbcTemplate internally)
        List<ExceptionRecord> exceptionRecs = repository.getExceptions(startDate, endDate);
        transactionIterator = exceptionRecs.iterator();
    }
    
    @Override
    public ExceptionRecord read() throws Exception {
        // Return one record at a time
        if (transactionIterator.hasNext()) {
            return transactionIterator.next();
        }
        // Return null signals end of data
        return null;
    }
}
```

### How ItemReader Interacts with JdbcTemplate

| Step | Component | Action |
|------|-----------|--------|
| 1 | ItemReader | `@PostConstruct` method called on bean creation |
| 2 | ItemReader | Calls `repository.getExceptions()` |
| 3 | Repository | Creates `NamedParameterJdbcTemplate` from DataSource |
| 4 | JdbcTemplate | Executes SELECT query, returns List |
| 5 | ItemReader | Stores result as Iterator |
| 6 | Spring Batch | Calls `read()` repeatedly |
| 7 | ItemReader | Returns one record per call from Iterator |
| 8 | ItemReader | Returns `null` when Iterator exhausted (signals end) |

---

## Part 4: Job Configuration

### Source Reference

**File:** `com.example.batch.job.config.ReportJobConfig`

### Tasklet Job Configuration

```java
@Configuration
public class ReportJobConfig {

    @Autowired
    private JobRepository jobRepository;

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Autowired
    private ReportTasklet reportTasklet;

    @Bean
    public Job reportJob() {
        return new JobBuilder("reportJob", jobRepository)
                .start(reportStep())
                .build();
    }

    @Bean
    public Step reportStep() {
        return new StepBuilder("reportStep", jobRepository)
                .tasklet(reportTasklet, transactionManager)
                .build();
    }
}
```

### Chunk-Based Job Configuration

```java
@Configuration
public class ExceptionJobConfig {

    @Autowired
    private JobRepository jobRepository;

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Bean
    @StepScope
    public ExceptionReader exceptionReader(
            @Value("#{jobParameters['inputDateStr']}") String inputDate) {
        return new ExceptionReader(inputDate);
    }

    @Bean
    public Job exceptionJob() {
        return new JobBuilder("exceptionJob", jobRepository)
                .start(exceptionStep())
                .build();
    }

    @Bean
    public Step exceptionStep() {
        return new StepBuilder("exceptionStep", jobRepository)
                .<ExceptionRecord, ExceptionRecord>chunk(100, transactionManager)
                .reader(exceptionReader(null))
                .processor(exceptionProcessor())
                .writer(exceptionWriter())
                .build();
    }
}
```

---

## Scenario 2 Summary

### Complete Linkage Chain

1. **Job Configuration** → Defines Job, Step, and component beans
2. **Spring Batch** → Executes Job, calls Tasklet or ItemReader
3. **Tasklet/ItemReader** → Injects Repository via `@Autowired`
4. **Repository** → Injects DataSource, creates JdbcTemplate
5. **JdbcTemplate** → Executes SQL via HikariCP connection pool
6. **Database** → MySQL receives SQL queries

### Quick Reference

| Pattern | Use Case | Database Access |
|---------|----------|-----------------|
| Tasklet | Single operation, file generation, bulk updates | `repository.query()` / `repository.update()` |
| ItemReader | Chunk processing, large datasets | `@PostConstruct` loads via `repository.query()` |

### Tasklet vs ItemReader

| Aspect | Tasklet | ItemReader |
|--------|---------|------------|
| Processing | All at once | One record at a time |
| Memory | Loads all records | Iterator-based |
| Use Case | File generation, reports | Large dataset transformation |
| Return | `RepeatStatus.FINISHED` | Record or `null` |

---

# Overall Summary

## Scenario Comparison

| Aspect | Scenario 1 (Without Batch) | Scenario 2 (With Batch) |
|--------|---------------------------|------------------------|
| Entry Point | Service/Controller | Job Configuration |
| Database Access | Repository → JdbcTemplate | Tasklet/Reader → Repository → JdbcTemplate |
| Transaction | `@Transactional` | Spring Batch Transaction Manager |
| Use Case | API endpoints, scheduled tasks | Batch processing, file generation |

## Common Components

Both scenarios share:
- **Properties Configuration** (HikariCP, dual DataSource)
- **DatasourceConfiguration** (bean creation)
- **Repository Layer** (JdbcTemplate usage patterns)

## JdbcTemplate Quick Reference

| Class | Parameter Style | Use Case |
|-------|-----------------|----------|
| JdbcTemplate | Positional `?` | Simple INSERT/UPDATE |
| NamedParameterJdbcTemplate | Named `:param` | Complex queries with multiple params |

## RowMapper Patterns

| Pattern | Use Case |
|---------|----------|
| Lambda `(rs, rowNum) -> {}` | Simple inline mapping |
| RowMapper Class | Reusable complex mapping with logic |
| BeanPropertyRowMapper | Auto-map by column alias to bean property |
| Anonymous RowMapper | Single value extraction |
