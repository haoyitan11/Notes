# Spring Batch dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/084673dc-ca29-4640-9e36-d64dbed9af44" />


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
Step
JobRepository
```

#### Functions Used

Start step:

```java
JobBuilder.start(Step);
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

```java
@Bean
public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
    return new JobBuilder("staticQrJob", jobRepository)
            .start(staticQrStep)
            .build();
}
```

### 3. JobParameters

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
```

Build parameters:

```java
JobParametersBuilder.toJobParameters();
```

#### Responsibilities

- Stores execution parameters (runDate, inputDateStr, filename, billCurrency).
- Passed to JobLauncher when running a Job.
- Combined with the Job name to define a unique JobInstance.
- Accessible by Tasklet during execution.

### 4. JobExecution

#### Purpose

Stores the execution result of a Job.

#### Dependencies

```java
JobInstance
BatchStatus
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
```

#### Responsibilities

- Created and returned by JobLauncher.
- Tracks execution status (COMPLETED, FAILED, STOPPED).
- Stores execution metadata and exceptions.
- Used to determine whether error notifications should be sent.

### 5. JobRepository

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

### 6. JobBuilder

#### Purpose

Builds a fully configured Job.

#### Dependencies

```java
JobRepository
Step
```

#### Functions Used

Start step:

```java
JobBuilder.start(Step);
```

Build job:

```java
JobBuilder.build();
```

#### Responsibilities

- Gathers all dependencies required by a Job.
- Integrates JobRepository and Step(s).
- Creates a Job.

### 7. Step

#### Purpose

Defines the work to be executed within a Job.

#### Dependencies

```java
Tasklet
PlatformTransactionManager
JobRepository
```

#### Functions Used

Created via StepBuilder:

```java
StepBuilder.tasklet(Tasklet, PlatformTransactionManager);
StepBuilder.build();
```

#### Responsibilities

- Defines which Tasklet should be executed.
- Contains and executes the Tasklet.
- Acts as a logical unit of work within a Job.
- Created by StepBuilder.

#### Configuration Source

```java
@Bean
public Step staticQrStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("staticQrStep", jobRepository)
            .tasklet(staticQrFileJobTasklet(), transactionManager)
            .build();
}
```

### 8. StepBuilder

#### Purpose

Builds a fully configured Step.

#### Dependencies

```java
JobRepository
Tasklet
PlatformTransactionManager
```

#### Functions Used

Configure tasklet:

```java
StepBuilder.tasklet(Tasklet, PlatformTransactionManager);
```

Build step:

```java
StepBuilder.build();
```

#### Responsibilities

- Gathers all dependencies required by a Step.
- Integrates:
  - Tasklet (defines the business logic to execute)
  - TransactionManager (manages commit and rollback boundaries)
  - JobRepository (persists Step execution metadata)
- Creates a Step.

### 9. Tasklet

#### Purpose

Executes the actual batch business logic.

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

Return status:

```java
return RepeatStatus.FINISHED;
```

#### Responsibilities

- Reads JobParameters (via ChunkContext / StepExecution).
- Integrates with business services.
- Executes batch processing logic.
- Returns RepeatStatus.

## How Spring Batch execution works inside the launcher

1. A scheduled method in `QRBatchJob` builds `JobParameters` using `JobParametersBuilder` (runDate, inputDateStr, etc.).
2. `JobLauncher.run(job, jobParams)` is called with the target Job and its parameters.
3. `JobLauncher` uses the `JobRepository` to resolve or create a `JobInstance` (Job name + JobParameters) and a new `JobExecution`.
4. The `Job` (built by `JobBuilder`) executes its configured `Step`(s).
5. Each `Step` (built by `StepBuilder`) runs its `Tasklet` within a transaction managed by the `PlatformTransactionManager`, and Step metadata is persisted through the `JobRepository`.
6. The `Tasklet` reads JobParameters, runs the business logic, and returns `RepeatStatus.FINISHED`.
7. `JobLauncher` returns the completed `JobExecution`.
8. `sendErrors(execution)` inspects `JobExecution.getStatus()`; if it is not `COMPLETED`, failure exceptions are collected and an error email is sent, otherwise the completion status is logged.
