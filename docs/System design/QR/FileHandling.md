# File Handling Implementation Guide

## Overview

This document explains file handling in batch applications - **what** Java dependencies are used, **how** they work, and **why** they are chosen.

---

# Part 1: File Download

## 1.1 HTTP Download

### What - Dependencies Used

| Dependency | Class | Package |
|------------|-------|---------|
| `spring-web` | `RestClient` | `org.springframework.web.client` |
| `spring-web` | `RestClientResponseException` | `org.springframework.web.client` |
| Java Standard | `Files` | `java.nio.file` |
| Java Standard | `Paths` | `java.nio.file` |
| Java Standard | `File` | `java.io` |

### Why - When to Use HTTP Download

- The source provides a **REST API endpoint** (URL) to download files
- Authentication is via **HTTP headers** (API keys, OAuth tokens)
- File transfer is **one-time/on-demand** (not scheduled batch polling)
- Files are **small to medium** size (< 100MB) - fits in memory

### How - Implementation

```java
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientResponseException;
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

@Service
public class FileDownloadServiceImpl {

    private final RestClient restClient;

    public File downloadFile(String downloadUrl, String targetFolder, String fileName) {
        String targetPath = targetFolder + fileName;

        try {
            // RestClient sends HTTP GET and buffers entire response
            byte[] fileBytes = restClient.get()
                    .uri(downloadUrl)
                    .retrieve()
                    .body(byte[].class);

            if (fileBytes == null || fileBytes.length == 0) {
                throw new IOException("Downloaded file is empty");
            }

            // Write byte array to filesystem
            Files.write(Paths.get(targetPath), fileBytes);
            return new File(targetPath);

        } catch (RestClientResponseException e) {
            throw new RuntimeException("HTTP download failed", e);
        } catch (IOException e) {
            throw new RuntimeException("File write failed", e);
        }
    }
}
```

### How It Works Internally

| Step | What Happens |
|------|--------------|
| 1 | `RestClient.get().uri()` - Creates HTTP GET request |
| 2 | Server responds with `Content-Length` header or chunked transfer |
| 3 | `RestClient` internally buffers the entire response body |
| 4 | `.body(byte[].class)` - Returns complete byte array |
| 5 | `Files.write()` - Writes byte array to disk atomically |

**Key Point**: You don't need to know file size beforehand - HTTP protocol handles it.

---

## 1.2 SFTP Download

### What - Dependencies Used

| Dependency | Class | Package | Why Needed |
|------------|-------|---------|------------|
| `spring-integration-sftp` | `DefaultSftpSessionFactory` | `org.springframework.integration.sftp.session` | Create SFTP connection |
| `spring-integration-sftp` | `SftpRemoteFileTemplate` | `org.springframework.integration.sftp.session` | SFTP operations (list, get, put) |
| `spring-integration-core` | `SessionFactory` | `org.springframework.integration.file.remote.session` | Session factory interface |
| `spring-integration-core` | `LiteralExpression` | `org.springframework.expression.common` | Set remote directory path |
| `spring-core` | `FileCopyUtils` | `org.springframework.util` | Copy stream to file efficiently |
| `apache-sshd` | `SftpClient` | `org.apache.sshd.sftp.client` | List remote directory entries |
| `commons-io` | `FileUtils` | `org.apache.commons.io` | List local files recursively |
| `commons-io` | `TrueFileFilter` | `org.apache.commons.io.filefilter` | Filter for listing all files |
| Java Standard | `File` | `java.io` | File object |
| Java Standard | `FileOutputStream` | `java.io` | Write bytes to file |

### Why - When to Use SFTP Download

- The source is an **SFTP server** (not a REST API)
- Authentication is via **username/password** or **SSH keys**
- Need to **poll for new files** periodically (batch processing)
- Files can be **large** - need streaming (not loading entire file to memory)
- Need to **list directory** and download multiple files

### How - Configuration

```java
import org.springframework.integration.sftp.session.DefaultSftpSessionFactory;
import org.springframework.integration.sftp.session.SftpRemoteFileTemplate;
import org.springframework.integration.file.remote.session.SessionFactory;
import org.springframework.expression.common.LiteralExpression;

@Configuration
public class SftpDownloadConfiguration {

    @Value("${batch.sftp.remote.host}")
    private String sftpHost;

    @Value("${batch.sftp.remote.port:22}")
    private int sftpPort;

    @Value("${batch.sftp.remote.username}")
    private String sftpUser;

    @Value("${batch.sftp.remote.password}")
    private String sftpPass;

    // Creates SFTP session with connection details
    @Bean
    public SessionFactory sftpSession() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
        factory.setHost(sftpHost);
        factory.setPort(sftpPort);
        factory.setUser(sftpUser);
        factory.setPassword(sftpPass);
        factory.setAllowUnknownKeys(true);
        return factory;
    }

    // Template for SFTP operations
    public SftpRemoteFileTemplate template() {
        SftpRemoteFileTemplate sftpTemplate = new SftpRemoteFileTemplate(sftpSession());
        sftpTemplate.setRemoteDirectoryExpression(new LiteralExpression(inputDirectory));
        return sftpTemplate;
    }
}
```

### How - Download Implementation

```java
import org.apache.sshd.sftp.client.SftpClient;
import org.springframework.util.FileCopyUtils;
import org.apache.commons.io.FileUtils;
import org.apache.commons.io.filefilter.TrueFileFilter;
import java.io.File;
import java.io.FileOutputStream;

public Map<String, File> downloadFiles() {
    SftpRemoteFileTemplate sftpTemplate = template();
    
    // List files on remote SFTP server
    SftpClient.DirEntry[] files = sftpTemplate.list(inputDirectory + "*");
    
    Map<String, File> downloadedFiles = new HashMap<>();
    List<String> processedList = getProcessedFileNames();

    for (SftpClient.DirEntry fileLs : files) {
        if (isFilenameValid(fileLs.getFilename()) && 
            !processedList.contains(fileLs.getFilename())) {
            
            // Streaming download - memory efficient
            sftpTemplate.get(inputDirectory + fileLs.getFilename(), inputStream -> {
                File file = new File(receivedFilePath + fileLs.getFilename());
                FileOutputStream outputStream = new FileOutputStream(file);
                
                // Stream directly to file (not loaded to memory)
                FileCopyUtils.copy(inputStream, outputStream);
                
                downloadedFiles.put(extractDate(fileLs.getFilename()), file);
            });
        }
    }
    
    return downloadedFiles;
}

// Uses commons-io to list already processed files
private List<String> getProcessedFileNames() {
    File processedDir = new File(processedFilePath);
    Collection<File> files = FileUtils.listFiles(processedDir, 
                                                  TrueFileFilter.TRUE, 
                                                  TrueFileFilter.TRUE);
    return files.stream().map(File::getName).collect(Collectors.toList());
}
```

---

## 1.3 HTTP vs SFTP Download - Comparison

| Aspect | HTTP (`RestClient`) | SFTP (`SftpRemoteFileTemplate`) |
|--------|---------------------|----------------------------------|
| **Protocol** | HTTP/HTTPS | SSH/SFTP |
| **Memory** | Loads entire file into `byte[]` | Streams directly to file |
| **File Size** | Best for < 100MB | Handles any size |
| **List Files** | Not supported | `sftpTemplate.list()` |
| **Authentication** | HTTP headers (API keys) | Username/password, SSH keys |
| **Use Case** | REST API download | Server-to-server batch transfer |

---

# Part 2: File Upload (SFTP)

### What - Dependencies Used

| Dependency | Class | Package | Why Needed |
|------------|-------|---------|------------|
| `spring-integration-sftp` | `SftpMessageHandler` | `org.springframework.integration.sftp.outbound` | Handles SFTP upload |
| `spring-integration-sftp` | `DefaultSftpSessionFactory` | `org.springframework.integration.sftp.session` | Create SFTP connection |
| `spring-integration-core` | `@MessagingGateway` | `org.springframework.integration.annotation` | Declare upload interface |
| `spring-integration-core` | `@Gateway` | `org.springframework.integration.annotation` | Map method to channel |
| `spring-integration-core` | `@ServiceActivator` | `org.springframework.integration.annotation` | Connect channel to handler |
| `spring-integration-core` | `@EnableIntegration` | `org.springframework.integration.config` | Enable Spring Integration |
| `spring-integration-core` | `FileNameGenerator` | `org.springframework.integration.file` | Custom remote filename |
| `spring-integration-core` | `SessionFactory` | `org.springframework.integration.file.remote.session` | Session factory interface |
| `spring-messaging` | `Message` | `org.springframework.messaging` | Message abstraction |
| `spring-messaging` | `MessageHandler` | `org.springframework.messaging` | Handler interface |
| `spring-core` | `LiteralExpression` | `org.springframework.expression.common` | Set remote directory |
| Java Standard | `File` | `java.io` | File object |

### Why - When to Use SFTP Upload

- The destination is an **SFTP server** (partner system, external server)
- This is a **batch application** (not a web app with browser uploads)
- Need **server-to-server** file transfer (automated, scheduled)
- The receiving system only accepts **SFTP** (common in financial industry)
- Need to upload to **multiple different SFTP servers**

### How - Session Factory Configuration

```java
import org.springframework.integration.sftp.session.DefaultSftpSessionFactory;
import org.springframework.integration.file.remote.session.SessionFactory;

@Bean
public SessionFactory uploadSftpSession() {
    DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
    factory.setHost(sftpHost);        // SFTP server hostname
    factory.setPort(sftpPort);        // Usually 22
    factory.setUser(sftpUser);        // SFTP username
    factory.setPassword(sftpPass);    // SFTP password
    factory.setAllowUnknownKeys(true); // Skip host key verification
    return factory;
}
```

### How - Message Handler Configuration

```java
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.sftp.outbound.SftpMessageHandler;
import org.springframework.integration.file.FileNameGenerator;
import org.springframework.expression.common.LiteralExpression;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHandler;
import java.io.File;

@Bean
@ServiceActivator(inputChannel = "toSftpChannel")
public MessageHandler sftpUploadHandler() {
    // Create handler with SFTP session
    SftpMessageHandler handler = new SftpMessageHandler(uploadSftpSession());
    
    // Set remote directory where files will be uploaded
    handler.setRemoteDirectoryExpression(new LiteralExpression(outputDirectory));
    
    // Define how to name the file on remote server
    handler.setFileNameGenerator(new FileNameGenerator() {
        @Override
        public String generateFileName(Message<?> message) {
            if (message.getPayload() instanceof File) {
                return ((File) message.getPayload()).getName();
            }
            throw new IllegalArgumentException("File expected");
        }
    });
    
    return handler;
}
```

### How - Gateway Interface

```java
import org.springframework.integration.annotation.Gateway;
import org.springframework.integration.annotation.MessagingGateway;
import java.io.File;

@MessagingGateway
public interface UploadGateway {
    
    @Gateway(requestChannel = "toSftpChannel")
    void upload(File file);
    
    // Multiple methods for different SFTP destinations
    @Gateway(requestChannel = "toServer2Channel")
    void uploadToServer2(File file);
}
```

### How - Using the Gateway

```java
@Service
public class BatchServiceImpl {

    @Autowired
    private UploadGateway gateway;

    public void processFile(String filename) {
        File generatedFile = new File(generatedFolder + filename);
        
        if (generatedFile.exists()) {
            // Single line upload - Spring Integration handles SFTP
            gateway.upload(generatedFile);
            
            // Move to sent folder after upload
            moveFile(generatedFile, sentFolder);
        }
    }
}
```

---

# Part 3: File Generation (Writing Files)

### What - Dependencies Used

| Dependency | Class | Package | Why Needed |
|------------|-------|---------|------------|
| Java Standard | `BufferedWriter` | `java.io` | Efficient buffered writing |
| Java Standard | `FileWriter` | `java.io` | Write characters to file |
| Java Standard | `File` | `java.io` | File representation |
| Java Standard | `FileInputStream` | `java.io` | Read file for checksum |
| Java Standard | `Files` | `java.nio.file` | Delete file on error |
| Java Standard | `CRC32` | `java.util.zip` | Calculate CRC32 checksum |
| Java Standard | `MessageDigest` | `java.security` | Calculate SHA-256 hash |
| Jakarta EE | `DatatypeConverter` | `jakarta.xml.bind` | Convert bytes to hex string |

### Why - These Dependencies

| Class | Why Used |
|-------|----------|
| `BufferedWriter` | Wraps `FileWriter` for **efficient writing** (buffers data, reduces I/O calls) |
| `FileWriter` | **Writes characters** to file (converts String to bytes using default charset) |
| `File.createNewFile()` | **Creates empty file** - fails if file exists (prevents accidental overwrite) |
| `Files.deleteIfExists()` | **Cleanup on error** - delete partial file if writing fails |
| `CRC32` | **Checksum validation** - receiver can verify file integrity |
| `MessageDigest` | **Hash for security** - SHA-256 for file fingerprint |
| `DatatypeConverter` | **Hex encoding** - convert hash bytes to readable hex string |

### How - File Writing Pattern

```java
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;

protected File writeFile(List<Record> data, String filename) throws IOException {
    File file = new File(generatedFilePath, filename);
    BufferedWriter writer = null;
    
    try {
        // Create new file (fails if exists)
        file.createNewFile();
        
        // BufferedWriter wraps FileWriter for efficiency
        writer = new BufferedWriter(new FileWriter(file));
        
        // Write header
        writer.write(Record.printHeader());
        writer.write("\r\n");
        
        // Write data rows
        for (Record record : data) {
            writer.write(record.toCsv());
            writer.write("\r\n");
        }
        
        // Write trailer
        writer.write(Record.printTrailer(data.size()));
        writer.write("\r\n");
        
        // Flush buffer to disk
        writer.flush();
        
    } catch (Exception e) {
        // Cleanup: delete partial file on error
        Files.deleteIfExists(file.toPath());
        throw e;
    } finally {
        // Always close writer
        if (writer != null) {
            writer.close();
        }
    }
    
    return file;
}
```

### How - CRC32 Checksum

```java
import java.util.zip.CRC32;
import java.io.FileInputStream;

public String calculateCRC32(File file) {
    CRC32 crc = new CRC32();
    
    try (FileInputStream fis = new FileInputStream(file)) {
        byte[] buffer = new byte[8192];
        int bytesRead;
        
        // Read file in chunks, update CRC
        while ((bytesRead = fis.read(buffer)) != -1) {
            crc.update(buffer, 0, bytesRead);
        }
    }
    
    // Format as 8-character hex string
    return String.format("%08X", crc.getValue());
}
```

### How - SHA-256 Hash

```java
import java.security.MessageDigest;
import java.io.FileInputStream;
import java.io.InputStream;
import jakarta.xml.bind.DatatypeConverter;

public String calculateSHA256(File file) {
    try (InputStream in = new FileInputStream(file)) {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        
        byte[] block = new byte[4096];
        int length;
        
        // Read file in chunks, update digest
        while ((length = in.read(block)) > 0) {
            digest.update(block, 0, length);
        }
        
        // Convert to hex string
        return DatatypeConverter.printHexBinary(digest.digest());
    } catch (Exception e) {
        return null;
    }
}
```

---

# Part 4: File Reading (CSV Parsing)

### What - Dependencies Used

| Dependency | Class | Package | Why Needed |
|------------|-------|---------|------------|
| Java Standard | `BufferedReader` | `java.io` | Efficient buffered reading |
| Java Standard | `FileReader` | `java.io` | Read characters from file |
| Java Standard | `File` | `java.io` | File representation |
| Java Standard | `Collectors` | `java.util.stream` | Collect stream to list |

### Why - These Dependencies

| Class | Why Used |
|-------|----------|
| `BufferedReader` | Wraps `FileReader` for **efficient reading** (buffers data, reduces I/O calls) |
| `FileReader` | **Reads characters** from file (converts bytes to String using default charset) |
| `br.lines()` | Returns **Stream** of lines - enables functional processing |
| `Collectors.toList()` | Collects stream into **List** for indexed access |

### How - CSV File Reading

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.File;
import java.util.List;
import java.util.stream.Collectors;

protected List<Record> readFile(File downloadedFile) {
    List<Record> records = new ArrayList<>();
    
    try (BufferedReader br = new BufferedReader(new FileReader(downloadedFile))) {
        // Read all lines into list
        List<String> allLines = br.lines().collect(Collectors.toList());
        
        // Skip if only header and trailer
        if (allLines.size() <= 2) return records;

        // Process data lines (skip header at 0, trailer at end)
        for (int i = 1; i < allLines.size() - 1; i++) {
            String line = allLines.get(i);
            String[] parts = line.split(",");
            
            records.add(new Record(
                parts[0].trim(),
                parts[1].trim(),
                parts[2].trim()
            ));
        }
    } catch (IOException e) {
        logger.error("Error reading file", e);
    }

    return records;
}
```

---

# Part 5: Summary

## Dependencies by Operation

| Operation | Key Dependency | Main Classes |
|-----------|----------------|--------------|
| HTTP Download | `spring-web` | `RestClient` |
| SFTP Download | `spring-integration-sftp` | `SftpRemoteFileTemplate`, `DefaultSftpSessionFactory` |
| SFTP Upload | `spring-integration-sftp` | `SftpMessageHandler`, `@MessagingGateway`, `@Gateway` |
| File Writing | Java Standard | `BufferedWriter`, `FileWriter` |
| File Reading | Java Standard | `BufferedReader`, `FileReader` |
| Checksum | Java Standard | `CRC32`, `MessageDigest` |
| File Utilities | `commons-io` | `FileUtils`, `FileCopyUtils` |

## When to Use What

| Scenario | Use |
|----------|-----|
| Download from REST API | `RestClient` (HTTP) |
| Download from partner SFTP server | `SftpRemoteFileTemplate` (SFTP) |
| Upload to partner SFTP server | `SftpMessageHandler` + `@MessagingGateway` |
| Generate CSV/report file | `BufferedWriter` + `FileWriter` |
| Parse downloaded CSV file | `BufferedReader` + `FileReader` |
| Verify file integrity | `CRC32` or `MessageDigest` (SHA-256) |
