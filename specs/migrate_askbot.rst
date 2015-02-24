::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Migrate ask.openstack.org to infra
==================================

https://storyboard.openstack.org/#!/story/2000158

The OpenStack Q&A site located at ask.openstack.org currently
running in an instance deployed by a third-party. The main
goal of this migration to make the entire deployment process
repeatable, and move the project to proper infrastructure
puppet repositories.

Problem Description
===================

The ask.openstack.org site was deployed manually and contains
the minimal application stack required for running the site.
Currently the site is missing regular backups, security
updates and deployment documentation.

Proposed Change
===============

Create the missing puppet modules, prepare the changes in
openstack-infra/system-config repo.

Alternatives
------------

Leave ask.openstack.org as is.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  marton-kiss

Gerrit Topic
------------

Use Gerrit topic "askbot-site" for all patches related to this spec.

.. code-block:: bash

    git review -t askbot-site

Work Items
----------

* Lower ask.openstack.org DNS TTL to 300
* Create puppet-askbot split-out module
* Add vamsee-solr module and puppet-askbot to modules.env
* Make the system-config changes, add ask.pp
* Add SSL certificates and passwords to hiera
* Launch new ask.openstack.org server
* Restore database and static files from original ask.openstack.org site
* Silent testing with /etc/hosts override
* Backup / restore of original ask.openstack.org data
* Update DNS entry of ask.openstack.org with the new server address
* Redirect html traffic using nginx to new ask.openstack.org to avoid db sync issues
* Restore ask.openstack.org DNS TTL to 3600

Repositories
------------

A new puppet-askbot repository will need to be created, along with updates to
system-config to consume this module.

Servers
-------

An ask.openstack.org will need to be created.

DNS Entries
-----------

The ask.openstack.org zone must be point to the newly created server as the
last step of this migration process.

Documentation
-------------

Askbot documentation need to be added to ci.openstack.org documentation.

Security
--------

The services will run on Ubuntu, so core operating system not requires any
special attention.

The application stack have some elements that must be deployed from
tar.gz or pypy instead of OS packages:

* Apache Solr (4.7.2)
* askbot

Testing
-------

Askbot don't have integration tests implemented. After instance creation
and initial data migration, I suggest to do a 1-2 week long silent test
of the UI and address upcoming bugs during that period.

Dependencies
============

We are using vamsee-solr module 0.0.7 from puppetforge, and it is forcing
Us to use solr 4.7.2 because 4.10.x requires some extra patches to work and
this upgrade also means a schema change.