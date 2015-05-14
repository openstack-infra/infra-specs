::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Store Build Logs in Swift
=========================

Rather than store test build logs in a very large filesystem, store them
in swift.

Problem Description
===================

For a while, we have been storing test logs from builds in a very
large filesystem on static.openstack.org.  It is not large enough to
store all of the data that we wish, and has already reached the
maximum capacity that we can allocate to it, and it incurs some system
administration overhead to maintain.

Proposed Change
===============

Store log files in Swift instead.  Zuul will provide per-job
credentials to the workers based on the swift tempurl and formpost
facilities.

Alternatives
------------

Put logs in AFS or otherwise scale out the static filesystem solution.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jhesketh


Gerrit Topic
------------

Use Gerrit topic "enable_swift" for all patches related to this spec.

.. code-block:: bash

    git-review -t enable_swift

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
