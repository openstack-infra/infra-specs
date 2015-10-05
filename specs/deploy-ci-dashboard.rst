::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Host a CI systems monitoring dashboard
======================================

https://storyboard.openstack.org/#!/story/2000303

We need a way to be able to quickly see if a CI system is failing,
or passing incorrectly, any given patch pushed to gerrit for review.
As a service to our developers and project communities, we need to
provide a service that does this gathering and display of CI system
status.

Problem Description
===================

There is no convenient place where you can:
  * If you are an operator, gage how well your system is doing;

    * which patches your CI commented on or missed, with the result;
    * which jobs are unstable;
    * upstream jenkins health.

  * If you are a developer, establish a level of trust so you can better focus
    your time on code;

    * get a quick indication of a third-party CI past history.

  * If you are a PTLs: see who's got CI and who doesn't;

Note: This list is not complete, and this spec is not aimed at properly
defining the requirements.

Proposed Change
===============

Deploy an existing dashboard that offers enough functionality to cover basic
third-party CI monitoring requirements. This dashboard is called CI Watch
and the source code can be found in the Third Party CI Working Group repository.
https://git.openstack.org/cgit/stackforge/third-party-ci-tools/tree/monitoring/ciwatch

We would create a puppet module to deploy the CI Watch dashboard and serve it
from a dedicated virtual machine under care of the OpenStack Infrastructure
Project. A proof-of-concept deployment is running at
http://ci-watch.tintri.com/project?project=nova

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mmedvede

Gerrit Topic
------------

Use Gerrit topic "ci-dashboard" for all patches related to this spec.

.. code-block:: bash

    git-review -t ci-dashboard

Work Items
----------

1. Initiate a change in ``openstack-infra/project-config`` to create
   an empty ``openstack-infra/puppet-ciwatch`` Git repository as
   described in the `Repository Creator's
   Guide <http://docs.openstack.org/infra/manual/creators.html>`_.
   This should be accompanied by a related change to
   ``openstack/governance`` adding the new repo as part of the
   Infrastructure project.
2. Create a new ``ciwatch`` Puppet module in
   ``openstack-infra/puppet-ciwatch`` which enables an opinionated
   and working deployment of the service. Existing
   ``openstack-infra/puppet-.*`` should be examined for inspiration
   and consistency.
3. A change will be proposed to the
   ``openstack-infra/system-config`` Git repo lightly documenting
   deployment and management of the service and adding that document
   to the ``doc/source/index.rst`` file. Examples in the
   ``doc/source`` directory can be used for inspiration and
   consistency of style. This same change can also add a
   ``ci-dashboard.openstack.org`` node to the global
   ``manifests/site.pp`` file along with (if needed) a
   ``modules/openstack_project/manifests/ci-dashboard.pp`` class file
   and any accompanying custom config files or templates.
4. Deployment of the above changes will be performed by a root admin
   onto a new ``ci-dashboard.openstack.org`` server, and related A,
   AAAA and PTR resource records will be created in DNS for it.
5. Create a launchpad account ``ci-dashboard`` for listening to the gerrit
   system ssh stream as described in the `Developer's Guide
   <http://docs.openstack.org/infra/manual/developers.html>`_.
6. If necessary, use of the service can be documented in the
   ``openstack-infra/infra-manual`` repo as well.
7. Once the service appears stable and working as intended, announce
   it to the
   `openstack-dev <http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-dev>`_,
   `openstack-infra <http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-infra>`_
   and
   `openstack-operators <http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-operators>`_
   mailing lists.

Repositories
------------

An ``openstack-infra/puppet-ciwatch`` Git repo will be created as part
of this plan.

Servers
-------

A ``ci-dashboard.openstack.org`` server will be added.

DNS Entries
-----------

A, AAAA and PTR resource records will be created for the
``ci-dashboard.openstack.org`` server.

Documentation
-------------

The service's deployment and management will be documented in the
``openstack-infra/system-config`` repo, published to the
http://docs.openstack.org/infra/system-config/ site. Optionally, use
of the service can be documented in the `Infrastructure
Manual <http://docs.openstack.org/infra/manual>`_.

Security
--------

This is not a trusted service, needs no authentication for normal
use, and it runs on its own dedicated virtual machine. HTTPS should
not be necessary for this service, so no X.509 certificate will be
ordered.

Testing
-------

The configuration management for this service will be tested via
existing apply/syntax CI jobs.

Dependencies
============

- None identified.
