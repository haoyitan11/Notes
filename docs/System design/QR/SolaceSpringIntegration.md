# Solace Spring Integration Components

## Overview

This document covers the Spring Integration components for asynchronous message processing flows.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      SOLACE SPRING INTEGRATION COMPONENTS                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         INBOUND LAYER                                        │    │
│  │                                                                              │    │
│  │  Solace Queue ──► DefaultMessageListenerContainer ──► message-driven-       │    │
│  │                   (Concurrent Consumers)              channel-adapter        │    │
│  │                                                       (extract-payload=true) │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         CHANNEL LAYER                                        │    │
│  │                                                                              │    │
│  │  jmsInChannel ──► json-to-object-transformer ──► inChannel                   │    │
│  │  (wire-tap)       (JSON → Domain Object)                                     │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         PROCESSING LAYER                                     │    │
│  │                                                                              │    │
│  │  service-activator ──► @ServiceActivator method ──► Business Logic          │    │
│  │  (ProcessingService)   process(Request)              (REST API calls)        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         OUTBOUND LAYER                                       │    │
│  │                                                                              │    │
│  │  outputChannel ──► object-to-json-transformer ──► outbound-channel-adapter  │    │
│  │                    (Domain Object → JSON)         (destination-expression)   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│                                          ▼                                           │
│                                   Solace Topic                                       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. DefaultMessageListenerContainer

#### Purpose

Provides asynchronous JMS consumption with concurrent consumers for high-throughput message processing.

#### Dependencies

```java
ConnectionFactory (solaceConnectionFactory)
Destination (Queue from JndiObjectFactoryBean)
```

#### XML Declaration (upi-proxy-transactions.xml)

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

#### Additional Listener Containers

**FX Hub (fx-hub-request.xml):**
```xml
<bean id="fx.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.fx.query.request.queue.name}"/>
</bean>

<bean id="fx.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="fx.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    <property name="concurrentConsumers" value="${low.usage.concurrent.consumers}"/>
    <property name="maxConcurrentConsumers" value="${low.usage.max.concurrent.consumers}"/>
    <property name="idleConsumerLimit" value="${low.usage.idle.consumer.limit}"/>
    <property name="receiveTimeout" value="${low.usage.receive.timeout}"/>
    <property name="idleTaskExecutionLimit" value="${low.usage.idle.taskexecution.limit}"/>
</bean>
```

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<bean id="tsp.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.tsp.request.queue.name.enroll}"/>
</bean>

<bean id="tsp.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="tsp.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    <property name="concurrentConsumers" value="${high.usage.concurrent.consumers}"/>
    <property name="maxConcurrentConsumers" value="${high.usage.max.concurrent.consumers}"/>
    <property name="idleConsumerLimit" value="${high.usage.idle.consumer.limit}"/>
    <property name="receiveTimeout" value="${high.usage.receive.timeout}"/>
    <property name="idleTaskExecutionLimit" value="${high.usage.idle.taskexecution.limit}"/>
</bean>
```

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<bean id="tspcardsm.processingRequestQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.tsp.request.queue.name.cardsm}"/>
</bean>

<bean id="tspcardsm.messageListenerContainer"
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="tspcardsm.processingRequestQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory"/>
    <property name="concurrentConsumers" value="${high.usage.concurrent.consumers}"/>
    <property name="maxConcurrentConsumers" value="${high.usage.max.concurrent.consumers}"/>
    <property name="idleConsumerLimit" value="${high.usage.idle.consumer.limit}"/>
    <property name="receiveTimeout" value="${high.usage.receive.timeout}"/>
    <property name="idleTaskExecutionLimit" value="${high.usage.idle.taskexecution.limit}"/>
</bean>
```

#### Java Configuration (JmsConfiguration.java)

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

#### Properties

| Property | Source | Description |
|----------|--------|-------------|
| concurrentConsumers | `${high.usage.concurrent.consumers}` | Initial consumer threads |
| maxConcurrentConsumers | `${high.usage.max.concurrent.consumers}` | Maximum consumer threads |
| idleConsumerLimit | `${high.usage.idle.consumer.limit}` | Idle consumers before scaling down |
| receiveTimeout | `${high.usage.receive.timeout}` | Receive operation timeout |
| idleTaskExecutionLimit | `${high.usage.idle.taskexecution.limit}` | Idle task limit |

#### Responsibilities

- Listens to inbound queue asynchronously
- Creates and manages JMS consumers
- Scales consumers based on load (min to max)
- Provides graceful shutdown support

---

### 2. message-driven-channel-adapter

#### Purpose

Bridges JMS messages into Spring Integration channels for processing.

#### Dependencies

```java
DefaultMessageListenerContainer
```

#### XML Declaration (upi-proxy-transactions.xml)

```xml
<int-jms:message-driven-channel-adapter
        id="ap.jmsIn"
        channel="ap.jmsInChannel"
        container="ap.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

#### Additional Adapters

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

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<int-jms:message-driven-channel-adapter
        id="tspcardsm.jmsIn"
        channel="tspcardsm.jmsInChannel"
        container="tspcardsm.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

#### Properties

| Property | Value | Description |
|----------|-------|-------------|
| channel | jmsInChannel | Output Spring Integration channel |
| extract-payload | true | Extracts message body (not full Message object) |
| error-channel | errorChannel | Channel for error handling |

#### Responsibilities

- Converts incoming JMS messages to Spring Integration messages
- Extracts payload from TextMessage
- Routes to Spring Integration channel for processing
- Handles errors via error channel

---

### 3. Spring Integration Channels

#### Purpose

Provides message routing infrastructure between adapters and service activators.

#### Dependencies

```
None (Infrastructure)
```

#### XML Declaration (upi-proxy-transactions.xml)

```xml
<int:logging-channel-adapter id="apLog" level="INFO" log-full-message="true" logger-name="apLog"/>

<int:channel id="ap.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="apLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="ap.inChannel"/>
```

#### Additional Channels

**FX Hub (fx-hub-request.xml):**
```xml
<int:logging-channel-adapter id="fxQueryLog" level="INFO" log-full-message="true" logger-name="fxQueryLog"/>

<int:channel id="fx.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="fxQueryLog"/>
    </int:interceptors>
</int:channel>
```

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<int:logging-channel-adapter id="tspLog" level="INFO" log-full-message="true" logger-name="tspLog"/>

<int:channel id="tsp.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="tspLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="tsp.inChannel"/>
```

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<int:logging-channel-adapter id="tspcardsmLog" level="INFO" log-full-message="true" logger-name="tspcardsmLog"/>

<int:channel id="tspcardsm.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="tspcardsmLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="tspcardsm.inChannel"/>
```

#### Output Channels (common.xml)

```xml
<int:channel id="outputChannel" />

<int:object-to-json-transformer input-channel="outputChannel" output-channel="jmsOutChannel" />

<int:channel id="jmsOutChannel" />
```

#### Responsibilities

- Routes messages between components
- Provides wire-tap for logging
- Supports transformation pipeline
- Enables decoupled message processing

---

### 4. json-to-object-transformer

#### Purpose

Transforms JSON string messages to Java objects for service processing.

#### Dependencies

```
Input Channel
Output Channel
```

#### XML Declaration (upi-proxy-transactions.xml)

```xml
<int:json-to-object-transformer 
    input-channel="ap.jmsInChannel" 
    output-channel="ap.inChannel"
    type="com.nets.upi.domain.UpiProxyRequest">
</int:json-to-object-transformer>
```

#### Additional Transformers

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

**Note:** FX Hub flow does NOT use json-to-object-transformer - processes raw String directly.

#### Responsibilities

- Deserializes JSON string to specified Java type
- Enables type-safe message processing
- Supports domain object transformation

---

### 5. service-activator

#### Purpose

Connects Spring Integration channels to service methods for business logic processing.

#### Dependencies

```java
@Service bean
Input Channel
Output Channel
```

#### XML Declaration (upi-proxy-transactions.xml)

```xml
<int:service-activator
        input-channel="ap.inChannel"
        output-channel="outputChannel"
        ref="transactionUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

#### Additional Service Activators

**FX Hub (fx-hub-request.xml):**
```xml
<int:service-activator
        input-channel="fx.jmsInChannel"
        output-channel="jsonJMSOutChannelFx"
        ref="fxQueryUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

**TSP Enroll (tsp-request-enroll.xml):**
```xml
<int:service-activator
        input-channel="tsp.inChannel"
        output-channel="jsonJMSOutChannelCardEnroll"
        ref="tspUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<int:service-activator
        input-channel="tspcardsm.inChannel"
        output-channel="jsonJMSOutChannelCardsm"
        ref="lifecycleManagementUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

#### Responsibilities

- Invokes @ServiceActivator annotated method
- Passes message payload as method parameter
- Routes method return value to output channel

---

### 6. @ServiceActivator (Annotation)

#### Purpose

Marks a method as a Spring Integration service activator handler.

#### Dependencies

```java
@Service annotation
```

#### Java Implementation (TransactionUpiProxyProcessingService.java)

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

#### Additional Service Implementations

**FxQueryUpiProxyProcessingService.java:**
```java
@Service
public class FxQueryUpiProxyProcessingService {

    private Logger logger = LoggerFactory.getLogger(FxQueryUpiProxyProcessingService.class);

    @Autowired
    private ExchangeService exchangeService;

    @Autowired
    private GetUPIRequestURI getUPIRequestURI;
    
    @Value("#{${nets.upi.appgateway.instId.map}}")
    private Map<String, String> instIdMap;
    
    @ServiceActivator
    public String process(String fxQueryRequest) throws ParseException {
        logger.info("FxQueryUpiProxyProcessingService process method start:" + fxQueryRequest);
        FxQueryUPIRequest fxReq = (FxQueryUPIRequest) UtillComponents
                .getObjectFromString(fxQueryRequest, FxQueryUPIRequest.class);
        String netsInsID = fxReq.getMsgInfo().getInsID();

        String upiRestUrl = getUPIRequestURI.getUpiRequestUrl("EXCHANGE_RATE_INQUIRY");

        String upiProxyResponse = exchangeService.getUPIResponse(fxQueryRequest, upiRestUrl, netsInsID);
        // ... process response
        logger.info("FxQueryUpiProxyProcessingService process method end" + upiProxyResponse);
        return upiProxyResponse;
    }
}
```

**TspUpiProxyProcessingService.java:**
```java
@Service
public class TspUpiProxyProcessingService {

    private Logger logger = LoggerFactory.getLogger(TspUpiProxyProcessingService.class);

    @Autowired
    private ExchangeService exchangeService;

    @Autowired
    private GetUPIRequestURI getUPIRequestURI;

    @Autowired
    private KeyInfoService keyInfoService;
    
    @ServiceActivator
    public String process(CardEnrollmentUPIRequestTsp tspRequest) throws ParseException {
        logger.info("TspUpiProxyProcessingService process method start");
        // ... encryption and REST call logic
        logger.info("TspUpiProxyProcessingService process method end" + tspResponse);
        return tspResponse;
    }
}
```

**LifecycleManagementUpiProxyProcessingService.java:**
```java
@Service
public class LifecycleManagementUpiProxyProcessingService {

    private Logger logger = LoggerFactory.getLogger(LifecycleManagementUpiProxyProcessingService.class);

    @Autowired
    private ExchangeService exchangeService;

    @Autowired
    private GetUPIRequestURI getUPIRequestURI;
    
    @ServiceActivator
    public String process(LifecycleManagementUPIRequest request) throws ParseException {
        logger.info("LifecycleManagementUpiProxyProcessingService process method start");
        String upiRestUrl = getUPIRequestURI.getUpiRequestUrl(request.getMsgInfo().getMsgType());
        String requestString = UtillComponents.getStringFromObject(request);
        String tspResponse = exchangeService.getUPIResponse(requestString, upiRestUrl, request.getMsgInfo().getInsID());
        logger.info("LifecycleManagementUpiProxyProcessingService process method end" + tspResponse);
        return tspResponse;
    }
}
```

#### How @ServiceActivator Works

1. XML `ref="transactionUpiProxyProcessingService"` references the `@Service` bean
2. Spring Integration scans the bean for `@ServiceActivator` methods
3. Method parameter receives the transformed payload
4. Method return value is sent to output channel

#### Responsibilities

- Receives message payload as method parameter
- Performs business logic processing (REST calls to external systems)
- Returns response object (automatically sent to output channel)

---

### 7. object-to-json-transformer

#### Purpose

Transforms Java objects back to JSON string for JMS output.

#### Dependencies

```
Input Channel
Output Channel
```

#### XML Declaration (common.xml)

```xml
<int:object-to-json-transformer 
    input-channel="outputChannel" 
    output-channel="jmsOutChannel" />
```

#### Responsibilities

- Serializes Java objects to JSON string
- Prepares response for JMS transmission
- Supports domain object serialization

---

### 8. outbound-channel-adapter

#### Purpose

Sends Spring Integration messages back to Solace JMS topics.

#### Dependencies

```java
ConnectionFactory
Input Channel
```

#### XML Declaration - Dynamic Destination (common.xml)

```xml
<int-jms:outbound-channel-adapter 
    id="jmsOut" 
    pub-sub-domain="true"
    destination-expression="headers['jms_replyTo']"
    channel="jmsOutChannel"
    connection-factory="solaceConnectionFactory">
</int-jms:outbound-channel-adapter>
```

#### XML Declaration - Static Destinations

**TSP Cardsm (tsp-request-cardsm.xml):**
```xml
<int:channel id="jsonJMSOutChannelCardsm" />

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
<int:channel id="jsonJMSOutChannelCardEnroll" />

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
<int:channel id="jsonJMSOutChannelFx" />

<int-jms:outbound-channel-adapter 
    id="outputChannelFx"
    pub-sub-domain="true" 
    destination-expression="'${solace.upiproxy.to.fx.topic.name}'"
    channel="jsonJMSOutChannelFx" 
    connection-factory="solaceCachedConnectionFactory">
</int-jms:outbound-channel-adapter>
```

#### Properties

| Property | Value | Description |
|----------|-------|-------------|
| channel | jmsOutChannel | Input Spring Integration channel |
| pub-sub-domain | true | Uses Topic mode |
| destination-expression | `headers['jms_replyTo']` or static topic | Dynamic/static destination |
| connection-factory | `solaceConnectionFactory` | Connection factory reference |

#### Responsibilities

- Publishes response messages to Solace topics
- Dynamically routes to `JMSReplyTo` destination from original request
- Supports static destination configuration
- Converts Spring Integration message to JMS message

---

## Message Flow Pattern: Asynchronous Request Processing

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

---

## Flow Summary Table

| Flow | Config File | Queue Property | Service | Output |
|------|-------------|----------------|---------|--------|
| UPI Proxy Transactions | upi-proxy-transactions.xml | `solace.upi.proxy.transaction.request.queue.name` | TransactionUpiProxyProcessingService | `headers['jms_replyTo']` |
| FX Hub Query | fx-hub-request.xml | `solace.fx.query.request.queue.name` | FxQueryUpiProxyProcessingService | `${solace.upiproxy.to.fx.topic.name}` |
| TSP Enroll | tsp-request-enroll.xml | `solace.tsp.request.queue.name.enroll` | TspUpiProxyProcessingService | `${solace.tsp.response.topic.name.enroll}` |
| TSP Card Management | tsp-request-cardsm.xml | `solace.tsp.request.queue.name.cardsm` | LifecycleManagementUpiProxyProcessingService | `${solace.tsp.response.topic.name.cardsm}` |

---

## Bean Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                    ConnectionFactory                             │
│                  (solaceConnectionFactory)                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
┌───────────────────────────┐   ┌─────────────────────────────────┐
│ DefaultMessageListener    │   │ outbound-channel-adapter        │
│ Container(s)              │   │ (jmsOut, outputChannel*)        │
│ - ap.messageListener...   │   └─────────────────────────────────┘
│ - fx.messageListener...   │
│ - tsp.messageListener...  │
│ - tspcardsm.message...    │
└───────────────┬───────────┘
                │
                ▼
┌───────────────────────────┐
│ message-driven-channel-   │
│ adapter(s)                │
│ - ap.jmsIn                │
│ - fx.jmsIn                │
│ - tsp.jmsIn               │
│ - tspcardsm.jmsIn         │
└───────────────┬───────────┘
                │
                ▼
┌───────────────────────────┐
│ Spring Integration        │
│ Channels & Transformers   │
└───────────────┬───────────┘
                │
                ▼
┌───────────────────────────┐
│ @Service beans            │
│ (@ServiceActivator)       │
└───────────────────────────┘
```

---

## Configuration Files

| File | Contents |
|------|----------|
| `upi-proxy-transactions.xml` | Spring Integration flow for UPI proxy transactions |
| `fx-hub-request.xml` | Spring Integration flow for FX queries |
| `tsp-request-enroll.xml` | Spring Integration flow for TSP enrollment |
| `tsp-request-cardsm.xml` | Spring Integration flow for TSP card management |
| `common.xml` | Shared output channels, object-to-json transformer, outbound-channel-adapter |
| `JmsConfiguration.java` | DefaultJmsListenerContainerFactory, TaskExecutor |

---

## Properties Reference

### Queue Properties

```properties
# UPI Proxy Transactions
solace.upi.proxy.transaction.request.queue.name=Q.CPS.00.P101.REQ.JSON.LOC.UPI.CPSADP.UPIPROXY

# FX Hub
solace.fx.query.request.queue.name=Q.CPS.00.P101.REQ.JSON.SUB.FXHUB.FXENQRATE.UPI.CNY

# TSP
solace.tsp.request.queue.name.enroll=Q.CPS.00.P101.REQ.JSON.SUB.TSP.ENROLL.UPIPROXY
solace.tsp.request.queue.name.cardsm=Q.CPS.00.P101.REQ.JSON.SUB.TSP.CARDSM.UPIPROXY
```

### Response Topic Properties

```properties
# FX Hub
solace.upiproxy.to.fx.topic.name=P101/G/A/SUB/RES/TRX/CPS00/FX/FXENQRATE/UPI/FXHUB00/NIL/V1/JSON/CNY/>

# TSP
solace.tsp.response.topic.name.enroll=P101/G/A/SUB/RES/TRX/UPIPROXY00/ENROLL/TSP/NIL/V1/JSON/>
solace.tsp.response.topic.name.cardsm=P101/G/A/SUB/RES/TRX/UPIPROXY00/CARDSM/TSP/NIL/V1/JSON/>
```

### Consumer Properties

```properties
# High Usage (UPI Proxy, TSP)
high.usage.concurrent.consumers=10
high.usage.max.concurrent.consumers=20
high.usage.idle.consumer.limit=5
high.usage.receive.timeout=5000
high.usage.idle.taskexecution.limit=10

# Low Usage (FX Hub)
low.usage.concurrent.consumers=2
low.usage.max.concurrent.consumers=5
low.usage.idle.consumer.limit=2
low.usage.receive.timeout=5000
low.usage.idle.taskexecution.limit=5

# JMS Listener Factory
jms.concurrent.consumers.min=2
jms.concurrent.consumers.max=10
jms.concurrent.executor.queue.capacity=100
jms.receive.timeout=30000
jms.listener.gracefulshutdown.timeout.second=20
```
