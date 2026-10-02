# PrivateKey Dependencies Handle

## Overview

This document provides a complete guide for obtaining PrivateKey from files using Java dependencies. It covers three loading patterns:

1. **PEM File Loading** - Using Bouncy Castle to parse PEM-formatted private keys
2. **Raw PKCS8 Binary File Loading** - Using native Java to load DER-encoded PKCS8 files
3. **KeyStore Loading** - Using Java KeyStore to extract keys from JKS/PKCS12 files

---

# Part 1: PEM File Loading (Bouncy Castle)


## Purpose

Loads PrivateKey from PEM-formatted files using Bouncy Castle library.

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/37d28281-919b-4266-9b8d-886703c73fdb" />

### 1.1 BouncyCastleProvider

#### Purpose

Acts as the cryptographic provider for PEM parsing and RSA key conversion.

#### Dependencies

```java
org.bouncycastle.jce.provider.BouncyCastleProvider
java.security.Security
```

#### Actual Usage

```java
Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
```

#### Responsibilities

- Registers Bouncy Castle with Java Security.
- Provides cryptographic algorithms and ASN.1 implementations.
- Supports PEM processing.
- Used by PEMParser and JcaPEMKeyConverter.

---

### 1.2 Files & Paths

#### Purpose

Loads the PEM file content from the filesystem.

#### Dependencies

```java
java.nio.file.Files
java.nio.file.Paths
```

#### Actual Usage

```java
String pem = new String(Files.readAllBytes(Paths.get(path)));
```

#### Responsibilities

- Locates PEM file by path.
- Reads file content into memory.
- Converts file content into a String.
- Used by StringReader for character stream creation.

---

### 1.3 Reader (StringReader)

#### Purpose

Provides character stream access to the PEM content.

#### Dependencies

```java
java.io.StringReader
java.io.Reader
```

#### Actual Usage

```java
Reader privateKeyReader = new StringReader(pem);
```

#### Responsibilities

- Wraps PEM String.
- Exposes content as a Reader.
- Supplies character stream to PEMParser.

---

### 1.4 PEMParser

#### Purpose

Parses PEM-formatted content into Bouncy Castle key objects.

#### Dependencies

```java
org.bouncycastle.openssl.PEMParser
java.io.Reader
```

#### Actual Usage

```java
PEMParser privatePemParser = new PEMParser(privateKeyReader);
Object privateObject = privatePemParser.readObject();
```

#### Responsibilities

- Reads PEM content from Reader.
- Detects PEM type (PKCS#1, PKCS#8, encrypted key).
- Converts PEM text into Java objects.
- Returns PEMKeyPair (PKCS#1) or PrivateKeyInfo (PKCS#8).

---

### 1.5A PEMKeyPair

#### Purpose

Represents a PKCS#1 RSA key pair parsed from PEM.

#### Dependencies

```java
org.bouncycastle.openssl.PEMKeyPair
```

#### Actual Usage

```java
if (privateObject instanceof PEMKeyPair) {
    PEMKeyPair pemKeyPair = (PEMKeyPair) privateObject;
    // Extract PrivateKeyInfo from the key pair
    privateKey = converter.getPrivateKey(pemKeyPair.getPrivateKeyInfo());
}
```

#### Responsibilities

- Contains RSA public key information.
- Contains RSA private key information.
- Provides access to PrivateKeyInfo via `getPrivateKeyInfo()`.
- Used when PEM contains PKCS#1 format (`-----BEGIN RSA PRIVATE KEY-----`).

---

### 1.5B PrivateKeyInfo

#### Purpose

Represents the ASN.1 private key structure.

#### Dependencies

```java
org.bouncycastle.asn1.pkcs.PrivateKeyInfo
```

#### Actual Usage

Obtained directly from PKCS#8 PEM:

```java
if (privateObject instanceof PrivateKeyInfo) {
    PrivateKeyInfo privateKeyInfo = (PrivateKeyInfo) privateObject;
    privateKey = converter.getPrivateKey(privateKeyInfo);
}
```

Or from PEMKeyPair (PKCS#1):

```java
privateKey = converter.getPrivateKey(pemKeyPair.getPrivateKeyInfo());
```

#### Responsibilities

- Holds private key metadata.
- Holds ASN.1 encoded key data.
- Intermediate representation used by Bouncy Castle.
- Used by JcaPEMKeyConverter for PrivateKey generation.

---

### 1.6 JcaPEMKeyConverter

#### Purpose

Converts Bouncy Castle key objects into standard Java Security objects.

#### Dependencies

```java
org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter
```

#### Actual Usage

```java
JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
privateKey = converter.getPrivateKey(pemKeyPair.getPrivateKeyInfo());
```

#### Responsibilities

- Converts PrivateKeyInfo to Java PrivateKey.
- Uses the registered Bouncy Castle provider ("BC").
- Bridges Bouncy Castle APIs and Java Security APIs.
- Returns standard `java.security.PrivateKey`.

---

### 1.7 PrivateKey

#### Purpose

Final RSA private key object used by application code.

#### Dependencies

```java
java.security.PrivateKey
```

#### Actual Usage

```java
PrivateKey privateKey = converter.getPrivateKey(privateKeyInfo);
```

#### Responsibilities

- Digital signature generation.
- RSA decryption operations.

---

## Execution Flow

**Step 1:** BouncyCastleProvider is registered with Java Security:

```java
Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
```

**Step 2:** Files.readAllBytes() reads PEM file content:

```java
String pem = new String(Files.readAllBytes(Paths.get(path)));
```

**Step 3:** StringReader wraps PEM content as character stream:

```java
Reader privateKeyReader = new StringReader(pem);
```

**Step 4:** PEMParser parses PEM content:

```java
PEMParser privatePemParser = new PEMParser(privateKeyReader);
Object privateObject = privatePemParser.readObject();
```

**Step 5:** Object type is checked and PrivateKeyInfo is obtained:

```java
PrivateKey privateKey = null;

if (privateObject instanceof PEMKeyPair) {
    // PKCS#1 format: -----BEGIN RSA PRIVATE KEY-----
    PEMKeyPair pemKeyPair = (PEMKeyPair) privateObject;
    JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
    privateKey = converter.getPrivateKey(pemKeyPair.getPrivateKeyInfo());
} else if (privateObject instanceof PrivateKeyInfo) {
    // PKCS#8 format: -----BEGIN PRIVATE KEY-----
    PrivateKeyInfo privateKeyInfo = (PrivateKeyInfo) privateObject;
    JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
    privateKey = converter.getPrivateKey(privateKeyInfo);
}
```

**Step 6:** Close the parser and return PrivateKey:

```java
privatePemParser.close();
return privateKey;
```

---

## PEM Format Reference

### PKCS#1 Format (RSA Private Key)

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA...
-----END RSA PRIVATE KEY-----
```

- Parsed as **PEMKeyPair**.
- Requires `pemKeyPair.getPrivateKeyInfo()` to extract.

### PKCS#8 Format (Private Key)

```
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAAOCAQ8A...
-----END PRIVATE KEY-----
```

- Parsed directly as **PrivateKeyInfo**.
- No additional conversion needed before JcaPEMKeyConverter.

---
# Part 2: Raw PKCS8 Binary File Loading (Native Java)

## Purpose

Loads PrivateKey from DER-encoded PKCS8 binary files using native Java APIs (no Bouncy Castle required).

## Components
<img width="3200" height="1830" alt="image" src="https://github.com/user-attachments/assets/728fdb47-8973-4b07-ad72-736c4e273658" />

### 2.1 Files & Paths

#### Purpose

Loads the binary key file content from the filesystem.

#### Dependencies

```java
java.nio.file.Files
java.nio.file.Path
java.io.File
```

#### Actual Usage

```java
Path privateKeyPath = new File(jwsKeyPath).toPath();
byte[] priKeyBytes = Files.readAllBytes(privateKeyPath);
```

#### Responsibilities

- Locates key file by path.
- Reads file content as byte array.
- Used by **PKCS8EncodedKeySpec** for key specification creation.

---

### 2.2 PKCS8EncodedKeySpec

#### Purpose

Wraps raw PKCS8 encoded bytes into a key specification.

#### Dependencies

```java
java.security.spec.PKCS8EncodedKeySpec
```

#### Actual Usage

```java
PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(priKeyBytes);
```

#### Responsibilities

- Wraps raw bytes in key specification format.
- Provides encoded key data to **KeyFactory**.
- Represents PKCS#8 private key structure.

---

### 2.3 KeyFactory

#### Purpose

Generates PrivateKey from encoded key specification.

#### Dependencies

```java
java.security.KeyFactory
```

#### Actual Usage

```java
KeyFactory kf = KeyFactory.getInstance("RSA");
PrivateKey signingKey = kf.generatePrivate(spec);
```

#### Responsibilities

- Creates KeyFactory instance for RSA algorithm.
- Generates **PrivateKey** from **PKCS8EncodedKeySpec**.
- Bridges encoded key data to Java Security PrivateKey.

---

### 2.4 PrivateKey

#### Purpose

Final RSA private key object used by application code.

#### Dependencies

```java
java.security.PrivateKey
```

#### Actual Usage

```java
PrivateKey signingKey = kf.generatePrivate(spec);
```

#### Responsibilities

- Digital signature generation.
- RSA decryption operations.

---

## Execution Flow

1. **Files.readAllBytes()** reads binary key file:

   ```java
   Path privateKeyPath = new File(jwsKeyPath).toPath();
   byte[] priKeyBytes = Files.readAllBytes(privateKeyPath);
   ```

2. **KeyFactory.getInstance()** creates RSA KeyFactory:

   ```java
   KeyFactory kf = KeyFactory.getInstance("RSA");
   ```

3. **PKCS8EncodedKeySpec** wraps raw bytes:

   ```java
   PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(priKeyBytes);
   ```

4. **KeyFactory.generatePrivate()** creates PrivateKey:

   ```java
   PrivateKey signingKey = kf.generatePrivate(spec);
   ```

---

# Part 3: KeyStore Loading (JKS/PKCS12)

## Purpose

Extracts PrivateKey from Java KeyStore files (JKS or PKCS12 format).

## Components
<img width="3200" height="2482" alt="image" src="https://github.com/user-attachments/assets/cb82e3b2-4a22-46ad-a644-a818d14ca838" />

### 3.1 FileInputStream

#### Purpose

Opens the KeyStore file for reading.

#### Dependencies

```java
java.io.FileInputStream
java.io.InputStream
```

#### Actual Usage

```java
InputStream inputStream = new FileInputStream(keyStorePath);
```

#### Responsibilities

- Opens KeyStore file by path.
- Provides byte stream to KeyStore loader.

---

### 3.2 KeyStore

#### Purpose

Loads and manages the KeyStore container.

#### Dependencies

```java
java.security.KeyStore
```

#### Actual Usage

```java
KeyStore keyStore = KeyStore.getInstance(keyStoreType);
keyStore.load(inputStream, keyStorePassword.toCharArray());
```

#### Responsibilities

- Creates KeyStore instance with specified type (JKS, PKCS12).
- Loads KeyStore data from InputStream.
- Decrypts KeyStore using password.
- Stores multiple key entries by alias.

---

### 3.3A KeyStore.getKey()

#### Purpose

Extracts PrivateKey directly by alias and password.

#### Dependencies

```java
java.security.KeyStore
java.security.Key
```

#### Actual Usage

```java
PrivateKey privateKey = (PrivateKey) keyStore.getKey(alias, keyPassword.toCharArray());
```

#### Responsibilities

- Looks up key entry by alias.
- Decrypts key entry using key password.
- Returns Key object (cast to PrivateKey).

---

### 3.3B KeyStore.PrivateKeyEntry

#### Purpose

Alternative method to extract PrivateKey with associated certificate chain.

#### Dependencies

```java
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

---

### 3.4 PrivateKey

#### Purpose

Final RSA private key object used by application code.

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

#### Responsibilities

- Digital signature generation.
- RSA decryption operations.

---

## Execution Flow

1. **FileInputStream** opens KeyStore file:

   ```java
   InputStream inputStream = new FileInputStream(keyStorePath);
   ```

2. **KeyStore.getInstance()** creates KeyStore instance:

   ```java
   KeyStore keyStore = KeyStore.getInstance(keyStoreType);
   ```

3. **KeyStore.load()** loads and decrypts KeyStore:

   ```java
   keyStore.load(inputStream, keyStorePassword.toCharArray());
   ```

4. **KeyStore.getKey()** or **KeyStore.getEntry()** extracts PrivateKey:

   ```java
   // Method 1: Direct key retrieval
   PrivateKey privateKey = (PrivateKey) keyStore.getKey(alias, keyPassword.toCharArray());
   
   // Method 2: Entry retrieval with certificate chain
   KeyStore.PrivateKeyEntry entry = (KeyStore.PrivateKeyEntry) keyStore.getEntry(
           alias, new KeyStore.PasswordProtection(keyPassword.toCharArray()));
   PrivateKey privateKey = entry.getPrivateKey();
   ```

---

# Summary

## Comparison of Loading Patterns

| Pattern | File Format | Library Required | Use Case |
|---------|-------------|------------------|----------|
| PEM File Loading | `.pem` (Base64 text) | Bouncy Castle | PEM-formatted keys from external sources |
| Raw PKCS8 Binary | `.der` / `.key` (binary) | None (native Java) | DER-encoded PKCS8 binary files |
| KeyStore Loading | `.jks` / `.p12` | None (native Java) | Enterprise keystores with multiple keys |

## File Format Reference

| Format | Header | Pattern |
|--------|--------|---------|
| PKCS#1 PEM | `-----BEGIN RSA PRIVATE KEY-----` | Part 1 (Bouncy Castle) |
| PKCS#8 PEM | `-----BEGIN PRIVATE KEY-----` | Part 1 (Bouncy Castle) |
| PKCS#8 DER | Binary (no header) | Part 2 (Native Java) |
| JKS | Binary KeyStore | Part 3 (KeyStore) |
| PKCS#12 | Binary KeyStore | Part 3 (KeyStore) |
