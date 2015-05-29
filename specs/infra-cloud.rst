::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===========
Infra-cloud
===========

Story: https://storyboard.openstack.org/#!/story/2000175

With donated hardware and datacenter space, we can run an optimized
semi-private cloud for the purpose of adding testing capacity and also
with an eye for "dog fooding" OpenStack itself.

Problem Description
===================

Currently all of the test resources that we use are provided by public
clouds.  This is very useful to us and is also a good demonstration of
a cross-public-cloud OpenStack application.  Some organizations are
also able to provide hardware instead of (or in addition to) public
cloud resources.  By operating that hardware as a private cloud with
only OpenStack Infrastructure as a tenant, we can expand our test
capacity and also demonstrate a public-private hybrid OpenStack
application.

Further, we can operate the cloud in a completely transparent manner
as we do the rest of the project infrastructure and help bridge the
gap between developers and operators.

Proposed Change
===============

This spec describes the process of standing up the initial
infra-cloud, but intentionally does not delve into technical detail.
Many of those decisions will need to be made and updated as the
process unfolds, and also need to be recorded as system documentation.
Therefore, most of the actual technical decisions and documentation
will happen in the system-config repository in the
doc/source/infra-cloud.rst file.

In order to accept a donation of hardware from an organization, we
will also need to be provided a contact from the organization that can
help us with any hands-on work needed to maintain the machines.  Newly
donated hardware will be inventoried and standardized, and then an
infra-cloud region will be deployed on it, as described in
system-config.

We have an initial hardware donation from HP in two data centers,
which we will stand up as two clouds.  Once these clouds are well
established, we can consider adding new clouds based on further
donations as needed.  We may consider running a single centralized
keystone in the future and combine the separate clouds into one cloud
with multiple regions.

Infra-cloud is run like any other infra managed service. Puppet
modules and Ansible do the bulk of configuring hosts, and Gerrit code
review drives 99% of activities, with logins used only for debugging
and repairing the service.

Our CI system itself is fault-tolerant across clouds, so no individual
region of infra-cloud (or even infra-cloud as a whole) should be
considered a critical piece of infrastructure.  There is no uptime
guarantee and in the case of any error, we should be content to
operate the CI system without part or all of infra-cloud until the
error can be corrected in due course.

In order to focus on our initial deployment goals, we are strictly
limiting the scope of infra-cloud.  In particular it is not a general
purpose cloud to provide services to any user other than the project
infrastructure, and it is not intended to provide a special test
environment (e.g., bare metal) not otherwise provided by public
clouds.  It is also not intended as a test system for OpenStack
itself; the update frequency of the version of OpenStack deployed on
infra-cloud is not defined.  We may deploy new versions of OpenStack
as needed, or we may continue to run a stable version for a length of
time.

Alternatives
------------

Continue to use only externally provided clouds.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  SpamapS

Gerrit Topic
------------

Use Gerrit topic "infra-cloud" for all patches related to this spec.

.. code-block:: bash

    git-review -t infra-cloud

Work Items
----------

* Normalize HP hardware
* Agree on initial deployment choices (in system-config)
* Write puppet implementation
* Deploy
* Begin use in nodepool

Repositories
------------

No new repos are currently anticipated.

Servers
-------

Many, as specified in system-config documentation.

DNS Entries
-----------

A DNS entry should be registered for each Keystone auth endpoint.
Other individual servers may also get their own DNS entries.

Documentation
-------------

The system should be documented from before implementation and kept up
to date in system-config.

Security
--------

The only tenant will be OpenStack Infrastructure, and the tenant
credentials will be managed in the normal way (using hiera) for CI
clouds.  The administrative credentials will be similarly managed.  We
would like to make a considerable amount of operational logging
available publicly.  We will need to be concerned about leaking
credentials through that process.

Testing
-------

If nodepool works with it, it's good.  We could run tempest or
refstack against it as well if we want.

Dependencies
============

The technical decisions will be made in the system-config repository.
