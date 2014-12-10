PEP: 014  
Title: OpenShift 3 SSH to Containers  
Status: draft  
Author: Andy Goldstein <agoldste@redhat.com>  
Arch Priority: medium  
Complexity: 100  
Affected Components: apiserver  
Affected Teams: Runtime, Infrastructure  
User Impact: medium  
Epic: TBD

Abstract
--------
Users and administrators are used to being able to SSH to their servers to perform various tasks. We need to devise a means for users to access cluster-managed containers via SSH.


Motivation
----------
- Provide SSH access to cluster-managed containers


Specification
-------------

### Container identification

In OpenShift, the smallest deployable unit is a pod (a group of 1 or more containers), not a single container. A container belongs to a pod, which belongs to a namespace. To uniquely identify a specific container, you must specify all 3 elements: namespace, pod name, container name.

To support as many SSH clients as possible, we really only have two items we can use when specifying which container is the target of an SSH request: username and hostname. If SSH supported HTTP-style virtual hosts, things would be easy, as we could just use something like `$namespace-$pod-$container.openshift.com` for the hostname, and that would identify the container. Unfortunately, this feature doesn't exist (although it has been discussed [here](https://groups.google.com/forum/#!topic/mailing.unix.openssh-dev/cyf3bhxpc8U)). For smaller environments, if it's possible to assign 1 IP address per namespace/pod/container, hostname could be a viable identification mechanism, but this would require a lot of dynanmic infrastructure changes (DNS updates, sshd configuration updates) on the fly, so it's probably not practical.

That leaves username as the sole means to identify the container. Linux usernames are limited to 32 characters, and it's highly likely that the namespace/pod/container combination will exceed the 32 character limit.

We could potentially use Docker container IDs, although they are 64 characters and look like `718b86bd82e457d802f7077d79ab62e4ff386aaeb4fb1f5d00ec1475c42b7e5f`. A container's ID can reasonably be shortened while still avoiding collisions; Docker shortens them to 12 characters when displaying them in `docker ps` (e.g. `718b86bd82e4`). Docker container IDs are not stored in etcd nor are they available in any of the API server's data types (i.e. pod and its related types don't have the container ID).

When the cluster creates a container, it assigns the container a name such as `k8s_dockerregistry.b1c4fcf9_registry.default.etcd_1417618667_5c7a85e8`, which is clearly more than 32 characters, and not a good candidate for the identifier.

A pod may be deleted from one node and recreated on the same node or on another one depending on various factors (scaling events, node evacuation, etc.). When this happens, the Docker container IDs and Docker names of all the pod's containers change. There are things that remain consistent: the pod's namespace, the pod's name, and the pod's containers' names (as specified in the pod definition; not the Docker container names for them). However, if replication controllers are used, the name of each pod created by a replication controller is a randomly-generated UUID such as `f9bc6e3d-7b15-11e4-a6c4-001c42c44ee1`.

We looked at sending an environment variable (`-o SendEnv ...`) to specify the container, but unfortunately you can't set the value in an entry in the `~/.ssh/config` file; you can only specify that the variable be passed from client to server, using its current value. An example would look something like this:

	CONTAINER=$ns/$pod/$container ssh -o 'SendEnv Container' ssh@ssh.openshift.com

This isn't elegant, and the user experience is suboptimal (or maybe even impossible if an SSH client doesn't support sending evironment variables).

Out of all of the possibilities listed above, the one that seems best is using namespace/pod/container as the username, along with a custom NSS module to overcome the 32 character limit (more on this below).

#### Identifying replication controller managed pods

A pod in a replication controller is assigned a randomly-generated UUID as its name. This makes it difficult to provide a constant SSH URL. Imagine a replication controller named `foo` that defines some pod with a replica count of 3. You might end up with 3 pods named like this:

```
f9bc6e3d-7b15-11e4-a6c4-001c42c44ee1
f9bc9428-7b15-11e4-a6c4-001c42c44ee1
f9bca1c2-7b15-11e4-a6c4-001c42c44ee1
```

The SSH URL for a container in the first pod might be `somenamespace/f9bc6e3d-7b15-11e4-a6c4-001c42c44ee1/apache`, which will remain constant as long as that pod exists. We could try to simplify things and support `namespace/replicationController[index]/container`, which would make the above example `somenamespace/foo.0/apache`, but even then, you can't be certain that you're consistently referring to the same exact pod, in the event that 1 of the members of the replica set is deleted and recreated, or if the order of the pods retrieved when matching the replication controller's label selector changes.

This type of simplification might be useful, but it is not a method that guarantees that a user always gets the same container everytime `namespace/replicationController.index/container` is resolved.

### Proxying SSH

Using namespace/pod/container as the username tells us the container, but it doesn't give any information about the host where the container lives. We can have users SSH directly to the container's node to reach the container, or we could have all SSH connections go through a proxy instead.

Users that get an SSH URL referencing a specific node for the hostname will only be able to use it as long as the pod is on that node. At this time and for the near future, Kubernetes does not move containers across nodes.

If we want to have a constant URL for a container regardless of the host where it's running, we need a proxy with a constant URL. To SSH to a container using a proxy, we'd have a URL such as `namespace/pod/container@ssh.openshift.com`. Even if the container moves to a different node, the SSH URL remains the same. The proxy would be responsible for inspecting the username, determining the container's node, and forwarding the request to that node.

The proxy approach provides a better user experience, but adds complexity to the overall implementation. The client must authenticate with both the proxy's sshd and the node's sshd, but we don't want the client to know about the 2 different sshd servers and ask them to authenticate twice. We could require the client to use an SSH agent and accept the same public keys from the client in both the proxy and node, but that would require SSH agent forwarding, which we shouldn't allow on a multi-tenant server.

As an alternative to agent forwarding, we could have the proxy obtain some form of authentication (token or key pair) from OpenShift as soon as the user authenticates to the proxy. The proxy would then pass this authentication to the node's sshd server.

#### Proxied containers: pods deployed by OpenShift

OpenShift deployments, the common way users get a new version of their code running, are implemented using replication controllers.  When a deployment occurs, the replication controllers in the deployment change.  The replication controllers servicing the old deployments are deleted, and new ones are introduced to service the new deployment.  Because replication controller managed pods receive unpredictable names, the container names associated with an OpenShift deployment will be inconsistent across deployment versions.

In the future, the naming scheme for pods managed by replication controllers may change to become more deterministic.  A possible format might be:

    <replication controller name>-<replica number>

If the naming scheme changed in this way, it would become possible to address the pods associated with an OpenShift deployment in a predictable manner.  If the replication controller name was driven by the deployment name, the container naming scheme might be approximately:

    <namespace>-<deployment config name>-<deployment version>-<replica number>
    <----       replication controller name       ---------->

    # Replica 2 for deployment config frontend version 1 in namespace test
    test-frontend-1-2

    # Replica 8 for deployment config backend version 12 in namespace production
    production-backend-12-8

Even with this scheme, in order to provide stable ssh URLs, Openshift would need a pluggable resolution strategy  to resolve a version independent URL (`test-frontend-2`) to the correct versioned name: `test-frontend-1-2`.  It is worth discussing the merits of a strategy for methods like OpenShift deployments that manipulate replication controllers.

#### Proxied containers: port forwarding, scp, sftp

If the flow with a proxy is client <-> proxy sshd <-> node sshd, is it possible to support port forwarding, scp, and sftp? Of the 3, scp is the easiest, as it simply runs `scp` as a remote command.

Port forwarding and sftp are more challenging. With both of these, the ssh client sends SSH protocol messages to the sshd server, and these messages are not visible to whatever shell or process is executed by sshd upon client connection. Port forwarding is feasible with a flow such as this:

client `ssh -o "ProxyCommand ssh -W %h:%p ssh.openshift.com" namespace/pod/container@node123.openshift.com` -> proxy sshd -> node sshd -> `nsenter -t $containerPid -m -u -i -n -p socat - tcp4-listen:$port,fork`

With `ProxyCommand ssh -W %h:%p ssh.openshift.com`, the client's ssh client establishes a connection and authenticates to ssh.openshift.com, opens a connection to the node's sshd at `namespace/pod/container@node123.openshift.com`, and then forwards all stdin/stdout between the ssh client and the node's sshd. This allows SSH protocol messages to work between the client and node sshd. Without `ProxyCommand ssh -W %h:%p`, the SSH protocol messages would be between the client and the proxy sshd server, which isn't what we want. While this looks feasible, it is not a good user experience and may not be supported by all SSH clients.

SFTP might not be possible with a stock OpenSSH sshd server. For this to work, the SFTP SSH protocol messages would need to be between the client and the node's sshd, just like with port forwarding. While this is feasible with the `ProxyCommand` option described above, that would only get us access to the node's file system. We'd ideally want to `nsenter` the mount namespace of the container to access its files, but that would presumably require modifications to OpenSSH itself, as it is currently responsible for launching `sftp-server` for an incoming SFTP request, and it doesn't appear to be possible to alter that process flow to insert `nsenter` as part of it.

### Identification: User id (uid) & isolation

If we use namespace/pod/container as the username, we need a uid to correspond to the username. Note, this uid is just used for the SSH session; processes in the container do not run as this uid. We could attempt to assign a unique uid to each container, but given that the number of containers created over the cluster's lifetime could potentially be in the millions or billions, this doesn't seem like a viable path. We also need to avoid a collision where 2 SSH sessions to 2 different containers share the same uid.

It is probably best to define a maximum number of containers that a given node may run simultaneously. When a container is scheduled onto a node, it should be assigned a unique uid from the pool of available uids for that node. Whenever a container is removed from a node, its uid is returned to the pool for possible future reassignment.

After a client has been authenticated, sshd drops privileges and switches to run as the authenticated user. When multiple clients connect to sshd, we want to ensure that one client is not allowed to see another client's information. We can achieve isolation by giving each container a unique uid for SSH sessions and by setting a unique SELinux MCS label for the execution context of these processes and their descendents.

OpenShift 2 uses a custom PAM module, [pam_openshift](https://github.com/openshift/origin-server/tree/master/pam_openshift), to set the SELinux context consistently based on the uid when a user interacts with a gear via SSH. OpenShift 3 could continue to use this module.

### Containers without shells

Some container images are designed to be extremely small, comprised of just a single executable. These types of containers don't have a shell that can be used for SSH access. There are a couple of ways we could approach this:

1. Tell users their container must have a shell if they want shell access
2. Automatically add a special utility volume to every pod (bind mount into containers) with helpful tools such as a shell

Supplying a utility volume has some possible pros and cons:

**Pros:**
1. We can potentially add a shell to a container that does not have one

**Cons:**
1. The architecture of the shell in the utility volume might not match that of the container
2. The mount point of the utility volume in the container could potentially conflict with a real directory in the container
3. The name of the utility volume could potentially conflict with a user-supplied volume name for a pod
4. Automatically adding a utility volume to every pod feels hackish

### Where `sshd` runs

There are a few ways we could run sshd in front of a container:

#### `sshd` as a node level service, also used by administrators to access the node

A custom NSS module delegates passwd database information lookups (username, uid, gid, shell, home directory) to OpenShift. A custom PAM module delegates authentication and authorization to OpenShift. Once a client has been authenticated and authorized to access the target container, sshd executes a custom `ForceCommand` that uses `nsenter` (or possibly `docker exec`) to run a shell or the supplied command in the container. Because this command runs as the container's "uid" (from the NSS module and OpenShift), it will need privileged access to `nsenter`/`docker exec`, potentially via a setuid-enabled helper, sudo, or some other mechanism.

Using a single sshd for both administrators and end users eliminates the need to manage an additional sshd service, as nodes will likely already have sshd running. Sharing 1 sshd would mean that the custom NSS and PAM modules would handle lookups for non-container users (i.e. administrators) and fail before other modules such as files/ldap have a chance to perform their lookups, or vice versa, depending on the module lookup order in the NSS and PAM configuration files.

#### `sshd` as a node level service, not used by administrators

This is identical to the previous option, but using a 2nd sshd process and port instead of running a single `sshd`. This sshd would use an independent configuration file from that of the administrators' sshd. This has the advantage of being able to apply a more restrictive set of configuration settings for this sshd. Either this sshd or the one for administrators would need to run on a nonstandard port (or they both could), which makes it more difficult for attack (but far from impossible).

#### `sshd` as a node level service via a pod

This ideally would be a "core" system pod, something that's deployed as part of the cluster itself (a feature that doesn't really exist today), and there would be 1 per node. The container running sshd would need to run privileged, and it would need to share the host's pid namespace if using `nsenter` (support for this is currently pending [here](https://github.com/docker/docker/pull/9339)) or the node's docker socket if using `docker exec` (more R&D is needed to determine if the broad scope of these privileges can be reduced). The port for sshd would need to be accessible from outside the host, either by specifying the host port in the container's specification, or by using a Kubernetes service.

One benefit to this approach is that the custom NSS and PAM modules could be installed only inside the sshd container, leaving the node's NSS/PAM configurations untouched.

One significant detractor is that a privileged container that shares the host's pid namespace is required. This potentially opens up the door to additional attack vectors, as there are additional components involved (Docker, kernel namespaces).

#### `sshd` as an always-on sshd container in a pod

For this option, OpenShift automatically adds an sshd container to every pod in the system. Ideally this container wouldn't need the privileges described in the previous option, but they may be necessary. If so, this is a "con" for this option, as we want to avoid allowing user containers to run with privileges.

This option is also not desirable due to the excessive number of additional containers that will likely sit idle close to 100% of the time for the majority of pods.

#### `sshd` as an on-demand sshd container in a pod

For this, we either need the ability to add a new container to an existing pod, or we need a way to create a new pod and ensure the scheduler places it on the same node as the target pod/container. Either way, this option is essentially the same as the always-on sshd container in a pod. The sshd container still needs to share the host's pid namespace for `nsenter` or it needs the host's docker socket bind mounted in for `docker exec`.

### Authentication
#### Public key authentication
sshd uses a custom `AuthorizedKeysCommand` to ask OpenShift for a list of public keys allowed to access the target container. This command runs as `AuthorizedKeysCommandUser`, which must be an isolated user that has a private means of authenticating with OpenShift (e.g. a client certificate or token). sshd performs the key exchange with the client and allows the client to proceed if the client's key is in the list of authorized keys retrieved from OpenShift.

#### Token (password) authentication
The client presents a token as a password to `sshd`. `sshd` delegates password authentication to PAM. A custom PAM module delegates authentication to OpenShift, and OpenShift validates the presented token.

#### Kerberos authentication
**TODO**


### CONTENT BELOW HERE IS BEING REVISED

### SSH proxy components
The SSH proxy is comprised of the following pieces:

- sshd from OpenSSH
- a custom AuthorizedKeysCommand
- a custom NSS module to delegate user lookups to OpenShift
- a custom PAM module to
	- delegate password authentication to OpenShift
	- securely lookup and store a user identifier in the session's environment
- a custom executable to perform the proxying logic

The SSH proxy is stateless, in as much as multiple proxies can exist behind a load balancer and/or something like round-robin DNS. Doing so eliminates the proxy layer from being a single point of failure.

--

### Basic flow
The SSH proxy accepts incoming requests from remote clients (users), asserts authentication and authorization, and forwards the requests to the appropriate backend cluster resources:

![basic flow](images/pep-014-basic-flow.png)

This is a simplified version of what actually happens, as we need to handle authentication, authorization, and determine to which backend resource the original request should be forwarded.

--

### Client request
Let's look at what would happen when SSHing to a container. First, the client would run a command such as

```
ssh $container@ssh.openshift.com
```

where $container is the ID of the desired container.

![client request](images/pep-014-client.png)

### Proxy actions

`sshd` running in the proxy receives the request and performs the following sequence of steps relevant to OpenShift:

![proxy](images/pep-014-proxy.png)

#### Steps 1 & 2: lookup $container via NSS
`sshd` looks up information about the $container (user) via the `getpwnam` system call. This uses NSS to retrieve the information based on the configuration in `/etc/nsswitch.conf`. For the SSH proxy, this file must be configured to use a custom NSS module that asks OpenShift for this user information.

#### Steps 3 - 8: authentication
##### Public key authentication
`sshd` uses a custom `AuthorizedKeysCommand` to ask OpenShift for a list of public keys allowed to access $container. This command runs as `AuthorizedKeysCommandUser`, which must be an isolated user that has a private means of authenticating with OpenShift (e.g. a client certificate or token). `sshd` performs the key exchange with the client and allows the client to proceed if the client's key is in the list of authorized keys retrieved from OpenShift.

OpenShift also returns a "user reference token" environment variable that specifies the actual user associated with the public key. This variable is used later on as the password when the proxy SSHes to the container's node.

##### Token (password) authentication
The client presents a token as a password to `sshd`. `sshd` delegates password authentication to PAM. PAM is configured to delegate authentication to OpenShift. If authentication succeeds, OpenShift returns the "user reference token" and the PAM module sets it as an environment variable for the session.

##### Kerberos authentication
**TODO**

#### Step 9: execute proxy command
After the user has successfully logged in, `sshd` executes the `ForceCommand` specified in `sshd_config`, a custom executable that provides the proxying logic.

#### Steps 10 & 11: determine container's node
The proxy asks OpenShift on which node the container resides.

#### Step 12: forward request to node
The proxy forwards the original SSH request to the node, using $USER_REF as the password token for authentication.

--

### Node actions
![](images/pep-014-backend.png)

#### Steps 1 & 2: lookup user ($container) via NSS
`sshd` looks up information about the $container (user) via the `getpwnam` system call. This uses NSS to retrieve the information based on the configuration in `/etc/nsswitch.conf`. For the node, this file must be configured to use a custom NSS module that asks OpenShift for this user information.

#### Step 3: authenticate
The SSH proxy presents the user reference token as a password to `sshd`. `sshd` delegates password authentication to PAM. PAM is configured to delegate authentication to OpenShift.

#### Steps 4 & 5: execute original command
`sshd` executes a custom shell that `nsenter`s the target container's namespaces and executes $SSH_ORIGINAL_COMMAND.

--

### MCS label assignment
Distinct users in a multi-tenant environment (SSH proxy container, Git backend container, etc.) should not be allowed to view each others' files. Giving each Linux user its own SELinux context and setting the execution context of a user's processes provides this inter-user isolation.

OpenShift 2 uses a custom PAM module, [pam_openshift](https://github.com/openshift/origin-server/tree/master/pam_openshift), to set the SELinux context when a user interacts with a gear via SSH. OpenShift 3 could continue to use this module, or it might be possible to use `pam_selinux` and the custom dynamic environment variable PAM module described below to set `SELINUX_LEVEL_REQUESTED` above `pam_selinux` in the PAM stack.

### SSSD integration?
Consider using SSSD in the SSH proxy container and the various backend containers (Git, etc.) to cache public keys.
