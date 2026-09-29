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
- Integrates Job (defines what to execute).
- Integrates JobParameters (runtime inputs).
- Integrates JobRepository (creates JobInstance and JobExecution).
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

- Integrates Step(s) (defines the work units to execute).
- Integrates JobRepository (persists execution state).
- Executed by JobLauncher.
- Iterates through and executes each Step in sequence.

#### Configuration Source

```java
@Bean
public Job staticQrJob(JobRepository jobRepository, Step staticQrStep) {
    return new JobBuilder("staticQrJob", jobRepository)
            .start(staticQrStep)
            .build();
}
```

### 3. Step

#### Purpose

Defines a unit of work to be executed within a Job.

#### Dependencies

```java
Tasklet
PlatformTransactionManager
JobRepository
```

#### Functions Used

Execute tasklet:

```java
Step.execute(StepExecution);
```

#### Responsibilities

- Integrates Tasklet (the actual business logic to execute).
- Integrates PlatformTransactionManager (manages commit/rollback boundaries).
- Integrates JobRepository (persists StepExecution state).
- Executed by Job.
- Calls Tasklet.execute() within a transaction.

#### Configuration Source

```java
@Bean
public Step staticQrStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("staticQrStep", jobRepository)
            .tasklet(staticQrFileJobTasklet(), transactionManager)
            .build();
}
```

### 4. Tasklet

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

Access job parameters:

```java
chunkContext.getStepContext().getJobParameters().get("inputDateStr");
```

Return status:

```java
return RepeatStatus.FINISHED;
```

#### Responsibilities

- Integrates StepContribution (reports step metrics back to Step).
- Integrates ChunkContext (provides access to JobParameters).
- Executed by Step.
- Performs batch processing logic (file generation, database operations, SFTP).
- Returns RepeatStatus.FINISHED when complete.

#### Configuration Source

```java
@Bean
public Tasklet staticQrFileJobTasklet() {
    return new StaticQrFileTasklet();
}
```

### 5. JobRepository

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

- Integrates DataSource (persists batch metadata to database).
- Integrates PlatformTransactionManager (manages metadata transactions).
- Used by JobLauncher (to create JobInstance and JobExecution).
- Used by Job (to update JobExecution state).
- Used by Step (to create and update StepExecution).
- Stores JobInstance, JobExecution, StepExecution.

### 6. JobParameters

#### Purpose

Stores runtime inputs for a batch execution.

#### Dependencies

```java
JobParametersBuilder
```

#### Functions Used

Add parameters:

```java
JobParametersBuilder.addDate("runDate", new Date());
JobParametersBuilder.addString("inputDateStr", day);
```

Build:

```java
JobParametersBuilder.toJobParameters();
```

#### Responsibilities

- Built by JobParametersBuilder.
- Passed to JobLauncher.run() as runtime inputs.
- Combined with Job name to identify unique JobInstance.
- Accessible by Tasklet via ChunkContext.getStepContext().getJobParameters().

### 7. JobExecution

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

#### Responsibilities

- Created by JobRepository when JobLauncher starts a Job.
- Integrates JobInstance (identifies which Job + JobParameters).
- Integrates BatchStatus (COMPLETED, FAILED, STOPPED).
- Integrates ExitStatus and failure exceptions.
- Returned by JobLauncher.run() for status checking.

## How Spring Batch execution works inside the launcher

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              JobLauncher                                     │
│                                                                              │
│  Integrates:                                                                 │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐                      │
│  │     Job      │  │ JobParameters │  │ JobRepository│                      │
│  └──────┬───────┘  └───────────────┘  └──────┬───────┘                      │
│         │                                     │                              │
│         │         creates JobInstance ◄───────┤                              │
│         │         creates JobExecution ◄──────┤                              │
│         ▼                                     │                              │
└─────────┼─────────────────────────────────────┼──────────────────────────────┘
          │                                     │
          ▼                                     │
┌─────────────────────────────────────────────────────────────────────────────┐
│                                  Job                                         │
│                                                                              │
│  Integrates:                                                                 │
│  ┌──────────────┐  ┌──────────────┐                                         │
│  │     Step     │  │ JobRepository│ ◄─────────────────────────────────────┐ │
│  └──────┬───────┘  └──────────────┘   (updates JobExecution state)        │ │
│         │                                                                  │ │
│         ▼                                                                  │ │
└─────────┼──────────────────────────────────────────────────────────────────┼─┘
          │                                                                  │
          ▼                                                                  │
┌─────────────────────────────────────────────────────────────────────────────┐
│                                  Step                                        │
│                                                                              │
│  Integrates:                                                                 │
│  ┌──────────────┐  ┌─────────────────────────┐  ┌──────────────┐            │
│  │   Tasklet    │  │PlatformTransactionManager│  │ JobRepository│            │
│  └──────┬───────┘  └─────────────────────────┘  └──────────────┘            │
│         │                     │                         │                    │
│         │          manages transaction ◄────────────────┤                    │
│         │          persists StepExecution ◄─────────────┘                    │
│         ▼                                                                    │
└─────────┼────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                Tasklet                                       │
│                                                                              │
│  Integrates:                                                                 │
│  ┌──────────────────┐  ┌──────────────┐                                     │
│  │  StepContribution │  │ ChunkContext │                                     │
│  └──────────────────┘  └──────┬───────┘                                     │
│                               │                                              │
│                               ▼                                              │
│                      ┌──────────────┐                                        │
│                      │ JobParameters │ (accessed via ChunkContext)           │
│                      └──────────────┘                                        │
│                                                                              │
│  Returns: RepeatStatus.FINISHED                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Execution Flow

1. **QRBatchJob** (scheduled method) builds **JobParameters** using **JobParametersBuilder**.

2. **JobLauncher.run(job, jobParams)** is called:
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
   - Tasklet integrates **StepContribution** to report metrics.
   - Tasklet integrates **ChunkContext** to access **JobParameters**.

6. **Tasklet** executes business logic and returns **RepeatStatus.FINISHED**.

7. **Step** commits transaction via **PlatformTransactionManager**.

8. **Job** updates **JobExecution** status via **JobRepository**.

9. **JobLauncher** returns **JobExecution** to caller.

10. **sendErrors(execution)** checks **JobExecution.getStatus()** for error handling.
