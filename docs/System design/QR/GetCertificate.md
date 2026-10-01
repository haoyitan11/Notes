# Certificate Dependencies Handle

## Overview

This document provides a complete guide for obtaining Certificate (specifically X509Certificate) from files using Java dependencies. It covers three loading patterns:

1. **PEM Certificate File Loading** - Using Bouncy Castle to parse PEM-formatted certificate files

2. **KeyStore Certificate Loading** - Using Java KeyStore to extract certificates from JKS/PKCS12 files

3. **DER/CER File Loading** - Using native Java CertificateFactory to load binary or Base64 encoded certificate files

---

<img width="3200" height="3784" alt="image" src="https://github.com/user-attachments/assets/3019f80d-ee0e-44d9-965e-88d5282ceb8b" />

# Part 1: PEM Certificate File Loading (Bouncy Castle)

## Purpose

Loads X509Certificate from PEM-formatted certificate files using Bouncy Castle library.

## Components
<img width="3200" height="2336" alt="image" src="https://github.com/user-attachments/assets/2e28d0df-ac7c-49e0-9d90-85ef1a6862f2" />

### 1.1 BouncyCastleProvider

#### Purpose

Acts as the cryptographic provider for PEM parsing.

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
- Provides PEM processing capabilities.
- Supports certificate parsing.

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
String pem = new String(Files.readAllBytes(Paths.get(certPath)));
```

#### Responsibilities

- Locates PEM file by path.
- Reads file content into memory.
- Converts file content into a String.

---

### 1.3 PEMParser

#### Purpose

Parses PEM-formatted content into Bouncy Castle certificate objects.

#### Dependencies

```java
org.bouncycastle.openssl.PEMParser
java.io.Reader
java.io.StringReader
```

#### Actual Usage

```java
Reader certReader = new StringReader(pem);
PEMParser pemParser = new PEMParser(certReader);
Object pemObject = pemParser.readObject();
```

#### Responsibilities

- Reads PEM content from Reader.
- Detects PEM type (certificate, key, etc.).
- Returns X509CertificateHolder for certificates.

---

### 1.4 X509CertificateHolder

#### Purpose

Represents the parsed certificate from PEM content.

#### Dependencies

```java
org.bouncycastle.cert.X509CertificateHolder
```

#### Actual Usage

```java
if (pemObject instanceof X509CertificateHolder) {
    X509CertificateHolder certHolder = (X509CertificateHolder) pemObject;
    // Convert to X509Certificate
}
```

#### Responsibilities

- Holds certificate data from PEM parsing.
- Contains encoded certificate bytes.
- Used by JcaX509CertificateConverter for conversion.

---

### 1.5 JcaX509CertificateConverter

#### Purpose

Converts Bouncy Castle X509CertificateHolder to standard Java X509Certificate.

#### Dependencies

```java
org.bouncycastle.cert.jcajce.JcaX509CertificateConverter
```

#### Actual Usage

```java
JcaX509CertificateConverter converter = new JcaX509CertificateConverter()
        .setProvider("BC");
X509Certificate certificate = converter.getCertificate(certHolder);
```

#### Responsibilities

- Converts X509CertificateHolder to Java X509Certificate.
- Uses the registered Bouncy Castle provider ("BC").
- Bridges Bouncy Castle APIs and Java Security APIs.

---

### 1.6 X509Certificate

#### Purpose

Final certificate object used by application code.

#### Dependencies

```java
java.security.cert.X509Certificate
```

#### Actual Usage

```java
X509Certificate certificate = converter.getCertificate(certHolder);
PublicKey pk = certificate.getPublicKey();
```

#### Responsibilities

- Contains X.509 certificate data.
- Provides access to PublicKey.
- Used for signature verification and encryption.

---

## Execution Flow

**Step 1:** BouncyCastleProvider is registered with Java Security:

```java
Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
```

**Step 2:** Files.readAllBytes() reads PEM file content:

```java
String pem = new String(Files.readAllBytes(Paths.get(certPath)));
```

**Step 3:** StringReader wraps PEM content as character stream:

```java
Reader certReader = new StringReader(pem);
```

**Step 4:** PEMParser parses PEM content:

```java
PEMParser pemParser = new PEMParser(certReader);
Object pemObject = pemParser.readObject();
```

**Step 5:** Object type is checked and certificate is converted:

```java
X509Certificate certificate = null;
if (pemObject instanceof X509CertificateHolder) {
    X509CertificateHolder certHolder = (X509CertificateHolder) pemObject;
    JcaX509CertificateConverter converter = new JcaX509CertificateConverter()
            .setProvider("BC");
    certificate = converter.getCertificate(certHolder);
}
```

**Step 6:** Close the parser and return certificate:

```java
pemParser.close();
return certificate;
```

## PEM Certificate Format Reference

### X.509 Certificate Format

```
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAJC1HiIAZAiUMA...
-----END CERTIFICATE-----
```

- Parsed as **X509CertificateHolder**.
- Requires JcaX509CertificateConverter for Java X509Certificate.

---

# Part 2: KeyStore Certificate Loading (JKS/PKCS12)

## Purpose

Extracts X509Certificate from Java KeyStore files (JKS or PKCS12 format).

## Components
<img width="3200" height="2676" alt="image" src="https://github.com/user-attachments/assets/fa51aac8-0cd6-44aa-a107-302c050f2f95" />

### 2.1 FileInputStream

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

### 2.2 KeyStore

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
- Stores certificates and key entries by alias.

---

### 2.3A KeyStore.getCertificate()

#### Purpose

Extracts a single Certificate by alias.

#### Dependencies

```java
java.security.KeyStore
java.security.cert.Certificate
```

#### Actual Usage

```java
Certificate cert = keyStore.getCertificate(alias);
X509Certificate x509Cert = (X509Certificate) cert;
```

#### Responsibilities

- Looks up certificate entry by alias.
- Returns Certificate object (cast to X509Certificate).
- Works for trusted certificate entries.

---

### 2.3B KeyStore.getCertificateChain()

#### Purpose

Extracts the complete certificate chain for a key entry.

#### Dependencies

```java
java.security.KeyStore
java.security.cert.Certificate[]
```

#### Actual Usage

```java
Certificate[] certChain = keyStore.getCertificateChain(alias);
X509Certificate[] x509Chain = new X509Certificate[certChain.length];
for (int i = 0; i < certChain.length; i++) {
    x509Chain[i] = (X509Certificate) certChain[i];
}
```

#### Responsibilities

- Retrieves complete certificate chain for key alias.
- Returns array of Certificate objects.
- First certificate is the end-entity certificate.
- Remaining certificates are CA certificates.

---

### 2.3C KeyStore.PrivateKeyEntry.getCertificateChain()

#### Purpose

Alternative method to extract certificate chain from PrivateKeyEntry.

#### Dependencies

```java
java.security.KeyStore.PrivateKeyEntry
java.security.KeyStore.PasswordProtection
```

#### Actual Usage

```java
KeyStore.PrivateKeyEntry privateKeyEntry = (KeyStore.PrivateKeyEntry) keyStore.getEntry(
        alias, new KeyStore.PasswordProtection(keyPassword.toCharArray()));
Certificate[] certChain = privateKeyEntry.getCertificateChain();
X509Certificate certificate = (X509Certificate) privateKeyEntry.getCertificate();
```

#### Responsibilities

- Retrieves complete key entry with certificate chain.
- Provides access to both PrivateKey and certificates.
- Returns end-entity certificate via `getCertificate()`.
- Returns full chain via `getCertificateChain()`.

---

### 2.4 X509Certificate

#### Purpose

Final certificate object used by application code.

#### Dependencies

```java
java.security.cert.X509Certificate
```

#### Actual Usage

```java
X509Certificate certificate = (X509Certificate) keyStore.getCertificate(alias);
PublicKey pk = certificate.getPublicKey();
```

#### Responsibilities

- Contains X.509 certificate data.
- Provides access to PublicKey.
- Used for signature verification and encryption.

---

## Execution Flow

**Method 1: Direct Certificate Retrieval**

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

4. **KeyStore.getCertificate()** extracts certificate:

   ```java
   X509Certificate certificate = (X509Certificate) keyStore.getCertificate(alias);
   ```

**Method 2: Certificate Chain Retrieval**

```java
Certificate[] certChain = keyStore.getCertificateChain(alias);
```

**Method 3: From PrivateKeyEntry**

```java
KeyStore.PrivateKeyEntry entry = (KeyStore.PrivateKeyEntry) keyStore.getEntry(
        alias, new KeyStore.PasswordProtection(keyPassword.toCharArray()));
X509Certificate certificate = (X509Certificate) entry.getCertificate();
Certificate[] chain = entry.getCertificateChain();
```

# Part 3: DER/CER File Loading (Native Java)
<img width="3200" height="2536" alt="image" src="https://github.com/user-attachments/assets/c36a4a0e-5638-4e81-8417-750af9996dce" />

## Purpose

Loads X509Certificate from DER-encoded binary files or Base64-encoded CER files using native Java APIs (no external libraries required).

## Components

### 3.1 FileInputStream

#### Purpose

Opens the certificate file for reading as a byte stream.

#### Dependencies

```java
java.io.FileInputStream
java.io.InputStream
```

#### Actual Usage

```java
FileInputStream fin = new FileInputStream(certPath);
```

#### Responsibilities

- Opens certificate file by path.
- Provides byte stream to CertificateFactory.
- Used for file-based certificate loading.

---

### 3.2 ByteArrayInputStream

#### Purpose

Wraps raw certificate bytes into an InputStream for processing.

#### Dependencies

```java
java.io.ByteArrayInputStream
java.io.InputStream
```

#### Actual Usage

```java
InputStream inputStream = new ByteArrayInputStream(certBytes);
```

#### Responsibilities

- Wraps byte array as InputStream.
- Provides byte stream to CertificateFactory.
- Used for in-memory certificate loading.

---

### 3.3 CertificateFactory

#### Purpose

Generates X509Certificate objects from encoded certificate data.

#### Dependencies

```java
java.security.cert.CertificateFactory
java.security.cert.CertificateException
```

#### Actual Usage

```java
CertificateFactory f = CertificateFactory.getInstance("X.509");
X509Certificate certificate = (X509Certificate) f.generateCertificate(inputStream);
```

#### Responsibilities

- Creates CertificateFactory instance for X.509 type.
- Parses encoded certificate data from InputStream.
- Generates X509Certificate object.
- Supports both DER (binary) and Base64-encoded formats.

---

### 3.4 X509Certificate

#### Purpose

Final certificate object containing public key and certificate metadata.

#### Dependencies

```java
java.security.cert.X509Certificate
java.security.cert.Certificate
```

#### Actual Usage

```java
X509Certificate certificate = (X509Certificate) f.generateCertificate(fin);
PublicKey pk = certificate.getPublicKey();
```

#### Responsibilities

- Contains X.509 certificate data.
- Provides access to PublicKey via `getPublicKey()`.
- Contains subject, issuer, validity period, and extensions.
- Used for signature verification and encryption.

---

## Execution Flow

**Step 1:** FileInputStream opens certificate file:

```java
FileInputStream fin = new FileInputStream(certPath);
```

**Step 2:** CertificateFactory.getInstance() creates X.509 factory:

```java
CertificateFactory f = CertificateFactory.getInstance("X.509");
```

**Step 3:** CertificateFactory.generateCertificate() creates certificate:

```java
X509Certificate certificate = (X509Certificate) f.generateCertificate(fin);
```

**Step 4:** Extract PublicKey from certificate:

```java
PublicKey pk = certificate.getPublicKey();
```

**Step 5:** Close the input stream:

```java
fin.close();
```

# Part 4: Certificate Utility Methods

## Purpose

Common utility operations performed on X509Certificate objects.

## 4.1 Extract PublicKey from Certificate

```java
PublicKey publicKey = certificate.getPublicKey();
```

## 4.2 Get Certificate Subject

```java
X500Principal subject = certificate.getSubjectX500Principal();
String subjectDN = certificate.getSubjectDN().getName();
```

## 4.3 Get Certificate Issuer

```java
X500Principal issuer = certificate.getIssuerX500Principal();
String issuerDN = certificate.getIssuerDN().getName();
```

## 4.4 Get Certificate Validity Period

```java
Date notBefore = certificate.getNotBefore();
Date notAfter = certificate.getNotAfter();
```

## 4.5 Validate Certificate Date

```java
try {
    certificate.checkValidity(); // Check against current date
    certificate.checkValidity(new Date()); // Check against specific date
} catch (CertificateExpiredException e) {
    // Certificate has expired
} catch (CertificateNotYetValidException e) {
    // Certificate is not yet valid
}
```

## 4.6 Get Certificate Serial Number

```java
BigInteger serialNumber = certificate.getSerialNumber();
```

## 4.7 Get Certificate Signature Algorithm

```java
String sigAlgName = certificate.getSigAlgName();
String sigAlgOID = certificate.getSigAlgOID();
```

## 4.8 Verify Certificate Signature

```java
try {
    certificate.verify(issuerPublicKey);
    // Signature is valid
} catch (SignatureException e) {
    // Signature verification failed
}
```

---

# Summary

## Comparison of Loading Patterns

| Pattern | File Format | Library Required | Use Case |
|---------|-------------|------------------|----------|
| PEM Certificate Loading | `.pem` (Base64 text with headers) | Bouncy Castle | PEM-formatted certificates |
| KeyStore Loading | `.jks` / `.p12` | None (native Java) | Enterprise keystores |
| DER/CER File Loading | `.der` / `.cer` (binary or Base64) | None (native Java) | Standard certificate files |

## File Format Reference

| Format | Header | Pattern |
|--------|--------|---------|
| PEM | `-----BEGIN CERTIFICATE-----` | Part 1 (Bouncy Castle) |
| JKS | Binary KeyStore | Part 2 (KeyStore) |
| PKCS#12 | Binary KeyStore | Part 2 (KeyStore) |
| DER | Binary (no header) | Part 3 (Native Java) |
| CER/CRT | Binary or Base64 | Part 3 (Native Java) |
