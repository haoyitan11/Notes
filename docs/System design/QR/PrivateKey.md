# PrivateKey Dependencies Handle

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/37d28281-919b-4266-9b8d-886703c73fdb" />

### 1. BouncyCastleProvider

#### Purpose

Acts as the cryptographic provider for PEM parsing and RSA key conversion.

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
- Used by **PEMParser** and **JcaPEMKeyConverter**.

---

### 2. Files & Paths

#### Purpose

Loads the PEM file content from the filesystem.

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

- Locates PEM file by path.
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

Parses PEM-formatted content into Bouncy Castle key objects.

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
- Detects PEM type (PKCS#1, PKCS#8, encrypted key).
- Converts PEM text into Java objects.
- Returns **PEMKeyPair** (PKCS#1) or **PrivateKeyInfo** (PKCS#8).
- Used by key conversion logic to obtain key information.

---

### 5A. PEMKeyPair

#### Purpose

Represents a PKCS#1 RSA key pair parsed from PEM.

#### Dependencies

```java
org.bouncycastle.openssl.PEMKeyPair
```

#### Actual Usage

```java
if (obj instanceof PEMKeyPair) {
    PEMKeyPair pemKeyPair = (PEMKeyPair) obj;
    PrivateKeyInfo privateKeyInfo = pemKeyPair.getPrivateKeyInfo();
}
```

#### Responsibilities

- Contains RSA public key information.
- Contains RSA private key information.
- Provides access to **PrivateKeyInfo**.
- Used when PEM contains PKCS#1 format (`-----BEGIN RSA PRIVATE KEY-----`).

---

### 5B. PrivateKeyInfo

#### Purpose

Represents the ASN.1 private key structure.

#### Dependencies

```java
org.bouncycastle.asn1.pkcs.PrivateKeyInfo
```

#### Actual Usage

Obtained directly from PKCS#8 PEM:

```java
if (obj instanceof PrivateKeyInfo) {
    PrivateKeyInfo privateKeyInfo = (PrivateKeyInfo) obj;
}
```

Or from PEMKeyPair (PKCS#1):

```java
PrivateKeyInfo privateKeyInfo = pemKeyPair.getPrivateKeyInfo();
```

#### Responsibilities

- Holds private key metadata.
- Holds ASN.1 encoded key data.
- Intermediate representation used by Bouncy Castle.
- Used by **JcaPEMKeyConverter** for PrivateKey generation.

---

### 6. JcaPEMKeyConverter

#### Purpose

Converts Bouncy Castle key objects into standard Java Security objects.

#### Dependencies

```java
org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter
```

#### Actual Usage

```java
JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
PrivateKey privateKey = converter.getPrivateKey(privateKeyInfo);
```

#### Responsibilities

- Converts **PrivateKeyInfo** to Java **PrivateKey**.
- Uses the registered Bouncy Castle provider.
- Bridges Bouncy Castle APIs and Java Security APIs.
- Returns standard **java.security.PrivateKey**.

---

### 7. PrivateKey

#### Purpose

Final RSA private key object used by application code.

#### Dependencies

```java
java.security.PrivateKey
```

#### Actual Usage

Returned from converter:

```java
PrivateKey privateKey = converter.getPrivateKey(privateKeyInfo);
```

Used for signing:

```java
Signature signature = Signature.getInstance("SHA256withRSA");
signature.initSign(privateKey);
```

Used for decryption:

```java
Cipher cipher = Cipher.getInstance("RSA");
cipher.init(Cipher.DECRYPT_MODE, privateKey);
```

#### Responsibilities

- Digital signature generation.
- Signature verification workflows.
- RSA decryption operations.

---

## Execution Flow

1. **BouncyCastleProvider** is registered with Java Security:

   ```java
   Security.addProvider(new BouncyCastleProvider());
   ```

2. **Files.readString()** reads PEM file content:

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

5. **Object type** is checked and **PrivateKeyInfo** is obtained:

   ```java
   PrivateKeyInfo privateKeyInfo;
   
   if (obj instanceof PEMKeyPair) {
       // PKCS#1 format: -----BEGIN RSA PRIVATE KEY-----
       PEMKeyPair pemKeyPair = (PEMKeyPair) obj;
       privateKeyInfo = pemKeyPair.getPrivateKeyInfo();
   } else if (obj instanceof PrivateKeyInfo) {
       // PKCS#8 format: -----BEGIN PRIVATE KEY-----
       privateKeyInfo = (PrivateKeyInfo) obj;
   }
   ```

6. **JcaPEMKeyConverter** converts to Java PrivateKey:

   ```java
   JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
   PrivateKey privateKey = converter.getPrivateKey(privateKeyInfo);
   ```

7. **PrivateKey** is used for cryptographic operations:

   ```java
   // Signing
   Signature signature = Signature.getInstance("SHA256withRSA");
   signature.initSign(privateKey);
   signature.update(data);
   byte[] signedData = signature.sign();
   
   // Decryption
   Cipher cipher = Cipher.getInstance("RSA");
   cipher.init(Cipher.DECRYPT_MODE, privateKey);
   byte[] decryptedData = cipher.doFinal(encryptedData);
   ```

---

## Complete Example

```java
import org.bouncycastle.asn1.pkcs.PrivateKeyInfo;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.openssl.PEMKeyPair;
import org.bouncycastle.openssl.PEMParser;
import org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter;

import java.io.Reader;
import java.io.StringReader;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.PrivateKey;
import java.security.Security;

public class PEMPrivateKeyLoader {

    static {
        Security.addProvider(new BouncyCastleProvider());
    }

    public static PrivateKey loadPrivateKey(String filePath) throws Exception {
        // Read PEM file
        String pemContent = Files.readString(Paths.get(filePath));
        
        // Parse PEM content
        Reader reader = new StringReader(pemContent);
        PEMParser parser = new PEMParser(reader);
        Object obj = parser.readObject();
        parser.close();
        
        // Extract PrivateKeyInfo
        PrivateKeyInfo privateKeyInfo;
        if (obj instanceof PEMKeyPair) {
            PEMKeyPair pemKeyPair = (PEMKeyPair) obj;
            privateKeyInfo = pemKeyPair.getPrivateKeyInfo();
        } else if (obj instanceof PrivateKeyInfo) {
            privateKeyInfo = (PrivateKeyInfo) obj;
        } else {
            throw new IllegalArgumentException("Unsupported PEM format");
        }
        
        // Convert to Java PrivateKey
        JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
        return converter.getPrivateKey(privateKeyInfo);
    }
}
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
