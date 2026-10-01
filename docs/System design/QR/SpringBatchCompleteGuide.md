# Spring Batch Dependencies Handle

## Overview

This document provides a complete guide for the Spring Batch implementation in the application. It covers two processing patterns:

1. **Tasklet-Based Processing** - Single operation batch execution

2. **Chunk-Based Processing** - Large data processing with pagination

---

# Part 1: Tasklet-Based Processing

![Spring Batch Job Components](https://github.com/user-attachments/assets/c8ff112c-d564-4e45-984b-afc797ce6bfd)

## Purpose

Provides batch execution framework for simple operations where all logic executes in a single transaction.

---

## Components

### 1.1 JobLauncher

#### Purpose

Acts as the central batch execution manager that starts and runs a Job.

#### Dependencies

```java
Job
JobParameters
JobRepository
```

#### Actual Usage

```java
@Autowired
public JobLauncher jobLauncher;

@Autowired
@Qualifier("reconciliationQrJob")
private Job reconciliationQrJob;
```

Run job:

```java
JobExecution execution = jobLauncher.run(reconciliationQrJob, jobParams);
```

#### Responsibilities

- Central coordinator for batch execution.
- Integrates **Job** (defines what to execute).
- Integrates **JobParameters** (runtime inputs).
- Integrates **JobRepository** (creates JobInstance and JobExecution).
- Starts Job execution.
- Returns **JobExecution** holding the execution result.

---

### 1.2 JobParametersBuilder

#### Purpose

Builds runtime parameters for job execution.

#### Dependencies

```java
None (Spring Batch class)
```

#### Actual Usage

```java
String day = DateTime.now().minusDays(1).toString(CommonConsts.DATETIME_FORMAT_CCYYMMDD);
JobParametersBuilder paramBuilder = new JobParametersBuilder();
paramBuilder.addDate("runDate", new Date());
paramBuilder.addString("inputDateStr", day);

JobParameters jobParams = paramBuilder.toJobParameters();
```

#### Responsibilities

- Creates **JobParameters** instance.
- Adds typed parameters (Date, String, Long, Double).
- Builds immutable parameter set.

---

### 1.3 JobParameters

#### Purpose

Stores runtime inputs for a batch execution.

#### Dependencies

```java
JobParametersBuilder
```

#### Actual Usage

Build parameters:

```java
JobParameters jobParams = paramBuilder.toJobParameters();
```

Access in Tasklet:

```java
String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

#### Responsibilities

- Holds runtime inputs.
- Passed to **JobLauncher.run()**.
- Combined with Job name to identify unique **JobInstance**.
- Accessible by Tasklet via **ChunkContext.getStepContext().getJobParameters()**.

---

### 1.4 Job

#### Purpose

Defines the workflow to execute.

#### Dependencies

```java
Step
JobRepository
```

#### Actual Usage

```java
@Bean
public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
    return new JobBuilder("staticQrJob", jobRepository)
            .start(staticQrStep)
            .build();
}
```

#### Responsibilities

- Integrates **Step(s)** (defines the work units to execute).
- Integrates **JobRepository** (persists execution state).
- Executed by **JobLauncher**.
- Iterates through and executes each Step in sequence.

#### Configuration Source

```java
@Configuration
public class StaticQrJobConfig {

    @Bean
    public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
        return new JobBuilder("staticQrJob", jobRepository)
                .start(staticQrStep)
                .build();
    }
}
```

---

### 1.5 Step

#### Purpose

Defines a unit of work to be executed within a Job.

#### Dependencies

```java
Tasklet
PlatformTransactionManager
JobRepository
```

#### Actual Usage

```java
@Bean
public Step staticQrStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("staticQrStep", jobRepository)
            .tasklet(staticQrFileJobTasklet(), transactionManager)
            .build();
}
```

#### Responsibilities

- Integrates **Tasklet** (the actual business logic to execute).
- Integrates **PlatformTransactionManager** (manages commit/rollback boundaries).
- Integrates **JobRepository** (persists StepExecution state).
- Executed by **Job**.
- Calls Tasklet.execute() within a transaction.

#### Configuration Source

```java
@Bean
public Step creditAdjustmentStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("creditAdjustmentStep", jobRepository)
            .tasklet(creditAdjustmentFileTasklet(), transactionManager)
            .build();
}
```

---

### 1.6 Tasklet

#### Purpose

Executes the actual batch business logic.

#### Dependencies

```java
StepContribution
ChunkContext
```

#### Actual Usage

```java
@Component
public class StaticQrFileTasklet implements Tasklet {

    @Value("${batch.sftp.local.dir.generated}")
    private String generatedPath;

    @Value("${batch.sftp.local.dir.staticqr.sent}")
    protected String sentFilePath;

    @Autowired
    QRBatchRepository repository;

    @Autowired
    BatchService service;

    @Autowired
    EmailService email;

    @Value("${sgqr.membercode}")
    protected String memberCode;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        List<StaticQrDetailRecord> dataList = repository.getBulkFileUploadData();

        int fileUploadSequence = service.getNextFileId(DAILY_DIRECTORY_DATE_FORMAT.format(new Date()));
        String formattedFileId = String.format("%03d", fileUploadSequence);
        String fileName = generateBulkFileName(formattedFileId);
        String fileHeader = generateBulkFileHeader(dataList.size(), formattedFileId);
        String fileFooter = generateBulkFileFooter(dataList.size() + 1);

        File file = writeFile(dataList, fileHeader, fileName, fileFooter);
        service.processStaticQrFile(file.getName(), sentFilePath + PATH_DELIMITER + DAILY_DIRECTORY_DATE_FORMAT.format(new Date()), DAILY_DIRECTORY_DATE_FORMAT.format(new Date()));
        updatePosQRStatusToPostedToCR(dataList);

        createFileUploadEntry(formattedFileId, fileName, file.getAbsolutePath());
        return RepeatStatus.FINISHED;
    }
}
```

Access job parameters:

```java
String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

Return status:

```java
return RepeatStatus.FINISHED;
```

#### Responsibilities

- Integrates **StepContribution** (reports step metrics back to Step).
- Integrates **ChunkContext** (provides access to JobParameters).
- Executed by **Step**.
- Performs batch processing logic (file generation, database operations, SFTP).
- Returns **RepeatStatus.FINISHED** when complete.

#### Configuration Source

```java
@Bean
public Tasklet staticQrFileJobTasklet() {
    return new StaticQrFileTasklet();
}
```

---

### 1.7 JobRepository

#### Purpose

Stores and manages Spring Batch execution metadata.

#### Dependencies

```java
DataSource
PlatformTransactionManager
```

#### Actual Usage

Auto-configured by Spring Boot. Used implicitly by JobBuilder and StepBuilder:

```java
@Bean
public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
    return new JobBuilder("staticQrJob", jobRepository)
            .start(staticQrStep)
            .build();
}

@Bean
public Step staticQrStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("staticQrStep", jobRepository)
            .tasklet(staticQrFileJobTasklet(), transactionManager)
            .build();
}
```

#### Responsibilities

- Integrates **DataSource** (persists batch metadata to database).
- Integrates **PlatformTransactionManager** (manages metadata transactions).
- Used by **JobLauncher** (to create JobInstance and JobExecution).
- Used by **Job** (to update JobExecution state).
- Used by **Step** (to create and update StepExecution).
- Stores **JobInstance**, **JobExecution**, **StepExecution**.

---

### 1.8 JobExecution

#### Purpose

Stores the execution result of a Job.

#### Dependencies

```java
JobInstance
BatchStatus
ExitStatus
```

#### Actual Usage

```java
JobExecution execution = jobLauncher.run(reconciliationQrJob, jobParams);
sendErrors(execution);
```

Check status and handle errors:

```java
private void sendErrors(JobExecution execution) {
    if (!execution.getStatus().equals(BatchStatus.COMPLETED)) {
        StringBuilder errors = new StringBuilder();
        for (Throwable t : execution.getAllFailureExceptions()) {
            errors.append(t.getMessage() + "\n");
        }
        logger.error("Batch Job " + execution.getJobInstance().getJobName() + 
                " exited with Status Code " + execution.getStatus() + 
                " with exit message of : " + errors.toString());
        batchEmailService.sendError("Batch Job " + execution.getJobInstance().getJobName() + 
                " exited with Status Code " + execution.getStatus() + 
                " with exit message of : " + errors.toString());
    } else {
        logger.info("Batch Job " + execution.getJobInstance().getJobName() + 
                " exited with Status Code " + execution.getStatus());
    }
}
```

#### Responsibilities

- Created by **JobRepository** when JobLauncher starts a Job.
- Integrates **JobInstance** (identifies which Job + JobParameters).
- Integrates **BatchStatus** (COMPLETED, FAILED, STOPPED).
- Integrates **ExitStatus** and failure exceptions.
- Returned by **JobLauncher.run()** for status checking.

---

## Execution Flow

1. **QRBatchJob** (scheduled method) builds **JobParameters** using **JobParametersBuilder**:

   ```java
   @Scheduled(cron = "${batch.job.scheduling.daily.reconciliation.report}")
   public void executeReconciliationJob() {
       String day = DateTime.now().minusDays(1).toString(CommonConsts.DATETIME_FORMAT_CCYYMMDD);
       JobParametersBuilder paramBuilder = new JobParametersBuilder();
       paramBuilder.addDate("runDate", new Date());
       paramBuilder.addString("inputDateStr", day);
       JobParameters jobParams = paramBuilder.toJobParameters();
   }
   ```

2. **JobLauncher.run(job, jobParams)** is called:

   ```java
   JobExecution execution = jobLauncher.run(reconciliationQrJob, jobParams);
   ```

   - JobLauncher integrates **Job** (what to execute).
   - JobLauncher integrates **JobParameters** (runtime inputs).
   - JobLauncher integrates **JobRepository** to create **JobInstance** and **JobExecution**.

3. **JobLauncher** delegates to **Job.execute()**:

   - Job integrates **Step** (the work unit).
   - Job integrates **JobRepository** to update **JobExecution** state.

4. **Job** delegates to **Step.execute()**:

   - Step integrates **Tasklet** (the business logic).
   - Step integrates **PlatformTransactionManager** to manage transaction boundaries.
   - Step integrates **JobRepository** to persist **StepExecution** state.

5. **Step** calls **Tasklet.execute(StepContribution, ChunkContext)**:

   ```java
   @Override
   public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
       String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
       // Business logic execution
       return RepeatStatus.FINISHED;
   }
   ```

6. **Tasklet** executes business logic and returns **RepeatStatus.FINISHED**.

7. **Step** commits transaction via **PlatformTransactionManager**.

8. **Job** updates **JobExecution** status via **JobRepository**.

9. **JobLauncher** returns **JobExecution** to caller.

10. **sendErrors(execution)** checks **JobExecution.getStatus()** for error handling:

    ```java
    if (!execution.getStatus().equals(BatchStatus.COMPLETED)) {
        // Handle errors and send notifications
    }
    ```

---

# Part 2: Chunk-Based Processing

![Spring Batch Chunk Components](https://github.com/user-attachments/assets/dd1088da-e942-4745-bf47-5c57151abc94)

## Purpose

Provides batch execution framework for large data processing where data is read, processed, and written in configurable chunks with transaction per chunk.

---

## Components

### 2.1 JobLauncher

#### Purpose

Acts as the central batch execution manager that starts and runs a Job.

#### Dependencies

```java
Job
JobParameters
JobRepository
```

#### Actual Usage

```java
@Autowired
public JobLauncher jobLauncher;

@Autowired
@Qualifier("getRefundExceptionJob")
private Job refundExceptionJob;
```

Run job:

```java
JobExecution execution = jobLauncher.run(refundExceptionJob, jobParams);
```

#### Responsibilities

- Central coordinator for batch execution.
- Integrates **Job** (defines what to execute).
- Integrates **JobParameters** (runtime inputs).
- Integrates **JobRepository** (creates JobInstance and JobExecution).
- Starts Job execution.
- Returns **JobExecution** holding the execution result.

---

### 2.2 JobParametersBuilder

#### Purpose

Builds runtime parameters for job execution.

#### Dependencies

```java
None (Spring Batch class)
```

#### Actual Usage

```java
String day = DateTime.now().minusDays(1).toString(CommonConsts.DATETIME_FORMAT_CCYYMMDD);
JobParametersBuilder paramBuilder = new JobParametersBuilder();
paramBuilder.addDate("runDate", new Date());
paramBuilder.addString("inputDateStr", day);
paramBuilder.addString("filename", "Refund_Exception_Report_" + day + ".csv");

JobParameters jobParams = paramBuilder.toJobParameters();
```

#### Responsibilities

- Creates **JobParameters** instance.
- Adds typed parameters (Date, String, Long, Double).
- Builds immutable parameter set.

---

### 2.3 JobParameters

#### Purpose

Stores runtime inputs for a batch execution.

#### Dependencies

```java
JobParametersBuilder
```

#### Actual Usage

Build parameters:

```java
JobParameters jobParams = paramBuilder.toJobParameters();
```

Access via @StepScope late binding:

```java
@Value("#{jobParameters[inputDateStr]}") String inputDateStr
@Value("#{jobParameters[filename]}") String filename
```

#### Responsibilities

- Holds runtime inputs.
- Passed to **JobLauncher.run()**.
- Combined with Job name to identify unique **JobInstance**.
- Accessible via **@StepScope** late binding with **#{jobParameters[key]}**.

---

### 2.4 Job (Multi-Step)

#### Purpose

Defines multi-step workflow combining chunk processing and tasklet steps.

#### Dependencies

```java
Step
JobRepository
RunIdIncrementer
```

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
```

#### Responsibilities

- Integrates **Step(s)** via **flow().next()** chaining.
- Integrates **JobRepository** (persists execution state).
- Integrates **RunIdIncrementer** (ensures unique instances).
- Executed by **JobLauncher**.
- Executes steps in sequence.

---

### 2.5 Step (Chunk-Oriented)

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
@Autowired
private JobRepository jobRepository;

@Autowired
private PlatformTransactionManager transactionManager;

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

#### Responsibilities

- Integrates **ItemReader** (reads items one at a time).
- Integrates **ItemProcessor** (transforms each item).
- Integrates **ItemWriter** (writes chunk of items).
- Integrates **PlatformTransactionManager** (commits per chunk).
- Integrates **JobRepository** (persists StepExecution state).
- Processes data in chunks of configurable size.

---

### 2.6 DataContainer

#### Purpose

Holds accumulated state across chunk processing for footer generation.

#### Dependencies

```java
None (POJO)
```

#### Actual Usage

```java
public class DataContainer {
    
    private Integer totalCount = 0;
    private BigDecimal totalRefundAmount = new BigDecimal(0);
    
    public void incrementCount() {
        this.totalCount += 1;
    }
    
    public void addRefund(BigDecimal amount) {
        this.totalRefundAmount = this.totalRefundAmount.add(amount);
    }
    
    public Integer getTotalCount() {
        return this.totalCount;
    }

    public BigDecimal getTotalRefundAmount() {
        return this.totalRefundAmount;
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

### 2.7 ItemReader

#### Purpose

Reads data items one at a time from a data source for chunk-based processing.

#### Dependencies

```java
QRBatchRepository
Iterator
```

#### Actual Usage

```java
public class RefundExceptionReader implements ItemReader<RefundExceptionRecord> {

    @Autowired
    QRBatchRepository repository;
    
    @Autowired
    EmailService email;
    
    private String inputDate;
    private Iterator<RefundExceptionRecord> transactionIterator;
    
    public RefundExceptionReader(String inputDate) {
        this.inputDate = inputDate;
    }
    
    @PostConstruct
    public void afterConstruct() throws BatchException {
        List<RefundExceptionRecord> exceptionRecs = new ArrayList<>();
        String startDate = CommonConstants.dateBatchInputFormat.parseDateTime(inputDate).withTime(0, 0, 0, 0).toString(CommonConstants.dateTimeBatchDbFormat);
        String endDate = CommonConstants.dateBatchInputFormat.parseDateTime(inputDate).withTime(23, 59, 59, 59).toString(CommonConstants.dateTimeBatchDbFormat);
        exceptionRecs = repository.getOutboundRefundExceptions(startDate, endDate);
        transactionIterator = exceptionRecs.iterator();
    }
    
    @Override
    public RefundExceptionRecord read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        if (transactionIterator.hasNext()) {
            logger.debug("reading record from iterator...");
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

- Integrates **QRBatchRepository** (data source for reading).
- Integrates **JobParameters** via **@StepScope** late binding.
- Returns one item at a time until exhausted (returns null).
- Used by **Step** in chunk-based processing.

---

### 2.8 ItemProcessor

#### Purpose

Transforms or processes each item read by ItemReader before writing.

#### Dependencies

```java
DataContainer
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
        data.setConsumerBank("UOB");
        data.setProcessingStatus("FAILED");

        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

        LocalDateTime originalTxnDt = LocalDateTime.parse(transaction.getTxnDate(), formatter);
        LocalDateTime refundDt = LocalDateTime.parse(transaction.getRefundDate(), formatter);

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

### 2.9 FlatFileItemWriter

#### Purpose

Writes processed items to a flat file destination.

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
public FlatFileItemWriter<RefundExceptionRecord> transactionsFileWriter(@Value("#{jobParameters[filename]}") String filename) {
    FlatFileItemWriter<RefundExceptionRecord> writer = new FlatFileItemWriter<>();
    
    writer.setResource(new FileSystemResource(generatedFolder + filename));
    writer.setShouldDeleteIfEmpty(true);
    
    writer.setLineAggregator(new DelimitedLineAggregator<RefundExceptionRecord>() {
        {
            setDelimiter(",");
            setFieldExtractor(new BeanWrapperFieldExtractor<RefundExceptionRecord>() {
                {
                    setNames(new String[] { "recordType", "txnDate", "txnRRN", "refundRRN", "consumerBank",
                            "refundAmt", "failureReason", "processingStatus", "refundDate", "txnAmount",
                            "currency", "fxRate", "foreignTerminalId", "foreignMerchantId", "merchantName",
                            "netsReferenceNumber"});
                    FileHeaderWriter header = new FileHeaderWriter("HD,Transaction_Date,Original_RRN,Refund_RRN,"
                            + "Consumer_Bank,Refund_Amt,Failure_Reason,Processing_Status,Refund_Timestamp,ORI_PURC_AMOUNT,CCY,FX_Rate,"
                            + "Foreign_Terminal_ID,Foreign_Merchant_ID,Merchant_Name,NETS_Reference_Number");
                    writer.setHeaderCallback(header);
                    FileFooterWriter footer = new FileFooterWriter(refundDataContainer());
                    writer.setFooterCallback(footer);
                }
            });
        }
    });
    
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

### 2.10 FlatFileHeaderCallback

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

### 2.11 FlatFileFooterCallback

#### Purpose

Writes footer record at the end of a flat file.

#### Dependencies

```java
Writer
DataContainer
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

### 2.12 Tasklet (Upload Step)

#### Purpose

Executes file upload after chunk processing completes.

#### Dependencies

```java
StepContribution
ChunkContext
BatchService
```

#### Actual Usage

```java
@Component
public class UploadFileTasklet implements Tasklet {

    BatchService service;

    public UploadFileTasklet(BatchService service) {
        this.service = service;
    }

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        try {
            String filename = (String) chunkContext.getStepContext().getJobParameters().get("filename");
            logger.info("start processing of file with name " + filename);
            service.processRefundExceptionFile(filename);
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
            throw new BatchException(e.getMessage());
        }
        return RepeatStatus.FINISHED;
    }
}
```

Step configuration:

```java
@Bean
public Step uploadTransactionsFileStep() {
    return new StepBuilder("uploadTransactionsFileStep", jobRepository)
            .tasklet(new UploadFileTasklet(service), transactionManager)
            .build();
}
```

#### Responsibilities

- Integrates **BatchService** for file upload.
- Accesses **JobParameters** via **ChunkContext**.
- Executed after chunk processing step.
- Returns **RepeatStatus.FINISHED** when complete.

---

### 2.13 JobExecution

#### Purpose

Stores the execution result of a Job.

#### Dependencies

```java
JobInstance
BatchStatus
ExitStatus
```

#### Actual Usage

```java
JobExecution execution = jobLauncher.run(refundExceptionJob, jobParams);
sendErrors(execution);
```

#### Responsibilities

- Created by **JobRepository** when JobLauncher starts a Job.
- Integrates **JobInstance** (identifies which Job + JobParameters).
- Integrates **BatchStatus** (COMPLETED, FAILED, STOPPED).
- Integrates **ExitStatus** and failure exceptions.
- Returned by **JobLauncher.run()** for status checking.

---

## Execution Flow

1. **QRBatchJob** (scheduled method) builds **JobParameters** using **JobParametersBuilder**:

   ```java
   @Scheduled(cron = "${batch.job.scheduling.daily.refundexception.report}")
   public void executeRefundExceptionJob() {
       String day = DateTime.now().minusDays(1).toString(CommonConsts.DATETIME_FORMAT_CCYYMMDD);
       JobParametersBuilder paramBuilder = new JobParametersBuilder();
       paramBuilder.addDate("runDate", new Date());
       paramBuilder.addString("inputDateStr", day);
       paramBuilder.addString("filename", "Refund_Exception_Report_" + day + ".csv");
       JobParameters jobParams = paramBuilder.toJobParameters();
   }
   ```

2. **JobLauncher.run(job, jobParams)** is called:

   ```java
   JobExecution execution = jobLauncher.run(refundExceptionJob, jobParams);
   ```

   - JobLauncher integrates **Job** (what to execute).
   - JobLauncher integrates **JobParameters** (runtime inputs).
   - JobLauncher integrates **JobRepository** to create **JobInstance** and **JobExecution**.

3. **RunIdIncrementer** adds unique run.id parameter to ensure unique **JobInstance**.

4. **Job** executes first step (chunk processing):

   ```java
   .flow(getRefundExceptionStep())
   ```

5. **@JobScope** bean **DataContainer** is created for state accumulation.

6. **@StepScope** beans are created with late-bound **JobParameters**:

   ```java
   @Value("#{jobParameters[inputDateStr]}") String inputDateStr
   @Value("#{jobParameters[filename]}") String filename
   ```

7. **Step** begins chunk processing loop:

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

   c. **FlatFileItemWriter.write()** writes chunk of items:
   
   - **FlatFileHeaderCallback.writeHeader()** called once at start.
   - **DelimitedLineAggregator** formats each item using **BeanWrapperFieldExtractor**.
   - **FlatFileFooterCallback.writeFooter()** called once at end.

8. **Step** commits transaction after each chunk via **PlatformTransactionManager**.

9. **Step** repeats chunk processing until **ItemReader** returns null.

10. **Job** proceeds to next step (upload):

    ```java
    .next(uploadTransactionsFileStep())
    ```

11. **UploadFileTasklet** executes file upload:

    ```java
    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        String filename = (String) chunkContext.getStepContext().getJobParameters().get("filename");
        service.processRefundExceptionFile(filename);
        return RepeatStatus.FINISHED;
    }
    ```

12. **Job** updates **JobExecution** status via **JobRepository**.

13. **JobLauncher** returns **JobExecution** to caller.

14. **sendErrors(execution)** checks **JobExecution.getStatus()** for error handling.

---

# Summary

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

---

## Component Summary

| Component | Class | Part | Purpose |
|-----------|-------|------|---------|
| JobLauncher | JobLauncher | Part 1 & 2 | Central batch execution manager |
| JobParametersBuilder | JobParametersBuilder | Part 1 & 2 | Builds runtime parameters |
| JobParameters | JobParameters | Part 1 & 2 | Runtime inputs |
| Job | JobBuilder | Part 1 & 2 | Workflow definition |
| RunIdIncrementer | RunIdIncrementer | Part 2 | Unique job instance generator |
| Step | StepBuilder | Part 1 & 2 | Work unit definition |
| Tasklet | Tasklet | Part 1 & 2 | Single operation business logic |
| JobRepository | JobRepository | Part 1 & 2 | Batch metadata persistence |
| JobExecution | JobExecution | Part 1 & 2 | Job execution result |
| @StepScope | Annotation | Part 2 | Late binding scope |
| @JobScope | Annotation | Part 2 | Job-level scope |
| DataContainer | POJO | Part 2 | State accumulation |
| ItemReader | RefundExceptionReader | Part 2 | Data reading |
| ItemProcessor | RefundExceptionProcessor | Part 2 | Data transformation |
| FlatFileItemWriter | FlatFileItemWriter | Part 2 | Data writing |
| DelimitedLineAggregator | DelimitedLineAggregator | Part 2 | CSV line formatting |
| BeanWrapperFieldExtractor | BeanWrapperFieldExtractor | Part 2 | Field extraction |
| FlatFileHeaderCallback | FileHeaderWriter | Part 2 | File header writing |
| FlatFileFooterCallback | FileFooterWriter | Part 2 | File footer writing |

---

## Configuration Files

| File | Purpose |
|------|---------|
| StaticQrJobConfig.java | Static QR job configuration (Tasklet-based) |
| ReconciliationJobConfig.java | Reconciliation job configuration (Tasklet-based) |
| CreditAdjustmentJobConfig.java | Credit adjustment job configuration (Tasklet-based) |
| RefundReportJobConfig.java | Refund exception job configuration (Chunk-based) |

---

## Properties Reference

### Scheduling Properties

```properties
batch.job.scheduling.daily.reconciliation.report=0 0 2 * * ?
batch.job.scheduling.daily.static.qr.report=0 0 3 * * ?
batch.job.scheduling.daily.creditadjustment.report=0 0 4 * * ?
batch.job.scheduling.daily.refundexception.report=0 0 5 * * ?
```

### Processing Properties

```properties
batch.processing.chunk.size=100
```

### File Path Properties

```properties
batch.sftp.local.dir.generated=/path/to/generated
batch.sftp.local.dir.reconciliation.sent=/path/to/sent
batch.sftp.local.dir.staticqr.sent=/path/to/staticqr/sent
batch.sftp.local.dir.creditadjustment.sent=/path/to/creditadjustment/sent
```
