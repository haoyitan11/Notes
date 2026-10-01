# Solace Core Infrastructure

## Purpose
Provides the foundational connectivity layer between the application and Solace Broker.

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e8c0924f-7ecf-490d-b328-478a48920e35" />

## Components

### 1. JndiTemplate

#### Purpose

Provides JNDI context for Solace resource lookup.

#### Dependencies

```
None (Root Bean)
```

#### Implementation (SolaceConfiguration.java)

```java
@Configuration
public class SolaceConfiguration {

    @Value("${solace.url}")
    private String url;

    @Value("${solace.username}")
    private String username;

    @Value("${solace.vpn}")
    private String vpn;

    @Bean
    public JndiTemplate jndiTemplate() {
        Properties props = new Properties();
        props.put(InitialContext.PROVIDER_URL, url);
        props.put(InitialContext.INITIAL_CONTEXT_FACTORY, SolJNDIInitialContextFactory.class.getName());
        props.put(InitialContext.SECURITY_PRINCIPAL, username);
        props.put(SupportedProperty.SOLACE_JMS_VPN, vpn);
        // Additional properties as needed...

        JndiTemplate jndi = new JndiTemplate();
        jndi.setEnvironment(props);
        return jndi;
    }
}
```

#### Responsibilities

- Creates JNDI context with Solace broker connection details
- Provides lookup capability for Solace-managed resources
- Configures authentication credentials

---

### 2. JndiObjectFactoryBean (ConnectionFactory)

#### Purpose

Looks up ConnectionFactory from Solace JNDI.

#### Dependencies

```
JndiTemplate
```

#### Implementation (JmsConfiguration.java)

```java
@Configuration
@EnableJms
public class JmsConfiguration {

    @Value("${solace.connection.factory}")
    private String jndiName;

    @Bean
    public JndiObjectFactoryBean connectionFactory(JndiTemplate jndiTemplate) {
        final JndiObjectFactoryBean jndiFactory = new JndiObjectFactoryBean();
        jndiFactory.setJndiTemplate(jndiTemplate);
        jndiFactory.setJndiName(jndiName);
        jndiFactory.setProxyInterface(ConnectionFactory.class);
        return jndiFactory;
    }
}
```

#### Responsibilities

- Retrieves Solace ConnectionFactory via JNDI lookup
- Exposes ConnectionFactory as Spring bean

---

### 3. JndiObjectFactoryBean (Queue)

#### Purpose

Looks up Queue destinations from Solace JNDI.

#### Dependencies

```
JndiTemplate
```

#### Implementation (QueueConfiguration.java)

```java
@Configuration
public class QueueConfiguration {

    // Queue JNDI names from properties
    @Value("${solace.queue.response.name}")
    private String responseQueueName;

    @Value("${solace.queue.processor.response.name}")
    private String processorResponseQueueName;

    // Response Queue
    @Bean
    public JndiObjectFactoryBean responseConsumerQueue(JndiTemplate jndiTemplate) {
        JndiObjectFactoryBean factory = new JndiObjectFactoryBean();
        factory.setJndiTemplate(jndiTemplate);
        factory.setJndiName(responseQueueName);
        return factory;
    }

    // Processor Response Queue
    @Bean
    public JndiObjectFactoryBean processorConsumerQueue(JndiTemplate jndiTemplate) {
        JndiObjectFactoryBean factory = new JndiObjectFactoryBean();
        factory.setJndiTemplate(jndiTemplate);
        factory.setJndiName(processorResponseQueueName);
        return factory;
    }
}
```

#### Responsibilities

- Retrieves Solace Queue via JNDI lookup
- Exposes Queue as Spring bean for JmsTemplate and Listener containers

---

### 4. CachingConnectionFactory

#### Purpose

Caches connections and sessions for performance.

#### Dependencies

```
ConnectionFactory (from JndiObjectFactoryBean)
```

#### Implementation (JmsConfiguration.java)

```java
@Bean
public CachingConnectionFactory cachedConnectionFactory(ConnectionFactory connectionFactory) {
    final CachingConnectionFactory cachedConnectionFactory = new CachingConnectionFactory();
    cachedConnectionFactory.setTargetConnectionFactory(connectionFactory);
    cachedConnectionFactory.setSessionCacheSize(10);
    cachedConnectionFactory.setReconnectOnException(true);
    cachedConnectionFactory.setCacheConsumers(false);
    return cachedConnectionFactory;
}
```

#### Responsibilities

- Wraps Solace ConnectionFactory
- Caches JMS sessions for improved performance
- Manages connection lifecycle with auto-reconnect

---

## Implementation Summary

| Component | Configuration Class |
|-----------|---------------------|
| JndiTemplate | SolaceConfiguration.java |
| JndiObjectFactoryBean (ConnectionFactory) | JmsConfiguration.java |
| JndiObjectFactoryBean (Queue) | QueueConfiguration.java |
| CachingConnectionFactory | JmsConfiguration.java |

---

## Configuration Files

| File | Contents |
|------|----------|
| `SolaceConfiguration.java` | JndiTemplate bean |
| `JmsConfiguration.java` | ConnectionFactory, CachingConnectionFactory |
| `QueueConfiguration.java` | Queue lookup beans |

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
solace.queue.response.name=<response-queue-jndi-name>
solace.queue.processor.response.name=<processor-response-queue-jndi-name>
```
