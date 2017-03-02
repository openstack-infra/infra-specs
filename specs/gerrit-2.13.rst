::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===================
Gerrit 2.13 Upgrade
===================

We will upgrade our current Gerrit 2.11 production instance to 2.13.

Problem Description
===================

Our production Gerrit server is two minor releases behind the latest
release of Gerrit.  We want to update gerrit from 2.11.4 to 2.13.x so
that we can get the latest features and bug fixes.  Also we don't
want to fall to far behind upstream otherwise it might be more
difficult to upgrade later.


Proposed Change
===============

Upgrade to Gerrit 2.13 in a maintenance window.

Alternatives
------------

1. Stay on 2.11. and go without useful new fixes/features.
Pros: We know the issues with this release.
Cons: We are behind upstream and don't get many new features.

2. Upgrade to 2.12.x
Many companies have already upgraded to 2.12.x which makes this a viable
option.
Pros: Other companies have vetted this version.
Cons: Fixes might be difficult to get into this branch.

3. Upgrade to master
Pros: May get fixes faster on master than on stable branches
Cons: Only Google and GerritForge is on master.

Risks
-----

2.13.x has not been installed by many companies so it really hasn't been
vetted by other companies yet.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zaro


Gerrit Topic
------------

Use Gerrit topic "gerrit-upgrade" for all patches related to this spec.

.. code-block:: bash

    git-review -t gerrit-upgrade

Work Items
----------

We don't have a zuul dev environemnt that uses review-dev. Currently zuul-dev.o.o
does not run jobs for review-dev.o.o. The first step is to update zuul-dev.o.o to
simulate production zuul.  We need to update it to the same version and get it to
run jobs triggered off of the gerrit stream events.  Once that happens we'll have
a valid dev environment for testing Gerrit 2.13.  There's more on testing gerrit
in the testing section.

In case we need to rollback, we should create a mysql database rollback script.

Create a fork from Gerrit stable-2.13 branch and add build jobs for 2.13 and it's
associated plugins.

Puppet up some changes to swap out new versions.


Repositories
------------

Create an 'openstack/2.13.1' fork from the latest Gerrit 2.13.x tag


Servers
-------

N/A

DNS Entries
-----------

N/A

Documentation
-------------

N/A

Security
--------

N/A

Testing
-------

We need to update review-dev.o.o with Gerrit 2.13 and test the
following integrations:

- data migration
- gerrit replication
- gerrit javascript (toggle-ci & test results)
- jeepyb integration
- zuul integration
- storyboard integration (its-storyboard plugin)
- launchpad integration
- gerrty
- rollback
- javamelody plugin
- gerrit hooks
- git-review


Dependencies
============

N/A
