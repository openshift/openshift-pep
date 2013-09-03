PEP: 007  
Title: OpenStack Integration  
Status: reviewed  
Author: Chris Alfonso <calfonso@redhat.com>, Luke Meyer <lmeyer@redhat.com>, Clayton Coleman <ccoleman@redhat.com>  
Arch Priority: high  
Complexity: 40  
Affected Components: broker, admin_tools, heat  
Affected Teams: Heat(1-2), Broker (1-3), Runtime(1-3), Enterprise (1-3)  
User Impact:  
Epic: Epic 14: OpenStack Alignment  


# Abstract
Provide several integration points between OpenShift and OpenStack, covering initial installation, capacity based scaling, authentication, and authorization.

* Installation - Installation will be accomplished by pre-creating VM images and creating post-boot configuration.
  * Utilize Disk Image Builder, which is an OpenStack incubation project to create a VM image that has packages pre-installed.
  * Utilize OpenStack's Heat project will be used to create all OpenStack resources.

* Autoscaling - When node or broker hosts are over/under utilized be able to scale up/down the node or broker hosts, respectively.
  * Utilize the Heat API project to communicate scale up/down changes with OpenStack resources.
  * Utilize the Heat LBaaS resource for broker host routing (broker scaling will only done when a separate broker support host is configured).

* AuthN/AuthZ
  * Utilize Keystone for user registration and authentication to gears
  * Utilize Keystone for authentication token creation and validation
  * Utilize Keystone for role creation and assignment
  * Utilize Keystone for group (team) management
  * Utilize Keystone for group and role checking for priviledged actions

# Motivation

OpenShift application scaling currently relies upon node hosts already being provisioned and available for picking up messages from a broker host. What this means is node hosts are online even if they are not needed. This also means once the node hosts are filled up with gears, no more applications can be created until a PaaS operator provisions another node host. When OpenShift is deployed on OpenStack resources, OpenShift should be able to provision and de-provision node hosts without PaaS operator intervention. Although OpenShift brokers aren't typically a point of resource contention, having more than one broker available to take requests provides a level of fault tolerance as well as potential mitigation of resource contention. For this reason, we'll introduce the ability to scale broker hosts using Heat's autoscaling. OpenShift authentication is fairly minimal in its implementation. There is a desire to expand the feature set to handle a fine grained access control using roles, teams(tenants), and groups.


# Specification

## Initial Installation
In order for additional features of OpenStack integration to work, the OpenShift broker hosts need to be instrumented with tooling and configuration that knows how to interact with OpenStack services. The recommended way to prepare a broker host is to use a Heat template to install and configure the host. This method of provisioning is consistent with the method of provisioning used in autoscaling as well. Node hosts will be installed in a similar manner, using Heat templates. The installation methods assume availability of rpm packages and puppet modules. Given that Red Hat Enterprise Linux installation and package updates require system registration and application of subscriptions, when using Red Hat Enterprise Linux it is necessary to use a Heat template that orchestrates the subscription management prior to any attempt to install packages.
* Heat Templates - Each Heat template declares each compute resource and how they are associated with one another to create an infrastructure.
  * **The first example template** should be a single broker host and single node host, with the ability to scale the node hosts.
  * **The second example template** should include the ability to scale brokers which would include a component to monitor broker utilization, bringing brokers in and out of rotation using Heat's load balancing (haproxy in an openstack managed host), and cross-broker node capacity checking (to avoid duplicate scale events). This template should set up a separate broker support node (activemq and mongo).  The job scheduling implementation the brokers use to track scaling events should be used to ensure no duplicate scale up or down events are sent to Heat.

## Autoscaling
The OpenStack Heat project provides AWS CloudFormation template parsing support and as part of that has implemented a way to pre-configure a host. One of the tools that can be laid down on a host is cfn-push-stats as seen [here](https://github.com/openstack/heat-templates/commit/0f99951257f0b9e5233fd636249939f2c9f09ead). All autoscaling with OpenShift requires that OpenShift districts be in use.

Although OpenStack has the ability to automatically scale node hosts without interacting with OpenShift infrastructure, it does not know anything about gear placement. Therefore, the OpenShift broker will handle the monitoring of resource utilization and trigger scale-up and scale-down events for node hosts. The capacity checking done by the broker just evaluate node capacity based upon full districts rather than just total node host capacity. For instance, it's possible for one district to be completely full and another district to have additional capacity. In such a case, the full district requires another node host. The result of the scale-up event needs to be a node host, configured to be added to the full district. Additionally, the capacity checking algorithm for compacting needs to ensure there is adequate resources on destination node hosts for gear moving before a node scale-down event is initiated. There are actually two kinds of capacity to watch - active capacity (basically, node capacity) and total capacity (basically, district capacity -- inactive gears still take up district space). The algorithm for determining where to add/remove nodes/districts is a little tricky (as in, no one has codified it yet - but there is a plan to do this as an admin library for the admin console, should be able to leverage that) and may depend on other parameters, particularly expected idle rate.

We will use OpenStack's ability to automaitically scale broker hosts when the broker is configured to use an externa broker support node (external activemq and mongo database). When the broker is set up in this manner, it is considered stateless and can be externally autoscaled. Heat will take care of the load balancing of traffic between the broker hosts.

* When a node-scale-up event is initiated by the OpenShift broker, based upon it's own capacity checking, the OpenShift broker invokes the scaling script noted above in OpenShiftAutoScaling.yaml. On scale-up, Heat will create a node host. When the configured node host is provisioned and the MCollective service starts, it will become available to pick up messages from the broker. The broker must be able to declare which district the node should be placed in. It's imperitive that when a node is provisioned, the broker must be able to specify a resource_limits.conf to apply to the node host so that the node can have a declared node profile. The broker should be able to track the scale-up event request along with the requested profile and the district it will be placed in. When the node comes online, it should have the the event id available as a fact. This fact will be readable by the broker and can them be used by the broker to add the node to a district and clean up the scaling event record it has been tracking.
* When a scale-down event is initiated by the OpenShift broker, based upon it's own capacity checking, the OpenShift broker will use oo-admin-move to find a node host in the district to move the gears to. The broker will also mark the node host has inactive in the district so that no new gears will be placed on the node host. Once all the gears have been removed, the broker will invoke cfn-push-stats like [this](https://github.com/openstack/heat-templates/commit/0f99951257f0b9e5233fd636249939f2c9f09ead). Currently, moving gears from one node host to another is a serial process that is extremely slow. It might make sense to add mass parallel gear movements as part of this effort.

Heat autoscaling does not currently have the ability to specify which host should be scaled down once the gears are moved off the node host. Heat autoscaling currently uses a LIFO strategy to de-provision hosts. An additive feature is needed to specify which host should be de-provisioned. A parameter should be declared in the scaling policy. When an alarm is set, a host argument should be provided and passed to the autoscaling policy to make sure a specific host is de-provisioned.

There is already a project in openshift-extras that has some of the plumbing to orchestrate the scaling functionality. The design of the node-manager capacity checking and event handler plugin could be pulled into the OpenShift broker infrastructure and installed as part of the OpenShift project. All arguments required for capacity checking and communication with OpenStack should be configuration values in broker.conf. When a scale-up is triggered, the OpenShift broker should be able to request the node host to be configured as part of a specific district. When a scale-down event is triggered, the gears on a particular host would be moved to another node host in the same district.

## AuthN/AuthZ
OpenShift has plans to introduce a more rich authentication and authorization feature set, outlined in the [OpenShift Auth PEP: TODO fix the link when the auth PEP exists](http://github.com/openshift/openshift-pep/openshift-pep-auth.md).
The following table maps OpenShift terms to KeyStone terms

| OpenShift    | [Keystone](http://docs.openstack.org/api/openstack-identity-service/2.0/content/Identity-Service-Concepts-e1362.html) |
| ------------ | ----------------------------------- |
| resource     | service                             |
| user         | user                                |
| team         | tenant                              |
| group        | tenant                              |
| member       | user, tenant (each have role lists) |
| permission   | role.description                    |
| role         | role                                |
| capability   | role                                |


# Future

**Load Balancing**  
Future versions of OpenStack will enable integration with the Neutron LBaaS service. That service is still actively being developed and is not available for integration work at this time. Additionally, OpenShift there is a [High Availability Web Applications PEP](https://github.com/openshift/openshift-pep/blob/master/openshift-pep-005.md) that will enable OpenShift to use eternal load balancers such the Neutron LBaaS service.  Heat currently offers a load balancer implementation using HAProxy, however it is not beneficial to use the Heat based HAProxy load balancer implementation over the current OpenShift based HAProxy routing implementation.

**As A Service Plugin Cartridges**  
Ideas of adding plugin cartridges to applications to implement application integration with external services such as Messaging, Storage, and Load Balancing.

* Swift - swift storage access is typically done via a rest API and has several [language bindings](https://wiki.openstack.org/wiki/SDKs) that can be used natively in an application. It's probably that we may introduce Swift integration for cartridges web framework cartridges written in a language that has associated native bindings. Additionally, we may implement Fuse based access to Swift storage. There are issue to be worked through in for Swift access including packaging of client language bindings, and an selinux policy that allows fuse based filesystem selinux labeling enforcement.

# Backwards Compatibility

No backwards compatible APIs are assumed. This is all net new functionality.


# Rationale

The rationale of this design allows the OpenShift broker to utilize the node-manager plugin based design around scale event handling. This allows specific even handling based upon what type of IaaS OpenShift is targetting. Additionally, the use of Heat based templates to set up the broker and node hosts is an implementation detail and an assumped prerequisite of using an openstack/heat based scaling event handler implementation. However, it does not tie OpenShift to Heat or OpenStack as the orchestration tool or the IaaS provider.
