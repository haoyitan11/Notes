
# MyBatis Database Integration Guide

## Overview

This document provides a complete guide for the MyBatis database integration in the application. It covers three layers:

1. **Core Infrastructure** - Foundation connectivity to MySQL Database via Spring Boot Auto-Configuration

2. **Mapper Layer** - MyBatis Mapper interfaces and XML mappings for SQL operations

3. **DAO Layer** - Data Access Objects providing business-level database operations with transaction management

---

# Part 1: Core Infrastructure

![MyBatis Core Infrastructure](https://github.com/user-attachments/assets/mybatis-core-infrastructure)

## Purpose

Provides the foundational connectivity layer between the application and MySQL Database using MyBatis-Spring Boot integration.

---

## 1.1 Maven Dependencies

### Purpose

Defines the required libraries for MyBatis-Spring Boot integration.

### Configuration (pom.xml)

```xml
<dependencies>
    <!-- MyBatis Spring Boot Starter -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>${spring.mybatis.version}</version>
    </dependency>

    <!-- MySQL JDBC Driver -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
    </dependency>
</dependencies>

<properties>
    <spring.mybatis.version>2.3.2</spring.mybatis.version>
</properties>
```

### Responsibilities

- Provides MyBatis core framework.
- Provides MyBatis-Spring integration.
- Provides Spring Boot auto-configuration for MyBatis.
- Provides MySQL JDBC driver for database connectivity.

---

## 1.2 DataSource Configuration

### Purpose

Configures the database connection pool and JDBC settings.

### Configuration (application.properties)

```properties
# Database Connection
spring.datasource.url=jdbc:mysql://172.19.66.201:3306/adaptor?characterEncoding=utf8&useSSL=true
spring.datasource.username=adaptor
spring.datasource.password=adaptor
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver

# Connection Pool Configuration (Tomcat JDBC Pool)
spring.datasource.tomcat.max-wait=10000
spring.datasource.tomcat.max-active=100
spring.datasource.tomcat.test-on-borrow=true
spring.datasource.tomcat.initial-size=5
```

### Responsibilities

- Configures JDBC URL with character encoding and SSL.
- Provides database credentials.
- Configures connection pool settings for performance.
- Auto-configures DataSource bean via Spring Boot.

---

## 1.3 MyBatis Configuration

### Purpose

Configures MyBatis-specific settings for mapper scanning and type aliases.

### Configuration (application.properties)

```properties
# MyBatis Configuration
mybatis.type-aliases-package=com.upi.adaptor.model
mybatis.mapper-locations=classpath:mapper/*.xml
```

### Configuration (HttpServiceMain.java)

```java
@EnableAutoConfiguration
@SpringBootApplication
@MapperScan("com.upi.adaptor.mapper")
@ImportResource("classpath*:/spring-context.xml")
public class HttpServiceMain {
    public static void main(String[] args) {
        ConfigurableApplicationContext ca = SpringApplication.run(HttpServiceMain.class, args);
        // Application startup logic
    }
}
```

### Responsibilities

- **@MapperScan**: Scans the specified package for Mapper interfaces and registers them as Spring beans.
- **mybatis.type-aliases-package**: Enables short class names in XML mappings instead of fully qualified names.
- **mybatis.mapper-locations**: Specifies the location of Mapper XML files.

---

## 1.4 Auto-Configuration Components

### Purpose

Spring Boot automatically configures the following MyBatis components.

### Components Created by Auto-Configuration

| Component | Class | Purpose |
|-----------|-------|---------|
| DataSource | HikariDataSource / TomcatDataSource | Database connection pool |
| SqlSessionFactory | SqlSessionFactory | Creates SqlSession instances |
| SqlSessionTemplate | SqlSessionTemplate | Thread-safe SqlSession wrapper |
| DataSourceTransactionManager | DataSourceTransactionManager | Transaction management |

### Auto-Configuration Flow

```
Application Start
       │
       ▼
┌──────────────────────────────────────┐
│  DataSourceAutoConfiguration         │
│  - Creates DataSource from properties │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  MybatisAutoConfiguration            │
│  - Creates SqlSessionFactory         │
│  - Creates SqlSessionTemplate        │
│  - Registers Mapper interfaces       │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  DataSourceTransactionManager        │
│  - Manages database transactions     │
└──────────────────────────────────────┘
```

---

# Part 2: Mapper Layer

![MyBatis Mapper Layer](https://github.com/user-attachments/assets/mybatis-mapper-layer)

## Purpose

Provides the SQL mapping layer that defines database operations through interfaces and XML configurations.

---

## 2.1 Mapper Interface

### Purpose

Defines method signatures for database operations. MyBatis generates implementation at runtime.

### Implementation Pattern

```java
package com.upi.adaptor.mapper;

import java.util.List;
import org.apache.ibatis.annotations.Mapper;
import com.upi.adaptor.model.TransLog;

@Mapper
public interface TransLogMapper {
    
    // Select all records
    public List<TransLog> findAll();
    
    // Select by primary key
    public TransLog findByID(String msgID);
    
    // Select by alternate key
    public TransLog findByQrcVouNo(String qrcVoucherNo);
    
    // Insert new record
    public void insert(TransLog transLog);
    
    // Delete by primary key
    public void delete(String msgID);
    
    // Update existing record
    public void update(TransLog transLog);
    
    // Select with row lock (FOR UPDATE)
    public TransLog findByIDForUpdate(String msgID);
}
```

### Responsibilities

- Defines method signatures for SQL operations.
- Annotated with **@Mapper** for Spring detection.
- Methods are mapped to SQL statements in XML file.

---

## 2.2 Mapper XML Configuration

### Purpose

Defines SQL statements and result mappings for each Mapper interface method.

### Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.upi.adaptor.mapper.TransLogMapper">
    
    <!-- Result Map: Column to Property Mapping -->
    <resultMap id="BaseResultMap" type="com.upi.adaptor.model.TransLog">
        <id column="msg_id" property="msgId" jdbcType="VARCHAR" />
        <result column="msg_type" property="msgType" jdbcType="VARCHAR" />
        <result column="acq_ins_code" property="acqInsCode" jdbcType="VARCHAR" />
        <!-- Additional column mappings -->
    </resultMap>

    <!-- Reusable SQL Fragment -->
    <sql id="Base_Column_List">
        msg_id, msg_type, acq_ins_code, trans_datetime, process_flag, 
        rev_flag, trans_curr, trans_amt, orig_amt, cost_amt
    </sql>

    <!-- SELECT: Find All -->
    <select id="findAll" parameterType="String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM tbl_trans_log 
    </select>
    
    <!-- SELECT: Find by ID -->
    <select id="findByID" parameterType="String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" /> 
        FROM tbl_trans_log 
        WHERE msg_id = #{0}
    </select>
    
    <!-- SELECT: Find by ID with Row Lock -->
    <select id="findByIDForUpdate" parameterType="String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" /> 
        FROM tbl_trans_log 
        WHERE msg_id = #{0} FOR UPDATE
    </select>
    
    <!-- INSERT -->
    <insert id="insert" parameterType="com.upi.adaptor.model.TransLog">
        INSERT INTO tbl_trans_log  
        VALUES(#{msgId}, #{msgType}, #{acqInsCode}, #{transDatetime}, 
               #{processFlag}, #{revFlag}, #{transCurr}, #{transAmt})
    </insert>
    
    <!-- UPDATE -->
    <update id="update" parameterType="com.upi.adaptor.model.TransLog">
        UPDATE tbl_trans_log SET 
            msg_type = #{msgType},         
            acq_ins_code = #{acqInsCode},      
            trans_datetime = #{transDatetime},   
            process_flag = #{processFlag}
        WHERE msg_id = #{msgId}
    </update>
    
    <!-- DELETE -->
    <delete id="delete" parameterType="String">
        DELETE FROM tbl_trans_log 
        WHERE msg_id = #{0}
    </delete>
    
</mapper>
```

### Responsibilities

- **namespace**: Links XML to Mapper interface (must match fully qualified interface name).
- **resultMap**: Maps database columns to Java object properties.
- **sql fragments**: Reusable column lists to avoid duplication.
- **CRUD operations**: SELECT, INSERT, UPDATE, DELETE statements.

---

## 2.3 Result Map Configuration

### Purpose

Handles column-to-property mapping when column names differ from Java property names.

### Configuration Example

```xml
<resultMap id="BaseResultMap" type="com.upi.adaptor.model.KeyInfo">
    <!-- Primary Key -->
    <id column="ins_code" property="insCode" jdbcType="VARCHAR" />
    
    <!-- Regular Columns -->
    <result column="sign_certid" property="signCertid" jdbcType="VARCHAR" />
    <result column="sign_publickey" property="signPublickey" jdbcType="VARCHAR" />
    <result column="sign_privatekey" property="signPrivatekey" jdbcType="VARCHAR" />
    <result column="enc_certid" property="encCertid" jdbcType="VARCHAR" />
    <result column="enc_publickey" property="encPublickey" jdbcType="VARCHAR" />
    <result column="enc_privatekey" property="encPrivatekey" jdbcType="VARCHAR" />
    <result column="uais_sign_certid" property="uaisSignCertid" jdbcType="VARCHAR" />
    <result column="uais_sign_publickey" property="uaisSignPublickey" jdbcType="VARCHAR" />
    <result column="uais_enc_certid" property="uaisEncCertid" jdbcType="VARCHAR" />
    <result column="uais_enc_publickey" property="uaisEncPublickey" jdbcType="VARCHAR" />
    <result column="signcert_update_time" property="signcertUpdateTime" jdbcType="VARCHAR" />
    <result column="enccert_update_time" property="enccertUpdateTime" jdbcType="VARCHAR" />
</resultMap>
```

### Mapping Elements

| Element | Purpose |
|---------|---------|
| `<id>` | Maps primary key column |
| `<result>` | Maps regular column |
| `column` | Database column name |
| `property` | Java property name |
| `jdbcType` | JDBC type for null handling |

---

## 2.4 SQL Fragments

### Purpose

Defines reusable SQL snippets to reduce duplication.

### Configuration Example

```xml
<!-- Define reusable column list -->
<sql id="Base_Column_List">
    curr_code, chn_desc, chn_abb_desc, eng_desc, eng_abb_desc, curr_exponent, curr_sign
</sql>

<!-- Use in SELECT statements -->
<select id="findAll" parameterType="String" resultMap="BaseResultMap">
    SELECT <include refid="Base_Column_List" />
    FROM tbl_curr_code 
</select>

<select id="findByID" parameterType="String" resultMap="BaseResultMap">
    SELECT <include refid="Base_Column_List" /> 
    FROM tbl_curr_code 
    WHERE curr_code = #{0}
</select>
```

### Responsibilities

- Defines column lists once.
- Includes via `<include refid="..."/>`.
- Maintains consistency across queries.

---

## 2.5 Parameter Binding

### Purpose

Binds method parameters to SQL placeholders.

### Parameter Binding Patterns

**Single Parameter (Positional):**
```xml
<select id="findByID" parameterType="String" resultMap="BaseResultMap">
    SELECT * FROM tbl_key_info 
    WHERE ins_code = #{0}
</select>
```

**Object Parameter (Property Names):**
```xml
<insert id="insert" parameterType="com.upi.adaptor.model.KeyInfo">
    INSERT INTO tbl_key_info 
    VALUES(#{insCode}, #{signCertid}, #{signPublickey}, #{signPrivatekey},
           #{encCertid}, #{encPublickey}, #{encPrivatekey})
</insert>
```

**Conditional Parameter:**
```xml
<select id="findAll" parameterType="String" resultMap="BaseResultMap">
    SELECT * FROM tbl_timeout_info 
    WHERE Rec_timeout_time &lt; #{0}
</select>
```

### Parameter Syntax

| Syntax | Usage |
|--------|-------|
| `#{0}` | First positional parameter |
| `#{propertyName}` | Object property |
| `#{param1}` | Named parameter |

---

# Part 3: DAO Layer

![MyBatis DAO Layer](https://github.com/user-attachments/assets/mybatis-dao-layer)

## Purpose

Provides business-level data access with transaction management and business logic encapsulation.

---

## 3.1 DAO Interface

### Purpose

Defines business-level data access methods.

### Implementation Pattern

```java
package com.upi.adaptor.dao;

import java.util.List;
import com.upi.adaptor.model.TransLog;

public interface TransLogDao {
    
    List<TransLog> getAllTransLog();
    
    TransLog getTransLogByID(String msgID);
    
    TransLog getTransLogByQrcVouNo(String qrcVoucherNo);
    
    void addTransLog(TransLog transLog);
    
    void delTransLog(String msgID);
    
    void updateTransLog(TransLog transLog);
    
    TransLog getTransLogByIDForUpdate(String msgID);
}
```

### Responsibilities

- Defines business-level method signatures.
- Provides meaningful method names for business operations.
- Abstracts MyBatis Mapper implementation details.

---

## 3.2 DAO Implementation

### Purpose

Implements DAO interface with transaction management and business logic.

### Implementation Pattern

```java
package com.upi.adaptor.dao.impl;

import java.util.List;
import javax.annotation.Resource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.upi.adaptor.dao.TransLogDao;
import com.upi.adaptor.mapper.TransLogMapper;
import com.upi.adaptor.model.TransLog;

@Service
@Transactional
public class TransLogDaoImpl implements TransLogDao {
    
    private static final Logger logger = LoggerFactory.getLogger(TransLogDaoImpl.class);

    @Resource
    private TransLogMapper transLogMapper;
    
    @Override
    public List<TransLog> getAllTransLog() {
        return transLogMapper.findAll();
    }

    @Override
    public TransLog getTransLogByID(String msgID) {
        return transLogMapper.findByID(msgID);
    }

    @Override
    public TransLog getTransLogByQrcVouNo(String qrcVoucherNo) {
        return transLogMapper.findByQrcVouNo(qrcVoucherNo);
    }

    @Override
    public void addTransLog(TransLog transLog) {
        transLogMapper.insert(transLog);
    }

    @Override
    public void delTransLog(String msgID) {
        transLogMapper.delete(msgID);
    }

    @Override
    public void updateTransLog(TransLog transLog) {
        transLogMapper.update(transLog);
    }
    
    @Override
    public TransLog getTransLogByIDForUpdate(String msgID) {
        return transLogMapper.findByIDForUpdate(msgID);
    }
}
```

### Responsibilities

- **@Service**: Registers as Spring bean.
- **@Transactional**: Enables transaction management.
- **@Resource**: Injects MyBatis Mapper.
- Delegates to Mapper methods.
- Adds logging and error handling.

---

## 3.3 Transaction Management

### Purpose

Manages database transactions with Spring's declarative transaction support.

### Configuration

```java
@Service
@Transactional
public class TransLogDaoImpl implements TransLogDao {
    // All methods run within a transaction
}
```

### Transaction Flow

```
Service Method Called
       │
       ▼
┌──────────────────────────────────────┐
│  @Transactional Interceptor          │
│  - Begin Transaction                 │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  DAO Method Execution                │
│  - Execute SQL via Mapper            │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Transaction Completion              │
│  - Commit (success)                  │
│  - Rollback (exception)              │
└──────────────────────────────────────┘
```

### Pessimistic Locking Example

```java
// Select with row-level lock
public TransLog timeoutUpdate(String msgID) {
    // FOR UPDATE locks the row
    TransLog transLog = getTransLogByIDForUpdate(msgID);
    
    if (transLog == null) {
        logger.error("can't find trans through msgID:" + msgID);
        return null;
    }
    
    if (!transLog.getProcessFlag().equals("01")) {
        logger.info("late trx status notification");
        return null;
    }
    
    // Update within same transaction
    transLog.setProcessFlag("07");   
    transLog.setUaisRespCode("98");
    updateTransLog(transLog);

    return transLog;
}
```

---

## 3.4 Business Logic Integration

### Purpose

Encapsulates complex business logic with data access operations.

### Implementation Example

```java
public TransLog notifyUpdate(MsgInfo msgInfo, HashMap<String, Object> trxInfo) 
        throws AdaptorException {
    
    TransLog transLog = new TransLog();
    String msgID = msgInfo.getMsgID();
    
    // 1. Lock and fetch record
    transLog = getTransLogByIDForUpdate("Z" + msgID.substring(1));
    
    if (transLog == null) {
        logger.error("can't find mpqrc verify trans through msgID:" + msgID);
        return null;
    }
    
    // 2. Validate current state
    if (!transLog.getProcessFlag().equals("01")) {
        logger.info("late trx status notification");
        return null;
    }
    
    // 3. Business validations
    if (!transLog.getMsgId().equals((String) trxInfo.get("originalMsgID"))) {
        logger.error("check msgID error.");
        return null;            
    }

    if (!transLog.getAcqInsCode().equals(msgInfo.getAcquirerIIN())) {
        logger.error("check AcquirerIIN error.");
        return null;            
    }
    
    // 4. Update based on response
    String respCode = (String) trxInfo.get("trxResponseCode");
    if (respCode.equals("00")) {
        transLog.setSettDate((String) trxInfo.get("settlementDate"));
        transLog.setSettCurr((String) trxInfo.get("settlementCurrency"));
        transLog.setSettAmt((String) trxInfo.get("settlementAmt"));
        transLog.setProcessFlag("02");   
    } else {
        transLog.setProcessFlag("05");   
    }
    
    // 5. Persist changes
    updateTransLog(transLog);
    
    return transLog;
}
```

---

# Part 4: Model Classes

![MyBatis Model Layer](https://github.com/user-attachments/assets/mybatis-model-layer)

## Purpose

Defines Java objects that represent database table records.

---

## 4.1 Entity Model

### Purpose

Maps database table rows to Java objects.

### Implementation Pattern

```java
package com.upi.adaptor.model;

public class TransLog implements Cloneable {

    // Primary Key
    private String msgId;
    
    // Regular Fields
    private String msgType;
    private String acqInsCode;
    private String transDatetime;
    private String processFlag;
    private String revFlag;
    private String transCurr;
    private String transAmt;
    private String origAmt;
    private String costAmt;
    private String billCurr;
    private String billAmt;
    private String billRate;
    private String settCurr;
    private String settAmt;
    private String settRate;
    private String currExponent;
    private String convDate;
    private String acctNum;
    private String merType;
    private String uaisRespCode;
    private String npsRespCode;
    private String settDate;
    private String uaisRetrievalRef;
    private String termId;
    private String merCode;
    private String merAddrName;
    private String merCountryCode;
    private String merCity;
    private String origMsgId;
    private String qrcPayload;
    private String discountDetails;
    private String qrcVoucherNo;
    private String qrcUseCase;
    private String npsRetrievalRef;
    private String npsStan;
    private String selfMsgHead;
    private String saSav1;
    private String recCreateTime;
    private String recUpdateTime;

    // Getters and Setters
    public String getMsgId() {
        return msgId;
    }

    public void setMsgId(String msgId) {
        this.msgId = msgId;
    }

    // ... additional getters/setters

    @Override
    public String toString() {
        return "TransInfo [msgID=" + msgId + ", msgType=" + msgType
                + ", transDateTime=" + transDatetime + ", npsRetrievalRef=" + npsRetrievalRef
                + ", updateTime=" + recUpdateTime + "]";
    }

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

### Responsibilities

- Represents database table structure.
- Provides getters/setters for property access.
- Implements Cloneable for object copying.
- Provides toString() for logging.

---

# Execution Flow

## Complete Request Flow

```
Application Service
       │
       ▼
┌──────────────────────────────────────┐
│  DAO Implementation                  │
│  @Service @Transactional             │
│  - Begin Transaction                 │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  MyBatis Mapper Interface            │
│  - Method invocation                 │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  SqlSessionTemplate                  │
│  - Creates SqlSession                │
│  - Manages session lifecycle         │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Mapper XML                          │
│  - Locates SQL by namespace.id       │
│  - Binds parameters                  │
│  - Executes SQL                      │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  JDBC / DataSource                   │
│  - Gets connection from pool         │
│  - Executes prepared statement       │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  ResultMap Processing                │
│  - Maps columns to properties        │
│  - Creates Java objects              │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Transaction Commit/Rollback         │
│  - Commits on success                │
│  - Rollbacks on exception            │
└──────────────────────────────────────┘
```

---

# Configuration Summary

## Configuration Files

| File | Type | Layer | Purpose |
|------|------|-------|---------|
| pom.xml | XML | Core | Maven dependencies |
| application.properties | Properties | Core | DataSource, MyBatis settings |
| HttpServiceMain.java | Java | Core | @MapperScan configuration |
| CurrencyCode.xml | XML | Mapper | SQL mappings for CurrencyCode |
| KeyInfo.xml | XML | Mapper | SQL mappings for KeyInfo |
| TimeoutInfo.xml | XML | Mapper | SQL mappings for TimeoutInfo |
| TransLog.xml | XML | Mapper | SQL mappings for TransLog |

---

## Bean Summary

| Bean Name | Class | Layer | Purpose |
|-----------|-------|-------|---------|
| dataSource | HikariDataSource | Core | Database connection pool |
| sqlSessionFactory | SqlSessionFactory | Core | Creates SqlSession instances |
| sqlSessionTemplate | SqlSessionTemplate | Core | Thread-safe SqlSession |
| transactionManager | DataSourceTransactionManager | Core | Transaction management |
| currencyCodeMapper | CurrencyCodeMapper | Mapper | Currency code operations |
| keyInfoMapper | KeyInfoMapper | Mapper | Key info operations |
| timeoutInfoMapper | TimeoutInfoMapper | Mapper | Timeout info operations |
| transLogMapper | TransLogMapper | Mapper | Transaction log operations |
| currencyCodeDaoImpl | CurrencyCodeDaoImpl | DAO | Currency code business logic |
| keyInfoDaoImpl | KeyInfoDaoImpl | DAO | Key info business logic |
| timeoutInfoDaoImpl | TimeoutInfoDaoImpl | DAO | Timeout info business logic |
| transLogDaoImpl | TransLogDaoImpl | DAO | Transaction log business logic |

---

## Mapper Summary

| Mapper Interface | XML File | Table | Operations |
|------------------|----------|-------|------------|
| CurrencyCodeMapper | CurrencyCode.xml | tbl_curr_code | findAll, findByID, insert, update, delete |
| KeyInfoMapper | KeyInfo.xml | tbl_key_info | findAll, findByID, insert, update, delete |
| TimeoutInfoMapper | TimeoutInfo.xml | tbl_timeout_info | findAll, findByID, findByQrcVouNo, insert, update, delete |
| TransLogMapper | TransLog.xml | tbl_trans_log | findAll, findByID, findByQrcVouNo, findByIDForUpdate, insert, update, delete |

---

## Properties Reference

### DataSource Properties

```properties
spring.datasource.url=jdbc:mysql://<host>:<port>/<database>?characterEncoding=utf8&useSSL=true
spring.datasource.username=<username>
spring.datasource.password=<password>
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
```

### Connection Pool Properties

```properties
# Maximum wait time for connection (ms)
spring.datasource.tomcat.max-wait=10000

# Maximum active connections
spring.datasource.tomcat.max-active=100

# Validate connection before use
spring.datasource.tomcat.test-on-borrow=true

# Initial pool size
spring.datasource.tomcat.initial-size=5
```

### MyBatis Properties

```properties
# Package for type aliases (short class names in XML)
mybatis.type-aliases-package=com.upi.adaptor.model

# Location of Mapper XML files
mybatis.mapper-locations=classpath:mapper/*.xml

# Optional: Enable underscore to camel case conversion
#mybatis.configuration.map-underscore-to-camel-case=true
```

---

## Database Table Summary

| Table Name | Primary Key | Model Class | Description |
|------------|-------------|-------------|-------------|
| tbl_curr_code | curr_code | CurrencyCode | Currency code reference data |
| tbl_key_info | ins_code | KeyInfo | Institution key/certificate info |
| tbl_timeout_info | msg_id | TimeoutInfo | Timeout tracking records |
| tbl_trans_log | msg_id | TransLog | Transaction log records |

---

## Column Mapping Reference

### tbl_trans_log

| Column Name | Java Property | JDBC Type |
|-------------|---------------|-----------|
| msg_id | msgId | VARCHAR |
| msg_type | msgType | VARCHAR |
| acq_ins_code | acqInsCode | VARCHAR |
| trans_datetime | transDatetime | VARCHAR |
| process_flag | processFlag | VARCHAR |
| rev_flag | revFlag | VARCHAR |
| trans_curr | transCurr | VARCHAR |
| trans_amt | transAmt | VARCHAR |
| orig_amt | origAmt | VARCHAR |
| cost_amt | costAmt | VARCHAR |
| billing_curr | billCurr | VARCHAR |
| billing_amt | billAmt | VARCHAR |
| bill_rate | billRate | VARCHAR |
| sett_curr | settCurr | VARCHAR |
| sett_amt | settAmt | VARCHAR |
| sett_rate | settRate | VARCHAR |
| curr_exponent | currExponent | VARCHAR |
| conv_date | convDate | VARCHAR |
| acct_num | acctNum | VARCHAR |
| mer_type | merType | VARCHAR |
| uais_resp_code | uaisRespCode | VARCHAR |
| nps_resp_code | npsRespCode | VARCHAR |
| sett_date | settDate | VARCHAR |
| uais_retrieval_ref | uaisRetrievalRef | VARCHAR |
| term_id | termId | VARCHAR |
| mer_code | merCode | VARCHAR |
| mer_addr_name | merAddrName | VARCHAR |
| mer_country_code | merCountryCode | VARCHAR |
| mer_city | merCity | VARCHAR |
| orig_msg_id | origMsgId | VARCHAR |
| qrc_payload | qrcPayload | VARCHAR |
| discount_details | discountDetails | VARCHAR |
| qrc_voucher_no | qrcVoucherNo | VARCHAR |
| qrc_use_case | qrcUseCase | VARCHAR |
| nps_retrieval_ref | npsRetrievalRef | VARCHAR |
| nps_stan | npsStan | VARCHAR |
| self_msg_head | selfMsgHead | VARCHAR |
| sa_sav1 | saSav1 | VARCHAR |
| rec_create_time | recCreateTime | VARCHAR |
| rec_update_time | recUpdateTime | VARCHAR |

### tbl_key_info

| Column Name | Java Property | JDBC Type |
|-------------|---------------|-----------|
| ins_code | insCode | VARCHAR |
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

### tbl_timeout_info

| Column Name | Java Property | JDBC Type |
|-------------|---------------|-----------|
| msg_id | msgId | VARCHAR |
| msg_type | msgType | VARCHAR |
| trans_datetime | transDatetime | VARCHAR |
| Qrc_Voucher_No | qrcVoucherNo | VARCHAR |
| Rec_create_time | recCreateTime | VARCHAR |
| Rec_timeout_time | recTimeoutTime | VARCHAR |

### tbl_curr_code

| Column Name | Java Property | JDBC Type |
|-------------|---------------|-----------|
| curr_code | currCode | VARCHAR |
| chn_desc | chnDesc | VARCHAR |
| chn_abb_desc | chnAbbrDesc | VARCHAR |
| eng_desc | engDesc | VARCHAR |
| eng_abb_desc | engAbbrDesc | VARCHAR |
| curr_exponent | currExponent | INTEGER |
| curr_sign | currSign | VARCHAR |
