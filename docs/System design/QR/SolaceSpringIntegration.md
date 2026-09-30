# Solace Spring Integration

## Components

### 1. DefaultMessageListenerContainer

#### Purpose

Provides asynchronous JMS consumption with concurrent consumers for high-throughput message processing.

#### Dependencies

```java
ConnectionFactory (solaceConnectionFactory)
Destination (Queue from JndiObjectFactoryBean)
```

#### Configuration (XML Only)

**Actual Usage (upi-proxy-transactions.xml)**

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

**Actual Usage (fx-hub-request.xml)**

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

**Actual Usage (tsp-request-enroll.xml)**

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

**Actual Usage (tsp-request-cardsm.xml)**

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

#### Configuration (XML Only)

**Actual Usage (upi-proxy-transactions.xml)**

```xml
<int-jms:message-driven-channel-adapter
        id="ap.jmsIn"
        channel="ap.jmsInChannel"
        container="ap.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

**Actual Usage (fx-hub-request.xml)**

```xml
<int-jms:message-driven-channel-adapter
        id="fx.jmsIn"
        channel="fx.jmsInChannel"
        container="fx.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

**Actual Usage (tsp-request-enroll.xml)**

```xml
<int-jms:message-driven-channel-adapter
        id="tsp.jmsIn"
        channel="tsp.jmsInChannel"
        container="tsp.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

**Actual Usage (tsp-request-cardsm.xml)**

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
| channel | *jmsInChannel | Output Spring Integration channel |
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

#### Configuration (XML Only)

**Actual Usage (upi-proxy-transactions.xml)**

```xml
<int:logging-channel-adapter id="apLog" level="INFO" log-full-message="true" logger-name="apLog"/>

<int:channel id="ap.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="apLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="ap.inChannel"/>
```

**Actual Usage (fx-hub-request.xml)**

```xml
<int:logging-channel-adapter id="fxQueryLog" level="INFO" log-full-message="true" logger-name="fxQueryLog"/>

<int:channel id="fx.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="fxQueryLog"/>
    </int:interceptors>
</int:channel>
```

**Actual Usage (tsp-request-enroll.xml)**

```xml
<int:logging-channel-adapter id="tspLog" level="INFO" log-full-message="true" logger-name="tspLog"/>

<int:channel id="tsp.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="tspLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="tsp.inChannel"/>
```

**Actual Usage (tsp-request-cardsm.xml)**

```xml
<int:logging-channel-adapter id="tspcardsmLog" level="INFO" log-full-message="true" logger-name="tspcardsmLog"/>

<int:channel id="tspcardsm.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="tspcardsmLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="tspcardsm.inChannel"/>
```

**Output Channels (common.xml)**

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

#### Configuration (XML Only)

**Actual Usage (upi-proxy-transactions.xml)**

```xml
<int:json-to-object-transformer 
    input-channel="ap.jmsInChannel" 
    output-channel="ap.inChannel"
    type="com.nets.upi.domain.UpiProxyRequest">
</int:json-to-object-transformer>
```

**Actual Usage (tsp-request-enroll.xml)**

```xml
<int:json-to-object-transformer 
    input-channel="tsp.jmsInChannel" 
    output-channel="tsp.inChannel"
    type="com.nets.upi.domain.CardEnrollmentUPIRequestTsp">
</int:json-to-object-transformer>
```

**Actual Usage (tsp-request-cardsm.xml)**

```xml
<int:json-to-object-transformer 
    input-channel="tspcardsm.jmsInChannel" 
    output-channel="tspcardsm.inChannel"
    type="com.nets.upi.domain.LifecycleManagementUPIRequest">
</int:json-to-object-transformer>
```

> **Note:** FX Hub flow does NOT use json-to-object-transformer - processes raw String directly.

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

#### Configuration (XML Only)

**Actual Usage (upi-proxy-transactions.xml)**

```xml
<int:service-activator
        input-channel="ap.inChannel"
        output-channel="outputChannel"
        ref="transactionUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

**Actual Usage (fx-hub-request.xml)**

```xml
<int:service-activator
        input-channel="fx.jmsInChannel"
        output-channel="jsonJMSOutChannelFx"
        ref="fxQueryUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

**Actual Usage (tsp-request-enroll.xml)**

```xml
<int:service-activator
        input-channel="tsp.inChannel"
        output-channel="jsonJMSOutChannelCardEnroll"
        ref="tspUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

**Actual Usage (tsp-request-cardsm.xml)**

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

### 6. @ServiceActivator (Java Annotation)

#### Purpose

Marks a method as a Spring Integration service activator handler.

#### Dependencies

```java
@Service annotation
```

#### Implementation (Java Only)

**Actual Usage (TransactionUpiProxyProcessingService.java)**

```java
@Service
public class TransactionUpiProxyProcessingService {

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

**Actual Usage (FxQueryUpiProxyProcessingService.java)**

```java
@Service
public class FxQueryUpiProxyProcessingService {

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
        
        FxQueryUPIResponse fxResp = (FxQueryUPIResponse) UtillComponents
                .getObjectFromString(upiProxyResponse, FxQueryUPIResponse.class);
        // ... process response
        
        logger.info("FxQueryUpiProxyProcessingService process method end" + upiProxyResponse);
        return upiProxyResponse;
    }
}
```

**Actual Usage (TspUpiProxyProcessingService.java)**

```java
@Service
public class TspUpiProxyProcessingService {

    @Autowired
    private ExchangeService exchangeService;

    @Autowired
    private GetUPIRequestURI getUPIRequestURI;

    @Autowired
    private KeyInfoService keyInfoService;
    
    @Value("#{${nets.upi.appgateway.instId.map}}")
    private Map<String, String> instIdMap;
    
    @ServiceActivator
    public String process(CardEnrollmentUPIRequestTsp tspRequest) throws ParseException {
        logger.info("TspUpiProxyProcessingService process method start");
        
        String netsInsID = tspRequest.getMsgInfo().getInsID();
        String upiInsId = instIdMap.get(netsInsID) == null ? netsInsID : instIdMap.get(netsInsID);
        
        String upiRestUrl = getUPIRequestURI.getUpiRequestUrl(tspRequest.getMsgInfo().getMsgType());
        
        // Retrieve keys based on insID for different banks
        KeyInfo keyInfo = keyInfoService.findById(tspRequest.getMsgInfo().getInsID());
        
        // Encrypt sensitive data
        String publicStr = keyInfo.getUaisEncPublickey();
        if (tspRequest.getTrxInfo().getCvmInfo() != null) {
            String cvmInforStr = UtillComponents.getStringFromObject(tspRequest.getTrxInfo().getCvmInfo());
            String encryptCvmInfo = JweUtil.encryptJwe(cvmInforStr, publicStr, keyInfo.getUaisEncCertid());
            // ... set encrypted value
        }
        
        // Call UPI service and process response
        String tspResponse = exchangeService.getUPIResponse(tspRequestUpiStr, upiRestUrl, keyInfo.getInsCode());
        
        logger.info("TspUpiProxyProcessingService process method end" + tspResponse);
        return tspResponse;
    }
}
```

**Actual Usage (LifecycleManagementUpiProxyProcessingService.java)**

```java
@Service
public class LifecycleManagementUpiProxyProcessingService {

    @Autowired
    private ExchangeService exchangeService;

    @Autowired
    private GetUPIRequestURI getUPIRequestURI;
    
    @Value("#{${nets.upi.appgateway.instId.map}}")
    private Map<String, String> instIdMap;
    
    @ServiceActivator
    public String process(LifecycleManagementUPIRequest request) throws ParseException {
        logger.info("LifecycleManagementUpiProxyProcessingService process method start");

        String upiRestUrl = getUPIRequestURI.getUpiRequestUrl(request.getMsgInfo().getMsgType());

        String requestString = UtillComponents.getStringFromObject(request);
        String netsInsID = request.getMsgInfo().getInsID();
        String upiInsId = instIdMap.get(netsInsID) == null ? netsInsID : instIdMap.get(netsInsID);
        
        String tspResponse = exchangeService.getUPIResponse(requestString, upiRestUrl, request.getMsgInfo().getInsID());
        tspResponse = tspResponse.replaceAll(upiInsId, netsInsID);
        
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

#### Configuration (XML Only)

**Actual Usage (common.xml)**

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

#### Configuration (XML Only)

**Actual Usage - Dynamic Destination (common.xml)**

```xml
<int-jms:outbound-channel-adapter 
    id="jmsOut" 
    pub-sub-domain="true"
    destination-expression="headers['jms_replyTo']"
    channel="jmsOutChannel"
    connection-factory="solaceConnectionFactory">
</int-jms:outbound-channel-adapter>
```

**Actual Usage - Static Destination (fx-hub-request.xml)**

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

**Actual Usage - Static Destination (tsp-request-enroll.xml)**

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

**Actual Usage - Static Destination (tsp-request-cardsm.xml)**

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


## Flow Summary Table

| Flow | Config File | Input Type | Service | Output Destination |
|------|-------------|------------|---------|---------------------|
| UPI Proxy Transactions | upi-proxy-transactions.xml | `UpiProxyRequest` | TransactionUpiProxyProcessingService | `headers['jms_replyTo']` (dynamic) |
| FX Hub Query | fx-hub-request.xml | `String` (raw JSON) | FxQueryUpiProxyProcessingService | `${solace.upiproxy.to.fx.topic.name}` (static) |
| TSP Enroll | tsp-request-enroll.xml | `CardEnrollmentUPIRequestTsp` | TspUpiProxyProcessingService | `${solace.tsp.response.topic.name.enroll}` (static) |
| TSP Card Management | tsp-request-cardsm.xml | `LifecycleManagementUPIRequest` | LifecycleManagementUpiProxyProcessingService | `${solace.tsp.response.topic.name.cardsm}` (static) |

---

## Implementation Summary

| Component | Config Type | Config File | Implementation |
|-----------|-------------|-------------|----------------|
| DefaultMessageListenerContainer | XML | Various flow XML files | Spring Framework |
| message-driven-channel-adapter | XML | Various flow XML files | Spring Integration |
| Spring Integration Channels | XML | Various flow XML files | Spring Integration |
| json-to-object-transformer | XML | Various flow XML files | Spring Integration |
| service-activator | XML | Various flow XML files | Spring Integration |
| @ServiceActivator methods | Java | *ProcessingService.java | Local implementation |
| object-to-json-transformer | XML | common.xml | Spring Integration |
| outbound-channel-adapter | XML | Various flow XML files | Spring Integration |

---

## Configuration Files

| File | Contents |
|------|----------|
| `spring-context.xml` | Imports all flow XML files, ConnectionFactory beans |
| `upi-proxy-transactions.xml` | Spring Integration flow for UPI proxy transactions |
| `fx-hub-request.xml` | Spring Integration flow for FX queries |
| `tsp-request-enroll.xml` | Spring Integration flow for TSP enrollment |
| `tsp-request-cardsm.xml` | Spring Integration flow for TSP card management |
| `common.xml` | Shared output channels, object-to-json transformer, dynamic outbound-channel-adapter |

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
# High Usage (UPI Proxy, TSP Enroll, TSP Cardsm)
high.usage.concurrent.consumers=4
high.usage.max.concurrent.consumers=8
high.usage.idle.consumer.limit=4
high.usage.receive.timeout=5000
high.usage.idle.taskexecution.limit=20

# Low Usage (FX Hub)
low.usage.concurrent.consumers=2
low.usage.max.concurrent.consumers=4
low.usage.idle.consumer.limit=2
low.usage.receive.timeout=5000
low.usage.idle.taskexecution.limit=20
```

---

## Key Differences Between Flows

| Aspect | UPI Proxy Transactions | FX Hub | TSP Enroll | TSP Cardsm |
|--------|------------------------|--------|------------|------------|
| Input Transformation | json-to-object | None (raw String) | json-to-object | json-to-object |
| Input Type | `UpiProxyRequest` | `String` | `CardEnrollmentUPIRequestTsp` | `LifecycleManagementUPIRequest` |
| Output Transformation | object-to-json | None | None | None |
| Output Destination | Dynamic (`jms_replyTo`) | Static topic | Static topic | Static topic |
| Consumer Profile | High usage | Low usage | High usage | High usage |
| Connection Factory | solaceConnectionFactory | solaceCachedConnectionFactory | solaceCachedConnectionFactory | solaceCachedConnectionFactory |
