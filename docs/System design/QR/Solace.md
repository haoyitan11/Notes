# Solace Message Broker Dependencies Handle
## Components

### 1. JndiTemplate
#### Purpose
Provides JNDI context for Solace resource lookup.

#### XML Declaration
```java
<bean id="solaceJndiTemplate"
    class="org.springframework.jndi.JndiTemplate">
</bean>
```

#### Responsibilities
- Creates JNDI context
- Looks up Solace resources

### 2. JndiObjectFactoryBean
#### Purpose
Looks up ConnectionFactory, Queue, and Topic from Solace JNDI.

#### XML Declaration
```java
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.connection.factory}" />
</bean>
```

#### Responsibilities
- Retrieves Solace-managed JMS objects.
- Exposes them as Spring beans.

### 3. CachingConnectionFactory
#### Purpose
Acts as the central JMS connection manager.

#### XML Declaration
```java
<bean id="solaceCachedConnectionFactory" class="org.springframework.jms.connection.CachingConnectionFactory">
    <property name="targetConnectionFactory" ref="solaceConnectionFactory" />
    <property name="sessionCacheSize" value="10"/>
</bean>
```

#### Responsibilities
- Wraps Solace ConnectionFactory.
- Caches JMS sessions.
- Reuses JMS connections.

### 4. Producer JmsTemplate
#### Purpose
Provides outbound JMS operations.

#### XML Declaration
```java
<bean id="producerJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <constructor-arg ref="solaceCachedConnectionFactory"/>
    <property name="pubSubDomain" value="true"/>
</bean>
```

#### Functions Used by Developer
JmsTemplate.send(destination, MessageCreator);

#### Responsibilities
- Sends JMS messages.
- Creates JMS sessions.

### 5. SolaceMessageSender
#### Purpose
Handles outbound messaging to Solace.

#### XML Declaration
```java
<bean id="solaceMessageSender" class="com.upi.adaptor.nets.SolaceMessageSender">
    <property name="jmsTemplate" ref="producerJmsTemplate"/>
    <property name="destination" value="${solace.upiadapter.to.inbound.topic.name}"/>
</bean>
```

#### Functions Used
```java
sendMessages(msg, correlationId);

sendMessages(msg, destination);

sendMessages(msg, destination, correlationId);
```

#### Responsibilities
- Sends outbound messages.
- Sets JMSCorrelationID.
- Sets JMSReplyTo.

### 6. MessageCreator
#### Purpose
Creates JMS messages.

#### Functions Used
```java
createMessage(Session session);
```

#### Responsibilities
- Creates TextMessage.
- Sets JMS headers.
- Sets Reply-To destination.

### 7. Session
#### Purpose
Provides JMS context.

#### Functions Used
```java
session.createTextMessage(msg);

session.createTopic(topicName);

session.createQueue(queueName);
```

#### Responsibilities
- Creates messages.
- Creates destinations.

### 8. Message (TextMessage)
#### Purpose
Carries payload and JMS headers.

#### Functions Used
```java
message.setJMSCorrelationID(correlationId);

message.setJMSReplyTo(destination);

textMessage.getText();
```

#### Responsibilities
- Stores payload.
- Stores Correlation ID.
- Stores Reply-To destination.

### 9. Destination (Topic / Queue)
#### Purpose
Represents messaging endpoint.

#### Functions Used
```java
session.createTopic(topicName);

session.createQueue(queueName);
```

#### Responsibilities
- Identifies message target.
- Identifies message source.

### 10. Receiver JmsTemplate
#### Purpose
Provides inbound JMS receive operations.

#### XML Declaration
<bean id="financialMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory"/>
    <property name="defaultDestination" ref="consumerQueue"/>
</bean>

#### Functions Used
```java
receiveSelected(selector);
```

#### Responsibilities
- Receives response messages.
- Filters using selectors.

### 11. SolaceMessageReceiver
#### Purpose
Receives response messages from Solace.

#### XML Declaration
```java
<bean id="solaceMessageReceiver" class="com.upi.adaptor.nets.SolaceMessageReceiver">
    <property name="jmsTemplate" ref="financialMessageReceiverJmsTemplate"/>
</bean>
```

#### Functions Used
```java
receiveMeassage(correlationId);

jmsTemplate.receiveSelected(selector);
```

#### Responsibilities
- Receives correlated messages.
- Extracts TextMessage payload.

### 12. DefaultMessageListenerContainer
#### Purpose
Provides asynchronous JMS consumption.

#### XML Declaration
```java
<bean id="jsonMessageListenerContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="destination" ref="jsonInboundQueue"/>
    <property name="connectionFactory" ref="solaceConnectionFactory
    <property name="concurrentConsumers" value="10" />
</bean>
```

#### Responsibilities
- Listens to inbound queue.
- Creates JMS consumers.

### 13. message-driven-channel-adapter
#### Purpose
Bridges JMS into Spring Integration.

#### XML Declaration
```java
<int-jms:message-driven-channel-adapter id="jsonJmsIn" container="jsonMessageListenerContainer" channel="jsonJmsInChannel" extract-payload="true"/>
```
