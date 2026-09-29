# Solace Message Broker Dependencies Handle
## Components



### 1. CachingConnectionFactory
#### Purpose
Acts as the central JMS connection manager that caches and manages Solace JMS connections.

#### Dependencies
ConnectionFactory

#### Functions Used
Create connection:
```java
CachingConnectionFactory.createConnection();
```

#### Responsibilities
- Central coordinator for JMS connection management
- Wraps SolConnectionFactory.
- Caches JMS connections for reuse.
- Manages connection lifecycle.
- Provides connections to JmsTemplate and listener containers.

### 2. SolConnectionFactory (ConnectionFactory)
#### Purpose
Provides the underlying Solace JMS connection implementation.

#### Dependencies
```java
Solace Broker
```

#### Functions Used
Create factory:
```java
SolJmsUtility.createConnectionFactory(host,vpn,username,password);
```

#### Responsibilities
- Creates native Solace JMS connections.
- Manages broker authentication.
- Configures VPN access.
- Manages broker communication.

### 3. JmsTemplate
#### Purpose
Acts as the central JMS messaging operations manager.

#### Dependencies
```java
CachingConnectionFactory
Destination
MessageCreator
```

#### Functions Used
Send message:
```java
JmsTemplate.send(destination,MessageCreator);
```

Receive selected:
```java
JmsTemplate.receiveSelected(selector);
```

#### Responsibilities
- Sends JMS messages.
- Receives JMS messages.
- Manages JMS sessions.
- Provides request-reply messaging support.
