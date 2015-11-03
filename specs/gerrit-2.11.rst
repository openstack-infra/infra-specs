::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===================
Gerrit 2.11 Upgrade
===================

We will upgrade our current Gerrit 2.8 production instance to 2.11.

Problem Description
===================

Our prior upgrade from 2.8 to 2.10 was reverted due to a bug which
caused repositories to be marked corrupt under load. This is now
suspected fixed but has resulted in us falling well behind in Gerrit
releases.

Proposed Change
===============

Upgrade to Gerrit 2.11 in a maintenance window.

Alternatives
------------

Stay on 2.8 and fork or go without useful new fixes/features.

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

N/A

Repositories
------------

N/A

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

N/A

Dependencies
============

N/A
