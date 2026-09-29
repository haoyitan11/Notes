# ZXing QR Code Generation Dependencies Handle

## Components
![QR Generation Flow Diagram](diagram-placeholder)

### 1. DynamicQrImageUtil
#### Purpose
Acts as the central QR image generation manager that orchestrates the entire QR code creation process.

### Dependencies
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

### Functions Used
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
