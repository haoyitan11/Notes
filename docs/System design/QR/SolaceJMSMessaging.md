# Solace JMS Messaging Components
## Components

### 1. SolaceMessageSender

#### Purpose

Handles outbound message sending to Solace message broker with support for request-reply patterns.

#### Dependencies

```java
JmsTemplate
```

#### Source File

`com.nets.nps.qr.common.solace.SolaceMessageSender`

#### Actual Implementation

```java
@Component
public class SolaceMessageSender {

    private JmsTemplate jmsTemplate;

    public void sendMessages(final String msg, String correlationId, String replyToTopic, String destination) {
        jmsTemplate.send(destination, new MessageCreator() {
            public Message createMessage(Session session) throws JMSException {
                Message message = session.createTextMessage(msg);
                message.setJMSCorrelationID(correlationId);
                Topic jmsReplyDestination = session.createTopic(replyToTopic);
                message.setJMSReplyTo(jmsReplyDestination);
                return message;
            }
        });
    }

    public void sendMessages(final String msg, String destination) {
        jmsTemplate.send(destination, new MessageCreator() {
            public Message createMessage(Session session) throws JMSException {
                Message message = session.createTextMessage(msg);
                return message;
            }
        });
    }

    public void sendMessages(final String msg, String destination, String correlationId) {
        jmsTemplate.send(destination, new MessageCreator() {
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

#### Available Methods

| Method Signature | Purpose |
|------------------|---------|
| `sendMessages(msg, correlationId, replyToTopic, destination)` | Send with correlation ID and reply-to topic |
| `sendMessages(msg, destination)` | Send simple message without headers |
| `sendMessages(msg, destination, correlationId)` | Send with correlation ID only |

#### Responsibilities

- Sends text messages to Solace destinations

- Sets `JMSCorrelationID` for request-reply matching

- Sets `JMSReplyTo` header for response routing

- Creates JMS TextMessage from string content

---

### 2. SolaceMessageReceiver

#### Purpose

Handles inbound message receiving from Solace message broker using correlation ID filtering.

#### Dependencies

```java
JmsTemplate
```

#### Source File

`com.nets.nps.qr.common.solace.SolaceMessageReceiver`

#### Actual Implementation

```java
@Component
public class SolaceMessageReceiver {

    private static final Logger logger = LoggerFactory.getLogger(SolaceMessageReceiver.class);

    protected JmsTemplate jmsTemplate;

    public String receiveMessage(String correlationId) {
        logger.info("SolaceMessageReceiver trying to receive message with correlation id: {}", correlationId);

        String resCorrelationId = "JMSCorrelationID = '" + correlationId + "'";

        try {
            Message message = jmsTemplate.receiveSelected(resCorrelationId);

            if (Objects.nonNull(message) && message instanceof TextMessage) {
                TextMessage txtMsg = (TextMessage) message;
                return txtMsg.getText();
            }
        } catch (JmsException | JMSException e) {
            logger.error("SolaceMessageReceiver Receiving message Error with correlation id: {}", correlationId, e);
        }

        return null;
    }

    public JmsTemplate getJmsTemplate() {
        return jmsTemplate;
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
}
```

#### Available Methods

| Method Signature | Purpose |
|------------------|---------|
| `receiveMessage(correlationId)` | Receive message filtered by correlation ID |

#### Responsibilities

- Receives messages filtered by correlation ID using JMS selector

- Builds JMS selector string: `JMSCorrelationID = '<correlationId>'`

- Extracts text content from TextMessage

- Returns `null` if no message received within timeout

---

### 3. ReplyToJmsForwardingService

#### Purpose

Manages JMS reply-to destination forwarding for multi-hop message routing in Spring Integration flows.

#### Dependencies

```java
CachingConnectionFactory (qualified as "solaceCachedConnectionFactory")
APSRequestWrapper
```

#### Source File

`com.nets.nps.qr.common.integration.service.impl.ReplyToJmsForwardingService`

#### Actual Implementation

```java
@Service
public class ReplyToJmsForwardingService {

    private static final ApsLogger logger = new ApsLogger(ReplyToJmsForwardingService.class);

    @Autowired(required = false)
    @Qualifier("solaceCachedConnectionFactory")
    private CachingConnectionFactory connectionFactory;
    
    private Map<String, Destination> topicDestinationCache = new HashMap<String, Destination>();
    
    public Destination getCachedTopicDestination(Object responseTopicObj) throws JMSException {
        
        Destination returnTopicDestination = null;
        
        if (responseTopicObj instanceof String) {
            String newResponseQueueString = (String) responseTopicObj;
            
            if (!StringUtils.isEmpty(newResponseQueueString)) {
                
                if (!topicDestinationCache.containsKey(newResponseQueueString)) {
                    Connection cachedConnection = connectionFactory.createConnection();
                    Destination newResponseTopic = cachedConnection.createSession(false, Session.AUTO_ACKNOWLEDGE)
                            .createTopic(newResponseQueueString);
                    
                    topicDestinationCache.put(newResponseQueueString, newResponseTopic);
                }
                returnTopicDestination = topicDestinationCache.get(newResponseQueueString);
            }
            
        } else if (responseTopicObj instanceof Destination) {
            returnTopicDestination = (Destination) responseTopicObj;
        }
        
        return returnTopicDestination;
    }
    
    @ServiceActivator
    public Message<APSRequestWrapper> handle(Message<APSRequestWrapper> input) throws BaseBusinessException, JMSException {

        APSRequestWrapper payload = input.getPayload();
        
        Object responseTopicObj = input.getHeaders().get("responseTopic");
        
        Destination newResponseTopic = getCachedTopicDestination(responseTopicObj);
        
        Object originalJmsReplyToObj = input.getHeaders().get("jms_replyTo");
        String originalJmsReplyTo = "";
        
        if (originalJmsReplyToObj instanceof String) {
            originalJmsReplyTo = (String) originalJmsReplyToObj;
        } else if (originalJmsReplyToObj instanceof Destination) {
            originalJmsReplyTo = ((Destination) originalJmsReplyToObj).toString();
        }

        if (null != newResponseTopic) {
            input = MessageBuilder.withPayload(payload)
                    .copyHeaders(input.getHeaders())
                    .setHeader("jms_replyTo", newResponseTopic)
                    .build();
            logger.info("Forwarding Reply to JMS:" + input.getHeaders());
            
            if (null != originalJmsReplyTo) {
                String replyToJms = originalJmsReplyTo;

                logger.info("Storing original reply to JMS:" + replyToJms, payload);
                payload.getReplyStack().push(replyToJms);
            }
        }
        
        return input;
    }
}
```

#### Available Methods

| Method Signature | Purpose |
|------------------|---------|
| `getCachedTopicDestination(responseTopicObj)` | Get or create cached topic destination |
| `handle(Message<APSRequestWrapper>)` | Spring Integration service activator |

#### Responsibilities

- Caches topic destinations for reuse

- Forwards messages with modified reply-to headers

- Stores original reply-to in wrapper stack for multi-hop routing

- Supports Spring Integration `@ServiceActivator` pattern

---

### 4. ReplyToJmsReplyService

#### Purpose

Restores original JMS reply-to destinations from reply stack for response routing.

#### Dependencies

```java
APSRequestWrapper
```

#### Source File

`com.nets.nps.qr.common.integration.service.impl.ReplyToJmsReplyService`

#### Actual Implementation

```java
@Service
public class ReplyToJmsReplyService {

    private static final ApsLogger logger = new ApsLogger(ReplyToJmsReplyService.class);

    @ServiceActivator
    public Message<APSRequestWrapper> handle(Message<APSRequestWrapper> input) throws BaseBusinessException {
        APSRequestWrapper payload = input.getPayload();

        if (payload.getReplyStack().isEmpty()) {
            return null;
        }

        String replyToJmsString = payload.getReplyStack().pop();
        logger.info("Setting Reply to JMS:" + replyToJmsString, payload);

        return MessageBuilder.withPayload(payload)
                .copyHeaders(input.getHeaders())
                .setHeader("jms_replyTo", replyToJmsString)
                .build();
    }
}
```

#### Available Methods

| Method Signature | Purpose |
|------------------|---------|
| `handle(Message<APSRequestWrapper>)` | Spring Integration service activator for reply routing |

#### Responsibilities

- Retrieves original reply-to from stack

- Sets reply-to header for response routing

- Returns `null` if reply stack is empty

---

### 5. GracefulShutdownThread

#### Purpose

Manages graceful shutdown of JMS message listeners to prevent message loss.

#### Dependencies

```java
DefaultMessageListenerContainer
ConfigurableApplicationContext
```

#### Source File

`com.nets.nps.qr.common.integration.service.impl.GracefulShutdownThread`

#### Actual Implementation

```java
public class GracefulShutdownThread extends Thread {

    private static final Logger logger = LoggerFactory.getLogger(GracefulShutdownThread.class);
    
    private List<DefaultMessageListenerContainer> jmsMessageListeners = new ArrayList<>();
    private ConfigurableApplicationContext context;
    
    public GracefulShutdownThread(ConfigurableApplicationContext context, 
            DefaultMessageListenerContainer... jmsMessageListeners) {
        this.context = context;
        this.jmsMessageListeners = Arrays.asList(jmsMessageListeners);
    }
    
    public GracefulShutdownThread(ConfigurableApplicationContext context) {
        this.context = context;
        Arrays.asList(context.getBeanNamesForType(DefaultMessageListenerContainer.class))
            .forEach(s -> this.jmsMessageListeners.add(
                    (DefaultMessageListenerContainer) context.getBean(s)));
    }
    
    public void run() {

        for (DefaultMessageListenerContainer listener : jmsMessageListeners) {
            listener.stop();
        }
        
        try {
            logger.info("Sleep start");
            sleep(20000);
            logger.info("Sleep end. System Shutdown Complete");
        } catch (Throwable e) {
            logger.error("Sleep interupted", e);
        }
        
        context.close();
    }
}
```

#### Available Constructors

| Constructor | Purpose |
|-------------|---------|
| `GracefulShutdownThread(context, listeners...)` | With explicit listener list |
| `GracefulShutdownThread(context)` | Auto-discover all listeners from context |

#### Responsibilities

- Stops all message listener containers

- Provides 20-second grace period before shutdown

- Closes application context

- Auto-discovers `DefaultMessageListenerContainer` beans if not specified

---

---

## JMS Headers Reference

| Header | Purpose | Set By |
|--------|---------|--------|
| `JMSCorrelationID` | Links request to response | SolaceMessageSender |
| `JMSReplyTo` | Destination for response | SolaceMessageSender |
| `JMSMessageID` | Unique message identifier | Solace Broker |
| `JMSTimestamp` | Message send time | Solace Broker |
| `jms_replyTo` | Spring Integration header | ReplyToJmsForwardingService |

---

## Configuration by Consuming Projects

Since this is a library, consuming projects must configure the JMS beans.

### Option A: XML Configuration

```xml
<!-- Connection Factory -->
<bean id="solaceCachedConnectionFactory" 
      class="org.springframework.jms.connection.CachingConnectionFactory">
    <constructor-arg ref="solaceConnectionFactory" />
    <property name="sessionCacheSize" value="10" />
</bean>

<!-- Producer JmsTemplate -->
<bean id="producerJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <constructor-arg ref="solaceCachedConnectionFactory" />
    <property name="pubSubDomain" value="true" />
</bean>

<!-- Receiver JmsTemplate -->
<bean id="receiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="responseQueue" />
    <property name="receiveTimeout" value="${message.receive.timeout}" />
</bean>

<!-- Message Sender Bean -->
<bean id="solaceMessageSender" 
      class="com.nets.nps.qr.common.solace.SolaceMessageSender">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>

<!-- Message Receiver Bean -->
<bean id="solaceMessageReceiver" 
      class="com.nets.nps.qr.common.solace.SolaceMessageReceiver">
    <property name="jmsTemplate" ref="receiverJmsTemplate" />
</bean>
```

### Option B: Java Configuration

```java
@Configuration
public class SolaceConfig {

    @Bean
    @Qualifier("solaceCachedConnectionFactory")
    public CachingConnectionFactory solaceCachedConnectionFactory(
            ConnectionFactory solaceConnectionFactory) {
        CachingConnectionFactory factory = new CachingConnectionFactory();
        factory.setTargetConnectionFactory(solaceConnectionFactory);
        factory.setSessionCacheSize(10);
        return factory;
    }

    @Bean
    public JmsTemplate producerJmsTemplate(
            @Qualifier("solaceCachedConnectionFactory") CachingConnectionFactory cf) {
        JmsTemplate template = new JmsTemplate(cf);
        template.setPubSubDomain(true);
        return template;
    }

    @Bean
    public JmsTemplate receiverJmsTemplate(
            @Qualifier("solaceCachedConnectionFactory") CachingConnectionFactory cf,
            @Value("${message.receive.timeout}") long timeout) {
        JmsTemplate template = new JmsTemplate(cf);
        template.setReceiveTimeout(timeout);
        return template;
    }

    @Bean
    public SolaceMessageSender solaceMessageSender(JmsTemplate producerJmsTemplate) {
        SolaceMessageSender sender = new SolaceMessageSender();
        sender.setJmsTemplate(producerJmsTemplate);
        return sender;
    }

    @Bean
    public SolaceMessageReceiver solaceMessageReceiver(JmsTemplate receiverJmsTemplate) {
        SolaceMessageReceiver receiver = new SolaceMessageReceiver();
        receiver.setJmsTemplate(receiverJmsTemplate);
        return receiver;
    }
}
```

## Component Summary

| Component | Type | Purpose |
|-----------|------|---------|
| SolaceMessageSender | Producer | Sends messages with correlation and reply-to support |
| SolaceMessageReceiver | Consumer | Receives messages by correlation ID selector |
| ReplyToJmsForwardingService | Integration | Multi-hop reply-to forwarding |
| ReplyToJmsReplyService | Integration | Reply stack management |
| GracefulShutdownThread | Lifecycle | Clean listener shutdown |

---
