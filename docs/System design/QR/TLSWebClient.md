# TLS WebClient Dependencies Handle

## Overview

This document provides a complete guide for building TLS-enabled WebClient in the application. It covers two TLS patterns:

1. **One-Way TLS** - Server certificate validation only (client trusts server)
2. **Mutual TLS (mTLS)** - Two-way certificate authentication (client and server authenticate each other)

---

# Part 1: One-Way TLS (Server Certificate Validation)

## Purpose

Provides TLS connectivity where only the server's certificate is validated. The client trusts the server but does not present its own certificate.

## Components
<img width="2624" height="1760" alt="image" src="https://github.com/user-attachments/assets/d40ee0c4-6c00-4183-b3ca-eb8aa279137a" />

### 1.1 FileInputStream & ResourceUtils

#### Purpose

Loads the TrustStore file content from the filesystem.

#### Dependencies

```java
java.io.FileInputStream
java.io.InputStream
org.springframework.util.ResourceUtils
```

#### Actual Usage

```java
try (InputStream inputStream = new FileInputStream(ResourceUtils.getFile(trustStorePath))) {
    // Load KeyStore from inputStream
}
```

#### Responsibilities

- Locates TrustStore file by path.
- Opens file as InputStream.
- Provides byte stream to KeyStore loader.
- Used by **KeyStore** for file loading.

---

### 1.2 KeyStore (TrustStore)

#### Purpose

Acts as the secure container that loads and stores trusted certificates.

#### Dependencies

```java
java.security.KeyStore
java.io.InputStream
```

#### Actual Usage

```java
KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
trustStore.load(inputStream, trustStorePass.toCharArray());
```

#### Responsibilities

- Creates KeyStore instance with specified type (JKS, PKCS12).
- Loads TrustStore data from InputStream.
- Decrypts TrustStore using password.
- Stores trusted CA certificates.
- Used by **TrustManagerFactory** for certificate validation.

#### Configuration Source

```java
@Configuration
public class KeyStoreConfiguration {

    @Value("${ssl.trust-store}")
    String trustStorePath;
    
    @Value("${ssl.trust-store.password}")
    String trustStorePass;

    @Bean
    public KeyStore trustStore() throws Exception {
        try (InputStream inputStream = new FileInputStream(ResourceUtils.getFile(trustStorePath))) {
            KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
            trustStore.load(inputStream, trustStorePass.toCharArray());
            return trustStore;
        }
    }
}
```

---

### 1.3 TrustManagerFactory

#### Purpose

Creates TrustManagers that validate server certificates against the TrustStore.

#### Dependencies

```java
javax.net.ssl.TrustManagerFactory
java.security.KeyStore
```

#### Actual Usage

```java
TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("SunX509");
trustManagerFactory.init(trustStore);
```

#### Responsibilities

- Creates TrustManagerFactory instance with specified algorithm.
- Initializes with TrustStore containing trusted CA certificates.
- Provides TrustManagers for TLS context.
- Used by **SslContextBuilder** to configure server certificate validation.

---

### 1.4 SslContextBuilder

#### Purpose

Builds Netty SslContext for TLS connections.

#### Dependencies

```java
io.netty.handler.ssl.SslContext
io.netty.handler.ssl.SslContextBuilder
javax.net.ssl.TrustManagerFactory
```

#### Actual Usage

```java
SslContext sslContext = SslContextBuilder.forClient()
        .trustManager(trustManagerFactory)
        .build();
```

#### Responsibilities

- Creates TLS context for client connections.
- Configures TrustManager for server certificate validation.
- Returns SslContext used by HttpClient.

#### Configuration Source

```java
@Bean
public SslContext oneWayTlsContext(KeyStore trustStore) throws Exception {
    TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("SunX509");
    trustManagerFactory.init(trustStore);

    return SslContextBuilder.forClient()
            .trustManager(trustManagerFactory)
            .build();
}
```

---

### 1.5 HttpClient

#### Purpose

Reactor Netty HTTP client configured with TLS context.

#### Dependencies

```java
reactor.netty.http.client.HttpClient
io.netty.handler.ssl.SslContext
io.netty.channel.ChannelOption
io.netty.handler.timeout.ReadTimeoutHandler
io.netty.handler.timeout.WriteTimeoutHandler
reactor.netty.transport.ProxyProvider
```

#### Actual Usage

```java
HttpClient httpClient = HttpClient.create()
        .disableRetry(true)
        .secure(sslSpec -> sslSpec.sslContext(oneWayTlsContext))
        .proxy(proxy -> proxy
                .type(ProxyProvider.Proxy.HTTP)
                .host(proxyHost)
                .port(proxyPort)
                .connectTimeoutMillis(connectTimeout))
        .doOnConnected(conn -> conn
                .addHandlerFirst(new ReadTimeoutHandler(responseTimeout, TimeUnit.MILLISECONDS))
                .addHandlerFirst(new WriteTimeoutHandler(requestTimeout, TimeUnit.MILLISECONDS)))
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectTimeout);
```

#### Responsibilities

- Creates reactive HTTP client.
- Configures TLS with SslContext.
- Configures proxy settings.
- Configures connection and read/write timeouts.
- Used by **ReactorClientHttpConnector** to bridge to WebClient.

---

### 1.6 ReactorClientHttpConnector

#### Purpose

Bridges Reactor Netty HttpClient to Spring WebClient.

#### Dependencies

```java
org.springframework.http.client.reactive.ReactorClientHttpConnector
reactor.netty.http.client.HttpClient
```

#### Actual Usage

```java
new ReactorClientHttpConnector(httpClient)
```

#### Responsibilities

- Wraps HttpClient for use with Spring WebClient.
- Adapts Reactor Netty to Spring reactive HTTP client interface.
- Used by **WebClient.Builder** to configure HTTP transport.

---

### 1.7 WebClient

#### Purpose

Spring reactive HTTP client for making HTTP requests.

#### Dependencies

```java
org.springframework.web.reactive.function.client.WebClient
org.springframework.http.client.reactive.ReactorClientHttpConnector
```

#### Actual Usage

```java
@Bean(name = "token-client")
public WebClient createTokenWebClient(SslContext oneWayTlsContext) {
    return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient(oneWayTlsContext)))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON.toString())
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON.toString())
            .filter(logRequest())
            .build();
}
```

#### Responsibilities

- Provides reactive HTTP client API.
- Configures default headers.
- Configures request/response filters.
- Used by application services to make HTTP calls.

---

## Execution Flow (One-Way TLS)

1. **KeyStore.load()** loads TrustStore from filesystem:

   ```java
   InputStream inputStream = new FileInputStream(ResourceUtils.getFile(trustStorePath));
   KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
   trustStore.load(inputStream, trustStorePass.toCharArray());
   ```

2. **TrustManagerFactory.init()** initializes with TrustStore:

   ```java
   TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("SunX509");
   trustManagerFactory.init(trustStore);
   ```

3. **SslContextBuilder.forClient()** creates TLS context:

   ```java
   SslContext sslContext = SslContextBuilder.forClient()
           .trustManager(trustManagerFactory)
           .build();
   ```

4. **HttpClient.secure()** configures TLS:

   ```java
   HttpClient httpClient = HttpClient.create()
           .secure(sslSpec -> sslSpec.sslContext(sslContext));
   ```

5. **WebClient.builder()** creates WebClient with HTTP client:

   ```java
   WebClient webClient = WebClient.builder()
           .clientConnector(new ReactorClientHttpConnector(httpClient))
           .build();
   ```

---

# Part 2: Mutual TLS (Two-Way Authentication)

## Purpose

Provides mTLS connectivity where both client and server authenticate each other using certificates. The client presents its certificate (via KeyStore) and validates the server's certificate (via TrustStore).

## Components
<img width="2624" height="2152" alt="image" src="https://github.com/user-attachments/assets/8156f380-adbc-4885-aaa3-fe8dbc99a9f8" />

### 2.1 FileInputStream (KeyStore)

#### Purpose

Loads the KeyStore file containing client private key and certificate.

#### Dependencies

```java
java.io.FileInputStream
```

#### Actual Usage

```java
try (FileInputStream keystoreInputStream = new FileInputStream(keyStorePath)) {
    // Load KeyStore from inputStream
}
```

#### Responsibilities

- Locates KeyStore file by path.
- Opens file as InputStream.
- Provides byte stream to KeyStore loader.

---

### 2.2 KeyStore (Client KeyStore)

#### Purpose

Acts as the secure container that stores client private key and certificate chain.

#### Dependencies

```java
java.security.KeyStore
java.io.FileInputStream
```

#### Actual Usage

```java
KeyStore keyStore = KeyStore.getInstance(
        Optional.ofNullable(keyStoreType).orElse(KeyStore.getDefaultType()));
char[] charKeyStorePswd = keyStorePassword.toCharArray();
keyStore.load(keystoreInputStream, charKeyStorePswd);
```

#### Responsibilities

- Creates KeyStore instance with specified type (JKS, PKCS12).
- Loads KeyStore data containing client private key.
- Decrypts KeyStore using password.
- Used by **KeyManagerFactory** for client authentication.

---

### 2.3 KeyManagerFactory

#### Purpose

Creates KeyManagers that provide client certificate for mTLS authentication.

#### Dependencies

```java
javax.net.ssl.KeyManagerFactory
java.security.KeyStore
```

#### Actual Usage

```java
KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(
        Optional.ofNullable(keyManagerFactoryAlgorithm).orElse(KeyManagerFactory.getDefaultAlgorithm()));
char[] charKeyPswd = keyPassword.toCharArray();
keyManagerFactory.init(keyStore, charKeyPswd);
```

#### Responsibilities

- Creates KeyManagerFactory instance with specified algorithm.
- Initializes with KeyStore containing client private key.
- Uses key password to unlock private key entry.
- Provides KeyManagers for TLS context.
- Used by **SslContextBuilder** to configure client certificate presentation.

#### Configuration Source

```java
protected KeyManagerFactory loadKeyManagerFactory(String keyStorePath, String keyStorePassword, 
        String keyStoreType, String keyManagerFactoryAlgorithm, String keyPassword) {
    KeyManagerFactory keyManagerFactory = null;

    try (FileInputStream keystoreInputStream = new FileInputStream(keyStorePath)) {
        KeyStore keyStore = KeyStore.getInstance(
                Optional.ofNullable(keyStoreType).orElse(KeyStore.getDefaultType()));
        char[] charKeyStorePswd = keyStorePassword.toCharArray();
        keyStore.load(keystoreInputStream, charKeyStorePswd);
        keyManagerFactory = KeyManagerFactory.getInstance(
                Optional.ofNullable(keyManagerFactoryAlgorithm).orElse(KeyManagerFactory.getDefaultAlgorithm()));
        char[] charKeyPswd = keyPassword.toCharArray();
        keyManagerFactory.init(keyStore, charKeyPswd);
    } catch (KeyStoreException | IOException | NoSuchAlgorithmException | CertificateException
            | UnrecoverableKeyException e) {
        logger.error("Load KeyStore from {}", keyStorePath, e);
    }

    return keyManagerFactory;
}
```

---

### 2.4 FileInputStream (TrustStore)

#### Purpose

Loads the TrustStore file containing trusted CA certificates.

#### Dependencies

```java
java.io.FileInputStream
```

#### Actual Usage

```java
try (FileInputStream truststoreInputStream = new FileInputStream(trustStorePath)) {
    // Load TrustStore from inputStream
}
```

#### Responsibilities

- Locates TrustStore file by path.
- Opens file as InputStream.
- Provides byte stream to KeyStore loader.

---

### 2.5 KeyStore (TrustStore)

#### Purpose

Acts as the secure container that stores trusted CA certificates for server validation.

#### Dependencies

```java
java.security.KeyStore
java.io.FileInputStream
```

#### Actual Usage

```java
KeyStore trustStore = KeyStore.getInstance(
        Optional.ofNullable(trustStoreType).orElse(KeyStore.getDefaultType()));
trustStore.load(truststoreInputStream, trustStorePassword.toCharArray());
```

#### Responsibilities

- Creates KeyStore instance with specified type.
- Loads TrustStore data containing CA certificates.
- Decrypts TrustStore using password.
- Used by **TrustManagerFactory** for server certificate validation.

---

### 2.6 TrustManagerFactory

#### Purpose

Creates TrustManagers that validate server certificates against the TrustStore.

#### Dependencies

```java
javax.net.ssl.TrustManagerFactory
java.security.KeyStore
```

#### Actual Usage

```java
TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(
        Optional.ofNullable(trustManagerFactoryAlgorithm).orElse(TrustManagerFactory.getDefaultAlgorithm()));
trustManagerFactory.init(trustStore);
```

#### Responsibilities

- Creates TrustManagerFactory instance with specified algorithm.
- Initializes with TrustStore containing trusted CA certificates.
- Provides TrustManagers for TLS context.
- Used by **SslContextBuilder** to configure server certificate validation.

#### Configuration Source

```java
protected TrustManagerFactory loadTrustManagerFactory(String trustStorePath, String trustStorePassword,
        String trustStoreType, String trustManagerFactoryAlgorithm) {
    TrustManagerFactory trustManagerFactory = null;

    try (FileInputStream truststoreInputStream = new FileInputStream(trustStorePath)) {
        KeyStore trustStore = KeyStore.getInstance(
                Optional.ofNullable(trustStoreType).orElse(KeyStore.getDefaultType()));
        trustStore.load(truststoreInputStream, trustStorePassword.toCharArray());
        trustManagerFactory = TrustManagerFactory.getInstance(
                Optional.ofNullable(trustManagerFactoryAlgorithm).orElse(TrustManagerFactory.getDefaultAlgorithm()));
        trustManagerFactory.init(trustStore);
    } catch (KeyStoreException | IOException | NoSuchAlgorithmException | CertificateException e) {
        logger.error("Load TrustStore from {}", trustStorePath, e);
    }

    return trustManagerFactory;
}
```

---

### 2.7 SslContextBuilder (mTLS)

#### Purpose

Builds Netty SslContext with both KeyManager and TrustManager for mTLS.

#### Dependencies

```java
io.netty.handler.ssl.SslContext
io.netty.handler.ssl.SslContextBuilder
javax.net.ssl.KeyManagerFactory
javax.net.ssl.TrustManagerFactory
```

#### Actual Usage

```java
SslContext sslContext = SslContextBuilder.forClient()
        .keyManager(keyManagerFactory)
        .trustManager(trustManagerFactory)
        .build();
```

#### Responsibilities

- Creates TLS context for client connections.
- Configures KeyManager for client certificate presentation.
- Configures TrustManager for server certificate validation.
- Returns SslContext used by HttpClient.

---

### 2.8 ConnectionProvider

#### Purpose

Manages HTTP connection pool for WebClient.

#### Dependencies

```java
reactor.netty.resources.ConnectionProvider
```

#### Actual Usage

```java
ConnectionProvider connectionProvider = ConnectionProvider.builder(name)
        .maxConnections(maxConnection)
        .pendingAcquireTimeout(Duration.ofMillis(pendingAcquireTimeout))
        .maxIdleTime(Duration.ofSeconds(maxIdleTime))
        .build();
```

#### Responsibilities

- Creates named connection pool.
- Configures maximum connections.
- Configures pending acquire timeout.
- Configures maximum idle time.
- Used by **HttpClient.create()** for connection management.

#### Configuration Source

```java
protected ConnectionProvider buildConnectionProvider(String name) {
    return ConnectionProvider.builder(name)
            .maxConnections(maxConnection)
            .pendingAcquireTimeout(Duration.ofMillis(pendingAcquireTimeout))
            .maxIdleTime(Duration.ofSeconds(maxIdleTime))
            .build();
}
```

---

### 2.9 HttpClient (mTLS)

#### Purpose

Reactor Netty HTTP client configured with mTLS context.

#### Dependencies

```java
reactor.netty.http.client.HttpClient
reactor.netty.resources.ConnectionProvider
io.netty.handler.ssl.SslContextBuilder
io.netty.channel.ChannelOption
reactor.netty.transport.ProxyProvider.Proxy
```

#### Actual Usage

```java
HttpClient httpClient = HttpClient.create(buildConnectionProvider("gw-" + gatewayId))
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectionTimeout)
        .responseTimeout(Duration.ofMillis(gateway.getTimeout()));

if (viaWebProxy) {
    httpClient = httpClient.proxy(spec -> spec.type(Proxy.HTTP).host(proxyHost).port(proxyPort)
            .connectTimeoutMillis(connectionTimeout));
}

if (enableMTLS) {
    httpClient = httpClient.secure(sslProviderBuilder -> {
        try {
            sslProviderBuilder.sslContext(SslContextBuilder.forClient()
                    .keyManager(loadKeyManagerFactory(keyStorePath, keyStorePassword, keyStoreType,
                            keyManagerFactoryAlgorithm, keyPassword))
                    .trustManager(loadTrustManagerFactory(trustStorePath, trustStorePassword,
                            trustStoreType, trustManagerFactoryAlgorithm))
                    .build());
        } catch (SSLException e) {
            logger.error("load key and cert for gateway {}", gateway.getName(), e);
        }
    });
}
```

#### Responsibilities

- Creates reactive HTTP client with connection pool.
- Configures connection timeout.
- Configures response timeout.
- Configures proxy settings (optional).
- Configures mTLS with KeyManager and TrustManager.
- Used by **ReactorClientHttpConnector** to bridge to WebClient.

---

### 2.10 WebClient (mTLS)

#### Purpose

Spring reactive HTTP client configured with mTLS for secure API calls.

#### Dependencies

```java
org.springframework.web.reactive.function.client.WebClient
org.springframework.http.client.reactive.ReactorClientHttpConnector
```

#### Actual Usage

```java
return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .baseUrl(gateway.getCommunicationUrl()
                .concat(StringUtils.isNotBlank(gateway.getCommunicationPort())
                        ? ":".concat(gateway.getCommunicationPort())
                        : StringUtils.EMPTY))
        .build();
```

#### Responsibilities

- Provides reactive HTTP client API.
- Configures base URL for API calls.
- Applies mTLS via HttpClient connector.
- Used by application services to make secure HTTP calls.

---

## Execution Flow (mTLS)

1. **ConnectionProvider** is created for connection pooling:

   ```java
   ConnectionProvider connectionProvider = ConnectionProvider.builder("gw-" + gatewayId)
           .maxConnections(maxConnection)
           .pendingAcquireTimeout(Duration.ofMillis(pendingAcquireTimeout))
           .maxIdleTime(Duration.ofSeconds(maxIdleTime))
           .build();
   ```

2. **HttpClient.create()** creates HTTP client with connection pool:

   ```java
   HttpClient httpClient = HttpClient.create(connectionProvider)
           .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectionTimeout)
           .responseTimeout(Duration.ofMillis(responseTimeout));
   ```

3. **HttpClient.proxy()** configures proxy (if required):

   ```java
   httpClient = httpClient.proxy(spec -> spec.type(Proxy.HTTP).host(proxyHost).port(proxyPort)
           .connectTimeoutMillis(connectionTimeout));
   ```

4. **loadKeyManagerFactory()** loads client certificate:

   ```java
   KeyStore keyStore = KeyStore.getInstance(keyStoreType);
   keyStore.load(keystoreInputStream, keyStorePassword.toCharArray());
   KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(keyManagerFactoryAlgorithm);
   keyManagerFactory.init(keyStore, keyPassword.toCharArray());
   ```

5. **loadTrustManagerFactory()** loads trusted certificates:

   ```java
   KeyStore trustStore = KeyStore.getInstance(trustStoreType);
   trustStore.load(truststoreInputStream, trustStorePassword.toCharArray());
   TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(trustManagerFactoryAlgorithm);
   trustManagerFactory.init(trustStore);
   ```

6. **HttpClient.secure()** configures mTLS:

   ```java
   httpClient = httpClient.secure(sslProviderBuilder -> {
       sslProviderBuilder.sslContext(SslContextBuilder.forClient()
               .keyManager(keyManagerFactory)
               .trustManager(trustManagerFactory)
               .build());
   });
   ```

7. **WebClient.builder()** creates WebClient with mTLS-enabled HTTP client:

   ```java
   WebClient webClient = WebClient.builder()
           .clientConnector(new ReactorClientHttpConnector(httpClient))
           .baseUrl(baseUrl)
           .build();
   ```

---

# Part 3: ExchangeFilterFunction (Request/Response Logging)

## Purpose

Provides request and response logging capabilities for WebClient.

## Components

### 3.1 Request Logging Filter

#### Purpose

Logs outbound HTTP requests including method, URL, headers, and body.

#### Dependencies

```java
org.springframework.web.reactive.function.client.ExchangeFilterFunction
org.springframework.web.reactive.function.client.ClientRequest
org.springframework.http.client.reactive.ClientHttpRequestDecorator
```

#### Actual Usage

```java
public ExchangeFilterFunction logRequest() {
    return (clientRequest, next) -> {
        logger.info("Request: " + clientRequest.method() + " " + clientRequest.url());
        clientRequest.headers().forEach((name, values) -> values.forEach(value -> logger.info(name + "=" + value)));

        ClientRequest loggedRequest = ClientRequest.from(clientRequest)
                .body((outputMessage, context) -> {
                    ClientHttpRequestDecorator decorator = new ClientHttpRequestDecorator(outputMessage) {
                        @Override
                        public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                            return super.writeWith(Flux.from(body).doOnNext(buffer -> {
                                byte[] bytes = new byte[buffer.readableByteCount()];
                                buffer.toByteBuffer().get(bytes);
                                logger.info("Body: " + new String(bytes, StandardCharsets.UTF_8));
                            }));
                        }
                    };
                    return clientRequest.body().insert(decorator, context);
                })
                .build();

        return next.exchange(loggedRequest);
    };
}
```

#### Responsibilities

- Logs HTTP method and URL.
- Logs request headers.
- Logs request body without consuming it.
- Passes request to next filter.

---

### 3.2 Response Logging Filter

#### Purpose

Logs inbound HTTP responses including headers and error bodies.

#### Dependencies

```java
org.springframework.web.reactive.function.client.ExchangeFilterFunction
```

#### Actual Usage

```java
public ExchangeFilterFunction logResponse() {
    return ExchangeFilterFunction.ofResponseProcessor(clientResponse -> {
        clientResponse.headers().asHttpHeaders()
                .forEach((name, values) -> values.forEach(value -> logger.info(name + ": " + value)));
        if (!HttpStatus.OK.equals(clientResponse.statusCode())) {
            return clientResponse.bodyToMono(String.class).flatMap(body -> {
                logger.info("Error response: " + body);
                return Mono.error(new Exception(body));
            });
        }
        return Mono.just(clientResponse);
    });
}
```

#### Responsibilities

- Logs response headers.
- Logs error response body for non-200 status.
- Returns error Mono for non-200 responses.

---

# Summary

## Comparison of TLS Patterns

| Pattern | KeyStore | TrustStore | Use Case |
|---------|----------|------------|----------|
| One-Way TLS | Not Required | Required | Client trusts server only |
| Mutual TLS (mTLS) | Required | Required | Both client and server authenticate |

## Component Summary

| Component | Purpose | Used In |
|-----------|---------|---------|
| KeyStore | Stores private key and certificates | mTLS client authentication |
| TrustStore | Stores trusted CA certificates | Server certificate validation |
| KeyManagerFactory | Provides client certificate | mTLS client authentication |
| TrustManagerFactory | Validates server certificate | TLS handshake |
| SslContextBuilder | Builds TLS context | Netty HttpClient configuration |
| ConnectionProvider | Manages connection pool | HttpClient creation |
| HttpClient | Reactor Netty HTTP client | WebClient transport |
| ReactorClientHttpConnector | Bridges HttpClient to WebClient | WebClient configuration |
| WebClient | Spring reactive HTTP client | Application HTTP calls |

## Properties Reference

### Proxy Configuration

```properties
webproxy.host=proxy.example.com
webproxy.port=8080
webproxy.connection.timeout=30000
```

### Connection Pool Configuration

```properties
webclient.pool.max.connection=16
webclient.pool.pending.acquire.timeout=1000
webclient.pool.max.idle.time=30
```

### Timeout Configuration

```properties
response.read.timeout=30000
oauth2.client.connect.timeout=5000
oauth2.client.request.timeout=3000
oauth2.client.response.timeout=3000
```

### KeyStore Configuration (mTLS)

```properties
webproxy.keystore.path=/path/to/keystore.jks
webproxy.keystore.type=JKS
webproxy.keystore.password=keystorePassword
webproxy.keystore.key.manager.factory.algorithm=SunX509
ssl.key.password=keyPassword
```

### TrustStore Configuration

```properties
webproxy.truststore.path=/path/to/truststore.jks
webproxy.truststore.type=JKS
webproxy.truststore.password=truststorePassword
webproxy.truststore.trust.manager.factory.algorithm=SunX509
```

### One-Way TLS Configuration

```properties
ssl.trust-store=/path/to/truststore.jks
ssl.trust-store.password=truststorePassword
```

---

## Bean Configuration Example

### One-Way TLS WebClient

```java
@Configuration
public class KeyStoreConfiguration {

    @Value("${ssl.trust-store}")
    String trustStorePath;
    
    @Value("${ssl.trust-store.password}")
    String trustStorePass;

    @Bean
    public KeyStore trustStore() throws Exception {
        try (InputStream inputStream = new FileInputStream(ResourceUtils.getFile(trustStorePath))) {
            KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
            trustStore.load(inputStream, trustStorePass.toCharArray());
            return trustStore;
        }
    }

    @Bean
    public SslContext oneWayTlsContext(KeyStore trustStore) throws Exception {
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("SunX509");
        trustManagerFactory.init(trustStore);

        return SslContextBuilder.forClient()
                .trustManager(trustManagerFactory)
                .build();
    }
}

@Configuration
public class APIGatewayClient {

    @Bean
    public HttpClient httpClient(SslContext oneWayTlsContext) {
        return HttpClient.create()
                .secure(sslSpec -> sslSpec.sslContext(oneWayTlsContext))
                .proxy(proxy -> proxy
                        .type(ProxyProvider.Proxy.HTTP)
                        .host(proxyHost)
                        .port(proxyPort));
    }

    @Bean(name = "token-client")
    public WebClient createTokenWebClient(SslContext oneWayTlsContext) {
        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient(oneWayTlsContext)))
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON.toString())
                .build();
    }
}
```

### mTLS WebClient

```java
@Service
public class GatewayWebClientBuilder {

    @Value("${webproxy.host}")
    protected String proxyHost;

    @Value("${webproxy.port}")
    protected Integer proxyPort;

    @Value("${webproxy.connection.timeout}")
    protected Integer connectionTimeout;

    @Value("${webclient.pool.max.connection:16}")
    private Integer maxConnection;

    @Value("${webclient.pool.pending.acquire.timeout:1000}")
    private Integer pendingAcquireTimeout;

    @Value("${webclient.pool.max.idle.time:30}")
    private Integer maxIdleTime;

    @Value("${webproxy.keystore.path}")
    private String keyStorePath;

    @Value("${webproxy.keystore.type}")
    private String keyStoreType;

    @Value("${webproxy.keystore.key.manager.factory.algorithm}")
    private String keyManagerFactoryAlgorithm;

    @Value("${webproxy.truststore.path}")
    protected String trustStorePath;

    @Value("${webproxy.truststore.type}")
    protected String trustStoreType;

    @Value("${webproxy.truststore.trust.manager.factory.algorithm}")
    protected String trustManagerFactoryAlgorithm;

    @Value("${webproxy.keystore.password}")
    private String keyStorePassword;

    @Value("${webproxy.truststore.password}")
    protected String trustStorePassword;

    @Value("${ssl.key.password}")
    private String keyPassword;

    public WebClient buildWebClient(Long gatewayId, boolean viaWebProxy, boolean enableMTLS) {
        HttpClient httpClient = HttpClient.create(buildConnectionProvider("gw-" + gatewayId))
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectionTimeout)
                .responseTimeout(Duration.ofMillis(responseTimeout));

        if (viaWebProxy) {
            httpClient = httpClient.proxy(spec -> spec.type(Proxy.HTTP).host(proxyHost).port(proxyPort));
        }

        if (enableMTLS) {
            httpClient = httpClient.secure(sslProviderBuilder -> {
                sslProviderBuilder.sslContext(SslContextBuilder.forClient()
                        .keyManager(loadKeyManagerFactory(keyStorePath, keyStorePassword, keyStoreType,
                                keyManagerFactoryAlgorithm, keyPassword))
                        .trustManager(loadTrustManagerFactory(trustStorePath, trustStorePassword,
                                trustStoreType, trustManagerFactoryAlgorithm))
                        .build());
            });
        }

        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .baseUrl(baseUrl)
                .filter(logRequest())
                .filter(logResponse())
                .build();
    }

    protected ConnectionProvider buildConnectionProvider(String name) {
        return ConnectionProvider.builder(name)
                .maxConnections(maxConnection)
                .pendingAcquireTimeout(Duration.ofMillis(pendingAcquireTimeout))
                .maxIdleTime(Duration.ofSeconds(maxIdleTime))
                .build();
    }

    protected KeyManagerFactory loadKeyManagerFactory(String keyStorePath, String keyStorePassword,
            String keyStoreType, String keyManagerFactoryAlgorithm, String keyPassword) {
        KeyManagerFactory keyManagerFactory = null;

        try (FileInputStream keystoreInputStream = new FileInputStream(keyStorePath)) {
            KeyStore keyStore = KeyStore.getInstance(
                    Optional.ofNullable(keyStoreType).orElse(KeyStore.getDefaultType()));
            keyStore.load(keystoreInputStream, keyStorePassword.toCharArray());
            keyManagerFactory = KeyManagerFactory.getInstance(
                    Optional.ofNullable(keyManagerFactoryAlgorithm).orElse(KeyManagerFactory.getDefaultAlgorithm()));
            keyManagerFactory.init(keyStore, keyPassword.toCharArray());
        } catch (Exception e) {
            logger.error("Load KeyStore from {}", keyStorePath, e);
        }

        return keyManagerFactory;
    }

    protected TrustManagerFactory loadTrustManagerFactory(String trustStorePath, String trustStorePassword,
            String trustStoreType, String trustManagerFactoryAlgorithm) {
        TrustManagerFactory trustManagerFactory = null;

        try (FileInputStream truststoreInputStream = new FileInputStream(trustStorePath)) {
            KeyStore trustStore = KeyStore.getInstance(
                    Optional.ofNullable(trustStoreType).orElse(KeyStore.getDefaultType()));
            trustStore.load(truststoreInputStream, trustStorePassword.toCharArray());
            trustManagerFactory = TrustManagerFactory.getInstance(
                    Optional.ofNullable(trustManagerFactoryAlgorithm).orElse(TrustManagerFactory.getDefaultAlgorithm()));
            trustManagerFactory.init(trustStore);
        } catch (Exception e) {
            logger.error("Load TrustStore from {}", trustStorePath, e);
        }

        return trustManagerFactory;
    }
}
```
