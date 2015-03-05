::
  Copyright 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

============================
Common OpenStack CI Solution
============================

https://storyboard.openstack.org/#!/story/2000101

Rather than having every Third Party CI operator create and manage their
own Third Party CI Solution, let's create a simpler version of what the
OpenStack Infrastructure team uses that also fits the needs and scale of Third
Party CI operators.

Problem Description
===================

There are over 100 Third Party CI accounts, presumably each with their own Third
Party CI solution. Many of them are closed-source, a few are open-source hosted in
personal repositories such as github, and as a whole, require significant
effort to share and maintain.

Jay Pipes created an open source version hosted in GitHub, but it is no longer
being actively maintained. (https://github.com/jaypipes/os-ext-testing).
A few others have forked his repository, including one by Ramy Asselin who is
attempting to keep one up-to-date with many of the recent changes in the
dependent openstack-infra repositories. However, since they are out of tree, updates are
done after the system breaks rather than before.

Part of the reason why the system breaks is that, in the case of the Jay Pipe's
forks, these CI systems use a copy-paste-customize approach to reuse many
of OpenStack Infrastructure's puppet scripts, while using others directly via
installed modules. This eventually leads to inconsistencies that have to be
manually reconciled.

Proposed Change
===============

Refactor the existing OpenStack Infrastructure CI puppet scripts to be reusable by
Third Party CI. These puppet scripts are located in
http://git.openstack.org/cgit/openstack-infra/system-config/tree/modules/openstack_project

Create a new puppet module to host the refactored and reusable puppet scripts
such that they are consumable by both the OpenStack Infrastructure CI and
Third Party CIs. The puppet scripts already support a project-config plugin
(using puppet-project_config) to pass in custom jobs, nodepool, and zuul
configuration. In other areas, make use of heira and puppet
arguments as appropriate.

Add tests to validate an OpenStack CI setup can all be successfully installed and
run on a single VM.

Alternatives
------------

* Continue to individually maintain numerous independent Third Party CI
  solutions out-of-tree.
* Only refactor the OpenStack-infra CI puppet scripts to be reusable and leave them
  in the system-config project.

Implementation
==============

Third Party CI currently uses the following components:
  - Apache
  - Jenkins
  - Jenkins Job Builder
  - Zuul
  - Nodepool
  - Log Server / os-loganalyze
  - Logstash / kibana (optional)

Create a new project and puppet module, puppet-openstackci, in openstack-infra that
contains the puppet scripts necessary to install and configure the above
components.

The OpenStack Infrastructure CI & Third Party CI operators will reuse these
puppet script to manage their Third Party CI system, along with a privately
maintained Hiera configuration and site
specific fork / non-fork replica of openstack-infra/project-config plugged in via
puppet-project_config.

Create a puppet script that integrates all the components on a single node.

Assignee(s)
-----------

Primary assignee(s):
  - asselin
  - sweston

Additional assignee(s):
  - krtaylor
  - Chris Hoge

Gerrit Topic
------------

Use Gerrit topic "downstream-puppet" for all patches related to this spec.

.. code-block:: bash

    git-review -t downstream-puppet

Work Items
----------

  - Create a new project in openstack-infra called 'puppet-openstackci'

  - Refactor the following openstack_project components into sharable puppet
    scripts, and save them in openstack-infra/puppet-openstackci

    - Jenkins server
    - Jenkins Job Builder
    - Nodepool
    - Zuul
    - Log Server / os-loganalyze
    - Logstash / kibana

  - Migrate the system-config/module/openstack_project to use
    the puppet-openstackci

  - Add new check/gate job to verify puppet-openstackci integrated puppet script
    succeeds

  - Update Third Party CI Documentation

Repositories
------------

* openstack-infra/puppet-openstackci

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

Add documentation to use new Third Party CI puppet script here:

Published: http://ci.openstack.org/third_party.html

Source: https://github.com/openstack-infra/system-config/blob/master/doc/source/third_party.rst

Security
--------

None

Testing
-------

* Unit tests:
  Reuse the lint, syntax-check, etc. tests.

* Integration tests:
  Add a new test to verify the OpenStack CI system can be launched successfully
  on a single node.

Dependencies
============

- Refactoring of the OpenStack project components would likely be more easily
  accomplished after applying the changes documented in https://review.openstack.org/#/c/137471/
- New Puppet Module: puppet-openstackci
