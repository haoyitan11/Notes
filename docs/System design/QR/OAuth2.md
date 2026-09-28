# Oauth2 dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aa4769e4-7344-476c-92a6-0bf5d30f6c1d" />

### 1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
Purpose : Acts as the central OAuth2 lifecycle manager.

Dependency : 
```java
org.springframework.security.oauth2.client.AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
```
Functions Used : 

Constructor:
```java
new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(
clientRegistrationRepository,
authorizedClientService
)
```
Configure provider : 
```java
authorizedClientManager.setAuthorizedClientProvider(
authorizedClientProvider
);
```
Authorize client : 
```java
authorizedClientManager.authorize(authorizeRequest)
```

Responsibilities :
- Central coordinator for OAuth2 authorization.
- Loads client configuration from ReactiveClientRegistrationRepository.
- Retrieves existing authorized clients from ReactiveOAuth2AuthorizedClientService.
- Checks token validity and expiration.
- Requests new access tokens when required.
- Stores newly authorized clients.

### 2. ReactiveClientRegistrationRepository
Purpose : Provides OAuth2 client configuration information.

Dependency :
```java
org.springframework.security.oauth2.client.registration.ReactiveClientRegistrationRepository
```
Common Implementation : InMemoryReactiveClientRegistrationRepository

Functions Used : 
- Find registration:
```java
findByRegistrationId("scb")
```

Responsibilities :
- Loads ClientRegistration
- Maps registrationId to client configuration
- Provides OAuth2 metadata

Configuration Source
```java
spring:
  security:
    oauth2:
      client:
        registration:
          scb:
            client-id: xxx
            client-secret: xxx
```

Information Returned :
```java
Client ID
Client Secret
Authorization Grant Type
Scopes
Token URI
Client 
```

### 3. ReactiveOAuth2AuthorizedClientService
Function : Acts as the OAuth2 authorized client storage layer.
- Stores and retrieves OAuth2AuthorizedClient instances.
- In this implementation, InMemoryReactiveOAuth2AuthorizedClientService is used to store authorized clients and access tokens in JVM memory.

### 4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider
Function : Handles the OAuth2 Client Credentials grant flow.
- Determines whether a new access token needs to be requested.
- Delegates the token request to a ReactiveOAuth2AccessTokenResponseClient.
- Returns an OAuth2AuthorizedClient containing the newly acquired access token.

## How OAuth2 token works inside manager
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7ce2bebc-0708-4156-9ec9-731a7c467f02" />

