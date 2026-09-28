# Oauth2 dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aa4769e4-7344-476c-92a6-0bf5d30f6c1d" />

### 1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
Purpose : Acts as the central OAuth2 lifecycle manager.

Functions Used : 

Constructor:
```java
new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(clientRegistrationRepository,authorizedClientService)
```
Configure provider : 
```java
authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);
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

Common Implementation :
```java
InMemoryReactiveClientRegistrationRepository
```

Functions Used : 

Find registration:
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
Purpose : Acts as the OAuth2 authorized client storage layer.

Common Implementation :
```java
InMemoryReactiveOAuth2AuthorizedClientService
```

Functions Used :

Load authorized client:
```java
loadAuthorizedClient(registrationId,principalName)
```

Save authorized client: 
```java
saveAuthorizedClient(authorizedClient,principal)
```

Remove authorized client:
```java
removeAuthorizedClient(registrationId,principalName)
```

Responsibilities
- Stores OAuth2AuthorizedClient
- Retrieves OAuth2AuthorizedClient
- Maintains access token information
- Supports token reuse
- Supports removal of expired or invalid clients

### 4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider
Purpose : Handles the OAuth2 Client Credentials grant flow.

Function used : 

Configure access token client:
```java
setAccessTokenResponseClient(accessTokenResponseClient)
```

Responsibilities
- Determines whether authorization is required
- Checks token expiry status
- Requests new access tokens
- Delegates token requests to a ReactiveOAuth2AccessTokenResponseClient
- Creates a new OAuth2AuthorizedClient

### 5. WebClientReactiveClientCredentialsTokenResponseClient
Purpose : Executes the OAuth2 token endpoint request.

Functions Used :

Request access token:
```java
getTokenResponse(clientCredentialsGrantRequest)
```

Responsibilities
- Sends HTTP POST request to token endpoint
- Supplies Client ID and Client Secret
- Receives OAuth2 token response
- Converts response into OAuth2AccessTokenResponse

### 6. OAuth2AuthorizedClient
Purpose : Represents an authorized OAuth2 client

Dependency :
```java
org.springframework.security.oauth2.client.OAuth2AuthorizedClient
```
Functions Used:

Get access token:
```java
authorizedClient.getAccessToken()
```

Get client registration:
```java
authorizedClient.getClientRegistration()
```

Responsibilities
- Stores client registration
- Stores access token
- Stores authorization information
- Returned after successful authorization

### 7. OAuth2AccessToken
Purpose : Represents the OAuth2 access token
Functions Used :

Get token value:
```java
accessToken.getTokenValue()
```
Check expiry:
```java
accessToken.getExpiresAt()
```

Responsibilities
- Holds bearer token value
- Holds issue time
- Holds expiry time
- Used when calling protected APIs

## How OAuth2 token works inside manager
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7ce2bebc-0708-4156-9ec9-731a7c467f02" />

