# Solace Core Infrastructure
## Components

### 1. JndiTemplate

#### Purpose

Provides JNDI context for Solace resource lookup with SSL certificate authentication support.

#### Dependencies

```
None (Root Bean)
```

#### Configuration Source

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

#### Responsibilities

- Creates JNDI context with Solace broker connection details
- Provides lookup capability for Solace-managed resources
- Configures authentication (username/password for local, SSL certificates for production)
- Establishes connection to Solace VPN

---

### 2. JndiObjectFactoryBean

#### Purpose

Looks up ConnectionFactory and Queue from Solace JNDI.

#### Dependencies

```java
JndiTemplate
```

#### XML Declaration (spring-context.xml)

```xml
<bean id="solaceConnectionFactory" class="org.springframework.jndi.JndiObjectFactoryBean" 
      lazy-init="default" autowire="default">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
    <property name="jndiName" value="${solace.connection.factory}"/>
</bean>
```

#### Java Configuration (JmsConfiguration.java)

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

#### Actual Usage - Queue Lookup (upi-request-adapter.xml)

```xml
<bean id="upiAdapterReq.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>

<bean id="processor.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.processor.response.queue.name}" />
</bean>

<bean id="reversal.consumerQueue" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiTemplate" ref="solaceJndiTemplate" />
    <property name="jndiName" value="${solace.upiproxy.debit.response.queue.name}" />
</bean>
```

#### Responsibilities

- Retrieves Solace-managed JMS objects via JNDI lookup
- Exposes ConnectionFactory and Queue as Spring beans
- Acts as bridge between JNDI and Spring context

---

### 3. CachingConnectionFactory

#### Purpose

Acts as the central JMS connection manager that caches connections and sessions for performance.

#### Dependencies

```java
ConnectionFactory (from JndiObjectFactoryBean)
```

#### XML Declaration (spring-context.xml)

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

#### Java Configuration (JmsConfiguration.java)

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

#### Properties

| Property | Value | Description |
|----------|-------|-------------|
| sessionCacheSize | 10 | Number of JMS sessions to cache |
| reconnectOnException | true | Auto-reconnect on connection failure |
| cacheConsumers | false | Consumer caching disabled for dynamic destinations |

#### Responsibilities

- Wraps Solace ConnectionFactory
- Caches JMS sessions for improved performance
- Manages connection lifecycle with auto-reconnect
- Provides thread-safe session management

---

### 4. JndiDestinationResolver

#### Purpose

Resolves destination names to actual Solace Queue/Topic objects via JNDI.

#### Dependencies

```java
JndiTemplate
```

#### XML Declaration (spring-context.xml)

```xml
<bean id="jndiDestinationResolver"
      class="org.springframework.jms.support.destination.JndiDestinationResolver">
    <property name="jndiTemplate" ref="solaceJndiTemplate"/>
</bean>
```

#### Responsibilities

- Resolves string destination names to JMS Destination objects
- Enables dynamic destination lookup
- Supports both Queue and Topic resolution

## Configuration Files

| File | Contents |
|------|----------|
| `SolaceConfiguration.java` | JndiTemplate bean (Production with SSL) |
| `LocalSolaceConfiguration.java` | JndiTemplate bean (Local with password) |
| `JmsConfiguration.java` | ConnectionFactory, CachingConnectionFactory beans |
| `spring-context.xml` | XML-based JndiTemplate (profile-based), ConnectionFactory, CachingConnectionFactory |

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

### Alternative SSL Properties (Java Config)

```properties
solace.jms.ssl.validate_certificate=true
solace.jms.ssl.keystore=/path/to/keystore.jks
solace.jms.ssl.keystore.password=ENC(<encrypted>)
solace.jms.ssl.authentication.scheme=AUTHENTICATION_SCHEME_CLIENT_CERTIFICATE
solace.jms.ssl.privatekey.alias=<alias>
solace.jms.ssl.privatekey.password=ENC(<encrypted>)
solace.jms.ssl.truststore=/path/to/truststore.jks
solace.jms.ssl.truststore.password=ENC(<encrypted>)
```
