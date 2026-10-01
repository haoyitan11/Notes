# ZXing QR Code Generation Dependencies Handle

## Overview
<img width="3200" height="2918" alt="image" src="https://github.com/user-attachments/assets/9b705931-a138-4dc4-8fff-9b3b99d12cbb" />

This document provides a complete guide for the ZXing QR code generation implementation in the application. It covers two processing layers:

1. **QR Payload Generation** - Building the QR data string with transaction information

2. **QR Image Generation** - Converting payload to visual QR code image with logo overlay

---

# Part 1: QR Payload Generation

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/fd734ce8-8d3f-4381-b260-427a35c12e5c" />

## Purpose

Builds the QR data string containing transaction, merchant, and security information before image generation.

---

## Components

### 1.1 DynamicQrCodeUtil

#### Purpose

Acts as the central QR payload builder that constructs the complete QR data string.

#### Dependencies

```java
DateUtil
```

#### Actual Usage

```java
DynamicQrCodeUtil qrUtil = new DynamicQrCodeUtil();
String qrData = qrUtil.getQrString(tid, merchantId, stan, needCardAuth, amount, merchantName, channel);
```

#### Responsibilities

- Central coordinator for QR payload generation.
- Integrates **DateUtil** (generates Julian timestamp in hex).
- Assembles header, version, and transaction identifiers.
- Formats merchant ID with padding.
- Applies payment acceptance flags based on PIN requirement.
- Generates random hex identifier for uniqueness.
- Formats merchant name to fixed length.
- Calculates CRC16 checksum for data integrity.
- Returns complete QR payload string.

#### Configuration Source

```java
public class DynamicQrCodeUtil {

    private static String header = "QRPAY";
    private static String version = "0";
    private static String operation = "0"; // 0 - Sale
    private static String paymentAcceptWithPin = "8001";
    private static String paymentAcceptWithoutPin = "0001";
    private static String currency = "0"; // 0 - SGD

    public String getQrString(String tid, String merchantId, String stan, boolean needCardAuth, 
                              String amount, String merchantName, int channel) {
        
        String paymentAccept = paymentAcceptWithoutPin;
        if (needCardAuth) {
            paymentAccept = paymentAcceptWithPin;
        }
        
        StringBuffer qrData = new StringBuffer();
        qrData.append(header)
              .append(version)
              .append(tid)
              .append(String.format("%15s", merchantId).replace(' ', '#'))
              .append(stan)
              .append(DateUtil.getCurrentJulianTimeStampInHex())
              .append(operation)
              .append(channel)
              .append(getAmountWith8Digit(amount))
              .append(paymentAccept)
              .append(currency)
              .append(getRandomHexString(8))
              .append(getQrMerchantName(merchantName));

        String crc16 = calculateCRC16(qrData.toString());
        qrData.append(crc16);
        return qrData.toString();
    }
}
```

---

### 1.2 DateUtil

#### Purpose

Provides date and time utility functions including Julian timestamp generation.

#### Dependencies

```java
Calendar
TimeZone
```

#### Actual Usage

```java
String timestamp = DateUtil.getCurrentJulianTimeStampInHex();
```

#### Responsibilities

- Integrates **Calendar** (retrieves current time in GMT).
- Integrates **TimeZone** (ensures consistent GMT timezone).
- Calculates Julian date from Unix timestamp.
- Converts Julian timestamp to hexadecimal string.
- Returns 8-character hex timestamp.
- Used by **DynamicQrCodeUtil** for QR payload timestamp.

#### Configuration Source

```java
public static String getCurrentJulianTimeStampInHex() {
    Calendar cal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
    int time = (int) (cal.getTimeInMillis() / 1000);
    int julianDate = time + 16 * 60 * 60 - 9131 * 86400;
    return Integer.toHexString(julianDate);
}
```

---

### 1.3 CRC16 Calculator

#### Purpose

Calculates CRC16 checksum for QR data integrity verification.

#### Dependencies

```java
None (algorithm implementation)
```

#### Actual Usage

```java
String crc16 = DynamicQrCodeUtil.calculateCRC16(qrData.toString());
```

#### Responsibilities

- Implements CRC16 polynomial lookup table.
- Processes each byte of QR data string.
- Calculates checksum using standard CRC16 algorithm.
- Returns 4-character hexadecimal checksum.
- Appended to QR payload for integrity verification.

#### Configuration Source

```java
public static String calculateCRC16(String qrData) { 
    int[] table = {
        0x0000, 0xC0C1, 0xC181, 0x0140, 0xC301, 0x03C0, 0x0280, 0xC241,
        0xC601, 0x06C0, 0x0780, 0xC741, 0x0500, 0xC5C1, 0xC481, 0x0440,
        // ... complete lookup table
    };

    byte[] bytes = qrData.getBytes();
    int crc = 0x0000;
    for (byte b : bytes) {
        crc = (crc >>> 8) ^ table[(crc ^ b) & 0xff];
    }
    return String.format("%04x", crc & 0xffff);
}
```

---

### 1.4 Helper Methods

#### Purpose

Provides formatting utilities for QR payload fields.

#### getAmountWith8Digit

```java
public static String getAmountWith8Digit(String amount) {
    String retAmount = null;
    if (amount.length() > 8) {
        retAmount = amount.substring(Math.max(0, amount.length() - 8));
    } else {
        retAmount = String.format("%8s", amount).replace(' ', '0');
    }
    return retAmount;
}
```

#### getRandomHexString

```java
public static String getRandomHexString(int numchars) {
    Random r = new Random();
    StringBuffer sb = new StringBuffer();
    while (sb.length() < numchars) {
        sb.append(Integer.toHexString(r.nextInt()));
    }
    return sb.toString().substring(0, numchars);
}
```

#### getQrMerchantName

```java
public static String getQrMerchantName(String name) {
    String returnMerchantName = "";
    if (name == null || name.length() == 0) {
        returnMerchantName = new String(new char[17]).replace('\0', ' ');
    } else if (0 < name.length() && name.length() < 17) {
        returnMerchantName = String.format("%-17s", name);
    } else {
        returnMerchantName = name.substring(0, 17);
    }
    return returnMerchantName;
}
```

#### Responsibilities

- **getAmountWith8Digit**: Formats amount to exactly 8 digits with leading zeros.
- **getRandomHexString**: Generates random hex string of specified length for uniqueness.
- **getQrMerchantName**: Formats merchant name to exactly 17 characters.

---

## QR Payload Structure

| Field | Length | Description | Example |
|-------|--------|-------------|---------|
| Header | 8 | Fixed identifier | QRPAY |
| Version | 1 | Protocol version | 0 |
| TID | 8 | Terminal ID | ABC12345 |
| Merchant ID | 15 | Padded with # | ###MERCHANT1234 |
| STAN | 6 | System Trace Number | 000001 |
| Timestamp | 8 | Julian timestamp (hex) | 2a3b4c5d |
| Operation | 1 | Transaction type | 0 (Sale) |
| Channel | 1 | Payment channel | 2 (NPS), 0 (EFT POS) |
| Amount | 8 | Transaction amount | 00001000 |
| Payment Accept | 4 | PIN requirement flag | 8001/0001 |
| Currency | 1 | Currency code | 0 (SGD) |
| Random Hex | 8 | Unique identifier | a1b2c3d4 |
| Merchant Name | 17 | Padded name | SHOP NAME________ |
| CRC16 | 4 | Checksum | f3e2 |

---

# Part 2: QR Image Generation

<img width="3200" height="3024" alt="image" src="https://github.com/user-attachments/assets/58466446-a815-487c-9f65-7ca7ea626bf4" />

## Purpose

Converts QR payload string into visual QR code image with logo overlay and Base64 encoding.

---

## Components

### 2.1 DynamicQrImageUtil

#### Purpose

Acts as the central QR image generation manager that orchestrates the entire QR code creation process.

#### Dependencies

```java
QRCodeWriter
EncodeHintType
ErrorCorrectionLevel
MatrixToImageConfig
MatrixToImageWriter
BitMatrix
BufferedImage
Graphics2D
ImageIO
ImageWriter
ImageWriteParam
Base64.Encoder
```

#### Actual Usage

```java
DynamicQrImageUtil imageUtil = new DynamicQrImageUtil();
String base64Image = imageUtil.generateQrImage(qrData, 300);
```

#### Responsibilities

- Central coordinator for QR image generation.
- Integrates **QRCodeWriter** (encodes data to BitMatrix).
- Integrates **MatrixToImageWriter** (converts BitMatrix to image).
- Integrates **Graphics2D** (overlays logo on QR code).
- Integrates **ImageIO** (loads logo and writes PNG).
- Integrates **Base64.Encoder** (encodes image bytes).
- Configures error correction level for logo tolerance.
- Returns Base64-encoded PNG image string.

#### Configuration Source

```java
public class DynamicQrImageUtil {
    private static final Logger logger = LoggerFactory.getLogger(DynamicQrImageUtil.class);
    static String path = "/images/logo.png";

    public String generateQrImage(String qrData, int imageSize) {
        Map<EncodeHintType, ErrorCorrectionLevel> hints = new HashMap<>();
        hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);

        QRCodeWriter writer = new QRCodeWriter();
        BitMatrix bitMatrix = null;
        ByteArrayOutputStream baos = new ByteArrayOutputStream();

        try {
            bitMatrix = writer.encode(qrData, BarcodeFormat.QR_CODE, imageSize, imageSize, hints);
            
            MatrixToImageConfig config = new MatrixToImageConfig(
                MatrixToImageConfig.BLACK, MatrixToImageConfig.WHITE);
            BufferedImage qrImage = MatrixToImageWriter.toBufferedImage(bitMatrix, config);
            
            BufferedImage logoImage = ImageIO.read(this.getClass().getResourceAsStream(path));
            
            int deltaHeight = qrImage.getHeight() - logoImage.getHeight();
            int deltaWidth = qrImage.getWidth() - logoImage.getWidth();

            BufferedImage combined = new BufferedImage(
                qrImage.getHeight(), qrImage.getWidth(), BufferedImage.TYPE_INT_ARGB);
            Graphics2D g = (Graphics2D) combined.getGraphics();
            g.drawImage(qrImage, 0, 0, null);
            g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_OVER, 1f));
            g.drawImage(logoImage, 
                (int) Math.round(deltaWidth / 2), 
                (int) Math.round(deltaHeight / 2), null);

            // PNG encoding with compression...
            Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("png");
            ImageWriter pngWriter = writers.next();
            ImageWriteParam param = pngWriter.getDefaultWriteParam();
            if (param.canWriteCompressed()) {
                param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
                param.setCompressionQuality(0.1F);
            }
            // ... write to output stream
            
        } catch (WriterException | IOException e) {
            logger.error("Exception occurred", e);
        }

        byte[] dataInBytes = baos.toByteArray();
        byte[] encodedBytes = java.util.Base64.getEncoder().encode(dataInBytes);
        return new String(encodedBytes);
    }
}
```

---

### 2.2 QRCodeWriter

#### Purpose

Encodes text data into a QR code represented as a BitMatrix.

#### Dependencies

```java
BarcodeFormat
EncodeHintType
BitMatrix
```

#### Actual Usage

```java
QRCodeWriter writer = new QRCodeWriter();
BitMatrix bitMatrix = writer.encode(qrData, BarcodeFormat.QR_CODE, imageSize, imageSize, hints);
```

#### Responsibilities

- Converts string data into QR code format.
- Integrates **BarcodeFormat** (specifies QR_CODE type).
- Integrates **EncodeHintType** (applies encoding hints).
- Generates **BitMatrix** representing the QR code pattern.
- Handles QR code version selection based on data size.
- Throws **WriterException** if encoding fails.
- Used by **DynamicQrImageUtil** for matrix generation.

---

### 2.3 EncodeHintType

#### Purpose

Enum that specifies encoding configuration hints for QR code generation.

#### Dependencies

```java
None (enum type)
```

#### Actual Usage

```java
Map<EncodeHintType, ErrorCorrectionLevel> hints = new HashMap<>();
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
```

#### Responsibilities

- Provides encoding configuration options.
- **ERROR_CORRECTION**: Sets error correction level.
- **CHARACTER_SET**: Specifies text encoding (UTF-8).
- **MARGIN**: Sets quiet zone size around QR code.
- Passed to **QRCodeWriter.encode()** method.

#### Available Hints

| Hint Type | Value Type | Description |
|-----------|------------|-------------|
| ERROR_CORRECTION | ErrorCorrectionLevel | Data recovery capability |
| CHARACTER_SET | String | Text encoding (UTF-8, ISO-8859-1) |
| MARGIN | Integer | Quiet zone modules around QR |
| QR_VERSION | Integer | Force specific QR version (1-40) |

---

### 2.4 ErrorCorrectionLevel

#### Purpose

Enum that defines the error correction capability of the QR code.

#### Dependencies

```java
None (enum type)
```

#### Actual Usage

```java
hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
```

#### Responsibilities

- Defines data recovery capability.
- Higher levels allow more damage tolerance.
- Level H required for logo overlay (30% damage tolerance).
- Used by **QRCodeWriter** during encoding.

#### Available Levels

| Level | Recovery Capacity | Use Case |
|-------|-------------------|----------|
| L | ~7% | Minimal damage tolerance |
| M | ~15% | Standard tolerance |
| Q | ~25% | Better tolerance |
| H | ~30% | **Used for logo overlay** |

---

### 2.5 BitMatrix

#### Purpose

Represents the QR code as a 2D matrix of boolean values (black/white modules).

#### Dependencies

```java
None (core data structure)
```

#### Actual Usage

```java
BitMatrix bitMatrix = writer.encode(qrData, BarcodeFormat.QR_CODE, imageSize, imageSize, hints);
int width = bitMatrix.getWidth();
int height = bitMatrix.getHeight();
boolean module = bitMatrix.get(x, y);
```

#### Responsibilities

- Stores QR code pattern as 2D boolean array.
- Represents black modules as true, white as false.
- Provides access to individual module positions.
- Created by **QRCodeWriter.encode()**.
- Used as input for **MatrixToImageWriter**.

---

### 2.6 BarcodeFormat

#### Purpose

Enum that specifies the type of barcode to generate.

#### Dependencies

```java
None (enum type)
```

#### Actual Usage

```java
bitMatrix = writer.encode(qrData, BarcodeFormat.QR_CODE, imageSize, imageSize, hints);
```

#### Responsibilities

- Defines barcode type for encoding.
- QR_CODE value used for QR code generation.
- Passed to **QRCodeWriter.encode()** method.

#### Available Values

```java
BarcodeFormat.QR_CODE        // QR Code 2D barcode
BarcodeFormat.AZTEC          // Aztec 2D barcode
BarcodeFormat.DATA_MATRIX    // Data Matrix 2D barcode
BarcodeFormat.PDF_417        // PDF417 barcode
BarcodeFormat.CODE_128       // Code 128 1D barcode
```

---

### 2.7 MatrixToImageWriter

#### Purpose

Converts BitMatrix to image formats (BufferedImage, file, stream).

#### Dependencies

```java
BitMatrix
MatrixToImageConfig
BufferedImage
```

#### Actual Usage

```java
MatrixToImageConfig config = new MatrixToImageConfig(MatrixToImageConfig.BLACK, MatrixToImageConfig.WHITE);
BufferedImage qrImage = MatrixToImageWriter.toBufferedImage(bitMatrix, config);
```

#### Responsibilities

- Converts **BitMatrix** to visual image format.
- Integrates **MatrixToImageConfig** for color configuration.
- Produces **BufferedImage** output.
- Part of ZXing javase module.
- Used by **DynamicQrImageUtil** for image conversion.

#### Alternative Methods

```java
// Write directly to file
MatrixToImageWriter.writeToPath(bitMatrix, "PNG", Path.of("qr.png"));

// Write to stream
MatrixToImageWriter.writeToStream(bitMatrix, "PNG", outputStream);
```

---

### 2.8 MatrixToImageConfig

#### Purpose

Configures colors for QR code image rendering.

#### Dependencies

```java
None (configuration class)
```

#### Actual Usage

```java
MatrixToImageConfig config = new MatrixToImageConfig(MatrixToImageConfig.BLACK, MatrixToImageConfig.WHITE);
```

#### Responsibilities

- Defines foreground color (QR modules).
- Defines background color (empty space).
- Colors specified as ARGB integers.
- Passed to **MatrixToImageWriter** methods.

#### Configuration Options

```java
// Default black and white
MatrixToImageConfig config = new MatrixToImageConfig(MatrixToImageConfig.BLACK, MatrixToImageConfig.WHITE);

// Custom colors (ARGB format)
MatrixToImageConfig customConfig = new MatrixToImageConfig(0xFF0000FF, 0xFFFFFFFF); // Blue QR on white
```

---

### 2.9 BufferedImage

#### Purpose

Represents the QR code as a raster image in memory.

#### Dependencies

```java
Graphics2D
ImageIO
```

#### Actual Usage

Create combined image canvas:

```java
BufferedImage combined = new BufferedImage(qrImage.getHeight(), qrImage.getWidth(), BufferedImage.TYPE_INT_ARGB);
```

Get graphics context:

```java
Graphics2D g = (Graphics2D) combined.getGraphics();
```

#### Responsibilities

- Holds QR code as pixel data.
- Supports image manipulation (logo overlay).
- Provides **Graphics2D** for drawing operations.
- Can be encoded to PNG/JPEG formats.
- Part of Java AWT (java.awt.image).

---

### 2.10 Graphics2D

#### Purpose

Provides 2D rendering capabilities for image composition.

#### Dependencies

```java
BufferedImage
AlphaComposite
```

#### Actual Usage

```java
Graphics2D g = (Graphics2D) combined.getGraphics();
g.drawImage(qrImage, 0, 0, null);
g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_OVER, 1f));
g.drawImage(logoImage, (int) Math.round(deltaWidth / 2), (int) Math.round(deltaHeight / 2), null);
```

#### Responsibilities

- Draws QR code image to canvas at (0, 0).
- Integrates **AlphaComposite** for transparency handling.
- Overlays logo image at center position.
- Manages image composition layers.
- Part of Java AWT (java.awt).

---

### 2.11 AlphaComposite

#### Purpose

Controls how overlapping images are blended together.

#### Dependencies

```java
None (Java AWT class)
```

#### Actual Usage

```java
g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_OVER, 1f));
```

#### Responsibilities

- Defines blending mode for image composition.
- SRC_OVER mode places logo over QR code.
- Alpha value (1f) maintains full opacity.
- Used by **Graphics2D** for logo overlay.

---

### 2.12 ImageIO

#### Purpose

Handles image read/write operations for logo loading and PNG output.

#### Dependencies

```java
BufferedImage
InputStream
ImageWriter
```

#### Actual Usage

Load logo image:

```java
BufferedImage logoImage = ImageIO.read(this.getClass().getResourceAsStream(path));
```

Get PNG writer:

```java
Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("png");
ImageWriter pngWriter = writers.next();
```

Create output stream:

```java
ImageOutputStream ios = ImageIO.createImageOutputStream(baos);
```

#### Responsibilities

- Loads logo image from classpath resources.
- Provides **ImageWriter** for PNG encoding.
- Creates **ImageOutputStream** for byte output.
- Part of Java ImageIO (javax.imageio).

---

### 2.13 ImageWriter and ImageWriteParam

#### Purpose

Handles PNG encoding with compression configuration.

#### Dependencies

```java
ImageWriteParam
ImageOutputStream
IIOImage
IIOMetadata
```

#### Actual Usage

```java
Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("png");
ImageWriter pngWriter = writers.next();

ImageWriteParam param = pngWriter.getDefaultWriteParam();
if (param.canWriteCompressed()) {
    param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
    param.setCompressionQuality(0.1F);
}

try (ImageOutputStream ios = ImageIO.createImageOutputStream(baos)) {
    pngWriter.setOutput(ios);
    IIOMetadata metadata = pngWriter.getDefaultImageMetadata(
        new ImageTypeSpecifier(combined), param);
    pngWriter.write(null, new javax.imageio.IIOImage(combined, null, metadata), param);
}
pngWriter.dispose();
```

#### Responsibilities

- Integrates **ImageWriteParam** for compression settings.
- Sets compression quality (0.1F = high compression).
- Integrates **ImageOutputStream** for byte output.
- Writes **IIOImage** with metadata to output.
- Disposes writer resources after completion.

---

### 2.14 Base64.Encoder

#### Purpose

Encodes PNG image bytes to Base64 string for API response.

#### Dependencies

```java
None (Java standard library)
```

#### Actual Usage

```java
byte[] dataInBytes = baos.toByteArray();
byte[] encodedBytes = java.util.Base64.getEncoder().encode(dataInBytes);
return new String(encodedBytes);
```

#### Responsibilities

- Converts binary PNG data to text format.
- Enables embedding QR image in JSON responses.
- Returns Base64 string suitable for API transmission.

---

## Execution Flow

1. **Application** (caller) builds **QR payload** using **DynamicQrCodeUtil**:

   ```java
   DynamicQrCodeUtil qrUtil = new DynamicQrCodeUtil();
   String qrData = qrUtil.getQrString(tid, merchantId, stan, needCardAuth, amount, merchantName, channel);
   ```

   - DynamicQrCodeUtil integrates **DateUtil** for timestamp.
   - DynamicQrCodeUtil builds payload string with all fields.
   - DynamicQrCodeUtil calculates **CRC16** checksum.
   - Returns complete QR data string.

2. **DynamicQrImageUtil.generateQrImage(qrData, imageSize)** is called:

   ```java
   DynamicQrImageUtil imageUtil = new DynamicQrImageUtil();
   String base64Image = imageUtil.generateQrImage(qrData, 300);
   ```

   - Configures **EncodeHintType** with **ErrorCorrectionLevel.H**.
   - Creates **QRCodeWriter** instance.

3. **QRCodeWriter.encode()** encodes data to matrix:

   ```java
   QRCodeWriter writer = new QRCodeWriter();
   BitMatrix bitMatrix = writer.encode(qrData, BarcodeFormat.QR_CODE, imageSize, imageSize, hints);
   ```

   - QRCodeWriter integrates **BarcodeFormat.QR_CODE**.
   - QRCodeWriter integrates **EncodeHintType** hints.
   - Returns **BitMatrix** representing QR pattern.

4. **MatrixToImageWriter.toBufferedImage()** converts matrix to image:

   ```java
   MatrixToImageConfig config = new MatrixToImageConfig(MatrixToImageConfig.BLACK, MatrixToImageConfig.WHITE);
   BufferedImage qrImage = MatrixToImageWriter.toBufferedImage(bitMatrix, config);
   ```

   - MatrixToImageWriter integrates **MatrixToImageConfig** for colors.
   - Returns **BufferedImage** of QR code.

5. **ImageIO.read()** loads logo image from resources:

   ```java
   BufferedImage logoImage = ImageIO.read(this.getClass().getResourceAsStream("/images/logo.png"));
   ```

6. **Graphics2D** overlays logo on QR image:

   ```java
   int deltaHeight = qrImage.getHeight() - logoImage.getHeight();
   int deltaWidth = qrImage.getWidth() - logoImage.getWidth();
   
   BufferedImage combined = new BufferedImage(qrImage.getHeight(), qrImage.getWidth(), BufferedImage.TYPE_INT_ARGB);
   Graphics2D g = (Graphics2D) combined.getGraphics();
   g.drawImage(qrImage, 0, 0, null);
   g.setComposite(AlphaComposite.getInstance(AlphaComposite.SRC_OVER, 1f));
   g.drawImage(logoImage, (int) Math.round(deltaWidth / 2), (int) Math.round(deltaHeight / 2), null);
   ```

   - Creates combined **BufferedImage** canvas.
   - Draws QR code at (0, 0).
   - Sets **AlphaComposite.SRC_OVER** for overlay.
   - Draws logo at center position.

7. **ImageWriter** encodes combined image to PNG:

   ```java
   Iterator<ImageWriter> writers = ImageIO.getImageWritersByFormatName("png");
   ImageWriter pngWriter = writers.next();
   ImageWriteParam param = pngWriter.getDefaultWriteParam();
   param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
   param.setCompressionQuality(0.1F);
   pngWriter.write(null, new IIOImage(combined, null, metadata), param);
   ```

   - Configures **ImageWriteParam** with compression.
   - Writes to **ByteArrayOutputStream**.

8. **Base64.Encoder** encodes PNG bytes:

   ```java
   byte[] dataInBytes = baos.toByteArray();
   byte[] encodedBytes = java.util.Base64.getEncoder().encode(dataInBytes);
   return new String(encodedBytes);
   ```

   - Returns Base64 string for API response.

9. **Application** receives Base64-encoded QR image:

   ```java
   String qrImage = imageUtil.generateQrImage(qrData, 300);
   // Use in API response: { "qrImage": "iVBORw0KGgoAAAANSUhEUgAA..." }
   ```

---

# Configuration Summary

## Class Summary

| Class Name | Package | Layer | Purpose |
|------------|---------|-------|---------|
| DynamicQrCodeUtil | com.qr.common.utils | Payload | Builds QR data string |
| DynamicQrImageUtil | com.qr.common.utils | Image | Generates QR image with logo |
| DateUtil | com.qr.common.utils | Utility | Timestamp utilities |
| QRCodeWriter | com.google.zxing.qrcode | ZXing | Encodes data to BitMatrix |
| BitMatrix | com.google.zxing.common | ZXing | 2D QR code representation |
| MatrixToImageWriter | com.google.zxing.client.j2se | ZXing | Converts matrix to image |
| MatrixToImageConfig | com.google.zxing.client.j2se | ZXing | Image color configuration |

---

## Error Handling

### WriterException

Thrown by QRCodeWriter.encode() when:

- Data is too large for QR code capacity
- Invalid characters for selected encoding
- Invalid encoding hints provided

```java
try {
    bitMatrix = writer.encode(qrData, BarcodeFormat.QR_CODE, imageSize, imageSize, hints);
} catch (WriterException e) {
    logger.error("WriterException occurred", e);
}
```

### IOException

Thrown during image operations:

- Logo image not found in resources
- PNG encoding failure
- Stream write errors

```java
try {
    BufferedImage logoImage = ImageIO.read(this.getClass().getResourceAsStream(path));
    // ... image processing
} catch (IOException e) {
    logger.error("IOException occurred", e);
}
```
