# File Handling Implementation Guide

## Overview

This document explains file handling in batch applications — **what** Java dependencies are used, **how** they work, and **why** they are chosen.

---

# Part 1: File Download

File download has two methods based on source protocol:
- **HTTP Method** - Download from REST API endpoints
- **SFTP Method** - Download from SFTP servers

---

## Method 1: HTTP Download

### 1. RestClient

#### Purpose

HTTP client for making REST API calls and downloading files.

#### Dependencies

```java
org.springframework.web.client.RestClient
org.springframework.web.client.RestClientResponseException
```

#### Actual Usage

```java
byte[] fileBytes = restClient.get()
        .uri(downloadUrl)
        .retrieve()
        .body(byte[].class);
```

#### Responsibilities

- Creates HTTP GET request to the specified URL
- Handles HTTP response status codes
- Buffers entire response body into memory
- Returns response as `byte[]` for file content
- Throws `RestClientResponseException` on HTTP errors

---

### 2. Files (java.nio.file)

#### Purpose

Utility class for writing byte array to filesystem.

#### Dependencies

```java
java.nio.file.Files
java.nio.file.Paths
```

#### Actual Usage

```java
Files.write(Paths.get(targetPath), fileBytes);
```

#### Responsibilities

- Writes byte array atomically to disk
- Creates parent directories if needed
- Handles file system operations

---

### 3. File (java.io)

#### Purpose

Represents the downloaded file on local filesystem.

#### Dependencies

```java
java.io.File
```

#### Actual Usage

```java
return new File(targetPath);
```

#### Responsibilities

- Provides file path abstraction
- Used for returning file reference to caller

---

### Execution Flow

1. **Service** calls `restClient.get().uri(downloadUrl)` to create HTTP GET request
2. **RestClient** sends request to remote HTTP server
3. **Remote Server** responds with file content (Content-Length header or chunked transfer)
4. **RestClient** buffers entire response into `byte[]` array
5. **Files.write()** writes byte array atomically to disk
6. **File** object is returned to caller

---

### Complete Implementation

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
            byte[] fileBytes = restClient.get()
                    .uri(downloadUrl)
                    .retrieve()
                    .body(byte[].class);

            if (fileBytes == null || fileBytes.length == 0) {
                throw new IOException("Downloaded file is empty");
            }

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

---

## Method 2: SFTP Download

### 1. DefaultSftpSessionFactory

#### Purpose

Creates and manages SFTP connections with authentication.

#### Dependencies

```java
org.springframework.integration.sftp.session.DefaultSftpSessionFactory
org.springframework.integration.file.remote.session.SessionFactory
```

#### Actual Usage

```java
DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
factory.setHost(sftpHost);
factory.setPort(sftpPort);
factory.setUser(sftpUser);
factory.setPassword(sftpPass);
factory.setAllowUnknownKeys(true);
```

#### Responsibilities

- Stores SFTP connection details (host, port, username, password)
- Creates SSH session to SFTP server
- Manages connection pooling
- Handles host key verification (`setAllowUnknownKeys`)

#### Configuration Source

```properties
batch.sftp.remote.host=sftp.partner.com
batch.sftp.remote.port=22
batch.sftp.remote.username=batchuser
batch.sftp.remote.password=${SFTP_PASSWORD}
```

---

### 2. SftpRemoteFileTemplate

#### Purpose

Template for performing SFTP operations (list, get, put).

#### Dependencies

```java
org.springframework.integration.sftp.session.SftpRemoteFileTemplate
org.springframework.expression.common.LiteralExpression
```

#### Actual Usage

```java
SftpRemoteFileTemplate sftpTemplate = new SftpRemoteFileTemplate(sftpSession());
sftpTemplate.setRemoteDirectoryExpression(new LiteralExpression(inputDirectory));
```

List files:

```java
SftpClient.DirEntry[] files = sftpTemplate.list(inputDirectory + "*");
```

Download file:

```java
sftpTemplate.get(inputDirectory + filename, inputStream -> {
    // Process input stream
});
```

#### Responsibilities

- Integrates **DefaultSftpSessionFactory** (gets SFTP connection)
- Sets remote directory path via **LiteralExpression**
- Lists files on remote SFTP server
- Downloads files as streaming `InputStream`
- Provides callback-based file processing

---

### 3. SftpClient.DirEntry

#### Purpose

Represents a file entry in remote SFTP directory listing.

#### Dependencies

```java
org.apache.sshd.sftp.client.SftpClient
```

#### Actual Usage

```java
SftpClient.DirEntry[] files = sftpTemplate.list(inputDirectory + "*");
for (SftpClient.DirEntry fileLs : files) {
    String filename = fileLs.getFilename();
}
```

#### Responsibilities

- Provides filename of remote file
- Provides file metadata (size, permissions, timestamps)
- Used for iterating through directory contents

---

### 4. FileCopyUtils

#### Purpose

Efficiently copies input stream to output stream.

#### Dependencies

```java
org.springframework.util.FileCopyUtils
```

#### Actual Usage

```java
sftpTemplate.get(inputDirectory + filename, inputStream -> {
    FileOutputStream outputStream = new FileOutputStream(file);
    FileCopyUtils.copy(inputStream, outputStream);
});
```

#### Responsibilities

- Streams data directly from input to output (memory efficient)
- Handles buffer management internally
- Closes streams after copy completes

---

### 5. FileUtils (commons-io)

#### Purpose

Utility for listing local files recursively.

#### Dependencies

```java
org.apache.commons.io.FileUtils
org.apache.commons.io.filefilter.TrueFileFilter
```

#### Actual Usage

```java
Collection<File> files = FileUtils.listFiles(
    processedDir, 
    TrueFileFilter.TRUE,
    TrueFileFilter.TRUE
);
```

#### Responsibilities

- Lists files in directory recursively
- Filters files by criteria
- Used to track already-processed files

---

### Execution Flow

1. **Service** creates `SftpRemoteFileTemplate` with `DefaultSftpSessionFactory`
2. **SftpRemoteFileTemplate** sets remote directory via `LiteralExpression`
3. **sftpTemplate.list()** connects to SFTP server and lists files
4. **SftpClient.DirEntry[]** returns list of remote files with metadata
5. **FileUtils.listFiles()** gets list of already-processed local files
6. For each unprocessed file:
   - **sftpTemplate.get()** opens streaming connection to remote file
   - **FileCopyUtils.copy()** streams content to **FileOutputStream**
   - File is saved to local filesystem

---

### Complete Implementation

```java
import org.apache.sshd.sftp.client.SftpClient;
import org.springframework.util.FileCopyUtils;
import org.apache.commons.io.FileUtils;
import org.apache.commons.io.filefilter.TrueFileFilter;
import org.springframework.integration.sftp.session.DefaultSftpSessionFactory;
import org.springframework.integration.sftp.session.SftpRemoteFileTemplate;
import org.springframework.integration.file.remote.session.SessionFactory;
import org.springframework.expression.common.LiteralExpression;
import java.io.File;
import java.io.FileOutputStream;

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

    @Value("${batch.sftp.remote.input.dir}")
    private String inputDirectory;

    @Value("${batch.sftp.local.dir.received}")
    private String receivedFilePath;

    @Value("${batch.sftp.local.dir.processed}")
    private String processedFilePath;

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

    public SftpRemoteFileTemplate template() {
        SftpRemoteFileTemplate sftpTemplate = new SftpRemoteFileTemplate(sftpSession());
        sftpTemplate.setRemoteDirectoryExpression(new LiteralExpression(inputDirectory));
        return sftpTemplate;
    }

    public Map<String, File> downloadFiles() {
        SftpRemoteFileTemplate sftpTemplate = template();
        SftpClient.DirEntry[] files = sftpTemplate.list(inputDirectory + "*");

        Map<String, File> downloadedFiles = new HashMap<>();
        List<String> processedList = getProcessedFileNames();

        for (SftpClient.DirEntry fileLs : files) {
            if (isFilenameValid(fileLs.getFilename()) &&
                !processedList.contains(fileLs.getFilename())) {

                sftpTemplate.get(inputDirectory + fileLs.getFilename(), inputStream -> {
                    File file = new File(receivedFilePath + fileLs.getFilename());
                    FileOutputStream outputStream = new FileOutputStream(file);
                    FileCopyUtils.copy(inputStream, outputStream);
                    downloadedFiles.put(extractDate(fileLs.getFilename()), file);
                });
            }
        }
        return downloadedFiles;
    }

    private List<String> getProcessedFileNames() {
        File processedDir = new File(processedFilePath);
        if (!processedDir.exists()) {
            processedDir.mkdir();
        }
        Collection<File> files = FileUtils.listFiles(processedDir, TrueFileFilter.TRUE, TrueFileFilter.TRUE);
        return files.stream().map(File::getName).collect(Collectors.toList());
    }
}
```

---

## HTTP vs SFTP Download - Comparison

| Aspect | HTTP (`RestClient`) | SFTP (`SftpRemoteFileTemplate`) |
|--------|---------------------|----------------------------------|
| **Protocol** | HTTP/HTTPS | SSH/SFTP |
| **Memory** | Loads entire file into `byte[]` | Streams directly to file |
| **File Size** | Best for < 100MB | Handles any size |
| **List Files** | Not supported | `sftpTemplate.list()` |
| **Authentication** | HTTP headers (API keys) | Username/password, SSH keys |
| **Use Case** | REST API download | Server-to-server batch transfer |

---

# Part 2: File Upload

File upload uses SFTP protocol via Spring Integration.

---

## Method: SFTP Upload

### 1. @MessagingGateway / UploadGateway

#### Purpose

Declares a clean Java interface for SFTP upload operations.

#### Dependencies

```java
org.springframework.integration.annotation.MessagingGateway
org.springframework.integration.annotation.Gateway
```

#### Actual Usage

```java
@MessagingGateway
public interface UploadGateway {

    @Gateway(requestChannel = "toSftpChannel")
    void upload(File file);

    @Gateway(requestChannel = "toPartnerSftpChannel")
    void uploadToPartner(File file);
}
```

Calling the gateway:

```java
gateway.upload(generatedFile);
```

#### Responsibilities

- Provides simple Java interface for file uploads
- Converts method call to Spring Integration **Message**
- Routes message to specified channel (`requestChannel`)
- Abstracts SFTP complexity from caller

---

### 2. @ServiceActivator / SftpMessageHandler

#### Purpose

Handles SFTP upload when message arrives on the channel.

#### Dependencies

```java
org.springframework.integration.annotation.ServiceActivator
org.springframework.integration.sftp.outbound.SftpMessageHandler
org.springframework.integration.file.FileNameGenerator
org.springframework.messaging.MessageHandler
```

#### Actual Usage

```java
@Bean
@ServiceActivator(inputChannel = "toSftpChannel")
public MessageHandler handlerUpload() {
    SftpMessageHandler handler = new SftpMessageHandler(sftpSession());
    handler.setRemoteDirectoryExpression(new LiteralExpression(outputDirectory));
    handler.setFileNameGenerator(new FileNameGenerator() {
        @Override
        public String generateFileName(Message<?> message) {
            if (message.getPayload() instanceof File) {
                return ((File) message.getPayload()).getName();
            }
            throw new IllegalArgumentException("File expected as payload.");
        }
    });
    return handler;
}
```

#### Responsibilities

- Integrates **DefaultSftpSessionFactory** (SFTP connection)
- Listens on specified input channel for messages
- Sets remote directory via **LiteralExpression**
- Generates remote filename via **FileNameGenerator**
- Uploads file content to SFTP server

---

### 3. DefaultSftpSessionFactory

#### Purpose

Creates and manages SFTP connections for upload.

#### Dependencies

```java
org.springframework.integration.sftp.session.DefaultSftpSessionFactory
org.springframework.integration.file.remote.session.SessionFactory
```

#### Actual Usage

```java
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
```

#### Responsibilities

- Stores SFTP connection credentials
- Creates SSH/SFTP session to remote server
- Used by **SftpMessageHandler** for upload operations

---

### 4. FileNameGenerator

#### Purpose

Determines the filename to use on the remote SFTP server.

#### Dependencies

```java
org.springframework.integration.file.FileNameGenerator
org.springframework.messaging.Message
```

#### Actual Usage

```java
handler.setFileNameGenerator(new FileNameGenerator() {
    @Override
    public String generateFileName(Message<?> message) {
        if (message.getPayload() instanceof File) {
            return ((File) message.getPayload()).getName();
        }
        throw new IllegalArgumentException("File expected as payload.");
    }
});
```

#### Responsibilities

- Extracts filename from message payload
- Can transform filename (e.g., add timestamp, remove prefix)
- Returns final filename for remote server

---

### 5. LiteralExpression

#### Purpose

Sets static remote directory path for SFTP upload.

#### Dependencies

```java
org.springframework.expression.common.LiteralExpression
```

#### Actual Usage

```java
handler.setRemoteDirectoryExpression(new LiteralExpression(outputDirectory));
```

#### Responsibilities

- Wraps static string as SpEL expression
- Used to set remote directory path
- Can be replaced with dynamic expression if needed

---

### Execution Flow

1. **Service** calls `gateway.upload(file)` with local file
2. **@MessagingGateway** converts File to `Message<File>` and sends to channel
3. **@ServiceActivator** receives message on `toSftpChannel`
4. **SftpMessageHandler** processes the message:
   - Gets SFTP session from **DefaultSftpSessionFactory**
   - Gets remote directory from **LiteralExpression**
   - Gets remote filename from **FileNameGenerator**
5. **SftpMessageHandler** uploads file content to remote SFTP server

---

### Complete Implementation

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.expression.common.LiteralExpression;
import org.springframework.integration.annotation.Gateway;
import org.springframework.integration.annotation.MessagingGateway;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.integration.file.FileNameGenerator;
import org.springframework.integration.file.remote.session.SessionFactory;
import org.springframework.integration.sftp.outbound.SftpMessageHandler;
import org.springframework.integration.sftp.session.DefaultSftpSessionFactory;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHandler;
import java.io.File;

@Configuration
@EnableIntegration
public class SftpConfiguration {

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
    public SessionFactory sftpSession() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory();
        factory.setHost(sftpHost);
        factory.setPort(sftpPort);
        factory.setUser(sftpUser);
        factory.setPassword(sftpPass);
        factory.setAllowUnknownKeys(true);
        return factory;
    }

    @Bean
    @ServiceActivator(inputChannel = "toSftpChannel")
    public MessageHandler handlerUpload() {
        SftpMessageHandler handler = new SftpMessageHandler(sftpSession());
        handler.setRemoteDirectoryExpression(new LiteralExpression(outputDirectory));
        handler.setFileNameGenerator(new FileNameGenerator() {
            @Override
            public String generateFileName(Message<?> message) {
                if (message.getPayload() instanceof File) {
                    return ((File) message.getPayload()).getName();
                }
                throw new IllegalArgumentException("File expected as payload.");
            }
        });
        return handler;
    }

    @MessagingGateway
    public interface UploadGateway {

        @Gateway(requestChannel = "toSftpChannel")
        void upload(File file);
    }
}
```

---

# Part 3: File Generation (Writing)

File generation writes data to local filesystem as CSV/report files.

---

## Method: Buffered File Writing

### 1. BufferedWriter

#### Purpose

Provides efficient buffered writing to file.

#### Dependencies

```java
java.io.BufferedWriter
java.io.FileWriter
```

#### Actual Usage

```java
BufferedWriter writer = new BufferedWriter(new FileWriter(file));
writer.write(header);
writer.write("\r\n");
writer.flush();
writer.close();
```

#### Responsibilities

- Wraps **FileWriter** for efficient I/O (reduces system calls)
- Buffers data in memory before writing to disk
- `flush()` forces buffer to disk
- `close()` releases file handle

---

### 2. FileWriter

#### Purpose

Writes character data to file.

#### Dependencies

```java
java.io.FileWriter
```

#### Actual Usage

```java
writer = new BufferedWriter(new FileWriter(plain));
```

#### Responsibilities

- Converts String characters to bytes
- Uses default charset for encoding
- Wrapped by **BufferedWriter** for efficiency

---

### 3. File.createNewFile()

#### Purpose

Creates empty file on filesystem.

#### Dependencies

```java
java.io.File
```

#### Actual Usage

```java
File plain = new File(generatedFilePath, filename);
if (!plain.createNewFile()) {
    logger.error("Error during create new file");
}
```

#### Responsibilities

- Creates new empty file
- Returns `false` if file already exists (prevents overwrite)
- Creates file path directories if parent exists

---

### 4. Files.deleteIfExists()

#### Purpose

Cleanup partial file on error.

#### Dependencies

```java
java.nio.file.Files
```

#### Actual Usage

```java
catch (Exception e) {
    Files.deleteIfExists(file.toPath());
    throw e;
}
```

#### Responsibilities

- Deletes file if it exists
- Returns `true` if deleted, `false` if not exists
- Used for error cleanup to prevent corrupt files

---

### 5. CRC32

#### Purpose

Calculates CRC32 checksum for file integrity verification.

#### Dependencies

```java
java.util.zip.CRC32
java.io.FileInputStream
```

#### Actual Usage

```java
CRC32 crc = new CRC32();
try (FileInputStream contents = new FileInputStream(file)) {
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = contents.read(buffer)) != -1) {
        crc.update(buffer, 0, bytesRead);
    }
}
String checksum = String.format("%08X", crc.getValue());
```

#### Responsibilities

- Reads file in chunks (memory efficient)
- Updates CRC with each chunk
- Returns checksum as 8-character hex string
- Used for quick integrity verification

---

### 6. MessageDigest (SHA-256)

#### Purpose

Calculates cryptographic hash for file fingerprint.

#### Dependencies

```java
java.security.MessageDigest
java.io.FileInputStream
jakarta.xml.bind.DatatypeConverter
```

#### Actual Usage

```java
MessageDigest digest = MessageDigest.getInstance("SHA-256");
try (InputStream in = new FileInputStream(file)) {
    byte[] block = new byte[4096];
    int length;
    while ((length = in.read(block)) > 0) {
        digest.update(block, 0, length);
    }
}
return DatatypeConverter.printHexBinary(digest.digest());
```

#### Responsibilities

- Reads file in chunks (memory efficient)
- Computes cryptographic hash (SHA-256)
- **DatatypeConverter** converts bytes to hex string
- Used for secure file fingerprinting

---

### Execution Flow

1. **Service** creates `File` object with target path
2. **File.createNewFile()** creates empty file (fails if exists)
3. **BufferedWriter** wraps **FileWriter** for efficient writing
4. Write file content:
   - Header row
   - Data rows (loop through records)
   - Trailer row
5. **writer.flush()** forces buffer to disk
6. Calculate checksum:
   - **CRC32** for quick integrity check, OR
   - **MessageDigest** for cryptographic hash
7. Append checksum to file (if required)
8. On error: **Files.deleteIfExists()** cleans up partial file
9. **writer.close()** releases file handle

---

### Complete Implementation

```java
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.util.zip.CRC32;

protected File writeFile(List<Record> data, String filename) throws IOException {
    File file = new File(generatedFilePath, filename);
    BufferedWriter writer = null;

    try {
        if (!file.createNewFile()) {
            logger.error("Error during create new file");
        }

        writer = new BufferedWriter(new FileWriter(file));

        writer.write(Record.printHeader());
        writer.write("\r\n");

        for (Record record : data) {
            writer.write(record.toCsv());
            writer.write("\r\n");
        }

        writer.write(Record.printTrailer(data.size()));
        writer.write("\r\n");

        writer.flush();

        CRC32 crc = new CRC32();
        try (FileInputStream contents = new FileInputStream(file)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = contents.read(buffer)) != -1) {
                crc.update(buffer, 0, bytesRead);
            }
        }
        String checksum = String.format("%08X", crc.getValue());

        writer.write(Record.printChecksum(checksum));
        writer.write("\r\n");
        writer.flush();

    } catch (Exception e) {
        Files.deleteIfExists(file.toPath());
        throw e;
    } finally {
        if (writer != null) {
            writer.close();
        }
    }

    return file;
}
```

---

# Part 4: File Reading (Parsing)

File reading parses CSV files from local filesystem.

---

## Method: Buffered File Reading

### 1. BufferedReader

#### Purpose

Provides efficient buffered reading from file.

#### Dependencies

```java
java.io.BufferedReader
java.io.FileReader
```

#### Actual Usage

```java
try (BufferedReader br = new BufferedReader(new FileReader(downloadedFile))) {
    List<String> allLines = br.lines().collect(Collectors.toList());
}
```

#### Responsibilities

- Wraps **FileReader** for efficient I/O (reduces system calls)
- Buffers data in memory for faster reading
- `lines()` returns Stream of lines
- Auto-closed by try-with-resources

---

### 2. FileReader

#### Purpose

Reads character data from file.

#### Dependencies

```java
java.io.FileReader
```

#### Actual Usage

```java
new BufferedReader(new FileReader(downloadedFile))
```

#### Responsibilities

- Converts file bytes to characters
- Uses default charset for decoding
- Wrapped by **BufferedReader** for efficiency

---

### 3. Stream.collect() / Collectors.toList()

#### Purpose

Collects stream of lines into List for indexed access.

#### Dependencies

```java
java.util.stream.Stream
java.util.stream.Collectors
```

#### Actual Usage

```java
List<String> allLines = br.lines().collect(Collectors.toList());
```

#### Responsibilities

- `br.lines()` returns `Stream<String>` of file lines
- `collect(Collectors.toList())` converts stream to `List<String>`
- Enables indexed access (header at 0, trailer at end)

---

### 4. String.split()

#### Purpose

Splits CSV line into column values.

#### Dependencies

```java
java.lang.String
```

#### Actual Usage

```java
String[] parts = line.split(",");
records.add(new Record(
    parts[0].trim(),
    parts[1].trim(),
    parts[2].trim()
));
```

#### Responsibilities

- Splits line by comma delimiter
- Returns array of column values
- `trim()` removes leading/trailing whitespace

---

### Execution Flow

1. **BufferedReader** wraps **FileReader** for efficient reading
2. **br.lines()** returns stream of all lines in file
3. **collect(Collectors.toList())** converts stream to `List<String>`
4. Check if file has data (`size > 2` for header + data + trailer)
5. Loop through data lines (skip index 0 header, skip last trailer):
   - **split(",")** parses CSV columns
   - Create **Record** from column values
6. Return `List<Record>` of parsed data

---

### Complete Implementation

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

protected List<Record> readFile(File downloadedFile) {
    List<Record> records = new ArrayList<>();

    try (BufferedReader br = new BufferedReader(new FileReader(downloadedFile))) {
        List<String> allLines = br.lines().collect(Collectors.toList());

        if (allLines.size() <= 2) return records;

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

| Operation | Method | Key Dependency | Main Classes |
|-----------|--------|----------------|--------------|
| File Download | HTTP | `spring-web` | `RestClient` |
| File Download | SFTP | `spring-integration-sftp` | `SftpRemoteFileTemplate`, `DefaultSftpSessionFactory` |
| File Upload | SFTP | `spring-integration-sftp` | `SftpMessageHandler`, `@MessagingGateway`, `@Gateway` |
| File Writing | Buffered | Java Standard | `BufferedWriter`, `FileWriter` |
| File Reading | Buffered | Java Standard | `BufferedReader`, `FileReader` |
| Checksum | CRC32 | Java Standard | `CRC32` |
| Checksum | SHA-256 | Java Standard | `MessageDigest`, `DatatypeConverter` |
| File Utilities | - | `commons-io` | `FileUtils`, `FileCopyUtils` |

## When to Use What

| Scenario | Method |
|----------|--------|
| Download from REST API | HTTP Download (`RestClient`) |
| Download from partner SFTP server | SFTP Download (`SftpRemoteFileTemplate`) |
| Upload to partner SFTP server | SFTP Upload (`SftpMessageHandler` + `@MessagingGateway`) |
| Generate CSV/report file | Buffered Writing (`BufferedWriter` + `FileWriter`) |
| Parse downloaded CSV file | Buffered Reading (`BufferedReader` + `FileReader`) |
| Verify file integrity (fast) | CRC32 Checksum |
| Verify file integrity (secure) | SHA-256 Hash (`MessageDigest`) |
