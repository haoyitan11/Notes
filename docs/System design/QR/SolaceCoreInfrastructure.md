# Solace Core Infrastructure
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/f1d2ee7a-506a-42b6-9c64-afabdb952d23" />

### 1. JndiTemplate

#### Purpose

Provides JNDI context for Solace resource lookup.

#### Dependencies

```
None (Root Bean)
```

#### Java Declaration (SolaceConfiguration.java)

```java
@Bean
public JndiTemplate jndiTemplate() {
    Properties props = new Properties();
    props.put(InitialContext.PROVIDER_URL, url);
    props.put(InitialContext.INITIAL_CONTEXT_FACTORY, SolJNDIInitialContextFactory.class.getName());
    props.put(InitialContext.SECURITY_PRINCIPAL, username);
    props.put(SupportedProperty.SOLACE_JMS_VPN, vpn);
    // SSL properties for production...

    JndiTemplate jndi = new JndiTemplate();
    jndi.setEnvironment(props);
    return jndi;
}
```

#### Functions Used

```java
JndiTemplate jndi = new JndiTemplate();
jndi.setEnvironment(Properties props);
```

#### Responsibilities

- Creates JNDI context with Solace broker connection details
- Provides lookup capability for Solace-managed resources
- Configures authentication credentials
- Establishes connection to Solace VPN

---

### 2. JndiObjectFactoryBean (ConnectionFactory)

#### Purpose

Looks up ConnectionFactory from Solace JNDI.

#### Dependencies

```java
JndiTemplate
```

#### XML Declaration (spring-context.xml)

```xml
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean" 
      lazy-init="default" autowire="default">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.connection.factory}"/>
</bean>
```

#### Java Declaration (JmsConfiguration.java)

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

#### Functions Used

```java
JndiObjectFactoryBean jndiFactory = new JndiObjectFactoryBean();
jndiFactory.setJndiTemplate(JndiTemplate jndiTemplate);
jndiFactory.setJndiName(String jndiName);
jndiFactory.setProxyInterface(Class<?> proxyInterface);
```

#### Responsibilities

- Retrieves Solace ConnectionFactory via JNDI lookup
- Exposes ConnectionFactory as Spring bean
- Acts as bridge between JNDI and Spring context

---

### 3. JndiObjectFactoryBean (Queue)

#### Purpose

Looks up Queue destinations from Solace JNDI.

#### Dependencies

```java
JndiTemplate
```

#### Actual Usage (upi-request-adapter.xml)

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

#### Actual Usage (Spring Integration XML files)

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

#### XML Declaration (spring-context.xml)

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

#### Java Declaration (JmsConfiguration.java)

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

#### Functions Used

```java
CachingConnectionFactory cachedConnectionFactory = new CachingConnectionFactory();
cachedConnectionFactory.setTargetConnectionFactory(ConnectionFactory connectionFactory);
cachedConnectionFactory.setSessionCacheSize(int sessionCacheSize);
cachedConnectionFactory.setReconnectOnException(boolean reconnectOnException);
cachedConnectionFactory.setCacheConsumers(boolean cacheConsumers);
```

#### Responsibilities

- Wraps Solace ConnectionFactory
- Caches JMS sessions for improved performance
- Manages connection lifecycle with auto-reconnect

---

## Bean Dependencies

```
┌─────────────────────────┐
│     JndiTemplate        │  ◄── Root Bean (No Dependencies)
│  (solaceJndiTemplate)   │
└───────────┬─────────────┘
            │
            ├───────────────────────────────────────────────────────┐
            │                                                       │
            ▼                                                       ▼
┌───────────────────────────────┐       ┌───────────────────────────────────────┐
│  JndiObjectFactoryBean        │       │  JndiObjectFactoryBean (Queues)       │
│  (solaceConnectionFactory)    │       │  - upiAdapterReq.consumerQueue        │
│  [ConnectionFactory Lookup]   │       │  - reversal.consumerQueue             │
└───────────────┬───────────────┘       │  - processor.consumerQueue            │
                │                       │  - ap.processingRequestQueue          │
                ▼                       │  - fx.processingRequestQueue          │
┌───────────────────────────────┐       │  - tsp.processingRequestQueue         │
│  CachingConnectionFactory     │       │  - tspcardsm.processingRequestQueue   │
│  (solaceCachedConnectionFactory)      └───────────────────────────────────────┘
│  [Session Caching]            │
└───────────────┬───────────────┘
                │
                ▼
        [JMS Messaging Layer]
        [Spring Integration Layer]
```

---

## Configuration Files

| File | Contents |
|------|----------|
| `SolaceConfiguration.java` | JndiTemplate bean |
| `JmsConfiguration.java` | JndiObjectFactoryBean (ConnectionFactory), CachingConnectionFactory |
| `spring-context.xml` | XML-based beans (JndiTemplate, ConnectionFactory, CachingConnectionFactory) |
| `upi-request-adapter.xml` | JndiObjectFactoryBean (Consumer Queues) |
| `upi-proxy-transactions.xml` | JndiObjectFactoryBean (ap.processingRequestQueue) |
| `fx-hub-request.xml` | JndiObjectFactoryBean (fx.processingRequestQueue) |
| `tsp-request-enroll.xml` | JndiObjectFactoryBean (tsp.processingRequestQueue) |
| `tsp-request-cardsm.xml` | JndiObjectFactoryBean (tspcardsm.processingRequestQueue) |

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
