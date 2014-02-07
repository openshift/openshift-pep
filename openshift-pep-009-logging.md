PEP: 009  
Title: Standardized conventions for logging, log rotation, & aggregation
Status: draft  
Author: Brenton Leanhardt <bleanhar@redhat.com>, Andy Goldstein <agoldste@redhat.com>, Dan Mace <dmace@redhat.com>, Julian Prokay <jprokay@redhat.com>  
Arch Priority: medium  
Complexity: 20  
Affected Components: broker, runtime, cartridges   
Affected Teams: Runtime (0-5), UI (0-5), Broker (0-5), Enterprise (0-5)  
User Impact: medium  
Epic: *url to a story, wiki page, or doc*  

Abstract
--------
Standardize conventions for logging in OpenShift to make it easier for users and system administrators to access logs in a consistent manner. Begin to provide support for external aggregation of log messages. Provide a means to perform log rotation for OpenShift environments that write log files to disk.

Motivation
----------
Today, log messages related to OpenShift are placed in a variety of different log files. These include broker logs (Apache, Rails application, user actions, usage, MCollective client), node logs (Apache, MCollective, platform), site/console logs (Apache, Rails application), and gear/cartridge logs (varies by cartridge). Administrators wishing to aggregate all the logs in a centralized location such as Splunk or ElasticSearch must configure those tools to process each log file, reading from disk and shipping the log messages off to the remote aggregation server. This can be wasteful of resources, as writing log files to disk that are then immediately read and shipped over the network to a remote aggregation server adds the overhead of additional disk IO and possibly CPU usage that could be avoided by eliminating the disk writes entirely.

Having a large number of log files spread across multiple locations makes logging aggregation difficult. Aggregation would be much simpler if all the log messages had the ability to go through a single logging transport. Syslog is a good choice for this role because it is or can be supported by all of the OpenShift components with minimal effort. Additionally, most Syslog implementations are configurable, meaning that the administrator can choose to write logs to disk, forward logs to an aggregation server, etc.

Today, log files in a gear are generally located in a subdirectory in each cartridge's directory. For example, if a gear has both the Ruby and Postgres cartridges installed, logs for Ruby would be in `$OPENSHIFT_HOMEDIR/ruby/logs` and logs for Postgres would be in `$OPENSHIFT_HOMEDIR/postgresql/logs`. It would be nice if all of these log files could be consolidated to a single location within the gear, such as `$OPENSHIFT_DATA_DIR/logs`.

For application developers, there is also a tactical need for log rotation. Currently, to avoid log files growing so large that they consume all of a gear's disk quota (thus leading to a localized gear outage), developers must execute the `tidy` operation on their applications. The behavior of this operation is cartridge-specific, and often results in log files being deleted entirely, instead of using rotation. In some cases, OpenShift administrator assistance may be required to unwedge such a gear.


Specification
-------------
### Syslog Enablement
Via configuration, a system administrator can instruct OpenShift to log to Syslog instead of to files.

**Platform Components**  
The following platform components may be configured to write log messages to Syslog instead of to the given files:

- Broker
	- `/var/log/openshift/broker/production.log`
	- `/var/log/openshift/broker/usage.log`
	- `/var/log/openshift/broker/user_action.log`
- Node
	- `/var/log/openshift/node/platform.log`
	- `/var/log/openshift/node/platform-trace.log`
- Site
	- `/var/log/openshift/site/production.log`
- Console
	- `/var/log/openshift/console/production.log`
- Frontend Apache (Gear access logging)
  - `/var/log/httpd/openshift_log`

**Gears and Cartridges**  
Gear and cartridge log messages may be directed to Syslog by setting the `GEAR_SYSLOG_ENABLED` configuration key to `true` in `node.conf`.

**Other Components**  
Some log files will not be controlled by the configuration options listed above and instead must be configured independently/elsewhere:

- Broker
	- `/var/log/openshift/broker/ruby193-mcollective-client.log`
	- `/var/log/openshift/broker/httpd/access_log`
	- `/var/log/openshift/broker/httpd/error_log`
- Node
	- `/var/log/ruby193-mcollective.log`
- Site
	- `/var/log/openshift/site/httpd/access_log`
	- `/var/log/openshift/site/httpd/error_log`
- Console
	- `/var/log/openshift/console/httpd/access_log`
	- `/var/log/openshift/console/httpd/error_log`


### Using stdout and stderr for logging
Cartridge and authors should configure their cartridges so all log messages are delivered to stdout instead of to one or more files. Application authors should do the same. OpenShift will capture anything sent to stdout as well as to stderr and ensure it is logged appropriately (assuming cartridge authors implement the cartridge modifications listed below).


### Cartridge modifications
Cartridges will need to be modified so they can take advantage of the unified 
logging approach described here.

When a cartridge is started, the command being executed now needs to have both stdout and stderr redirected to the `openshift-logger` command (see below). The invocation of `openshift-logger` should also specify the program name (e.g. "php" or "web" for the PHP cartridge).


### openshift-logger
The `openshift-logger` executable is responsible for forwarding log messages it receives via stdin to Syslog or to a file, depending on OpenShift's logging configuration. If `GEAR_SYSLOG_ENABLED` is true, gear and cartridge log messages will be delivered to Syslog; otherwise, they will be written to log files.

Callers of `openshift-logger` must specify a program name which will be used to identify the program that generated the log message. The program name will be used both when sending to Syslog and writing to a file. When writing to a file, the program name will be used as the file name. For example, if the program name is "php", the corresponding log file will be created at `$OPENSHIFT_DATA_DIR/logs/php.log`.


### Gear log message attribution
When an administrator has configured OpenShift to send gear log messages to syslog, there needs to be some way to include OpenShift metadata with each message to aid in  the aggregation of related messages. Because the primary processes running for each cartridge are owned by the gear user, we need to be able to include the OpenShift metadata in a trusted manner. This means we can't rely on the gear processes to send the information in the log messages because the information could become tainted.

Fortunately, when communicating with Syslog via the Unix datagram socket, Syslog can look up the UID of the process sending the log message in a trusted manner using SO_PASSCRED. A custom plugin for Rsyslog can be written that uses the trusted UID property to look up the gear UUID, which can then be used to look up the app UUID and possibly other OpenShift metadata. These values will be associated with the message and can be used for filtering and aggregation.


### Node Apache access log changes
Incoming HTTP requests first arrive at Apache running on a node and are then proxied to the appropriate gear to handle the request. The node Apache access should be augmented to include additional OpenShift metadata (app uuid, gear uuid) to aid in correlation/aggregation of access log messages.


### Log rotation
Log rotation of OpenShift platform log files (all log files listed above **excluding** gear and cartridge logs) is not in scope for this PEP. The platform log files are all items that a system administrator can easily configure for log rotation using `logrotate` or some other means for log rotation.

Log rotation of gear and cartridge logs is in scope for this PEP, time permitting.

TODO...

Backwards Compatibility
-----------------------
If Syslog is enabled for the platform, OpenShift system log messages will go to Syslog instead of to files. It will then be up to the OpenShift system administrator to configure log destinations.

If Syslog is enabled for gears, cartridge and application log messages will **only** be delivered to Syslog, meaning they will no longer be written to log files inside each gear. Additionally, `rhc tail` will no longer work because it requires access to the log files on disk.

Otherwise, if Syslog is not enabled for gears, cartridge and application log messages will continue to be written to disk as they are today, but log files all be placed in `$OPENSHIFT_DATA_DIR/logs` instead of spread across multiple cartridge directories.

If the node Apache access log (openshift_log) is not written to a file in its current format, tools such as `oo-last-access` will not be able to function, meaning that gear idling will not work any more, as determining if a gear is idle currently depends on openshift_log.


Rationale
---------
Writing to `stdout` and `stderr` is easy and supported by every language/runtime/framework. By using these mechanisms, cartridge and application authors can easily generate log messages with minimal effort. Other options such as sending log messages to a web server or message bus may require additional services to be written, and there's no guarantee that every language will be able to support the destination service. Because of this, `stdout/stderr` seems the most appropriate option.

For transporting log messages, Syslog is quite ubiquitous, being the logging system of choice for a large number of *nix distributions. Implementations such as Rsyslog are highly configurable, offering a great amount of flexibility for system administrators. The combination of `stdout/stderr` and Syslog is the clear winner in terms of ease of use, language availability, and flexibility.