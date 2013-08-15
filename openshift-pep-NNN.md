PEP: _TBD_  
Title: OpenShift Origin Installation Tool  
Status: draft  
Author: N. Harrison Ripps <hripps@redhat.com>  
Arch Priority: _TBD_  
Complexity: 40  
Affected Components: runtime, cartridges, broker, admin_tools  
Affected Teams: UI (1), Origin (1)  
User Impact: high  
Epic: [Origin v3 Installation Wizard](https://trello.com/c/H01J2eo4/30-origin-v3-installation-wizard-design)

## Abstract
This document describes an improved installation system for OpenShift Origin. The proposed system will use a CLI-based wizard and Puppet scripts to deploy OpenShift onto one or more hosts. The all-in-one VM that is currently used to deliver an all-in-one OpenShift Origin instance will be enhanced to configure itself, via the wizard and scripts, to selectively serve different roles (broker, node, etc.) in a multi-instance deployment.

## Motivation
This PEP is the continuation of an effort that began with a [call to the OpenShift Origin Community](https://www.openshift.com/blogs/help-us-fix-the-openshift-origin-unboxing-experience) to help improve the system's installation process. Around one hundred community members [participated in a survey](https://www.openshift.com/blogs/survey-results-the-openshift-origin-unboxing-experience) to help us understand what their most important use cases were, and which installation tools would be most valuable to them. Based on the survey results, it seems clear that the all-in-one virtual machine where OpenShift Origin is _showcased_ can also become a platform from which Origin can be _deployed_.

## Specification
In broad terms, the design proposal for this installation tool is as follows:

* The Origin VM will continue to function as an all-in-one PaaS
* Additionally, the VM will have a text-based wizard that will assist a user with the following deployment options:
    * "In Place" Role assignment - the VM can take on a specific role in an Origin deployment (broker, node or messaging server)
    * Puppet deployment - the VM can connect to a bare Fedora/RHEL/CentOS instance via SSH and configure it for an Origin role 
	* Puppet script templates - Users can also copy the puppet scripts from the VM to modify and run on their own.

<h3 id="text-based-wizard">The Text-Based Wizard</h3>

The Origin VM completes its boot-up procedure by invoking [oo-login#login()](https://github.com/openshift/puppet-openshift_origin/blob/master/templates/custom_shell/oo-login). This is the method that presents the user with some summary information about the running system:

    OpenShift Origin
    ----------------
    
    OpenShift console: https://broker.openshift.local/console
    Console login:     admin
    Console password:  admin
    
    IP address: X.X.X.X
    SSH user:   openshift
    Password:   openshift
    
    Press any key for root console

This utility will be extended to provide the additional user interaction that is described in this document. It may make sense to rename the utility to `oo-wizard` once this additional development work has been done. Throughout this document, the expanded utility will be referred to as "oo-wizard" to differentiate the proposed functionality from the current.

The new wizard will begin with an intro screen that summarizes the user's options:

    OpenShift Origin
    ----------------
    
    Welcome to OpenShift.
    
    This VM is currently running a complete OpenShift installation.
    You can connect to it by pointing your browser to:
    
    https://broker.openshift.local/console
    
    To do more with this VM, select from the following options:
    
    <1> Use this VM in a multi-instance deployment
    <2> Install OpenShift on another system
    <3> Download Puppet templates
    <4> See login information for this Origin VM
    <5> Exit the wizard

Each choice leads to a follow-on screen, defined below.

<h4 id="oo-wizard-cfg">Behind the Scenes: Recording the System Configuration</h4>

When invoked, the oo-wizard utility will look for a configuration file at `~/.openshift/oo-wizard-cfg.yml`. Users can manually specify a config file location by passing an argument to the oo-wizard. As users work with the utility, general information about the Origin system and specific information about the configuration choices the user is making will be recorded here. The Origin VM will be shipped with a default configuration file that describes the all-in-one Origin system running on the VM itself.

The [puppet scripts](#roles-driven-puppet-scripts) that drive the actual system configurations will, in turn, read from the configuration file using [hiera](http://docs.puppetlabs.com/hiera/1/puppet.html). The configuration file will be organized in deference to hiera's [data format requirements](http://docs.puppetlabs.com/hiera/1/data_sources.html#data-format), but with respect to supporting other installation models in the future, hiera's [hierarchies](http://docs.puppetlabs.com/hiera/1/hierarchy.html) and [variable interpolation](http://docs.puppetlabs.com/hiera/1/variables.html) will be avoided in favor of a single, comprehensive file.

<h4 id="multi-instance-deployment">Workflow: Using the VM in a Multi-Instance Deployment</h4>

The goal of this installation option is to make it possible for a user to set up an entire distributed, multi-instance OpenShift system by running multiple pre-built OpenShift VMs. Normally each VM would run its own complete system, but this installation path turns off services and configures the remaining services to interact with other servers. Those other servers can be other Origin VM instances, or any host that is running a portion of the OpenShift system.

    OpenShift Origin: Multi-Instance Deployment
    -------------------------------------------
    
    What role should this VM fill in the Origin system?    
    <1> Broker
    <2> Node
    <3> Message queueing server
    
    <esc> - Go to main menu

Once a user selects the role for this VM, the wizard will ask relevant configuration questions.

- - -

**NOTE**: These roles are defined in more technical detail in the section on [Roles-Driven Puppet Scripts](#roles-driven-puppet-scripts). The specific configuration questions to be asked will be driven by the requirements of those scripts, as well.

- - -

During the configuration process, value tests will be performed immediately wherever possible. For instance, IP addresses will be pinged and remote services will be connected to. In any instance where remote targets are unreachable, the utility will notify the user but will not _force_ the user to change the value.

    Messaging queue server 10.10.0.37 could not be reached.
    Proceed anyway? <Y|N>
    
    <esc> - Go to main menu

As the user makes choices, the provided values will be written to [oo-wizard-cfg.yml](#oo-wizard-cfg). Functionally this will serve two purposes:

1. If the user reruns the utility, the responses from the file will be suggested as default values
2. The file can be copied to another VM instance to provide reference values to a system with a different role

-  - -

**To expand on point #2**: If a user configures their first OpenShift instance to be a Broker, and then copies their configuration file to a new instance, which will be a Node, then the copied file will be able to provide the Node with the Broker instance public IP address.

- - -

Once the configuration is complete, the user will be presented with a confirmation screen that summarizes their settings.

    OpenShift Origin: Multi-Instance Deployment
    -------------------------------------------
    
    Confirm these settings to finalize the configuration.    
    
    System role:     Role name
    <Config item>:   <Config value>
    ...
    
    <Y> - Confirm settings
    <N> - Re-edit settings
    <esc> - Go to main menu

Upon confirmation, the oo-login script will trigger the reconfiguration of the running instance.

    OpenShift Origin: Multi-Instance Deployment
    -------------------------------------------
    
    <role_name> setup complete! To setup other components of the system, copy
    
        ~/.openshift/oo-wizard-cfg.yml
    
    to the next participating host and rerun oo-wizard there.
    
    <esc> - Go to main menu
    <X> - Exit to the bash shell

Once a system has been configured as part of a multi-instance deployment, the main menu of the utility will appear slightly different:

    OpenShift Origin
    ----------------
    
    Welcome to OpenShift.
    
    This VM is currently performing as a <role_name> instance in a
    multi-instance deployment. You can connect to <it/this system's broker> by
    pointing your browser to:
    
    https://<broker_ip_address>/console
    
    To do more with this VM, select from the following options:
    
    <1> Install OpenShift on another system
    <2> Download Puppet templates
    <X> Exit the wizard

This behavior is triggered by status information from `~/.openshift/oo-wizard-cfg.yml`.

- - -

**Limitations:** Currently, oo-login is only provided with the Origin VM. It is not included with the Origin RPMs. Therefore the ability to transfer and reuse the wizard configuration file is limited to instances of the Origin VM. However - as described in the next section, a single VM instance can also be used to configure arbitrary remote Fedora hosts to perform the various Origin roles. This instance can rely on its single copy of the config file to keep state for the remote systems that are being set up.

- - -

<h4 id="remote-system-deployment">Workflow: Installing OpenShift on Remote Systems</h4>

This workflow is almost identical to the [Multi-Instance Deployment](#multi-instance-deployment) with the notable exception that the configuration work must be performed on a remote system. The concept of instance roles will be reused here, which means that the same set of Puppet scripts can be maintained for both workflows.

    OpenShift Origin: Remote System Setup
    -------------------------------------
    
    What is the hostname or IP address of the target system? []:
    What login should the wizard use? (Must have root access on <remote_host>) []:
    What password should the wizard use? :
    
    <return> - Continue
    <esc> - Go to main menu

When the remote host name/address and login credentials are provided, the wizard will attempt to connect to the remote system via SSH and confirm that:

* The login user has root access
* The system has `yum`
* The necessary RPMs for a complete OpenShift system are available through a `yum search`

Provided the remote system passes these checks, the utility enters the workflow described in the [Multi-Instance Deployment](#multi-instance-deployment), updating the [wizard config file](#oo-wizard-cfg) as indicated.

#### Workflow: Download Puppet Templates
This workflow exists simply to call attention to the various ways that users can gain access to the Puppet templates that are used by the installation wizard.

    OpenShift Origin: Puppet Templates Download
    -------------------------------------------
    
    The templates that are packaged with this Origin VM can be downloaded by
    pointing your web browser at:
    
        https://broker.openshift.local/puppet_templates.zip
    
    The latest versions of the puppet templates are always available on GitHub:
    
        https://github.com/openshift/puppet-openshift_origin
    
    <esc> - Go to main menu

The only technical requirement is that the Origin VM packager will need to be instrumented to create and place the zip file referenced in the wizard.

#### Workflow: Display VM Login Info
This workflow simply displays the login information that the `oo-login` utility displayed by default.

    OpenShift Origin: Login Details
    -------------------------------
    
    To connect to this Origin VM, use the following information:
    
    OpenShift console: https://broker.openshift.local/console
    Console login:     admin
    Console password:  admin
    
    IP address: X.X.X.X
    SSH user:   openshift
    Password:   openshift

- - -

**NOTE**: This screen is not available to users after the current VM has been made a part of a [Multi-Instance Deployment](#multi-instance-deployment). This is because the multi-instance deployment requires the user to change the passwords on the VM to increase security.

- - -

<h3 id="roles-driven-puppet-scripts">Roles-Driven Puppet Scripts</h3>

The new installation options will require the development of roles-driven puppet scripts. From an engineering standpoint, this means that the [current Puppet scripts](https://github.com/openshift/puppet-openshift_origin) will need refactoring. Some points to consider here:

* Our [remote deployment](#remote-system-deployment) option should work for target systems that are running Fedora or RHEL/CentOS. In order to get the necessary Ruby packages for the latter case, the scripts will need to make use of Software Collections ([SCL](http://developerblog.redhat.com/2013/01/28/software-collections-on-red-hat-enterprise-linux/)).

* As mentioned in the section on the [wizard configuration file](#oo-wizard-cfg), the revised puppet scripts will use [hiera](http://docs.puppetlabs.com/hiera/1/index.html) to get at the configuration information in `oo-wizard-cfg.yml`.

But first and foremost, the prerequisite to this work is to agree upon the definition of "reference configurations" that the puppet scripts will build.

#### Pre-Determined Reference Configurations
As implied by the installation options described in the [Text-Based Wizard](#text-based-wizard) section, the puppet scripts will enforce the use of particular services within the Origin system. For instance: while any [AMQP](http://www.amqp.org/)-compliant messaging server will work with OpenShift, the puppet scripts will use [ActiveMQ](http://activemq.apache.org/). The complete breakdown of roles and software packages will be as follows:

* Role: Broker
    * Broker RPM
    * MongoDB
    * MCollective
* Role: Node
    * Node RPM
    * MCollective
* Role: Message queueing server
    * ActiveMQ

The major design goals of the roles-driven Puppet scripts will be:

1. To support both the [Multi-Instance Deployment](#multi-instance-deployment) and the [Remote System Deployment](#remote-system-deployment) from the same scripts
2. To support the deployment of multiple roles to the same host

#### Configuration: Broker Role
The information required to configure the Broker role is as follows:

**TBD**

#### Configuration: Node Role
The information required to configure the Node role is as follows:

**TBD**

#### Configuration: Message Queueing Server Role
The information required to configure the message queuing server role is as follows:

**TBD**

## Backwards Compatibility
Most of the work that needs to be done will be done in the [puppet-openshift_origin](https://github.com/openshift/puppet-openshift_origin) repository, so from that perspective, impact on the rest of the Origin codebase will be limited. Any changes that have to be made to meet the requirements of the [roles-driven puppet scripts](#roles-driven-puppet-scripts) may affect any downstream puppet repos.

## Rationale
**Why a Text-Based Wizard?**  
The choice to use a text-based wizard for this installer was two-fold. First off, per the [survey](https://www.openshift.com/blogs/survey-results-the-openshift-origin-unboxing-experience), the option of a more graphical installer was low in popularity. Point number two is that a text-based installer can deliver a better user experience while reusing techology that was already there in the form of `oo-login`.

**Why Puppet?**  
On the actual instrumentation side, it is possible that the two most popular deployment methods (Puppet script templates and the mutli-instance VM deployment)  can be deployed using the same set of Puppet scripts. This PEP actually includes a third derivative of this instrumentation as well--the ability to deploy a [Role](#roles-driven-puppet-scripts) to an arbitrary host. While the same resultant functionality can probably be delivered by Chef or Ansible, we already have the foundation laid for Puppet and the feedback for Puppet was very strong.

**Where Did "Roles-Driven Deployment" Come From?**  
This facet of the installation problem wasn't directly addressed by the survey, but delivering upon the multi-instance deployment means making decisions up front about the tools to be used and the division of labor between roles. It is true that the configuration of the roles [as defined](#roles-driven-puppet-scripts) is only one of many viable OpenShift Origin deployments, and users who want to further modify their deployments are not locked in by the roles. That said, defining these roles paves the way for easy multi-instance setup and will also help focus the efforts of users that are interested in replacing Puppet with other configuration options.

