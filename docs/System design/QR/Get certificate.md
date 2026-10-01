# Certificate Dependencies Handle

## Components

### 1. BouncyCastleProvider

#### Purpose

Acts as the cryptographic provider for PEM parsing and certificate conversion.

#### Dependencies

```java
org.bouncycastle.jce.provider.BouncyCastleProvider
java.security.Security
```

#### Actual Usage

```java
Security.addProvider(new BouncyCastleProvider());
```

#### Responsibilities

- Registers Bouncy Castle with Java Security.
- Provides cryptographic algorithms and ASN.1 implementations.
- Supports PEM processing.
- Used by **PEMParser** and **JcaX509CertificateConverter**.

---

### 2. Files & Paths

#### Purpose

Loads the PEM certificate file content from the filesystem.

#### Dependencies

```java
java.nio.file.Files
java.nio.file.Paths
```

#### Actual Usage

```java
String pemContent = Files.readString(Paths.get(filePath));
```

#### Responsibilities

- Locates certificate file by path.
- Reads file content into memory.
- Converts file content into a String.
- Used by **StringReader** for character stream creation.

#### Exceptions

```java
IOException
```

---

### 3. Reader (StringReader)

#### Purpose

Provides character stream access to the PEM content.

#### Dependencies

```java
java.io.StringReader
java.io.Reader
```

#### Actual Usage

```java
Reader reader = new StringReader(pemContent);
```

#### Responsibilities

- Wraps PEM String.
- Exposes content as a Reader.
- Supplies character stream to **PEMParser**.

---

### 4. PEMParser

#### Purpose

Parses PEM-formatted content into Bouncy Castle certificate objects.

#### Dependencies

```java
org.bouncycastle.openssl.PEMParser
java.io.Reader
```

#### Actual Usage

```java
PEMParser parser = new PEMParser(reader);
Object obj = parser.readObject();
```

#### Responsibilities

- Reads PEM content from Reader.
- Detects PEM type (certificate, key, etc.).
- Converts PEM text into Java objects.
- Returns **X509CertificateHolder** for certificates.
- Used by certificate conversion logic.

---

### 5. X509CertificateHolder

#### Purpose

Represents an X.509 certificate in Bouncy Castle format.

#### Dependencies

```java
org.bouncycastle.cert.X509CertificateHolder
```

#### Actual Usage

```java
if (obj instanceof X509CertificateHolder) {
    X509CertificateHolder certHolder = (X509CertificateHolder) obj;
}
```

#### Responsibilities

- Holds X.509 certificate data.
- Contains subject and issuer information.
- Contains public key information.
- Contains validity period.
- Intermediate representation used by Bouncy Castle.
- Used by **JcaX509CertificateConverter** for certificate generation.

---

### 6. JcaX509CertificateConverter

#### Purpose

Converts Bouncy Castle certificate objects into standard Java Security certificates.

#### Dependencies

```java
org.bouncycastle.cert.jcajce.JcaX509CertificateConverter
```

#### Actual Usage

```java
JcaX509CertificateConverter converter = new JcaX509CertificateConverter().setProvider("BC");
X509Certificate certificate = converter.getCertificate(certHolder);
```

#### Responsibilities

- Converts **X509CertificateHolder** to Java **X509Certificate**.
- Uses the registered Bouncy Castle provider.
- Bridges Bouncy Castle APIs and Java Security APIs.
- Returns standard **java.security.cert.X509Certificate**.

---

### 7. X509Certificate

#### Purpose

Final X.509 certificate object used by application code.

#### Dependencies

```java
java.security.cert.X509Certificate
java.security.cert.Certificate
```

#### Actual Usage

Returned from converter:

```java
X509Certificate certificate = converter.getCertificate(certHolder);
```

Get public key:

```java
PublicKey publicKey = certificate.getPublicKey();
```

Verify signature:

```java
certificate.verify(issuerPublicKey);
```

Check validity:

```java
certificate.checkValidity();
certificate.checkValidity(date);
```

Get certificate information:

```java
X500Principal subject = certificate.getSubjectX500Principal();
X500Principal issuer = certificate.getIssuerX500Principal();
Date notBefore = certificate.getNotBefore();
Date notAfter = certificate.getNotAfter();
BigInteger serialNumber = certificate.getSerialNumber();
```

#### Responsibilities

- Provides public key extraction.
- Signature verification.
- Certificate validity checking.
- Access to certificate metadata (subject, issuer, validity dates).
- Used for SSL/TLS, encryption, and signature verification.

---

### 8. PublicKey

#### Purpose

Public key extracted from certificate for cryptographic operations.

#### Dependencies

```java
java.security.PublicKey
```

#### Actual Usage

Extract from certificate:

```java
PublicKey publicKey = certificate.getPublicKey();
```

Used for encryption:

```java
Cipher cipher = Cipher.getInstance("RSA");
cipher.init(Cipher.ENCRYPT_MODE, publicKey);
```

Used for signature verification:

```java
Signature signature = Signature.getInstance("SHA256withRSA");
signature.initVerify(publicKey);
signature.update(data);
boolean valid = signature.verify(signatureBytes);
```

#### Responsibilities

- RSA encryption operations.
- Signature verification.
- Key agreement protocols.

---

## Execution Flow

1. **BouncyCastleProvider** is registered with Java Security:

   ```java
   Security.addProvider(new BouncyCastleProvider());
   ```

2. **Files.readString()** reads PEM certificate file content:

   ```java
   String pemContent = Files.readString(Paths.get(filePath));
   ```

3. **StringReader** wraps PEM content as character stream:

   ```java
   Reader reader = new StringReader(pemContent);
   ```

4. **PEMParser** parses PEM content:

   ```java
   PEMParser parser = new PEMParser(reader);
   Object obj = parser.readObject();
   ```

5. **X509CertificateHolder** is obtained from parsed object:

   ```java
   X509CertificateHolder certHolder;
   
   if (obj instanceof X509CertificateHolder) {
       certHolder = (X509CertificateHolder) obj;
   }
   ```

6. **JcaX509CertificateConverter** converts to Java X509Certificate:

   ```java
   JcaX509CertificateConverter converter = new JcaX509CertificateConverter().setProvider("BC");
   X509Certificate certificate = converter.getCertificate(certHolder);
   ```

7. **X509Certificate** is used to extract public key and perform operations:

   ```java
   // Extract public key
   PublicKey publicKey = certificate.getPublicKey();
   
   // Verify certificate validity
   certificate.checkValidity();
   
   // Get certificate info
   X500Principal subject = certificate.getSubjectX500Principal();
   Date notAfter = certificate.getNotAfter();
   ```

8. **PublicKey** is used for cryptographic operations:

   ```java
   // Encryption
   Cipher cipher = Cipher.getInstance("RSA");
   cipher.init(Cipher.ENCRYPT_MODE, publicKey);
   byte[] encryptedData = cipher.doFinal(plainData);
   
   // Signature verification
   Signature signature = Signature.getInstance("SHA256withRSA");
   signature.initVerify(publicKey);
   signature.update(data);
   boolean valid = signature.verify(signatureBytes);
   ```

---

## Complete Example

```java
import org.bouncycastle.cert.X509CertificateHolder;
import org.bouncycastle.cert.jcajce.JcaX509CertificateConverter;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.openssl.PEMParser;

import java.io.Reader;
import java.io.StringReader;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.PublicKey;
import java.security.Security;
import java.security.cert.X509Certificate;

public class PEMCertificateLoader {

    static {
        Security.addProvider(new BouncyCastleProvider());
    }

    public static X509Certificate loadCertificate(String filePath) throws Exception {
        // Read PEM file
        String pemContent = Files.readString(Paths.get(filePath));
        
        // Parse PEM content
        Reader reader = new StringReader(pemContent);
        PEMParser parser = new PEMParser(reader);
        Object obj = parser.readObject();
        parser.close();
        
        // Extract X509CertificateHolder
        X509CertificateHolder certHolder;
        if (obj instanceof X509CertificateHolder) {
            certHolder = (X509CertificateHolder) obj;
        } else {
            throw new IllegalArgumentException("Unsupported PEM format");
        }
        
        // Convert to Java X509Certificate
        JcaX509CertificateConverter converter = new JcaX509CertificateConverter().setProvider("BC");
        return converter.getCertificate(certHolder);
    }

    public static PublicKey loadPublicKey(String filePath) throws Exception {
        X509Certificate certificate = loadCertificate(filePath);
        return certificate.getPublicKey();
    }
}
```

---

## Alternative: CertificateFactory (Without Bouncy Castle)

For DER or simple PEM certificates, Java's built-in CertificateFactory can be used:

### Components

#### 1. FileInputStream

```java
FileInputStream fis = new FileInputStream(filePath);
```

#### 2. CertificateFactory

```java
CertificateFactory cf = CertificateFactory.getInstance("X.509");
X509Certificate certificate = (X509Certificate) cf.generateCertificate(fis);
```

### Complete Example (Native Java)

```java
import java.io.FileInputStream;
import java.security.PublicKey;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;

public class NativeCertificateLoader {

    public static X509Certificate loadCertificate(String filePath) throws Exception {
        try (FileInputStream fis = new FileInputStream(filePath)) {
            CertificateFactory cf = CertificateFactory.getInstance("X.509");
            return (X509Certificate) cf.generateCertificate(fis);
        }
    }

    public static PublicKey loadPublicKey(String filePath) throws Exception {
        X509Certificate certificate = loadCertificate(filePath);
        return certificate.getPublicKey();
    }
}
```

---

## PEM Format Reference

### X.509 Certificate

```
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAJC1HiIAZAiUMA0Gcz...
-----END CERTIFICATE-----
```

- Parsed as **X509CertificateHolder** by PEMParser.
- Can also be loaded by CertificateFactory.

### Certificate Chain

```
-----BEGIN CERTIFICATE-----
(end-entity certificate)
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
(intermediate certificate)
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
(root certificate)
-----END CERTIFICATE-----
```

- Multiple certificates in one file.
- Parse each sequentially with PEMParser.
