PEP: 7  
Title: OpenShift Containerization  
Status: reviewed  
Author: Tobias Kunze <tkunze@redhat.com>  
Arch Priority: high  
Complexity: 100
Affected Components: runtime, broker  
Affected Teams: Runtime (0-5), UI (0-5), Broker (0-5), Enterprise (0-5)  
User Impact: none  
Epic: 

# Abstract

The current design of OpenShift containers uses SELinux, CGroups, Pam namespaces, quota to provide isolation. Recent improvements in the Linux kernel provide
additional features like namespacing which can be leveraged to improve the containerization and provide additional security. In addition, the libvirt-sandbox
package has made setting up containers and management of containers easier and can be used with OpenShift.

# Motivation (Background)

Kernel has added a few new namespaces which provide additional isolation for gears:

	1. PID namespace: Provides each gear with its own set of PIDs starting at 1 so the are unable to see processes from other gears.
	2. Network namespace: Provides each gear with its own 127.x.x.x IP address space and allowes adding custom network devices.
	3. IPC namespace: Isolates the queues, semaphors etc. used by a gear from other gears
	4. Mount namespace: Allows for file system mounting that is only visible to the gear.
	5. UTS namespace: Allows the gear to have its own hostname

**Note** There are also user and memory namespaces but those are not enabled in Fedora by default.

The LibVirt-Sandbox project[http://libvirt.org/git/?p=libvirt-sandbox.git] has packaged up these namespaces and provided a convenient way of setting up isolated containers. These containers can also be easily managed through the libvirt API.

It is desirable to introduce LibVirt and Kernel namespace based containerization into OpenShift. For hosters, containerization helps make the economics work by allowing for a higher degree of overcommit. For anyone else running OpenShift, containerization massively simplifies setup and operations by making virtualization optional.

# Goals

The following is a list of goals, in descending order:

1. Achieve a very high **density** of containers. In the order of 1,000 or more active small gears per EC2 m2.4xlarge host, plus approx. 10,000 idle containers.

2. Minimal additional resource **overhead** as compared to current OpenShift isolation

3. Allow host root to **attach** to the container and run commands inside it, both as container root and container user

4. Gears must be able to bind to **localhost**

5. Allow for gears to be moved between hosts with minimal reconfiguration.

6. **Security**

7. **Memory usage** must be controllable in both user and kernel land (separate goal because the respective cgroups controller is not available)

8. Applications should be able to run **contained root commands**

9. Users must to be able to **connect** to their containerized gears

10. Support **patching** of container packages

11. Some form of **virsh compatibility** for ecosystem reasons

12. **Portability** at least between RHT distros and Debian

13. Prepare for shared filesystem access within the gear.

14. Leverage SystemdD capabilities like socket-activation where possible.

# Solution

## Architecture

The proposed solution is based on the git version of LibVirt and libvirt-sandbox and the newest Fedora 19. LibVirt builds on top of cgroups and kernel namespaces to provide the container technology. SELinux is managed through libvirt-sandbox.

**Note:** a Fedora 19 kernel is required for underlying `lxc-attach` support. However, even Fedora 19 does not include the memory cgroups controller (see goal 6 above) and user kernel namespace support.

A wrapper around `lxc-attach`, `virt-login-shell` is available as a LibVirt patch and is being reviewed for merge.

Each container runs a gear and is bootstrapped by a small `oo-gear-init` process, which puts itself to sleep as soon as the container is up and waits for SIGCHILD signals.

## Container Density

With the current container model, each container does not have any extra processes running in it, however the LibVirt model requires an init process to be running to keep the namespace and container alive. In order to maximize density, this init process will need to have a very minimal footprint.

**Proposal:**

	1. Use a init process instead of SystemD based services.
	
	2. The init program will respond to SIGCHILD events in order to clean up processes.
	
	3. The init process will have a very small memory footprint.

## Filesystem

Root is mounted `ro` in a container and the gear dir `/var/lib/openshift/gear/<uuid>` is bind mounted `rw `on `/var/lib/openshift/gear`. As a result, anything that the gear needs to access needs to be outside the `/var/lib/openshift/gear` directory. `/tmp` and `/var/tmp` are tmpfs mounts.

**Proposal:**

    1. `<PREFIX>/gear/<uuid>` bind mounted on `<PREFIX>/gear` (singular)

    2. `<PREFIX>/gear/.httpd.d` and similar hidden due to bind mount

    3. `<PREFIX>/cartridge` (singular, to match gear) visible since not hidden by bind mount

    4. All of `<PREFIX>` can remain one mount

    5. `<PREFIX>` can be `/openshift` for simplicity

    6. Use non-file based config for sssd to hide /etc/passwd,shadow,group files from gear

## Network

The biggest change on the network side is related to switching from localhost gear addresses to routable addresses. If we use a routable address, port_proxy could be dropped and replaced with iptables. 

Currently the broker assign a UID to a gear and the 127.x.x.* ip range and external proxy ports is based on this UID. This makes moving gears between hosts difficult as it requires reconfiguring cartridges (not just connections) if a gear is moved.

**Proposal:**

	1. Each LibVirt container will be assigned a 172.x.x.x NAT'd address based on its UID
	
	2. Within the container the conatiner will have 127.*.*.* network space and cartridges will bind to these addresses.
	
	3. IPTables will be used to port-forward the 127.x.x.x address to the container external 172.x.x.x address.
	
	4. IPTables/FirewallD will be used to port-forward the 172.x.x.x conatiner address to an node external IP.
	
	5. When gears are moved, their internal 127.*.*.* IP space will not change. The 172.x.x.x address and the externally
	   proxied ports will change. Connection hooks will be re-executed to reconnect services.

**Update:** recent Fedora/RHEL kernel changes [mmcgrath: kernel version is unclear] allow localhost to be routed, so the localhost option would be available today.

**Note:** This still ties the network IP to the UID of the gear but since cartridges do not listen on this address, it will not be affected by any changes. 

**Note:** The actual network ip range of the internal network will need to be configurable as some on-prem environments may already be using one of more the private address spaces.

**Note**: Even if 127.x.x.x IPs are not routable, F19 allows for creation of a dummy interface inside the container and assigning it a link-local (169.254.x.x) address. This allows the use of same DNAT iptables rules.

## User SSH access to the container

Current container model uses standard *nix users and does not require any additional support for SSH. With LibVirt, SSH access to the gear will require the container to be started on demand.

**Proposal:**

A wrapper around `lxc-attach`, `virt-login-shell` is available as a LibVirt patch and is being reviewed for merge. `virt-login-shell` is a SUID program which will start the LibVirt container and switch the current process into the container namespace.

## Memory and CPU limits

The current container model directly modified cgroups to monitor and control memory and CPU limits. We need an equivalent with LibVirt.

**Proposal:**

	1. The current resource module will be made pluggable.

	2. A new plugin will be created which will use LibVirt APIs to query CPU usage and set CPU and Memory limits

## Filesystem limits

The current container model uses quotas to limit filesystem usage. With LibVirt containers, this is more complex as the container directory structure does not match up with the host, and thus the container is unable to read the quota DB.

**Proposal:**

Still under investigation.

## Shared Filesystem

In the current container model, each gear is assigned a SELinux MCS label based on its UID. This means that files cannot be shared between gears even if they are part of the same application.

**Proposal:**

	1. All gears that are part of the same application will be assigned a common GID by the broker.

	2. MCS labels will be based on GID, rather than UID.

**Note:** We expect SELinux enabled NFS shares to be available in a few months in the Fedora kernel.

**Note:** A filesystem capable of providing high throughput cluster-wide storage still needs to be investigated.

## Idler

In the current container model, un-idling an application is a complex and multi step process. SystemD socket-activation might provide a solution however, it requires cartridges to understand socket activation and use pre-created sockets which may not be possible.

**Proposal:**

	1. When container is idle, update iptables/firewalld and apache to point to a socket assigned to SystemD.
	
	2. When there is a connection attemp on that socket, SystemD will activate the container.
	
	3. Once container is actve, update iptables/firewalld rules to point to the correct sockets on the container.
	
This will result in the first TCP SYN packet being lost, but the when a connection retry will attempted, it will reach the correct service. It also does not require the cartridge process to know about SystemdD socket activation.

## Design

Containerization is introduced into Origin in a pluggable manner (see new directory tree under `plugins/container`) where everything was baked into the code based on SELinux before.

The new (pluggable) container API shall be compatible with Docker ([http://www.docker.io/](http://www.docker.io/)). Docker is another implementation of LXC which works on Debian systems, without SELinux.

**Note:** we are not planning to implement a plugin for Docker, but if a community member decides they want it, they can write one.

**Note:** Docker relies on AUFS ([http://aufs.sourceforge.net/](http://aufs.sourceforge.net/)), which did not make it into upstream Kernel and will not be supported.

## Miscellaneous Changes

To support the execution goal above, the old exec commands, kernel.exec and utils.oo_spawn have been replaced/wrapped with new methods:

* `run_in_container_context`

* `run_in_container_root_context`

The old network interface methods have been fully implemented to assign routable addresses.

**Note:** Existing architecture relies on uid=gid for all gears. If the node machine had a gid that is allocated to some other user, then the sync between uid/gid breaks and causes gear creation failures. See [bug# 967146](https://bugzilla.redhat.com/show_bug.cgi?id=967146).

**Note:** We don’t need to pre-label IP addresses anymore once IP namespacing is available, so we shouldn’t be limited in how many UIDs and GIDs we can give out.

**Note:** Since the pam folks didn’t like namespace switching, ssh logins have been changed from relying on pam to a new setuid script performing the necessary routing.

# Open Issues

* Disk quotas don’t work because processes inside a container can’t see the host-wide quota db.

* Gear move needs to be flushed out as UIDs may change and require connection hooks to be re-run.

* The memory cgroups controller is not in Fedora. As a result, a container's memory usage can not be fully controlled. As of kernel 3.8.2, the memory cgroups controller appears to be still WIP. Dan Berrange says it should be available in F19, but at least Tobias hasn't been able to get it turned on. /Documentation/cgroups/memory.txt claims it to be only a matter of enabling CONFIG_CGROUP_MEM_RES_CTLR and CONFIG_CGROUP_MEM_RES_CTLR_SWAP. From a Fedora perspective, Justin Forbes says that both MEMCG and MEMCG_SWAP are enabled in Fedora now, but that it is missing HUGETLB. Issue with HUGETLB is that it doesn't support page reclaim, i.e. containers would have to set page limits per application statically before launch and if apps would attempt to use more, they'd simply get a bus error.

* Kernel user namespaces are not yet in any released kernel, albeit there are patches that can be applied to newest kernels beyond the current stable 3.8.2. Dan Walsh suggest to hold off on user namespaces and thinks they are highly experimental. Justin also pointed out that they are incompatible with xfs. The solution is to fix xfs, but I'm not aware of any work in that area. As a result, though, Fedora will not enable it until xfs is fixed.

* Security in general is an issue.

* Package upgrades are not generally supported inside containers, due to a variety of differences from real systems such as bind-mounting of files (that thus can't be replaced), bad interactions between udev and device mounting, security rules, dropped capabilities (Ubuntu), etc. This means that security updates would either require QA upfront to determine if they succeed or the container needs to be shut down.

**Note:** in-container package upgrades or package isolation is out of scope for now.

**Note:** [mmcgrath:] sticking with system-side package upgrades has important benefits w/respect to security and operational efficiency.

# Near-term Requirements for Other Packages

* libvirt-sandbox

    * /tmp should be bind-mounted instead of being a tmpfs so the git work OpenShift performs within /tmp doesn’t end up squeezing RAM.

    * We need a good way to hide /etc/passwd and friends until user namespaces are supported kernel-side.

    * /proc/meminfo has an SELinux issue (bad label); Dan Walsh is looking into it.

	* virt-login-shell and sandbox execute commands need to push the process they spawn into the same cgroup as the container.

# Future Work

Key functionality the container infrastructure should support in the future include gear/application idling and live migration.

Future customer requirements may include

* Exposing an interface to iptables

* Adding a shared filesystem

# Appendix A: Use Cases for Goals

## Terminology

The following terminology is used below:

* "**User**" := user of  an OpenShift application

* "**Customer**" := OpenShift application developer

* "**Cartridge Developer**"

* "**Developer**" := member of OpenShift development team or development community

* "**Operator**" := member of OpenShift ops team

* "**Owner**" := OpenShift installation owner

* "**Application**" := OpenShift application

* "**Cartridge**" := OpenShift cartridge

## 1 High Container Density

Achieve a **density** of 1,000 or more active containers per host, plus approx. 10,000 idle containers.

**Use Case 1.1**

As an operator

I can pack enough containers on a host

So that I make a nice profit

**Test**

Given an hourly cost *C* for a node *N*

And a price *p* per container

When I pack *n* containers on *N*

Then *n* * *p* is *mucho mas* than *C*

**Use Case 1.2**

As an operator

I can pack 10x more idle than active containers on a host

So that I make even more profit

**Test**

Given *x* active containers on a node

When I add 10 * *x* idle containers

Then the node still runs fine

## 2 Low Resource Overhead

Low additional resource **overhead** compared to current isolation.

**Use Case 2.1**

As an operator

I can run roughly the same amount of containerized gears on a node than previously

So that I can migrate to the container model without having to adjust capacity

**Test**

Given *n* gears on a uncontainerized node

When I move them to a containerized node of the same size

Then they run just the same as before

## 3 Attach to Container

Allow host root to **attach** to the container and run commands inside it, both as container root and container user.

**Use Case 3.1**

As a developer

I can write OpenShift code

That can execute commands in a container under the container’s uid

**Test**

Given a container

When my code calls `run_in_container_context(cmd)`

Then `cmd `executes in the container and under the container’s uid

**Use Case 3.2 (nice-to-have)**

As a developer

I can write OpenShift code

That can execute commands in a container as the container’s root user

**Test**

Given a container

When my code calls `run_in_container_root_context(cmd)`

Then `cmd `executes in the container and as the container’s root user

## 4 Ability to Bind to localhost

Gears must be able to bind to **localhost.**

**Use Case 4.1**

As a cartridge developer

I can bind to localhost as necessary

So that I can write cartridges for software that requires this

**Test**

Given a cartridge that binds to localhost

When I deploy and start it

Then t listens on localhost

## 5 Security

The containerization needs to be **secure** in all relevant senses of the word.

**Use Case 5.1**

As a customer 

I want my application to be securely hosted

So that I can sleep at night

**Test 1**

Given a malicious application

When I try to access other applications’ processes

Then my access is blocked

**Test 2**

Given a malicious application

When I try and consume all the CPU on the machine

Then I am limited to my allocation

**Test 3**

Given a malicious application

When I try and consume all the memory on the machine

Then I am limited to my allocation

**Test 4**

Given a malicious application

When I try a fork bomb 

Then I am limited to a fixed process count

**Use Case 5.2**

As a user

I want the application to be secure

So that I can trust it with my data

**Test**

Given a known vulnerability

When I try to exploit it

Then the SELinux policies limit my ability to exploit it

**Use Case 5.3**

As an operator

I want OpenShift to be secure

So that rogue apps can’t compromise my installation or other customers

**Test**

Given an infected cartridge

When it tries to install itself outside of the container

Then the installation fails and is logged

## 6 Memory Control

**Memory usage** must be controllable in both user and kernel land (separate goal because the respective cgroups controller is not available).

**Use Case 6.1**

As a customer

I can not craft code that allocates arbitrary memory in kernel space

So that I can’t DoS the node

**Test**

Given an application

When I attempt to start a fork bomb in a hook

Then it fails as soon as task_structs exceed my allotted kernel memory

## 7 Contained Root Commands

Applications should be able to run **contained root commands**.

**Note:** this need not be comprehensive—running as root within a container and making root changes to the container filesystem may be enough.

**Note:** this can be rephrased as: running as root within a container but not getting system or unrestricted SELinux labels.

**Use Case 7.1**

As a customer

I can become root

So that I can bind to ports < 1024

**Test**

Given an application

When I run a sudo command in a hook

Then that command succeeds

**Use Case 7.2**

As a customer

I can become root

So that I can read my system logs

**Test**

Given an application

When I ssh into my gear

Then I can sudo to read my (container) system logs

## 8 Routing to Containerized Gears

Users and customers must to be able to **connect** to their containerized gears.

**Use Case 8.1**

As a user

I can reach an application

So that I can reach an application

**Test**

Given an application

When I go to its root URL

Then I see the application

**Use Case 8.2**

As a customer

I can supply iptable-like rules

So that I can control network access to my application

**Test**

Given a single-gear application

When I supply a rule to block IP address *A* and redeploy the application

Then traffic from *A* to it is blocked

## 9 Patching

Support **patching** of container packages.

**Use Case 9.1**

As a customer

I can install a debug version of a system library

So that I can see why my application bombs

**Test**

Given an application

When I install a debug library

Then I see debug output in my logs

**Use Case 9.2**

As a customer

I can downgrade an upgraded system library

So that my application continues to run with the library it was certified against

**Test**

Given an application

When I install an older version of a system library

Then it runs with that older version

**Use Case 9.3**

As a cartridge developer 

I can bundle my cartridge with newer versions of system libraries as necessary

So that the software in my cartridge runs

**Test**

Given a cartridge software that requires a newer version of a system library

When I bundle that version with my cartridge

Then the cartridge runs

## 10 virsh Compatibility

Some form of **virsh compatibility** for ecosystem reasons.

[Need help for use cases here.]

## 11 Portability

**Portability** at least between RHT distros and Debian.

**Use Case 11.1**

As an operator

I can install OpenShift on both RHEL and Ubuntu

So that I can offer customers a choice

**Test**

Given a RHEL and Ubuntu image

When I install OpenShift on both

Then I can run binary-dependent applications on their respective distro

And binary-independent applications on both

