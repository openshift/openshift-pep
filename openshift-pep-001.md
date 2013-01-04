Status: draft
Author: Clayton Coleman <ccoleman@redhat.com>
Arch Priority: medium
Complexity: 40
Affected Components: broker, admin_tools
User Impact: high
PEP: 1  
Title: Admin Console  
Status: draft  
Author: Clayton Coleman <ccoleman@redhat.com>  
Arch Priority: medium  
Complexity: 40  
Affected Components: broker, admin_tools  
Affected Teams: UI (3), Broker (1), Enterprise (1)  
User Impact: high  
Epic:  

Abstract
--------
Create a server component that can provide visualization and information retrieval to an admin running an OpenShift deployment.  The component must:

* Be independent from the broker so that it can be network routed or secured appropriately
* Provide simple logical visualization of the nodes, districts, and gears within an openshift deployment so that an admin can get a quick-glance capacity overview
* Allow administrators to perform specific task focused information retrieval jobs easily
* Be based on the core broker model objects to insulate from maintenance costs associated with data model changes, and provide reusable pieces of query / format code for CLI or shell based interaction in the future
* Use technologies that are already in use within the OpenShift stack

The admin console will NOT:

* Provide a general monitoring solution for the servers in OpenShift (broker/message queue/mongo)


Motivation
----------
OpenShift currently lacks a "logical" view of the resources in the system that is easily available to an administrator - we introduce a new layer between IaaS/hardware and user's applications.  When we demo OpenShift to customers, it is important that we can show IT purchasers something that is a summary of the system. We need to expose information about that logical mapping - user -> app, app -> gear, gear -> node to administrative users in a manner that is suitable for integration into their systems.  


Specification
-------------

Create a new, standalone rails application that reuses the new broker mongoid model to display a logical view of the OpenShift nodes, districts, and gears, and provide the framework to allow admins to get answers to the following questions:

When writing the rails application, add simple, consistent mongoid model queries to the core models so that multiple sources can reuse those queries.  Those queries will enable admins to retrieve this data from the following locations:

1) The admin console
2) A simple read-only REST API for admins exposed through this rails application
3) Command line tools that will expose output that is formatted like #2
4) The rails console via direct code execution.

The console does not need a user or authentication model.  It will be run from servers that have access to mongo and should use standard broker configuration files to access that info.  Administrators will protect it via apache, firewalls, and binding to local loopbacks.  The console should be very similar in concept to the beanstalkd_view and the resque admin server https://github.com/defunkt/resque#the-front-end.

When users want to extend and enhance the console, it is expected that they will make changes directly to the codebase to add new features (and contribute them back to the community).  The console should avoid non-Rails abstractions wherever possible - we want administrators and developers with only a minimum of Rails experience to be able to make changes to the application.  Rails engines are too complex and difficult to implement.  We should use common Rails patterns whenever possible to reduce duplicate code, but it is acceptable to have a slightly lower standard of completeness vs. a higher level of code clarity and readability.

The console will begin as a read-only experience that makes it easy for administrators to answer questions about the deployment.  If a task requires the user make a change to the data in Mongo, we should inform the user in the console how to accomplish the task using the administrative command line tools and make it as easy as possible for them to run those tasks.  

### Example: Admin needs to view and manage quotas for users who are near their quota limit

* The console exposes a view that shows those users by querying mongo
* In order to change the quota, the admin will have to run a CLI command oo-set-quota
* The UI should tell the admin what command to run, and where possible substitute variables

This will reduce maintenance and development costs and keep the focus a comprehensive CLI experience.  The admin console should connect to mongo in read only mode.


### Primary Use Cases

The top tasks that admins must accomplish from within the console are:

1. See a view of all nodes that includes the number of gears, idle count, and capacity planning info.  
    * Group nodes by districts, which are clustered by gear size, sort nodes by gears per node, users per node, MCS labels remaining.  
    * Show the sorted stat per node (eventually show multiple stats per node?)
       * Gears per node
       * Idled gears per node
       * Users per node
       * MCS labels allocated
       * MCS labels remaining (this is an OpenShift wide constant
    * When grouped, show summaries of the stats of the group next to each section.  

2. Given an application id, display nodes the application gears are deployed to, including the cartridges and IDs in an easy to read format.
    * Include capsule information about that node if necessary
    * Display relevant commands for admin tasks on that app/gear/node

3. Given a user id or a user login, display a list of all owned/accessible applications so the user can quickly jump to the items in #2
    * Summarize their total gears used
    * Display the list of cartridges on each gear by id
    * Also show the list of identities for a user (when identities are added)

4. Given a gear id, display the node, cartridges, and owning user
    * Include the list of SSH keys that user has access to.

5. Allow an administrator to ping a set of nodes (corresponding to a view) and show the status on a page
    * This can begin as a simple listing view, and over time become more integrated into other views

6. Given a node id, display the list of gears deployed to that node
    * Include the list of cartridges installed on those gears, the owning application id, and the owners id
    * Prominently display the district uuid, the cartridges installed, and a summary of the node capacity

7. Display a list of applications created in the last hour/4 hours/day
    * Include owner

8. Display a list of gears provisioned within the last hour/4 hours/day

9. An admin should be able to estimate capacity at a glance by configuring/tweaking the default settings we provide
    * Each gear type, machine type, and cartridge type will interact in customer environments to define capacity
    * The values that result should be tunable in the UI or in a simple config file so that customers can display view #1 in a capacity format relevant to them
    * This may require that certain facts for each node are surfaced in the view (swap, quota, memory, cpu count)

10. Display gears which are near their quota

Many of these operations are not optimized for in our current schema, or require both mcollective and mongo data.  The admin console should seek to minimize the number and frequency of calls, and correctly identity new indices for use within the console.  Some information can be cached within the app (using the simple rails cache) - some information should require the user to click a button to see or to refresh.

The admin console should avoid introducing significant new technologies or concepts into the OpenShift stack.  Graphing or visualization libraries may be useful to enhance display of certain info, but should be limited to their core purpose.  We should reuse the bootstrap CSS and markup patterns and spend visualization effort only the most important scenarios (#1).


### Future considerations

Each of the new queries displayed above, especially #1 and #2, will also be exposed as administrative output scripts.  Those scripts should be fast to start and execute and reuse code from the console, so that they can be frequently executed by an admin team to summarize their application.

At some point, we will expose some of these scenarios as REST API endpoints - these should map as cleanly as possible to existing Rails concepts so as to reduce cost, and reuse code as described above.  The CLI will not use these endpoints.

When background jobs are added administrators will need a way to estimate job backlog and identify failed jobs.  It may be sufficient to do this through beanstalkd_view for the near term.


Rationale
---------
The admin console should be as simple as possible to maintain and build to provide the maximum return on development investment.  Only a small fraction of the user base will access it (admins) and it should be focused on functional simplicity and retrieval of info.  Admins with a minimal amount of Rails and Mongo experience should be able to extend it to retrieve or alter the display.  By limiting the technologies we employ in the design of the console, we can ensure that the widest possible audience can modify it.  There should be minimal interdependency between the broker and the admin console - limited to only core models - so that it can be easily deployed and launched and to reduce the maintenance impact.  The admin console is designed as read-only for the foreseeable future to avoid the significant investment and security considerations involved in duplicating admin level operations in a CLI and a UI.

* The CLI is decoupled and independent of the admin console.  They will rely on code reuse to avoid development costs.

