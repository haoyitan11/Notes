# Payment Service Provider QR System Design
## Message Broker
1. Solace

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

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/OAuth2.md

## Spring Batch dependencies handle
### Components
1. JobLauncher
2. Job
3. JobParameters
4. JobExecution
5. JobRepository
6. JobBuilder
7. Step
8. StepBuilder
9. Tasklet

More details on https://github.com/haoyitan11/Notes/blob/main/docs/System%20design/QR/SpringBatch.md

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

## QR code generate dependencies handle
Zxing


