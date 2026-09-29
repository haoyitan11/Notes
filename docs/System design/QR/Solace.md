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

