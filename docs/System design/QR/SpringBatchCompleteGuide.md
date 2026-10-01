# Spring Batch Complete Implementation Guide

## Overview

This document provides a complete guide for the Spring Batch implementation in the application. It covers two processing patterns:

1. **Tasklet-Based Processing** - Single operation batch execution (Synchronous Processing)

2. **Chunk-Based Processing** - Large data processing with pagination (Asynchronous Processing)

---

# Part 1: Tasklet-Based Processing

![Spring Batch Job Components](https://github.com/user-attachments/assets/c8ff112c-d564-4e45-984b-afc797ce6bfd)

## Purpose

Provides batch execution framework for simple operations where all logic executes in a single transaction.

---

## Components

### 1.1 @Scheduled

#### Purpose

Triggers batch job execution based on cron expression.

#### Dependencies

```java
None (Spring Framework annotation)
```

#### Actual Usage

```java
@Scheduled(cron = "${batch.job.scheduling.daily.reconciliation.report}")
public void executeReconciliationJob() {
    // Job execution logic
}
```

#### Responsibilities

- Triggers job execution at scheduled intervals.
- Uses cron expression from properties.
- Entry point for batch processing.

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

### 1.4 JobLauncher

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

### 1.5 JobRepository

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
```

#### Responsibilities

- Integrates **DataSource** (persists batch metadata to database).
- Integrates **PlatformTransactionManager** (manages metadata transactions).
- Used by **JobLauncher** (to create JobInstance and JobExecution).
- Used by **Job** (to update JobExecution state).
- Used by **Step** (to create and update StepExecution).
- Stores **JobInstance**, **JobExecution**, **StepExecution**.

---

### 1.6 JobInstance

#### Purpose

Represents a logical job execution identified by Job name and JobParameters.

#### Dependencies

```java
Job
JobParameters
```

#### Actual Usage

```java
execution.getJobInstance().getJobName()
```

#### Responsibilities

- Uniquely identifies a job run.
- Created by **JobRepository**.
- Used for restart logic.

---

### 1.7 Job

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

### 1.8 Step

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

---

### 1.9 PlatformTransactionManager

#### Purpose

Manages transaction boundaries for step execution.

#### Dependencies

```java
DataSource
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

- Begins transaction before step execution.
- Commits transaction on success.
- Rolls back transaction on failure.
- Integrates with **DataSource**.

---

### 1.10 Tasklet

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
public class CreditAdjustmentTasklet implements Tasklet {

    @Autowired
    QRBatchRepository repository;

    @Autowired
    BatchService service;

    @Value("${batch.sftp.local.dir.generated}")
    protected String generatedFilePath;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
        String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
        
        // Business logic: find transactions, write file, upload
        List<CreditAdjustmentDetailRecord> records = findTransactions(startDate, endDate);
        File creditAdjustmentFile = writeFile(records, inputDateStr);
        service.processCreditAdjustmentFile(creditAdjustmentFile.getName(), sentFolderDir, records);
        
        return RepeatStatus.FINISHED;
    }
}
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

### 1.11 StepContribution

#### Purpose

Holds step execution metrics and contribution data.

#### Dependencies

```java
StepExecution
```

#### Actual Usage

```java
@Override
public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
    // contribution tracks read/write/skip counts
    return RepeatStatus.FINISHED;
}
```

#### Responsibilities

- Tracks read count, write count, skip count.
- Reports metrics back to **StepExecution**.
- Passed to **Tasklet.execute()**.

---

### 1.12 ChunkContext

#### Purpose

Provides access to job parameters and step context within Tasklet.

#### Dependencies

```java
StepContext
```

#### Actual Usage

```java
String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

#### Responsibilities

- Provides access to **StepContext**.
- Provides access to **JobParameters**.
- Passed to **Tasklet.execute()**.

---

### 1.13 StepContext

#### Purpose

Provides step-level context information.

#### Dependencies

```java
StepExecution
JobParameters
```

#### Actual Usage

```java
chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

#### Responsibilities

- Provides access to **JobParameters**.
- Provides access to **StepExecution**.
- Accessed via **ChunkContext**.

---

### 1.14 RepeatStatus

#### Purpose

Indicates whether step should continue or finish.

#### Dependencies

```java
None (Spring Batch enum)
```

#### Actual Usage

```java
return RepeatStatus.FINISHED;
```

#### Responsibilities

- **FINISHED**: Step should not repeat.
- **CONTINUABLE**: Step may be called again.
- Returned by **Tasklet.execute()**.

---

### 1.15 StepExecution

#### Purpose

Stores the execution result of a Step.

#### Dependencies

```java
JobExecution
BatchStatus
ExitStatus
```

#### Actual Usage

```java
// Accessed internally by Spring Batch
// StepExecution is created by JobRepository
```

#### Responsibilities

- Stores step execution state.
- Holds read/write/commit counts.
- Linked to parent **JobExecution**.

---

### 1.16 JobExecution

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

### 1.17 BatchStatus

#### Purpose

Enumeration representing batch execution status.

#### Dependencies

```java
None (Spring Batch enum)
```

#### Actual Usage

```java
if (!execution.getStatus().equals(BatchStatus.COMPLETED)) {
    // Handle errors
}
```

#### Responsibilities

- Represents status: COMPLETED, FAILED, STOPPED, STARTING, STARTED, STOPPING, ABANDONED, UNKNOWN.
- Used by **JobExecution** and **StepExecution**.

---

### 1.18 ExitStatus

#### Purpose

Represents the exit status of a batch execution.

#### Dependencies

```java
None (Spring Batch class)
```

#### Actual Usage

```java
execution.getExitStatus().getExitCode();
```

#### Responsibilities

- Holds exit code and description.
- Default codes: COMPLETED, FAILED, STOPPED, NOOP.
- Used by **JobExecution** and **StepExecution**.

---

## Tasklet-Based Execution Flow

1. **@Scheduled** triggers **executeReconciliationJob()** method based on cron expression.

2. **JobParametersBuilder** creates **JobParameters**:

   ```java
   JobParametersBuilder paramBuilder = new JobParametersBuilder();
   paramBuilder.addDate("runDate", new Date());
   paramBuilder.addString("inputDateStr", day);
   JobParameters jobParams = paramBuilder.toJobParameters();
   ```

3. **JobLauncher.run(job, jobParams)** is called:

   ```java
   JobExecution execution = jobLauncher.run(reconciliationQrJob, jobParams);
   ```

   - JobLauncher integrates **Job** (what to execute).
   - JobLauncher integrates **JobParameters** (runtime inputs).
   - JobLauncher integrates **JobRepository** to create **JobInstance** and **JobExecution**.

4. **JobRepository** creates **JobInstance** and **JobExecution**:

   - JobInstance uniquely identifies job by name + parameters.
   - JobExecution tracks this particular run.

5. **Job** executes via **Job.execute()**:

   - Job integrates **Step** (the work unit).
   - Job integrates **JobRepository** to update **JobExecution** state.

6. **Step** executes via **Step.execute()**:

   - Step integrates **Tasklet** (the business logic).
   - Step integrates **PlatformTransactionManager** to begin transaction.
   - Step integrates **JobRepository** to create **StepExecution**.

7. **PlatformTransactionManager** begins transaction.

8. **Step** calls **Tasklet.execute(StepContribution, ChunkContext)**:

   ```java
   @Override
   public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
       String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
       // Business logic execution
       return RepeatStatus.FINISHED;
   }
   ```

   - **ChunkContext** provides access to **StepContext**.
   - **StepContext** provides access to **JobParameters**.
   - **StepContribution** tracks execution metrics.

9. **Tasklet** executes business logic and returns **RepeatStatus.FINISHED**.

10. **PlatformTransactionManager** commits transaction.

11. **Step** updates **StepExecution** with **BatchStatus** and **ExitStatus**.

12. **Job** updates **JobExecution** status via **JobRepository**.

13. **JobLauncher** returns **JobExecution** to caller.

14. **sendErrors(execution)** checks **JobExecution.getStatus()** for error handling:

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

### 2.1 @Scheduled

#### Purpose

Triggers batch job execution based on cron expression.

#### Dependencies

```java
None (Spring Framework annotation)
```

#### Actual Usage

```java
@Scheduled(cron = "${batch.job.scheduling.daily.refundexception.report}")
public void executeRefundExceptionJob() {
    // Job execution logic
}
```

#### Responsibilities

- Triggers job execution at scheduled intervals.
- Uses cron expression from properties.
- Entry point for batch processing.

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

### 2.4 JobLauncher

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

### 2.5 JobRepository

#### Purpose

Stores and manages Spring Batch execution metadata.

#### Dependencies

```java
DataSource
PlatformTransactionManager
```

#### Actual Usage

```java
@Autowired
private JobRepository jobRepository;

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

- Integrates **DataSource** (persists batch metadata to database).
- Integrates **PlatformTransactionManager** (manages metadata transactions).
- Used by **JobLauncher** (to create JobInstance and JobExecution).
- Used by **Job** (to update JobExecution state).
- Used by **Step** (to create and update StepExecution).
- Stores **JobInstance**, **JobExecution**, **StepExecution**.

---

### 2.6 RunIdIncrementer

#### Purpose

Ensures unique JobInstance for each job execution.

#### Dependencies

```java
JobParameters
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

- Adds unique **run.id** parameter to each execution.
- Ensures each job run creates a new **JobInstance**.
- Allows same job parameters to run multiple times.

---

### 2.7 Job (Multi-Step)

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

### 2.8 Chunk-Oriented Step

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

### 2.9 @StepScope

#### Purpose

Creates bean instance per step execution with late binding of JobParameters.

#### Dependencies

```java
JobParameters
StepExecution
```

#### Actual Usage

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

- Creates bean instance per step execution.
- Enables **#{jobParameters[key]}** expression for late binding.
- Required for dynamic parameter injection at runtime.

---

### 2.10 @JobScope

#### Purpose

Creates bean instance per job execution, shared across steps.

#### Dependencies

```java
JobExecution
```

#### Actual Usage

```java
@Bean
@JobScope
DataContainer refundDataContainer() {
    return new DataContainer();
}
```

#### Responsibilities

- Creates bean instance per job execution.
- Shared across all steps in the job.
- Maintains state across steps.

---

### 2.11 ItemReader

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

Bean configuration:

```java
@Bean
@StepScope
ItemReader<RefundExceptionRecord> transactionsReader(
        @Value("#{jobParameters[inputDateStr]}") String inputDateStr) {
    return new RefundExceptionReader(inputDateStr);
}
```

#### Responsibilities

- Integrates **Repository** (data source for reading).
- Integrates **JobParameters** via **@StepScope** late binding.
- Returns one item at a time until exhausted (returns null).
- Used by **Step** in chunk-based processing.

---

### 2.12 ItemProcessor

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

### 2.13 DataContainer

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

Bean configuration:

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

### 2.14 ItemWriter (FlatFileItemWriter)

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

### 2.15 FileSystemResource

#### Purpose

Represents file system location for writing output.

#### Dependencies

```java
None (Spring Core class)
```

#### Actual Usage

```java
writer.setResource(new FileSystemResource(generatedFolder + filename));
```

#### Responsibilities

- Points to output file location.
- Used by **FlatFileItemWriter**.

---

### 2.16 DelimitedLineAggregator

#### Purpose

Aggregates item fields into a delimited line (CSV).

#### Dependencies

```java
BeanWrapperFieldExtractor
```

#### Actual Usage

```java
writer.setLineAggregator(new DelimitedLineAggregator<RefundExceptionRecord>() {
    {
        setDelimiter(",");
        setFieldExtractor(new BeanWrapperFieldExtractor<RefundExceptionRecord>() {
            // ...
        });
    }
});
```

#### Responsibilities

- Converts item to delimited string.
- Integrates **BeanWrapperFieldExtractor** for field extraction.
- Used by **FlatFileItemWriter**.

---

### 2.17 BeanWrapperFieldExtractor

#### Purpose

Extracts field values from item bean for line aggregation.

#### Dependencies

```java
None (Spring Batch class)
```

#### Actual Usage

```java
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
```

#### Responsibilities

- Extracts specified fields from item bean.
- Fields are extracted in order specified.
- Used by **DelimitedLineAggregator**.

---

### 2.18 FlatFileHeaderCallback

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

### 2.19 FlatFileFooterCallback

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

### 2.20 Tasklet (Upload Step)

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
        String filename = (String) chunkContext.getStepContext().getJobParameters().get("filename");
        service.processRefundExceptionFile(filename);
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

### 2.21 JobExecution

#### Purpose

Stores the execution result of a Job.

#### Dependencies

```java
JobInstance
BatchStatus
ExitStatus
StepExecution
```

#### Actual Usage

```java
JobExecution execution = jobLauncher.run(refundExceptionJob, jobParams);
sendErrors(execution);
```

#### Responsibilities

- Created by **JobRepository** when JobLauncher starts a Job.
- Contains collection of **StepExecution** objects.
- Integrates **JobInstance** (identifies which Job + JobParameters).
- Integrates **BatchStatus** (COMPLETED, FAILED, STOPPED).
- Integrates **ExitStatus** and failure exceptions.
- Returned by **JobLauncher.run()** for status checking.

---

## Chunk-Based Execution Flow

1. **@Scheduled** triggers **executeRefundExceptionJob()** method based on cron expression.

2. **JobParametersBuilder** creates **JobParameters**:

   ```java
   JobParametersBuilder paramBuilder = new JobParametersBuilder();
   paramBuilder.addDate("runDate", new Date());
   paramBuilder.addString("inputDateStr", day);
   paramBuilder.addString("filename", "Refund_Exception_Report_" + day + ".csv");
   JobParameters jobParams = paramBuilder.toJobParameters();
   ```

3. **JobLauncher.run(job, jobParams)** is called:

   ```java
   JobExecution execution = jobLauncher.run(refundExceptionJob, jobParams);
   ```

4. **JobRepository** creates **JobInstance** and **JobExecution**:

   - **RunIdIncrementer** adds unique run.id parameter.
   - JobInstance uniquely identifies job by name + parameters.
   - JobExecution tracks this particular run.

5. **Job** executes first step (chunk processing):

   ```java
   .flow(getRefundExceptionStep())
   ```

6. **@JobScope** bean **DataContainer** is created for state accumulation.

7. **@StepScope** beans are created with late-bound **JobParameters**:

   ```java
   @Value("#{jobParameters[inputDateStr]}") String inputDateStr
   @Value("#{jobParameters[filename]}") String filename
   ```

8. **Step** begins chunk processing loop:

   a. **PlatformTransactionManager** begins transaction.
   
   b. **ItemReader.read()** reads items one by one until chunk size reached:
   
   ```java
   @Override
   public RefundExceptionRecord read() throws Exception {
       if (transactionIterator.hasNext()) {
           return transactionIterator.next();
       }
       return null;
   }
   ```

   c. **ItemProcessor.process()** transforms each item:
   
   ```java
   @Override
   public RefundExceptionRecord process(RefundExceptionRecord transaction) {
       // Transform data
       dataContainer.incrementCount();
       dataContainer.addRefund(new BigDecimal(data.getRefundAmt()));
       return data;
   }
   ```

   d. **ItemWriter.write()** writes chunk of items:
   
   - **FlatFileHeaderCallback.writeHeader()** called once at start.
   - **DelimitedLineAggregator** formats each item.
   - **BeanWrapperFieldExtractor** extracts field values.
   - **FlatFileFooterCallback.writeFooter()** called once at end.

   e. **PlatformTransactionManager** commits transaction.

9. **Step** repeats chunk processing until **ItemReader** returns null.

10. **Step** updates **StepExecution** with **BatchStatus** and **ExitStatus**.

11. **Job** proceeds to next step (upload):

    ```java
    .next(uploadTransactionsFileStep())
    ```

12. **UploadFileTasklet** executes file upload:

    ```java
    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
        String filename = (String) chunkContext.getStepContext().getJobParameters().get("filename");
        service.processRefundExceptionFile(filename);
        return RepeatStatus.FINISHED;
    }
    ```

13. **Job** updates **JobExecution** status via **JobRepository**.

14. **JobLauncher** returns **JobExecution** to caller.

15. **sendErrors(execution)** checks **JobExecution.getStatus()** for error handling.

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
| @Scheduled | Annotation | Part 1 & 2 | Triggers job execution |
| JobParametersBuilder | JobParametersBuilder | Part 1 & 2 | Builds runtime parameters |
| JobParameters | JobParameters | Part 1 & 2 | Runtime inputs |
| JobLauncher | JobLauncher | Part 1 & 2 | Central batch execution manager |
| JobRepository | JobRepository | Part 1 & 2 | Batch metadata persistence |
| JobInstance | JobInstance | Part 1 & 2 | Logical job identification |
| Job | JobBuilder | Part 1 & 2 | Workflow definition |
| RunIdIncrementer | RunIdIncrementer | Part 2 | Unique job instance generator |
| Step | StepBuilder | Part 1 & 2 | Work unit definition |
| PlatformTransactionManager | PlatformTransactionManager | Part 1 & 2 | Transaction management |
| Tasklet | Tasklet | Part 1 & 2 | Single operation business logic |
| StepContribution | StepContribution | Part 1 & 2 | Step execution metrics |
| ChunkContext | ChunkContext | Part 1 & 2 | Step context access |
| StepContext | StepContext | Part 1 & 2 | Step-level context |
| RepeatStatus | RepeatStatus | Part 1 & 2 | Step continuation indicator |
| StepExecution | StepExecution | Part 1 & 2 | Step execution result |
| JobExecution | JobExecution | Part 1 & 2 | Job execution result |
| BatchStatus | BatchStatus | Part 1 & 2 | Execution status enum |
| ExitStatus | ExitStatus | Part 1 & 2 | Exit status |
| @StepScope | Annotation | Part 2 | Late binding scope |
| @JobScope | Annotation | Part 2 | Job-level scope |
| ItemReader | ItemReader | Part 2 | Data reading |
| ItemProcessor | ItemProcessor | Part 2 | Data transformation |
| DataContainer | POJO | Part 2 | State accumulation |
| ItemWriter | FlatFileItemWriter | Part 2 | Data writing |
| FileSystemResource | FileSystemResource | Part 2 | File location |
| DelimitedLineAggregator | DelimitedLineAggregator | Part 2 | CSV line formatting |
| BeanWrapperFieldExtractor | BeanWrapperFieldExtractor | Part 2 | Field extraction |
| FlatFileHeaderCallback | FlatFileHeaderCallback | Part 2 | File header writing |
| FlatFileFooterCallback | FlatFileFooterCallback | Part 2 | File footer writing |

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
