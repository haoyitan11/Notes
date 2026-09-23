# QR System Design
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
<img width="947" height="457" alt="image" src="https://github.com/user-attachments/assets/d6b592b4-df51-4c3d-8ad4-113e16a305b1" />

1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
Function: Acts as the central OAuth2 lifecycle manager.
- Integrates ReactiveClientRegistrationRepository, ReactiveOAuth2AuthorizedClientService and ClientCredentialsReactiveOAuth2AuthorizedClientProvider.
- Coordinates token retrieval, token reuse, token expiration checks, and authorized client storage.
  
2. ReactiveClientRegistrationRepository
- Loads the ClientRegistration based on the registrationId (for example: scb).
- Maps the registrationId to OAuth2 client configurations defined in .properties or .yaml.
- Provides OAuth2 metadata such as Client ID, Client Secret, Token Endpoint, Grant Type, and Scopes.

3. ReactiveOAuth2AuthorizedClientService
Function : Acts as the OAuth2 authorized client storage layer.
- Stores and retrieves OAuth2AuthorizedClient instances.
- In this implementation, InMemoryReactiveOAuth2AuthorizedClientService is used to store authorized clients and access tokens in JVM memory.

4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider
Function : Handles the OAuth2 Client Credentials grant flow.
- Determines whether a new access token needs to be requested.
- Delegates the token request to a ReactiveOAuth2AccessTokenResponseClient.
- Returns an OAuth2AuthorizedClient containing the newly acquired access token.

### How OAuth2 token works inside manager
<img width="940" height="414" alt="image" src="https://github.com/user-attachments/assets/90540499-b89a-4857-9150-02bffd5717ee" />
