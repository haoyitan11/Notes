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
Purpose : Provides character stream access to the PEM content.
Dependency 
```java
java.io.StringReader
```
Functions Used : 
```java
new StringReader(pemContent)
```
Responsibilities :
- Wraps PEM String
- Exposes content as a Reader
- Supplies character stream to PEMParser

### 4. PEMParser
Purpose : Parses PEM-formatted content into Bouncy Castle key objects.
Dependency
```java
<dependency>
<groupId>org.bouncycastle</groupId>
<artifactId>bcpkix-jdk18on</artifactId>
<version>${bouncycastle.version}</version>
</dependency>
```
Function Used : 
```java
PEMParser parser = new PEMParser(reader);
Object obj = parser.readObject();
```

Responsibilities : 
- Reads PEM content
- Detects PEM type
- Converts PEM text into Java objects

### 5A. PEMKeyPair
Purpose : Represents a PKCS#1 RSA key pair.
Dependency :
```java
org.bouncycastle.openssl.PEMKeyPair
```
Functions Used :
```java
pemKeyPair.getPrivateKeyInfo()
```
Responsibilities :
- Contains RSA public key information
- Contains RSA private key information
- Provides access to PrivateKeyInfo

### 5B. PrivateKeyInfo
Purpose : Represents the ASN.1 private key structure.
Dependency : 
```java
org.bouncycastle.asn1.pkcs.PrivateKeyInfo
```
Functions Used : 
Obtained directly from
```java
(PrivateKeyInfo) obj
```
or from:
```java
pemKeyPair.getPrivateKeyInfo()
```
Responsibilities : 
- Holds private key metadata.
- Holds ASN.1 encoded key data.
- Intermediate representation used by Bouncy Castle.

### 6. JcaPEMKeyConverter
Purpose : Converts Bouncy Castle key objects into standard Java Security objects.
Dependency : 
```java
org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter
```
Functions Used :
```java
new JcaPEMKeyConverter().setProvider("BC")
converter.getPrivateKey(privateKeyInfo)
```
Responsibilities : 
- Converts PrivateKeyInfo to Java PrivateKey
- Uses the registered Bouncy Castle provider
- Bridges Bouncy Castle APIs and Java Security APIs

### 7. PrivateKey
Purpose : Final RSA private key object used by application code.
Dependency : 
```java
java.security.PrivateKey
```
Functions Used :
up
Returned from:
```java
converter.getPrivateKey(privateKeyInfo)
```
Used for signing : 
```java
signature.initSign(privateKey);
```
Used for decryption :
```java
cipher.init(Cipher.DECRYPT_MODE, privateKey);
```
Responsibilities : 
- Digital signature generation.
- Signature verification workflows.
- RSA decryption operations.
