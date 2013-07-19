PEP: 00x  
Title: OpenStack Integration  
Status: draft  
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
Integration with the Keystone service will allow OpenShift to apply a more
feature rich authentication and authorization implemetation than it does today.
Attempts have been made (where possible) to be consistent with OpenStack -
[Keystone Use
Cases](https://wiki.openstack.org/wiki/KeystoneUseCases#Keystone_Use_Cases) -
but some terminology is already in use / has slightly different meanings for
OpenShift.  The following concepts also take some direction from Amazon,
although certain things like policies are too complex to consider in the short
term.  I believe we can evolve access control in a manner that allows us to
step from simple to complex gradually. The [Keystone
architecture](http://docs.openstack.org/developer/keystone/architecture.html)
lends itself fairly well to a 1-1 data model mapping with OpenShift's entities.

### Overview

When a client program connects to the OpenShift REST API, or via SSH to a gear,
there must be an *authentication* step that maps the client session to an
underlying OpenShift user.    After authentication, OpenShift checks that the
user is *authorized* to perform certain actions based on the resources they own
or are granted access to.  Authorization can be restricted by *scoping* an
authentication - an authorization token does this automatically - and only
permissions (known as capabilities in other ACL systems) allowed by one of the
active scopes can be performed. [Keystone calls these Rules and Capabilities](http://docs.openstack.org/developer/keystone/architecture.html#approach-to-authorization-policy).

**Terms used below:**

 * resource   - an OpenShift entity such as Domain or Application
 * user       - an OpenShift entity representing a person that logs into the service
               (in the future, there may be a subclass of a user called an organization)
 * team       - a set of users that can be referenced as a single unit
 * group      - an abstract set of users represented by an external system like LDAP or Kerberos principals
 * member     - a user, team, or group that has been assigned a role on a resource
 * permission - any action that may be taken in the system, 
               "view application", "edit alias", "create application" are all permissions
 * role       - a set of permissions that may be taken on a given resource.
 * capability - a flag or config value on a user that entitles the user to access certain features of OpenShift.

Within each section below are some short term and longer term objectives for
changes that will allow team collaboration as simply as possible, and leave
flexibility for future integration.


### Authentication

#### Broker

OpenShift has a pluggable Authentication Service that allows a 3rd party to
handle the details of taking an incoming HTTP request (or username/password)
and validating it against some backend.  This happens on a per request basis
for every REST API request.  The service is responsible for handing OpenShift a
user unique identifier (user uid) - a value that should not change over the
lifetime of the user that will be used to find the user on subsequent
authentications.  It's important to pick a user uid that does not change - for
instance, email addresses in companies often change when people change their
last names, which can mean that a user is no longer to access their old
account.

Today, the authentication service cannot set default values on the user when
they are created, but we would like to support it.  That would allow
preinitialization of users based on information the authentication service has
access to, such as LDAP groups.  Integrators may also update user capabilities
directly based on data stored in Mongo or an external store like LDAP.

Authentication may also be done via an OAuth2 Bearer token (created as an
authorization on a single OpenShift account).  When an API request is made
using that token, the client is acting as the user who created the
authorization, and has access to do everything that user does within the scopes
allowed.  In the future, full OAuth2 support would be desirable for 3rd party
integrators.

We would like to implement a plugin to create users, associate them with roles, and associate them with tentants using the [Keystone
API](http://docs.openstack.org/trunk/openstack-compute/admin/content/adding-users-tenants-and-roles-with-python-keystoneclient.html).

#### Application Gears (git and ssh)

On the gears, authentication is performed today via SSH public/private keys.  A
user associates a public key with their user account, and then that key is
propagated to all of the individual gears in the applications they have access
to.  

**In the future:**

  * other key types may be associated with the user directly or indirectly (such as Kerberos principals)
  * those other types should be propagated to the gear
  * an integrator should be able to map a custom key type to an action on the node (via a node level plugin) so that other forms of authentication via SSH are possible.  
  * it should be possible for an integrator to disable public key authentication from the REST API, and have the clients respect that.
  * at user creation time or afterwards it should be possible for an integrator to add arbitrary key types to the user (and mark them undeletable or system managed)


### Authorization

Every API request is limited to the permissions the currently authenticated
user is granted.  A user has full access to all resources they "own" - that
means all domains that they created, and all applications created under that
domain.  The "cost" of the resources is assessed to the owner, and the
capabilities of the owner determine the remaining available actions on domains
and applications.

In the short term, OpenShift will allow the owner of a domain to grant other
users "membership" to their domain.  Membership on a domain is the explicit
grant of a *role* to a user, where the role entitles that user to a set of
permissions (simple RBAC model).  Roles are strictly inherited - a higher role
has all of the permissions allowed by a lower role.  The following roles are
suggested as a first pass:

| Role     | Allows                                                                                  |
| -------- | --------------------------------------------------------------------------------------- |
|  read    | Viewing the domain                                                                      |
|          | Viewing the applications within that domain                                             |
|          | Viewing the other members of the domain and their role (limited personal information)   |
|  control | Everything allowed by "read"                                                            |
|          | Starting and stopping applications and cartridges                                       |
|          | Scale cartridges                                                                        |
|          | See threaddumps                                                                         |
|          | Change quota                                                                            |
|  edit    | Everything allowed by "control"                                                         |
|          | Add and remove cartridges                                                               |
|          | Add and remove aliases                                                                  |
|          | Remote read/write SSH access to the gears and the Git repository                        |
|          | Create and delete applications                                                          |
|  manage  | Everything allowed by "edit"                                                            |
|          | Change the namespace of a domain                                                        |
|          | Add more members to a domain                                                            |
|          | Limit the gear sizes allowed to applications within a domain                            |
|          | Limit the number of gears created within a domain                                       |
|          | Delete a domain                                                                         |
**Note:** these roles are simply examples based on existing use cases and more feedback is desired.  SSH access in the future might be split into a separate flag/role (gear role).

The owner cannot be removed from a domain, but managers are allowed to add members with manage access.  A user will be able to view the domains and applications they are members of.  Their keys will only be propagated to applications in domains where they have the "edit" role or higher.

Because domains are containers for applications, it is very valuable for an organizational user in multi-tenant environments to be able to limit what specific domains are entitled to do.  We envision this beginning with gear size limitations on the domain and a loose gear limit, which allows an owner to subdivide a large set of gears into domains that have specific uses and members (dev apps, staging, production).  The gear sizes available to create an application in a  domain are the gear sizes of the domain owner, intersected with the domain's allowed gear sizes.

### Teams and Groups

Longer term, we would like to add the concept of Teams and/or Groups - a set of users (and potentially other groups) that can be added in bulk to a domain as a member with a specific role (defined when the team/group is added).  The difference is that teams are created and maintained within the application, whereas groups would map to an external resource like an LDAP group.  Integrators would be able to synchronize (best performance) or lookup at runtime (faster authorization) the underlying group information when constructing queries.  Roles would not be stored within LDAP.

When a team is added as a member, all of the users within that team would be added as "indirect" members.  A user who was already a member of the resource, and was granted indirect access via a team, and would have the higher of the two roles granted to them.  If their explicit access were removed, they would still have indirect access via their team at the team's role.  The denormalization of the team would allow very high performance access control queries (index lookups) which is an important characteristic of the system.  We expect a roughly long tail distribution of application membership, but even with very large fanout the mongoid index will be tractable.  Teams would be limited to a certain installation wide membership limit (100?) which would address the worst of the problem.  LDAP integration would either only materialize existing users of the system, or be opt in.

### User Privacy

In highly multitenant environments access control needs to be cognizant of user privacy.  User A should be able to share a resource with user B without being able to catalog and list all of the users of OpenShift.  To that end, an invitation model would allow a user to send an email or copy a URL that will allow another user to accept the invitation (one time).  Users are implicitly able to view other users that they share membership on resources with - so subsequent grants require no invitation.  This is not a significant limitation in many enterprise deployments and so there should be a config flag to allow global invite.


### Scopes

Every REST API request to OpenShift occurs within one or more *scopes*.  By default, when a user authenticates directly via BASIC auth, Kerberos, or some other direct authentication, they will have the "session" scope which allows any action under their account.  However, the user may also create an authorization token which is granted a specific set of scopes, and when using that authorization token to access the API only actions allowed by those scopes can be performed (this includes viewing items as well as updating items).  A token with no scopes would allow no actions, and a token with multiple scopes allows the union of the permissions each scope grants.  Scopes do not allow you to perform any operation you cannot already perform, they merely limit the actions an API action could take.  Scopes allow another client to act on your behalf, but with a limited set of permissions.

**The following scopes are either implemented or planned:**

  * userinfo             - retrieve basic info about the user
  * read                 - view all the resources the user owns (except for authorization tokens), but no changes or updates are permitted
  * application/:id/read   - read access to a single application, and read access to the domain that contains the application (but not other applications within the domain)
  * application/:id/scale  - same as above, but with the ability to execute scale up / scale down events and set the scale multiplier
  * application/:id/manage - grants access to perform all operations on a single app, and to have read access on the domain that contains the application
  * domain/:id/read      - read access to a domain, and all of the applications within that domain
  * domain/:id/edit      - all permissions on the domain and its applications that a member with the edit role on the domain could perform
  * domain/:id/manage    - all permissions on the domain and its applications that a member with the manage role on the domain could perform
  * session              - any operation can be performed as the user

Each scope has a timeout - we envision more limited scopes having much longer timeouts.

**Scopes are implemented with three mechanisms:**

  * Reject access to an incoming web request by method and controller (read, userinfo)
  * Limit the scope of queries made against the database by type (application/domain scopes)
  * Authorize individual permissions against a known resource

By default, all code is organized so that permission is denied unless an implementer takes a specific action to enable a permission.  


### Changes to REST API

This is very early, but we would want to change a few core resources to map to "the things I have access to" including:

  * LIST_DOMAINS      /domains      - the domains I have access to
  *  LIST_APPLICATIONS /applications - the applications I have access to

We would introduce a new relation

  * LIST_DOMAINS_BY_OWNER /domains  - a new required parameter "owner" would be the user identifier or login that owns the domain

Older API versions would return:

  * LIST_DOMAINS /domains?owner= self

...so that old clients don't become confused.  To bridge the gap with old clients,
the sort order for the returned domains should display the domains sorted first
by whether it is owned by the current user, and then by creation date.  This
ensures that old clients continue to see the owner's first domain at the top of
the list until they are ready to switch.

We want to expose membership, roles, and ownership on each resource that can
have membership (domains and applications).  For applications, the membership
list at first will simply be a mirror of the domain and be unalterable.  It
should be easy to see who else has access to your resource, to edit that list,
and to find members.  A provisional REST API list would be a sub resource of
both Applications and Domains as:

    members: [
    {
    'id' => <unique id of the member>,
    'type' => <type of the member> [user|group|team],
    'role' => <the role the member has> [read|control|edit|manage],
    'owner' => <true if this member owns this resource>,
    'from' => <an array of the different ways this user was granted access>,
    ... TBD
    },
    ...
    ]

Membership changes would be POST/PUT/DELETE operations on a sub resource of
both domain and application, and also able to be set during creation as well.
Other things, like invitations, team changes, and user searches need some more
thought still.

When a user does not have view access to a particular resource, the response
will always be 404 (to hide the existence of the object).  When a request is
denied for authorization, role, or scope, the HTTP 403 response would be
returned along with a message about why the request was rejected.  We will
likely need to distinguish between various type of role failures for clients to
be able to display appropriate messages.

[More discusion on AuthN/AuthZ](http://lists.openshift.redhat.com/openshift-archives/dev/2013-July/msg00196.html)
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
