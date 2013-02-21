PEP: 004
Title: OpenShift Multi-Geo Support
Status: draft
Author: Mike McGrath <mmcgrath@redhat.com>
Arch Priority: High
Complexity: 30
Affected Components: *api, runtime, broker, admin_tools, cli* 
Affected Teams: Runtime (25%), UI (25%), Broker (50%), Enterprise (N/A)
User Impact: Medium
Epic: TBD

Outstanding Questions
---------------------

Abstract
--------
Provide a way for a set of brokers to manage several distinct geographies.  While most people will think of a geo as being in a different country or datacenter, this same technique could be used to provide network level separation between node environments.  For the first version of geo support any individual application will only be allowed to exist in a single geo at a time.  Future versions will likely provide DR, scale and failover support.


Motivation
----------
Several users and the OpenShift Online operations team have requested this feature as a way to provide higher availability and DR functions.  This technique can also be used to provide massive scale architecture to potentially host millions of applications across a single install of OpenShift.

Specification
-------------

The Multi-Geo specification takes our existing single geo and multiplies it starting at the mcollective layer.  To better understand lets review the current single-geo installation.

Client ->
  via API ->
     Broker ->
       mcollective client ->
         via active mq ->
           mcollective server ->
             Node farm

The broker and mcollective client are all part of what is considered the broker. ActiveMQ is a messaging bus typically on a dedicated host and both the broker and mcollective server connect to the ActiveMQ ports to communicate with each other.  

To add additional geo's the goal is to create a dedicated ActiveMQ server for each geo.  All of the nodes in that geo will contact their local ActiveMQ server and only that ActiveMQ server.  The broker will need to know about every Geo by way of it's ActiveMQ server.  This is basically just creating a new /etc/mcollective/client.cfg for every geo and making sure the Broker knows how to find the activemq server for that geo..

Remaining R&D work is still required to determine if the best approach is with subcollectives or with dedicated mcollective installs.  The work-flow will be the same but how the broker interacts with the nodes will be different.  This is just a note that one of those methods needs to be chosen but some research needs to be done first.  Our general opinion is that subcollectives will require less code to get done, but we're not actually 100% sure that it'll actually work for this use case.

Considerations included for version 1:
 * Let the user specify what geo to create an application in
 * Let admins control the default geo for an application (for when no geo is specified)
 * Add geo naming abilities to the broker (listed below in implementation)
 * Individual namespace for different geo's is likely but more research needs to be done here particularly around large scaling.  This would be like name-domain.us1.rhcloud.com and name2-domain.uk4.rhcloud.com.  The question is whether DNS can handle 100 million entries in a single domain just as well as it can handle 1 million entries in 100 subdomains.


Considerations explicitly excluded for a future version:
 * Applications located across multiple geo's for HA, DR or scale.
 * Migration of applications or gears from one geo to another
 * Private connectivity between different geo's.  If you want to communicate cross geo with this PEP you'll have to do it on a public network (SSL encouraged) or via some other in-house solution.

Implementation Details
----------------------
* If we do not choose subcollectives: Every geo will have a /etc/mcollective/client.cfg file called $GEO.cfg.  So, for example if we have a us1 and a au1 geo, we would have a us1.cfg and an au1.cfg file
* If we do choose subcollectives: Every geo will have it's own filter id and mcollective will communicate directly with that geo via it's id.
* Geo's should be added to the broker.conf file as a GEO_LIST variable similar to how the mongo brokers are listed.  The geo's listed in that variable match to a $GEO.cfg as listed above.
* Admins can specify the default geo via the DEFAULT_GEO variable in broker.conf
* Users, via the command line tooling and UI could specify what GEO to use via the broker API.  If no GEO was specified the admins could decide what GEO to use.
* Once a GEO is specified, that geo's mcollective configs will be used just as normal.
* The Mongo database will need to store what districts are in what geo's.
* It should be assumed that no node hostnames will be the same, even across geo's. Best practice would be to name them ex-std-node1.prod.us1.rhcloud.com and ex-std-node1.prod.us2.rhcloud.com
* Access to a geo will be restricted at the broker by capabilities

Backwards Compatibility
-----------------------
* The only concern with backwards compatibility is making sure we don't force the user to select a geo.  As long as a default is selected when not explicitly specified we should be fine.


Rationale
---------
We believe a completely disconnected messaging bus is the best way to ensure availability during outage scenarios as well as high scaling.  This model seems to make Mongo the bottleneck as the broker and node layers should be able to scale perfectly as more capacity is needed.

We've decided to focus on a solid infrastructure and foundation for which we'll build future DR and failover scenarios on top of.  This PEP stops at that foundation to just get support across multiple geographies.  Additionally we know this will solve some requests we've had in enterprise to separate environments better.
