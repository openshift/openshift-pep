PEP: 005  
Title: Highly Available Web Applications  
Status: accepted  
Author: Rajat Chopra <rchopra@redhat.com>, Mrunal Patel <mpatel@redhat.com>, Mike McGrath <mmcgrath@redhat.com>, Clayton Coleman <ccoleman@redhat.com>  
Arch Priority: high  
Complexity: 40  
Affected Components: *web, api, runtime, broker, admin_tools, cli*  
Affected Teams: Runtime (5), UI (3), Broker (4), Enterprise (1)  
User Impact: high  
Epic: E01 https://trello.com/card/epic-01-i-can-run-an-application-with-a-highly-available-web-tier-ha/512a75379a7ef45063008db9/4  


Abstract
--------

Provide highly available web support for OpenShift applications to ensure that single node and gear failures do not result in downtime, and allow integration into existing routing and load-balancer infrastructures in customer environments.  At high scale, applications should remain manageable and problems with individual gears can be corrected.


Motivation
----------

OpenShift must be able to offer the option of a highly available web tier for production applications.  While the current OpenShift architecture uses a single HAProxy in a gear listening on the public DNS address for the application, future applications must be able to survive the failure of the node or gear containing that proxy.  

Many customer environments include infrastructure that may offer availability and/or load balancing to HTTP servers.  Leveraging that infrastructure will allow deployments of OpenShift that can fit into existing production environments.


Terms
-----

**system proxy** - the HTTP (today apache) server that listens at port 80/443 on each node, terminates SSL (via SNI), routes traffic to gears located on that node via their internal address/port, and manages idling

**router / routing layer** - a set of servers that provides highly available routing and/or load balancing for a set of applications.  An OpenShift infrastructure may have multiple sets of routers.  OpenShift has no routing layer implementation by default - instead, the application DNS is mapped to a specific node, which has a system proxy which chooses which gear receives the traffic.

**load balancing layer** - a set of systems intended to distribute load for one or more applications.  OpenShift implements a default load balancer mechanism within applications using one or more HAProxies inside of web gears.  An implementation may choose to move load balancing above the node level and route traffic directly to public web gears - in this case the implementation may need to be notified of rotate-in/out actions on the individual gears.

**head gear** - in OSE 1.2, the web gear within an application that has HAProxy and corresponds to the node the DNS entry for the app points to.  In this pep, referred to as "web load balancer gear"


Specification
-------------

Within an OpenShift deployment certain applications may need to be highly available in the event of node, gear, or network failures.  Today, each application is load balanced by a single HAProxy instance within a gear of that application (known as the web load balancer gear).  HTTP traffic to the application DNS entry goes directly to the node containing that gear, through the system proxy, into HAProxy, and then out to other gears on other nodes.

Within an application, gears may also behave as TCP load balancers.  A TCP balancer must perform many of the same roles as the web balancer.  The current web load balancer could be extended to support TCP balancing of arbitrary backends by exposing new external ports and advertising those routes to the broker, or a new TCP balancer cartridge could be created that uses its own gears and also advertises those routes.

The mapping between application DNS entries, load balancer gears, and backend gears is known as the OpenShift **routing table**.  The OpenShift broker is the system of record for the routing table, with nodes reserving and releasing ports as necessary to satisfy installed cartridges.  The broker also is responsible for reliably updating a DNS registry with application names and correct servers.  The routing table is maintained for all endpoints, not just those that are HTTP specific.

This PEP specifies:

*  A reliable system programming interface (routing SPI) on the OpenShift broker for notifying an external system of changes to the global routing table.  The interface will be a Ruby plugin to the broker, and receive a stream of change notifications for the routing table of all applications. A series of default implementations and examples will be provided, starting simply with a message queue and moving up to full OpenStack Quantum API integration.  A set of REST APIs for retrieving the application routing tables will also be provided for clients.

*  A new optional system component - a "router" - that proxies incoming HTTP requests from one or more application DNS entries to nodes, web load balancer gears, or web gears.  The component may also perform load balancing, although that is not its primary objective.  This component is typically two or more servers that are capable of high load proxying with fast failover in the event of problems.  The router is aware of some or all of the state of the OpenShift routing table.  The router may provide global TCP balancing, or delegate to a TCP load balancing cartridge within an application.  It is the router's responsibility to provide external TCP port mapping on the appropriate DNS address.

*  An OpenShift application should be able to contain multiple web load balancer gears, which distribute the incoming traffic load and provide redundancy.  The system is capable of automatically creating the appropriate web load balancer gears on scale-up and scale-down and also allows an application owner to manage the exact number of web load balancer gears.  Autoscaling should continue to function on the load balancer gears even in the event one or more balancer gears are unavailable.  When an application becomes highly available, it acquires a new DNS entry.

*  A recommended design for an out of the box router configuration for OpenShift that utilizes the routing SPI to provision, configure, and update a set of systems that take application HTTP traffic and reliably route it to the web load balancer gears within an application.  This configuration should be suitable for very large deployments of OpenShift and be able to be sharded under high load.

*  New operations on OpenShift applications via the REST API to handle management at extreme scale, including the removal of specific gears, notifying the web balancer gears to ignore or send traffic to specific web gears within an application (rotate-in and rotate-out).  This includes the necessary operations to ensure failure of a web balancer gear or node does not halt the flow of operations within an application.


#### Topology diagram


               | Web HTTP traffic to                           | Changes to application (1)
               | app1-me.ha.rhcloud.com                        | Scale, add cart, etc
               |                                               v
               |                                           +-------------+
               |              Heartbeat IP                 | REST API    |
               v              Failover to Proxy 2          +-------------+-----+
         +----------+   +----------+                       |                   |
         |Proxy 1   |   |Proxy 2   |                       |        Broker     |
         |IP 1      +-->|          |                       |                   |
         |          |<--|          |                       +-------------------+
         |          |   |          |<---------------------+| Routing SPI |  ^
         +-----+----+   +----------+      Changes to       +-------------+  |
               |                          applications                      | Persist 
       +-------+-----+                    (3)                               | changes (2)
       |             |                                                      v
    +--|--------+ +--|--------+ +----------+        +----------------------------------------+
    |  v   Node | |  v   Node | |     Node |        | MongoDB (routing table model)          |
    +-------+   | +-------+   | |          |        |                                        |
    |Gear1  |   | |Gear2  |   | +------+   |        | App1  = has HA DNS                     |
    |HAProxy|   | |HAProxy|   | |Gear3 |   |        | Gear1 = Load balance + PHP @ Host+Port |
    |PHP    |   | |PHP    |   | |PHP   |   |        | Gear2 = Load balance + PHP @ Host+Port |
    +----+--+---+ +-----+-+---+ +----------+        | Gear3 = PHP                @ Host+Port |
       ^ |           ^  |         ^ ^               +----------------------------------------+
       | +-----------+--|---------+ |
       +----------------+-----------+


##### Changing an application:

1. A user changing an application via the REST API invokes a command to scale up a PHP web application
2. The broker provisions a new gear on a node, and then records those changes in Mongo along with the host and port of the gear endpoints
3. As part of the provision, the broker sends a notification via the routing SPI about a new gear, along with the necessary info.  The SPI may write that info to a file, a DB, a message bus, or directly to Apache, Nginx, or HAProxy configuration on a remote system.

Because the application is marked as being highly available, it has an HA DNS entry for app1-me.ha.rhcloud.com in addition to its normal app1-me.rhcloud.com DNS entry.

##### Failover of a proxy

In this hypothetical configuration, proxy 1 and proxy 2 are both configured to virtual host route HTTP requests to app1-me.ha-rhcloud.com to any of the web load balancer gears within app1 (gear1 and gear2).  Proxy 1 and proxy 2 are in a heartbeat IP failover setup - if proxy2 detects that proxy1 is not responding, it will be assigned IP address 1 and continue handling traffic.  This is just one possible configuration.  Other applications may be routed via proxy1 and proxy2, but a large system may choose to deploy multiple sets of proxies, each handling a different subset of applications.

#### Related PEPs

This PEP interacts with the deploy/zero-downtime PEP and the Geo PEP.  This PEP does not attempt to define the following topics:

*  TCP load balancing within OpenShift applications or at the infrastructure level.  It is assumed that external TCP balancing will require public ports to be exposed, and also must define how the endpoints are described to clients.  The routing SPI and the REST API will be built to support the possibility of TCP load balancing.
*  Multi-master / failover for pushes/build/deployments (covered by zero-downtime PEP)
*  Websocket support, although it is assumed that web sockets can be load balanced and proxied in a similar fashion to HTTP edge traffic.

   Flowchart for websockets with proxy servers http://imgur.com/cZOuS1Y (from The Definitive Guide to HTML5 WebSocket)


### Routing System Programming Interface (SPI)

OpenShift will reliably distribute notifications of the following changes to applications:

*  Creation of an application

   lists the application DNS and the web load balancer / web gear it points to.  Also lists any other routes created with the app

*  Deletion of an application

   lists the application DNS and the web load balancer / web gear it points to.  Also lists any other routes created with the app

*  Addition/removal of routes to an application

   When a new cartridge is added or removed to an app, additional routes may be available on individual gears.  Those changes would be sent as a list of dns + port entries, types of traffic on that port (http, ssh, https), and whether it's a load-balancer or not

*  Addition/removal/changes to SSL certs of aliases from an application

   An alias is an external route - the mapping between the alias DNS and the application DNS must be known to a routing implementor.  Sends the alias+port and the underlying DNS address+port it should map to.
   
*  Addition/removal of geography changes to an application

   A geo (see [Geo PEP 004](https://github.com/openshift/openshift-pep/blob/master/openshift-pep-004.md)) may convey information about the placement of an application.  Sends the application DNS, the Geo, and the application id
   
*  Changes to the stop/start/rotate-in/rotate-out status of an application or gear

   When components within OpenShift request changes to the running status of a gear or application or whether traffic flows to items in that app.  Sends a list of affected dns + port entries and the types of traffic.

The broker will invoke a Ruby service interface for each of these events when changes are being made to the system in a way that guarantees at-least-once delivery of the event to the interface.  A standard implementation of that interface would typically put those events onto a reliable message queue (such as ActiveMQ) for processing by an external script.  This ensures that multiple workers can process events and notify an external service.  

In addition, OpenShift must support a REST API call per application to retrieve the full state of the routing table for that application, including all DNS entries for endpoints, ports, and the types and responsibilities of each endpoint in the application.  The underlying broker data model will support high performance retrieval of that map.

#### Routing table data model [WIP]

The developer clients and consumers of the routing SPI need to be able to answer the following questions:

*  What is the public http URL for this app? (what if there are multiple?)  

   answers: app dns url, aliases, ha app dns url.  Aliases should take priority, but all need to be listed with the appropriate endpoints (what they resolve to)
  
*  What are the hosts and ports of my web load balancers?

   The mongo data model will contain information about the external HTTP port and host for each load balancer.  
  
*  What are the proxy-ports for each of my db gears?  Do I get a display name for the type of what this is (since most time it's pretty simple)
*  What are the proxy ports for each of my web gears?  When I ask for this, I shouldn't get the URL for phpmyadmin
*  What is the URL of phpmyadmin?  (this implementation can be deferred)
*  What is my git url? (and what protocol is it - git+ssh is all we support today)  What if there are multiple Git URLs?
*  What are all of the proxy ports, protocls, and hostnames for my app so I can establish a bulk port-forward session

Possible API is a list of all endpoints.  Each endpoint has:

*  One or more types/roles - i.e. web, load-balanced, ssh, git, tcp (generic), mysql:master, mysql:slave, redis
   *  Roles/types are used to identify the logical use for the endpoint, and are stable over time
*  A single protocol (maybe 2)
   *  http, https, tcp, git+ssh, mysql, tcp seems like a default
*  A single port number, that maps to the external gear PROXY_PORT 
*  A hostname that uniquely identifies the server that resolves that request.
*  Zero or more additional hostnames associated with this endpoint  
   *  For the app endpoint, the app DNS is the primary entry and the aliases 
*  The gear id that owns that endpoint, or null if no gear handles this endpoint.



### OpenShift Router component

An OpenShift may optionally include one or more router components that route web traffic from application DNS entries to web load balancer gears or individual web gears.  The router(s) are notified of changes to the routing table via the routing SPI, although the exact implementation can vary between deployments.  The most common router configuration would be to distribute incoming HTTP traffic amongst the various web balancer gears for an application, potentially sharded into several separate router components each handling a section of the overall application traffic.  The recommended configuration would be a pair of high volume traffic proxies connected via heartbeat in an active-active or active-passive configuration with fast IP reassignment.

A router component MUST perform the following operations:

*  Provide HTTP virtual host passing to individual web backends for one or more applications.  The router is expected to map external DNS aliases (including application aliases) as well as the application DNS entry associated with high availability.  The standard model would be the external web load balancer port (not via the system proxy) as backends, although implementors may consider external availability and load balancing by going directly to web gears.
*  Detect dead backends and gracefully reallocate traffic

A router component SHOULD perform the following operations:

*  Provide HTTPS termination including SNI for individual aliases

A router component MAY perform the following operations:

*  Divide traffic among multiple frontends via multihomed DNS entries
*  Support global TCP balancing by routing external ports to backends advertised by port/protocol within applications.  Because generic TCP balancing cannot take advantage of virtual hosting, it is expected that a router either listens on  random ports, or listens on multiple physical or virtual IP interfaces at standard ports (3306 for mysql, etc).

It is expected that integrators may choose to use commercial load balancer hardware or software load balancers.


### Multiple web load balancer gears per application

OpenShift exposes a single instance of HAProxy within a scalable application today that is bound to the application DNS - that web balancer gear, if down, prevents traffic from reaching the apps.  To that end, an application should be able to activate multiple HAProxies instances and ensure that those instances can properly cooperate to distribute load.  An application should be able to scale cleanly from 1 gear to 1000 gears, with the infrastructure making correct decisions as necessary to allocate new web balancer gears.  

This section deals primarily with HTTP load balancing, although multi-master TCP balancing is an application of the same principles when embedded withxn other web gears.  The implementation of the load balancer cart and the broker's interaciton with it would be similar on a web gear group as on a db gear group.

In general, all load balancers within an app should be considered active, and it is the responsibility of higher infrastructures (routers) to maintain their own mechanism for tracking backend status.  A load balancer gear should be stoppable, just like other gears.

In general, the minimum availability level is to have two web load balancers, each serving traffic to all web gear backends.  To scale to a traffic load beyond what a single web load balancer gear can handle will require multiple balancers.  A user may elect to make their application capable of multiple balancers - when they do so they are assigned a new DNS application entry that represents the aggregation of the traffic to multiple balancers, while their application DNS entry continues to point to their first load balancer.  By default applications are created with only a single load balancer.  An application owner may choose to return to a single web gear configuration at any time.

There are a few important limits to be aware of:

*  Network traffic a single, in-gear load balancer can handle - effective bandwidth limit of the node (1GB/s)

   Traffic to a single load balancer is limited by the inbound/outbound traffic supported by the kernel/machine.  There may be competition between gears on that node for a fair share of bandwidth.
  
*  Number of backends HAProxy can performantly monitor - at least hundreds

   HAProxy should scale reasonably well to hundreds of backends, but every change to the config (backends) requires a SIGUSR1 signal and a reload of the config, which can affect uptime and status.
  
*  Number of HAProxies that must drain an individual gear of traffic before a restart can continue - expect to limit to 3-10

   If multiple HAProxies point to a single gear, a rotate-out operation must wait for ALL HAProxies to successfully mark the backend down before the gear can be safely stopped.  Because HAProxy failures are independent events, every additional front end load balancer significantly increases the probability of a failure or timeout.  This number should be kept small, and in large topologies will require partition of balancer->gear responsibilities.  This is limited by the coverage percentage (the percent of web gears within an app that a given balancer points to)

We define a few variables:

*  M - the number of web load balancer gears in the application
*  N - the number of web gears in the application
*  C - the percent of gears in N that a given load balancer talks to
*  R - the number of web gears that can route traffic through a single node
*  X - the number of gears that must be drained in order to rotate a node out

In practice, a realistic small gear -> load balancer ratio that saturates the limits of a single node might be 15 (R ~ 15).
At small scales, C=100% is not limited by any of the factors above.  The minimum configuration required for availability is 2 web gears, each with the load balancer active.  This is likely to work quite well up to tens of gears.

At larger scales, with C=100% (each balancer talks to all web gears), the last constraint is the most important - all masters have to drain the traffic for a single gear to be rotated out.  The gear limit is expressed (for a given value of X) as:

    N = X * R / C

From the suggested limit above, a cautious X=3 to a somewhat larger X=10 with C=100% practically limits the number of gears to 3*15/100% => 10*15/100% or 45-150 per application.  That upper limit is flexible, but increases the load on other parts of system and impacts the latency of rotate-out requests.  If C is lower, larger numbers of gears can be serviced - C=50% would be 3*15/50% => 10*15/50% or 90-300, C=33% would be 145-450, etc. 

Therefore, the system will be designed with C=100% as the low scale preference, while at higher scale levels effective C must be tested experimentally, and should be a value that users can modify.


#### Auto-scaling multiple load balancers

The web load balancer cartridge is responsible for making decisions about scaling, as the mechanisms for measuring traffic load are specific to the cartridge providing load balancing (in this case HAProxy).  OpenShift will expect the cartridge to invoke the appropriate REST API endpoint using broker key authorization as it does today.  The load balancer cartridge is responsible for ensuring that scale decisions are consistent across multiple web load balancers.

The simplest cartridge implementation is to use the connection hook information provided by the broker to determine a single gear that is responsible for sending scale up and scale down messages.  For instance, the first gear created might enable the daemon for sending scale up events, but later gears might not.  To allow change in the event of a failure, the cartridge might support an application environment variable specifying the gear that should run the scaler.  A user would add an application environment variable, then restart the load balancer gears.  This mechanism could use the load information just for its own gear, or also look at the load information for the other gears. 

A more complicated implementation would attempt to make scale decisions independently of other web load balancer gears, attempting to estimate the traffic the balancer is handling in isolation.  This has the advantage of requiring no coordination, but may result in additional gears being provisioned unnecessarily.  

An alternative implementation would be for the load balancer components to communicate and reach a decision about scale via consensus or via leader election.  The broker may or may not be used to handle election.

In all scenarios, manual scaling can be used to increase capacity temporarily in the event of a failure on a head gear.


#### Data Model Changes

Each gear that is a load balancer (has a cartridge installed and activated that can provide load balancing duties for the web framework cartridge) will be tracked in the broker data model.  A single gear group contains the web framework and load balancer cartridges, and all data related to the scaling of either is stored with that gear group.  The broker will decide whether a new gear should have only the web framework (e.g. PHP) or the web framework and a load balancer (in this example, the HAProxy load balancer cart), and then record that decision on the gear object in the model.

To change to the number of load balancer gears within an application, a user may set either the minimum and maximum scale on the web load balancer cartridge, or rely on a simple ratio provided by the cartridge.  That ratio determines how many web gears should exist for each one load balancer, and is applied when a new gear is created.  A user may be able to set that ratio directly on the load balancer cartridge.  In both cases, the scale factors are modelled with the gear group and are applied through the cartridge REST API.


### New gear operations

In order to properly scale to hundreds of gears, application owners must be able to remove broken gears, manage individual gears, and handle rotation duties.

#### Remove a specific gear

In the event that a node or gear is permanently lost, or a sustained network outage, it should be possible for an application owner to instruct the REST API / platform to remove/scale down a specific gear.  This should include load-balancer gears, and the platform should recognize that that removal requires a replacement.  It should not be possible to remove the last gear or the last node.

#### Restart an individual gear

It should be possible via the REST API to restart an individual gear in order to clear wedged processes.

#### Instruct the web load balancers and/or router to take backends in and out of rotation

*  **Rotate-Out** <gearid> <timeout=2min>

    Remove the requested gear from receiving traffic inside and outside of  the application, on all endpoints except SSH.  This operation is  **graceful** - it instructs the elements within the node to remove the  affected gear from any active traffic in a non-disruptive manner.  This  operation should be persistent until a rotate-in is executed for that gear.
    
    It should be possible to execute this operation asynchronously, without waiting (prereq: scheduler)
   
    The operation must always be provided a timeout, since some clients  will refuse to release their connection.  The default timeout should be  reasonable for HTTP traffic (minutes) and should be configurable per  application (env var)
   
*  **Rotate-In** <gearid>
   
     Allow the requested gear to receive traffic inside and outside of the  application, on all endpoints except SSH.  This operation should be  persistent until a rotate-out is executed for that gear.
   
    The operation is expected to complete deterministically in a short (< 3s?) timeframe, and thus does not need a timeout.

A number of other operations within OpenShift will use these two actions  either individually or in bulk to protect the users of an application  from errors and downtime.  Parallel operation (instructing an  application to rotate-out multiple gears) will be important at scale.

1.  Restarting a gear without downtime requires the following operations today:

    Rotate-out / full stop cartridges / full start cartridges / rotate-in
    
     !! Many web technologies do not require full restarts to pick up new  versions - we should be prepared to allow individual cartridges to make  that decision, by sending a gear level event to restart and let the  cartridges decide whether they need to be rotated out. !!
    
2.  Deploying a new version of code requires the following operations today:

    Rotate-out / full stop cartridges / replace version / run deploy / full start cartridges / rotate-in

     !! Some consumers may have cartridges or applications that can cleanly  reload new code without any of these steps being executed (because they  fully load source into memory) - we should look for opportunities to  enable this
    
3.   If a problematic gear is detected, administrators of an application may  wish to take it out of rotation in order to troubleshoot problems.

     The gear may not be down as seen by HAProxy - instead it may be functioning incorrectly.  Stop / delete may destroy important debugging  info.
    
4.  Administrators of an application may take large sets of gears out of rotation to perform special, non-deployment operations

    This may include driving a deployment outside of OpenShift's control, by targeting discrete operations.


#### Resilience of the application pending operations queue 

The application pending operations queue (used to transactionally manage distributed resources like gears) must be resilient to node failures, and allow certain operations to continue even in the presence of operations that cannot complete at the current time.  The minimal operations are:

*  Scale down a specific gear
*  Scale up a new load balancer gear

It **must** be possible for an administrator to unblock a specific application so it may continue to scale in a deployment of OpenShift.  Application owners **should** have the capability to cancel operations that cannot complete.  The infrastructure and job infrastructure **should** be designed so that operations that must operate on gears that are not available can be bypassed / skipped / delayed correctly.


### Broker operations for multiple web load balancers

Coordination for the router and the web load balancer gears (HAProxy) will be managed by the broker in a series of events managed by the operation queue.  The create HA application flow would be:

* Create app called through the rest api
* Two pending ops are added to the queue.  One to create the app and one to add HA.  The second op is no different than someone adding HA to an existing app.
* Pending ops to create the app are elaborated (same as they are already)
* Application is created as it is today without HA (the app in non form will be accessible at this point)
* Pending ops to add HA are elaborated.  Rough steps are:
  * Add additional HAProxy gear
  * Configure connections with additional HAProxy gear with all existing gears (so identical haproxy.cfg in both haproxy gears)
    * Expose HAProxy servers on external ports will need to be an additional connection for HA apps)
  * Add HAProxy servers of app to the routing layer tables.  Note: The communication between the routing layer and the HAProxy servers should skip the apache proxy on the nodes and go directly to HAProxy on an exposed port.
  * Add add HA CNAME for the app pointing to the routing layer.  Probably something like ha-myapp-mydom.rhcloud.com
  * Notify the routing SPI
* Execute the pending ops (rollbacks and retries at many points are possible.

Remove HA from an application:

* Pending ops to remove HA are elaborated.  Rough steps are:
  * Remove additional HAProxy gear
  * Execute connections
  * Re HAProxy servers of app to the routing layer tables.
  * Remove HA CNAME
  * Notify the routing SPI
* Execute the pending ops (rollbacks and retries at many points are possible.

Git Push/Build/Deployments

* A build / Deployment PEP is being designed that will cover more exotic changes to the build process.
* This PEP only covers the basic use case of pushing to the master HAProxy gear and having it sync to all other gears just as it is today.


Backwards Compatibility
-----------------------

* No backward incompatible API changes are planned.  All APIs and SPIs are new.
* The routers and the routing SPI operate independently of the nodes.
* An existing application can continue to serve traffic to its original DNS entry, and can configure it's aliases to point to the HA layer as necessary.


Rationale
---------

Highly available systems should require multiple independent failures before a service disruption occurs.  Adding multiple HAProxy servers allows a large distributed environment to provide HA for several applications.  By making the router infrastructure pluggable, enterprises can inject their own load balancers into individual applications (for example a redline balancer).  Thus the pluggable balancer system is outside the scope of this document, but considered while implementing the routing SPI.

Having multiple backend haproxy servers provides easier customization of the configs on a per application basis and further assists in application containment to mitigate different applications from impacting the performance of other applications.  At the same time, operations on the existing HAProxy gear (the head gear) are made less special - over time all operations that could span multiple HAProxy masters can be executed in a distributed fashion (build, deploy, restart, rotate-out).

Instead of having a thick or smart routing tier, OpenShift has picked a more traditional route using light weight existing technologies to balance applications across different load balancers.  Users will not be able to ssh in to these servers and won't have access to them just as they don't the system reverse proxy tier today.  An implementor may choose to develop a more complex infrastructure using the routing SPI.
