# File Handling Complete Implementation Guide

## Overview

This document covers file upload and download operations in batch applications using Spring Framework:

1. **File Download** - HTTP download and SFTP download patterns
2. **File Upload** - SFTP upload via Spring Integration Gateway

---

# Part 1: File Download

## 1.1 HTTP Download (RestClient)

### Purpose

Downloads files from REST API endpoints using Spring's `RestClient`.

### How It Works

When you call `.body(byte[].class)`, you don't need to know the file size beforehand:

| Step | What Happens |
|------|--------------|
| 1 | HTTP GET request sent to download URL |
| 2 | Server responds with `Content-Length` header OR `Transfer-Encoding: chunked` |
| 3 | RestClient internally reads and buffers the entire response |
| 4 | Java dynamically allocates the byte array based on received data |
| 5 | Complete byte array returned to your code |

### Implementation

```java
@Service
public class FileDownloadServiceImpl implements FileDownloadService {

    private final RestClient restClient;

    public FileDownloadServiceImpl(RestClient restClient) {
        this.restClient = restClient;
    }

    @Override
    public File downloadFile(String downloadUrl, String targetFolder, String fileName) 
                             throws BaseBusinessException {
        String targetPath = targetFolder + fileName;

        try {
            // Step 1: HTTP GET - RestClient auto-buffers entire response into byte[]
            byte[] fileBytes = restClient.get()
                    .uri(downloadUrl)
                    .retrieve()
                    .body(byte[].class);

            // Step 2: Validate downloaded content
            if (fileBytes == null || fileBytes.length == 0) {
                throw new IOException("Downloaded file is empty for " + fileName);
            }

            // Step 3: Write byte array to local filesystem
            Files.write(Paths.get(targetPath), fileBytes);
            logger.info("Downloaded file to {}", targetPath);
            
            return new File(targetPath);
            
        } catch (RestClientResponseException e) {
            logger.error("Failed to download file from {}", downloadUrl);
            throw new BaseBusinessException(BusinessValidationErrorCodes.SYSTEM_ERROR);
        } catch (IOException e) {
            logger.error("Error writing file to disk", e);
            throw new BaseBusinessException(BusinessValidationErrorCodes.SYSTEM_ERROR);
        }
    }
}
```

### Key Points

- **No size calculation needed** - HTTP protocol and RestClient handle buffering automatically
- **Memory usage** - Entire file loaded into memory as `byte[]`
- **Best for** - Small to medium files (< 100MB)

---

## 1.2 SFTP Download (Spring Integration)

### Purpose

Downloads files from SFTP servers using Spring Integration's `SftpRemoteFileTemplate`.

### Configuration

```java
@Configuration
public class SftpDownloadConfiguration {

    @Value("${batch.sftp.remote.in.host}")
    private String sftpHost;

    @Value("${batch.sftp.remote.in.port:22}")
    private int sftpPort;

    @Value("${batch.sftp.remote.in.username}")
    private String sftpUser;

    @Value("${batch.sftp.remote.in.password}")
    private String sftpPass;

    @Value("${batch.sftp.remote.input.dir}")
    private String inputDirectory;

    @Value("${batch.sftp.local.dir.received}")
    private String receivedFilePath;

    @Value("${batch.job.raw.filename}")
    private String rawFilename;  // e.g., "Report_%s.csv"

    @Bean
    public SessionFactory sftpSession() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
        factory.setHost(sftpHost);
        factory.setPort(sftpPort);
        factory.setUser(sftpUser);
        factory.setPassword(sftpPass);
        factory.setAllowUnknownKeys(true);  // Skip host key verification
        return factory;
    }

    public SftpRemoteFileTemplate template() {
        SftpRemoteFileTemplate sftpTemplate = new SftpRemoteFileTemplate(sftpSession());
        sftpTemplate.setRemoteDirectoryExpression(new LiteralExpression(inputDirectory));
        return sftpTemplate;
    }
}
```

### Implementation

```java
public Map<String, File> downloadFiles() {
    SftpRemoteFileTemplate sftpTemplate = template();
    
    // Step 1: List remote files matching pattern
    SftpClient.DirEntry[] files = sftpTemplate.list(inputDirectory + "*");
    
    Map<String, File> downloadedFiles = new HashMap<>();
    List<String> processedList = getProcessedFileNames();
    String[] prefixAndType = rawFilename.split("%s");
    
    try {
        for (SftpClient.DirEntry fileLs : files) {
            // Step 2: Filter files by name pattern and skip already processed
            if (isFilenameValid(fileLs.getFilename()) && 
                !processedList.contains(fileLs.getFilename())) {
                
                // Step 3: Download using streaming callback (memory efficient)
                sftpTemplate.get(inputDirectory + fileLs.getFilename(), inputStream -> {
                    File file = new File(receivedFilePath + fileLs.getFilename());
                    FileOutputStream outputStream = new FileOutputStream(file);
                    
                    // Step 4: Stream copy - data flows directly to file
                    FileCopyUtils.copy(inputStream, outputStream);
                    
                    // Step 5: Extract identifier from filename
                    String dateString = fileLs.getFilename()
                            .replaceAll(prefixAndType[0], "")
                            .replaceAll(prefixAndType[1], "");
                    
                    if (isDateValid(dateString)) {
                        downloadedFiles.put(dateString, file);
                    }
                });
            }
        }
        
        logger.info("Downloaded [{}] files", downloadedFiles.size());
        
        if (downloadedFiles.isEmpty()) {
            throw new IOException("No files found to process.");
        }
        
        return downloadedFiles;
        
    } catch (Exception e) {
        throw new RuntimeException("SFTP download failed: " + e.getMessage(), e);
    }
}

private boolean isFilenameValid(String filename) {
    String[] prefixAndType = rawFilename.split("%s");
    return filename.startsWith(prefixAndType[0]) && filename.endsWith(prefixAndType[1]);
}

private boolean isDateValid(String date) {
    DateFormat sdf = new SimpleDateFormat("yyyyMMdd");
    sdf.setLenient(false);
    try {
        sdf.parse(date);
        return true;
    } catch (ParseException e) {
        return false;
    }
}

private List<String> getProcessedFileNames() {
    List<String> names = new ArrayList<>();
    File processedDir = new File(processedFilePath);
    
    if (!processedDir.exists()) {
        processedDir.mkdir();
    }
    
    if (processedDir.exists()) {
        Collection<File> files = FileUtils.listFiles(processedDir, 
                                                      TrueFileFilter.TRUE, 
                                                      TrueFileFilter.TRUE);
        for (File file : files) {
            names.add(file.getName());
        }
    }
    
    return names;
}
```

### Key Points

- **Streaming** - Data flows directly from SFTP to local file without loading into memory
- **Memory efficient** - Suitable for large files
- **Batch processing** - Can download multiple files with filtering

---

## 1.3 HTTP vs SFTP Download Comparison

| Aspect | HTTP (RestClient) | SFTP (Spring Integration) |
|--------|-------------------|---------------------------|
| **Protocol** | HTTP/HTTPS | SSH/SFTP |
| **Memory Usage** | Loads entire file into `byte[]` | Streams directly to file |
| **Best For** | Small-medium files (< 100MB) | Large files, batch transfers |
| **Size Handling** | HTTP Content-Length or chunked | SFTP protocol handles internally |
| **Authentication** | Headers (OAuth, API keys) | Username/Password, SSH keys |
| **Connection** | Stateless (per request) | Session-based |

---

# Part 2: File Upload (SFTP via Spring Integration)

## 2.1 Architecture Components

| Component | Purpose |
|-----------|---------|
| `@MessagingGateway` | Interface that abstracts SFTP operations |
| `@Gateway` | Maps method calls to message channels |
| `@ServiceActivator` | Connects channels to SFTP handlers |
| `SftpMessageHandler` | Performs actual SFTP upload |
| `SessionFactory` | Manages SFTP connections |

---

## 2.2 SFTP Session Factory Configuration

```java
@Configuration
@EnableIntegration
public class SftpUploadConfiguration {

    @Value("${batch.sftp.remote.host}")
    private String sftpHost;

    @Value("${batch.sftp.remote.port:22}")
    private int sftpPort;

    @Value("${batch.sftp.remote.username}")
    private String sftpUser;

    @Value("${batch.sftp.remote.password}")
    private String sftpPass;

    @Value("${batch.sftp.remote.output.dir}")
    private String outputDirectory;

    @Bean
    public SessionFactory uploadSftpSession() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
        factory.setHost(sftpHost);
        factory.setPort(sftpPort);
        factory.setUser(sftpUser);
        factory.setPassword(sftpPass);
        factory.setAllowUnknownKeys(true);
        return factory;
    }
}
```

---

## 2.3 Message Handler Configuration

```java
@Bean
@ServiceActivator(inputChannel = "toSftpChannel")
public MessageHandler sftpUploadHandler() {
    SftpMessageHandler handler = new SftpMessageHandler(uploadSftpSession());
    
    // Set remote directory
    handler.setRemoteDirectoryExpression(new LiteralExpression(outputDirectory));
    
    // Configure filename generation
    handler.setFileNameGenerator(new FileNameGenerator() {
        @Override
        public String generateFileName(Message<?> message) {
            if (message.getPayload() instanceof File) {
                return ((File) message.getPayload()).getName();
            } else {
                throw new IllegalArgumentException("File expected as payload.");
            }
        }
    });
    
    return handler;
}
```

### Custom Filename Generation

You can modify the filename during upload:

```java
handler.setFileNameGenerator(new FileNameGenerator() {
    @Override
    public String generateFileName(Message<?> message) {
        if (message.getPayload() instanceof File) {
            // Example: Remove prefix before '-'
            // "20231015-A2231015" -> "A2231015"
            return ((File) message.getPayload()).getName().split("-")[1];
        }
        throw new IllegalArgumentException("File expected as payload.");
    }
});
```

---

## 2.4 Gateway Interface

```java
@MessagingGateway
public interface UploadGateway {
 
    @Gateway(requestChannel = "toSftpChannel")
    void upload(File file);
    
    // Multiple gateways for different destinations
    @Gateway(requestChannel = "toSftpChannel2")
    void uploadToSecondServer(File file);

    @Gateway(requestChannel = "toSftpChannel3")
    void uploadToThirdServer(File file);
}
```

---

## 2.5 Using the Upload Gateway

```java
@Service
public class BatchServiceImpl implements BatchService {

    @Autowired
    private SftpConfiguration.UploadGateway gateway;
    
    @Autowired
    private FileService file;
    
    @Autowired
    private EmailService email;

    @Value("${batch.sftp.local.dir.generated}")
    private String generatedFolder;

    @Override
    public void processFile(String filename, String sentFolder) {
        String generatedFilename = generatedFolder + filename;
        File generatedFile = new File(generatedFilename);
        
        if (generatedFile.exists()) {
            logger.debug("Generated file exists");
            
            // Step 1: Upload via gateway (Spring Integration handles SFTP)
            gateway.upload(generatedFile);
            
            // Step 2: Move to sent folder after successful upload
            file.moveFile(generatedFilename, sentFolder);
            
            // Step 3: Send notification
            email.sendMessage("File " + filename + " has been sent.");
        } else {
            email.sendError("No file generated for " + filename + ".");
        }
    }
}
```

---

## 2.6 Multiple SFTP Destinations

For uploading to different SFTP servers, create separate session factories and handlers:

```java
@Configuration
@EnableIntegration
public class SftpConfiguration {

    // Server 1 Configuration
    @Bean
    public SessionFactory server1SftpSession() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
        factory.setHost(server1Host);
        factory.setPort(server1Port);
        factory.setUser(server1User);
        factory.setPassword(server1Pass);
        factory.setAllowUnknownKeys(true);
        return factory;
    }

    @Bean
    @ServiceActivator(inputChannel = "toServer1Channel")
    public MessageHandler server1Handler() {
        SftpMessageHandler handler = new SftpMessageHandler(server1SftpSession());
        handler.setRemoteDirectoryExpression(new LiteralExpression(server1OutputDir));
        handler.setFileNameGenerator(message -> ((File) message.getPayload()).getName());
        return handler;
    }

    // Server 2 Configuration
    @Bean
    public SessionFactory server2SftpSession() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
        factory.setHost(server2Host);
        factory.setPort(server2Port);
        factory.setUser(server2User);
        factory.setPassword(server2Pass);
        factory.setAllowUnknownKeys(true);
        return factory;
    }

    @Bean
    @ServiceActivator(inputChannel = "toServer2Channel")
    public MessageHandler server2Handler() {
        SftpMessageHandler handler = new SftpMessageHandler(server2SftpSession());
        handler.setRemoteDirectoryExpression(new LiteralExpression(server2OutputDir));
        handler.setFileNameGenerator(message -> ((File) message.getPayload()).getName());
        return handler;
    }

    // Gateway with multiple upload methods
    @MessagingGateway
    public interface UploadGateway {
        @Gateway(requestChannel = "toServer1Channel")
        void uploadToServer1(File file);
        
        @Gateway(requestChannel = "toServer2Channel")
        void uploadToServer2(File file);
    }
}
```

---

# Part 3: Configuration Reference

## 3.1 Properties Example

```properties
# Local Directories
batch.sftp.local.dir.generated=/app/data/generated
batch.sftp.local.dir.received=/app/data/received
batch.sftp.local.dir.processed=/app/data/processed
batch.sftp.local.dir.failed=/app/data/failed
batch.sftp.local.dir.sent=/app/data/sent

# SFTP Download Configuration
batch.sftp.remote.in.host=sftp.download.example.com
batch.sftp.remote.in.port=22
batch.sftp.remote.in.username=download_user
batch.sftp.remote.in.password=download_pass
batch.sftp.remote.input.dir=/inbound/

# SFTP Upload Configuration
batch.sftp.remote.out.host=sftp.upload.example.com
batch.sftp.remote.out.port=22
batch.sftp.remote.out.username=upload_user
batch.sftp.remote.out.password=upload_pass
batch.sftp.remote.output.dir=/outbound/

# File Pattern
batch.job.raw.filename=Report_%s.csv
```

---

## 3.2 Summary Table

| Operation | Technology | Class/Component | Method |
|-----------|------------|-----------------|--------|
| HTTP Download | RestClient | `FileDownloadServiceImpl` | `downloadFile()` |
| SFTP Download | Spring Integration | `SftpRemoteFileTemplate` | `get()` with callback |
| SFTP Upload | Spring Integration | `UploadGateway` | `upload()` |
| File Move | Java NIO / FileService | `FileService` | `moveFile()` |
