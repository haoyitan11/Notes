# Spring Batch dependencies handle
## Components
<img width="1536" height="1024" alt="Designer (16)" src="https://github.com/user-attachments/assets/c3dcf510-8de8-4590-bc80-9fe0faa9ae3c" />

## Components
### 1. JobLauncher
#### Purpose
Acts as the central batch execution manager that starts and runs a Job.

#### Dependencies
```java
Job
JobParameters
JobRepository
```

#### Functions Used
Run job:
```java
JobLauncher.run(Job, JobParameters);
```

#### Responsibilities
- Central coordinator for batch execution.
- Accepts a Job and its JobParameters.
- Creates a JobInstance and JobExecution via the JobRepository.
- Starts Job execution.
- Returns a JobExecution holding the execution result.

### 2. Job
#### Purpose
Defines the workflow to execute.

#### Dependencies
```java
JobBuilder
Step
JobRepository
```

#### Functions Used
Start step:
```java
JobBuilder.start(Step);
```

Chain steps (flow-based):
```java
JobBuilder.flow(Step).next(Step).end();
```

Configure incrementer:
```java
JobBuilder.incrementer(new RunIdIncrementer());
```

Build job:
```java
JobBuilder.build();
```

#### Responsibilities
- Defines which Step(s) should be executed.
- Acts as a container for Step(s).
- Created by JobBuilder.
- Executed by JobLauncher.

#### Configuration Source
Simple job (single step):
```java
@Bean
public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
    return new JobBuilder("staticQrJob", jobRepository)
            .start(staticQrStep)
            .build();
}
```

Flow-based job (multiple steps):
```java
@Bean
public Job getRefundExceptionJob() {
    return new JobBuilder("getRefundExceptionJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .flow(getRefundExceptionStep())
            .next(uploadTransactionsFileStep())
            .end()
            .build();
}
```

### 3. JobBuilder
#### Purpose
Builds a fully configured Job.

#### Dependencies
```java
JobRepository
Step
```

#### Functions Used
Constructor:
```java
new JobBuilder("jobName", jobRepository);
```

Start step:
```java
JobBuilder.start(Step);
```

Flow-based configuration:
```java
JobBuilder.flow(Step).next(Step).end();
```

Build job:
```java
JobBuilder.build();
```

#### Responsibilities
- Gathers all dependencies required by a Job.
- Integrates JobRepository and Step(s).
- Creates a Job.

### 4. Step
#### Purpose
Defines the work to be executed within a Job.

#### Dependencies
```java
StepBuilder
Tasklet (or ItemReader, ItemProcessor, ItemWriter)
PlatformTransactionManager
JobRepository
```

#### Functions Used
Created via StepBuilder (Tasklet-based):
```java
StepBuilder.tasklet(Tasklet, PlatformTransactionManager);
StepBuilder.build();
```

Created via StepBuilder (Chunk-based):
```java
StepBuilder.<I, O>chunk(chunkSize, PlatformTransactionManager);
StepBuilder.reader(ItemReader);
StepBuilder.processor(ItemProcessor);
StepBuilder.writer(ItemWriter);
StepBuilder.build();
```

#### Responsibilities
- Defines which Tasklet or chunk-based processing should be executed.
- Contains and executes the Tasklet (or Reader/Processor/Writer chain).
- Acts as a logical unit of work within a Job.
- Created by StepBuilder.

#### Configuration Source
Tasklet-based step:
```java
@Bean
public Step staticQrStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("staticQrStep", jobRepository)
            .tasklet(staticQrFileJobTasklet(), transactionManager)
            .build();
}
```

Chunk-based step:
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

### 5. StepBuilder
#### Purpose
Builds a fully configured Step.

#### Dependencies
```java
JobRepository
Tasklet (or ItemReader, ItemProcessor, ItemWriter)
PlatformTransactionManager
```

#### Functions Used
Constructor:
```java
new StepBuilder("stepName", jobRepository);
```

Configure tasklet:
```java
StepBuilder.tasklet(Tasklet, PlatformTransactionManager);
```

Configure chunk processing:
```java
StepBuilder.<I, O>chunk(chunkSize, PlatformTransactionManager);
```

Build step:
```java
StepBuilder.build();
```

#### Responsibilities
- Gathers all dependencies required by a Step.
- Integrates:
  - Tasklet (defines the business logic to execute) OR
  - ItemReader, ItemProcessor, ItemWriter (chunk-based processing)
  - TransactionManager (manages commit and rollback boundaries)
  - JobRepository (persists Step execution metadata)
- Creates a Step.

### 6. Tasklet
#### Purpose
Executes the actual batch business logic in a single execution unit.

#### Dependencies
```java
StepContribution
ChunkContext
```

#### Functions Used
Execute:
```java
Tasklet.execute(StepContribution, ChunkContext);
```

Access job parameters:
```java
chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

Return status:
```java
return RepeatStatus.FINISHED;
```

#### Responsibilities
- Reads JobParameters (via `chunkContext.getStepContext().getJobParameters()`).
- Integrates with business services (repository, file operations, SFTP).
- Executes batch processing logic.
- Returns RepeatStatus (FINISHED or CONTINUABLE).

#### Configuration Source
```java
@Bean
public Tasklet creditAdjustmentFileTasklet() {
    return new CreditAdjustmentTasklet();
}
```

### 7. ItemReader / ItemProcessor / ItemWriter (Chunk-based Processing)
#### Purpose
Handles chunk-oriented batch processing: read items, process them, and write results.

#### Dependencies
```java
DataSource (for database readers)
FileSystemResource (for file writers)
```

#### Functions Used
Reader:
```java
ItemReader.read();
```

Processor:
```java
ItemProcessor.process(item);
```

Writer:
```java
ItemWriter.write(items);
```

#### Responsibilities
- **ItemReader**: Reads data from a source (database, file) one item at a time.
- **ItemProcessor**: Transforms or filters each item.
- **ItemWriter**: Writes processed items to a destination (file, database).
- Spring Batch commits in chunks (e.g., every 100 items).

#### Configuration Source
```java
@Bean
@StepScope
ItemReader<RefundExceptionRecord> transactionsReader(@Value("#{jobParameters[inputDateStr]}") String inputDateStr) {
    return new RefundExceptionReader(inputDateStr);
}

@Bean
@StepScope
ItemProcessor<RefundExceptionRecord, RefundExceptionRecord> transactionDataProcessor() {
    return new RefundExceptionProcessor(refundDataContainer());
}

@Bean
@StepScope
public FlatFileItemWriter<RefundExceptionRecord> transactionsFileWriter(@Value("#{jobParameters[filename]}") String filename) {
    FlatFileItemWriter<RefundExceptionRecord> writer = new FlatFileItemWriter<>();
    writer.setResource(new FileSystemResource(generatedFolder + filename));
    // ... configuration
    return writer;
}
```

### 8. JobRepository
#### Purpose
Stores and manages Spring Batch execution metadata.

#### Dependencies
```java
DataSource
PlatformTransactionManager
```

#### Functions Used
Provided to builders:
```java
new JobBuilder("staticQrJob", jobRepository);
new StepBuilder("staticQrStep", jobRepository);
```

#### Responsibilities
- Persists Job and Step execution information.
- Used by JobBuilder and StepBuilder.
- Automatically updated by Spring Batch during execution.

Stores:
1. JobInstance - Unique Job + JobParameters
2. JobExecution - Job execution status and results
3. StepExecution - Step execution status and results

### 9. JobParameters
#### Purpose
Stores runtime inputs for a batch execution and uniquely identifies a JobInstance.

#### Dependencies
```java
JobParametersBuilder
```

#### Functions Used
Add date parameter:
```java
JobParametersBuilder.addDate("runDate", new Date());
```

Add string parameter:
```java
JobParametersBuilder.addString("inputDateStr", day);
JobParametersBuilder.addString("filename", "Refund_Exception_Report_" + day + ".csv");
JobParametersBuilder.addString("billCurrency", billCurrency);
```

Build parameters:
```java
JobParametersBuilder.toJobParameters();
```

#### Responsibilities
- Stores execution parameters (runDate, inputDateStr, filename, billCurrency).
- Passed to JobLauncher when running a Job.
- Combined with the Job name to define a unique JobInstance.
- Accessible by Tasklet via `chunkContext.getStepContext().getJobParameters()`.
- Accessible by @StepScope beans via `@Value("#{jobParameters[paramName]}")`.

### 10. JobExecution
#### Purpose
Stores the execution result of a Job.

#### Dependencies
```java
JobInstance
BatchStatus
ExitStatus
```

#### Functions Used
Get status:
```java
JobExecution.getStatus();
```

Get failure exceptions:
```java
JobExecution.getAllFailureExceptions();
```

Get job instance:
```java
JobExecution.getJobInstance();
JobExecution.getJobInstance().getJobName();
```

#### Responsibilities
- Created and returned by JobLauncher.
- Tracks execution status (COMPLETED, FAILED, STOPPED).
- Stores execution metadata and exceptions.
- Used to determine whether error notifications should be sent.
