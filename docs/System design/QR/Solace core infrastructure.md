# Solace Message Broker Dependencies Handle

## Components Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           SOLACE MESSAGE BROKER ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         JNDI LAYER (Connection Bootstrap)                    │    │
│  │  ┌───────────────────┐    ┌─────────────────────────┐                       │    │
│  │  │   JndiTemplate    │───►│  JndiObjectFactoryBean  │                       │    │
│  │  │ (Solace Context)  │    │ (Resource Lookup)       │                       │    │
│  │  └───────────────────┘    └─────────────────────────┘                       │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                      CONNECTION LAYER (Session Management)                   │    │
│  │  ┌────────────────────────┐    ┌──────────────────────────────┐             │    │
│  │  │  ConnectionFactory     │───►│  CachingConnectionFactory    │             │    │
│  │  │  (Solace Native)       │    │  (Spring Wrapper)            │             │    │
│  │  └────────────────────────┘    └──────────────────────────────┘             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│              ┌───────────────────────────┼───────────────────────────┐              │
│              ▼                           ▼                           ▼              │
│  ┌───────────────────────┐   ┌───────────────────────┐   ┌────────────────────┐    │
│  │   PRODUCER LAYER      │   │   CONSUMER LAYER      │   │  ASYNC LISTENER    │    │
│  │  ┌─────────────────┐  │   │  ┌─────────────────┐  │   │  ┌──────────────┐  │    │
│  │  │   JmsTemplate   │  │   │  │   JmsTemplate   │  │   │  │DefaultMsg    │  │    │
│  │  │  (Send Mode)    │  │   │  │ (Receive Mode)  │  │   │  │ListenerCont. │  │    │
│  │  └─────────────────┘  │   │  └─────────────────┘  │   │  └──────────────┘  │    │
│  │          │            │   │          │            │   │        │           │    │
│  │          ▼            │   │          ▼            │   │        ▼           │    │
│  │  ┌─────────────────┐  │   │  ┌─────────────────┐  │   │  ┌──────────────┐  │    │
│  │  │SolaceMessage    │  │   │  │SolaceMessage    │  │   │  │message-driven│  │    │
│  │  │SenderOneWay     │  │   │  │Receiver         │  │   │  │channel-adapt │  │    │
│  │  └─────────────────┘  │   │  └─────────────────┘  │   │  └──────────────┘  │    │
│  └───────────────────────┘   └───────────────────────┘   └────────────────────┘    │
│                                                                    │                │
│                          SPRING INTEGRATION LAYER                  ▼                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  Channel ──► Transformer ──► ServiceActivator ──► Transformer ──► Adapter   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. JndiTemplate

### Purpose

Provides JNDI context for Solace resource lookup with SSL certificate authentication support.

### Configuration Source

**Java Configuration (Production - SolaceConfiguration.java):**

```java
@Configuration
@Profile("!local")
public class SolaceConfiguration {

    @Value("${solace.url}")
    private String url;

    @Value("${solace.username}")
    private String username;

    @Value("${solace.vpn}")
    private String vpn;

    @Value("${solace.jms.ssl.validate_certificate}")
    private String certificate;

    @Value("${solace.jms.ssl.keystore}")
    private String keystore;

    @Value("${solace.jms.ssl.keystore.password}")
    private String keystorePassword;

    @Value("${solace.jms.ssl.authentication.scheme}")
    private String authenticationScheme;

    @Value("${solace.jms.ssl.privatekey.alias}")
    private String privateKeyAlias;

    @Value("${solace.jms.ssl.privatekey.password}")
    private String privateKeyPassword;

    @Value("${solace.jms.ssl.truststore}")
    private String trustStore;

    @Value("${solace.jms.ssl.truststore.password}")
    private String trustStorePassword;

    @Bean
    public JndiTemplate jndiTemplate() {
        Properties props = new Properties();
        props.put(InitialContext.PROVIDER_URL, url);
        props.put(InitialContext.INITIAL_CONTEXT_FACTORY, SolJNDIInitialContextFactory.class.getName());
        props.put(InitialContext.SECURITY_PRINCIPAL, username);
        props.put(SupportedProperty.SOLACE_JMS_VPN, vpn);
        props.put(SupportedProperty.SOLACE_JMS_SSL_VALIDATE_CERTIFICATE, true);
        props.put(SupportedProperty.SOLACE_JMS_SSL_VALIDATE_CERTIFICATE_HOST, false);
        props.put(SupportedProperty.SOLACE_JMS_AUTHENTICATION_SCHEME, authenticationScheme);
        props.put(SupportedProperty.SOLACE_JMS_SSL_KEY_STORE, keystore);
        props.put(SupportedProperty.SOLACE_JMS_SSL_KEY_STORE_PASSWORD, keystorePassword);
        props.put(SupportedProperty.SOLACE_JMS_SSL_PRIVATE_KEY_ALIAS, privateKeyAlias);
        props.put(SupportedProperty.SOLACE_JMS_SSL_PRIVATE_KEY_PASSWORD, privateKeyPassword);
        props.put(SupportedProperty.SOLACE_JMS_SSL_TRUST_STORE, trustStore);
        props.put(SupportedProperty.SOLACE_JMS_SSL_TRUST_STORE_PASSWORD, trustStorePassword);

        JndiTemplate jndi = new JndiTemplate();
        jndi.setEnvironment(props);
        return jndi;
    }
}
```

**Java Configuration (Local - LocalSolaceConfiguration.java):**

```java
@Configuration
@Profile("local")
public class LocalSolaceConfiguration {

    @Value("${solace.url}")
    private String url;

    @Value("${solace.username}")
    private String username;

    @Value("${solace.vpn}")
    private String vpn;

    @Value("${solace.password}")
    private String password;

    @Bean
    public JndiTemplate jndiTemplate() {
        Properties props = new Properties();
        props.put(InitialContext.PROVIDER_URL, url);
        props.put(InitialContext.INITIAL_CONTEXT_FACTORY, SolJNDIInitialContextFactory.class.getName());
        props.put(InitialContext.SECURITY_PRINCIPAL, username);
        props.put(InitialContext.SECURITY_CREDENTIALS, password);
        props.put(SupportedProperty.SOLACE_JMS_VPN, vpn);

        JndiTemplate jndi = new JndiTemplate();
        jndi.setEnvironment(props);
        return jndi;
    }
}
```

**XML Declaration (spring-context.xml) - Alternative Profile-based:**

```xml
<beans profile="dev">
    <bean id="solaceJndiTemplate" class="org.springframework.jndi.JndiTemplate">
        <property name="environment">
            <map>
                <entry key="java.naming.provider.url" value="${solace.url}" />
                <entry key="java.naming.factory.initial" 
                       value="com.solacesystems.jndi.SolJNDIInitialContextFactory" />
                <entry key="java.naming.security.principal" value="${solace.username}" />
                <entry key="java.naming.security.credentials" value="${solace.password}" />
                <entry key="Solace_JMS_VPN" value="${solace.vpn}" />
            </map>
        </property>
    </bean>
</beans>

<beans profile="SIT,UAT,PROD">
    <bean id="solaceJndiTemplate" class="org.springframework.jndi.JndiTemplate">
        <property name="environment">
            <map>
                <entry key="java.naming.provider.url" value="${solace.url}" />
                <entry key="java.naming.factory.initial" 
                       value="com.solacesystems.jndi.SolJNDIInitialContextFactory" />
                <entry key="java.naming.security.principal" value="${solace.username}" />
                <entry key="Solace_JMS_VPN" value="${solace.vpn}" />
                <entry key="Solace_JMS_SSL_ValidateCertificate" value-type="java.lang.Boolean" 
                       value="${Solace_JMS_SSL_ValidateCertificate}" />
                <entry key="Solace_JMS_SSL_ValidateCertificateHost" value-type="java.lang.Boolean" 
                       value="false" />
                <entry key="Solace_JMS_SSL_KeyStore" value="${Solace_JMS_SSL_KeyStore}" />
                <entry key="Solace_JMS_SSL_KeyStorePassword" value="${Solace_JMS_SSL_KeyStorePassword}" />
                <entry key="Solace_JMS_Authentication_Scheme" value="${Solace_JMS_Authentication_Scheme}" />
                <entry key="Solace_JMS_SSL_PrivateKeyAlias" value="${Solace_JMS_SSL_PrivateKeyAlias}" />
                <entry key="Solace_JMS_SSL_TrustStore" value="${Solace_JMS_SSL_TrustStore}" />
                <entry key="Solace_JMS_SSL_TrustStorePassword" value="${Solace_JMS_SSL_TrustStorePassword}" />
                <entry key="Solace_JMS_SSL_PrivateKeyPassword" value="${Solace_JMS_SSL_PrivateKeyPassword}" />
            </map>
        </property>
    </bean>
</beans>
```

### Responsibilities

- Creates JNDI context with Solace broker connection details
- Provides lookup capability for Solace-managed resources
- Configures authentication (username/password for local, SSL certificates for production)
- Establishes connection to Solace VPN

---

## 2. JndiObjectFactoryBean

### Purpose

Looks up ConnectionFactory and Queue from Solace JNDI.

### XML Declaration (spring-context.xml)

```xml
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean" 
      lazy-init="default" autowire="default">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.connection.factory}"/>
</bean>
```

### Java Configuration (JmsConfiguration.java)

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

### Queue Lookup Examples (upi-request-adapter.xml)

```xml
<bean id="upiAdapterReq.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>

<bean id="processor.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.processor.response.queue.name}" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jndiTemplate | `solaceJndiTemplate` or `jndiTemplate` |

### Responsibilities

- Retrieves Solace-managed JMS objects via JNDI lookup
- Exposes ConnectionFactory and Queue as Spring beans
- Acts as bridge between JNDI and Spring context

---

## 3. CachingConnectionFactory

### Purpose

Acts as the central JMS connection manager that caches connections and sessions for performance.

### XML Declaration (spring-context.xml)

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

### Java Configuration (JmsConfiguration.java)

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

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| targetConnectionFactory | `solaceConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| sessionCacheSize | 10 | Number of JMS sessions to cache |
| reconnectOnException | true | Auto-reconnect on connection failure |
| cacheConsumers | false | Consumer caching disabled for dynamic destinations |

### Responsibilities

- Wraps Solace ConnectionFactory
- Caches JMS sessions for improved performance
- Manages connection lifecycle with auto-reconnect
- Provides thread-safe session management

---

## 4. Producer JmsTemplate

### Purpose

Provides outbound JMS operations for sending messages to topics.

### XML Declaration (upi-request-adapter.xml)

```xml
<bean id="producerJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <constructor-arg ref="solaceCachedConnectionFactory" />
    <property name="pubSubDomain" value="true" />
</bean>
```

### Java Configuration (JmsConfiguration.java)

```java
@Bean
public JmsTemplate jmsTemplate(CachingConnectionFactory connectionFactory, 
                               MessageConverter jackson2MessageConverter) {
    final JmsTemplate jmsTemplate = new JmsTemplate(connectionFactory);
    jmsTemplate.setPubSubDomain(true);
    jmsTemplate.setMessageConverter(jackson2MessageConverter);
    jmsTemplate.setReceiveTimeout(receiveTimeout);
    return jmsTemplate;
}
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| connectionFactory | `solaceCachedConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| pubSubDomain | true | Uses Topic mode (publish-subscribe) |
| messageConverter | jackson2MessageConverter | JSON message converter |

### Actual Usage

```java
jmsTemplate.send(destination, new MessageCreator() {
    public Message createMessage(Session session) throws JMSException {
        Message message = session.createTextMessage(msg);
        message.setJMSCorrelationID(correlationId);
        return message;
    }
});
```

### Responsibilities

- Sends JMS messages to topics
- Creates and manages JMS sessions internally
- Provides send operations with MessageCreator callback
- Handles message conversion automatically

---

## 5. Receiver JmsTemplate

### Purpose

Provides inbound JMS receive operations with selector support for correlation-based message retrieval.

### XML Declaration (upi-request-adapter.xml)

```xml
<bean id="upiAdapterReqMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="upiAdapterReq.consumerQueue" />
    <property name="receiveTimeout" value="${upi.message.receive.timeout}" />
</bean>

<bean id="reversalMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="reversal.consumerQueue" />
    <property name="receiveTimeout" value="${upi.message.receive.timeout}" />
</bean>

<bean id="processorMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="processor.consumerQueue" />
    <property name="receiveTimeout" value="${upi.message.receive.timeout}" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| connectionFactory | `solaceCachedConnectionFactory` |
| defaultDestination | Queue bean (e.g., `upiAdapterReq.consumerQueue`) |

### Properties

| Property | Source | Description |
|----------|--------|-------------|
| receiveTimeout | `${upi.message.receive.timeout}` | Timeout for blocking receive |

### Actual Usage

```java
String resCorrelationId = "JMSCorrelationID = '" + correlationId + "'";
Message message = jmsTemplate.receiveSelected(resCorrelationId);
```

### Responsibilities

- Receives response messages from queue
- Filters messages using JMS selectors (correlation ID)
- Provides synchronous receive with configurable timeout
- Supports request-reply pattern

---

## 6. SolaceMessageSenderOneWay

### Purpose

Application-level component that handles outbound messaging to Solace topics with optional correlation support.

### XML Declaration (upi-request-adapter.xml)

```xml
<bean id="upiDebitMessageProducer"
    class="com.nets.upi.processing.integration.upi.proxy.service.impl.SolaceMessageSenderOneWay">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>
```

### Java Implementation (SolaceMessageSenderOneWay.java)

```java
@Component
public class SolaceMessageSenderOneWay {
    
    private JmsTemplate jmsTemplate;

    public void sendMessages(final String msg, String destination) {
        getJmsTemplate().send(destination, new MessageCreator() {
            public Message createMessage(Session session) throws JMSException {
                Message message = session.createTextMessage(msg);
                return message;
            }
        });
    }
    
    public void sendMessages(final String msg, String destination, String correlationId) {
        getJmsTemplate().send(destination, new MessageCreator() {
            public Message createMessage(Session session) throws JMSException {
                Message message = session.createTextMessage(msg);
                message.setJMSCorrelationID(correlationId);
                return message;
            }
        });
    }

    public JmsTemplate getJmsTemplate() {
        return jmsTemplate;
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
}
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jmsTemplate | `producerJmsTemplate` |

### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

```java
@Qualifier("upiDebitMessageProducer")
@Autowired
private SolaceMessageSenderOneWay solaceMessageSenderOneway;

@Value("${solace.upiproxy.mpqrc.request.issueAdapter.topic.name}")
private String mpqrcPaymentDestination;

// Send message with correlation ID
solaceMessageSenderOneway.sendMessages(
    mapper.writeValueAsString(upiProxyRequest), 
    mpqrcPaymentDestination, 
    correlationId
);
```

### Responsibilities

- Sends outbound text messages to specified topic
- Optionally sets `JMSCorrelationID` for request-reply pattern
- Provides simple one-way messaging capability
- Uses MessageCreator for message construction

---

## 7. SolaceMessageSender (External Library)

### Purpose

Application-level component from `nps-qr-common` library that handles outbound messaging with reply-to topic support.

### XML Declaration (upi-request-adapter.xml)

```xml
<bean id="upiAdapterReqMessageProducer"
    class="com.nets.nps.qr.common.solace.SolaceMessageSender">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jmsTemplate | `producerJmsTemplate` |

### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

```java
@Qualifier("upiAdapterReqMessageProducer")
@Autowired
private SolaceMessageSender solaceMessageSender;

@Value("${solace.upiproxy.processor.request.topic.name}")
private String processorResultDestination;

@Value("${solace.upiproxy.processor.response.topic.name}")
private String processorReplyToTopic;

// Send message with correlation ID and reply-to topic
solaceMessageSender.sendMessages(
    mapper.writeValueAsString(upiProxyRequest), 
    correlationId,
    processorReplyToTopic, 
    processorResultDestination
);
```

### Responsibilities

- Sends outbound text messages
- Sets `JMSCorrelationID` for request-reply matching
- Sets `JMSReplyTo` header for response routing
- Supports request-reply messaging pattern

---

## 8. SolaceMessageReceiver (External Library)

### Purpose

Application-level component from `nps-qr-common` library that receives response messages using correlation ID filtering.

### XML Declaration (upi-request-adapter.xml)

```xml
<bean id="upiAdapterReqMessageReceiver"
    class="com.nets.nps.qr.common.solace.SolaceMessageReceiver">
    <property name="jmsTemplate" ref="upiAdapterReqMessageReceiverJmsTemplate" />
</bean>

<bean id="reversalMessageReceiver"
    class="com.nets.nps.qr.common.solace.SolaceMessageReceiver">
    <property name="jmsTemplate" ref="reversalMessageReceiverJmsTemplate" />
</bean>

<bean id="processorMessageReceiver"
    class="com.nets.nps.qr.common.solace.SolaceMessageReceiver">
    <property name="jmsTemplate" ref="processorMessageReceiverJmsTemplate" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jmsTemplate | Receiver JmsTemplate (e.g., `upiAdapterReqMessageReceiverJmsTemplate`) |

### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

```java
@Qualifier("upiAdapterReqMessageReceiver")
@Autowired
private SolaceMessageReceiver solaceMessageReceiver;

@Qualifier("processorMessageReceiver")
@Autowired
private SolaceMessageReceiver processorSolaceMessageReceiver;

// Receive message by correlation ID
String response = solaceMessageReceiver.receiveMessage(correlationId);

// Or from processor queue
String response = processorSolaceMessageReceiver.receiveMessage(correlationId);
```

### Responsibilities

- Receives messages filtered by correlation ID using JMS selector
- Extracts TextMessage payload
- Handles receive timeout gracefully
- Returns null if no message received within timeout

---

## 9. SolaceReversalMessageReceiver

### Purpose

Application-level component for receiving reversal response messages with correlation ID filtering.

### Java Implementation (SolaceReversalMessageReceiver.java)

```java
@Component
public class SolaceReversalMessageReceiver {

    private static final ApsLogger logger = new ApsLogger(SolaceReversalMessageReceiver.class);
    
    protected JmsTemplate jmsTemplateReversal;

    public String receiveMessage(String correlationId) {
        logger.info("SolaceReversalMessageReceiver trying to receive message correlation id:" + correlationId);
        String resCorrelationId = "JMSCorrelationID = '" + correlationId + "'";
        logger.info("getJmsTemplate value ::" + getJmsTemplate());
        Message message = getJmsTemplate().receiveSelected(resCorrelationId);
        if (message instanceof TextMessage) {
            TextMessage txtMsg = (TextMessage) message;
            try {
                Object msgTextObj = txtMsg.getText();
                logger.info("msgTextObj :: " + msgTextObj.toString());
                return msgTextObj.toString();
            } catch (JMSException e) {
                logger.error("SolaceReversalMessageReceiver Receiving message Error");
                e.printStackTrace();
            }
        } else {
            return null;
        }
        return null;
    }

    public JmsTemplate getJmsTemplate() {
        return jmsTemplateReversal;
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplateReversal = jmsTemplate;
    }
}
```

### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

```java
@Qualifier("reversalMessageReceiver")
@Autowired
private SolaceMessageReceiver solaceReversalMessageReceiver;

public String sendToReversalAndGetResponse(String reversalRequestStr, String correlationId)
        throws IOException {
    logger.info("inside sendToReversalAndGetResponse");
    solaceMessageSender.sendMessages(reversalRequestStr, correlationId,
            reversalReplyToTopic, reversalDestination);
    String response = solaceReversalMessageReceiver.receiveMessage(correlationId);
    if (response != null) {
        logger.info("Response from sendToReversalAndGetResponse :: ");
    } else {
        logger.info("No response For Reversal ");
    }
    return response;
}
```

### Responsibilities

- Receives messages filtered by correlation ID
- Builds JMS selector string: `JMSCorrelationID = '<correlationId>'`
- Extracts text content from TextMessage
- Handles JMS exceptions gracefully

---

## 10. DefaultMessageListenerContainer

### Purpose

Provides asynchronous JMS consumption with concurrent consumers for high-throughput message processing.

### XML Declaration (upi-proxy-transactions.xml)

```xml
<bean id="ap.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.upi.proxy.transaction.request.queue.name}"/>
</bean>

<bean id="ap.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="ap.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    <property name="concurrentConsumers" value="${high.usage.concurrent.consumers}"/>
    <property name="maxConcurrentConsumers" value="${high.usage.max.concurrent.consumers}"/>
    <property name="idleConsumerLimit" value="${high.usage.idle.consumer.limit}"/>
    <property name="receiveTimeout" value="${high.usage.receive.timeout}"/>
    <property name="idleTaskExecutionLimit" value="${high.usage.idle.taskexecution.limit}"/>
</bean>
```

### Additional Listener Containers

**FX Hub (fx-hub-request.xml):**
```xml
<bean id="fx.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="fx.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    <property name="concurrentConsumers" value="${low.usage.concurrent.consumers}"/>
    <property name="maxConcurrentConsumers" value="${low.usage.max.concurrent.consumers}"/>
    ...
</bean>
```

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<bean id="tsp.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="tsp.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    ...
</bean>
```

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<bean id="tspcardsm.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="tspcardsm.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    ...
</bean>
```

### Java Configuration (JmsConfiguration.java)

```java
@Bean
public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
        CachingConnectionFactory connectionFactory,
        MessageConverter jackson2MessageConverter,
        TaskExecutor defaultJmsExecutor) {
    final DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setErrorHandler(new JmsExceptionHandler());
    factory.setConcurrency(concurrentConsumersMin + "-" + concurrentConsumersMax);
    factory.setReceiveTimeout(receiveTimeout);
    factory.setMessageConverter(jackson2MessageConverter);
    factory.setTaskExecutor(defaultJmsExecutor);
    return factory;
}

@Bean
public TaskExecutor defaultJmsExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(concurrentConsumersMin);
    executor.setMaxPoolSize(concurrentConsumersMax);
    executor.setQueueCapacity(queueCapacity);
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(timeoutInSec);
    return executor;
}
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| destination | Queue bean (e.g., `ap.processingRequestQueue`) |
| connectionFactory | `solaceConnectionFactory` |

### Properties

| Property | Source | Description |
|----------|--------|-------------|
| concurrentConsumers | `${high.usage.concurrent.consumers}` | Initial consumer threads |
| maxConcurrentConsumers | `${high.usage.max.concurrent.consumers}` | Maximum consumer threads |
| idleConsumerLimit | `${high.usage.idle.consumer.limit}` | Idle consumers before scaling down |
| receiveTimeout | `${high.usage.receive.timeout}` | Receive operation timeout |

### Responsibilities

- Listens to inbound queue asynchronously
- Creates and manages JMS consumers
- Scales consumers based on load (min to max)
- Provides graceful shutdown support

---

## 11. message-driven-channel-adapter

### Purpose

Bridges JMS messages into Spring Integration channels for processing.

### XML Declaration (upi-proxy-transactions.xml)

```xml
<int-jms:message-driven-channel-adapter
        id="ap.jmsIn"
        channel="ap.jmsInChannel"
        container="ap.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

### Additional Adapters

**FX Hub (fx-hub-request.xml):**
```xml
<int-jms:message-driven-channel-adapter
        id="fx.jmsIn"
        channel="fx.jmsInChannel"
        container="fx.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<int-jms:message-driven-channel-adapter
        id="tsp.jmsIn"
        channel="tsp.jmsInChannel"
        container="tsp.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| container | `ap.messageListenerContainer` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| channel | ap.jmsInChannel | Output Spring Integration channel |
| extract-payload | true | Extracts message body (not full Message object) |
| error-channel | errorChannel | Channel for error handling |

### Responsibilities

- Converts incoming JMS messages to Spring Integration messages
- Extracts payload from TextMessage
- Routes to Spring Integration channel for processing
- Handles errors via error channel

---

## 12. Spring Integration Channels

### Purpose

Provides message routing infrastructure between adapters and service activators.

### XML Declaration (upi-proxy-transactions.xml)

```xml
<int:logging-channel-adapter id="apLog" level="INFO" log-full-message="true" logger-name="apLog"/>

<int:channel id="ap.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="apLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="ap.inChannel"/>
```

### Output Channels (common.xml)

```xml
<int:channel id="outputChannel" />

<int:object-to-json-transformer input-channel="outputChannel" output-channel="jmsOutChannel" />

<int:channel id="jmsOutChannel" />
```

### Responsibilities

- Routes messages between components
- Provides wire-tap for logging
- Supports transformation pipeline
- Enables decoupled message processing

---

## 13. json-to-object-transformer

### Purpose

Transforms JSON string messages to Java objects for service processing.

### XML Declaration (upi-proxy-transactions.xml)

```xml
<int:json-to-object-transformer 
    input-channel="ap.jmsInChannel" 
    output-channel="ap.inChannel"
    type="com.nets.upi.domain.UpiProxyRequest">
</int:json-to-object-transformer>
```

### Additional Transformers

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<int:json-to-object-transformer 
    input-channel="tsp.jmsInChannel" 
    output-channel="tsp.inChannel"
    type="com.nets.upi.domain.CardEnrollmentUPIRequestTsp">
</int:json-to-object-transformer>
```

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<int:json-to-object-transformer 
    input-channel="tspcardsm.jmsInChannel" 
    output-channel="tspcardsm.inChannel"
    type="com.nets.upi.domain.LifecycleManagementUPIRequest">
</int:json-to-object-transformer>
```

### Responsibilities

- Deserializes JSON string to specified Java type
- Enables type-safe message processing
- Supports domain object transformation

---

## 14. Service Activators

### Purpose

Service components that process messages and generate responses.

### XML Declaration (upi-proxy-transactions.xml)

```xml
<int:service-activator
        input-channel="ap.inChannel"
        output-channel="outputChannel"
        ref="transactionUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

### Java Implementation (TransactionUpiProxyProcessingService.java)

```java
@Service
public class TransactionUpiProxyProcessingService {

    private final Logger logger = LoggerFactory.getLogger(TransactionUpiProxyProcessingService.class);
    
    @Autowired
    private GetUPIRequestURI getUPIRequestURI;
    
    @Autowired
    private ExchangeService exchangeService;

    @ServiceActivator
    public UpiProxyRequest process(UpiProxyRequest upiProxyRequest) {
        logger.info("TransactionUpiProxyProcessingService process method start");
        
        String upiRestUrl = getUPIRequestURI.getUpiRequestUrl(upiProxyRequest.getTransactionType());
        String upiProxyResponse = "";
        try {
            upiProxyResponse = exchangeService.getUPIResponse(
                upiProxyRequest.getUpiProxyRequestJsonData(), 
                upiRestUrl, 
                upiProxyRequest.getInstCode()
            );
        } catch (ResourceAccessException ex) {
            logger.error("Exception occurred while processing ", ex);
        } catch (Exception e) {
            logger.error("Exception occurred while processing ", e);
        }
        upiProxyRequest.setUpiProxyResponseJsonData(upiProxyResponse);
        logger.info("TransactionUpiProxyProcessingService process method end");
        return upiProxyRequest;
    }
}
```

### Additional Service Activators

**FxQueryUpiProxyProcessingService:**
```java
@Service
public class FxQueryUpiProxyProcessingService {

    @ServiceActivator
    public String process(String fxQueryRequest) throws ParseException {
        logger.info("FxQueryUpiProxyProcessingService process method start:" + fxQueryRequest);
        // Process FX query request
        return upiProxyResponse;
    }
}
```

**TspUpiProxyProcessingService:**
```java
@Service
public class TspUpiProxyProcessingService {

    @ServiceActivator
    public String process(CardEnrollmentUPIRequestTsp tspRequest) throws ParseException {
        logger.info("TspUpiProxyProcessingService process method start");
        // Process TSP enrollment request
        return tspResponse;
    }
}
```

**LifecycleManagementUpiProxyProcessingService:**
```java
@Service
public class LifecycleManagementUpiProxyProcessingService {

    @ServiceActivator
    public String process(LifecycleManagementUPIRequest request) throws ParseException {
        logger.info("LifecycleManagementUpiProxyProcessingService process method start");
        // Process lifecycle management request
        return tspResponse;
    }
}
```

### How @ServiceActivator Works

1. XML `ref="transactionUpiProxyProcessingService"` references the `@Service` bean
2. Spring Integration invokes the method annotated with `@ServiceActivator`
3. Method parameter receives the transformed payload
4. Method return value is sent to output channel

### Responsibilities

- Receives message payload as method parameter
- Performs business logic processing (REST calls to external systems)
- Returns response object (automatically sent to output channel)

---

## 15. object-to-json-transformer

### Purpose

Transforms Java objects back to JSON string for JMS output.

### XML Declaration (common.xml)

```xml
<int:object-to-json-transformer 
    input-channel="outputChannel" 
    output-channel="jmsOutChannel" />
```

### Responsibilities

- Serializes Java objects to JSON string
- Prepares response for JMS transmission
- Supports domain object serialization

---

## 16. outbound-channel-adapter

### Purpose

Sends Spring Integration messages back to Solace JMS topics.

### XML Declaration (common.xml)

```xml
<int-jms:outbound-channel-adapter 
    id="jmsOut" 
    pub-sub-domain="true"
    destination-expression="headers['jms_replyTo']"
    channel="jmsOutChannel"
    connection-factory="solaceConnectionFactory">
</int-jms:outbound-channel-adapter>
```

### Static Destination Adapters

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<int-jms:outbound-channel-adapter 
    id="outputChannelCardsm"
    pub-sub-domain="true" 
    destination-expression="'${solace.tsp.response.topic.name.cardsm}'"
    channel="jsonJMSOutChannelCardsm" 
    connection-factory="solaceCachedConnectionFactory">
</int-jms:outbound-channel-adapter>
```

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<int-jms:outbound-channel-adapter 
    id="outputChannelCardEnroll"
    pub-sub-domain="true" 
    destination-expression="'${solace.tsp.response.topic.name.enroll}'"
    channel="jsonJMSOutChannelCardEnroll" 
    connection-factory="solaceCachedConnectionFactory">
</int-jms:outbound-channel-adapter>
```

**FX Hub (fx-hub-request.xml):**
```xml
<int-jms:outbound-channel-adapter 
    id="outputChannelFx"
    pub-sub-domain="true" 
    destination-expression="'${solace.upiproxy.to.fx.topic.name}'"
    channel="jsonJMSOutChannelFx" 
    connection-factory="solaceCachedConnectionFactory">
</int-jms:outbound-channel-adapter>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| connection-factory | `solaceConnectionFactory` or `solaceCachedConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| channel | jmsOutChannel | Input Spring Integration channel |
| pub-sub-domain | true | Uses Topic mode |
| destination-expression | `headers['jms_replyTo']` or static topic | Dynamic/static destination |

### Responsibilities

- Publishes response messages to Solace topics
- Dynamically routes to `JMSReplyTo` destination from original request
- Supports static destination configuration
- Converts Spring Integration message to JMS message

---

## Bean Dependencies Summary

| Bean | Depends On |
|------|------------|
| `jndiTemplate` / `solaceJndiTemplate` | (none - root bean) |
| `solaceConnectionFactory` / `connectionFactory` | `jndiTemplate` |
| `solaceCachedConnectionFactory` / `cachedConnectionFactory` | `solaceConnectionFactory` |
| `*.processingRequestQueue` / `*.consumerQueue` | `solaceJndiTemplate` |
| `producerJmsTemplate` | `solaceCachedConnectionFactory` |
| `*MessageReceiverJmsTemplate` | `solaceCachedConnectionFactory`, Queue bean |
| `upiAdapterReqMessageProducer` | `producerJmsTemplate` |
| `upiDebitMessageProducer` | `producerJmsTemplate` |
| `*MessageReceiver` | Receiver JmsTemplate |
| `*.messageListenerContainer` | Queue bean, `solaceConnectionFactory` |
| `*.jmsIn` (message-driven-channel-adapter) | `*.messageListenerContainer` |
| `*ProcessingService` | (none - @Service) |
| `jmsOut` / `outputChannel*` (outbound-channel-adapter) | `solaceConnectionFactory` |

---

## Message Flow Patterns

### Pattern 1: Asynchronous Request Processing (Spring Integration)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    SPRING INTEGRATION MESSAGE FLOW                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Solace Queue                                                                    │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ DefaultMessageListener     │  Concurrent consumers receive from queue        │
│  │ Container                  │                                                  │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ message-driven-channel-    │  Converts JMS → Spring Integration message      │
│  │ adapter                    │  extract-payload="true"                         │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ jmsInChannel              │  With wire-tap for logging                       │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ json-to-object-transformer│  JSON String → Domain Object                     │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ inChannel                 │                                                   │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ service-activator         │  @ServiceActivator method processes request      │
│  │ (ProcessingService)       │  Calls external REST APIs                        │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ outputChannel             │                                                   │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ object-to-json-transformer│  Domain Object → JSON String                     │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ jmsOutChannel             │                                                   │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  ┌────────────────────────────┐                                                 │
│  │ outbound-channel-adapter  │  destination-expression="headers['jms_replyTo']" │
│  └────────────────────────────┘                                                 │
│       │                                                                          │
│       ▼                                                                          │
│  Solace Topic (ReplyTo)                                                         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Synchronous Request-Reply (JmsTemplate)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    REQUEST-REPLY MESSAGE FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. Build Request                                                                │
│     ┌────────────────────────────────────────────────┐                          │
│     │ ObjectMapper mapper = new ObjectMapper();      │                          │
│     │ String json = mapper.writeValueAsString(req);  │                          │
│     │ String correlationId = UUID.randomUUID();      │                          │
│     └────────────────────────────────────────────────┘                          │
│                          │                                                       │
│                          ▼                                                       │
│  2. Send Message                                                                 │
│     ┌────────────────────────────────────────────────┐                          │
│     │ solaceMessageSender.sendMessages(              │                          │
│     │     json,                                      │                          │
│     │     correlationId,                             │                          │
│     │     replyToTopic,                              │                          │
│     │     destinationTopic                           │                          │
│     │ );                                             │                          │
│     └────────────────────────────────────────────────┘                          │
│                          │                                                       │
│                          ▼                                                       │
│     ┌────────────────────────────────────────────────┐                          │
│     │           Solace Topic (Destination)           │                          │
│     │     JMSCorrelationID = correlationId           │                          │
│     │     JMSReplyTo = replyToTopic                  │                          │
│     └────────────────────────────────────────────────┘                          │
│                          │                                                       │
│                          ▼                                                       │
│                   [External System Processes]                                    │
│                          │                                                       │
│                          ▼                                                       │
│     ┌────────────────────────────────────────────────┐                          │
│     │           Solace Queue (Response)              │                          │
│     │     JMSCorrelationID = correlationId           │                          │
│     └────────────────────────────────────────────────┘                          │
│                          │                                                       │
│                          ▼                                                       │
│  3. Receive Response                                                             │
│     ┌────────────────────────────────────────────────┐                          │
│     │ String response = solaceMessageReceiver        │                          │
│     │     .receiveMessage(correlationId);            │                          │
│     │                                                │                          │
│     │ // Internal: JMS Selector                      │                          │
│     │ // "JMSCorrelationID = '<correlationId>'"      │                          │
│     └────────────────────────────────────────────────┘                          │
│                          │                                                       │
│                          ▼                                                       │
│  4. Process Response                                                             │
│     ┌────────────────────────────────────────────────┐                          │
│     │ if (response != null) {                        │                          │
│     │     ResponseObj resp = mapper.readValue(       │                          │
│     │         response, ResponseObj.class);          │                          │
│     │ }                                              │                          │
│     └────────────────────────────────────────────────┘                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Configuration Files

| File | Contents |
|------|----------|
| `spring-context.xml` | JndiTemplate (profile-based), ConnectionFactory, CachingConnectionFactory, JndiDestinationResolver |
| `upi-request-adapter.xml` | Producer/Receiver JmsTemplate, SolaceMessageSender/Receiver beans, Queue lookups |
| `upi-proxy-transactions.xml` | Spring Integration flow for UPI proxy transactions |
| `tsp-request-enroll.xml` | Spring Integration flow for TSP enrollment |
| `tsp-request-cardsm.xml` | Spring Integration flow for TSP card management |
| `fx-hub-request.xml` | Spring Integration flow for FX queries |
| `common.xml` | Shared output channels, object-to-json transformer, outbound-channel-adapter |
| `SolaceConfiguration.java` | JndiTemplate bean (Production with SSL) |
| `LocalSolaceConfiguration.java` | JndiTemplate bean (Local with password) |
| `JmsConfiguration.java` | ConnectionFactory, CachingConnectionFactory, JmsTemplate, DefaultJmsListenerContainerFactory |
| `application.properties` | Solace broker URLs, credentials, queue/topic names, SSL settings |

---

## Properties Reference

### Connection Properties

```properties
solace.url=smfs://<broker-host>:<port>
solace.username=<username>
solace.vpn=<vpn-name>
solace.password=<password>  # Local only
solace.connection.factory=jndi/cf/<connection-factory-name>
```

### SSL Properties (Production)

```properties
Solace_JMS_SSL_ValidateCertificate=true
Solace_JMS_SSL_KeyStore=/path/to/keystore.jks
Solace_JMS_SSL_KeyStorePassword=ENC(<encrypted>)
Solace_JMS_Authentication_Scheme=AUTHENTICATION_SCHEME_CLIENT_CERTIFICATE
Solace_JMS_SSL_PrivateKeyAlias=<alias>
Solace_JMS_SSL_TrustStore=/path/to/truststore.jks
Solace_JMS_SSL_TrustStorePassword=ENC(<encrypted>)
Solace_JMS_SSL_PrivateKeyPassword=ENC(<encrypted>)
```

### Queue/Topic Properties

```properties
# UPI Proxy Transactions
solace.upi.proxy.transaction.request.queue.name=Q.CPS.00.P101.REQ.JSON.LOC.UPI.CPSADP.UPIPROXY

# Debit Request/Response
solace.upiproxy.request.issueAdapter.topic.name=P101/G/A/LOC/REQ/TRX/UPIADP00/PAY/UPI/NIL/V1/JSON/>
solace.upiproxy.debit.response.queue.name=Q.CPS.00.P101.RES.JSON.LOC.DEBITRESPONSE.CPSPRO.CPSADP

# Processor Communication
solace.upiproxy.processor.request.topic.name=P101/G/A/LOC/REQ/TRX/UPIADP00/PAY/UPI/NIL/V1/JSON/>
solace.upiproxy.processor.response.queue.name=Q.CPS.00.P101.RES.JSON.LOC.UPI.CPSADP.UPIPROXY
solace.upiproxy.processor.response.topic.name=P101/G/A/LOC/RES/TRX/UPIADP00/PAY/UPI/NIL/V1/JSON/>

# FX Hub
solace.fx.query.request.queue.name=Q.CPS.00.P101.REQ.JSON.SUB.FXHUB.FXENQRATE.UPI.CNY
solace.upiproxy.to.fx.topic.name=P101/G/A/SUB/RES/TRX/CPS00/FX/FXENQRATE/UPI/FXHUB00/NIL/V1/JSON/CNY/>

# TSP
solace.tsp.request.queue.name.enroll=Q.CPS.00.P101.REQ.JSON.SUB.TSP.ENROLL.UPIPROXY
solace.tsp.response.topic.name.enroll=P101/G/A/SUB/RES/TRX/UPIPROXY00/ENROLL/TSP/NIL/V1/JSON/>
solace.tsp.request.queue.name.cardsm=Q.CPS.00.P101.REQ.JSON.SUB.TSP.CARDSM.UPIPROXY
solace.tsp.response.topic.name.cardsm=P101/G/A/SUB/RES/TRX/UPIPROXY00/CARDSM/TSP/NIL/V1/JSON/>
```

### Consumer Properties

```properties
jms.concurrent.consumers.min=2
jms.concurrent.consumers.max=10
jms.concurrent.executor.queue.capacity=100
jms.receive.timeout=30000
jms.listener.gracefulshutdown.timeout.second=20

high.usage.concurrent.consumers=10
high.usage.max.concurrent.consumers=20
high.usage.idle.consumer.limit=5
high.usage.receive.timeout=5000
high.usage.idle.taskexecution.limit=10

low.usage.concurrent.consumers=2
low.usage.max.concurrent.consumers=5
```
