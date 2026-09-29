# OAuth2 Dependencies Handle

## Components

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e114e6ed-22e6-4094-af43-0cdb1064bb21" />

### 1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager

#### Purpose

Acts as the central OAuth2 authorization and token lifecycle manager.

#### Dependencies

```java
ReactiveClientRegistrationRepository
ReactiveOAuth2AuthorizedClientService
ClientCredentialsReactiveOAuth2AuthorizedClientProvider
```

#### Actual Usage

```java
AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager manager =
        new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(reactiveClientRegistrationRepository, clientService);
manager.setAuthorizedClientProvider(provider);

this.authorizedClientManager = manager;
```

Authorize request:

```java
return authorizedClientManager.authorize(authorizeRequest)
        .map(OAuth2AuthorizedClient::getAccessToken)
        .map(OAuth2AccessToken::getTokenValue);
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

#### Actual Usage

Wrapping blocking repository to reactive:

```java
ReactiveClientRegistrationRepository reactiveClientRegistrationRepository =
        registrationId -> Mono.justOrEmpty(clientRegistrationRepository.findByRegistrationId(registrationId));
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

```properties
spring.security.oauth2.client.registration.partner.provider=partner
spring.security.oauth2.client.provider.partner.token-uri=${OAUTH2_TOKEN_URI}
spring.security.oauth2.client.registration.partner.authorization-grant-type=client_credentials
spring.security.oauth2.client.registration.partner.client-id=${OAUTH2_CLIENT_ID}
spring.security.oauth2.client.registration.partner.client-authentication-method=none
spring.security.oauth2.client.registration.partner.scope=${OAUTH2_SCOPE}
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

#### Actual Usage

```java
ReactiveOAuth2AuthorizedClientService clientService =
        new InMemoryReactiveOAuth2AuthorizedClientService(reactiveClientRegistrationRepository);
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

#### Actual Usage

```java
ClientCredentialsReactiveOAuth2AuthorizedClientProvider provider =
        new ClientCredentialsReactiveOAuth2AuthorizedClientProvider();
provider.setAccessTokenResponseClient(this.tokenResponseClient);
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

#### Actual Usage

```java
private final WebClientReactiveClientCredentialsTokenResponseClient tokenResponseClient = 
        new WebClientReactiveClientCredentialsTokenResponseClient();
```

Configure WebClient:

```java
this.tokenResponseClient.setWebClient(this.webClient);
```

#### Responsibilities

- Integrates **WebClient** (HTTP client for token requests).

- Sends HTTP POST request to token endpoint.

- Supplies Client ID and Client Secret in request.

- Receives OAuth2 token response.

- Converts response into **OAuth2AccessTokenResponse**.

- Used by **ClientCredentialsReactiveOAuth2AuthorizedClientProvider** to execute token requests.

---

### 6. WebClient

#### Purpose

HTTP client used for OAuth2 token requests and partner business API requests.

#### Dependencies

```java
HttpClient
```

#### Actual Usage

```java
this.webClient = gatewayWebClientBuilder.buildWebClient(partner, viaWebProxy, enableSSL);
this.tokenResponseClient.setWebClient(this.webClient);
```

#### Responsibilities

- Sends OAuth2 token requests to token endpoint.

- Sends business API requests to partner.

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

#### Actual Usage

```java
OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
        .withClientRegistrationId(clientRegistrationId)
        .principal(PRINCIPAL_NAME)
        .build();
```

Where constants are defined as:

```java
private static final String PRINCIPAL_NAME = "partner-reversal";
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

#### Actual Usage

```java
return authorizedClientManager.authorize(authorizeRequest)
        .map(OAuth2AuthorizedClient::getAccessToken)
        .map(OAuth2AccessToken::getTokenValue);
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

#### Actual Usage

```java
return authorizedClientManager.authorize(authorizeRequest)
        .map(OAuth2AuthorizedClient::getAccessToken)
        .map(OAuth2AccessToken::getTokenValue);
```

#### Responsibilities

- Holds bearer token value.

- Holds issue time.

- Holds expiry time.

- Used when calling protected APIs via **HttpHeaders.setBearerAuth()**.

- Returned from **OAuth2AuthorizedClient.getAccessToken()**.

---

## How OAuth2 Token Works Inside Manager
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/c44a283f-d2e9-4fb8-a10f-39d3f00ab96c" />

---

## Execution Flow

1. **APIGateway** (caller method) builds **OAuth2AuthorizeRequest** using builder:

   ```java
   OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
           .withClientRegistrationId(clientRegistrationId)
           .principal(PRINCIPAL_NAME)
           .build();
   ```

2. **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager.authorize(OAuth2AuthorizeRequest)** is called:

   ```java
   return authorizedClientManager.authorize(authorizeRequest)
           .map(OAuth2AuthorizedClient::getAccessToken)
           .map(OAuth2AccessToken::getTokenValue);
   ```

   - Manager integrates **ReactiveClientRegistrationRepository** to load **ClientRegistration**.

   - Manager integrates **ReactiveOAuth2AuthorizedClientService** to check for existing **OAuth2AuthorizedClient**.

3. **ReactiveOAuth2AuthorizedClientService.loadAuthorizedClient()** checks cache:

   - If **OAuth2AuthorizedClient** exists and token is valid → return cached client (skip to step 9).

   - If not found or token expired → proceed to step 4.

4. **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** delegates to **ClientCredentialsReactiveOAuth2AuthorizedClientProvider.authorize()**:

   ```java
   ClientCredentialsReactiveOAuth2AuthorizedClientProvider provider =
           new ClientCredentialsReactiveOAuth2AuthorizedClientProvider();
   provider.setAccessTokenResponseClient(this.tokenResponseClient);
   ```

   - Provider integrates **ClientRegistration** (client credentials).

   - Provider determines new token is required.

5. **ClientCredentialsReactiveOAuth2AuthorizedClientProvider** delegates to **WebClientReactiveClientCredentialsTokenResponseClient.getTokenResponse()**:

   ```java
   this.tokenResponseClient.setWebClient(this.webClient);
   ```

   - Token client integrates **WebClient** (HTTP client).

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

   ```java
   ReactiveOAuth2AuthorizedClientService clientService =
           new InMemoryReactiveOAuth2AuthorizedClientService(reactiveClientRegistrationRepository);
   ```

   - Caches **OAuth2AuthorizedClient** for future reuse.

   - Keyed by registration ID and principal name.

9. **AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager** returns **OAuth2AuthorizedClient** to caller.

10. **APIGateway** extracts token and calls business API:

    ```java
    public Mono<String> getOAuth2AccessToken(PartnerEntity partner, String clientRegistrationId) {
        createWebClient(partner);

        OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
                .withClientRegistrationId(clientRegistrationId)
                .principal(PRINCIPAL_NAME)
                .build();

        return authorizedClientManager.authorize(authorizeRequest)
                .map(OAuth2AuthorizedClient::getAccessToken)
                .map(OAuth2AccessToken::getTokenValue);
    }
    ```

11. **APIGateway** sends business request with Bearer token:

    ```java
    public Mono<ApiResponse> sendRequest(Mono<String> token, ApiRequest request) {
        return token.flatMap(accessToken ->
                this.webClient
                .post()
                .uri(apiUrl)
                .headers(h -> {
                    h.setBearerAuth(accessToken);
                    h.set(HEADER_MARKET, market);
                })
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .bodyValue(request)
                .retrieve()
                .bodyToMono(ApiResponse.class));
    }
    ```

12. **WebClient** returns **ApiResponse** to **APIGateway** for processing.
