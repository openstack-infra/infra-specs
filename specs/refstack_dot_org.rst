::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Host refstack.org and api.refstack.org on infra
===============================================

https://storyboard.openstack.org/#!/story/2000277

The RefStack UI and API servers have matured to a point where the DefCore
committee and Foundation staff feel it can be an OpenStack hosted project
to collect and report voluntarily collected test results.

Problem Description
===================

RefStack was conceived as a semi-anonymous test reporting tool to support
the efforts of the DefCore committee. Vendors and users can use the
refstack-client to anonymously upload results from test runs against
cloud apis for reporting of widely deployed capabilities and verification
of testing standards for the DefCore process.

As a board initiated and community supported process, RefStack is best
hosted as a community resource.

Proposed Change
===============

To begin with refstack can start with a simple deployment and host the front end
ui and api on the same vm with a single ip. If traffic demands, refstack is
fully capable of being load balanced.

- Allocate a virtual machine for website and api hosting with 2gb of ram.
- Edit DNS record to point refstack.org VM.
- Allocate trusted certificate for refstack.org
- Allocate a trove database instance, start with 5gb of disk and base level cpu.
- Create puppet modules do deploy RefStack. Modules will:
    - Deploy RefStack UI webserver at https://refstack.org
    - Deploy RefStack API webserver at https://refstack.org/api
- Import existing data from refstack.net. (david lenwell can provide)

Alternatives
------------

Continue to self-host RefStack at http://refstack.net, unsecured and
privately held.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Michael Krotcheck (krotcheck)

Secondary asignees:
  David Lenwell (davidlenwell)
  Chris Hoge (hogepodge)

Gerrit Topic
------------

Use Gerrit topic "refstack" for all patches related to this spec.

.. code-block:: bash

    git review -t refstack

Work Items
----------

* Allocate TLS certificates for refstack.org.
* Allocate VM.
* Edit refstack.org to point to allocated VM.
* Write Puppet module to deploy refstack.org and database.
* Allocate trove database.
* Entry in infra trove database manifest needs to be created.
* Import existing database from refstack.net.

Repositories
------------

* Refstack is hosted at stackforge/refstack.
* Puppet deployment prototype is hosted at
  https://github.com/krotscheck/puppet-refstack and will be relocated to infra.

Servers
-------

A server to host refstack.org and will need to be created.

DNS Entries
-----------

refstack.org must point to the newly created server.

Documentation
-------------

RefStack documentation, for both deployment and usage, is located in the
stackforge/refstack repository.

http://git.openstack.org/cgit/stackforge/refstack/tree/doc/refstack.md

Security
--------

* The services will run on Ubuntu trusty

* Only TLS enabled connections will be allowed to refstack.org.
  (i.e. forward http traffic to https)

Testing
-------

RefStack has continuous testing in place as part of standard OpenStack
development processes. Failures and bugs identified in deployed code
will be tested for and added to the continuous testing process.

Backup / recovery steps
=======================

Backup assets
-------------

Traditional backups like any other trove hosted infra database.

Invoke management commands
--------------------------

None

Stop apache
---------------------------------------

.. code-block:: bash

    $ sudo service apache2 stop

Restore database dump
---------------------

Standard MySQL database dump and restore.

Restart apache
--------------

.. code-block:: bash

    $ sudo service apache2 restart

Dependencies
============

Dependencies are captured in the RefStack project, and will be managed
by the puppet module.
