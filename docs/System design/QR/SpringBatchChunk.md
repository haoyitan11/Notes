# Spring Batch Chunk Processing Dependencies Handle

## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/dd1088da-e942-4745-bf47-5c57151abc94" />

### 1. Chunk-Oriented Step

#### Purpose

Defines a Step that processes data in chunks using ItemReader, ItemProcessor, and ItemWriter.

#### Dependencies

```java
ItemReader
ItemProcessor
ItemWriter
PlatformTransactionManager
JobRepository
```

#### Actual Usage

```java
@Bean
public Step getRefundExceptionStep() {
    return new StepBuilder("getRefundExceptionStep", jobRepository)
            .<RefundExceptionRecord, RefundExceptionRecord>chunk(chunkSize, transactionManager)
            .reader(transactionsReader(to_be_injected))
            .processor(transactionDataProcessor())
            .writer(transactionsFileWriter(to_be_injected))
            .build();
}
```

#### Responsibilities

- Integrates **ItemReader** (reads items one at a time).
- Integrates **ItemProcessor** (transforms each item).
- Integrates **ItemWriter** (writes chunk of items).
- Integrates **PlatformTransactionManager** (commits per chunk).
- Integrates **JobRepository** (persists StepExecution state).
- Processes data in chunks of configurable size.

#### Configuration Source

```java
@Value("${batch.processing.chunk.size}")
private int chunkSize;

@Bean
public Step getRefundExceptionStep() {
    return new StepBuilder("getRefundExceptionStep", jobRepository)
            .<RefundExceptionRecord, RefundExceptionRecord>chunk(chunkSize, transactionManager)
            .reader(transactionsReader(to_be_injected))
            .processor(transactionDataProcessor())
            .writer(transactionsFileWriter(to_be_injected))
            .build();
}
```

---

### 2. ItemReader

#### Purpose

Reads data items one at a time from a data source for chunk-based processing.

#### Dependencies

```java
Repository (data source)
JobParameters (via @StepScope)
```

#### Actual Usage

```java
public class RefundExceptionReader implements ItemReader<RefundExceptionRecord> {

    @Autowired
    QRBatchRepository repository;
    
    private String inputDate;
    private Iterator<RefundExceptionRecord> transactionIterator;
    
    public RefundExceptionReader(String inputDate) {
        this.inputDate = inputDate;
    }
    
    @PostConstruct
    public void afterConstruct() throws BatchException {
        List<RefundExceptionRecord> exceptionRecs = repository.getOutboundRefundExceptions(startDate, endDate);
        transactionIterator = exceptionRecs.iterator();
    }
    
    @Override
    public RefundExceptionRecord read() throws Exception {
        if (transactionIterator.hasNext()) {
            return transactionIterator.next();
        }
        return null;
    }
}
```

Bean configuration with @StepScope for late binding:

```java
@Bean
@StepScope
ItemReader<RefundExceptionRecord> transactionsReader(@Value("#{jobParameters[inputDateStr]}") String inputDateStr) {
    return new RefundExceptionReader(inputDateStr);
}
```

#### Responsibilities

- Integrates **Repository** (data source for reading).
- Integrates **JobParameters** via @StepScope late binding.
- Returns one item at a time until exhausted (returns null).
- Used by **Step** in chunk-based processing.

---

### 3. ItemProcessor

#### Purpose

Transforms or processes each item read by ItemReader before writing.

#### Dependencies

```java
DataContainer (for accumulating state across items)
```

#### Actual Usage

```java
public class RefundExceptionProcessor implements ItemProcessor<RefundExceptionRecord, RefundExceptionRecord> {

    DataContainer dataContainer;
    
    public RefundExceptionProcessor(DataContainer container) {
        this.dataContainer = container;
    }
    
    @Override
    public RefundExceptionRecord process(RefundExceptionRecord transaction) {
        RefundExceptionRecord data = transaction;
        data.setRecordType("DT");
        data.setConsumerBank("BANK");
        data.setProcessingStatus("FAILED");

        // Business logic for failure reason
        long daysBetween = Math.abs(ChronoUnit.DAYS.between(originalTxnDt, refundDt));
        if (daysBetween > 30) {
            data.setFailureReason("LATE_RFD");
        } else {
            data.setFailureReason("ACQ_INIT");
        }
        
        dataContainer.incrementCount();
        dataContainer.addRefund(new BigDecimal(data.getRefundAmt()));
        return data;
    }
}
```

Bean configuration:

```java
@Bean
@JobScope
DataContainer refundDataContainer() {
    return new DataContainer();
}

@Bean
@StepScope
ItemProcessor<RefundExceptionRecord, RefundExceptionRecord> transactionDataProcessor() {
    return new RefundExceptionProcessor(refundDataContainer());
}
```

#### Responsibilities

- Transforms input item to output item.
- Integrates **DataContainer** for state accumulation.
- Returns null to filter out items.
- Used by **Step** in chunk-based processing between reader and writer.

---

### 4. ItemWriter (FlatFileItemWriter)

#### Purpose

Writes processed items to a destination (file, database, etc.).

#### Dependencies

```java
FileSystemResource
DelimitedLineAggregator
BeanWrapperFieldExtractor
FlatFileHeaderCallback
FlatFileFooterCallback
```

#### Actual Usage

```java
@Bean
@StepScope
public FlatFileItemWriter<RefundExceptionRecord> transactionsFileWriter(
        @Value("#{jobParameters[filename]}") String filename) {
    
    FlatFileItemWriter<RefundExceptionRecord> writer = new FlatFileItemWriter<>();
    
    writer.setResource(new FileSystemResource(generatedFolder + filename));
    writer.setShouldDeleteIfEmpty(true);
    
    writer.setLineAggregator(new DelimitedLineAggregator<RefundExceptionRecord>() {
        {
            setDelimiter(",");
            setFieldExtractor(new BeanWrapperFieldExtractor<RefundExceptionRecord>() {
                {
                    setNames(new String[] { 
                        "recordType", "txnDate", "txnRRN", "refundRRN", "consumerBank",
                        "refundAmt", "failureReason", "processingStatus", "refundDate", 
                        "txnAmount", "currency", "fxRate", "foreignTerminalId", 
                        "foreignMerchantId", "merchantName", "netsReferenceNumber"
                    });
                }
            });
        }
    });
    
    // Header callback
    FileHeaderWriter header = new FileHeaderWriter("HD,Transaction_Date,Original_RRN,...");
    writer.setHeaderCallback(header);
    
    // Footer callback
    FileFooterWriter footer = new FileFooterWriter(refundDataContainer());
    writer.setFooterCallback(footer);
    
    return writer;
}
```

#### Responsibilities

- Writes items to file using **FileSystemResource**.
- Integrates **DelimitedLineAggregator** for CSV formatting.
- Integrates **BeanWrapperFieldExtractor** for field mapping.
- Integrates **FlatFileHeaderCallback** and **FlatFileFooterCallback** for file structure.
- Used by **Step** in chunk-based processing.

---

### 5. FlatFileHeaderCallback

#### Purpose

Writes header record at the beginning of a flat file.

#### Dependencies

```java
Writer
```

#### Actual Usage

```java
public class FileHeaderWriter implements FlatFileHeaderCallback {
    
    protected String headerString;

    public FileHeaderWriter(String header) {
        this.headerString = header;
    }

    @Override
    public void writeHeader(Writer writer) throws IOException {
        writer.write(headerString);        
    }
}
```

#### Responsibilities

- Writes header record before data records.
- Called once when file writing begins.
- Used by **FlatFileItemWriter**.

---

### 6. FlatFileFooterCallback

#### Purpose

Writes footer record at the end of a flat file.

#### Dependencies

```java
Writer
DataContainer (for accumulated totals)
```

#### Actual Usage

```java
public class FileFooterWriter implements FlatFileFooterCallback {

    DataContainer dataContainer;

    public FileFooterWriter(DataContainer container) {
        this.dataContainer = container;
    }
    
    @Override
    public void writeFooter(Writer writer) throws IOException {
        StringBuffer footer = new StringBuffer();
        footer.append("TR,");
        footer.append(dataContainer.getTotalCount()).append(",");
        footer.append(dataContainer.getTotalRefundAmount());
        footer.append("\r\n");
        writer.write(footer.toString());    
    }
}
```

#### Responsibilities

- Writes footer record after all data records.
- Integrates **DataContainer** to access accumulated values from ItemProcessor.
- Called once when file writing ends.
- Used by **FlatFileItemWriter**.

---

### 7. DataContainer

#### Purpose

Holds accumulated state across chunk processing for footer generation.

#### Dependencies

```java
None (POJO)
```

#### Actual Usage

```java
public class DataContainer {
    private int totalCount = 0;
    private BigDecimal totalRefundAmount = BigDecimal.ZERO;
    
    public void incrementCount() {
        totalCount++;
    }
    
    public void addRefund(BigDecimal amount) {
        totalRefundAmount = totalRefundAmount.add(amount);
    }
    
    public int getTotalCount() {
        return totalCount;
    }
    
    public BigDecimal getTotalRefundAmount() {
        return totalRefundAmount;
    }
}
```

Bean configuration with @JobScope:

```java
@Bean
@JobScope
DataContainer refundDataContainer() {
    return new DataContainer();
}
```

#### Responsibilities

- Stores accumulated values across chunk processing.
- Updated by **ItemProcessor** for each processed item.
- Read by **FlatFileFooterCallback** to write totals.
- Scoped to Job via **@JobScope** to maintain state across steps.

---

### 8. @StepScope and @JobScope

#### Purpose

Enables late binding of JobParameters and maintains state across processing.

#### Dependencies

```java
JobParameters
StepExecution
```

#### Actual Usage

@JobScope - Bean created once per job execution:

```java
@Bean
@JobScope
DataContainer refundDataContainer() {
    return new DataContainer();
}
```

@StepScope - Bean created once per step execution with late binding:

```java
@Bean
@StepScope
ItemReader<RefundExceptionRecord> transactionsReader(
        @Value("#{jobParameters[inputDateStr]}") String inputDateStr) {
    return new RefundExceptionReader(inputDateStr);
}

@Bean
@StepScope
public FlatFileItemWriter<RefundExceptionRecord> transactionsFileWriter(
        @Value("#{jobParameters[filename]}") String filename) {
    // writer configuration
}
```

#### Responsibilities

- **@JobScope**: Creates bean instance per job execution, shared across steps.
- **@StepScope**: Creates bean instance per step execution with access to job parameters.
- Enables **#{jobParameters[key]}** expression for late binding.
- Required for dynamic parameter injection at runtime.

---

## Multi-Step Job with Chunk Processing

#### Purpose

Combines chunk-based processing step with tasklet-based upload step.

#### Actual Usage

```java
@Bean(name="getRefundExceptionJob")
public Job getRefundExceptionJob() {
    return new JobBuilder("getRefundExceptionJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .flow(getRefundExceptionStep())
            .next(uploadTransactionsFileStep())
            .end()
            .build();
}

@Bean
public Step getRefundExceptionStep() {
    return new StepBuilder("getRefundExceptionStep", jobRepository)
            .<RefundExceptionRecord, RefundExceptionRecord>chunk(chunkSize, transactionManager)
            .reader(transactionsReader(to_be_injected))
            .processor(transactionDataProcessor())
            .writer(transactionsFileWriter(to_be_injected))
            .build();
}

@Bean
public Step uploadTransactionsFileStep() {
    return new StepBuilder("uploadTransactionsFileStep", jobRepository)
            .tasklet(new UploadFileTasklet(service), transactionManager)
            .build();
}
```

Upload Tasklet:

```java
@Component
public class UploadFileTasklet implements Tasklet {

    BatchService service;

    public UploadFileTasklet(BatchService service) {
        this.service = service;
    }

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        String filename = (String) chunkContext.getStepContext().getJobParameters().get("filename");
        service.processRefundExceptionFile(filename);
        return RepeatStatus.FINISHED;
    }
}
```

#### Responsibilities

- **RunIdIncrementer**: Ensures unique JobInstance for each execution.
- **flow().next()**: Chains multiple steps in sequence.
- Step 1: Chunk-based processing (read, process, write).
- Step 2: Tasklet-based upload of generated file.

---

## Execution Flow

1. **Scheduler** builds **JobParameters** with filename parameter:

   ```java
   paramBuilder.addString("inputDateStr", day);
   paramBuilder.addString("filename", "Refund_Exception_Report_" + day + ".csv");
   JobParameters jobParams = paramBuilder.toJobParameters();
   ```

2. **JobLauncher.run()** creates execution and delegates to **Job**.

3. **Job** executes chunk-based **Step**:

   ```java
   .<RefundExceptionRecord, RefundExceptionRecord>chunk(chunkSize, transactionManager)
       .reader(transactionsReader(to_be_injected))
       .processor(transactionDataProcessor())
       .writer(transactionsFileWriter(to_be_injected))
   ```

4. **@StepScope** beans are created with late-bound JobParameters:

   ```java
   @Value("#{jobParameters[inputDateStr]}") String inputDateStr
   @Value("#{jobParameters[filename]}") String filename
   ```

5. **Step** repeats chunk processing until reader returns null:

   a. **ItemReader.read()** reads items one by one:
   
   ```java
   @Override
   public RefundExceptionRecord read() throws Exception {
       if (transactionIterator.hasNext()) {
           return transactionIterator.next();
       }
       return null;
   }
   ```

   b. **ItemProcessor.process()** transforms each item:
   
   ```java
   @Override
   public RefundExceptionRecord process(RefundExceptionRecord transaction) {
       // Transform data
       dataContainer.incrementCount();
       dataContainer.addRefund(new BigDecimal(data.getRefundAmt()));
       return data;
   }
   ```

   c. **ItemWriter.write()** writes chunk of items:
   
   - Header written via **FlatFileHeaderCallback** (once at start).
   - Data records written via **DelimitedLineAggregator**.
   - Footer written via **FlatFileFooterCallback** (once at end).

6. **Step** commits transaction after each chunk.

7. **Job** proceeds to next step (upload):

   ```java
   .flow(getRefundExceptionStep())
   .next(uploadTransactionsFileStep())
   ```

8. **UploadFileTasklet** executes file upload:

   ```java
   @Override
   public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
       String filename = (String) chunkContext.getStepContext().getJobParameters().get("filename");
       service.processRefundExceptionFile(filename);
       return RepeatStatus.FINISHED;
   }
   ```

9. **JobExecution** returned with final status.

---

## Tasklet vs Chunk Comparison

| Aspect | Tasklet-Based | Chunk-Based |
|--------|---------------|-------------|
| **Use Case** | Simple operations, file generation | Large data processing with pagination |
| **Transaction** | Single transaction per tasklet | Transaction per chunk |
| **Memory** | Loads all data at once | Processes data in chunks |
| **Components** | Tasklet only | ItemReader + ItemProcessor + ItemWriter |
| **Parameter Access** | ChunkContext.getStepContext() | @StepScope with #{jobParameters[]} |
| **State Accumulation** | Managed within tasklet | DataContainer shared via @JobScope |
| **Error Recovery** | Restart from beginning | Restart from last committed chunk |
