::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Use Diskimage Builder in Nodepool
=================================

To ensure identical test environments in different clouds, use
Diskimage Builder to create images which are then uploaded to each
cloud used by Nodepool.

Problem Description
===================

Each cloud provider in use by Nodepool supplies different images which
can cause tests to perform differently.

Proposed Change
===============

Rather than try to correct the delta in each provider, create an image
locally that can be used in any provider and upload it to Glance.

Because it is difficult to use glance in all of its deployment
variations, move much of the cloud-interaction logic into the "shade"
library and use that to perform the uploads.

Alternatives
------------

Keep playing catch-up whenever clouds update their images.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mordred


Gerrit Topic
------------

Use Gerrit topic "dib-nodepool" for all patches related to this spec.

.. code-block:: bash

    git-review -t dib-nodepool

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
