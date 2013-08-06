PEP: 3  
Title: Scheduling of Broker Jobs  
Status: draft  
Author: Dan McPherson <dmcphers@redhat.com>, Abhishek Gupta <abhgupta@redhat.com>, Jordan Liggitt <jliggitt@redhat.com>
Arch Priority: medium  
Complexity: 100  
Affected Components: web, api, broker, cli  
Affected Teams: UI 40, Broker 40  
User Impact: medium  
Epic:  

Abstract
--------
Today in OpenShift, every operation that goes through the broker is executed immediately.  Adding a scheduler will change any potentially long running processes (anything involving node operations) to involve multiple API calls.  A first call to launch the task (such as create app) and secondary calls to check the status of the associated job(s).  Clients may choose to hide these requests behind a spinner/progress bar, or they might want to indicate to the user the job has been queued and have the user check the status amongst all of their jobs.  The major advantage of the latter being an immediate response to the user allowing them to move on within the client.

Motivation
----------
OpenShift has numerous tasks that take multiple seconds and even minutes (create application, add cartridge, scale up, etc).  The user experience for these requests today is unpredictable and not really suitable for web requests.  In addition, some operations may be blocked by concurrent operations already happening in the system (Ex: concurrent create app and delete from an impatient user).

The goal of adding a scheduler is to accomplish these long running jobs reliably in the background and allow the user to see the status of existing jobs.  


Specification
-------------

### Today the process for create application looks like:
1. User clicks a button in the web UI/CLI/IDE to create an application 
1. REST API request is made to create the app
1. A series of operations happen on the broker and nodes to create the app
1. REST API response is returned to the caller with the result


### Adding a scheduler will change this process to be:

#### From the perspective of the client:
1. User clicks a button in the web UI/CLI/IDE to create an application
1. REST API request is made to create the app
1. An entry is added to the broker's datastore indicating the job's details and a job is queued in the scheduler referencing back to the broker's job details
1. REST API responds with identifier for the job
1. Client can choose to show status for the job (in_progress, waiting, complete, failed)


#### From the perspective of the scheduler
1. A high level job is queued (create app) from step 2 above
1. The scheduler gets to the job and asks a broker worker to execute create app above and elaborates the high level job.
1. The broker worker elaborates the job into a series of smaller steps that can be retried or rolled back.  These elaborated steps are stored in the broker's datastore.
1. Each of the elaborated jobs are executed and marked as complete.  Failures are retried as appropriate.
1. Completed jobs are marked accordingly and pruned of elaborated steps.  Consistently failed jobs are logged with an opportunity for admin intervention.

### Technology Choices:

+ Our favored scheduling technology is beanstalkd: http://kr.github.com/beanstalkd/
 + Pros: Light weight, Highest performance, Decent UI, Good code style, Best operational characteristics(Process monitoring, Automatic HA, Already available in RHEL)
 + Cons: Concerns over memory leaks
+ Along with the backburner gem: https://github.com/nesquena/backburner


### Implementation Details:

#### Pre-requisites
Prior to the scheduler implementation, the sub-ops will be converted to using their own sub-classes to allow them to manage/dictate their own retry and rollback mechanisms. 
A separate trello card already exists to capture the details of this effort: https://trello.com/c/ZaRSiGj4/40-subclass-pendingops-for-each-op-type


#### Setup
1. The scheduler will be implemented using beanstalkd
1. A separate instance of the scheduler will be running on each broker machine
1. Beanstalkd will be run in persistent mode to persist the jobs to file on disk
1. The worker clients will be implemented using backburner or beaneater (client choice TBD based on prototyping results)
1. Multiple workers will be spawned on each broker machine to process the jobs from the scheduler queues
1. The workers will load the broker Rails environment
1. Workers will process jobs from one or more queues of the scheduler


#### Adding Jobs to the Scheduler
1. Jobs corresponding to REST API requests coming to a particular broker will be added to the scheduler specific to that broker 
1. The job will be added to the scheduler queue right after the pending op is added to mongo for the application/domain/user
1. After adding a job to the scheduler, the field job_creation_status in the pending op document is set to 'scheduled' to indicate that the job has been added to the scheduler
  + This will be a new field in the pending op embedded document
1. Any new user request coming in will make a check to see if there are any pending ops without the flag set to 'scheduled'
  + If so, then corresponding jobs will be added to the scheduler and the field set to 'scheduled'
1. Additionally, the admin script to clear the pending ops will also check for such pending ops and create jobs, if required
  + The admin script oo-admin-clear-pending-ops will have to be renamed ( new name: __TBD__ )
  + A cron job will need to be created to schedule the script to run at regular intervals ( __TBD__: 10, 30, or 60 minutes? ) 
1. The fallback mechanisms helps us avoid having to over-engineer the scheduler design for handling corner cases


#### Job Processing
1. A job_status document will be created in a separate collection in mongo
1. The job_status document will also keep tab of child/linked/dependent jobs
  + In case of domain/user jobs, this list will contain IDs of job_status corresponding to pending ops on applications
  + In case of jenkins application creation, this list will contain IDs of job_status corresponding to pending ops on applications to create environment variables and adding ssh keys
1. The job added to the scheduler will consist of ONLY the application/domain/user ID and not the ID for the pending operation
1. The workers will "reserve" a job to work on and execute the first pending operation within the application/domain/user
  + This will ensure that out of order execution of pending ops does not take place
1. Upon completion of the pending op:
  + The job will be removed from the scheduler queue
  + The result for the operation will be updated in the job_status document and its state will be updated as well
  + The pending op will be removed from mongo from the applications/domains/cloud_users collection
1. The workers are killed/recycled after a certain number of job completions
  + This number of job completions is configurable
  + This avoids issues of memory leaks with long running worker processes


#### Failure Scenarios:

##### What happens if the scheduler goes down?
 + It will be respun by a watcher process
 + All persisted jobs that the scheduler was handling before dying will be reloaded from the file on disk
 + Any jobs that were not added to the scheduler or not yet persisted, will be picked up by the oo-admin-clear-pending-ops script on its next run
 + The oo-admin-clear-pending-ops script will be run at regular intervals and will pick up any jobs that are older than an a certain time limit ( __TBD__: 10, 30, 60 minutes? )
 + We will assume that this time limit is sufficient for the workers to get to a pending op and start executing it as part of regular operation
 + If the worker tries to execute the application job and finds no ops, it will simply mark it job complete and delete it from the queue 
 + Workers will be resilient and will keep retrying connection to the scheduler 

##### What happens if worker process dies?
 + A moritoring script/utility (will be created) to re-spawn workers
 + The in-progress pending op will be placed back in the queue after the job timeout
 + In order to prevent requiring a large job timeout, the worker will frequently "touch" the job to renew the timeout (after every sub-op completion)

##### What happens if the job fails?
 + The failed pending op will be retried once immediately
 + If the retry attempt fails, the job will be added to the scheduler specifying a delay (job delay is beanstalkd functionality)
 + The job delay will be calculated as follows:
   + each sub-op will specify its own retry delay in seconds 
   + the actual delay for a particular retry attempt will be the retry delay multiplied by the retry attempt
   + delay = retry_delay * retry_count (retry attempts already made)
 + The retry_count for the sub-ops will be incremented to indicate the number of retries already performed
 + Each sub-op will specify its re-execution requiremcents, and they could be one of the following:
   + it could be re-executed/retried as-is without worrying about the state of the previous attempt
   + it could specify that the failed sub-op be first rolledback before retrying it
   + it could specify a list of earlier sub-ops (an array of sub-op IDs) that will need to be rolled back (along with any sub-ops that depend on them) before they can all be re-attempted
 + During a retry attempt, first any specified sub-ops are rolled back before retrying the pending op again
 + Each sub-op could fail a few times (as long as it is less than the retry limit for the sub-op) before being successfully executed and the pending op execution would continue 
 + The admin script to clear the pending ops will also look at failed pending ops that haven't exhausted the retry limit and have not been updated for longer than the retry delay (based on the retry count) + 10 minutes  (this delay is to allow the workers to get to the job)
   + If it finds any pending ops that fit the criteria, it adds jobs to the scheduler for them

##### What happens if the job fails even after all retry attempts?
 + Upon the failure of the last retry attempt, it will be rolled back immediately
 + In case of failure to rollback, the pending op rollback will be retried a fixed number of times
   + The number of retries are specified by each sub-op and managed at the sub-op level
   + Each sub-op could fail a few times (as long as it is less than the rollback retry limit for the sub-op) before being successfully rolled back and the pending op rollback operation would continue 
   + A new field rollback_retry_count will be added to the sub-ops to indicate the number of retries already performed
   + A job will be added to the scheduler specifying a certain retry delay (job delay is beanstalkd functionality) 
 + The rollback retry delay will be calculated in the same way as the retry delay for the sub-op
 + The admin script to clear the pending ops will also look at failed pending ops that haven't exhausted the rollback retry limit and have not been updated for longer than the rollback retry delay (based on rollback retry count) + 10 minutes (this delay is to allow the workers to get to the job)
   + If it finds any pending ops that fit the criteria, it adds jobs to the scheduler for them

##### What happens if the rollback fails for a pending op and it gets stuck?
 + A failed pending op that is stuck blocks the execution of any subsequent pending ops for that user/domain/application
 + We will not skip a failed job as we want to ensure that out-of-order execution of pending ops does not happen
 + We will continue to queue up additional pending ops based on user requests upto a certain limit
 + Once a certain number (configurable) of pending ops are already present, we will no longer create additional pending ops and return an error to the user instead
 + The admin script to clear pending ops (to be renamed) will also highlight failed jobs in its output

##### What happens if a worker gets a job for an application/domain/user with a failed/stuck job?
 + The worker will pick up the pending op and look at its retry_count as well as last update time
 + If the retry_count is less than the retry limit for the pending op AND the time since the pending op last update is more than the retry delay, then the job is retried, else the job is skipped and removed from the scheduler queue
 + If the retry limit is reached and the job is still stuck, no further action is taken and manual intervention will be required by Ops/Admin
 + If the retry attempts succeed in executing/rolling back the pending op, then jobs corresponding to any existing pending ops in the queue are added to the scheduler


#### Job Status:
+ The job status will be stored in a separate collection in mongo
+ To get the exact state, the pending op in the user/domain/application object will be queried
+ In case linked/child jobs are present, they will be queried to determine the job state
   + This will be done by the broker and will need to be very fast
   + The broker can get all the child jobs for a given job in a single shot from mongo 
+ If a job_status document is not created/added, then one is created/added when the worker finishes the job and has to save/store the job status

##### Job Status Attributes
The job status will contain the following fields in the response to the client:

+ job_id: identifier of the corresponding job
  + This will be the ID of the pending op within application/domain/user document
+ type: [ "create_app" | "add_component" | "start_app" | etc ]
+ title: job title in sentence text (eg: Create application 'xyz')
+ description: Job details or currently running steps in sentence text
  + This field may not be implemented initially and may be null
+ args: arguments for the corresponding job, stored here to help describe the job
  + Its a hash of key-value pairs
  + The argument list could get big for a few jobs once we allow users to access domains that they don't own
  + __TODO__: Need to investigate the possibility of NOT storing all the arguments
+ child_jobs: array of job_status IDs corresponding to the jobs that are children of this job
  + The field will be populated as child
+ parent_job: ID of the parent job in case this is a child job 
+ state: [ "queued" | "executing" | "reverting" | "complete" ]
+ completion_state: [ "success" | "partial_success" | "failed" ]
  + Field is populated only once the job is complete
  + partial_success can happen when a user/domain operation fails on some applications and succeeds on others
  + failed state indicates that the job execution failed and there were no rollbacks required/performed
+ retry_count: The number of retries performed to execute the sub-op
  + This field will be applicable only when the state is "executing"
  + If the sub-op retry is successful and the execution proceeds, this fields will be reset to 0
+ rollback_retry_count: The number of retries performed to rollback/revert a failed sub-op
  + This field will be applicable only when the state is "reverting"
  + If the sub-op rollback retry is successful and the rollback proceeds, this field will be reset to 0
+ percentage_complete: valid values are [ null |  0.0-100.0 ]
  + This field will be null if completion percentage cannot be determined
  + This could be the ratio of number of sub-ops completed vs total sub-ops
  + As the job execution progresses, we could update the percentage_complete value, so that it is not recalculated for every API call
+ result: result_io object returned by the operation
  + In the regular case it will be present only once the state is "complete"
  + In case of failed retries and rollbacks, the result field will include the output of the failures
  + At a later point, we may start storing intermediate results from sub-op executions
+ object_type: [ null | "application" | "domain" | "user" | "key" | "cartridge" ]
+ application_id: application ID
+ application_name: application name
+ domain_name: domain namespace
+ owner_login: user login of the owner of the application/domain/user object on which the job is being performed
+ creator_login: login of the user who created the job (made the user request)
  + Initially this field will be the same as the owner_login
  + Once we have multiple users accessing a domain, this could be different
+ object_url: URL for the resource that was created/modified for the client to query it
  + This field could be null in some cases, for instance when the request was to delete the application/domain  

##### Job State Transition Examples
The format is: state(completion_state)(retry_count)(rollback_retry_count)

+ Successful job completion
  + queued(nil)(0)(0) --> executing(nil)(0)(0) --> complete(success)

+ Successful job after 3 retries
  + queued(nil)(0)(0) --> executing(nil)(0)(0) --> executing(nil)(1)(0) --> queued(nil)(1)(0) --> executing(nil)(2)(0) --> queued(nil)(2)(0) --> executing(nil)(3)(0) --> complete(success)

+ Successful rollback of a failed job after 3 execution retries and 2 rollback retries
  + queued(nil)(0)(0) --> executing(nil)(0)(0) --> executing(nil)(1)(0) --> queued(nil)(1)(0) --> executing(nil)(2)(0) --> queued(nil)(2)(0) --> executing(nil)(3)(0) --> reverting(nil)(3)(0) --> queued(nil)(3)(0) --> reverting(nil)(3)(1) --> queued(nil)(3)(1) --> reverting(nil)(3)(2) --> complete(failed)

+ Failed rollback of a job after 3 execution attempts and 3 rollback attempts  
  + queued(nil)(0)(0) --> executing(nil)(0)(0) --> executing(nil)(1)(0) --> queued(nil)(1)(0) --> executing(nil)(2)(0) --> queued(nil)(2)(0) --> executing(nil)(3)(0) --> reverting(nil)(3)(0) --> queued(nil)(3)(0) --> reverting(nil)(3)(1) --> queued(nil)(3)(1) --> reverting(nil)(3)(2) --> queued(nil)(3)(2) --> reverting(nil)(3)(3)
 
##### Get Job Status
GET /jobs/\<job_id\>  

##### List of Job Statuses visible to the user (will require pagination)
GET /jobs

##### List of Job Statuses for a user (will require pagination)
GET /user/jobs

##### List of Job Statuses for a domain (will require pagination)
GET /domains/\<dom_name\>/jobs  

##### List of Job Statuses for an app (will require pagination)
GET /domains/\<dom_name\>/applications/\<app_name\>/jobs  
GET /applications/\<app_id\>/jobs

##### Job status filtering and sorting
The REST API could allow options for:
+ Sort by time (eg: most recent first)
+ Filter by creator (eg: show only jobs I created)
+ Filter by status (eg: show queued and active jobs)
+ Filter by IDs
  + The UI will typically poll status for the jobs a user has launched
  + Having a way to query n jobs with a single request (upto a certain limit) would be helpful

### REST API Changes:
The following APIs will return a 202(Accepted) with the body being the url of the status API for the job:  
PUT /domains/\<dom_name\>  

POST /user/keys  
PUT /user/keys/\<key_name\>  
DELETE /user/keys/\<key_name\>  

POST /domains/\<dom_name\>/applications  
DELETE /domains/\<dom_name\>/applications/\<app_name\>  

POST /domains/\<dom_name\>/applications/\<app_name\>/events  

POST /domains/\<dom_name\>/applications/\<app_name\>/cartridges  
DELETE /domains/\<dom_name\>/applications/\<app_name\>/cartridges/\<cart_name\>  

POST /domains/\<dom_name\>/applications/\<app_name\>/cartridges/\<cart_name\>/events  


Backwards Compatibility
-----------------------
1. This change will require a new version of the REST API. 
1. For requests made with the earlier REST API versions, the REST layer will loop through the job status and return the result back to the client
1. For newer API versions, the default behavior will be to return a HTTP 202 status with the job URL
1. __TBD__: The new version of the REST API can potentially accept a parameter (something like 'async=false') from the user to determine whether to send the job URL with 202 status or poll the job status and return the result on completion


Rationale
---------
### Other scheduling technologies considered:
+ Resque
  + Pros: Best UI, Good code style, Solid job workers 
  + Cons: Requires Redis (More heavyweight and complex to get HA)
+ delayed_job
  + Pros: Lighter weight, Mongo synergy
  + Cons: UI not as polished, Mongo version compatibility concerns


FAQs
----
##### Who maintains what queue is fed by which brokers?
+ Broker configuration points to the IP/Port of the queue

##### What exactly is pushed into the queue(s)?
+ A "job" data structure which would primarily be a pointer to the application (application ID) and, if needed, some other details (not sure what else would be required)

##### What are the benefits of having a separate scheduler for each broker?
+ automatic job load balancing
+ federated queues
+ less job contention between workers
+ reduced impact in case of scheduler process on a single broker failure/dying
+ avoiding network latency - both for adding and reserving a job

##### Who pops from the queue(s)?
+ The workers (queue clients) reserve jobs from the queue
+ Whenever workers are spun, they are given the location of the queue

##### How easy is it to add more machines/cores to start chewing more ops?
+ A new broker would need to be added
+ User requests coming to the new broker will be handled by the workers on it

##### What if a broker goes down?
+ If the broker rails app goes down, the scheduler process is still running, the jobs are intact, and the worker processes can continue
+ If the broker machine along with the scheduler crashes, then upon restart, the jobs persisted to disk are reloaded by the scheduler and can be picked up by the workers

##### What if ops want to remove a broker?
+ If Ops wants to shut down a broker, similar to ensuring that the broker processes are not working, the scheduler workers will need to be monitored to make sure that workers are done and also that the queue is empty
+ Shutting down a broker would also require that the broker not get any new user requests from the controllers - a way of "deactivating" it or taking it out of the "load balancer" or whatever we use to distribute user requests to our brokers

