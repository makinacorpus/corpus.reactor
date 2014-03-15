corpus, the PaaS Platform -  RFC
==================================

The origin
------------
Nowoday no one of the P/I/S/aaS existing platforms fit our needs and habits.
No matter of the gret quality of docker, heoru or openshift, they did'nt make it.
And, really, those software are great, they inspired corpus a lot !

For exemple, this is not a critisism at all, but that's why we were not enougth
to choose one of those platforms (amongst all of the others):

    `heroku`_

        non free

    `docker`_

        - Not enougth stable yet
        - do not implement all of our needs, it is more tied to the 'guest' part
          (see next paragraphs)
        - But ! Will certainly replace the LXC guests driver in the near future.

    `openshift`_
        Tied to SElinux and RedHat (we are more on the Debian front ;)).
        However, its great design inspired a lot of the corpus one.

The needs
----------
That's why we created a set of tools to build the best flexible PaaS platform
ever.

- Indeed, what we want is more of a CloudController + ToolBox + Dashboards +
  API.
- This one will be in charge of making projects install any kind of compute nodes
  smoothly and flawlessly.
- Those projects will never ever be installed directly on compute nodes but rather
  be isolated.

    - They will be isolated primarly by isolation-level virtualisation
      systems (LXC, docker, VServer)
    - Bue it can also be plain VMs (KVM, Xen).

- We don't want any PaaS platform which will put use in some sort of lockin.
- We prefer a generic deployment solution that scale, and better AUTOSCALE !
- All the glue making the configuration must be centralized and automatically
  orchestrated.
- This solution must not be tied to a specific compute node type (baremetal,
  EC2)nor a guest driver type (LXC, docker, XEN).
- Corrolary, the **low level** daily tasks consists in managment of:

    - network (<OSI L3)
    - DNS
    - Mail
    - operationnal supervision
    - Security, IDS & Firewalling
    - storage
    - user management
    - baremetal machines
    - hybrid clouds
    - public clouds
    - VMs
    - containers (vserver, LXC, docker)
    - operationnal supervision

- Eventually, on top of that  orchestrate projects on that infrastructure

    - installation
    - upgrade
    - continenous delivery
    - intelligent test reports, deployment reports, statistic, delivery & supervision dashboards
    - backups
    - autoscale

Here is for now the pieces or technologies we are planning or already using to
achieve all of those goals:

- Developer environments

  - makina-corpus/vms + mcorpus.reator + makina-corpus/makina-states +
    saltstack/salt

- Bare metal machines provision

  - salt.cloud saltify + makina-states + saltstack
  - Ubuntu server

- VMs (guests)

    - lxc-utils LXC containers + makina-states + saltstack

- DNS:

    - makina-states + powerdns: dynamic  managment of all DNS zones
    - makina-states + bind: local cache dns servers

- Filesystem Backup

    - rdiff-backup (legacy)
    - bacula and/or burp (future)

- Database backup

  - `db_smart_backup <https://github.com/kiorky/db_smart_backup>`_

- Network:

    - ceph, openvswitch

- Logs, stats:

    - logstash + kibana
    - graphite

- operationnal supervision

    - centreon (legacy)
    - icinga2 (future)

- Mail

    - postfix

- User managment

    - Fusion directory + openldap

- Security

    - shorewall, psad & so on

- CloudController

    - powerdns
    - makina-states
    - mastersalt
    - salt.cloud
    - corpus.web + corpus.reactor

- projects installation, upgrades & contineous delivery

    - States in makina-states (makina-states.project)

- autoscale

    - corpus.reactor + salt.cloud + makina-states

The whole idea
----------------------
The basic parts of corpus PaaS platform:

    - The cloud controller
    - The cloud controller client applications
    - The compute nodes

        - Where are hosted guests

            - Where projects run on


The first thing we will have is a classical makina-states installation in
mastersalt mode.
We then will have salt cloud as a cloud controller to control compute nodes
via **makina-states.services.cloud.{lxc, saltify, ...}** (lxc or saltify)
Those compute nodes will install guests.
Those guests will eventually run the final projects pushed by users.

Hence an api and web interface to the controller we can:
- Add one or more ssh key to link to the host
- Request to link a new compute node
- Request to initialize a new compute node
- List compute nodes with their metadata (ip, dns, available slots, guest type)
- Get container base informations (ssh ip / port, username, pasword, dns names)
- Link more dns to the box
- Manage (add or free) the local storage.
- Destroy a container
- Unlink a compute node

Directly on or to the guest we can:
- Push the new code to deploy and it will do the delivery procedure which will
be different weither the environment we are on.
- Connect via ssh to do extra manual stuff if any

Permission accesses
--------------------
- We will use an ldap server to perform authentication

The different environment platforms
-------------------------------------
We also want to distinguish at least those 3 environments

:dev: The developper environments (laptop)
:staging: the stagings and any other QA platform
:prod:  the production platform

Objectives
------------
The layout and projects implementation  must allow us to

- Automaticly rollback any unsucessful deployment
- In production and staging, archive application content from N last deployments
- In production and staging, archive the application data from N last deployments
- In the near future, do warm/live migration
- Make the development environment easily editable
- Make the staging environment a production battletest server
- Make the staging environment a production deliverables producer
- Production can deploy from non complex builds, and the less possible dependant of external services

This way, we can manage and provision anything we need on those nodes, but we also separates security concerns.

In most cases building things on the production nodes is really a bad idea and error prone to lot of factors (network, build scripts bugs).
We handle this by providing a simili PAAS approach were we assemble artifacts to produce production ready deliverables.
Those artifacts will be able to run directly on production environments minus little provisionning, reonfiguration and upgrade paths
This is non so far from an **-extract-and-run-**.
For this, we inspired ouselves a lot from openshift_ and dheroku_ (custom buildpacks) models.

Actual layout
-------------
Overview of the project source code repositories
+++++++++++++++++++++++++++++++++++++++++++++++++
A project woill have at least 2 repositories
- A repository where lives its sourcecode and deployment recipes

This repository master branch consequently has the minimal following structure::

    master
        |- what/ever/files/you/want
        |- .salt -> the salt deployment structure
        |- .salt/top.sls -> the salt sls file to execute to deploy the project
        |- .salt/standalone.sls -> the salt sls file to execute to deploy the
                                   project in non full mode

- A private repository with restricted access with any configuration data needed to deploy the
  application on the PAAS platform. This is in our case the project pillar tree::

    pillar master
       |- init.sls the pillar configuration

As anyways, you ll push changes to the PAAS platform, no matter what you push,
the PAAS platform will construct according to the pushed code :).

Overview of the paas directories
+++++++++++++++++++++++++++++++++
/srv/projects/myproject/git/project.git/
    Remote to push the project & salt branch to
/srv/projects/myproject/git/pillar.git/
    Remote to push the project pillar branch to

/srv/projects/myproject/project/
    The local clone of the project branch from where we run in all modes.
    In other words, this is where the application runtimes files are.
    In applicatio speaking

        * **django/python ala pip:** the virtualenv & root of runtime generated configuration files
        * **zope:** this will the root where the bin/instance will be lauched
          and where the buildout.cfg is
        * **php webapps:** this will be your document root + all resources
        * **nodejs:** etc, this will be where nginx search for static files and
          where the nodejs app resides.
/srv/projects/myproject/pillar
    The project specific states pillar tree local clone.

/srv/projects/myproject/data/
    Where must live any persistent data
/srv/projects/myproject/build/
    Directory in which we can build or deal with extra builds steps
    which need a temporary space to build on.
/srv/projects/myproject/deploy/
    A directory to copy files into to construct archives to be deployed in final
    environments

/srv/projects/myproject/releases/deployed/current/ -> /srv/projects/myproject/releases/deployed/<DATETIME>-<-UUID>/
    In **cooking** mode, where all archives needed to be deployed must be stored
/srv/projects/myproject/releases/deployed/<DATETIME>-<ANOTHER-UUID>/
    A previous deployment archives directory
/srv/projects/myproject/releases/failed/<DATETIME>-<ANOTHER-UUID>/
    A previous failed deployment archives directory

/srv/pillar/makina-projects/myproject -> /srv/projects/myproject/pillar
    pillar symlink
/srv/salt/makina-projects/myproject -> /srv/projects/myproject/.salt/
    state tree project symlink

The **.salt** directory will contain at least those following saltstack sls.
Dont worry, those are generated the first time you issue the init_project procedure.

Each of those sls will run one common procedure (you choose a project installer and
then you ll have this common procedure) and you can also write extra stuff to be
done on that specific stage to perfect your deploment.

All those sls files cannot be run with state.sls but via the mc_project.<method>
functions. Indeed, they need a special environment which is only setted that
way.

/srv/projects/myproject/.salt/deploy.sls
    include the installer deploy procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/archive.sls
    include the installer archive procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/initialization.sls
    include the installer initialization procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/release-sync.sls
    include the installer release sync procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/configure.sls
    include the installer configure procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/build.sls
    include the installer build procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/reconfigure.sls
    include the installer reconfigure procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/activate.sls
    include the installer activate procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/upgrade.sls
    include the installer upgrade procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/rollback.sls
    include the installer rollback procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/notification.sls
    include the installer notification procedure and maybe do extra
    stuff
/srv/projects/myproject/.salt/post_install.sls
    include the installer post_install  procedure and maybe do extra
    stuff


* The **persistent configuration directories**

    /etc
         static global configuration (/etc)

* The **persistent data directories**
    If you want to deploy something inside, make a new archive in the release
    directory with a dump or a copy of one of those files/directories.

    /var
        Global data directories (data & logs) (/var)

    /srv/projects/project/data

        * Specific application datas (/srv/projects/project/data)

            * Datafs and logs in zope world
            * drupal thumbnails
            * mongodb documentroot
            * ...

* The **build working directory** where all build time procedure will operate before placing the results
  in the **project** directory.

* **Networkly speaking**, to enable switch of one container to another
  we have some solutions but in any case, **no ports** must be
  **directly** wired to the container. **Never EVER**.

Either:

* Make the host receive the inbound traffic data and redirect (NAT) it to the underlying container
* Make a proxy container receive all dedicated traffic and then this specific container will redirect the traffic to the real underlying production container.

For the big data containers, this will handled case per case by for exemple mounting the persistent volumes between both containers.

Project operation modes
-----------------------
The editable mode
++++++++++++++++++
This mode will be mainly used in **development**.
The difference from the other modes is the workflow to update repositories.
Here the directories are pulled inside local directories and pushed onto local
git repositories.
This allows users to directly edit and play with the local files without having
first to push to the PaaS platform which is certainly in this case a VM on their
working computer.
The building stuff is handled via the **build** related macros.
In **editable** mode, the later quoted **bundle** and **deploy** macros are
skipped.

The cooking mode
+++++++++++++++++
The **cooking** mode is a environment more suitable for **staging** environments.
The idea is there to add the cooking of production ready deliverables artifacts as a
part of the build & deploy procedure.  At the end of the build steps, if it is sucessfull,
we will synchronnise the **project** directory with the **deploy** directory.
After this synchonnisation we will make one or many **release deliverable archive** to be deployed later in production.
Those release archives will eventually be placed in the **releases** directory.
If you need additionnal files to be deployed, add more archives to the release
directory.
The cooking stuff is done via all **bundle** related macros.

The final mode
+++++++++++++++
In production, we will mainly and mostly use the **final** mode.
In this mode, we do not run any complicated building states.
In other words we will totally skip the **build** and **bundle** macros.
Indeed, all the generated during build stuff which lands in archives that we
will grab and extract directly to the **deploy** directory.
This deploy directory will then be synced identically (rsync --delete) to the
**project** directory.
Please take care then to **NEVER EVER** have persistent stuff in the project
directory onto production which is not a part of the release artifacts.

Git remotes default configuration
----------------------------------
origin
    The real distant remote
local
    The local bare git repositories

Local git working copies will have those 2 remotes configured.

In **editable** mode, the init_project will use the **origin** remote.
In **cooking** and **final** mode, the init_project will use the **local** remote.

Procedures
-------------
Those procedure will be implemented by either:

    - Manual user operations or commands
    - Git hooks
    - salt execution modules
    - jinja macros (collection of saltstack states)

Deployment trigger procedure
++++++++++++++++++++++++++++
**cooking** / **final** mode

    - issue a git push (--force) onto the git pillar the project remotes

        - shutdown any service (normally not that much as we are on a fresh or a copy
          container/vm)
        - run the archive procedure

**editable** mode

    - User launchs the mc_project.deploy execution module function

From there, project deployment is continued

Project initialization/sync procedure
+++++++++++++++++++++++++++++++++++++
- Initiate the project specific user
- Initiate the ssh keys if any
- Initiate the pillar and project bare git repositories inside the git folder
- Clone local copies inside the project, pillar and salt directories
- If the salt folder does not exists, create it
- If any of default slses procedures are not yet present, create them
- If we are in editable mode, clone from origin remote
- Wire the pillar configuration inside the pillar root
- Wire the pillar init.sls file to the global pillar top file
- Wire the salt configuration inside the salt root

Project archive procedure
++++++++++++++++++++++++++
- If size is low, we enlarge the container
- run the pre archive hooks
- archive the **project** directory in an **archive/deployed** subdirectory
- run the post archive hooks (make extra dumps or persistent data copies)
- run the archives rotation job

Project Release-sync procedure
++++++++++++++++++++++++++++++
- Fetch / Ask for each archives defines in the release_artifacs_urls
  into a **release** subdirectory.
- Wipe and recreate the **deploy** directory
- Unpack the **project** archive to the deploy folder
- Sync exactly this content to the **project** folder (rsync --delete)

Project configure procedure
++++++++++++++++++++++++++++
- Install build pre requisites
- Run any pre build step like:

    - Setting user accesses
    - Apply patches to local files
    - Reorganizing files
    - Clone extra repositories
    - Configure/install prerequisites local services (apache, mysql,
      local pypy server)

Project build procedure
++++++++++++++++++++++++++
- Wipe and recreate the **build** directory
- Do eventual compilations here
- Run here any heavy build or network related steps
  EG calls to:

        - buildout
        - grunt
        - gulp
        - npm
        - rake
        - ant, mvn
        - drush make

- If it is possible, we should to the build inside the **build** directory with
  having the substancial result living in the **project** directory

Project reconfigure procedure
++++++++++++++++++++++++++++++
- settings file generation even if already done done in configure & build steps as they
  will not be launched inside **final** environments). Do macros ;)
- aintenance procedures registrations (logrotates, crons)

Project activation procedure
++++++++++++++++++++++++++++
- start any service (normally not that much as we are on a fresh or a copy container)

Project upgrade procedure
++++++++++++++++++++++++++
- We check if the upgrade step has already be done (
  We check on the filesystem for the upgrade step file marker)
  and fail entirely the deployment if already done.
- If ok, we run upgrade steps defined in the upgrade file

Project post_install procedure
+++++++++++++++++++++++++++++++
- do any user defined custom extra post install steps


Project notification  procedure
+++++++++++++++++++++++++++++++
- We sent via the configured mean the result of deployment to user (mail,
  stdout)

Rollback procedure
+++++++++++++++++++++
- We move the failed **project** directory in the deployment
  **archives/rollback** sub directory
- We sync back the previous deployment code to the **project** directory
- We execute the rollback hook (user can input database dumps reload)
- We run the deploy procedure

Workflows
---------
Full procedure
+++++++++++++++++
- project deployment trigger procedure
- project archive procedure
- project initialization/sync procedure
- project release-sync procedure
- project configure procedure
- project build procedure
- project reconfiguration procedure
- project activation procedure
- project upgrade procedure
- project bundle procedure
- project post install procedure
- In error: rollback procedure
- In any cases (error, success):  project notification procedure

In editable mode
+++++++++++++++++
- **modified**: **editable** deployment trigger procedure
- **modified**: archive procedure is skipped
- **modified**: release-sync procedure is skipped
- project initialization/sync procedure
- project configure procedure
- project build procedure
- **modified**: project reconfiguration procedure is skipped
- project activation procedure
- project upgrade procedure
- **modified**: project bundle procedure is skipped
- project post install procedure
- In error: rollback procedure is skipped
- In any cases (error, success):  project notification procedure

In staging mode
+++++++++++++++++
- **modified**: **editable** deployment trigger procedure
- **modified**: archive procedure is skipped
- **modified**: release-sync procedure is skipped
- project initialization/sync procedure
- project configure procedure
- project build procedure
- project reconfiguration procedure
- project activation procedure
- project upgrade procedure
- project bundle procedure
- project post install procedure
- In error: rollback procedure
- In any cases (error, success):  project notification procedure

In Final mode
+++++++++++++++++
- project deployment trigger procedure
- project archive procedure
- release-sync procedure
- project initialization/sync procedure
- project configure procedure
- **modified**: project build procedure is skipped
- project reconfiguration procedure
- project activation procedure
- project upgrade procedure
- project bundle procedure
- project post install procedure
- In error: rollback procedure
- In any cases (error, success):  project notification procedure


IMPLEMENTATION: How a project is built and deployed
----------------------------------------------------
For now, at makinacorpus, we think this way:

- Installing somewhere a mastersalt master controlling compute nodes and only accessible by sysadmins
- Installing elsewhere at least one compute node which will receive project
  nodes (containers):

    - linked to this mastersalt as a mastersalt minion
    - a salt minion linked to a salt master which is probably local
      and controlled by project members

Initialisation of a cloud controller
-----------------------------------------
MANUAL and complex, contact @makinacorpus

This incude
- Setting up powerdns for the DNS configuration and multi domain stuff.
- Setting up postgres
- Setting up a basic pillar and mastersalt setup to finnish the box install
- Configuring up mastersalt to use pgsql extpillar
- Configuring up corpus.reactor and corpus.web on top of mastersalt


Request of a compute node
--------------------------------

Request of a container
--------------------------------

Initialisation of a compute node
--------------------------------
This will in order:

- auth user
- check infos to attach a node via salt cloud
- Register DNS in powerdns
  In a first time use a wildcarded DNS host on the specific endpoint target.
  Any additional dns setup (like client domain) will require some extra manual work to wire.
- generate a new ssh key pair
- install the guest_type base system (eg: makina-states.services.virt.lxc)
- Generate root credentials and store them in grains on mastersalt
- Configure the basic container pillar on mastersalt

    - root credentials
    - dns
    - firewall rules
    - defaultenv (dev, prod, preprod)
    - compute mode override if any (dev, cooking, final)

- Run the mastersalt highstate.
- Send a mail to sysadmins and initial initer with the infos of the new platform access

    - basic http & https url access
    - ssh accces
    - root credentials

Initialisation of a container environment
-----------------------------------------
This will in order:

- auth user
- Create a new container on endpoint with those root credentials
- Register DNS in powerdns
  In a first time use a wildcarded DNS host on the specific endpoint target.
  Any additional dns setup (like client domain) will require some extra manual work to wire.
- Create the layout
- use the desired salt cloud driver to attach the distant host as a new minion
- install the key pair to access the box as root
- Generate root credentials and store them in grains on mastersalt
- Configure the basic container pillar on mastersalt

    - root credentials
    - dns
    - firewall rules

- Run the mastersalt container highstate.
- Run the mastersalt container registration sls to wire the new container configuration (eg: firewall, redirections)
- We run the initalization/sync project procedure
- Send a mail to sysadmins, or a  bot, and initial igniter with the infos of the new platform access

    - basic http & https url access
    - ssh accces
    - root credentials

Initialisation of a project in staging
++++++++++++++++++++++++++++++++++++++
The code is not pull by production server it will be pushed with git to the environment ssh endpoint:

- Trigggered by a push on the remotes
- By the user itself, hence he as enougth access

In staging mode, before each build:

- we shutdown all services
- We move the **project** directory to an archive directory
- We create a new and empty **project** directory
- We run

After each build where produced files are putted inside the **project** directory, we will launch/restart/upgrade the project from there.

upgrade  of a project
+++++++++++++++++++++
The code is not pull by production server it will be pushed with git to the environment ssh endpoint:

- Triggered either by an automatted bot (jenkins)
- By the user itself, hence he as enought access

In staging mode, before each build

- we shutdown all services
- We move the **project** directory to an arcchive directory
- We create a new and empty **project** directory

After each build where produced files are putted inside the **project** directory, we will launch/restart/upgrade the project from there.

The nerve of the war: jinja macros and states, and execution modules
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Project states writing is done by layering a set of macros in a certain order.
Those macros will define and order salt states to deploy and amintain object from end to end.
The salt states and macros will bose abuse of execution modules to gather informations but also act on the underlying system.

The project common data structure
++++++++++++++++++++++++++++++++++
Overview
^^^^^^^^
- to factorize the code but also keep track of specific settings, those macros will use a common data mapping structure.
- all those macros will take as input the **configuration** data structure which is a mapping containing all variables and metadata about your project.
- this common data mapping is not copied over but passed always as a reference, this mean that you can change settings in a macro and see those changes in later macros.

Local configuration state
^^^^^^^^^^^^^^^^^^^^^^^^^^
As a project can stay in production for a while without be redeployed, we need
to gather static informations on how he got deploymed.
The previous quoted mapping should be partially and enoughtly saved to know
enought of local installation not to break it.

The project state must save:
    - all configuration variables
    - the project api_version

This must be done:

    - After a sucessful deployment
    - After a sucessful initialization
    - By calling the set_configuration method with one or more specified
      arguments in the form parameters=value

The project configuration registry execution module helper
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
The base execution module used for project management is mc_project module + all
api specific mc_project_APIN modules.
This will define methods for:

- Crafting the base **configuration** data structure
- initialising the project filesystem layout, pillar and downloading the base sourcecode for deployment (salt branch)
- deploying and upgrading an already installed project.
- Setting a project configuration

This module should know then how to redirect to the desired API specific
mc_project module (eg: mc_project_2 for the project APIV2)

If there are too many changes in a project layout, obviously a new project API
module should be created and registered for the others to keep stability.

APIV2
++++++
The project execution module interface (APIV2)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
**name** is the project name.

mc_project.init_project(name, \*\*kwargs)
    initialise the local project layout and configuration.
    any kwarg will override its counterpart in default project configuration

mc_project.deploy_project(name)
    (re)play entirely the project deployment

mc_project.get_configuration(name)
    get the local project configuration mapping

mc_project.set_configuration(name, cfg=None, \*\*kwargs)
    save a total configuration or particular configuration paramaters locally

mc_project.archive(name, \*args, \*\*kwargs)
    do the archive procedure

mc_project.release_sync(name, \*args, \*\*kwargs)
    do the release-sync procedure

mc_project.configure(name, \*args, \*\*kwargs)
    do the configure procedure

mc_project.build(name, \*args, \*\*kwargs)
    do the build procedure

mc_project.reconfigure(name, \*args, \*\*kwargs)
    do the reconfigure procedure

mc_project.activate(name, \*args, \*\*kwargs)
    do the activate  procedure

mc_project.upgrade(name, \*args, \*\*kwargs)
    do the upgrade procedure

mc_project.bundle(name, \*args, \*\*kwargs)
    do the bundle procedure

mc_project.notify(name, \*args, \*\*kwargs)
    do the notifiation procedure

mc_project.post_install(name, \*args, \*\*kwargs)
    do the post_install procedure

mc_project.rollback(name, \*args, \*\*kwargs)
    do the rollback procedure

The project sls interface (APIV2)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Each project must define a set of common sls which will be the interfaced and
orchestred by the project execution module.
**The important thing to now is that those special sls files cannot be run
without the project runner**

Indeed, we inject in those sls contextes a special **cfg** variable which is the
project configuration.

We have two sets of sls

    - set of sls providen by an **installer**

        This set can be either

        an official makina-states on
            found in the makina-states/projects/<apiver> folder

        an absolute path referenced one
            /path/to/my/installer

        a shipped via the salt install itself one (in a subdirectory)
            path/to/my/installer -> project/.salt/path/to/my/installer


    - set of sls providen by the **project**

Each sls must exists even if empty.


CLI Tools
---------
All of those commands will require you to be authenticated via a config file::

    ~/.makinastates.conf

This is a yaml configuration file::

    envnickname:
        url: <ENDPOINTURL>
        id: <dientifier
        password <password>

EG:

     prod:
        url: masteralt.foo.net
        id: someone@foo.net
        password s3cr3t
     dev:
        url: devhost.local
        id: someone@foo.net
        password s3cr3t3

Commands
+++++++++

Authenticated and distant call

- corpus computenode_list

List all available hosts to install projects onto

- corpus computenode_init <ENDPOINT> <platform_type>  [host] -> returns new platform UUID

<platform_type>
staging
prod
dev [MAY BE DEACTIVATED]
<host>
eventual host selection

create a container/vm to deploy our future project

- corpus computenode_switchmode <ENDPOINT> <ENV_UUID> <operation_mode>

Request for the sitch of an operation mode to another

- corpus computenode_init <ENDPOINT> <platform_type> [host] [space separted list of guest types]-> returns new platform UUID

<platform_type>
staging
prod
dev [MAY BE DEACTIVATED]
<host>
eventual host selection

request for the link of an host for container/vm to deploy our futures guests

- corpus computenode_infos <ENDPOINT> <ENV_UUID>

List for a specific compute node tenant

        - available guest slots
        - a list of slots with the number at a minium and hence we have access
          the guests metadatas

- corpus project_create <API_ENDPOINT> project <- return uuid

    Create a new project to link containers onto

- corpus guest_create <API_ENDPOINT> guest <- return guest_id

    Create a new guest to push project code onto

- corpus push <ENDPOINT> <guest_id> <project>
    deploy our future project

    This will in turn:

        - push the pillar code
        - push the salt code triggering the local deploy hook

- corpus guest_delete <API_ENDPOINT> <guest_id>

  Delete a guest

- corpus project_destroy <API_ENDPOINT> <UUID_ENV> <project>

  Destroys and free any project resources on a located endpoint

- corpus trim <API_ENDPOINT> <UUID_ENV> <guest> <size>

  Remove <size> from project storage disk usage.

- corpus enlarge <API_ENDPOINT> <UUID_ENV> <guest> <size>

  Resize the project storage size with <size>

For now size is not configurable and will be fixed at 5gb

.. _docker:  http://docker.io
.. _heroku: http://heroku.com/
.. _dheroku: https://devcenter.heroku.com/articles/buildpack-api
.. _openshift: https://www.openshift.com/developers/deploying-and-building-applications
