# Solace JMS Messaging Components

## Overview

This document covers the JMS messaging components for synchronous request-reply pattern using JmsTemplate.

> **Note:** JMS messaging components are configured via **XML only**. Java classes are implementation classes that use the XML-configured beans.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         SOLACE JMS MESSAGING COMPONENTS                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           PRODUCER LAYER                                     │    │
│  │                                                                              │    │
│  │  ┌─────────────────────┐         ┌─────────────────────────────┐            │    │
│  │  │  Producer           │         │  SolaceMessageSender        │            │    │
│  │  │  JmsTemplate        │────────►│  (nps-qr-common library)    │            │    │
│  │  │  (pubSubDomain=true)│         │  - sendMessages(msg, corrId,│            │    │
│  │  └─────────────────────┘         │    replyTo, destination)    │            │    │
│  │           │                      └─────────────────────────────┘            │    │
│  │           │                                                                  │    │
│  │           ▼                      ┌─────────────────────────────┐            │    │
│  │  ┌─────────────────────┐         │  SolaceMessageSenderOneWay  │            │    │
│  │  │  MessageCreator     │────────►│  (Local implementation)     │            │    │
│  │  │  - createMessage()  │         │  - sendMessages(msg, dest)  │            │    │
│  │  │  - setCorrelationID │         │  - sendMessages(msg, dest,  │            │    │
│  │  │  - setReplyTo       │         │    correlationId)           │            │    │
│  │  └─────────────────────┘         └─────────────────────────────┘            │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                           CONSUMER LAYER                                     │    │
│  │                                                                              │    │
│  │  ┌─────────────────────┐         ┌─────────────────────────────┐            │    │
│  │  │  Receiver           │         │  SolaceMessageReceiver      │            │    │
│  │  │  JmsTemplate        │────────►│  (nps-qr-common library)    │            │    │
│  │  │  (defaultDestination│         │  - receiveMessage(corrId)   │            │    │
│  │  │   receiveTimeout)   │         └─────────────────────────────┘            │    │
│  │  └─────────────────────┘                                                     │    │
│  │           │                      ┌─────────────────────────────┐            │    │
│  │           │                      │  SolaceReversalMessage      │            │    │
│  │           ▼                      │  Receiver                   │            │    │
│  │  ┌─────────────────────┐         │  (Local implementation)     │            │    │
│  │  │  receiveSelected()  │────────►│  - receiveMessage(corrId)   │            │    │
│  │  │  JMS Selector:      │         │  - JMS Selector filtering   │            │    │
│  │  │  "JMSCorrelationID  │         └─────────────────────────────┘            │    │
│  │  │   = '<corrId>'"     │                                                     │    │
│  │  └─────────────────────┘                                                     │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Producer JmsTemplate

#### Purpose

Provides outbound JMS operations for sending messages to topics.

#### Dependencies

```java
CachingConnectionFactory
```

#### Configuration (XML Only)

**Actual Usage (upi-request-adapter.xml)**

```xml
<bean id="producerJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <constructor-arg ref="solaceCachedConnectionFactory" />
    <property name="pubSubDomain" value="true" />
</bean>
```

#### Functions Used

```java
jmsTemplate.send(String destination, MessageCreator messageCreator);
```

#### Responsibilities

- Sends JMS messages to topics
- Creates and manages JMS sessions internally
- Provides send operations with MessageCreator callback

---

### 2. Receiver JmsTemplate

#### Purpose

Provides inbound JMS receive operations with selector support.

#### Dependencies

```java
CachingConnectionFactory
Queue (from JndiObjectFactoryBean)
```

#### Configuration (XML Only)

**Actual Usage (upi-request-adapter.xml)**

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

#### Functions Used

```java
Message message = jmsTemplate.receiveSelected(String messageSelector);
```

#### Responsibilities

- Receives response messages from queue
- Filters messages using JMS selectors (correlation ID)
- Provides synchronous receive with configurable timeout

---

### 3. MessageCreator

#### Purpose

Functional interface for creating JMS messages within a session context.

#### Dependencies

```java
Session (provided by JmsTemplate)
```

#### Functions Used

```java
Message createMessage(Session session) throws JMSException;
```

#### Actual Usage (SolaceMessageSenderOneWay.java)

```java
getJmsTemplate().send(destination, new MessageCreator() {
    public Message createMessage(Session session) throws JMSException {
        Message message = session.createTextMessage(msg);
        message.setJMSCorrelationID(correlationId);
        return message;
    }
});
```

#### Responsibilities

- Creates TextMessage with payload
- Sets JMS headers (CorrelationID, ReplyTo)

---

### 4. SolaceMessageSenderOneWay (Local Implementation)

#### Purpose

Handles outbound messaging with optional correlation support.

#### Dependencies

```java
JmsTemplate (producerJmsTemplate)
```

#### Configuration (XML + Java Implementation)

**Bean Declaration (upi-request-adapter.xml)**

```xml
<bean id="upiDebitMessageProducer"
    class="com.nets.upi.processing.integration.upi.proxy.service.impl.SolaceMessageSenderOneWay">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>
```

**Java Implementation (SolaceMessageSenderOneWay.java)**

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

#### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

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

#### Responsibilities

- Sends outbound text messages to specified topic
- Optionally sets `JMSCorrelationID` for request-reply pattern

---

### 5. SolaceMessageSender (External Library)

#### Purpose

Handles outbound messaging with reply-to topic support (from `nps-qr-common` library).

#### Dependencies

```java
JmsTemplate (producerJmsTemplate)
```

#### Configuration (XML Only)

**Actual Usage (upi-request-adapter.xml)**

```xml
<bean id="upiAdapterReqMessageProducer"
    class="com.nets.nps.qr.common.solace.SolaceMessageSender">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>
```

#### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

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

#### Actual Usage (ProcessorCommunicationHandler.java)

```java
@Qualifier("upiAdapterReqMessageProducer")
@Autowired
private SolaceMessageSender solaceMessageSender;

public UpiProxyRequest sendAndReceiveResponse(UpiProxyRequest upiProxyRequest, String correlationId) 
        throws JsonProcessingException {
    ObjectMapper mapper = new ObjectMapper();
    solaceMessageSender.sendMessages(
        mapper.writeValueAsString(upiProxyRequest), 
        correlationId, 
        processorReplyToTopic, 
        processorResultDestination
    );
    // ... receive response
}
```

#### Responsibilities

- Sends outbound text messages
- Sets `JMSCorrelationID` for request-reply matching
- Sets `JMSReplyTo` header for response routing

---

### 6. SolaceMessageReceiver (External Library)

#### Purpose

Receives response messages using correlation ID filtering (from `nps-qr-common` library).

#### Dependencies

```java
JmsTemplate (Receiver JmsTemplate)
```

#### Configuration (XML Only)

**Actual Usage (upi-request-adapter.xml)**

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

#### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

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

#### Responsibilities

- Receives messages filtered by correlation ID using JMS selector
- Extracts TextMessage payload
- Returns null if no message received within timeout

---

### 7. SolaceReversalMessageReceiver (Local Implementation)

#### Purpose

Receives reversal response messages with correlation ID filtering.

#### Dependencies

```java
JmsTemplate (reversalMessageReceiverJmsTemplate)
```

#### Configuration (XML + Java Implementation)

**Bean Declaration (upi-request-adapter.xml)**

```xml
<bean id="solaceReversalMessageReceiver"
    class="com.nets.upi.processing.integration.upi.proxy.service.impl.SolaceReversalMessageReceiver">
    <property name="jmsTemplate" ref="reversalMessageReceiverJmsTemplate" />
</bean>
```

**Java Implementation (SolaceReversalMessageReceiver.java)**

```java
@Component
public class SolaceReversalMessageReceiver {

    private static final ApsLogger logger = new ApsLogger(SolaceReversalMessageReceiver.class);
    
    protected JmsTemplate jmsTemplateReversal;

    public String receiveMessage(String correlationId) {
        logger.info("SolaceReversalMessageReceiver trying to receive message correlation id:" + correlationId);
        String resCorrelationId = "JMSCorrelationID = '" + correlationId + "'";
        Message message = getJmsTemplate().receiveSelected(resCorrelationId);
        if (message instanceof TextMessage) {
            TextMessage txtMsg = (TextMessage) message;
            try {
                Object msgTextObj = txtMsg.getText();
                return msgTextObj.toString();
            } catch (JMSException e) {
                logger.error("SolaceReversalMessageReceiver Receiving message Error");
                e.printStackTrace();
            }
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

#### Actual Usage (UpiDebitIssuerAdapterCommunicationHandler.java)

```java
public String sendToReversalAndGetResponse(String reversalRequestStr, String correlationId)
        throws IOException {
    solaceMessageSender.sendMessages(reversalRequestStr, correlationId,
            reversalReplyToTopic, reversalDestination);
    String response = solaceReversalMessageReceiver.receiveMessage(correlationId);
    return response;
}

public String sendToRefundAndGetResponse(String reversalRequestStr, String correlationId)
        throws IOException {
    solaceMessageSender.sendMessages(reversalRequestStr, correlationId,
            reversalReplyToTopic, reversalInternalDestination);
    String response = solaceReversalMessageReceiver.receiveMessage(correlationId);
    return response;
}
```

#### Responsibilities

- Receives messages filtered by correlation ID
- Builds JMS selector string: `JMSCorrelationID = '<correlationId>'`
- Extracts text content from TextMessage

---

### 8. Message (TextMessage)

#### Purpose

Carries payload and JMS headers between sender and receiver.

#### JMS Headers

| Header | Purpose |
|--------|---------|
| JMSCorrelationID | Links request to response |
| JMSReplyTo | Destination for response |
| JMSMessageID | Unique message identifier |
| JMSTimestamp | Message send time |

#### Actual Usage - Setting Headers (Sender)

```java
public Message createMessage(Session session) throws JMSException {
    Message message = session.createTextMessage(msg);
    message.setJMSCorrelationID(correlationId);
    message.setJMSReplyTo(session.createTopic(replyToTopic));
    return message;
}
```

#### Actual Usage - Getting Content (Receiver)

```java
if (message instanceof TextMessage) {
    TextMessage txtMsg = (TextMessage) message;
    String payload = txtMsg.getText();
}
```

---

### 9. Session

#### Purpose

Provides JMS context for creating messages and destinations.

#### Functions Used

```java
TextMessage message = session.createTextMessage(String text);
Topic topic = session.createTopic(String topicName);
Queue queue = session.createQueue(String queueName);
```

#### Actual Usage

```java
getJmsTemplate().send(destination, new MessageCreator() {
    public Message createMessage(Session session) throws JMSException {
        Message message = session.createTextMessage(msg);
        message.setJMSCorrelationID(correlationId);
        return message;
    }
});
```

---

## Message Flow Pattern: Synchronous Request-Reply

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

## Bean Dependencies

```
┌─────────────────────────────────────┐
│      CachingConnectionFactory       │
│   (solaceCachedConnectionFactory)   │
└───────────────┬─────────────────────┘
                │
    ┌───────────┴───────────┐
    ▼                       ▼
┌─────────────────┐   ┌─────────────────────────────┐
│ Producer        │   │ Receiver JmsTemplate(s)     │
│ JmsTemplate     │   │ - upiAdapterReqMessage...   │
│                 │   │ - reversalMessage...        │
└────────┬────────┘   │ - processorMessage...       │
         │            └──────────────┬──────────────┘
         │                           │
    ┌────┴────┐              ┌───────┴───────┐
    ▼         ▼              ▼               ▼
┌────────┐ ┌────────┐   ┌────────────┐  ┌────────────┐
│Solace  │ │Solace  │   │Solace      │  │Solace      │
│Message │ │Message │   │Message     │  │Reversal    │
│Sender  │ │Sender  │   │Receiver    │  │Message     │
│        │ │OneWay  │   │(external)  │  │Receiver    │
└────────┘ └────────┘   └────────────┘  └────────────┘
```

---

## Implementation Summary

| Component | Config Type | Config File | Implementation |
|-----------|-------------|-------------|----------------|
| Producer JmsTemplate | XML | upi-request-adapter.xml | Spring Framework |
| Receiver JmsTemplate(s) | XML | upi-request-adapter.xml | Spring Framework |
| SolaceMessageSender | XML | upi-request-adapter.xml | nps-qr-common library |
| SolaceMessageSenderOneWay | XML + Java | upi-request-adapter.xml | Local implementation |
| SolaceMessageReceiver | XML | upi-request-adapter.xml | nps-qr-common library |
| SolaceReversalMessageReceiver | XML + Java | upi-request-adapter.xml | Local implementation |

---

## Configuration Files

| File | Contents |
|------|----------|
| `upi-request-adapter.xml` | Producer/Receiver JmsTemplate beans, SolaceMessageSender/Receiver beans |

---

## Properties Reference

### Queue/Topic Properties

```properties
# Debit Request/Response
solace.upiproxy.request.issueAdapter.topic.name=P101/G/A/LOC/REQ/TRX/UPIADP00/PAY/UPI/NIL/V1/JSON/>
solace.upiproxy.mpqrc.request.issueAdapter.topic.name=P101/G/A/LOC/REQ/TRX/UPIADP00/PAY/DEBITREQUEST/NIL/V1/JSON/>
solace.upiproxy.debit.response.queue.name=Q.CPS.00.P101.RES.JSON.LOC.DEBITRESPONSE.CPSPRO.CPSADP

# Processor Communication
solace.upiproxy.processor.request.topic.name=P101/G/A/LOC/REQ/TRX/UPIADP00/PAY/UPI/NIL/V1/JSON/>
solace.upiproxy.processor.response.queue.name=Q.CPS.00.P101.RES.JSON.LOC.UPI.CPSADP.UPIPROXY
solace.upiproxy.processor.response.topic.name=P101/G/A/LOC/RES/TRX/UPIADP00/PAY/UPI/NIL/V1/JSON/>

# Reversal Communication
solace.upiproxy.request.reversal.topic.name=P101/G/A/LOC/REQ/TRX/CPSINB00/PAY/INTERNALREVERSAL/NIL/V1/JSON/>
solace.upiproxy.adapter.request.reversal.topic.name=P101/G/A/LOC/REQ/TRX/CPSPRO00/PAY/PAYLAH/NIL/V1/JSON/>
solace.upiproxy.response.reversal.topic.name=P101/G/A/LOC/RES/TRX/CPSADP00/PAY/DEBITRESPONSE/NIL/V1/JSON/>
```

### Timeout Properties

```properties
upi.message.receive.timeout=30000
```
