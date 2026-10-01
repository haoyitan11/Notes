# Payment Service Provider QR System Design

## Solace Message Broker

### Core Infrastructure Components

1. JndiTemplate
2. JndiObjectFactoryBean
3. ConnectionFactory
4. CachingConnectionFactory
5. JndiDestinationResolver

### JMS Messaging Components

1. Producer JmsTemplate
2. SolaceMessageSender
3. SolaceMessageSenderOneWay
4. Receiver Queue
5. Receiver JmsTemplate
6. SolaceMessageReceiver

### Spring Integration Components

1. DefaultMessageListenerContainer
2. MessageDrivenChannelAdapter
3. Spring Integration Channels
4. JsonToObjectTransformer
5. ServiceActivator
6. @ServiceActivator Business Service
7. ObjectToJsonTransformer
8. OutboundChannelAdapter

### Runtime

1. Solace Broker (Topic / Queue)

More details on: https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SolaceCompleteGuide.md

---

## Database

### Database

1. MySQL

### Database Query Dependencies Handle

1. JdbcTemplate
2. Hibernate

---

## HTTP Clients Dependencies Handle

### Components

1. Spring WebFlux (WebClient)
2. Apache HttpClient (CloseableHttpClient)
3. RestClient

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

More details on: https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/OAuth2.md

---

## Get PrivateKey Dependencies Handle

### Part 1. PEM File Loading (Bouncy Castle)

1. BouncyCastleProvider
2. Files & Paths
3. StringReader
4. PEMParser
5. PEMKeyPair
6. PrivateKeyInfo
7. JcaPEMKeyConverter
8. PrivateKey

### Part 2. Raw PKCS8 Binary File Loading (Native Java)

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

More details on: https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/GetPrivateKey.md

---

## Get Certificate Dependencies Handle

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

### Part 3. DER / CER File Loading (Native Java)

1. FileInputStream
2. ByteArrayInputStream
3. CertificateFactory
4. X509Certificate

More details on: https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/GetCertificate.md

---

## QR Processing

### ZXing QR Code Generate Dependencies Handle

#### QR Payload Generation Components

1. DynamicQrCodeUtil
2. DateUtil
3. CRC16 Calculator
4. Helper Methods

#### QR Image Generation Components

1. DynamicQrImageUtil
2. QRCodeWriter
3. BitMatrix
4. MatrixToImageWriter
5. MatrixToImageConfig
6. BufferedImage
7. Graphics2D
8. ImageIO

More details on: https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/ZXing%20QR.md

---

## Spring Batch Dependencies Handle

### Spring Batch Job Components

1. JobLauncher
2. JobParameters
3. Job
4. Step
5. Tasklet
6. JobRepository
7. JobExecution
8. JobBuilder
9. StepBuilder

### Spring Batch Chunk Components

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

More details on: https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SpringBatchCompleteGuide.md
