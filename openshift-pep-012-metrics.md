PEP: 012  
Title: Metrics collection and aggregation  
Status: draft  
Author: Andy Goldstein <agoldste@redhat.com>, Dan Mace <dmace@redhat.com>,  
Julian Prokay <jprokay@redhat.com>, Teddy Funger <tfunger@redhat.com>  
Arch Priority: medium  
Complexity:  43 *story points*  
Affected Components: cartridges, node  
Affected Teams: Cartridge (30), Node (13)  
User Impact: low  
Epic:    
   
 
Abstract
--------
Create an OpenShift service that allows for an administrator to gather metrics from applications, including information about gears and cartridges. Provide a means for metrics aggregation with configurable backend destinations.


Motivation
----------
Metrics provide important insight into the performance of a system and environment. These metrics can be used for auditing purposes or to ensure that services are allocated an appropriate amount of resources (performance tuning). In addition, metrics can be used by monitoring services to generate alerts and for performance monitoring.

OpenShift is a complex system comprised of many independent components that, when combined, make up one or more user applications. System administrators and application owners need a complete picture of the technical inner workings and statistics of all of these components as a whole to have a complete picture of how the application functions as a unit. The following is an example of the types of information that chould be aggregated and correlated to provide that holistic view:

- Gear metrics
	- cgroups data
	- disk quota
	- active process information
	- memory usage
	- I/O statistics
- Cartridge metrics
	- HAProxy statistics
	- Apache statistics
	- JVM data (thread count, heap memoory information)
- Applicaition metrics
	- specific to each application

Specification
-------------
### Metrics Collection
Metrics will be collected using a hybrid push/pull model, with both methods being supported options. In some cases, there may be too much overhead (time and/or resources) to spawn an external process to gather metrics, while in others it may be entirely acceptable.

In both cases, the act of reporting metrics to OpenShift is as simple as writing them to `STDOUT` using the metrics message format defined below. If cartridge and application developers write their metrics to `STDOUT`, those metrics will be captured by OpenShift and either written to Syslog or to log files in $OPENSHIFT_DATA_DIR/logs, depending on the OpenShift environment's configuration (see the [Logging PEP](https://github.com/openshift/openshift-pep/blob/master/openshift-pep-009-logging.md) for more details).

#### Pulling metrics
A node-level service, `oo-gather-metrics`, is responsible for scheduling the pull-based metrics available per gear. The daemon will also be responsible for reporting statistics about the metrics gathering process, such as the time taken to gather metrics.

`oo-gather-metrics` will query each gear for metrics at a configurable interval. During each iteration, it will perform the following tasks:

- collect metrics common to all gears such as cgroups information
- execute `bin/control metrics` for each cartridge whose manifest indicates the cartridge supports metrics
- execute the application's `metrics` action hook (if preset) to allow the application to report its metrics

Metrics reported via `bin/control metrics` and the `metrics` action hook must be printed to `STDOUT`, as OpenShift will handle delivering the metrics to the appropriate destination (Syslog or log files).

Because an OpenShift node may have dozens or hundreds of gears, it is imperative that metrics collection complete as quickly as possible, using as few resources as possible so as to minimize the potential impact to the normal execution of each gear's processes. Cartridge and application metrics may be time limited, and metrics gathering processes that exceed their allocated time may be terminated.


#### Pushing metrics
A cartridge or application author can opt to schedule metrics reporting on their own, instead of relying on `oo-gather-metrics` to do so. 

Metrics must be printed to `STDOUT` and OpenShift will handle delivering them to the appropriate destination (Syslog or log files).


### Cartridge manifest changes
A cartridge author must include a `Metrics` entry in the cartridge's `manifest.yml` to inform OpenShift that it supports metrics. Initially, the `Metrics` entry will look something like this:

	Metrics:
	- thread.count
	- thread.active
	- heap.permgen.size

where each element specifies the name of a metric that the cartridge will be reporting.

Ulitmately, the structure may evolve to include additional metadata about the metrics, such as:

	Metrics:
		thread.count:
			graph_type: line
			display_name: Thread Count

Additional tooling in the future may be able to take advantage of the metadata in the manifest to automatically display metrics in the appropriate format.


### Message format
A metrics message must include the following fields (order does not matter):

- `type=metric`
- `<metric name>=<metric value>`

For example:

	type=metric thread.count=5

OpenShift may support multiple metrics per line (this is still TBD), such as:

	type=metric thread.count=5 thread.active=2 heap.permgen.size=25000000


### Metrics Aggregation
Assuming OpenShift is configured to send gear log messages to Syslog, all metrics for all gears on a given node will be processed by that node's Syslog service. It will be up to the system administrator to configure Syslog appropriately to forward metrics messages to the appropriate service(s) (such as graphite, statsd, elasticsearch).


Backwards Compatibility
-----------------------
This PEP specifies the introduction of new features only; it does not affect any existing functionality.


Rationale
---------
The Logging PEP specifies a solid foundation for transporting all log messages from a gear (including all cartridge and application messages) to a single destination: Syslog (by way of `STDOUT`). It makes sense to use the same transport for metrics because writing `STDOUT` is simple and supported by every language. Additionally, Syslog implementations are typically very flexible and configurable, so it should be easy to direct metrics messages to a system that is equipped to process them (such as Graphite).