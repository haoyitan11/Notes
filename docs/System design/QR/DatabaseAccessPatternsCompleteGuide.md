# Java Database Access Patterns Guide

A comprehensive reference for database access patterns in Spring Boot applications, covering Hibernate/JPA, JdbcTemplate, and MyBatis approaches.

---

## Table of Contents

1. [Common Foundation](#common-foundation)
2. [Hibernate / Spring Data JPA](#hibernate--spring-data-jpa)
3. [JdbcTemplate](#jdbctemplate)
4. [MyBatis](#mybatis)
5. [Pattern Comparison](#pattern-comparison)
6. [Best Practices](#best-practices)

---

## Common Foundation

All three approaches share common database connectivity infrastructure in Spring Boot.

### DataSource Configuration

```properties
# Application Properties
spring.datasource.url=jdbc:mysql://localhost:3306/appdb
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Connection Pool (HikariCP - Spring Boot default)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.connection-timeout=20000
```

### Connection Flow

```
Application Request
       ↓
   DataSource (HikariCP Pool)
       ↓
   JDBC Connection
       ↓
   JDBC Driver (mysql-connector-java)
       ↓
   Database Server
```

### Transaction Management

All approaches use Spring's `@Transactional` annotation:

```java
@Service
public class SomeService {
    
    @Transactional
    public void performOperation() {
        // Operations within this method share the same transaction
    }
    
    @Transactional(readOnly = true)
    public Data readData() {
        // Read-only transaction optimization
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void independentOperation() {
        // Runs in a new transaction, suspending current if exists
    }
}
```

---

## Hibernate / Spring Data JPA

Object-Relational Mapping (ORM) approach that maps Java objects directly to database tables.

### Architecture Overview

```
Repository Interface (JpaRepository)
         ↓
Spring Data JPA (auto-generates implementation)
         ↓
EntityManager (JPA API)
         ↓
Hibernate SessionFactory (ORM Engine)
         ↓
DataSource → JDBC → Database
```

### Configuration

```properties
# JPA/Hibernate Properties
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
```

```java
@Configuration
@EnableJpaRepositories(basePackages = "com.example.repository")
@EntityScan(basePackages = "com.example.entity")
public class JpaConfig {
    // Spring Boot auto-configures most JPA beans
}
```

### Entity Definition

```java
@Entity
@Table(name = "tbl_key_info")
public class KeyInfo implements Serializable {
    
    private static final long serialVersionUID = 1L;
    
    @Id
    @Column(name = "inst_id", length = 20)
    private String instId;
    
    @Column(name = "key_type", length = 10, nullable = false)
    private String keyType;
    
    @Column(name = "key_value", length = 500)
    private String keyValue;
    
    @Column(name = "create_time")
    @Temporal(TemporalType.TIMESTAMP)
    private Date createTime;
    
    @Column(name = "update_time")
    @Temporal(TemporalType.TIMESTAMP)
    private Date updateTime;
    
    @Column(name = "status", length = 1)
    private String status;
    
    // Constructors
    public KeyInfo() {}
    
    public KeyInfo(String instId, String keyType) {
        this.instId = instId;
        this.keyType = keyType;
    }
    
    // Getters and Setters
    public String getInstId() { return instId; }
    public void setInstId(String instId) { this.instId = instId; }
    
    public String getKeyType() { return keyType; }
    public void setKeyType(String keyType) { this.keyType = keyType; }
    
    public String getKeyValue() { return keyValue; }
    public void setKeyValue(String keyValue) { this.keyValue = keyValue; }
    
    // ... other getters/setters
}
```

### Repository Layer

```java
@Repository
public interface KeyInfoRepository extends JpaRepository<KeyInfo, String> {
    
    // Derived query methods (Spring Data generates SQL)
    List<KeyInfo> findByKeyType(String keyType);
    
    Optional<KeyInfo> findByInstIdAndKeyType(String instId, String keyType);
    
    List<KeyInfo> findByStatusOrderByCreateTimeDesc(String status);
    
    // Custom JPQL query
    @Query("SELECT k FROM KeyInfo k WHERE k.keyType = :type AND k.status = 'A'")
    List<KeyInfo> findActiveByType(@Param("type") String keyType);
    
    // Native SQL query
    @Query(value = "SELECT * FROM tbl_key_info WHERE inst_id LIKE :prefix%", 
           nativeQuery = true)
    List<KeyInfo> findByInstIdPrefix(@Param("prefix") String prefix);
    
    // Modifying query
    @Modifying
    @Query("UPDATE KeyInfo k SET k.status = :status WHERE k.instId = :id")
    int updateStatus(@Param("id") String instId, @Param("status") String status);
}
```

### Service Layer

```java
public interface KeyInfoService {
    KeyInfo findByInstId(String instId);
    List<KeyInfo> findByKeyType(String keyType);
    KeyInfo save(KeyInfo keyInfo);
    void deleteByInstId(String instId);
}

@Service
@Transactional
public class KeyInfoServiceImpl implements KeyInfoService {
    
    private static final Logger logger = LoggerFactory.getLogger(KeyInfoServiceImpl.class);
    
    private final KeyInfoRepository keyInfoRepository;
    
    @Autowired
    public KeyInfoServiceImpl(KeyInfoRepository keyInfoRepository) {
        this.keyInfoRepository = keyInfoRepository;
    }
    
    @Override
    @Transactional(readOnly = true)
    public KeyInfo findByInstId(String instId) {
        logger.debug("Finding KeyInfo by instId: {}", instId);
        return keyInfoRepository.findById(instId)
            .orElseThrow(() -> new EntityNotFoundException("KeyInfo not found: " + instId));
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<KeyInfo> findByKeyType(String keyType) {
        return keyInfoRepository.findByKeyType(keyType);
    }
    
    @Override
    public KeyInfo save(KeyInfo keyInfo) {
        keyInfo.setUpdateTime(new Date());
        if (keyInfo.getCreateTime() == null) {
            keyInfo.setCreateTime(new Date());
        }
        logger.info("Saving KeyInfo: {}", keyInfo.getInstId());
        return keyInfoRepository.save(keyInfo);
    }
    
    @Override
    public void deleteByInstId(String instId) {
        logger.info("Deleting KeyInfo: {}", instId);
        keyInfoRepository.deleteById(instId);
    }
}
```

### JPA Annotations Reference

| Annotation | Purpose |
|------------|---------|
| `@Entity` | Marks class as JPA entity |
| `@Table` | Specifies table name and schema |
| `@Id` | Marks primary key field |
| `@GeneratedValue` | Auto-generation strategy for ID |
| `@Column` | Column mapping and constraints |
| `@Temporal` | Date/Time type specification |
| `@Transient` | Excludes field from persistence |
| `@OneToMany` | One-to-many relationship |
| `@ManyToOne` | Many-to-one relationship |
| `@JoinColumn` | Foreign key column specification |

### Entity Relationships

```java
// One-to-Many Example
@Entity
@Table(name = "institution")
public class Institution {
    @Id
    private String instId;
    
    @OneToMany(mappedBy = "institution", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<KeyInfo> keys = new ArrayList<>();
}

@Entity
@Table(name = "tbl_key_info")
public class KeyInfo {
    @Id
    private String id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "inst_id", referencedColumnName = "inst_id")
    private Institution institution;
}
```

---

## JdbcTemplate

Low-level approach providing direct SQL control with Spring's exception handling.

### Architecture Overview

```
Service Layer
     ↓
JdbcTemplate / NamedParameterJdbcTemplate
     ↓
RowMapper (result → object conversion)
     ↓
DataSource → JDBC → Database
```

### Configuration

```java
@Configuration
public class JdbcConfig {
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public NamedParameterJdbcTemplate namedParameterJdbcTemplate(DataSource dataSource) {
        return new NamedParameterJdbcTemplate(dataSource);
    }
}
```

### Model Class (Plain POJO)

```java
public class Transaction {
    private String txnId;
    private String merchantId;
    private BigDecimal amount;
    private String currency;
    private String status;
    private LocalDateTime createTime;
    
    // Constructors
    public Transaction() {}
    
    public Transaction(String txnId, String merchantId, BigDecimal amount) {
        this.txnId = txnId;
        this.merchantId = merchantId;
        this.amount = amount;
    }
    
    // Getters and Setters
    public String getTxnId() { return txnId; }
    public void setTxnId(String txnId) { this.txnId = txnId; }
    
    public String getMerchantId() { return merchantId; }
    public void setMerchantId(String merchantId) { this.merchantId = merchantId; }
    
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    
    // ... other getters/setters
}
```

### RowMapper Implementation

```java
@Component
public class TransactionRowMapper implements RowMapper<Transaction> {
    
    @Override
    public Transaction mapRow(ResultSet rs, int rowNum) throws SQLException {
        Transaction txn = new Transaction();
        txn.setTxnId(rs.getString("txn_id"));
        txn.setMerchantId(rs.getString("merchant_id"));
        txn.setAmount(rs.getBigDecimal("amount"));
        txn.setCurrency(rs.getString("currency"));
        txn.setStatus(rs.getString("status"));
        
        Timestamp createTs = rs.getTimestamp("create_time");
        if (createTs != null) {
            txn.setCreateTime(createTs.toLocalDateTime());
        }
        
        return txn;
    }
}

// Lambda-based RowMapper (inline)
RowMapper<Transaction> rowMapper = (rs, rowNum) -> {
    Transaction txn = new Transaction();
    txn.setTxnId(rs.getString("txn_id"));
    txn.setAmount(rs.getBigDecimal("amount"));
    return txn;
};
```

### Repository/DAO Layer

```java
@Repository
public class TransactionDao {
    
    private static final Logger logger = LoggerFactory.getLogger(TransactionDao.class);
    
    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedTemplate;
    private final TransactionRowMapper rowMapper;
    
    @Autowired
    public TransactionDao(JdbcTemplate jdbcTemplate, 
                          NamedParameterJdbcTemplate namedTemplate,
                          TransactionRowMapper rowMapper) {
        this.jdbcTemplate = jdbcTemplate;
        this.namedTemplate = namedTemplate;
        this.rowMapper = rowMapper;
    }
    
    // Query for single object
    public Transaction findById(String txnId) {
        String sql = "SELECT * FROM transactions WHERE txn_id = ?";
        try {
            return jdbcTemplate.queryForObject(sql, rowMapper, txnId);
        } catch (EmptyResultDataAccessException e) {
            logger.debug("Transaction not found: {}", txnId);
            return null;
        }
    }
    
    // Query for list
    public List<Transaction> findByMerchantId(String merchantId) {
        String sql = "SELECT * FROM transactions WHERE merchant_id = ? ORDER BY create_time DESC";
        return jdbcTemplate.query(sql, rowMapper, merchantId);
    }
    
    // Using NamedParameterJdbcTemplate
    public List<Transaction> findByStatusAndDateRange(String status, 
                                                       LocalDateTime startDate, 
                                                       LocalDateTime endDate) {
        String sql = """
            SELECT * FROM transactions 
            WHERE status = :status 
            AND create_time BETWEEN :startDate AND :endDate
            ORDER BY create_time
            """;
        
        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("status", status)
            .addValue("startDate", startDate)
            .addValue("endDate", endDate);
        
        return namedTemplate.query(sql, params, rowMapper);
    }
    
    // Insert
    public int insert(Transaction txn) {
        String sql = """
            INSERT INTO transactions (txn_id, merchant_id, amount, currency, status, create_time)
            VALUES (?, ?, ?, ?, ?, ?)
            """;
        return jdbcTemplate.update(sql, 
            txn.getTxnId(),
            txn.getMerchantId(),
            txn.getAmount(),
            txn.getCurrency(),
            txn.getStatus(),
            txn.getCreateTime());
    }
    
    // Update
    public int updateStatus(String txnId, String status) {
        String sql = "UPDATE transactions SET status = ?, update_time = ? WHERE txn_id = ?";
        return jdbcTemplate.update(sql, status, LocalDateTime.now(), txnId);
    }
    
    // Batch insert
    public int[] batchInsert(List<Transaction> transactions) {
        String sql = """
            INSERT INTO transactions (txn_id, merchant_id, amount, currency, status)
            VALUES (?, ?, ?, ?, ?)
            """;
        
        return jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Transaction txn = transactions.get(i);
                ps.setString(1, txn.getTxnId());
                ps.setString(2, txn.getMerchantId());
                ps.setBigDecimal(3, txn.getAmount());
                ps.setString(4, txn.getCurrency());
                ps.setString(5, txn.getStatus());
            }
            
            @Override
            public int getBatchSize() {
                return transactions.size();
            }
        });
    }
    
    // Query for scalar value
    public int countByStatus(String status) {
        String sql = "SELECT COUNT(*) FROM transactions WHERE status = ?";
        Integer count = jdbcTemplate.queryForObject(sql, Integer.class, status);
        return count != null ? count : 0;
    }
    
    // Query for map
    public Map<String, Object> getTransactionSummary(String merchantId) {
        String sql = """
            SELECT COUNT(*) as total_count, SUM(amount) as total_amount 
            FROM transactions WHERE merchant_id = ?
            """;
        return jdbcTemplate.queryForMap(sql, merchantId);
    }
}
```

### Service Layer with JdbcTemplate

```java
@Service
@Transactional
public class TransactionService {
    
    private final TransactionDao transactionDao;
    
    @Autowired
    public TransactionService(TransactionDao transactionDao) {
        this.transactionDao = transactionDao;
    }
    
    @Transactional(readOnly = true)
    public Transaction getTransaction(String txnId) {
        return transactionDao.findById(txnId);
    }
    
    public Transaction createTransaction(Transaction txn) {
        txn.setCreateTime(LocalDateTime.now());
        txn.setStatus("PENDING");
        transactionDao.insert(txn);
        return txn;
    }
    
    public void processTransaction(String txnId) {
        Transaction txn = transactionDao.findById(txnId);
        if (txn == null) {
            throw new IllegalArgumentException("Transaction not found: " + txnId);
        }
        // Business logic...
        transactionDao.updateStatus(txnId, "COMPLETED");
    }
}
```

### Common JdbcTemplate Methods

| Method | Purpose |
|--------|---------|
| `query(sql, RowMapper, args)` | Query returning list of objects |
| `queryForObject(sql, RowMapper, args)` | Query returning single object |
| `queryForObject(sql, Class, args)` | Query returning scalar value |
| `queryForList(sql, args)` | Query returning list of maps |
| `queryForMap(sql, args)` | Query returning single map |
| `update(sql, args)` | INSERT, UPDATE, DELETE operations |
| `batchUpdate(sql, BatchPreparedStatementSetter)` | Batch operations |
| `execute(sql)` | DDL statements |

---

## MyBatis

SQL mapping framework using XML or annotations for query definition.

### Architecture Overview

```
Service Layer
     ↓
Mapper Interface (annotated or XML-bound)
     ↓
SqlSession (MyBatis core)
     ↓
SqlSessionFactory
     ↓
DataSource → JDBC → Database
```

### Configuration

```properties
# MyBatis Properties
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.type-aliases-package=com.example.model
mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.cache-enabled=true
```

```java
@Configuration
@MapperScan("com.example.mapper")
public class MyBatisConfig {
    // Additional MyBatis configuration if needed
}
```

### Model Class

```java
public class Payment {
    private String paymentId;
    private String orderId;
    private BigDecimal amount;
    private String currency;
    private String paymentMethod;
    private String status;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
    
    // Constructors, Getters, Setters
    public Payment() {}
    
    public String getPaymentId() { return paymentId; }
    public void setPaymentId(String paymentId) { this.paymentId = paymentId; }
    
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    
    // ... other getters/setters
}
```

### Mapper Interface (Annotation-based)

```java
@Mapper
public interface PaymentMapper {
    
    @Select("SELECT * FROM payments WHERE payment_id = #{paymentId}")
    Payment findById(@Param("paymentId") String paymentId);
    
    @Select("SELECT * FROM payments WHERE order_id = #{orderId}")
    List<Payment> findByOrderId(@Param("orderId") String orderId);
    
    @Insert("""
        INSERT INTO payments (payment_id, order_id, amount, currency, payment_method, status, create_time)
        VALUES (#{paymentId}, #{orderId}, #{amount}, #{currency}, #{paymentMethod}, #{status}, #{createTime})
        """)
    int insert(Payment payment);
    
    @Update("UPDATE payments SET status = #{status}, update_time = #{updateTime} WHERE payment_id = #{paymentId}")
    int updateStatus(@Param("paymentId") String paymentId, 
                     @Param("status") String status, 
                     @Param("updateTime") LocalDateTime updateTime);
    
    @Delete("DELETE FROM payments WHERE payment_id = #{paymentId}")
    int delete(@Param("paymentId") String paymentId);
    
    // Dynamic SQL with Provider
    @SelectProvider(type = PaymentSqlProvider.class, method = "findByConditions")
    List<Payment> findByConditions(@Param("status") String status, 
                                   @Param("paymentMethod") String paymentMethod);
}
```

### SQL Provider for Dynamic Queries

```java
public class PaymentSqlProvider {
    
    public String findByConditions(@Param("status") String status, 
                                   @Param("paymentMethod") String paymentMethod) {
        return new SQL() {{
            SELECT("*");
            FROM("payments");
            if (status != null) {
                WHERE("status = #{status}");
            }
            if (paymentMethod != null) {
                WHERE("payment_method = #{paymentMethod}");
            }
            ORDER_BY("create_time DESC");
        }}.toString();
    }
}
```

### Mapper XML (Alternative to Annotations)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.mapper.PaymentMapper">
    
    <!-- Result Map -->
    <resultMap id="paymentResultMap" type="com.example.model.Payment">
        <id property="paymentId" column="payment_id"/>
        <result property="orderId" column="order_id"/>
        <result property="amount" column="amount"/>
        <result property="currency" column="currency"/>
        <result property="paymentMethod" column="payment_method"/>
        <result property="status" column="status"/>
        <result property="createTime" column="create_time"/>
        <result property="updateTime" column="update_time"/>
    </resultMap>
    
    <!-- SQL Fragment for reuse -->
    <sql id="paymentColumns">
        payment_id, order_id, amount, currency, payment_method, status, create_time, update_time
    </sql>
    
    <!-- Select by ID -->
    <select id="findById" resultMap="paymentResultMap">
        SELECT <include refid="paymentColumns"/>
        FROM payments
        WHERE payment_id = #{paymentId}
    </select>
    
    <!-- Select with Dynamic Conditions -->
    <select id="findByConditions" resultMap="paymentResultMap">
        SELECT <include refid="paymentColumns"/>
        FROM payments
        <where>
            <if test="status != null and status != ''">
                AND status = #{status}
            </if>
            <if test="paymentMethod != null and paymentMethod != ''">
                AND payment_method = #{paymentMethod}
            </if>
            <if test="minAmount != null">
                AND amount >= #{minAmount}
            </if>
            <if test="maxAmount != null">
                AND amount &lt;= #{maxAmount}
            </if>
        </where>
        ORDER BY create_time DESC
    </select>
    
    <!-- Insert -->
    <insert id="insert" parameterType="com.example.model.Payment">
        INSERT INTO payments (<include refid="paymentColumns"/>)
        VALUES (#{paymentId}, #{orderId}, #{amount}, #{currency}, 
                #{paymentMethod}, #{status}, #{createTime}, #{updateTime})
    </insert>
    
    <!-- Update -->
    <update id="update" parameterType="com.example.model.Payment">
        UPDATE payments
        <set>
            <if test="orderId != null">order_id = #{orderId},</if>
            <if test="amount != null">amount = #{amount},</if>
            <if test="status != null">status = #{status},</if>
            update_time = #{updateTime}
        </set>
        WHERE payment_id = #{paymentId}
    </update>
    
    <!-- Batch Insert -->
    <insert id="batchInsert" parameterType="list">
        INSERT INTO payments (payment_id, order_id, amount, currency, status, create_time)
        VALUES
        <foreach collection="list" item="payment" separator=",">
            (#{payment.paymentId}, #{payment.orderId}, #{payment.amount}, 
             #{payment.currency}, #{payment.status}, #{payment.createTime})
        </foreach>
    </insert>
    
    <!-- Select with IN clause -->
    <select id="findByIds" resultMap="paymentResultMap">
        SELECT <include refid="paymentColumns"/>
        FROM payments
        WHERE payment_id IN
        <foreach collection="ids" item="id" open="(" separator="," close=")">
            #{id}
        </foreach>
    </select>
    
</mapper>
```

### Service Layer with MyBatis

```java
@Service
@Transactional
public class PaymentService {
    
    private static final Logger logger = LoggerFactory.getLogger(PaymentService.class);
    
    private final PaymentMapper paymentMapper;
    
    @Autowired
    public PaymentService(PaymentMapper paymentMapper) {
        this.paymentMapper = paymentMapper;
    }
    
    @Transactional(readOnly = true)
    public Payment getPayment(String paymentId) {
        return paymentMapper.findById(paymentId);
    }
    
    public Payment createPayment(Payment payment) {
        payment.setPaymentId(generatePaymentId());
        payment.setStatus("PENDING");
        payment.setCreateTime(LocalDateTime.now());
        
        int rows = paymentMapper.insert(payment);
        if (rows != 1) {
            throw new RuntimeException("Failed to insert payment");
        }
        
        logger.info("Created payment: {}", payment.getPaymentId());
        return payment;
    }
    
    public void updatePaymentStatus(String paymentId, String status) {
        int rows = paymentMapper.updateStatus(paymentId, status, LocalDateTime.now());
        if (rows != 1) {
            throw new IllegalArgumentException("Payment not found: " + paymentId);
        }
    }
    
    @Transactional(readOnly = true)
    public List<Payment> searchPayments(String status, String paymentMethod) {
        return paymentMapper.findByConditions(status, paymentMethod);
    }
    
    private String generatePaymentId() {
        return "PAY" + System.currentTimeMillis();
    }
}
```

### MyBatis Dynamic SQL Tags

| Tag | Purpose |
|-----|---------|
| `<if>` | Conditional inclusion |
| `<choose>/<when>/<otherwise>` | Switch-case logic |
| `<where>` | Smart WHERE clause (handles AND/OR) |
| `<set>` | Smart SET clause for updates |
| `<foreach>` | Iteration for IN clauses, batch ops |
| `<sql>/<include>` | Reusable SQL fragments |
| `<trim>` | Custom prefix/suffix handling |

---

## Pattern Comparison

### When to Use Each Approach

| Criteria | JPA/Hibernate | JdbcTemplate | MyBatis |
|----------|---------------|--------------|---------|
| **Learning Curve** | Steep | Low | Moderate |
| **SQL Control** | Limited | Full | Full |
| **Productivity** | High (CRUD) | Low | Moderate |
| **Complex Queries** | Challenging | Easy | Easy |
| **Performance Tuning** | Difficult | Easy | Easy |
| **Caching** | Built-in L1/L2 | Manual | Built-in |
| **Code Verbosity** | Low | High | Moderate |
| **Type Safety** | High | Low | Moderate |

### Recommended Use Cases

**Use JPA/Hibernate when:**
- Building CRUD-heavy applications
- Domain model closely matches database schema
- Team is familiar with ORM concepts
- You want rapid development with standard operations
- Object relationships are important

**Use JdbcTemplate when:**
- Maximum SQL control is required
- Working with legacy databases
- Complex reporting queries
- Performance-critical operations
- Simple data access without relationships

**Use MyBatis when:**
- SQL expertise should be leveraged
- Complex dynamic queries are common
- Database schema doesn't match object model well
- Migration from raw JDBC is needed
- Balance between control and convenience is desired

### Code Comparison

```java
// JPA - Finding by status
List<KeyInfo> findByStatus(String status) {
    return keyInfoRepository.findByStatus(status);
}

// JdbcTemplate - Finding by status
List<Transaction> findByStatus(String status) {
    String sql = "SELECT * FROM transactions WHERE status = ?";
    return jdbcTemplate.query(sql, rowMapper, status);
}

// MyBatis - Finding by status
// In XML: <select id="findByStatus">SELECT * FROM payments WHERE status = #{status}</select>
List<Payment> findByStatus(String status) {
    return paymentMapper.findByStatus(status);
}
```

---

## Best Practices

### General Best Practices

1. **Always use parameterized queries** - Never concatenate user input into SQL strings
2. **Use transactions appropriately** - Mark read-only operations with `@Transactional(readOnly = true)`
3. **Handle exceptions properly** - Catch and translate data access exceptions
4. **Use connection pooling** - HikariCP (Spring Boot default) is recommended
5. **Log SQL in development** - Disable in production for performance

### JPA-Specific

1. **Use `@Transactional(readOnly = true)`** for queries to optimize performance
2. **Prefer `FetchType.LAZY`** for relationships to avoid N+1 problems
3. **Use `@EntityGraph`** or `JOIN FETCH` when eager loading is needed
4. **Avoid returning entities from REST controllers** - Use DTOs
5. **Use `@Modifying` with `@Query`** for update/delete operations

### JdbcTemplate-Specific

1. **Use `NamedParameterJdbcTemplate`** for complex queries with many parameters
2. **Create reusable `RowMapper` classes** for consistent mapping
3. **Use `batchUpdate`** for bulk operations
4. **Handle `EmptyResultDataAccessException`** for single-result queries
5. **Use `MapSqlParameterSource`** for named parameters

### MyBatis-Specific

1. **Enable `map-underscore-to-camel-case`** for automatic column mapping
2. **Use `<sql>` fragments** to avoid duplication
3. **Prefer XML** for complex dynamic queries
4. **Use `@Param`** annotation for multiple parameters
5. **Leverage result maps** for complex object mapping

### Error Handling

```java
@Service
public class DataAccessService {
    
    private static final Logger logger = LoggerFactory.getLogger(DataAccessService.class);
    
    public void safeOperation() {
        try {
            // Database operation
        } catch (DataAccessException e) {
            logger.error("Database operation failed", e);
            throw new ServiceException("Operation failed", e);
        }
    }
}
```

### Logging Configuration

```xml
<!-- logback.xml -->
<logger name="org.springframework.jdbc" level="DEBUG"/>
<logger name="org.hibernate.SQL" level="DEBUG"/>
<logger name="org.mybatis" level="DEBUG"/>
```

---

## Quick Reference

### Annotation Summary

| Framework | Key Annotations |
|-----------|-----------------|
| JPA | `@Entity`, `@Table`, `@Id`, `@Column`, `@Repository`, `@Query` |
| JdbcTemplate | `@Repository`, `@Autowired` (no special annotations) |
| MyBatis | `@Mapper`, `@Select`, `@Insert`, `@Update`, `@Delete`, `@Param` |

### Configuration Properties

```properties
# Common DataSource
spring.datasource.url=jdbc:mysql://localhost:3306/db
spring.datasource.username=user
spring.datasource.password=pass

# JPA
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false

# MyBatis
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.configuration.map-underscore-to-camel-case=true
```

---

*This guide provides reference patterns for database access in Spring Boot applications. Choose the approach that best fits your project requirements and team expertise.*
