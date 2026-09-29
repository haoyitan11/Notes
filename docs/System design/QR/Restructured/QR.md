# Payment Service Provider QR System Design
## Solace Message Broker
### Components
#### Connection Layer
1. JndiTemplate
2. JndiObjectFactoryBean
3. CachingConnectionFactory

#### Messaging Layer
4. Producer JmsTemplate
5. SolaceMessageSender
6. MessageCreator
7. Session
8. Message (TextMessage)
9. Destination (Topic/Queue)

#### Consumer Layer
10. Receiver JmsTemplate
11. SolaceMessageReceiver

#### Spring Integration Layer
12. DefaultMessageListenerContainer
13. message-driven-channel-adapter
14. JsonMsgServiceImpl
15. outbound-channel-adapter

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/Solace.md

## Database
1. MySQL

## Database Query dependencies handle
1. JdbcTemplate <br>
2. Hibernate

## HTTP clients dependencies handle
1. Spring WebFlux (WebClient) 
2. Apache HttpClient (CloseableHttpClient)
3. RestClient (match with expcetion RestClientResponseException)

## Oauth2 dependencies handle
### Components
1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
2. ReactiveClientRegistrationRepository
3. ReactiveOAuth2AuthorizedClientService
4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider
5. WebClientReactiveClientCredentialsTokenResponseClien
6. WebClient
7. OAuth2AuthorizeRequest
8. OAuth2AuthorizedClient
9. OAuth2AccessToken

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/OAuth2.md

## Spring Batch dependencies handle
### Spring Batch Job components
1. JobLauncher
2. Job
3. JobParameters
4. JobExecution
5. JobRepository
6. JobBuilder
7. Step
8. StepBuilder
9. Tasklet

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SpringBatchJob.md

### Spring Batch Chunk components
1. Chunk-Oriented Step
2. ItemReader
3. ItemProcessor
4. ItemWriter (FlatFileItemWriter)
5. FlatFileHeaderCallback
6. FlatFileFooterCallback
7. DataContainer
8. @StepScope / @JobScope

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SpringBatchChunk.md

## PrivateKey dependencies handle
### Components
1. BouncyCastleProvider
2. Files & Paths
3. Reader (StringReader)
4. PEMParser
5. PEMKeyPair
6. PrivateKeyInfo
7. JcaPEMKeyConverter
8. PrivateKey

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/PrivateKey.md

## ZXing QR code generate dependencies handle
### Components
1. DynamicQrCodeUtil
2. DynamicQrImageUtil
3. QRCodeWriter
4. BitMatrix
5. MatrixToImageWriter
6. MatrixToImageConfig
7. BufferedImage
8. Graphics2D
9. ImageIO

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/ZXing%20QR.md

