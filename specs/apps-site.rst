::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Host OpenStack Apps Catalog Service
===================================

https://storyboard.openstack.org/#!/story/2000272

Host the http://apps.openstack.org/ OpenStack Apps Catalog Service
within the community-managed project infrastructure.

Problem Description
===================

The http://apps.openstack.org/ OpenStack Apps Catalog Service was
set up in a hurry for announcement during a the Liberty Summit
keynote presentation, and due to time constraints was done outside
our project infrastructure. This is an unfortunate situation and
should be rectified as soon as possible to avoid embarrassing
mischaracterizations of the site's current hosting status.

Proposed Change
===============

The OpenStack Apps Catalog Service is a very basic, static Web
application for now. It will need a small VM with a puppet module
to continuously deploy the stackforge/apps-catalog Git repository
and configure Apache to serve it.

Alternatives
------------

We could leave it as is or ask someone else to host it on
OpenStack's behalf, but these are unacceptable for long-term
maintainability within the community.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  docaedo

Infrastructure root shepherd:
  fungi

Gerrit Topic
------------

Use Gerrit topic "apps-site" for all patches related to this spec.

.. code-block:: bash

    git-review -t apps-site

Work Items
----------

An apps_site puppet module will need to be created, some glue for it
should be added to openstack-infra/system config, and then it will
need to be applied to a new apps.openstack.org server.

Repositories
------------

An openstack-infra/puppet-apps_site repo will be created as part of
this effort.

Servers
-------

An apps.openstack.org server will need to be created, no other
existing servers should need to be modified.

DNS Entries
-----------

The apps.openstack.org A and AAAA resource records will need to be
updated to the IP addresses of the new server.

Documentation
-------------

The system-config services documentation will be updated with a
summary of the OpenStack Apps Catalog Service.

Security
--------

The OpenStack Apps Catalog Service currently consists of static
content driven from a Git repository. It is being provisioned onto a
dedicated system with the expectation that it may later grow an
interactive API without risking the security posture of other sites
hosted from the same server.

Testing
-------

The existing puppet apply/integration jobs will suffice.

Dependencies
============

- https://git.openstack.org/cgit/stackforge/apps-catalog
- https://git.openstack.org/cgit/openstack-infra/puppet-apps_site
