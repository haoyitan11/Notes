# PrivateKey dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/37d28281-919b-4266-9b8d-886703c73fdb" />

### 1. BouncyCastleProvider
Purpose : Acts as the cryptographic provider for PEM parsing and RSA key conversion.
Dependency : 
```java
<dependency>
<groupId>org.bouncycastle</groupId>
<artifactId>bcprov-jdk18on</artifactId>
<version>${bouncycastle.version}</version>
</dependency>
```
Function used :
```java
Security.addProvider(new BouncyCastleProvider());
```
Responsibilities : 
- Registers Bouncy Castle with Java Security.
- Provides cryptographic algorithms and ASN.1 implementations.
- Supports PEM processing.
- Used by PEMParser and JcaPEMKeyConverter.

### 2. Files & Paths
Purpose : Loads the PEM file content from the filesystem.
Dependencies
```java
java.nio.file.Files
java.nio.file.Paths
```
Functions Used : Files.readString(Paths.get(filePath))

Responsibilities : 
- Locates PEM file
- Reads file content into memory
- Converts file content into a String

Exceptions : IOException

### 3. Reader (StringReader)
Function: Provides character stream access to PEM content.
- Wraps the PEM String.
- Provides character stream access to the PEM content.
- Supplies the content to PEMParser.

### 4. PEMParser
Function: Parses the PEM formatted content.
- Reads PEM content from the Reader. 
- Parses the PEM structure. 
- Returns an object depending on the PEM format:
- 1. PEMKeyPair for PKCS#1 key pairs.
- 2. PrivateKeyInfo for PKCS#8 private keys.

### 5A. PEMKeyPair
Function: Represents a parsed key pair (PKCS#1).
- Contains both public and private key information.
- Returned when parsing PEM content such as:
```java
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```
- Provides access to PrivateKeyInfo through:
```java
pemKeyPair.getPrivateKeyInfo()
```

### 5B. PrivateKeyInfo
Function: Represents the ASN.1 encoded private key structure (PKCS#8).
- Used as an intermediate Bouncy Castle representation of a private key.
- Returned directly for PKCS#8 keys:
```java
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```
- Can also be extracted from PEMKeyPair.
- Serves as input to JcaPEMKeyConverter.

### 6. JcaPEMKeyConverter
Function: Converts Bouncy Castle key objects into Java Security objects.
-	Converts PrivateKeyInfo into a Java PrivateKey.
-	Works with the configured BouncyCastleProvider.
-	Bridges Bouncy Castle APIs and standard Java Security APIs.
Example : 
```java
PrivateKey privateKey = converter.getPrivateKey(privateKeyInfo);
```

### 7. PrivateKey
Function: Final RSA private key output.
- Standard Java Security PrivateKey.
- Returned by RSAUtility.getPrivateKey().
- Used for digital signature generation.
- Passed into
```java
signature.initSign(privateKey);
```
- Can also be used for RSA decryption operations.
