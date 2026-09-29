# Solace Message Broker

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/f41c9255-1a15-4761-95e9-1ffc171ae15a" />


## 1. JndiTemplate

### Purpose

Provides JNDI context for Solace resource lookup.

### XML Declaration

```xml
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
```

### Responsibilities

- Creates JNDI context with Solace broker connection details
- Provides lookup capability for Solace-managed resources
- Configures authentication (username/password or SSL certificates)

---

## 2. JndiObjectFactoryBean

### Purpose

Looks up ConnectionFactory, Queue, and Topic from Solace JNDI.

### XML Declaration

```xml
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.connection.factory}" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jndiTemplate | `solaceJndiTemplate` |

### Responsibilities

- Retrieves Solace-managed JMS objects via JNDI lookup
- Exposes them as Spring beans
- Used for ConnectionFactory, Queue, and Topic lookups

---

## 3. CachingConnectionFactory

### Purpose

Acts as the central JMS connection manager that caches connections and sessions.

### XML Declaration

```xml
<bean id="solaceCachedConnectionFactory" 
      class="org.springframework.jms.connection.CachingConnectionFactory">
    <property name="targetConnectionFactory" ref="solaceConnectionFactory" />
    <property name="sessionCacheSize" value="10" />
    <property name="cacheConsumers" value="false" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| targetConnectionFactory | `solaceConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| sessionCacheSize | 10 | Number of sessions to cache |
| cacheConsumers | false | Consumer caching disabled |

### Responsibilities

- Wraps Solace ConnectionFactory
- Caches JMS sessions for performance
- Reuses JMS connections

---

## 4. Producer JmsTemplate

### Purpose

Provides outbound JMS operations for sending messages.

### XML Declaration

```xml
<bean id="producerJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <constructor-arg ref="solaceCachedConnectionFactory" />
    <property name="pubSubDomain" value="true" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| connectionFactory | `solaceCachedConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| pubSubDomain | true | Uses Topic mode (publish-subscribe) |

### Functions Used by Developer

```java
jmsTemplate.send(destination, MessageCreator);
```

### Responsibilities

- Sends JMS messages to topics
- Creates and manages JMS sessions
- Provides send operations with MessageCreator callback

---

## 5. SolaceMessageSender

### Purpose

Application-level component that handles outbound messaging to Solace with correlation support.

### XML Declaration

```xml
<bean id="solaceMessageSender" class="com.upi.adaptor.nets.SolaceMessageSender">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
    <property name="destination" value="${solace.upiadapter.to.inbound.topic.name}" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jmsTemplate | `producerJmsTemplate` |

### Properties

| Property | Source | Description |
|----------|--------|-------------|
| destination | `${solace.upiadapter.to.inbound.topic.name}` | Outbound topic name |
| replyToTopic | `${solace.inbound.to.upiadapter.topic.name}` | Reply-to topic (@Value injection) |

### Functions Used

```java
// Only method available - destination is a class property, not a parameter
sendMessages(String msg, String correlationId);
```

### Responsibilities

- Sends outbound text messages
- Sets `JMSCorrelationID` for request-reply pattern
- Sets `JMSReplyTo` header for response routing

---

## 6. MessageCreator

### Purpose

Functional interface for creating JMS messages within a session context.

### Functions Used

```java
Message createMessage(Session session) throws JMSException;
```

### Usage Example

```java
jmsTemplate.send(destination, new MessageCreator() {
    public Message createMessage(Session session) throws JMSException {
        Message message = session.createTextMessage(msg);
        message.setJMSCorrelationID(correlationId);
        message.setJMSReplyTo(session.createTopic(replyToTopic));
        return message;
    }
});
```

### Responsibilities

- Creates TextMessage with payload
- Sets JMS headers (CorrelationID, ReplyTo)
- Sets custom properties if needed

---

## 7. Session

### Purpose

Provides JMS context for creating messages and destinations.

### Functions Used

```java
// Create messages
TextMessage message = session.createTextMessage(String text);

// Create destinations
Topic topic = session.createTopic(String topicName);
Queue queue = session.createQueue(String queueName);
```

### Responsibilities

- Creates JMS messages (TextMessage, BytesMessage, etc.)
- Creates dynamic destinations (Topic, Queue)
- Provides transactional context

---

## 8. Message (TextMessage)

### Purpose

Carries payload and JMS headers between sender and receiver.

### Functions Used

```java
// Setting headers (sender side)
message.setJMSCorrelationID(String correlationId);
message.setJMSReplyTo(Destination destination);

// Getting content (receiver side)
String payload = textMessage.getText();
String correlationId = message.getJMSCorrelationID();
Destination replyTo = message.getJMSReplyTo();
```

### JMS Headers

| Header | Purpose |
|--------|---------|
| JMSCorrelationID | Links request to response |
| JMSReplyTo | Destination for response |
| JMSMessageID | Unique message identifier |
| JMSTimestamp | Message send time |

### Responsibilities

- Stores message payload
- Stores Correlation ID for request-reply matching
- Stores Reply-To destination for response routing

---

## 9. Destination (Topic / Queue)

### Purpose

Represents a messaging endpoint (source or target).

### Functions Used

```java
// Create dynamically via Session
Topic topic = session.createTopic(String topicName);
Queue queue = session.createQueue(String queueName);

// Or lookup via JNDI (see JndiObjectFactoryBean)
```

### Types

| Type | Model | Use Case |
|------|-------|----------|
| Topic | Publish-Subscribe | One-to-many broadcasting |
| Queue | Point-to-Point | One-to-one messaging |

### Responsibilities

- Identifies message target for sending
- Identifies message source for receiving

---

## 10. Receiver JmsTemplate

### Purpose

Provides inbound JMS receive operations with selector support.

### XML Declaration

```xml
<bean id="financialMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="consumerQueue" />
    <property name="receiveTimeout" value="${financial.message.receive.timeout}" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| connectionFactory | `solaceCachedConnectionFactory` |
| defaultDestination | `consumerQueue` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| receiveTimeout | Configurable | Timeout for blocking receive |

### Functions Used

```java
Message message = jmsTemplate.receiveSelected(String selector);
```

### Responsibilities

- Receives response messages from queue
- Filters messages using JMS selectors
- Provides synchronous receive with timeout

---

## 11. SolaceMessageReceiver

### Purpose

Application-level component that receives response messages using correlation ID.

### XML Declaration

```xml
<bean id="solaceMessageReceiver" class="com.upi.adaptor.nets.SolaceMessageReceiver">
    <property name="jmsTemplate" ref="financialMessageReceiverJmsTemplate" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| jmsTemplate | `financialMessageReceiverJmsTemplate` |

### Functions Used

```java
// Public API
String response = solaceMessageReceiver.receiveMeassage(String correlationId);

// Internal - builds selector and calls JmsTemplate
String selector = "JMSCorrelationID = '" + correlationId + "'";
Message message = jmsTemplate.receiveSelected(selector);
```

### Responsibilities

- Receives messages filtered by correlation ID
- Extracts TextMessage payload
- Handles receive timeout

---

## 12. DefaultMessageListenerContainer

### Purpose

Provides asynchronous JMS consumption with concurrent consumers.

### XML Declaration

```xml
<bean id="jsonMessageListenerContainer" 
      class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="jsonInboundQueue" />
    <property name="connectionFactory" ref="solaceConnectionFactory" />
    <property name="concurrentConsumers" value="10" />
</bean>
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| destination | `jsonInboundQueue` |
| connectionFactory | `solaceConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| concurrentConsumers | 10 | Number of parallel consumer threads |

### Responsibilities

- Listens to inbound queue asynchronously
- Creates and manages JMS consumers
- Supports concurrent message processing

---

## 13. message-driven-channel-adapter

### Purpose

Bridges JMS messages into Spring Integration channels.

### XML Declaration

```xml
<int-jms:message-driven-channel-adapter 
    id="jsonJmsIn" 
    container="jsonMessageListenerContainer" 
    channel="jsonJmsInChannel" 
    extract-payload="true" />
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| container | `jsonMessageListenerContainer` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| channel | jsonJmsInChannel | Output Spring Integration channel |
| extract-payload | true | Extracts message body (not full Message object) |

### Responsibilities

- Converts incoming JMS messages to Spring Integration messages
- Extracts payload from TextMessage
- Routes to Spring Integration channel

---

## 14. JsonMsgServiceImpl

### Purpose

Service activator that processes JSON requests and generates responses.

### Java Declaration

```java
@Service
public class JsonMsgServiceImpl {
    
    @ServiceActivator  // <-- Marks this as the handler method
    public String processJsonMsg(String request) {
        // Business logic processing
        // Returns response string
    }
}
```

### XML Wiring

```xml
<int:service-activator 
    input-channel="jsonJmsInChannel" 
    output-channel="jsonJMSOutChannel" 
    ref="jsonMsgServiceImpl" />
```

### How @ServiceActivator Works

1. XML `ref="jsonMsgServiceImpl"` references the `@Service` bean
2. Spring Integration scans the bean for `@ServiceActivator` methods
3. `processJsonMsg(String request)` is invoked for each message on `jsonJmsInChannel`
4. Method parameter receives the extracted payload (String)
5. Method return value is sent to `jsonJMSOutChannel`

### Functions Used

```java
@ServiceActivator
public String processJsonMsg(String request);
```

### Responsibilities

- Receives message payload as method parameter
- Performs business logic processing
- Returns response string (automatically sent to output channel)

---

## 15. outbound-channel-adapter

### Purpose

Sends Spring Integration messages back to Solace JMS.

### XML Declaration

```xml
<int-jms:outbound-channel-adapter 
    id="jsonJmsOut"
    pub-sub-domain="true" 
    destination-expression="headers['jms_replyTo']"
    channel="jsonJMSOutChannel" 
    connection-factory="solaceCachedConnectionFactory" />
```

### Dependencies

| Dependency | Bean Reference |
|------------|----------------|
| connection-factory | `solaceCachedConnectionFactory` |

### Properties

| Property | Value | Description |
|----------|-------|-------------|
| channel | jsonJMSOutChannel | Input Spring Integration channel |
| pub-sub-domain | true | Uses Topic mode |
| destination-expression | `headers['jms_replyTo']` | Dynamic destination from message header |

### Responsibilities

- Publishes response messages to Solace
- Dynamically routes to `JMSReplyTo` destination from original request
- Converts Spring Integration message to JMS message

---

## Bean Dependencies

| Bean | Depends On |
|------|------------|
| `solaceJndiTemplate` | (none - root bean) |
| `solaceConnectionFactory` | `solaceJndiTemplate` |
| `solaceCachedConnectionFactory` | `solaceConnectionFactory` |
| `jsonInboundQueue` | `solaceJndiTemplate` |
| `consumerQueue` | `solaceJndiTemplate` |
| `producerJmsTemplate` | `solaceCachedConnectionFactory` |
| `financialMessageReceiverJmsTemplate` | `solaceCachedConnectionFactory`, `consumerQueue` |
| `solaceMessageSender` | `producerJmsTemplate` |
| `solaceMessageReceiver` | `financialMessageReceiverJmsTemplate` |
| `jsonMessageListenerContainer` | `jsonInboundQueue`, `solaceConnectionFactory` |
| `jsonJmsIn` | `jsonMessageListenerContainer` |
| `jsonMsgServiceImpl` | (none - @Service) |
| `jsonJmsOut` | `solaceCachedConnectionFactory` |

---

## Configuration Files

| File | Contents |
|------|----------|
| `spring-context.xml` | Connection factories, JmsTemplates, Sender/Receiver beans |
| `json-inbound.xml` | Spring Integration flow (channels, adapters, service-activator) |
| `application.properties` | Solace broker URLs, credentials, queue/topic names |
