# Solace Core Infrastructure

## Purpose
Provides the foundational connectivity layer between the application and Solace Broker.

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e8c0924f-7ecf-490d-b328-478a48920e35" />

### 1. JndiTemplate

#### Purpose

Provides JNDI context for Solace resource lookup.

#### Dependencies

```
None (Root Bean)
```

#### Implementation Options

> **Choose ONE approach** - XML OR Java, not both.

**Option A: Java Configuration (SolaceConfiguration.java)**

```java
@Bean
public JndiTemplate jndiTemplate() {
    Properties props = new Properties();
    props.put(InitialContext.PROVIDER_URL, url);
    props.put(InitialContext.INITIAL_CONTEXT_FACTORY, SolJNDIInitialContextFactory.class.getName());
    props.put(InitialContext.SECURITY_PRINCIPAL, username);
    props.put(SupportedProperty.SOLACE_JMS_VPN, vpn);
    // SSL properties...

    JndiTemplate jndi = new JndiTemplate();
    jndi.setEnvironment(props);
    return jndi;
}
```

**Option B: XML Configuration (spring-context.xml)**

```xml
<bean id="solaceJndiTemplate" class="org.springframework.jndi.JndiTemplate">
    <property name="environment">
        <map>
            <entry key="java.naming.provider.url" value="${solace.url}" />
            <entry key="java.naming.factory.initial" 
                   value="com.solacesystems.jndi.SolJNDIInitialContextFactory" />
            <entry key="java.naming.security.principal" value="${solace.username}" />
            <entry key="Solace_JMS_VPN" value="${solace.vpn}" />
            <!-- SSL properties... -->
        </map>
    </property>
</bean>
```

#### Responsibilities

- Creates JNDI context with Solace broker connection details
- Provides lookup capability for Solace-managed resources
- Configures authentication credentials

---

### 2. JndiObjectFactoryBean (ConnectionFactory Lookup)

#### Purpose

Looks up ConnectionFactory from Solace JNDI.

#### Dependencies

```java
JndiTemplate
```

#### Implementation Options

> **Choose ONE approach** - XML OR Java, not both.

**Option A: Java Configuration (JmsConfiguration.java)**

```java
@Bean
public JndiObjectFactoryBean connectionFactory(JndiTemplate jndiTemplate) {
    final JndiObjectFactoryBean jndiFactory = new JndiObjectFactoryBean();
    jndiFactory.setJndiTemplate(jndiTemplate);
    jndiFactory.setJndiName(jndiName);
    jndiFactory.setProxyInterface(ConnectionFactory.class);
    return jndiFactory;
}
```

**Option B: XML Configuration (spring-context.xml)**

```xml
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean" 
      lazy-init="default" autowire="default">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.connection.factory}"/>
</bean>
```

#### Responsibilities

- Retrieves Solace ConnectionFactory via JNDI lookup
- Exposes ConnectionFactory as Spring bean

---

### 3. JndiObjectFactoryBean (Queue Lookup)

#### Purpose

Looks up Queue destinations from Solace JNDI.

#### Dependencies

```java
JndiTemplate
```

#### Implementation

> **XML Only** - Queue lookups are defined in XML configuration files.

**Actual Usage (upi-request-adapter.xml)**

```xml
<bean id="upiAdapterReq.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>

<bean id="reversal.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>

<bean id="processor.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.processor.response.queue.name}" />
</bean>
```

**Actual Usage (Spring Integration XML files)**

```xml
<!-- upi-proxy-transactions.xml -->
<bean id="ap.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.upi.proxy.transaction.request.queue.name}"/>
</bean>

<!-- fx-hub-request.xml -->
<bean id="fx.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.fx.query.request.queue.name}"/>
</bean>

<!-- tsp-request-enroll.xml -->
<bean id="tsp.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.tsp.request.queue.name.enroll}"/>
</bean>

<!-- tsp-request-cardsm.xml -->
<bean id="tspcardsm.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.tsp.request.queue.name.cardsm}"/>
</bean>
```

#### Responsibilities

- Retrieves Solace Queue via JNDI lookup
- Exposes Queue as Spring bean for JmsTemplate and Listener containers

---

### 4. CachingConnectionFactory

#### Purpose

Caches connections and sessions for performance.

#### Dependencies

```java
ConnectionFactory (from JndiObjectFactoryBean)
```

#### Implementation Options

> **Choose ONE approach** - XML OR Java, not both.

**Option A: Java Configuration (JmsConfiguration.java)**

```java
@Bean
public CachingConnectionFactory cachedConnectionFactoryfinal(ConnectionFactory connectionFactory) {
    final CachingConnectionFactory cachedConnectionFactory = new CachingConnectionFactory();
    cachedConnectionFactory.setTargetConnectionFactory(connectionFactory);
    cachedConnectionFactory.setSessionCacheSize(10);
    cachedConnectionFactory.setReconnectOnException(true);
    cachedConnectionFactory.setCacheConsumers(false);
    return cachedConnectionFactory;
}
```

**Option B: XML Configuration (spring-context.xml)**

```xml
<bean id="solaceCachedConnectionFactory" 
      class="org.springframework.jms.connection.CachingConnectionFactory"
      primary="true">
    <property name="targetConnectionFactory" ref="solaceConnectionFactory"/>
    <property name="sessionCacheSize" value="10"/>
    <property name="reconnectOnException" value="true"/>
    <property name="cacheConsumers" value="false"/>
</bean>
```

#### Responsibilities

- Wraps Solace ConnectionFactory
- Caches JMS sessions for improved performance
- Manages connection lifecycle with auto-reconnect

---

## Implementation Summary

| Component | Java Config | XML Config | Notes |
|-----------|-------------|------------|-------|
| JndiTemplate | ✅ SolaceConfiguration.java | ✅ spring-context.xml | Choose one |
| JndiObjectFactoryBean (ConnectionFactory) | ✅ JmsConfiguration.java | ✅ spring-context.xml | Choose one |
| JndiObjectFactoryBean (Queue) | ❌ | ✅ Various XML files | XML only |
| CachingConnectionFactory | ✅ JmsConfiguration.java | ✅ spring-context.xml | Choose one |

---

## Configuration Files

| File | Contents |
|------|----------|
| `SolaceConfiguration.java` | JndiTemplate bean (Java approach) |
| `JmsConfiguration.java` | ConnectionFactory, CachingConnectionFactory (Java approach) |
| `spring-context.xml` | JndiTemplate, ConnectionFactory, CachingConnectionFactory (XML approach) |
| `upi-request-adapter.xml` | Queue lookups (XML only) |
| `upi-proxy-transactions.xml` | Queue lookup for UPI transactions |
| `fx-hub-request.xml` | Queue lookup for FX queries |
| `tsp-request-enroll.xml` | Queue lookup for TSP enrollment |
| `tsp-request-cardsm.xml` | Queue lookup for TSP card management |

---

## Properties Reference

### Connection Properties

```properties
solace.url=smfs://<broker-host>:<port>
solace.username=<username>
solace.vpn=<vpn-name>
solace.connection.factory=jndi/cf/<connection-factory-name>
```

### Queue Properties

```properties
# Consumer Queues
solace.upiproxy.debit.response.queue.name=Q.CPS.00.P101.RES.JSON.LOC.DEBITRESPONSE.CPSPRO.CPSADP
solace.upiproxy.processor.response.queue.name=Q.CPS.00.P101.RES.JSON.LOC.UPI.CPSADP.UPIPROXY

# Listener Queues
solace.upi.proxy.transaction.request.queue.name=Q.CPS.00.P101.REQ.JSON.LOC.UPI.CPSADP.UPIPROXY
solace.fx.query.request.queue.name=Q.CPS.00.P101.REQ.JSON.SUB.FXHUB.FXENQRATE.UPI.CNY
solace.tsp.request.queue.name.enroll=Q.CPS.00.P101.REQ.JSON.SUB.TSP.ENROLL.UPIPROXY
solace.tsp.request.queue.name.cardsm=Q.CPS.00.P101.REQ.JSON.SUB.TSP.CARDSM.UPIPROXY
```
