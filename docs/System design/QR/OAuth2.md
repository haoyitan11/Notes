# Oauth2 dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aa4769e4-7344-476c-92a6-0bf5d30f6c1d" />

### 1. AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager
Function: Acts as the central OAuth2 lifecycle manager.
- Integrates ReactiveClientRegistrationRepository, ReactiveOAuth2AuthorizedClientService and ClientCredentialsReactiveOAuth2AuthorizedClientProvider.
- Coordinates token retrieval, token reuse, token expiration checks, and authorized client storage.

### 2. ReactiveClientRegistrationRepository
- Loads the ClientRegistration based on the registrationId (for example: scb).
- Maps the registrationId to OAuth2 client configurations defined in .properties or .yaml.
- Provides OAuth2 metadata such as Client ID, Client Secret, Token Endpoint, Grant Type, and Scopes.

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

