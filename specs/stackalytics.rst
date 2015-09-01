::

  Copyright 2015 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Host Stackalytics Service
=========================

https://storyboard.openstack.org/#!/story/2000274

Host the http://stackalytics.com Stackalytics Service within the
community-managed project infrastructure.

Problem Description
===================

The http://stackalytics.com Statalytics Service is currently maintained by
Mirantis, Inc and has become the de-facto standard stats reporting tool for
the OpenStack community. Mirantis staff have agreed the best home for this
service is within the community hosting infrastructure rather than under the
control of a single member company.

Proposed Change
===============

Create a puppet-stackalytics module and convert the running site into
puppet.  We should check with Mirantis if anything has already been
created.  A medium VM will be required since there will be cron job needed
for the stackalytics-processor.

Alternatives
------------

Allow Mirantis to continue providing support for a community resource.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  pabelanger

Intrastructure root shepherd:
  mordred
  SergeyLukjanov

Gerrit Topic
------------

Use Gerrit topic "puppet-stackalytics" for all patches related to this spec.

.. code-block:: bash

    git-review -t puppet-stackalytics

Work Items
----------

A puppet-stackalytics puppet module will needed to be created, some bits also
to be added into openstack-infra/system-config.  Finally a new node will need
to be launched for the stackalytics.openstack.org server.

Additionally, stackforge/stackalytics should be moved into
openstack-infra/stackalytics. Allowing for more contributors to support the
project.

Within stackalytics, default_data.json should be moved out of the project and
into openstack-infra/project-config.

Repositories
------------

A openstack-infra/puppet-stackalytics repo will be created.

Servers
-------

A stackalytics.openstack.org server will need to be created. No other existing
server should need to be modified.

DNS Entries
-----------

The stackalytics.openstack.org A and AAAA resource records will need to be
created. The stackalytics.com domain should be redirected to the new
stackalytics.openstack.org server.

Documentation
-------------

The openstack-infra/system-config documentation will be updated to include
a summary of the Stackalytics Service.

Security
--------

The Stackalytics Service is a static service driven from git. It will live
on a dedicated system with the ability to grow in the future.

Testing
-------

The existing puppet apply/integration jobs.

Dependencies
============

- https://review.openstack.org/#/c/187645/
- https://review.openstack.org/#/c/187269/
