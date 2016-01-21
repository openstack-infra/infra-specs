::

  Copyright 2016, Hewlett-Packard Enterprise

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Move docs.openstack.org/releases to releases.openstack.org
==========================================================

https://storyboard.openstack.org/#!/story/2000457

The release team would like to use a separate sub-site for the release
content, rather than pages on the docs site.

Problem Description
===================

Publishing the list of releases and the release schedules under the
documentation site is a poor fit, and it makes directing folks to the
relevant site more difficult than having a separate DNS entry. We may
also choose to theme the site differently, after we move out from
under the docs umbrella.

Proposed Change
===============

We need to move the content currently being published to
docs.openstack.org/releases to releases.openstack.org and put in place
a redirect for any of the old pages.

Alternatives
------------

We could leave everything where it is now.

Implementation
==============

Assignee(s)
-----------

Primary assignee: doug-hellmann

Gerrit Topic
------------

Use Gerrit topic "releases-openstack-org" for all patches related to
this spec.

.. code-block:: bash

    git-review -t releases-openstack-org

Work Items
----------

1. Create the new DNS entry.
2. Create new virtual server: https://review.openstack.org/266510
3. Update publishing instructions associated with the
   openstack/releases repository: https://review.openstack.org/266510
4. Set up a redirect from the old location to the new.

Repositories
------------

No new repositories.

Servers
-------

Not sure if the new virtual server counts for this or not.

DNS Entries
-----------

``releases.openstack.org``

Documentation
-------------

I will review the project team guide and other references to make sure
links point to the new location.

Security
--------

None

Testing
-------

None

Dependencies
============

None
