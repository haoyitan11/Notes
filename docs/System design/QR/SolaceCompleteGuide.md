# Solace Complete Implementation Guide

## Overview

This document provides a complete guide for the Solace messaging implementation in the UPI Proxy application. It covers three layers:

1. **Core Infrastructure** - Foundation connectivity to Solace Broker
2. **JMS Messaging** - Direct JMS messaging capabilities (Synchronous Request-Reply)
3. **Spring Integration** - Message flow processing with transformations (Asynchronous Processing)

> **Important Configuration Notes:**
> - **Core Infrastructure**: Uses **both** XML (`spring-context.xml`) and Java (`SolaceConfiguration.java`, `JmsConfiguration.java`) configuration
> - **JMS Messaging**: XML configuration only (`upi-request-adapter.xml`) with Java implementation classes
> - **Spring Integration**: XML configuration only (various flow XML files) with Java `@ServiceActivator` methods

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              SOLACE COMPLETE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    LAYER 1: CORE INFRASTRUCTURE                              │    │
│  │                        (XML + Java Configuration)                            │    │
│  │  ┌───────────────────┐   ┌─────────────────────┐   ┌───────────────────────┐│    │
│  │  │ solaceJndiTemplate│──►│ solaceConnection    │──►│solaceCachedConnection ││    │
│  │  │ (XML profiles)    │   │ Factory             │   │Factory                ││    │
│  │  └───────────────────┘   └─────────────────────┘   └───────────────────────┘│    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                           │
│              ┌───────────────────────────┴───────────────────────────┐              │
│              ▼                                                       ▼              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    LAYER 2: JMS MESSAGING (XML Config)                       │    │
│  │  ┌─────────────────┐   ┌───────────────────┐   ┌─────────────────────┐      │    │
│  │  │ producerJms     │   │ SolaceMessage     │   │ SolaceMessage       │      │    │
│  │  │ Template        │   │ Sender            │   │ Receiver            │      │    │
│  │  └─────────────────┘   └───────────────────┘   └─────────────────────┘      │    │
│  │  Pattern: Synchronous Request-Reply with Correlation ID                      │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    LAYER 3: SPRING INTEGRATION (XML Config)                  │    │
│  │  ┌──────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐   │    │
│  │  │Listener      │─►│ Channels    │─►│ Transformer │─►│@ServiceActivator │   │    │
│  │  │Container     │  │ (wire-tap)  │  │ (JSON↔Obj)  │  │methods           │   │    │
│  │  └──────────────┘  └─────────────┘  └─────────────┘  └──────────────────┘   │    │
│  │  Pattern: Asynchronous Processing via Spring Integration Flows               │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 1: Core Infrastructure
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/53540487-1374-45ab-9549-310d58fb5ed2" />

## Purpose

Provides the foundational connectivity layer between the application and Solace Broker.

## 1.1 JndiTemplate (XML Configuration)

### Purpose

Provides JNDI context for Solace resource lookup with profile-based configuration.

### Configuration (spring-context.xml)

**Development Profile:**

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
```

**SIT/UAT/PROD Profile (with SSL):**

```xml
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
- Configures authentication (password or SSL certificate)

---

## 1.2 JndiTemplate (Java Configuration - Alternative)

### Purpose

Alternative Java-based JNDI configuration with Spring profiles.

### Configuration (SolaceConfiguration.java - @Profile("!local"))

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

### Configuration (LocalSolaceConfiguration.java - @Profile("local"))

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

---

## 1.3 ConnectionFactory and CachingConnectionFactory (XML)

### Purpose

Looks up ConnectionFactory from Solace JNDI and wraps with caching.

### Configuration (spring-context.xml)

```xml
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean" 
      lazy-init="default" autowire="default">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.connection.factory}"/>
</bean>

<bean id="solaceCachedConnectionFactory" 
      class="org.springframework.jms.connection.CachingConnectionFactory"
      primary="true">
    <property name="targetConnectionFactory" ref="solaceConnectionFactory"/>
    <property name="sessionCacheSize" value="10"/>
    <property name="reconnectOnException" value="true"/>
    <property name="cacheConsumers" value="false"/>
</bean>
```

### Configuration (JmsConfiguration.java - Alternative)

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

    @Bean
    public CachingConnectionFactory cachedConnectionFactoryfinal(ConnectionFactory connectionFactory) {
        final CachingConnectionFactory cachedConnectionFactory = new CachingConnectionFactory();
        cachedConnectionFactory.setTargetConnectionFactory(connectionFactory);
        cachedConnectionFactory.setSessionCacheSize(10);
        cachedConnectionFactory.setReconnectOnException(true);
        cachedConnectionFactory.setCacheConsumers(false);
        return cachedConnectionFactory;
    }
}
```

### Responsibilities

- Retrieves Solace ConnectionFactory via JNDI lookup
- Caches JMS sessions for improved performance
- Manages connection lifecycle with auto-reconnect

---

## 1.4 JndiDestinationResolver

### Purpose

Resolves destination names to JNDI-looked-up destinations.

### Configuration (spring-context.xml)

```xml
<bean id="jndiDestinationResolver"
      class="org.springframework.jms.support.destination.JndiDestinationResolver">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
</bean>
```

### Responsibilities

- Resolves queue/topic names to JNDI destinations
- Used by outbound channel adapters with `destination-resolver` attribute

---

# Part 2: JMS Messaging

## Purpose

Provides direct JMS-based messaging capabilities for synchronous request-reply pattern.

## 2.1 Producer JmsTemplate

### Purpose

Provides JMS operations for sending messages to Solace topics.

### Configuration (upi-request-adapter.xml)

```xml
<bean id="producerJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <constructor-arg ref="solaceCachedConnectionFactory" />
    <property name="pubSubDomain" value="true" />
</bean>
```

### Responsibilities

- Sends messages to Solace destinations (topics)
- Provides send operations with MessageCreator callback

---

## 2.2 Receiver JmsTemplate(s)

### Purpose

Provides inbound JMS receive operations with selector support.

### Configuration (upi-request-adapter.xml)

```xml
<!-- UPI Adapter Request Receiver -->
<bean id="upiAdapterReq.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>

<bean id="upiAdapterReqMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="upiAdapterReq.consumerQueue" />
    <property name="receiveTimeout" value="${upi.message.receive.timeout}" />
</bean>

<!-- Reversal Message Receiver -->
<bean id="reversal.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>

<bean id="reversalMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="reversal.consumerQueue" />
    <property name="receiveTimeout" value="${upi.message.receive.timeout}" />
</bean>

<!-- Processor Message Receiver -->
<bean id="processor.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.processor.response.queue.name}" />
</bean>

<bean id="processorMessageReceiverJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="solaceCachedConnectionFactory" />
    <property name="defaultDestination" ref="processor.consumerQueue" />
    <property name="receiveTimeout" value="${upi.message.receive.timeout}" />
</bean>
```

### Responsibilities

- Receives response messages from queue
- Filters messages using JMS selectors (correlation ID)
- Provides synchronous receive with configurable timeout

---

## 2.3 SolaceMessageSender (nps-qr-common Library)

### Purpose

Handles outbound message sending to Solace with request-reply pattern support.

### Implementation (com.nets.nps.qr.common.solace.SolaceMessageSender)

```java
package com.nets.nps.qr.common.solace;

import jakarta.jms.JMSException;
import jakarta.jms.Message;
import jakarta.jms.Session;
import jakarta.jms.Topic;

import org.springframework.jms.core.JmsTemplate;
import org.springframework.jms.core.MessageCreator;
import org.springframework.stereotype.Component;

@Component
public class SolaceMessageSender {

    private JmsTemplate jmsTemplate;

    // Send with correlation ID and reply-to topic
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

    // Send simple message
    public void sendMessages(final String msg, String destination) {
        jmsTemplate.send(destination, new MessageCreator() {
            public Message createMessage(Session session) throws JMSException {
                Message message = session.createTextMessage(msg);
                return message;
            }
        });
    }

    // Send with correlation ID only
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

### Bean Configuration (upi-request-adapter.xml)

```xml
<bean id="upiAdapterReqMessageProducer"
    class="com.nets.nps.qr.common.solace.SolaceMessageSender">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>
```

### Responsibilities

- Sends text messages to Solace destinations
- Sets JMSCorrelationID for request-reply matching
- Sets JMSReplyTo header for response routing

---

## 2.4 SolaceMessageReceiver (nps-qr-common Library)

### Purpose

Handles inbound message receiving using correlation ID filtering.

### Implementation (com.nets.nps.qr.common.solace.SolaceMessageReceiver)

```java
package com.nets.nps.qr.common.solace;

import java.util.Objects;

import jakarta.jms.JMSException;
import jakarta.jms.Message;
import jakarta.jms.TextMessage;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jms.JmsException;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.stereotype.Component;

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

### Bean Configuration (upi-request-adapter.xml)

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

### Responsibilities

- Receives messages filtered by correlation ID
- Builds JMS selector: `JMSCorrelationID = '<correlationId>'`
- Extracts text content from TextMessage
- Returns null if no message received within timeout

---

## 2.5 SolaceMessageSenderOneWay (Local Implementation)

### Purpose

Handles outbound messaging with optional correlation support (no reply-to).

### Implementation (SolaceMessageSenderOneWay.java)

```java
package com.nets.upi.processing.integration.upi.proxy.service.impl;

import jakarta.jms.JMSException;
import jakarta.jms.Message;
import jakarta.jms.Session;

import org.springframework.jms.core.JmsTemplate;
import org.springframework.jms.core.MessageCreator;
import org.springframework.stereotype.Component;

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

    public JmsTemplate getJmsTemplate() {
        return jmsTemplate;
    }

    public void setJmsTemplate(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
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
}
```

### Bean Configuration (upi-request-adapter.xml)

```xml
<bean id="upiDebitMessageProducer"
    class="com.nets.upi.processing.integration.upi.proxy.service.impl.SolaceMessageSenderOneWay">
    <property name="jmsTemplate" ref="producerJmsTemplate" />
</bean>
```

### Responsibilities

- Sends outbound text messages to specified topic
- Optionally sets JMSCorrelationID for one-way messaging

# Part 3: Spring Integration

## Purpose

Receives messages from Solace queues, transforms payloads, invokes business services, and publishes responses asynchronously.

## Message Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           SPRING INTEGRATION FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────────┐   │
│  │  Solace Queue    │───►│ DefaultMessage   │───►│ message-driven-channel-      │   │
│  │                  │    │ ListenerContainer│    │ adapter                       │   │
│  └──────────────────┘    └──────────────────┘    └──────────────┬───────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                              jmsInChannel                                     │   │
│  │                         (with wire-tap logging)                               │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                        json-to-object-transformer                             │   │
│  │                       (String → Domain Object)                                │   │
│  │                       (Optional - some flows skip this)                       │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                              inChannel                                        │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                           service-activator                                   │   │
│  │                     ref="*ProcessingService" method="process"                 │   │
│  │                     (@ServiceActivator annotated method)                      │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                              outputChannel                                    │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                        object-to-json-transformer                             │   │
│  │                       (Domain Object → String)                                │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                  │                   │
│                                                                  ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                       outbound-channel-adapter                                │   │
│  │          destination-expression="headers['jms_replyTo']" or static            │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.1 DefaultMessageListenerContainer

### Purpose

Provides asynchronous JMS consumption with concurrent consumers.

### Configuration (upi-proxy-transactions.xml)

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

### Responsibilities

- Listens to inbound queue asynchronously
- Scales consumers based on load
- Provides graceful shutdown support

---

## 3.2 Message-Driven Channel Adapter

### Purpose

Bridges JMS messages into Spring Integration channels.

### Configuration (upi-proxy-transactions.xml)

```xml
<int-jms:message-driven-channel-adapter
        id="ap.jmsIn"
        channel="ap.jmsInChannel"
        container="ap.messageListenerContainer"
        extract-payload="true"
        error-channel="errorChannel"
/>
```

### Responsibilities

- Converts JMS messages to Spring Integration messages
- Extracts payload from TextMessage
- Routes to Spring Integration channel

---

## 3.3 Spring Integration Channels with Wire-Tap

### Purpose

Provides message routing infrastructure with logging.

### Configuration (upi-proxy-transactions.xml)

```xml
<int:logging-channel-adapter id="apLog" level="INFO" log-full-message="true" logger-name="apLog"/>

<int:channel id="ap.jmsInChannel">
    <int:interceptors>
        <int:wire-tap channel="apLog"/>
    </int:interceptors>
</int:channel>

<int:channel id="ap.inChannel"/>
```

### Responsibilities

- Routes messages between components
- Provides wire-tap for logging
- Supports transformation pipeline

---

## 3.4 JSON Transformers

### Purpose

Transforms between JSON strings and Java objects.

### Configuration (upi-proxy-transactions.xml)

```xml
<int:json-to-object-transformer 
    input-channel="ap.jmsInChannel" 
    output-channel="ap.inChannel"
    type="com.nets.upi.domain.UpiProxyRequest">
</int:json-to-object-transformer>
```

### Configuration (common.xml)

```xml
<int:channel id="outputChannel" />

<int:object-to-json-transformer 
    input-channel="outputChannel" 
    output-channel="jmsOutChannel" />

<int:channel id="jmsOutChannel" />
```

### Responsibilities

- Deserializes JSON to Java objects
- Serializes Java objects to JSON
- Enables type-safe processing

---

## 3.5 Service Activator

### Purpose

Connects Spring Integration channels to business service methods.

### Configuration (upi-proxy-transactions.xml)

```xml
<int:service-activator
        input-channel="ap.inChannel"
        output-channel="outputChannel"
        ref="transactionUpiProxyProcessingService"
        method="process">
</int:service-activator>
```

### Implementation (TransactionUpiProxyProcessingService.java)

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

### Responsibilities

- Receives transformed payload
- Executes business logic
- Returns response to output channel

---

## 3.6 Outbound Channel Adapter

### Purpose

Sends Spring Integration messages back to Solace.

### Configuration - Dynamic Destination (common.xml)

```xml
<int-jms:outbound-channel-adapter 
    id="jmsOut" 
    pub-sub-domain="true"
    destination-expression="headers['jms_replyTo']"
    channel="jmsOutChannel"
    connection-factory="solaceConnectionFactory">
</int-jms:outbound-channel-adapter>
```

### Configuration - Static Destination (fx-hub-request.xml)

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

### Responsibilities

- Publishes response messages to Solace topics
- Supports dynamic routing via JMSReplyTo
- Supports static destination configuration

---

## 3.7 Graceful Shutdown

### Purpose

Manages graceful shutdown of Tomcat connector.

### Implementation (GracefulShutdownConfiguration.java)

```java
@Configuration
public class GracefulShutdownConfiguration {

    @Value("${server.gracefulshutdown.timeout.second:50}")
    private long timeoutInSec;

    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown(timeoutInSec);
    }

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
        return factory -> factory.addConnectorCustomizers(gracefulShutdown());
    }

    private static class GracefulShutdown implements TomcatConnectorCustomizer {

        private static final Logger logger = LoggerFactory.getLogger(GracefulShutdown.class);
        private volatile long timeoutInSec;
        private volatile Connector connector;

        public GracefulShutdown(long timeoutInSec) {
            this.timeoutInSec = timeoutInSec;
        }

        @Override
        public void customize(Connector connector) {
            this.connector = connector;
        }

        @EventListener(ContextClosedEvent.class)
        public void onApplicationEvent(ContextClosedEvent event) {
            logger.info("shutting down the service.");
            this.connector.pause();

            Executor executor = this.connector.getProtocolHandler().getExecutor();
            if (executor instanceof ThreadPoolExecutor) {
                try {
                    ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                    threadPoolExecutor.shutdown();
                    if (!threadPoolExecutor.awaitTermination(timeoutInSec, TimeUnit.SECONDS)) {
                        logger.warn("Tomcat thread pool did not shut down gracefully within "
                                    + timeoutInSec + " seconds.");
                        threadPoolExecutor.shutdownNow();
                    }
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }
}
```

### Responsibilities

- Pauses Tomcat connector on shutdown
- Waits for in-flight requests to complete
- Ensures clean application shutdown

---

# Configuration Summary

## Configuration Files

| File | Type | Layer | Purpose |
|------|------|-------|---------|
| spring-context.xml | XML | Core | JndiTemplate, ConnectionFactory, CachingConnectionFactory, imports |
| SolaceConfiguration.java | Java | Core | JndiTemplate (@Profile("!local")) |
| LocalSolaceConfiguration.java | Java | Core | JndiTemplate (@Profile("local")) |
| JmsConfiguration.java | Java | Core | ConnectionFactory, CachingConnectionFactory, JmsTemplate |
| upi-request-adapter.xml | XML | JMS | Producer/Receiver JmsTemplate, SolaceMessageSender/Receiver |
| upi-proxy-transactions.xml | XML | Integration | UPI Proxy transaction flow |
| fx-hub-request.xml | XML | Integration | FX Hub query flow |
| tsp-request-enroll.xml | XML | Integration | TSP enrollment flow |
| tsp-request-cardsm.xml | XML | Integration | TSP card status management flow |
| common.xml | XML | Integration | Shared output channels and transformers |
| GracefulShutdownConfiguration.java | Java | Core | Tomcat graceful shutdown |

---

## Bean Summary

| Bean Name | Class | Config File | Layer |
|-----------|-------|-------------|-------|
| solaceJndiTemplate | JndiTemplate | spring-context.xml | Core |
| solaceConnectionFactory | JndiObjectFactoryBean | spring-context.xml | Core |
| solaceCachedConnectionFactory | CachingConnectionFactory | spring-context.xml | Core |
| jndiDestinationResolver | JndiDestinationResolver | spring-context.xml | Core |
| producerJmsTemplate | JmsTemplate | upi-request-adapter.xml | JMS |
| upiAdapterReqMessageProducer | SolaceMessageSender | upi-request-adapter.xml | JMS |
| upiDebitMessageProducer | SolaceMessageSenderOneWay | upi-request-adapter.xml | JMS |
| upiAdapterReqMessageReceiver | SolaceMessageReceiver | upi-request-adapter.xml | JMS |
| reversalMessageReceiver | SolaceMessageReceiver | upi-request-adapter.xml | JMS |
| processorMessageReceiver | SolaceMessageReceiver | upi-request-adapter.xml | JMS |
| ap.messageListenerContainer | DefaultMessageListenerContainer | upi-proxy-transactions.xml | Integration |
| fx.messageListenerContainer | DefaultMessageListenerContainer | fx-hub-request.xml | Integration |

---

## Properties Reference

### Connection Properties

```properties
solace.url=smfs://<broker-host>:<port>
solace.username=<username>
solace.vpn=<vpn-name>
solace.password=<password>  # dev/local only
solace.connection.factory=jndi/cf/<connection-factory-name>
```

### SSL Properties (SIT/UAT/PROD)

```properties
Solace_JMS_SSL_ValidateCertificate=true
Solace_JMS_SSL_KeyStore=/path/to/keystore.jks
Solace_JMS_SSL_KeyStorePassword=<password>
Solace_JMS_Authentication_Scheme=AUTHENTICATION_SCHEME_CLIENT_CERTIFICATE
Solace_JMS_SSL_PrivateKeyAlias=<alias>
Solace_JMS_SSL_PrivateKeyPassword=<password>
Solace_JMS_SSL_TrustStore=/path/to/truststore.jks
Solace_JMS_SSL_TrustStorePassword=<password>
```

### Consumer Properties

```properties
# High Usage
high.usage.concurrent.consumers=4
high.usage.max.concurrent.consumers=8
high.usage.idle.consumer.limit=4
high.usage.receive.timeout=5000
high.usage.idle.taskexecution.limit=20

# Low Usage
low.usage.concurrent.consumers=2
low.usage.max.concurrent.consumers=4
low.usage.idle.consumer.limit=2
low.usage.receive.timeout=5000
low.usage.idle.taskexecution.limit=20
```

### Timeout Properties

```properties
upi.message.receive.timeout=30000
server.gracefulshutdown.timeout.second=50
```

---

## JMS Headers Reference

| Header | Purpose | Set By |
|--------|---------|--------|
| JMSCorrelationID | Links request to response | SolaceMessageSender |
| JMSReplyTo | Destination for response | SolaceMessageSender |
| JMSMessageID | Unique message identifier | Solace Broker |
| JMSTimestamp | Message send time | Solace Broker |
| jms_replyTo | Spring Integration header | Preserved from JMS |

---

## External Dependencies

| Library | Package | Components |
|---------|---------|------------|
| com.solacesystems:sol-jms-jakarta | - | JMS implementation |
| com.nets.nps.qr:nps-qr-common | com.nets.nps.qr.common.solace | SolaceMessageSender, SolaceMessageReceiver |
