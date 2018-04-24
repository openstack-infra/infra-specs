.. code-block:: text

  Copyright 2018 Red Hat

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===========================
Post Zuul v3 Third Party CI
===========================

Our 3rd Party CI solutions have not quite kept up with the pace of
Zuul V3 development.  OpenStack Infra should provide 3rd Party CI
operators with guidance on how to incoporate these new technologies.

Problem Description
===================

There are many external operators who run external
continuous-integration systems against projects hosted within the
OpenStack Infra gerrit.  These have traditionally been referred to as
"Third Party CI".

The reasons are as varied as the operators, but usually involve
testing software or hardware components that are not suitable for
inclusion in the upstream infrastructure testing environments.

After the acceptance of the "Common OpenStack CI Solution" spec
(:doc:`openstackci`), effort was put into the `puppet-openstackci
<https://git.openstack.org/cgit/openstack-infra/puppet-openstackci/>`__
repository.  This acts as an abstraction layer between the OpenStack
Infra specific configuration, kept in `system-config
<https://git.openstack.org/cgit/openstack-infra/system-config/tree/>`__,
and the more specific puppet modules which install and manage
indivdual services.

.. blockdiag::

   diagram {

   sc [label = "system-config"];
   posci [label = "puppet-openstackci"]
   p1 [label = "puppet-nodepool"]
   p2 [label = "puppet-zuul"]
   p3 [shape = "dots"]
   p4 [label = "puppet-foo"]
   tpci [label = "3rd Party"]

   sc -> posci;
   tpci -> posci;

   posci -> p1;
   posci -> p2;
   posci -> p3 [style = "none"];
   posci -> p4;
   }

Operators have largely been following the instructions based in
`<https://docs.openstack.org/infra/openstackci/>`__ which promises,
via the main entry-point of `single_node_ci.pp
<http://git.openstack.org/cgit/openstack-infra/puppet-openstackci/tree/manifests/single_node_ci.pp>`__,
to provide a functional zuul and jenkins CI setup.

Unfortunately with the removal of Jenkins and the introduction of
nodepool and zuul v3, these instructions have become out of date.

The OpenStack infrastructure team should provide guidance to 3rd Party
CI operators about how to utilise this new environment.

Proposed Change
===============

`Windmill <https://git.openstack.org/cgit/openstack/windmill/>`__ is a
series of ansible roles for the deployment of nodepool, zuul, gerrit.
It uses a range of ``openstack/ansible-role-*`` projects to achieve
this; it is roughly analogous to `single_node_ci.pp
<http://git.openstack.org/cgit/openstack-infra/puppet-openstackci/tree/manifests/single_node_ci.pp>`__
for puppet.

We will focus on on creating a playbook(s) and related documentation
that does an all-in-one setup that focuses on 3rd party CI specifics.

Pros
----

* Gating across Fedora, Xenial (soon bionic) and possibility of other
  platforms too.
* Roles are generic enough anyone can build out a zuul stack
* Currently using well tested path of pypi and OS packages

Cons
----

* About 90% done
* Propper documentation to come
* Would need further updates for dealing with containerisation of services
* OpenStack Infra is not directly running these services in
  production, which leaves windows for issues to creep in.

Implementation
==============

Assignee(s)
-----------

TBD

Gerrit Topic
------------

TBD

Work Items
----------

To declare this spec as complete will require work across a
combination of documentation, Windmill and related projects.

The target audience is defined as an experienced Linux administrator,
with basic familiarity as a user of OpenStack clouds, Gerrit and
Ansible.

3rd Party CI will require

* A single (Ubuntu-based?) host to run Zuul, nodepool and
  nodepool-builder
* Access to an OpenStack cloud

  * ability to upload images
  * networking
  * resources to run at least one testing node

For success, they should be able to complete the following with a
combination of documentation and using the provided tools:

* Register an account for use with Gerrit
* Have an environment for running Windmill.

  * Setup a virtualenv with ansible and windmill+dependencies
  * Containers to be added

* Description of initially required playbook configuration
* Deploy requirements such as Zookeeper and gearman
* Use Windmill roles to create a single-node nodepool environment

  * nodepool-builder should be able to build a Xenial image
  * image uploaded to target cloud
  * node ready in nodepool

* Have a Zuul merger and executor running
* Start zuul scheduler and connect it to gerrit listening for changes
  on a single project of interest
* Interact with zuul-web to see status
* Trigger a single job on changes to monitored project
* Be able to report results to upstream gerrit

Repositories
------------

TDB

Servers
-------

Unlikely

DNS Entries
-----------

Unlikely

Documentation
-------------

The referenced 3rd Party CI documentation will need to be updated.

Security
--------

No

Testing
-------

Highly dependent on solutions.


Dependencies
============

TBD


Alternatives Discussed
======================

Initial spec reviews proposed a number of different alternatives.
Whilst they are no mutually exclusive with Windmill, that will be our
focus.  The alternatives are presented below.

Update OpenstackCI
------------------

We could direct effort to updating the OpenStackCI puppet module
described above to enable installation of zuulv3, nodepool and other
requirements such as zookeeper.

This would take a significant amount of puppet expertise and involve
"detangling" some parts where the ``system-config -> openstackci ->
puppet-*`` abstraction may have broken down slightly during
development.

Pros
----

* It (sort of) keeps the status-quo for people using puppet to deploy

Cons
----

* We're moving away from puppet, in general
* We've moving away from using puppet to deploy nodepool/zuul in
  openstack infra, in specific
* Who would actually do this?

Software Factory
----------------

`Software Factory <https://softwarefactory-project.io/docs/>`__ is an
out-of-the-box open source solution that can be used for 3rd party CI.

Pros
----

* all-in-one package
* can do 3rd party CI, but much more too
* very active project

Cons
----

* limited to CentOS deployments

Zuul-from-scratch
-----------------

Zuul provides a `Zuul From Scratch
<https://docs.openstack.org/infra/zuul/admin/zuul-from-scratch.html>`__
document that describes how to configure a zuul and nodepool
environment that can talk to gerrit.  It does not provide automation
such as puppet modules or ansible tasks.

We could show no preference to these deployment mechanisms and suggest
people implement the broad installation instructions as they feel.

These documents would need to be enhanced to cover details on setting
things up specifically for 3rd party on OpenStack's Zuul instance.

Pros
----

* Nothing to maintain

Cons
----

* Leaves a lot of work in the hands of potential users
* No possibility of CI/CD approach to ensure correctness


Something to deploy container images
------------------------------------

Infra is moving to containerised services

`<https://specs.openstack.org/openstack-infra/infra-specs/specs/update-config-management.html>`__

As at September 2018, work in this area is just beginning with the
`pbrx <https://pbrx.readthedocs.io/en/latest/readme.html>`__ container
builds.

We could put 3rd Party CI direction setting on hold for a short period
while this work bootstraps itself and the infra project gains some
experience running containerised CI services.  We could then offer
this externally.

Pros
----

* Containers
* Would only target the container runtime, rather than a wider range
  of platforms of interest to 3rd parties.

Cons
----

* Very new, so progress would likely come after infra have acquired
  better experience running containers.

Maintain modules, but no centralised driver
-------------------------------------------

Specific modules such as ``puppet-nodepool``, ``puppet-zuul``,
``puppet-zookeeper`` and equivalent ``ansible-role-*`` projects for
those that prefer ansible are provided.  The canonical reference for
use of these modules is ``system-config``; which retains its status as
a non-generic "top-level" specific to OpenStack Infra deployment.

We could support individual deployment modules, however not attempt to
maintain a generic driver for them.

This is effectively the current situation.

Pros
----

* ?

Cons
----

* Unlikely anyone could reasonably create a working system out of this
