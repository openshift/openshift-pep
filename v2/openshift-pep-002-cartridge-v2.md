PEP: 002  
Title: Refactor cartridge and node platform code to lower entry for cartridge authors  
Status: accepted  
Author: Jhon Honce <jhonce@redhat.com>  
Arch Priority: high  
Complexity: 100  
Affected Components: runtime, cartridges, broker  
Affected Teams: Runtime (5), UI (0), Broker (1), Enterprise (5) *jwh: Enterprise here is a "different thing"*  
User Impact: application developer (low), cartridge author (high)  
Epic: [US2765 Finalize Cartridge Format](https://rally1.rallydev.com/#/4670516379d/detail/userstory/7579329564)  

Abstract
--------

Refactor the cartridge format to allow for the easier community
development of cartridges and remove any requirements for system
administration (root) level access.

Motivation
----------
The current cartridge format suffers from the following major issues:

  1. There is not a clean demarcation between the code needed to provide
     the cartridges service and the code to configure the hosting node.

  2. Some operations require system administration (root) level access.

  3. We want to enable cartridge authors an iterative approach to
     cartridge development and testing. When sufficiently complete they
     can then turn that initial work into a consumable cartridge without
     inordinate work. Based on the success of DIY and the feedback we
     have received, moving all cartridges to a format in that style
     should provide community cartridge authors an easy platform to
     development and maintain their services/cartridges.

How a cartridge author makes their cartridge available for consumption by
application developers is out of scope for this PEP and will be addressed
in a later PEP.

Specification
-------------

Phase 1:

Remove unused/empty scripts, move functionality into ruby layer to
improve documentation and automated testing.

  1. Move all abstract cartridge functionality that is not specialized
     in the concrete cartridges into the node platform. This includes:
       a. Implement add-alias/remove-alias in ruby for node platform
       b. Implement force-stop in ruby for node platform
       c. Remove the pre-install/post-install/post-remove scripts
       d. Implement duplicated tidy code for platform, call hook in
          cartridge for specialization
       e. Implement Expose/Conceal port hooks in ruby for node platform.
          Configuration items added to cartridge manifest


Phase 2:

Change cartridge format and supporting code to better align with community
driven cartridges, faster installs, cleaner demarcation between node
platform and cartridge code. High level design goals:

  1. Gear-level operations are implemented in the node platform code not the cartridge
  2. Cartridge code is driven by node platform code
  3. Cartridge code uses convention not configuration
  4. Environment variables will continue to be the method of communication between
     cartridges and gears, and other cartridges

See [How To Write An OpenShift Origin Cartridge 2.0](https://github.com/openshift/origin-server/blob/master/node/README.writing_cartridges.md)
and [How To Write An OpenShift Origin Node Agent 2.0](https://github.com/openshift/origin-server/blob/master/node/README.node_module_design.md) for detailed work flows.
and [How To Write An Application To Host on OpenShift](https://github.com/openshift/origin-server/blob/master/node/README.writing_applications.md) for details on writing applications to be hosted on OpenShift

Backwards Compatibility
-----------------------

These changes will not impact rhc client code, as access to the cartridges
are via the broker and REST API.

The magnitude of the changes for the cartridges and node platform
make the argument that backwards compatibility should not be
maintained. Development will be done on a feature branch and merged
into master once work was finished and passed QE.

All existing cartridges for Origin, Online, Enterprise would need to be
ported to new code base.

At this time is it believed that with the Type-less gears work and once
Phase 1 is complete then a migration of current cartridge instances to
the new cartridge instances will be possible.

Migration work flow:
  1. Remove old cartridge rpm
  2. Install new cartridge rpm
  3. Rename existing cartridge instance preserving the data and configuration
  4. Add new cartridge instance, restoring data and configuration where needed.
  5. Remove old cartridge instance

Migration Concerns:
  1. Moving environment variables to cartridge dependent directories may effect
     applications if they are reading from the disk rather than
     the process environment.
       a. Node platform code will be modified to gather all the cartridge
          segregated environment variables in one process space when calling new
          cartridge scripts..

  2. It will overly complicate the migration if the version of the
     software controlled by the old cartridge is different than the new
     cartridge. This is not the time to look for a quick upgrade.

  3. The creation of additional cartridges will also impact the amount
     migration work.


Rationale
---------

The well-defined interfaces and demarcation between node platform and
cartridges will allow the cartridge author to write their software in
any language supported by hosting node. Moving platform functionality
out of the cartridges into the node platform code simplifies writing
cartridges. Suggested languages for cartridge development for Online usage
is Bash or Ruby. Origin or Enterprise is dependent on the installation.

"Convention over configuration" lowers the barrier to new cartridge
authors.

Methods will be included in the refactor to allow the cartridge author to
denote which files they need write access to while adding the cartridge
to a gear. The application developer will not be granted write access
to these same files.

ERB was chosen to replace sed to remove the cartridge author needing to
learn regex for simple variable replacement. 

The 'usr' directory, that is shared by all instantiated cartridge
instances via a symlink, was introduced to allow for disk usage savings
and to ease simple patching.

Copying the remaining data from a cartridge cache/repository into a gear
when adding a cartridge to a gear, is simple and fast. Most of these
files will be customized either as ERB files or via the cartridge's
'setup' script.

To help mitigate the total time required to build the new cartridges,
a harness will be written to verify the cartridge's structure and
functionality. It is assumed the harness will be a specialization of
Ruby's minitest or rspec runner.

Additionally, a mock-style cartridge will be written to allow testing
of the new node platform code independent of any controlled software.
