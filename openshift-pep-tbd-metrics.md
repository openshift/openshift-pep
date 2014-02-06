PEP: TBD  
Title: Metrics collection and aggregation: draft  
Status: draft  
Author: Andy Goldstein <agoldste@redhat.com>, Dan Mace <dmace@redhat.com>,  
Julian Prokay <jprokay@redhat.com>  
Arch Priority: medium  
Complexity: 8, 13, 20, 40, 100 *story points*  
Affected Components: runtime, cartridges, admin_tools, cli  
Affected Teams: Runtime (0-5), UI (0-5), Broker (0-5), Enterprise (0-5)  
User Impact: medium  
Epic: *url to a story, wiki page, or doc*  

Abstract
--------
Create an OpenShift service that allows for an administrator to gather metrics
from cartridges, gears, and nodes. Develop a method to aggregate the metrics
and deliver it to an endpoint.

Motivation
----------
The primary need for a metrics gathering mechanism is to give OpenShift
administrators insight into the performance characteristics of their
applications and the environment in general.

In addition, external monitoring tools can use the metrics for charting
performance and to set up alerting profiles. This functionality can be
invaluable to organizations looking to audit their system or during perofmrance
testing.

Specification
-------------


##### Initial Reasearch


1. Must be the responsiblity of cartridge authors to embed cartridge-specific
metric collection capabilities.

   In order for the metrics collection service to gather metrics, a
standardized location will be added to the cartridge SDK.

   The implementation of these capabilities can follow one of two models: push
vs. pull. A pull model would require a service to run metrics gathering scripts
at some interval and then send the metrics to the drain. For this approach to
work, metrics scripts must be timeboxed and their performance characteristics
must be acceptable to ensure that metrics are gathered from all Gears in a
timely fashion.

   A push model would instead require cartridges/applications to schedule
collections on their own and publish the metrics to a drain. Cartridge authors
and/or application developers would have the burden of creating collectors that
do not adversely affect performance.

   One other way would be to tell application developers to embed metrics
collection within their applications and have these collectors write out to
log files. With a proper identifier, the metrics messages can be distinguished
from log messages in the aggregation portion. This method would probably have
the least impact on performance, but would require a lot of reworking by
application developers.

1. Must include ability to collect metrics that are standard across all Gears.

   All Gears have items, e.g. cgroups, that contain valuable information for
OpenShift administrators. The OpenShift API will provide a way to gather these
metrics.

1. Must not cause outages

1. Can be disabled by an administrator

   There may be cases where an administrator does not want to collect metrics.




1. Must support sending metrics to a drain.

   May use the same drain as logging.

1. Must mark metrics with appropriate identifying information.

   Metrics need to be identified by the UUID of the Gear they came from, the
name of the application, the type of metric, timestamp, etc..


#### Metrics Collection

After discussion of the Metrics Collection feature, and with the implementation of the openshift-logger, it has been decided that we will use a hybrid push/pull model for collection where every metric, regardless of push/pull, will be written to stdout with the determined message format.

1. We will write a daemon(most likely Ruby) that will live on the node and will be responsible for scheduling the pull metrics available per gear. The daemon will also be responsible for reporting meta-metrics, such as time per gear, etc.

1. The OpenShift devs will create metrics scripts that retrieve information universal to all nodes such as cgroup information, gear usage, etc.

1. Cartridge authors will be responsible for writing metrics scripts relating to their cartridge. In their cartridge manifest they will also include a list of all the metrics that are to be run. The top level key will be Metrics, and initially the value will be a Set of metrics. Later, we will allow each metric to be a key, and the value will allow the user to specify additional behavior per metric. The ruby daemon will trigger the running of these scripts through an "oo" command

1. Application developers will have the ability to include metrics scripts in the git repository of their application, and we will expose a hook that the ruby daemon can hit to trigger the running of the scripts.

1. Both cartridges and applications will also be able to use the push method anywhere in their code by writing a message to the openshift-logger at anytime with the correct format


#### Metrics Aggregation
1. Aggregation will be tied into the functionality implemented with openshift-logger, the metrics will be written with a specified format out to the openshift-logger, and the sysadmin will be responsible for configuring rsyslog to route the messages to the appropriate location i.e. graphite server etc.

#### Possible Issues

1. rsyslog can only handle messages ~2kB in size. If we are going to be batching metrics that are pulled, we must guarentee that the service can handle messages with just one metric, and possibly support batched metrics

1. Metrics may take longer to run than the configured metrics interval. We must determine the correct failure behavior. A complete metrics run is more usefull every two intervals than half a metrics run every interval, but sometimes we will not be able to tell if the metric has completed(application metrics)

Backwards Compatibility
-----------------------
With the addition of this functionality, OpenShift administrators will have
more things to manage in their configuration. It will be up to these admins
to attach external tools to the metrics drain.

Depending on how metrics collection is implemented, different groups will be
affected. In both the push and pull model, cartridge authors would be required
to add in metrics collection functionality to their cartridges. The pull model
will also require adding in a service to OpenShift. The last route would
require application developers to log metrics, thus requiring a reworking of
old applications to take advantage of this functionality.


Rationale
---------
Metrics provide important insight into the performance of a system and
environment. These metrics can be used for auditing purposes or to ensure that
services are allocated an appropriate amount of resources. In addition, metrics
can be used by monitoring services to generate alerts and for performance
monitoring.

Metrics will flow to a centralized location where they can be drained by an
external tool or service. Any metrics that are not drained are instead
discarded. The first iteration of the drain will be something simple like
rsyslog which can easily be integrated with a tool like Logstash.

By taking the approach of having cartridge authors integrate metrics collection
capabilities, it ensures that end users don't have to figure out how to gather
metrics for each cartridge in their environment.
