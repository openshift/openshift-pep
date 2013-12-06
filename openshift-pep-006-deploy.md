PEP: 006  
Title: Zero downtime for scaled applications during deployment and restarts  
Status: accepted  
Author: Andy Goldstein <agoldste@redhat.com>, Dan McPherson <dmcphers@redhat.com>, Clayton Coleman <ccoleman@redhat.com>  
Arch Priority: TBD  
Complexity: 40 - 100  
Affected Components: web, api, runtime, broker, cli  
Affected Teams: Runtime (40), UI (20), Broker (8), Enterprise (5)  
User Impact: medium
Epic: E05 https://trello.com/c/krgSeomK  

Abstract
--------
Today in OpenShift, when a user pushes an update to an application, the application is taken offline temporarily during the deployment of the new code. This means incoming requests may fail for a short period of time. With some improvements to the application lifecycle, OpenShift users should be able to push updates or restart their scaled applications with zero downtime.


Motivation
----------
Uptime and availability are key considerations for application providers when choosing a deployment platform and environment. Users attempting to access an application running in OpenShift will possibly experience an outage when that application is in the middle of a deployment, which is undesirable.

The goal of this PEP is to improve the operational experience of managing applications in OpenShift. It will include features such as:

* zero downtime during a deployment or restart
* rolling deployments/restarts
* graceful shutdown of gears (drain will be in conjunction with the HA pep)
  - First pass won't include modcluster for jboss
* ability to roll back a deployment to a previous version
* ability to push code now but build and deploy it later
* ability to implement a custom build process by re-composing operations that make up the platform build process


Specification
-------------
OpenShift must allow efficient, scalable, and controllable builds and deployments, with the ability to extend portions of the workflow.

A **build** is a process that converts source code into a **build result**. A build result is eventually **prepared** for deployment, **distributed** to some or all of an application's gears, and finally **activated** on those gears. The prepare/distribute/activate deployment lifecycle may happen immediately as part of the familiar workflow wherein `git push` builds the application and deploys it. Alternatively, a user may disable automatic deployments and choose instead to deploy in the future via new methods described below.

Build results may be created either inside or outside of OpenShift. Build results created inside OpenShift are created via `git push` and are managed by the platform and/or build cartridges (i.e. Jenkins). Build results created outside of OpenShift must conform to a specific format, and are used as inputs when using the new binary deployment feature. A build result must include the application code and/or anything produced by the application's build process (e.g. a .war file). It is essentially the state of the app-root/runtime/repo and app-root/runtime/dependencies on the gear after a "build" has been executed (mvn/rake/npm/etc)

Preparing a build result for deployment involves the execution of a `prepare` platform action and a user-defined `prepare` action hook (if one exists). If the prepare platform action is passed a file as an input, it will extract the file (relative to app-archives) into the specified deployment directory. The prepare user action hook is a place where users may execute custom code to modify the build result prior to its distribution to all of the application's gears. An example use case for the prepare action hook is using it to download environment specific files that don't belong in the application's git repository to the build result directory. The build result may optionally be compressed to minimize disk space usage.

A build result that has been prepared for distribution is a **deployment artifact**. Deployment artifacts have unique identifiers (the sha1 sum of their contents) that are created/calculated by the prepare platform action, after the user-defined prepare action hook has been executed. Once a deployment artifact has been created, it may be distributed to some or all of an application's gears.

After a deployment artifact has been distributed, it may then be activated. Activating a deployment artifact makes it the active code running for a gear. When activating a new deployment artifact, the old deployment artifact may be deleted (e.g., to save space), or it may be preserved. If the prior artifact is preserved, it will now be possible to roll back the active deployment artifact to the previous one, if a user so chooses.

### New application configuration options
New application configuration options are required to support the improved build and deployment features. These should be maintained in the broker and made available to the gears as environment variables.

* **auto_deploy**

    Possible values: true, false
    Default: true

    Indicates if OpenShift should build and deploy automatically whenever the user executes `git push`. If not explicitly disabled, the current default behavior (to auto deploy) will remain the same.

  If disabled, a `git push` will not result in the latest code being deployed to the application. Instead, the user must use the new `rhc deploy` command described below to initiate a deployment.

  Additionally, the following marker files in the application git repository will be deprecated and replaced by optional arguments that can be passed to `rhc deploy`:

  * `force_clean_build`
  * `hot_deploy`

  Any future options that are appropriate for a per deploy decision will only be added as deploy options such as:

  * a non-clean Maven build
  * running database migrations

* **deployment_branch**

    Possible values: any valid git branch
    Default: master

    Indicates which branch should trigger an automatic deployment, if automatic deployment is enabled. If not specified, the current default behavior (master branch) will remain the same.

    Pushes of branches other than the value specified by `deployment_branch` will not trigger an automatic deployment.

* **keep_deployments**

    Possible values: any number >= 1 <= configurable max  
    Default: 1

    Indicates how many total deployments to preserve. If not specified, the current default behavior of only preserving the current/active deployment will remain the same. This value must be >= 2 for rollbacks to work.
    
* **deployment_type**

    Possible values: binary|git  
    Default: git  

    Indicates whether the app is setup for binary or git based deployments.  Deployments attempted of the type not configured will result in an error.

These options will be configurable via an addition to `rhc`:

`rhc config -a myapp <option> <value>`

For example: `rhc config -a myapp auto_deploy false`

### New ways to deploy
The following describes new ways to deploy a new version of code to an application. These assume that auto_deploy is false.

* `rhc deploy -a myapp <git ref>`

    Makes the specified git ref the active deployment running in the application. Possible valid git refs include:

    * a branch name, e.g. master. The tip of the branch will be deployed.
    * a commit id, such as d34dbe3f. That specific commit will be deployed.
    * a tag, such as v1.0. That specific tag will be deployed.

* `rhc deploy -a myapp http://example.com/artifact_id-1.0.tar.gz`

    Makes the binary deployment artifact at the specified URL the active deployment for the application.

* `rhc deploy -a myapp myartifact.tar.gz`

    Uploads a binary deployment artifact located on the user's computer to the application and makes it the active deployment.

### Deployment directory structure
The following directory structure is proposed:

    app-deployments/
      2013-08-16_14-36-36.881/
        dependencies/
        metadata/
          gitref
          id
          state
        repo/
        
      2013-08-16_18-12-15.746/
        dependencies/
        metadata/
          gitref
          id
          state
        repo/
        
      by-id
        fa93c9b -> ../2013-08-16_14-36-36.881
        9191a7e -> ../2013-08-16_18-12-15.746
        
    app-archives/
      myapp-1.0.tar.gz
      myapp-2.0.tar.gz
      
    app-root/
      dependencies -> runtime/dependencies
      repo -> runtime/repo
      runtime/
        dependencies -> ../../app-deployments/2013-08-16_14-36-36.881/dependencies
        repo -> ../../app-deployments/2013-08-16_14-36-36.881/repo

The **app-deployments** directory contains 1 or more deployments in their entirety. A deployment directory name has the following format: [date]_[time], such as `2013-08-16_14-36-36.881`.  The repo directory contains the contents of what will be in app-root/runtime/repo. The dependencies directory contains the contents of app-root/runtime/dependencies. The metadata directory contains information about the deployment: the git ref of the deployment (if applicable), the deployment id, and the deployment's state (N/A or DEPLOYED).

**app-root/runtime/repo** moves from being a standalone directory to a symlink that points at the active deployment in the deployments directory.

The **app-archives** directory contains binary deployment artifacts that are the inputs to the `prepare` platform action.

### Git deployments - current OpenShift workflow
The following process will take place when `auto_deploy` is enabled, `keep_deployments` = 1, and the user pushes code to the git repository:

1. User invokes `git push`
1. Application is stopped
1. The cartridge's `pre-repo-archive` command is invoked
1. A new deployment directory `app-deployments/$date_$time` is created with `repo` and `dependencies` subdirectories
1. All dependencies from the active deployment (app-root/runtime/dependencies) are moved to `app-deployments/$date_$time/dependencies`
1. Active deployment directory in `app-deployments/` is removed
1. The contents of the git repo for the current deployment branch are unpacked into `app-deployments/$date_$time/repo`
1. A build is performed
    1. The cartridge's `pre_build` command is invoked
    1. The user's `pre_build` hook is invoked, if it exists
    1. The cartridge's `build` command is invoked
    1. The user's `build` hook is invoked, if it exists
1. The platform's `prepare` command is invoked
    1. The user's `prepare` hook is invoked, if it exists
    1. The deployment id is calculated from the contents of the deployment directory
    1. The symlink `app-deployments/by-id/$deployment_id` is created and points to `../app-deployments/$date_time`
1. The platform's `distribute` command is invoked
    1. If the app is scalable, the new deployment will be synced to all child gears
1. The platform's `activate` command is invoked
    1. `app-root/runtime/repo` is updated to point at `../../app-deployments/$date_$time/repo`
    1. `app-root/runtime/dependencies` is updated to point at `../../app-deployments/$date_$time/dependencies`
    1. The primary cartridge's `update-configuration` control action is invoked
    1. The secondary cartridges are started
    1. The primary cartridge's `deploy` control action is invoked
    1. The user's `deploy` hook is invoked, if it exists
    1. The primary cartridge is started
    1. The primary cartridge's `post_deploy` control action is invoked
    1. The user's `post_deploy` hook is invoked, if it exists
    1. If the app is scalable and the options did not include --child, for each child gear:
        1. SSH to the child gear and execute `gear activate $deployment_id --child`
    1. Write DEPLOYED to app-deployments/$date_$time/metadata/state

### Git deployments - preserving previous deployments
The following process will take place when `auto_deploy` is enabled, `keep_deployments` > 1, and the user pushes code to the git repository:

1. User invokes `git push`
1. Application is stopped
1. The cartridge's `pre-repo-archive` command is invoked
1. A new deployment directory `app-deployments/$date_$time` is created with `repo` and `dependencies` subdirectories
1. All dependencies from the active deployment (app-root/runtime/dependencies) are copied (or moved if keep_deployments == 1) to `app-deployments/$date_$time/dependencies`
1. Starting with the oldest deployment, previous deployments are removed until the number of deployments in app-deployments <= the value of keep_deployments (if necessary)
1. The contents of the git repo for the current deployment branch are unpacked into `app-deployments/$date_$time/repo`
1. A build is performed
    1. The cartridge's `pre_build` command is invoked
    1. The user's `pre_build` hook is invoked, if it exists
    1. The cartridge's `build` command is invoked
    1. The user's `build` hook is invoked, if it exists
1. The platform's `prepare` command is invoked
    1. The user's `prepare` hook is invoked, if it exists
    1. The deployment id and checksum of deployment contents are calculated
    1. The symlink `app-deployments/by-id/$deployment_id` is created and points to `../app-deployments/$date_time`
1. The platform's `distribute` command is invoked
    1. If the app is scalable, the new deployment will be synced to all child gears
1. The platform's `activate` command is invoked
    1. `app-root/runtime/repo` is updated to point at `../../app-deployments/$date_$time/repo`
    1. `app-root/runtime/dependencies` is updated to point at `../../app-deployments/$date_$time/dependencies`
    1. The primary cartridge's `update-configuration` control action is invoked
    1. The secondary cartridges are started
    1. The primary cartridge's `deploy` control action is invoked
    1. The user's `deploy` hook is invoked, if it exists
    1. The primary cartridge is started
    1. The primary cartridge's `post_deploy` control action is invoked
    1. The user's `post_deploy` hook is invoked, if it exists
    1. If the app is scalable, SSH to each child gear and execute gear activate $deployment_id
    1. Write activation time to app-deployments/$date_$time/metadata.json

### Binary deployments

Sometimes it doesn't make sense to use git to deploy an application. Git is not a particularly efficient means of deploying a pre-built Java .war file, for example.

A binary deployment artifact could be one of the following formats:

* tar
* tar.gz
* tar.bz2
* zip

Because git-based deployments remain the default mechanism, a user will have to manually convert an application to use binary deployments using a new config option `deployment_type` which corresponds to the environment variable: OPENSHIFT_DEPLOYMENT_TYPE=binary. If this variable does not exist, the default behavior to use git for deployments will remain the same. The other possible value for this variable is OPENSHIFT_DEPLOYMENT_TYPE=git, which has the same effect as the variable not being set.

A binary artifact is deployed by running a command such as `rhc deploy -a <app> <url>`, where `<url>` refers to the binary artifact to be deployed. The `<url>` could be a remote link, or a local file.

The proposed format of a binary artifact is as follows:

    repo
      .openshift
        ...
      application-specific files
    dependencies
      ...

A .war binary deployment artifact might look like this:

    repo
      .openshift
        ...
      deployments
        ROOT.war
    dependencies
      ...

Essentially the artifact contains exactly what app-root/runtime/repo would have after a build has taken place.

The following process will take place when `auto_deploy` is disabled, `keep_deployments` > 1, and the user executes `rhc deploy -a myapp <url>`:

1. User invokes `rhc deploy -a myapp <url>`
1. A new deployment directory `app-deployments/$date_$time` is created with `repo` and `dependencies` subdirectories
1. Starting with the oldest deployment, previous deployments are removed until the number of deployments in `app-deployments` <= the value of `keep_deployments` (if necessary)
1. The platform's `prepare` command is invoked
    1. The file specified by <url> is downloaded and extracted to `app-deployments/$date_$time`
    1. The user's `prepare` hook is invoked, if it exists
    1. The deployment id and checksum of deployment contents are calculated
    1. The symlink `app-deployments/by-id/$deployment_id` is created and points to `../app-deployments/$date_time`
1. The platform's `distribute` command is invoked
    1. If the app is scalable, the new deployment will be synced to all child gears
1. The platform's `activate` command is invoked
    1. `app-root/runtime/repo` is updated to point at `../../app-deployments/$date_$time/repo`
    1. `app-root/runtime/dependencies` is updated to point at `../../app-deployments/$date_$time/dependencies`
    1. The primary cartridge's `update-configuration` control action is invoked
    1. The secondary cartridges are started
    1. The primary cartridge's `deploy` control action is invoked
    1. The user's `deploy` hook is invoked, if it exists
    1. The primary cartridge is started
    1. The primary cartridge's `post_deploy` control action is invoked
    1. The user's `post_deploy` hook is invoked, if it exists
    1. If the app is scalable, SSH to each child gear and execute gear activate $deployment_id
    1. Write activate time to app-deployments/$date_$time/metadata.json

### Node API

This section describes the API for the following foundational actions:

1. Build: create a deployment from the application's git-repo
1. Prepare: prepare a deployment for distribution and activation
1. Distribute: copy a deployment to web gears in an application
1. Activate: Begin running a deployment on the web gears in an application

Each workflow phase implements a contract:

**Build**

Inputs:

1.  `deployment_request_id`: use a prior deployment instead of the current repo for the build

Outputs:

1.  Combined output of all actions as a String (TODO: refactor)

Effects:

1.  The `dependencies` and `build_dependencies` symlinks are updated to point to the `deployment_request_id`, if it is supplied.
1.  `update-configuration`, `pre-build`, and `build` control actions are run on the primary cartridge

Failure Modes:

1.  If `deployment_request_id` is supplied and does not exist, raise an `ArgumentError`.
1.  Restart the application as the gear user if a build failure occurs and `$OPENSHIFT_KEEP_DEPLOYMENTS` is greater than 1.

**Prepare**

Prepare a deployment artifact from a deployment request.

Inputs:

1.  `deployment_request_id`: datetime of the deployment to prepare
1.  `file`: an archive file to use in place of a deployment

Outputs:

1.  Combined output of all actions as a String (TODO: refactor)

Effects:

1. A new deployment id is calculated from the deployment request
1. A symlink is created in the by-id directory for the deployment id
1. The deployment metadata for the id is updated to point to the new id

Failure Modes:

1. If `deployment_request_id` is not supplied, raise an ArgumentError
1. If a fault occurs after a link is created in the by-id directory, it is unlinked.

**Distribute**

Copy a deployment to gears in the application.

Inputs:

1. `gears`: the gears to distribute to.  Optional; defaults to other web framework gears in the application.
1. `deployment_id`: the ID of the deployment to distribute.

Outputs:

    {
      status: :success # or :failure,
      results: {
        #{gear_uuid}: {
          gear_uuid: #{gear_uuid},
          status: :success, # or :failure
          messages: [],
          errors: []
        }, …
      }
    }

Failure Modes:

1.  If no deployment ID is supplied, raise an `ArgumentError`.
1.  If distribute to a gear fails, retry up to two times before considering that gear to have failed.
1.  If distributing to any gear fails, the entire operation is a failure, but the platform will continue distributing to other gears in the application.

**Activate**

Run a deployment on gears in the application.

Inputs:

1.  `gears`: the gears on which to activate a deployment.  Optional; defaults to other web framework gears in the application.
1.  `deployment_id`: the ID of the deployment to activate.
1.  `init`: Run the web cartridge's `post-install` workflow after activating the deployment.  Optional; defaults to `false`.
1.  `hot_deploy`: Do not disable the gear in the cluster during activation.  Controlled by the app's hot deployment setting; defaults to `false`.
1.  `out`:

Outputs:

    {
      status: :success # or :failure,
      results: {
        #{gear_uuid}: {
          gear_uuid: #{gear_uuid},
          status: :success, # or :failure
          messages: [],
          errors: []
        }, …
      }
    }

Failure modes:

1.  If no deployment ID is supplied, raise an `ArgumentError`.
1.  If any gear activation fails, the entire operation is a failure, but the platform will continue activating on other gears in the app.
1.  If remote activation of a gear returns a non-zero exit code, generate a failure result for that gear.
1.  If disabling a gear in any proxy fails, activation of that gear is a failure.  App is now potentially partially disabled across the cluster.
1.  If enabling a gear in any proxy fails, activation of that gear is a failure.  App is not potentially partially disabled across the cluster.

**Update Proxy Status**

Change the status of a gear in the proxies for the app.

Inputs:

1.  `action`: action to perform (disable/enable)
1.  `gear_uuid`: gear to change status for
1.  `cartridge`: the web proxy cartridge
1.  `persist`: store the status change in the proxy config file
1.  `requesting_gear_uuid`: uuid of the gear requesting the status change

Outputs:

    {
      status: :success, # or :failure
      target_gear_uuid: #{gear_uuid}
      proxy_results: {
        #{proxy_gear_uuid}: {
          proxy_gear_uuid: #{proxy_gear_uuid},
          status: :success, # or :failure,
          messages: [], # strings
          errors: [] # strings
        }, ...
      }
    }

Failure Modes:

1.  If no valid action is supplied, raise an `ArgumentError`.
1.  If no gear uuid is supplied, raise an `ArgumentError`.
1.  If no cartridge is supplied, and there is no web proxy cartridge, raise an `ArgumentError`.
1.  If updating the status of the gear in any proxy fails, the operation is considered a failure, but the platform will continue updating the status of the gear in other proxies.


### Deployment rollback capability
OpenShift must support an easy way to rollback from one deployment to the previous one. To do so, the previous deployment must be preserved.

To perform a rollback, the following command will be added: `rhc deploy -a myapp --rollback`

This will execute the following sequence of actions:

1. Ensure that a previous deployment exists; return error to the user if not
1. Stop the application
1. Delete the current deployment directory (`app-root/runtime/repo/..`)
1. The previous deployment's deployment ID is read from metadata/id
1. The platform's `activate` command is invoked for the previous deployment ID (see above for details)
1. If the app is scalable and the options did not include --child, for each child gear:
    1. SSH to the child gear and execute `gear rollback --child`

### Types of applications and deployments

There are three likely scenarios under which applications are deployed:

1.  A simple, non scalable app that may have limited disk space
2.  A scaled app that may have limited disk space
3.  A scaled app with sufficient disk space for storing multiple deployments

Most production level apps are assumed to fall in category 3. Most development apps are assumed to fall in categories 1 (pure development) or 2 (testing).  Applications in #2 are supported, but not extensively optimized for availability.  The further into category #3 applications fall, the more they need to be optimized for performance and speed of deployment.

In addition, users may desire to deploy using two distinct methods:

1.  **Scale-replace**

    Create new gears with the new deployment artifact and scale down gears running the older deployment.  The new gears must have the new deployment artifact copied after creation.

2.  **In-place**

    Change gears in place, with or without altering the scale factor.  Predeploys the new deployment artifact to a temporary location, stops the gear, symlinks the new artifact to become the active deployment, and the starts the gear.

A concrete limitation of scale-replace is that the head gear in non HA applications must still be updated at low scale factors (N<3) via an in-place deploy (since the head gear cannot be scaled).  An advantage of the scale-replace is that it forces stateless gears and ensures movement across nodes of active content (beneficial for load balancing).  A downside is that gears are always cold.

The minimum deployment time for scale-replace is described by

    T(deploy) = ( N(gears) / N(extra_cap) - 1 ) * ( T(gear_create) + T(gear_deploy) + T(gear_activate) )

where N(gears) is the number of gears in the application, N(extra_cap) is the extra capacity in number of gears, T(gear_create) is the time to create a new gear with an instance of the cartridges, T(gear_deploy) is the time to copy the deployment artifact onto disk from the source, and T(gear_activate) is the time to swap the artifact to the newly deployed version and start the gear.

The minimum deployment time for in-place is:

    T(deploy) = ( N(gears) / ( N(extra_cap) + N(fraction) ) - 1 ) * ( T(gear_activate) )

because gear_deploy can be executed before the deployment begins.  There is no combination of factors where scale-replace is faster than in-place.

Some large installations may not have access to sufficient floating capacity to accomplish their update within specific time constraints - they will wish to trade increased disk usage for increased speed / decreased overall deployment time.  Some users will wish to try simple deploys with limited disk space - that can be accomplished with scale-replace.

Therefore, we recommend scale-replace when per-gear disk space (i.e., quota) is limited and availability is desired (category 2), and in-place for unscaled, disk limited apps (category 1) and high scale, disk unlimited apps (category 3).  It is recommended that application owners must provision sufficient capacity for in-place and opt-in to that deployment method.

### Zero-downtime deployments
It should be possible to deploy to a scaled application without incurring application downtime. To accomplish this, either scale-replace or in-place deployments could be used. If in-place, the application must have at least 2 gears to avoid downtime.

### Broker involvement
With the added ability to deploy any git ref as well as binary deployments, there needs to be a way to know what version of the application is deployed on a gear. This information should reside in the gear itself, and it should also be communicated back to the broker, so the broker can persist it.

The broker should maintain the current state of all of an application's gears, including the current deployment id and information about previous deployments. Whenever a deployment or rollback succeeds, the platform should inform the broker of the changes to each gear's deployments.

The gear will be treated as the system or record, so in the event of a discrepancy between the gear and the broker, the gear will "win."

If the broker is not available, it should still be possible to perform deployments and rollbacks. We should consider a "fallback" mode where a deployment can be directed by an external process using only the capabilities available on the gears. This fallback mode has already been implemented in a production environment, so the focus should be on integrating that logic into `rhc` in "director mode". This may be accomplished by ensuring that the build, prepare, distribute, and activate steps can be invoked via shell in any gear with a Git repo (they take simple arguments) - the remaining reporting and coordination can be maintained by an external tool.

An example application might consist of 3 redundant head gears, with 10 other scaled web gears. The coordinator would be able to execute the following SSH session to reproduce the in-place deploy of a commit f03435b:

    $ ssh gearid1@myapp-mydomain.rhcloud.com
    Connecting to gearid1@myapp-mydomain.rhcloud.com...
    > build f03435b
    Building Git commit f03435b....
    ....
    Build complete, prepared deployment artifacts in app-deployments/20130305_120034-d3948b93
    Deployment id is d3948b93
    > distribute d3948b93
    Copying d3948b93 to all gears
    .....
    [gear 4] connection denied
    d3948b93 distributed to 12/13 gears.  1 failure.
    > activate d3948b93 --disable-missing --step=2
    [gear 4] removed from active rotation
    [gear 1] updated to d3948b93
    [gear 2] updated to d3948b93
    ...
    d3948b93 activated on 12/13 gears. 1 gear removed from rotation.

The broker would also make use of the steps in this process, and report data in the appropriate format. More work is needed to isolate the exact separation points in the gear and platform.

Deploy will also be exposed through a REST API to be used from any non ssh clients.  The REST API will be (non binary deploy):

`POST /domains/<dom_name>/applications/<app_name>/events?event=deploy`

This operation will perform the build and deploy and take the same options provided through ssh (force_clean_build, git ref, etc).  Output will need to be accessible through standard gear logging facilities.

### Storage considerations
In OpenShift environments where users have constrained file quotas (e.g. 1GB in Online), it may not be desirable (or even possible) to keep 2 copies of different releases on disk at the same time (N, N+1).

If a gear runs out of disk space during a deployment, the platform will remove 1 or more previous deployments (if they exist) in an effort to keep the gear below its quota limit.

### HAProxy changes
The HAProxy cartridge is currently responsible for synchronizing an application's files from the head gear to the child gears. This synchronization happens automatically whenever a user executes `git push` to update an application. This functionality should be moved from the HAProxy cartridge to the platform. If other web proxies besides HAProxy are available at some point in the future, each would have to implement file synchronization; as a result, this should be handled by the platform to avoid duplicate file synchronization implementations.

When the platform synchronizes files, the synchronization will happen before all cartridge and user deploy hooks are executed.

### Additional dependencies
Some frameworks have external dependencies that are required at build time, run time, or both. These often do not reside with the application's source code and are often downloaded during a build or deployment. Examples include Java dependencies retrieved via Maven, node.js modules, Python virtenv files, etc. We will track the state of these external dependencies per deployment, and be able to tie the set of files that exist at a given point in time to a given deployment.  Cartridges can accomplish this by storing their dependencies under app-root/runtime/dependencies and linking to that directory if their framework requires a particular directory structure.  Ex:

php/phplib -> app-root/runtimedependencies/phplib

#### Publisher gears
1 possible way to implement these changes to deployments is to create a gear that is dedicated to performing builds and deployments to the real application gears. Individual deployments reside on the publisher gear, and they would be synchronized to the real application gears at deployment time. Instead of each application gear being required to maintain a copy of the last 1 or 2 deployments (to support rollback), the publisher gear could house the deployments, limiting the amount of duplication necessary (in the event that every child gear had to have a copy of deployments *n*, *n-1*, and possibly *n-2*.

Publisher gears would need to be redundant to avoid being a single point of failure.

#### Deployment failures/errors
In the event that a deployment to a gear fails, the user should be notified of the failure. No attempt at auto-correcting the failure will be made, and the user will be required to fix whatever the issue is.

#### Deployment validation
At the end of a deployment, the platform should validate that the deployment succeeded. This could be as simple as validating the contents of a deployment status file on a gear.

#### Targeting specific gears
It may be desirable to allow the user to target one or more specific gears when performing a deployment/restart. A potential scenario would involve deploying a new release to 10% of an application's gears, smoke testing the new release on those gears, and then continuing to deploy to the remaining gears if the smoke test passed.

#### Graceful shutdowns / draining connections
Before performing any deployment or restart operation, active connections to a gear should be gracefully drained. This is usually accomplished by modifying the load balancer (haproxy) to disallow new connections while allowing established connections to complete their work.

There may potentially be issues if versions *n* and *n+1* are both active at the same time and HTTP session replication is in use. Dealing with these is out of scope of this PEP.

#### Health Checks
OpenShift currently only supports HTTP health checks at "/". While this is acceptable for many applications, there are definitely times when an application is not deployed at /, or when a non-HTTP health check is desired. It may be useful for an application to be able to define 1 or more health checks that the platform should execute. This could be done in a file in the application's .openshift directory hierarchy, such as .openshift/configuration/health_checks.yml or .openshift/configuration/application.yml.

In this file, the user could specify 1 or more health checks. Each health check could be performed by the load balancer (haproxy) for web applications, or by the platform for non web applications. Non web health checks would require the platform to execute a script specified by the user in the configuration file. If the script returns success (0), the application is considered healthy. Any nonzero return code indicates unhealthy/failure.

#### Build when notified by an external source push

*NOTE: Speculative

A user desires to deploy when commits are pushed to an external GitHub repository.  They configure their application for build-on-push to "master", and ensure their internal application repository is a clone of the upstream.  They then enable web hook support via the broker for their application by creating a new authorization with application web-hook scope.  The broker will then accept a POST request to 

    /broker/rest/applications/<appuuid>/webhooks/<token_value>/<action>

which is passed directly to one of the application web gears for processing.  The platform code would look for a <code>.openshift/action_hooks/webhook</code> script and execute it with the arguments

    <action> <payload>

where payload is is the <code>payload</code> parameter passed to the broker in JSON.  The webhook script would execute a simple git fetch against an upstream, and then git push that upstream to the local repository in the background and return.


### Distributed Publishing (Multi-Master / Multi-HA)
A scaled application currently has a single master git repository, while the child gears only receive copies of files necessary for deployment. To remove the single master from being a single point of failure, all child gears should also receive a copy of the master git repository. Their copies would be read-only. If the current master dies at some point, any of the child gears should be able to be elevated to be the new master.

**Details to follow HA implementation

Backwards Compatibility
-----------------------
The details and changes described in this PEP should be fully backwards compatible with previous releases. None of the default behaviors change (`git push` still builds and deploys master). It is possible, maybe even likely that the directory structure for deployments may have to change slightly, and this could be addressed as part of a migration/upgrade script. Newer features, such as the ability to deploy binary artifacts, would only be available to a fully upgraded OpenShift installation. Older clients should continue to function without anything breaking.


