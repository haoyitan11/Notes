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

## QR code generate dependencies handle
Zxing


