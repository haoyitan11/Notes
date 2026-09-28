# PrivateKey dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/37d28281-919b-4266-9b8d-886703c73fdb" />

### 1. BouncyCastleProvider
Function: Acts as the cryptographic provider.
- Registered through Security.addProvider(...). 
- Provides support for PEM processing and RSA cryptographic operations. 
- Used by PEMParser and JcaPEMKeyConverter.

### 2. Files & Paths
Function: Reads the PEM file from the filesystem.
- Reads the PEM file from the filesystem.
- Loads the file content into memory.
- Converts the file content into a String.
- May throw IOException
  
### 3. Reader (StringReader)
Function: Provides character stream access to PEM content.
- Wraps the PEM String.
- Provides character stream access to the PEM content.
- Supplies the content to PEMParser.

### 4. PEMParser
Function: Parses the PEM formatted content.
- Reads PEM content from the Reader. 
- Parses the PEM structure. 
- Produces either PEMKeyPair or PrivateKeyInfo.

### 5A. PEMKeyPair
Function: Represents a parsed key pair (PKCS#1).
- Represents a parsed key pair containing both public and private key information.
- Contains both public and private key information.
- Provides access to PrivateKeyInfo via getPrivateKeyInfo().

### 5B. PrivateKeyInfo
Function: Represents the ASN.1 encoded private key structure (PKCS#8).
- Represents the ASN.1 encoded private key (PKCS#8).
- Can be obtained directly from PEMParser.
- Can be extracted from PEMKeyPair.
- Acts as the input for JcaPEMKeyConverter.

### 6. JcaPEMKeyConverter
Function: Converts Bouncy Castle key objects into Java Security objects.
-	Converts PrivateKeyInfo into a Java Security PrivateKey.
-	Integrates with BouncyCastleProvider.
-	Bridges Bouncy Castle objects and Java Security APIs.

### 7. PrivateKey
Function: Final RSA private key output.
- Final RSA private key output.
- Returned by RSAUtility.getPrivateKey().
- Used for digital signature generation.
- Passed into Signature.initSign().
