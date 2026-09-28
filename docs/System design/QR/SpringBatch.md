# Spring Batch dependencies handle
## Components
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/084673dc-ca29-4640-9e36-d64dbed9af44" />


### 1. JobLauncher
Function: Starts batch execution
-	Acts as the central batch execution manager.
-	Integrates Job and JobParameters.
-	Creates JobExecution.
-	Responsible for starting Job execution.

### 2. Job
Function: Defines the workflow to execute.
- Defines which Step(s) should be executed. 
- Acts as a container for Step(s). 
- Created by JobBuilder. 
- Executed by JobLauncher.

### 3. JobParameters
Function: Stores runtime inputs for batch execution.
- Stores execution parameters. 
- Passed to JobLauncher. 
- Accessible by Tasklet.

### 4. JobExecution
Function: Stores the execution result of a Job.
- Created and returned by JobLauncher.
- Tracks execution status.
- Stores execution metadata and exceptions.
Status like: COMPLETED, FAILED, STOPPED

### 5. JobRepository
Function: Stores and manages Spring Batch execution metadata.
-	Persists Job and Step execution information.
-	Used by JobBuilder and StepBuilder.
-	Automatically updated by Spring Batch during execution.
  
Stores
1.	JobInstance - Unique Job + JobParameters
2.	JobExecution - Job execution status and results
3.	StepExecution - Step execution status and results

### 6. JobBuilder
Function: Builds a fully configured Job.
- Gathers all dependencies required by a Job.
- Integrates: JobRepository and Step(s)
- Creates a Job.

### 7. Step
Function: Defines the work to be executed within a Job.
-	Defines which Tasklet should be executed.
-	Contains and executes the Tasklet.
-	Acts as a logical unit of work within a Job.
-	Created by StepBuilder.

### 8. StepBuilder
Function: Builds a fully configured Step.
-	Gathers all dependencies required by a Step.
-	Integrates: Tasklet (Defines the business logic to execute), TransactionManager (Manages commit and rollback boundaries), JobRepository (Persists Step execution metadata)
-	Creates a Step.

### 9. Tasklet
Function: Executes the actual batch business logic
- Reads JobParameters. 
- Integrates with business services. 
- Executes batch processing logic. 
- Returns RepeatStatus.
