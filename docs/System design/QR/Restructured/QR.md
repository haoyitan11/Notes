# Payment Service Provider (PSP) QR System Design

## Architecture Overview

```text
Payment Service Provider (PSP) QR System
│
├── Messaging Layer
│   └── Solace Message Broker
│
├── Database Layer
│   ├── MySQL
│   ├── Hibernate / Spring Data JPA
│   ├── JdbcTemplate
│   └── MyBatis
│
├── Security Layer
│   ├── TLS WebClient
│   ├── OAuth2
│   ├── PrivateKey Loading
│   └── Certificate Loading
│
├── External Communication Layer
│   └── HTTP Clients
│
├── QR Processing Layer
│   └── ZXing QR Processing
│
└── Batch Processing Layer
    └── Spring Batch
```

---

# Solace Message Broker

## Core Infrastructure Components

1. JndiTemplate
2. JndiObjectFactoryBean
3. ConnectionFactory
4. CachingConnectionFactory
5. JndiDestinationResolver

## JMS Messaging Components

1. Producer JmsTemplate
2. SolaceMessageSender
3. SolaceMessageSenderOneWay
4. Receiver Queue
5. Receiver JmsTemplate
6. SolaceMessageReceiver

## Spring Integration Components

1. DefaultMessageListenerContainer
2. MessageDrivenChannelAdapter
3. Spring Integration Channels
4. JsonToObjectTransformer
5. ServiceActivator
6. @ServiceActivator Business Service
7. ObjectToJsonTransformer
8. OutboundChannelAdapter

## Runtime Components

1. Solace Broker (Topic / Queue)

## Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SolaceCompleteGuide.md

---

# Database

## MySQL

### Database Engine

1. MySQL

### DataSource Infrastructure

1. DatasourceConfiguration
2. DataSourceBuilder
3. springDatasource
4. batchDatasource
5. HikariCP
6. Jasypt
7. MySQL Connector/J

### Transaction Management

1. @Transactional
2. Transaction Propagation

---

## Hibernate / Spring Data JPA

### Components

1. Entity
2. Repository Interface
3. Spring Data JPA
4. EntityManager
5. Hibernate SessionFactory
6. JpaRepository
7. Service Layer

---

## JdbcTemplate

### Components

1. JdbcTemplate
2. NamedParameterJdbcTemplate
3. RowMapper
4. DAO / Repository
5. Service Layer

---

## MyBatis

### Components

1. Mapper Interface
2. Mapper XML
3. SqlSession
4. SqlSessionFactory
5. Dynamic SQL Provider
6. Service Layer

## Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/DatabaseAccessPatternsCompleteGuide.md

---

# Security

## TLS WebClient Dependencies Handle

### Part 1. One-Way TLS

1. FileInputStream & ResourceUtils
2. KeyStore (TrustStore)
3. TrustManagerFactory
4. SslContextBuilder
5. HttpClient
6. ReactorClientHttpConnector
7. WebClient

### Part 2. Mutual TLS (mTLS)

1. FileInputStream (KeyStore)
2. KeyStore (Client KeyStore)
3. KeyManagerFactory
4. FileInputStream (TrustStore)
5. KeyStore (TrustStore)
6. TrustManagerFactory
7. SslContextBuilder
8. ConnectionProvider
9. HttpClient
10. ReactorClientHttpConnector
11. WebClient

### Part 3. ExchangeFilterFunction

1. Request Logging Filter
2. Response Logging Filter

### Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/TLSWebClient.md

---

## OAuth2 Dependencies Handle

### Components

1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
2. ReactiveClientRegistrationRepository
3. ReactiveOAuth2AuthorizedClientService
4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider
5. WebClientReactiveClientCredentialsTokenResponseClient
6. WebClient
7. OAuth2AuthorizeRequest
8. OAuth2AuthorizedClient
9. OAuth2AccessToken

### Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/OAuth2.md

---

## PrivateKey Dependencies Handle

### Part 1. PEM File Loading (Bouncy Castle)

1. BouncyCastleProvider
2. Files & Paths
3. StringReader
4. PEMParser
5. PEMKeyPair
6. PrivateKeyInfo
7. JcaPEMKeyConverter
8. PrivateKey

### Part 2. Raw PKCS8 Binary File Loading

1. Files & Paths
2. PKCS8EncodedKeySpec
3. KeyFactory
4. PrivateKey

### Part 3. KeyStore Loading (JKS / PKCS12)

1. FileInputStream
2. KeyStore
3. KeyStore.getKey()
4. KeyStore.PrivateKeyEntry
5. PrivateKey

### Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/GetPrivateKey.md

---

## Certificate Dependencies Handle

### Part 1. PEM Certificate File Loading (Bouncy Castle)

1. BouncyCastleProvider
2. Files & Paths
3. PEMParser
4. X509CertificateHolder
5. JcaX509CertificateConverter
6. X509Certificate

### Part 2. KeyStore Certificate Loading (JKS / PKCS12)

1. FileInputStream
2. KeyStore
3. KeyStore.getCertificate()
4. KeyStore.getCertificateChain()
5. KeyStore.PrivateKeyEntry.getCertificateChain()
6. X509Certificate

### Part 3. DER / CER File Loading

1. FileInputStream
2. ByteArrayInputStream
3. CertificateFactory
4. X509Certificate

### Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/GetCertificate.md

---

# External Communication

## HTTP Client Dependencies Handle

### Components

1. Spring WebFlux (WebClient)
2. Apache HttpClient (CloseableHttpClient)
3. RestClient

---

# QR Processing

## ZXing QR Code Processing

### QR Payload Generation Components

1. DynamicQrCodeUtil
2. DateUtil
3. CRC16 Calculator
4. Helper Methods

### QR Image Generation Components

1. DynamicQrImageUtil
2. QRCodeWriter
3. BitMatrix
4. MatrixToImageWriter
5. MatrixToImageConfig
6. BufferedImage
7. Graphics2D
8. ImageIO

### Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/ZXing%20QR.md

---

# Batch Processing

## Spring Batch Job Processing

### Components

1. JobLauncher
2. JobParameters
3. Job
4. Step
5. Tasklet
6. JobRepository
7. JobExecution
8. JobBuilder
9. StepBuilder

---

## Spring Batch Chunk Processing

### Components

1. JobLauncher
2. JobParameters
3. Job
4. Chunk-Oriented Step
5. ItemReader
6. ItemProcessor
7. ItemWriter (FlatFileItemWriter)
8. FlatFileHeaderCallback
9. FlatFileFooterCallback
10. DataContainer
11. @JobScope
12. @StepScope

### Reference

https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SpringBatchCompleteGuide.md
