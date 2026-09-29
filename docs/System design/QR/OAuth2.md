# OAuth2 Dependencies Handle

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/4c23852f-43b9-4113-8f95-7e4fe9434831" />

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

Constructor:

```java
new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(ReactiveClientRegistrationRepository, ReactiveOAuth2AuthorizedClientService);
```

Configure authorization provider:

```java
AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.setAuthorizedClientProvider(ClientCredentialsReactiveOAuth2AuthorizedClientProvider);
```

Authorize request:

```java
AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.authorize(OAuth2AuthorizeRequest);
```

#### Responsibilities

- Central coordinator for OAuth2 authorization.
- Integrates **ReactiveClientRegistrationRepository** (loads client configuration).
- Integrates **ReactiveOAuth2AuthorizedClientService** (stores and retrieves authorized clients).
- Integrates **ClientCredentialsReactiveOAuth2AuthorizedClientProvider** (handles token acquisition).
- Checks token validity and expiration.
- Requests new access tokens when required.
- Returns **OAuth2AuthorizedClient** containing the access token.

#### Configuration Source

```java
@PostConstruct
private void initAuthorizedClientManager() {
    ReactiveClientRegistrationRepository reactiveClientRegistrationRepository =
            registrationId -> Mono.justOrEmpty(clientRegistrationRepository.findByRegistrationId(registrationId));

    ClientCredentialsReactiveOAuth2AuthorizedClientProvider provider =
            new ClientCredentialsReactiveOAuth2AuthorizedClientProvider();
    provider.setAccessTokenResponseClient(this.tokenResponseClient);

    ReactiveOAuth2AuthorizedClientService clientService =
            new InMemoryReactiveOAuth2AuthorizedClientService(reactiveClientRegistrationRepository);

    AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager manager =
            new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(reactiveClientRegistrationRepository, clientService);
    manager.setAuthorizedClientProvider(provider);

    this.authorizedClientManager = manager;
}
```

---

### 2. ReactiveClientRegistrationRepository

#### Purpose

Provides OAuth2 client configuration information.

#### Dependencies

```java
ClientRegistrationRepository
```

#### Functions Used

Find registration:

```java
ReactiveClientRegistrationRepository.findByRegistrationId("partner-bank");
```

#### Responsibilities

- Integrates **ClientRegistrationRepository** (underlying blocking repository).
- Loads **ClientRegistration** containing:
  - Client ID
  - Client Secret
  - Authorization Grant Type
  - Token URI
  - OAuth2 metadata
- Used by **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** to load client configuration.

#### Configuration Source

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          partner-bank:
            client-id: ${OAUTH2_CLIENT_ID}
            client-secret: ${OAUTH2_CLIENT_SECRET}
            authorization-grant-type: client_credentials
        provider:
          partner-bank:
            token-uri: ${OAUTH2_TOKEN_URI}
```

---

### 3. ReactiveOAuth2AuthorizedClientService

#### Purpose

Acts as the OAuth2 authorized client storage layer.

#### Dependencies

```java
ReactiveClientRegistrationRepository
```

#### Implementation

```java
InMemoryReactiveOAuth2AuthorizedClientService
```

#### Functions Used

Load authorized client:

```java
ReactiveOAuth2AuthorizedClientService.loadAuthorizedClient(registrationId, principalName);
```

Save authorized client:

```java
ReactiveOAuth2AuthorizedClientService.saveAuthorizedClient(OAuth2AuthorizedClient, principal);
```

Remove authorized client:

```java
ReactiveOAuth2AuthorizedClientService.removeAuthorizedClient(registrationId, principalName);
```

#### Responsibilities

- Integrates **ReactiveClientRegistrationRepository** (for client registration lookup).
- Stores **OAuth2AuthorizedClient** instances.
- Retrieves **OAuth2AuthorizedClient** by registration ID and principal name.
- Maintains access token information for reuse.
- Removes expired or invalid clients.
- Acts as token cache.
- Used by **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** for token persistence.

---

### 4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider

#### Purpose

Handles the OAuth2 Client Credentials grant flow.

#### Dependencies

```java
WebClientReactiveClientCredentialsTokenResponseClient
```

#### Functions Used

Configure access token client:

```java
ClientCredentialsReactiveOAuth2AuthorizedClientProvider.setAccessTokenResponseClient(WebClientReactiveClientCredentialsTokenResponseClient);
```

Authorize:

```java
ClientCredentialsReactiveOAuth2AuthorizedClientProvider.authorize(OAuth2AuthorizationContext);
```

#### Responsibilities

- Integrates **WebClientReactiveClientCredentialsTokenResponseClient** (executes token requests).
- Determines whether authorization is required.
- Checks token expiry status.
- Requests new access tokens via token response client.
- Creates new **OAuth2AuthorizedClient** from token response.
- Used by **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** as the authorization provider.

---

### 5. WebClientReactiveClientCredentialsTokenResponseClient

#### Purpose

Executes the OAuth2 token endpoint request.

#### Dependencies

```java
WebClient
```

#### Functions Used

Configure WebClient:

```java
WebClientReactiveClientCredentialsTokenResponseClient.setWebClient(WebClient);
```

Request access token:

```java
WebClientReactiveClientCredentialsTokenResponseClient.getTokenResponse(OAuth2ClientCredentialsGrantRequest);
```

#### Responsibilities

- Integrates **WebClient** (HTTP client for token requests).
- Sends HTTP POST request to token endpoint.
- Supplies Client ID and Client Secret in request.
- Uses MTLS-enabled WebClient for secure communication.
- Receives OAuth2 token response.
- Converts response into **OAuth2AccessTokenResponse**.
- Used by **ClientCredentialsReactiveOAuth2AuthorizedClientProvider** to execute token requests.

---

### 6. WebClient

#### Purpose

HTTP client used for OAuth2 token requests and partner bank business API requests.

#### Dependencies

```java
SslContext
HttpClient
```

#### Functions Used

POST request:

```java
WebClient.post();
```

Define endpoint:

```java
WebClient.uri(apiEndpointUrl);
```

Set Bearer token:

```java
HttpHeaders.setBearerAuth(accessToken);
```

Set request body:

```java
WebClient.bodyValue(ApiRequest);
```

Retrieve response:

```java
WebClient.retrieve();
```

Convert response:

```java
WebClient.bodyToMono(ApiResponse.class);
```

#### Responsibilities

- Sends OAuth2 token requests to token endpoint.
- Sends business API requests to partner bank.
- Supports MTLS communication via **SslContext**.
- Supports Web Proxy routing.
- Handles reactive HTTP communication.
- Used by **WebClientReactiveClientCredentialsTokenResponseClient** for token requests.
- Used by **APIGateway** for business API calls.

---

### 7. OAuth2AuthorizeRequest

#### Purpose

Represents an authorization request submitted to the OAuth2 manager.

#### Dependencies

```java
OAuth2AuthorizeRequest.Builder
```

#### Functions Used

Specify registration:

```java
OAuth2AuthorizeRequest.withClientRegistrationId(clientRegistrationId);
```

Specify principal:

```java
OAuth2AuthorizeRequest.principal(principalName);
```

Build request:

```java
OAuth2AuthorizeRequest.build();
```

#### Responsibilities

- Identifies OAuth2 registration by registration ID.
- Identifies principal (user or service account).
- Triggers authorization process when passed to manager.
- Built by caller and passed to **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.authorize()**.

---

### 8. OAuth2AuthorizedClient

#### Purpose

Represents a successfully authorized OAuth2 client.

#### Dependencies

```java
ClientRegistration
OAuth2AccessToken
```

#### Functions Used

Get access token:

```java
OAuth2AuthorizedClient.getAccessToken();
```

Get registration:

```java
OAuth2AuthorizedClient.getClientRegistration();
```

Get principal name:

```java
OAuth2AuthorizedClient.getPrincipalName();
```

#### Responsibilities

- Integrates **ClientRegistration** (client configuration).
- Integrates **OAuth2AccessToken** (the bearer token).
- Stores authorization information for a specific client registration and principal.
- Returned by **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.authorize()**.
- Stored in **ReactiveOAuth2AuthorizedClientService** for token reuse.

---

### 9. OAuth2AccessToken

#### Purpose

Represents the OAuth2 access token.

#### Dependencies

```java
TokenType
Instant
```

#### Functions Used

Get token value:

```java
OAuth2AccessToken.getTokenValue();
```

Get expiry time:

```java
OAuth2AccessToken.getExpiresAt();
```

Get issued time:

```java
OAuth2AccessToken.getIssuedAt();
```

Get token type:

```java
OAuth2AccessToken.getTokenType();
```

#### Responsibilities

- Holds bearer token value.
- Holds issue time.
- Holds expiry time.
- Used when calling protected APIs via **HttpHeaders.setBearerAuth()**.
- Returned from **OAuth2AuthorizedClient.getAccessToken()**.

---

## How OAuth2 Token Works Inside Manager

![OAuth2 Token Flow Diagram](https://github.com/user-attachments/assets/7ce2bebc-0708-4156-9ec9-731a7c467f02)

---

## Execution Flow

1. **APIGateway** (caller method) builds **OAuth2AuthorizeRequest** using builder:
   - Sets **clientRegistrationId** (e.g., "partner-bank").
   - Sets **principalName** (e.g., "api-service").

2. **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.authorize(OAuth2AuthorizeRequest)** is called:
   - Manager integrates **ReactiveClientRegistrationRepository** to load **ClientRegistration**.
   - Manager integrates **ReactiveOAuth2AuthorizedClientService** to check for existing **OAuth2AuthorizedClient**.

3. **ReactiveOAuth2AuthorizedClientService.loadAuthorizedClient()** checks cache:
   - If **OAuth2AuthorizedClient** exists and token is valid → return cached client (skip to step 9).
   - If not found or token expired → proceed to step 4.

4. **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** delegates to **ClientCredentialsReactiveOAuth2AuthorizedClientProvider.authorize()**:
   - Provider integrates **ClientRegistration** (client credentials).
   - Provider determines new token is required.

5. **ClientCredentialsReactiveOAuth2AuthorizedClientProvider** delegates to **WebClientReactiveClientCredentialsTokenResponseClient.getTokenResponse()**:
   - Token client integrates **WebClient** (MTLS-enabled HTTP client).
   - Token client builds **OAuth2ClientCredentialsGrantRequest**.

6. **WebClientReactiveClientCredentialsTokenResponseClient** executes HTTP request:
   - Sends POST to token endpoint URI from **ClientRegistration**.
   - Includes Client ID and Client Secret.
   - Receives **OAuth2AccessTokenResponse** from token server.

7. **ClientCredentialsReactiveOAuth2AuthorizedClientProvider** creates **OAuth2AuthorizedClient**:
   - Integrates **ClientRegistration**.
   - Integrates **OAuth2AccessToken** from response.
   - Integrates principal name.

8. **ReactiveOAuth2AuthorizedClientService.saveAuthorizedClient()** stores the new client:
   - Caches **OAuth2AuthorizedClient** for future reuse.
   - Keyed by registration ID and principal name.

9. **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** returns **OAuth2AuthorizedClient** to caller.

10. **APIGateway** extracts token and calls business API:
    - **OAuth2AuthorizedClient.getAccessToken()** → **OAuth2AccessToken**.
    - **OAuth2AccessToken.getTokenValue()** → bearer token string.
    - **WebClient.post()** with **HttpHeaders.setBearerAuth(accessToken)** sends API request.

11. **WebClient** returns **ApiResponse** to **APIGateway** for processing.
