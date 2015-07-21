::

  Copyright 2015 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Host Trystack Web Content
=========================

https://storyboard.openstack.org/#!/story/2000302

Host the http://trystack.org Trystack web content within the
community-managed project infrastructure.

*NOTE* This specification does not affect how or where the sandbox
environment runs. That is outside the scope of this document.

Problem Description
===================

The http://trystack.org Trystack web content is currently hosted on Rackspace
and maintained by Red Hat, Inc. Red Hat staff have agreed the best home for
this content is within the community hosting infrastructure rather than under
the control of a single member company.

Proposed Change
===============

Import the git repository for the web contents into a new repo with in
openstack-infra along with creating a new trystack.o.o static site. There is
no need to provision a new VM as the contents can live under the static.o.o
while creating an vhost within the Apache configuration.

Alternatives
------------

Allow Rackspace to continue hosting the site while Red Hat maintains the
static content for a community resource.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  pabelanger

Gerrit Topic
------------

Use Gerrit topic "puppet-trystack" for all patches related to this spec.

.. code-block:: bash

    git-review -t puppet-trystack

Work Items
----------

- Import trystack static content into a new openstack-infra/trystack-site
  repo.
- Update the puppet modules for static.o.o to include vhost for trystack.o.o.
- Create DNS entries.

Repositories
------------

A openstack-infra/trystack-site repo will be created.

Servers
-------

No new servers required.

DNS Entries
-----------

The trystack.openstack.org A and AAAA resource records will need to be
created. The trystack.org domain should be redirected to the new
trystack.openstack.org server.

Documentation
-------------

The openstack-infra/system-config documentation will be updated to include
a summary of the Trystack site.

Security
--------

The Trystack site is a static content driven from git. It will live on a
shared system with the ability to be moved to a dedicated server in the
future.

Testing
-------

N/A

Dependencies
============

N/A
