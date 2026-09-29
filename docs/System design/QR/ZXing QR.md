# ZXing QR Code Generation Dependencies Handle

## Components
<img width="1536" height="1024" alt="Designer (15)" src="https://github.com/user-attachments/assets/6f213bf2-f013-4130-8465-10b682ddf86a" />

### 1. DynamicQrImageUtil
#### Purpose
Acts as the central QR image generation manager that orchestrates the entire QR code creation process.

#### Dependencies
```java
QRCodeWriter
EncodeHintType
ErrorCorrectionLevel
BitMatrix
MatrixToImageWriter
MatrixToImageConfig
BufferedImage
Graphics2D
ImageIO
```

#### Functions Used
Generate QR image:
```java
DynamicQrImageUtil.generateQrImage(String qrData,int imageSize);
```

#### Responsibilities
- Central coordinator for QR image generation
- Accepts QR payload and image size
- Configures QR encoding options
- Generates QR BitMatrix
- Converts BitMatrix into BufferedImage
- Overlays logo image
- Encodes image as PNG
- Returns Base64 encoded image

### 2. QRCodeWriter
#### Purpose
Encodes QR payload data into a QR code matrix.

#### Dependencies
```java
BarcodeFormat
EncodeHintType
ErrorCorrectionLevel
BitMatrix
```

#### Functions Used
Encode QR payload:
```java
QRCodeWriter.encode(qrData,BarcodeFormat.QR_CODE,imageSize,imageSize,hints);
```

#### Responsibilities
- Converts text into QR format.
- Applies encoding hints
- Generates BitMatrix

### 3. BitMatrix
#### Purpose
Represents the QR code as a 2D matrix

#### Functions Used
Get width :
```java
BitMatrix.getWidth();
```

Get height:
```java
BitMatrix.getHeight();
```

#### Responsibilities
- Stores QR pattern data
- Acts as input for image conversion

### 4. MatrixToImageWriter
#### Purpose
Converts BitMatrix into BufferedImage

#### Dependencies
```java
BitMatrix
MatrixToImageConfig
BufferedImage
```

#### Functions Used
Convert to image:
```java
MatrixToImageWriter.toBufferedImage(BitMatrix,MatrixToImageConfig);
```

#### Responsibilities
- Converts QR matrix into image format
- Produces BufferedImage output

### 5. MatrixToImageConfig
#### Purpose
Defines QR image color configuration.

#### Functions Used
Create configuration:
```java
new MatrixToImageConfig(MatrixToImageConfig.BLACK,MatrixToImageConfig.WHITE);
```

#### Responsibilities :
- Defines foreground color
- Defines background color

### 6. BufferedImage
#### Purpose
Represents the QR image in memory

#### Dependencies
```java
Graphics2D
ImageIO
```

#### Functions Used
Create image:
```java
new BufferedImage(width,height,BufferedImage.TYPE_INT_ARGB);
```

#### Responsibilities
- Holds QR image data
- Supports image manipulation

### 7. Graphics2D
#### Purpose
Handles logo overlay rendering

#### Functions Used
Draw QR image:
```java
new BufferedImage(width,height,BufferedImage.TYPE_INT_ARGB);
```

Draw logo image:
```java
Graphics2D.drawImage(logoImage,x,y,null);
```

#### Responsibilities
- Renders QR image
- Places logo at center

### 8. ImageIO
#### Purpose
Handles image loading and PNG encoding.

#### Functions Used
Load logo:
```java
ImageIO.read(InputStream);
```

#### Responsibilities
- Loads logo image
- Writes PNG image

### 9. DynamicQrCodeUtil
#### Purpose
Generates the QR payload string before image generation

#### Functions Used
Generate payload:
```java
DynamicQrCodeUtil.getQrString(...);
```

Calculate CRC:
```java
DynamicQrCodeUtil.calculateCRC16(...);
```

#### Responsibilities
- Builds QR payload
- Calculates CRC
- Returns QR data string
