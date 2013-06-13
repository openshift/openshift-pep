PEP: 3  
Title: Scheduling of Broker Jobs  
Status: draft  
Author: Dan McPherson <dmcphers@redhat.com>  
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

### Code Examples:
...

### API Changes:
The following APIs will return a 307(Temporary Redirect) with the body being the url of the status API for the job:  
PUT /domains/\<domain_id\>  

POST /user/keys  
PUT /user/keys/\<key_name\>  
DELETE /user/keys/\<key_name\>  

POST /domains/\<dom_name\>/applications  
DELETE /domains/\<dom_name\>/applications/\<app_name\>  

POST /domains/\<dom_name\>/applications/\<app_name\>/events  

POST /domains/\<dom_name\>/applications/\<app_name\>/cartridges  
DELETE /domains/\<dom_name\>/applications/\<app_name\>/cartridges/\<cart_name\>  

POST /domains/\<dom_name\>/applications/\<app_name\>/cartridges/\<cart_name\>/events  

#### Job Status
GET /jobs/\<job_id\>  

{"status":["complete"|"queued"|"in_progress"|"failed"], "percent_complete":[null|0.0-100.0]}  

#### List of Job Statuses visible to the user (will require pagination)
GET /jobs  

#### List of Job Statuses for a domain (will require pagination)
GET /domains/\<dom_name\>/jobs  

#### List of Job Statuses for an app (will require pagination)
GET /domains/\<dom_name\>/applications/\<app_name\>/jobs  

Backwards Compatibility
-----------------------
This change will require a new version of the REST API.  A decision will have to be made whether the scheduler based APIs will be the only option when they are an option. 

Rationale
---------
### Other scheduling technologies considered:
+ Resque
  + Pros: Best UI, Good code style, Solid job workers 
  + Cons: Requires Redis (More heavyweight and complex to get HA)
+ delayed_job
  + Pros: Lighter weight, Mongo synergy
  + Cons: UI not as polished, Mongo version compatibility concerns
