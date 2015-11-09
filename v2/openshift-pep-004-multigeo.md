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

Abstract
--------
Provide a way for a set of brokers to manage several distinct geographies.  While most people will think of a geo as being in a different country or datacenter, this same technique could be used to provide network level separation between node environments.  For the first version of geo support any individual application will only be allowed to exist in a single geo at a time.  Future versions will likely provide DR, scale and failover support.


Motivation
----------
Several users and the OpenShift Online operations team have requested this feature as a way to provide higher availability and DR functions.  This technique can also be used to provide massive scale architecture to potentially host millions of applications across a single install of OpenShift.

Specification
-------------

Included for version 1:
 * Let admins control the default geo for an application (for when no geo is specified)
 * Let admins plugin their own geo placement algorithm (Ex: based on user's geo).
 * Add non affinity zones (Ex: rack) for higher availability.
 * Add geo(region)/zone assignment to the broker (as metadata to nodes in districts)



Considerations explicitly excluded for a future version:
 * Let the user specify what geo to create an application in
 * Applications located across multiple geo's for HA, DR or scale.
 * Private connectivity between different geos.  If you want to communicate cross geo with this PEP you'll have to do it on a public network (SSL encouraged) or via some other in-house solution.

Implementation Details
----------------------
* The Mongo database will need to store what nodes are in what geos.

Backwards Compatibility
-----------------------
* The only concern with backwards compatibility is making sure we don't force the user to select a geo.  As long as a default is selected when not explicitly specified we should be fine.


Rationale
---------
We've decided to focus on a solid infrastructure and foundation for which we'll build future DR and failover scenarios on top of.  This PEP stops at that foundation to just get support across multiple geographies.  Additionally we know this will solve some requests we've had in enterprise to separate environments better.
