# Oauth2 dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aa4769e4-7344-476c-92a6-0bf5d30f6c1d" />

### 1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
#### Purpose 
Acts as the central OAuth2 authorization and token lifecycle manager.

#### Dependencies
```java
ReactiveClientRegistrationRepository
ReactiveOAuth2AuthorizedClientService
ClientCredentialsReactiveOAuth2AuthorizedClientProvider
```

#### Functions Used
Constructor :
```java
new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(ReactiveClientRegistrationRepository,ReactiveOAuth2AuthorizedClientService)
```
Configure authorization provider : 
```java
AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.setAuthorizedClientProvider(ClientCredentialsReactiveOAuth2AuthorizedClientProvider);
```
Authorize request : 
```java
AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.authorize(OAuth2AuthorizeRequest);
```

#### Responsibilities
- Central coordinator for OAuth2 authorization.
- Loads client configuration from ReactiveClientRegistrationRepository.
- Retrieves existing authorized clients from ReactiveOAuth2AuthorizedClientService.
- Checks token validity and expiration.
- Requests new access tokens when required.
- Uses ReactiveOAuth2AuthorizedClientService to store and retrieve authorized clients.

### 2. ReactiveClientRegistrationRepository
#### Purpose 
Provides OAuth2 client configuration information.

#### Dependencies
```java
ClientRegistrationRepository
```

#### Functions Used
Find registration : 
```java
ClientRegistrationRepository.findByRegistrationId("scb");
```

#### Responsibilities
- Loads ClientRegistration
- Provides Client ID
- Provides Client Secret
- Provides Authorization Grant Type
- Provides Token URI
- Provides OAuth2 metadata

#### Configuration Source
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

### 3. ReactiveOAuth2AuthorizedClientService
#### Purpose
Acts as the OAuth2 authorized client storage layer.

#### Implementation :
```java
InMemoryReactiveOAuth2AuthorizedClientService
```

#### Functions Used :
Load authorized client:
```java
ReactiveOAuth2AuthorizedClientService.loadAuthorizedClient(registrationId,principalName);
```

Save authorized client: 
```java
ReactiveOAuth2AuthorizedClientService.saveAuthorizedClient(OAuth2AuthorizedClient,principal);
```

Remove authorized client:
```java
ReactiveOAuth2AuthorizedClientService.removeAuthorizedClient(registrationId,principalName);
```

#### Responsibilities
- Stores OAuth2AuthorizedClient
- Retrieves OAuth2AuthorizedClient
- Maintains access token information
- Supports token reuse
- Removes expired or invalid clients
- Acts as token cache

### 4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider
#### Purpose
Handles the OAuth2 Client Credentials grant flow.

#### Dependencies
```java
WebClientReactiveClientCredentialsTokenResponseClient
```

#### Function used : 
Configure access token client:
```java
ClientCredentialsReactiveOAuth2AuthorizedClientProvider.setAccessTokenResponseClient(WebClientReactiveClientCredentialsTokenResponseClient);
```

#### Responsibilities
- Determines whether authorization is required
- Checks token expiry status
- Requests new access tokens
- Delegates token requests to a ReactiveOAuth2AccessTokenResponseClient
- Creates a new OAuth2AuthorizedClient

### 5. WebClientReactiveClientCredentialsTokenResponseClient
#### Purpose
Executes the OAuth2 token endpoint request.

#### Dependencies
```java
WebClient
```

#### Functions Used :
Configure WebClient
```java
WebClientReactiveClientCredentialsTokenResponseClient.setWebClient(WebClient);
```

Request access token
```java
WebClientReactiveClientCredentialsTokenResponseClient.getTokenResponse(OAuth2ClientCredentialsGrantRequest);
```

#### Responsibilities
- Sends HTTP POST request to token endpoint
- Supplies Client ID and Client Secret
- Uses MTLS-enabled WebClient
- Receives OAuth2 token response
- Converts response into OAuth2AccessTokenResponse

### 6. WebClient
#### Purpose
HTTP client used for OAuth2 token requests and SCB business API requests.

#### Functions Used
POST request
```java
WebClient.post();
```

Define endpoint
```java
WebClient.uri(reversalUrl);
```

Set Bearer token
```java
HttpHeaders.setBearerAuth(accessToken);
```

Set request body
```java
WebClient.bodyValue(ScbAPIRequest);
```

Retrieve response
```java
WebClient.retrieve();
```

Convert response
```java
WebClient.bodyToMono(ScbAPIResponse.class);
```

#### Responsibilities
- Sends OAuth2 token requests
- Sends SCB reversal requests
- Supports MTLS communication
- Supports Web Proxy routing
- Handles reactive HTTP communication

### 7. OAuth2AuthorizeRequest
#### Purpose
Represents an authorization request submitted to the OAuth2 manager.

#### Functions Used
Specify registration
```java
OAuth2AuthorizeRequest.withClientRegistrationId(clientRegistrationId);
```

Specify principal
```java
OAuth2AuthorizeRequest.principal(principalName);
```

Build request
```java
OAuth2AuthorizeRequest.build();
```

#### Responsibilities
- Identifies OAuth2 registration
- Identifies principal
- Triggers authorization process

### 8. OAuth2AuthorizedClient
#### Purpose
Represents a successfully authorized OAuth2 client.

#### Dependencies
```java
ClientRegistration
OAuth2AccessToken
```

#### Functions Used :
Get token value:
```java
OAuth2AuthorizedClient.getAccessToken();
```
Get registration:
```java
OAuth2AuthorizedClient.getClientRegistration();
```

#### Responsibilities
- Stores ClientRegistration
- Stores OAuth2AccessToken
- Returned after successful authorization
- Stores authorization information for a specific client registration and principal.

### 9. OAuth2AccessToken
#### Purpose
Represents the OAuth2 access token.

#### Functions Used
Get token value
```java
OAuth2AccessToken.getTokenValue();
```

Get expiry time
```java
OAuth2AccessToken.getExpiresAt();
```

#### Responsibilities
- Holds bearer token value
- Holds issue time
- Holds expiry time
- Used when calling protected APIs
- Returned from OAuth2AuthorizedClient.getAccessToken()

## How OAuth2 token works inside manager
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7ce2bebc-0708-4156-9ec9-731a7c467f02" />

