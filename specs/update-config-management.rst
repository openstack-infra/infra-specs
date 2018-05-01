::

  Copyright 2018 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

========================
Update Config Management
========================

Puppet 3 has been EOL since December 31, 2016.

We are also increasingly finding ourselves in a position where our
configuration management is a master we serve rather than a force-multiplier
to help us get things done quickly and more repeatably.

Since our current system was designed, we've also grown capabilities in our
CI infrastructure that we're ironically unsuited to fully take advantage of.

Our CI jobs are now written in Ansible, which means we have a large number
of people who are interfacing with Ansible on a regular basis making switching
to Puppet a mental context switch.

We tend to run a large amount of our software CD and often either from our
own sources or from a third party in such a manner that the traditional value
of installing software that comes from the distro channel is not as great as
it once was.

Problem Description
===================

Our current system uses Ansible to orchestrate the running of Puppet. It
has grown organically since it was first put in place back in 2011 and it
is truly amazing it still works as well as it does.

However, with the advent of Ansible-based job content in Zuul v3, we are
regularly writing large amounts of Ansible in service of OpenStack, so the
cognitive shift back to implementing support for services in Puppet feels
onerous. In our current system, we aren't taking advantage of the power
that Zuul's Ansible integration affords us.

We currently have an awkward hybrid system with some information in
Ansible inventory and some information in Puppet hiera with host group
generation based on a template file.

A majority of the services we deploy end up being compiled from source on
production servers with the help of various development toolchains.

While we could start systematically packaging our services and run a
distribution mirror, it incurs a significant overhead we'd prefer to avoid.

More and more of our systems are multi-node distributed systems which are
not well served by Puppet.

We do have a very large corpus of existing Puppet, so an all-in-one change to
anything is unreasonable.

Proposed Change
===============

OpenStack Infra should migrate its control plane from Ansible-orchestrated
Puppet to Ansible orchestrated Container-based deployments. Doing that has
three fundamental pieces:

* Upgrade the existing Puppet to Puppet 4 (and possibly Puppet 5 depending
  on how long other tasks take).
* Config management and orchestration should migrate from Puppet to Ansible.
* Software installation should migrate to container images built in Zuul
  jobs.

Puppet 4
--------

Upgrading to Puppet 4 gives us additional time to deal with the other larger
changes while not feeling pressure due to the Puppet 3 EOL.

We need to complete and enhance the puppet module functional tests so that they
are easier to manage and so they are capable of installing and validating
services with puppet 4.

When we are confident that all the modules needed for a given site.pp node work
with puppet 4, upgrade that node to puppet 4. We'll track that a node should
be running puppet 4 by adding a new puppet-4 group in groups.txt and adding
nodes to it one at a time. A new playbook will be written that runs an
idempotent upgrade for those nodes.

Ansible
-------

We should update to using at least Ansible 2.5 and the OpenStack Inventory
plugin instead of the exisitng inventory script. Inventory plugins are
stackable, so we should be able to rework the group membership and emergency
file system to be a collection of Inventory plugins instead of a static
generation script.

We should shift away from the ``run_all.sh`` model and move instead to
per-service ansible playbooks. The first version of the per-service playbooks
can simply call puppet as the existing playbook does. As we write these, we
should make individual cron jobs for each playbook as each of their running
does not depend on the other.

We should then replace the cron jobs with Zuul jobs in
``openstack-infra/system-config``. Those jobs should use ``add_host`` to
add ``puppetmaster.openstack.org`` with the secret key for ssh in a Zuul
secret. Using a secret and ``add_host`` will ensure the jobs can't be used by
other projects. Since the job will use ``add_host`` for puppetmaster, the job
itself can be nodeless, which should ensure we don't have issues running
deployment jobs while under periods of high build traffic.

We should `add a central ARA instance`_ that ansible is configured to log to
when run on puppetmaster. That way sysadmins running ansible by hand on
puppetmaster and Zuul-driven jobs will both log to a consistent place. The
ssh key for a zuul user account on the puppetmaster host can be stored in
Zuul as a secret. As a future improvement, improving ARA to be able to
export reports from one ARA and import them into a second ARA could allow us
to log Zuul-driven playbook runs to a per-job ARA - as well as have that same
report data go into the central ARA.

We should migrate the ``openstack_project::server`` base puppet pieces to
Ansible roles in the ``openstack-infra/system-config`` repo. This currently
involves creating users, setting timezone to UTC, setting up rsyslog,
configuring apt to retry and not pull translations, setting up ntp, setting
up the root ssh account for ansible management, setting up snmp, disabling
cloud-init and uninstalling some things we don't need. There are options for
installing the AFS client, managing exim, enabling unbound and managing
iptables rules that should just be turned in to roles and included in the
playbooks for a given service. Similarly, we install pip/virtualenv in
``openstack_project::server``. We should be able to just stop doing that since
we'll be shifting from installing things directly on the system with pip to
installing them in containers, although we still want to put it in the
ansible version of ``openstack_project::server`` so that we can transition
our puppet services one at a time.

We should keep the roles in the ``roles`` directory of
``openstack-infra/system-config`` for the time being. While we might want to
eventually split things out into role repositories in the future, there is
enough complication related to an in-place CD transition from puppet to ansible
without over-organizing at the outset.

Once we have per-service playbooks and base server roles, we can begin to
rework the services to be native Ansible one service at a time.

As we work on per-service Ansible, We should shift our secrets from hiera
to an Ansible inventory based set of host and group vars. They can continue to
be yaml files in a private git repo - and in fact the current structure may
even work directly. But rather than having ansible copy some variables to the
remote host so that the local puppet apply has access to the hiera data,
if the code being run is actual ansible roles and playbooks we can just have
Ansible use the secrets as variables and stop copying secret chunks to remote
hosts completely. Cleaning up and organizing the existing hiera data first is
likely a good idea.

We may want to investiagate use of Ansible Vault to store the secrets with
GPG for encrypting/decrypting secrets. The GPG private key for zuul can be
stored as a Zuul secret, and we can encrypt things for the union of Zuul and
infra-root. However, this would be more than what we're doing currently with
hiera, so should be considered a future improvement. If we shift from
``add_host`` for adding puppetmaster to using the static driver for
puppetmaster, then we may want to consider shifting to protecting the secrets
using vault with a GPG key in a Zuul secret. Doing so would be a
belt-and-suspenders for protection against the node being used in the wrong
context.

On a per-service basis, as we migrate from Puppet to Ansible, we may find
that updating to installing the software via containers at the same time is
more straightforward than breaking it into two steps.

Containers
----------

We should start installing the software for the services we run using
thin per-process containers based on images that we build in the CI system.

We should build and run those containers using Docker. We should install
Docker from the upstream Docker package repository.


Adoption of container technology can happen in phases and in parallel to the
Ansible migration so that we're not biting off too much at one time, nor
blocking progress on a phased approach. There is no need to go overboard
needlessly. If a service doesn't make sense in containers, such as potentially
AFS, we can just run those services as we are running them now except using
Ansible instead of Puppet. Services like AFS or exim, where we're installing
from distro packages anyway are less likely to see a win from bundling the
software into containers first. On the other hand, services where we're
installing from source in production like Zuul, or building artifacts in CI
like Gerrit (nearly all of our services) are the most likley to see a win and
should be focused on first.

Building container images in CI allows us to decouple essential dependency
versions from underlying distro releases. Where possible, we should prefer to
use ecosystem-specific base images rather than distro-specific base images.
For instance, we should build container images for each zuul service using the
``python:3.6-slim`` base image with Python 3.6, a container for etherpad using
the ``nodejs`` base image with the correct tag version of node and a container
for Gerrit with the ``openjdk`` base image. For our Python services, a new tool
is in work, `pbrx`_, which has a command for making single-process containers
from pbr setup.cfg files and bindep.txt.

The container images we make should be single-process containers and should
use `dumb-init`_ as an Entrypoint so that signals and forking work properly.
This will allow us to start building and using containers of various pieces
of software only by changing the software installation and init scripts even
while config files, data volumes and the like are still managed by puppet.
Config files and data volumes will be exposed to the running container via
normal bind mounts. Something like:

.. code-block:: console

  docker run -v /etc/zuul:/etc/zuul -v /var/log/zuul:/var/log/zuul zuul/zuul-scheduler

By doing this, we'll still have config files and log files in locations we
expect.


Our services are all currently designed with the assumption that they exist
in a direct internet networking environment. Network namespacing is not a
feature that provides value to us, so we should run docker with
``--network host`` to disable network namespacing.

We also have a need to run local commands, such as ``zuul enqueue``. Those
commands should all exist in the containers, so something like:

.. code-block:: console

  docker run -it --rm zuul/zuul -- enqueue

would do the trick, but is a bit unwieldy. We should add wrapper scripts to
the surrounding host that allow us to run utility scripts from the services
as needed. So that a ``/usr/local/bin/zuul`` script would be:

.. code-block:: console

  docker run -it --rm zuul/zuul -- $*

Generating those scripts could be a utility that we add to `pbrx`_ - or it
could be an Ansible role we write.

Alternatives
------------

- Stay on puppet 3 forever
- Stay on puppet forever but upgrade it
- Migrate to ansible but without containers
- Building distro packages of our software
- Use software other than docker for containers

There are alternative tools for building and running containers that can be
explored. To keep initial adoption simple, starting with Docker seems like the
best bet, but alternate technology can be explored as a follow on. The Docker
daemon is a piece of operational complexity that is not required to use Linux
namespaces.

For building images we (or `pbrx`_) can use Ansible playbooks or `img`_ or
`buildah`_ or `s2i`_. For running containers we can look at `rkt`_ or
`podman`_. Since `rkt`_ and `podman`_ follow a traditional fork/exec model
rather than having a daemon, we'd want to use systemd to ensure services run
on boot or are restarted appropriately. As we start working, it may end up
being an easier transition from systemd-process to systemd-podman-container
than to transition from systemd-process to docker-container to
systemd-podman-container.

If, in the future, we deploy a Container Orchestration Engine such as
Kubernetes, we'll should consider running it with `cri-o`_ to avoid the Docker
daemon on the backend.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mordred
  Colleen Murphy <colleen@gazlene.net>
  infra-root

mordred can help get it going, but in order for it to be successful, we'll
all need to be involved. Coleen has already done more of Puppet 4.

Gerrit Topic
------------

Use Gerrit topic "update-cfg-mgmt" for all patches related to this spec.

.. code-block:: bash

    git-review -t update-cfg-mgmt

Work Items
----------

Puppet
~~~~~~

#. Complete and enhance puppet module functional tests.

##. We need to ensure all modules have proper functional tests that at least
perform a basic smoke test.

##. The functional tests need to accept a puppet version parameter.

##. An experimental functional test job needs to be added to use the puppet
version parameter. The job should be graduated to non-voting and then to gating.

#. Audit all upstream modules in modules.env for version compatibility and take
steps to upgrade to a cross-compatible version if necessary.

#. Turn on the future parser in puppet.conf on all nodes in production. The
future parser will start interpreting manifests with puppet 4 syntax without
actually having to run the upgrade yet.

#. Enhance the install_puppet.sh script for puppet 4

##. The script already installs puppet 4 when PUPPET_VERSION=4 is set. Since
this script is currently only run during launch-node.py and not periodically, we
do not need to worry about the script accidentally downgrading puppet at some
point after the upgrade. However, in the event things go wrong and we want to
revert a node back to puppet 3, we need to be able to manually run the script
again to forcefully downgrade, so we most likely need to enhance the script to
ensure this works properly.

#. Write ansible logic to notate nodes that should be running puppet 4 and run
the upgrade.

## The playbook will need to run the install_puppet.sh script with
PUPPET_VERSION=4.

Ansible
~~~~~~~

#. Split run_all.yaml into service-specific playbooks.
#. Rewrite ``openstack_project::server`` in Ansible (infra.server).
#. Add a playbook targetting hosts: all that runs infra.server.
#. Either add "install docker" to infra.server or make an ansible hostgroup
   that contains it.
#. Either rewrite launch_node.py to bootstrap using infra.server or ensure we
   can use ansible-cloud-launcher instead.
#. Install a local container registry service as our first docker-based service.

On a service by service basis:

#. Add a Zuul job to build the software into container(s) and publish the
   containers into our local container registry (and to dockerhub)
#. Translate the puppet for the service into ansible that runs the software from
   the container.
#. Add a Zuul job that runs the new ansible with the container for testing.
#. Change the service's playbook to use the ansible/container deployment.
#. Retire the service's puppet.

Repositories
------------

We may need to create some repositories as a place to put
jobs/roles/Dockerfiles for services where we aren't tracking a git repo locally
already. For instance, etherpad doesn't have a great place for us to put
things.

When we're done, we'll have a LOT of ``puppet-.*`` repos that we will no longer
care about. We should soft-retire them, leaving them in place for people still
depending on them.

Servers
-------

All existing servers will be affected.

DNS Entries
-----------

Not explicitly.

Documentation
-------------

All of the Infra system config documentation on how to manage puppet things
will need to be rewritten. OpenStack developers should not notice any changes
in their daily workflow.

Security
--------

This change should help improve security since we'll be getting security
updates to Puppet.

We are not proposing using containers for any increased isolation at this point
other than as a build step and convenient software installation vehicle.
However, building container images in CI and then deploying them means we will
need to track software manifests of the built images so that we can know if
we need to trigger a container rebuild due to CVEs.

We should make sure we have a mechanism to trigger a rebuild / redeploy.

We could **also** periodically rebuild / redeploy our service containers just
in case we miss a CVE somewhere.

The playbooks will be run by Zuul on puppetmaster using the secrets system to
protect the private ssh key. Normal infra-core reviews in system-config should
be sufficient to protect this.

Testing
-------

Puppet 4 is already validated with the puppet-apply noop tests and this spec
proposes enhancing the module functional tests before proceeding with the
upgrade.

As we shift to Ansible, the functional tests for puppet need to to be shifted
as well. We should use `testinfra`_ for our Ansible testing.

We're currently using `serverspec`_ with our Puppet. However, `serverspec`_ is
Ruby-based which is additional context for admins to deal with and carrying
that additional context with a shift to Ansible seems less desirable.

`testinfra`_ is python-based, so fits with the larger majority of our
ecosystem, but will require us to write all new tests. It has ansible, docker
and kubectl backends, so should allow us to plug in to things where we'd like
to. It is implemented as a **py.test** plugin, which has a different
test-writing paradigm than we are used to with **testtools**, but the context
shift there is still likely less than the python to ruby context shift.

On a per-service basis, as we transition a service from Puppet to Ansible, we
should the deploy playbooks such that they can be run in Zuul. We should then
make jobs that use those playbooks against nodes in the jobs and then run
`testinfra`_ tests to validate the playbooks did the right thing.

Since the rest of our testing is **subunit** based, we may want to pick up
the work on `pytest-subunit`_.

Dependencies
============

Concurrent with `add a central ARA instance`_.

Docker will need to be installed and we'll want to decide if we want to use
the distro-supplied Docker or install it more directly from Docker upstream.

We'll need to run a container registry into which we can publish our container
images so that we are not dependent on hub.docker.com to update our system.
We should still publish our containers to hub.docker.com as well.

References
==========

- `Puppet Inc Upgrade Announcement <https://docs.puppet.com/upgrade>`_
- `Puppet 4 Release notes <https://docs.puppet.com/puppet/4.0/release_notes.html>`_
- `Features of the Puppet 4 Language <https://www.devco.net/archives/2015/07/31/shiny-new-things-in-puppet-4.php>`_

.. _`add a central ARA instance`: https://review.openstack.org/527500/
.. _`Puppet 4 Preliminary Testing spec`: http://specs.openstack.org/openstack-infra/infra-specs/specs/puppet_4_prelim_testing.html
.. _dumb-init: https://github.com/Yelp/dumb-init
.. _openshift-ansible: https://github.com/openshift/openshift-ansible
.. _oc cluster up: https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md
.. _cri-o: http://cri-o.io/
.. _pbrx: http://git.openstack.org/cgit/openstack/pbrx
.. _img: https://github.com/genuinetools/img
.. _buildah: https://github.com/projectatomic/buildah
.. _s2i: https://github.com/openshift/source-to-image
.. _rkt: https://coreos.com/rkt/
.. _podman: https://github.com/projectatomic/libpod
.. _testinfra: https://testinfra.readthedocs.io/en/latest/
.. _serverspec: https://serverspec.org/
.. _pytest-subunit: https://github.com/lukaszo/pytest-subunit
