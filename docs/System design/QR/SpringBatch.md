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
- Loads Job from Spring context.
- Accepts JobParameters as runtime inputs.
- Uses JobRepository to create JobInstance and JobExecution.
- Starts Job execution.
- Returns JobExecution holding the execution result.

### 2. Job
#### Purpose
Defines the workflow to execute.

#### Dependencies
```java
Step
JobRepository
```

#### Functions Used
Execute steps:
```java
Job.execute(JobExecution);
```

#### Responsibilities
- Contains one or more Step(s) to execute.
- Executed by JobLauncher.
- Uses JobRepository to persist execution state.
- Returns control to JobLauncher after all steps complete.

#### Configuration Source
```java
@Bean
public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
    return new JobBuilder("staticQrJob", jobRepository)
            .start(staticQrStep)
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

Configure steps:
```java
JobBuilder.start(Step);
JobBuilder.flow(Step).next(Step).end();
```

Build job:
```java
JobBuilder.build();
```

#### Responsibilities
- Accepts JobRepository for metadata persistence.
- Accepts Step(s) to define the workflow.
- Integrates JobRepository and Step(s) into a Job.
- Creates a fully configured Job.

### 4. Step
#### Purpose
Defines a unit of work to be executed within a Job.

#### Dependencies
```java
Tasklet (or ItemReader, ItemProcessor, ItemWriter)
PlatformTransactionManager
JobRepository
```

#### Functions Used
Execute tasklet:
```java
Step.execute(StepExecution);
```

#### Responsibilities
- Contains Tasklet or chunk-based components (ItemReader/ItemProcessor/ItemWriter).
- Uses PlatformTransactionManager for transaction boundaries.
- Uses JobRepository to persist StepExecution state.
- Executes the contained Tasklet or chunk processing.

#### Configuration Source
Tasklet-based:
```java
@Bean
public Step staticQrStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("staticQrStep", jobRepository)
            .tasklet(staticQrFileJobTasklet(), transactionManager)
            .build();
}
```

Chunk-based:
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

Configure chunk:
```java
StepBuilder.<I, O>chunk(chunkSize, PlatformTransactionManager)
           .reader(ItemReader)
           .processor(ItemProcessor)
           .writer(ItemWriter);
```

Build step:
```java
StepBuilder.build();
```

#### Responsibilities
- Accepts JobRepository for metadata persistence.
- Accepts Tasklet or ItemReader/ItemProcessor/ItemWriter for business logic.
- Accepts PlatformTransactionManager for transaction management.
- Integrates all dependencies into a Step.
- Creates a fully configured Step.

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
- Receives StepContribution to report back step metrics.
- Receives ChunkContext to access JobParameters and step context.
- Executes batch processing logic (file generation, database operations, SFTP).
- Returns RepeatStatus to indicate completion.

#### Configuration Source
```java
@Bean
public Tasklet creditAdjustmentFileTasklet() {
    return new CreditAdjustmentTasklet();
}
```

### 7. ItemReader / ItemProcessor / ItemWriter
#### Purpose
Handles chunk-oriented batch processing: read items, process them, and write results.

#### Dependencies
```java
DataSource (for database readers)
FileSystemResource (for file writers)
JobParameters (via @Value injection)
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
ItemWriter.write(chunk);
```

#### Responsibilities
- **ItemReader**: Reads data from source one item at a time, returns null when exhausted.
- **ItemProcessor**: Transforms or filters each item, returns null to skip.
- **ItemWriter**: Writes chunk of processed items to destination.
- Works together within a Step, managed by chunk transaction boundaries.

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
Create job execution:
```java
JobRepository.createJobExecution(JobInstance, JobParameters, String);
```

Update execution:
```java
JobRepository.update(JobExecution);
JobRepository.update(StepExecution);
```

#### Responsibilities
- Uses DataSource to persist batch metadata to database.
- Uses PlatformTransactionManager for metadata transaction management.
- Stores JobInstance, JobExecution, StepExecution.
- Provides execution history for restart and recovery.

### 9. JobParameters
#### Purpose
Stores runtime inputs for a batch execution and uniquely identifies a JobInstance.

#### Dependencies
```java
JobParametersBuilder
```

#### Functions Used
Add parameters:
```java
JobParametersBuilder.addDate("runDate", new Date());
JobParametersBuilder.addString("inputDateStr", day);
JobParametersBuilder.addString("filename", "Refund_Exception_Report_" + day + ".csv");
```

Build:
```java
JobParametersBuilder.toJobParameters();
```

#### Responsibilities
- Built by JobParametersBuilder with typed parameters.
- Passed to JobLauncher.run() to start execution.
- Combined with Job name to identify unique JobInstance.
- Accessible by Tasklet via ChunkContext.
- Accessible by @StepScope beans via @Value("#{jobParameters[name]}").

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

Get exceptions:
```java
JobExecution.getAllFailureExceptions();
```

Get job info:
```java
JobExecution.getJobInstance().getJobName();
```

#### Responsibilities
- Created by JobRepository when JobLauncher starts a Job.
- Contains reference to JobInstance (Job + JobParameters identity).
- Tracks BatchStatus (COMPLETED, FAILED, STOPPED).
- Stores ExitStatus and failure exceptions.
- Returned by JobLauncher.run() for status checking.
