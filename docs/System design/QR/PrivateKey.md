# PrivateKey Dependencies Handle

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/37d28281-919b-4266-9b8d-886703c73fdb" />

### 1. FileInputStream & ResourceUtils

#### Purpose

Loads the KeyStore file content from the filesystem.

#### Dependencies

```java
java.io.FileInputStream
java.io.InputStream
org.springframework.util.ResourceUtils
```

#### Actual Usage

```java
try (InputStream inputStream = new FileInputStream(ResourceUtils.getFile(privateKeyStorePath))) {
    // Load KeyStore from inputStream
}
```

#### Responsibilities

- Locates KeyStore file by path.
- Opens file as InputStream.
- Provides byte stream to KeyStore loader.
- Used by **KeyStore** for file loading.

---

### 2. KeyStore

#### Purpose

Acts as the secure container that loads and stores private keys from JKS/PKCS12 files.

#### Dependencies

```java
java.security.KeyStore
java.io.InputStream
```

#### Actual Usage

```java
KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
keyStore.load(inputStream, keyStorePassword.toCharArray());
```

#### Responsibilities

- Creates KeyStore instance with specified type (JKS, PKCS12).
- Loads KeyStore data from InputStream.
- Decrypts KeyStore using password.
- Stores multiple key entries by alias.
- Provides key retrieval methods.
- Used by **KeyStore.getKey()** and **KeyStore.getEntry()** for PrivateKey extraction.

---

### 3. KeyStore.getKey()

#### Purpose

Extracts PrivateKey from KeyStore by alias and password.

#### Dependencies

```java
java.security.KeyStore
java.security.Key
java.security.PrivateKey
```

#### Actual Usage

```java
PrivateKey privateKey = (PrivateKey) keyStore.getKey(alias, keyPassword.toCharArray());
```

#### Responsibilities

- Looks up key entry by alias.
- Decrypts key entry using key password.
- Returns Key object (cast to PrivateKey).
- Used by **SecurityService** for key initialization.

---

### 4. KeyStore.PrivateKeyEntry

#### Purpose

Alternative method to extract PrivateKey with associated certificate chain.

#### Dependencies

```java
java.security.KeyStore
java.security.KeyStore.PrivateKeyEntry
java.security.KeyStore.PasswordProtection
```

#### Actual Usage

```java
KeyStore.PrivateKeyEntry privateKeyEntry = (KeyStore.PrivateKeyEntry) keyStore.getEntry(
        alias, new KeyStore.PasswordProtection(keyPassword.toCharArray()));
PrivateKey signingKey = privateKeyEntry.getPrivateKey();
```

#### Responsibilities

- Retrieves complete key entry with certificate chain.
- Wraps password in PasswordProtection object.
- Returns PrivateKeyEntry containing PrivateKey.
- Provides access to associated certificates.
- Used by **JWTUtil** for key extraction.

---

### 5. PrivateKey

#### Purpose

Final RSA private key object used for cryptographic operations.

#### Dependencies

```java
java.security.PrivateKey
```

#### Actual Usage

```java
// From KeyStore.getKey()
PrivateKey privateKey = (PrivateKey) keyStore.getKey(alias, keyPassword.toCharArray());

// From KeyStore.PrivateKeyEntry
PrivateKey privateKey = privateKeyEntry.getPrivateKey();
```

Used for signing:

```java
Signature signature = Signature.getInstance("SHA256withRSA");
signature.initSign(privateKey);
```

Used for JWS:

```java
JWSSigner signer = new RSASSASigner(privateKey);
```

Used for JWE decryption:

```java
RSADecrypter decrypter = new RSADecrypter(privateKey);
```

#### Responsibilities

- Digital signature generation (SHA256withRSA).
- JWS signing (RS256, RS512).
- JWE decryption (RSA-OAEP-256).
- Certificate-based authentication.

---

## Execution Flow

1. **FileInputStream** opens KeyStore file from filesystem:

   ```java
   InputStream inputStream = new FileInputStream(ResourceUtils.getFile(privateKeyStorePath));
   ```

2. **KeyStore.getInstance()** creates KeyStore instance:

   ```java
   KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
   ```

3. **KeyStore.load()** loads and decrypts KeyStore from InputStream:

   ```java
   keyStore.load(inputStream, keyStorePassword.toCharArray());
   ```

4. **KeyStore.getKey()** or **KeyStore.getEntry()** extracts PrivateKey by alias:

   ```java
   // Method 1: Direct key retrieval
   PrivateKey privateKey = (PrivateKey) keyStore.getKey(alias, keyPassword.toCharArray());
   
   // Method 2: Entry retrieval with certificate chain
   KeyStore.PrivateKeyEntry entry = (KeyStore.PrivateKeyEntry) keyStore.getEntry(
           alias, new KeyStore.PasswordProtection(keyPassword.toCharArray()));
   PrivateKey privateKey = entry.getPrivateKey();
   ```

5. **PrivateKey** is used for cryptographic operations:

   ```java
   // Signing
   Signature signature = Signature.getInstance("SHA256withRSA");
   signature.initSign(privateKey);
   
   // JWS Signing
   JWSSigner signer = new RSASSASigner(privateKey);
   
   // JWE Decryption
   RSADecrypter decrypter = new RSADecrypter(privateKey);
   ```

---

## Configuration Source

```java
@Configuration
public class KeyStoreConfiguration {

    @Value("${private.key-store}")
    String privateKeyStorePath;
    
    @Value("${private.key-store.password}")
    String privateKeyStorePass;

    @Bean
    public KeyStore privateKeyStore() throws Exception {
        try (InputStream inputStream = new FileInputStream(ResourceUtils.getFile(privateKeyStorePath))) {
            KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
            keyStore.load(inputStream, privateKeyStorePass.toCharArray());
            return keyStore;
        }
    }
}
```

```java
@Component
public class SecurityService {

    @Autowired
    KeyStore privateKeyStore;

    @Value("${bank.api.sign.private.alias}")
    String signKeyAlias;
    
    @Value("${bank.api.sign.private.password}")
    String signKeyPassword;

    private PrivateKey privateKey;

    @PostConstruct
    public void init() {
        this.privateKey = (PrivateKey) privateKeyStore.getKey(signKeyAlias, signKeyPassword.toCharArray());
    }
}
```

---

## Properties Reference

```properties
# KeyStore Configuration
private.key-store=/path/to/keystore.jks
private.key-store.password=keystorePassword

# Key Alias Configuration
bank.api.sign.private.alias=signing-key
bank.api.sign.private.password=keyPassword
```
