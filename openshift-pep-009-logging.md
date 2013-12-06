PEP: 009  
Title: Log rotation and aggregation  
Status: draft  
Author: Brenton Leanhardt <bleanhar@redhat.com>  
Arch Priority: medium  
Complexity: 20  
Affected Components: runtime, cartridges   
Affected Teams: Runtime (0-5), UI (0-5), Broker (0-5), Enterprise (0-5)  
User Impact: medium  
Epic: *url to a story, wiki page, or doc*  

Abstract
--------
Standardize application log access and create an OpenShift service to handle
cartridge log rotation.  Develop service and plugin cartridges for log
aggregation.

Motivation
----------
There is a tactical need for log rotation.  Currently developers must tidy
applications to prevent gear disk quotas from being reached--leading to
localized outages.  In some cases OpenShift administrator assistance may be
required to unwedge such a gear.

Secondary to log rotation is aggregation.  Today accessing a gear disk is the
only way to reach cartridge logs.  This was never intended to be the long term
approach for log access.  It does not scale for the developer to have to
manually gather logs from dozens of gears.

In addition, accessing gear logs via ssh requires elevated privileges.  While
the first iteration of the log aggregation service may still require shell
access to the logs, it could be easily extended to expose multiple endpoints
such as HTTP or a websocket drain.  The latter is beneficial in cases where
logs need to be exposed via custom authentication mechanisms.

Lastly, we not only want to alleviate developers from the details of logging
infrastructure we also want to make steps towards standardizing cartridge
logging in the process.  This will pave the way for future partners to provide
additional logging solutions.

Specification
-------------

#### Log rotation service:

1. Must not be the responsibility of cartridge authors or developers

   In order for the platform log rotation service to locate cartridge logs a
standardized location will be added to the cartridge SDK.  This API will update
the log rotation server configuration.

   For efficiency it is possible that idler integration will be required.  As
applications change state the coresponding log rotation configuration can be
enabled or disabled as needed.

1. Must not lose new logs or cause outages

   It can be the case that older logs are purged to make room for new logs.

1. Can be disabled by the developer

   There may be cases where the application server running in a gear wants to
handle log rotation itself.

   For some cartridges this may require rolling restarts.

1. Rotate based on file size and provide a mechanism to automatically delete
  the oldest logs
1. Compress rotated logs
1. Defer to proven Linux tools such as logrotate

  It is advised that a proven open source tool such as logrotate be used and
wrapped with a service script similar to how cgroups is integrated into
OpenShift.

#### Log aggregation service cartridge:

1. Must support aggregating logs to disk as well at provide a drain.  This
drain should be suitable for future cartridges that provide advanced log
indexing.

  A desireable feature of this log aggregation service cartridge is that it's
configuration be editable by the gear owner to allow for maximum flexibility.

1. Must not require more than 1 extra gear per domain for normal usage

  This may require expose_port to be called for domain scoped applications.
However, the recent effort to allow ssl connections to gears may be all that
is necessary.

  A high performance gear profile may be needed due to high IO requirements.

1. May use the platform log rotation service to rotate the aggregated logs

#### Log aggregation plugin cartridge:

1. Must tolerate log aggregation service disruptions
1. Must be embeddable in all gears for an application

  This plugin may require extending the current Group-Overrides functionality to
allow it to be embedded on every gear for a particular application.

  It could be the case that a default plugin could be embedded in all gears.
This default could be overridden by the developer as needed.

1. Must monitor log files and avoid sending duplicate log messages
1. Must be decoupled from the log rotation configuration

  If a developer has disabled platform log rotation for a cartridge the log
aggregation plugin must continue to work provided the cartridge SDK log
specification is followed.

Backwards Compatibility
-----------------------
The ability for application developers to access log files on disk will not be
taken away.  The goal is to provide a superior solution to lure them away from
tradition logging.

This means an unmodified application may not benefit from log rotation or
aggregation out of the box.  The modifications required to benefit from
platform log rotation and aggregation will be trivial--writing to a file in a
particular directory or using a provided named pipe.

Rationale
---------
Several PaaS providers offer integrated logging services.  Applications are
required to log following strict guidelines and by doing so benefit from the
service.

A common concept for these providers is the log drain.  If logs are not drained
they are simply discarded.  The drains for the first iteration of the OpenShift
log aggregation service will be the filesytem as well as something like
websocket that make for easy integration with a log indexing service.  By
having the logs also persisted to disk and rotation the the only logs that
stand the chance of being discarded are old logs.  New logs will always be
stored.

Later iterations may provide other drains.  Implementing this service and
plugin as cartridges allows not only for rapid prototyping but removes many
concerns from the implementation that would greatly increase the effort of
implementation.  OpenShift's secure multitenancy allows the logging service to
build on a platform with proven scaling.  Those features should not be
reimplemented by this solution since they would greatly increase the test
scope.  Perhaps the greatest benefit of the cartridge-based approach is the
ability to collaborate with current prototypes in the OpenShift community.
