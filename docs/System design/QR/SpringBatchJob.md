# Spring Batch Dependencies Handle

## Components

![Spring Batch Components Diagram](https://github.com/user-attachments/assets/82448992-483a-4e62-a72f-88a411320ed8)

### 1. JobLauncher

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

### 2. Job

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

### 3. Step

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

### 4. Tasklet

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

### 5. JobRepository

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

### 6. JobParameters

#### Purpose

Stores runtime inputs for a batch execution.

#### Dependencies

```java
JobParametersBuilder
```

#### Actual Usage

Build parameters:

```java
JobParametersBuilder paramBuilder = new JobParametersBuilder();
paramBuilder.addDate("runDate", new Date());
paramBuilder.addString("inputDateStr", day);

JobParameters jobParams = paramBuilder.toJobParameters();
```

Access in Tasklet:

```java
String inputDateStr = (String) chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

#### Responsibilities

- Built by **JobParametersBuilder**.
- Passed to **JobLauncher.run()** as runtime inputs.
- Combined with Job name to identify unique **JobInstance**.
- Accessible by Tasklet via **ChunkContext.getStepContext().getJobParameters()**.

---

### 7. JobExecution

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

1. **Scheduler** (scheduled method) builds **JobParameters** using **JobParametersBuilder**:

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
   public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
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
