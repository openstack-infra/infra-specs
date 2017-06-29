::

  Copyright 2017 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

===========================
Gerrit ContactStore Removal
===========================

https://storyboard.openstack.org/#!/story/2001094

The time has come to stop relying on the ContactStore implementation
in Gerrit to limit code contributions to Foundation Individual
Members.

Problem Description
===================

According to the Bylaws of the OpenStack Foundation Appendix 4
`Technical Committee Member Policy`_ §3.b along with the OpenStack
Technical Committee Charter definitions for APC_ and ATC_, we limit
the voter rolls for technical elections to Foundation Individual
Members. In order to comply with this requirement, we currently
require all contributors to CLA-enforced Git repositories to submit
*contact info* to the Gerrit `contact store`_ which in turn pings a
simple API in the foundation member system to confirm the preferred
E-mail address in Gerrit matches the primary E-mail address of an
existing OpenStack Foundation Individual Member.

This has a number of drawbacks:

#. It forces contributors to join the OpenStack Foundation even if
   they have no interest in voting in technical elections or
   participating in other member benefits.

#. Our interpretation of the meaning of *contributor* for these
   purposes has been unnaturally limited to *change owners in
   Gerrit*, in part because commit authors and co-authors aren't
   constrained by the contact store process and so might not be
   members; manual listing as *extra ATCs* in the ``governance``
   repo has been the sole workaround, and requires cumbersome manual
   verification of foundation membership for each addition.

#. The model is inherently flawed since it's been possible for a
   couple years now for a member to officially resign or allow their
   membership to lapse, but contact store submission is only ever
   enforced once when the account is first set up and so we may be
   incorrectly allowing lapsed or resigned members to vote in
   technical elections.

#. The implementation is brittle and process confusing, resulting in
   opaque errors which often confound new contributors and overall
   inhibit onboarding.

#. Because the protocol only submits a single E-mail address and
   backend implementation in the current member system only queries
   against a single address field, it unnecessarily causes users to
   have the same primary/preferred address in both systems (at least
   initially).

#. Gerrit has `removed contact store functionality`_ upstream after
   2.11, and we'd like to be able to upgrade to a newer Gerrit
   release.

.. _Technical Committee Member Policy: https://www.openstack.org/legal/technical-committee-member-policy/
.. _APC: https://governance.openstack.org/tc/reference/charter.html#voters-for-ptl-seats-apc
.. _ATC: https://governance.openstack.org/tc/reference/charter.html#voters-for-tc-seats-atc
.. _contact store: https://review.openstack.org/Documentation/config-contact.html
.. _removed contact store functionality: https://gerrit-review.googlesource.com/c/70458

Proposed Change
===============

Very recently the OpenStackID Resources system has introduced a
`member directory API`_ which is public and anonymous. Integrating
this into the `change owners script`_ we use for generating
electoral rolls will allow us to expressly filter out non-member
contributors.

Side effect benefits include:

* it can help further identify duplicate contributors where there
  may be multiple E-mail addresses in the member system for a single
  membership, yet corresponding to multiple accounts in Gerrit with
  those different addresses

* it will also properly limit voting rights for *extra ATCs* who
  have not joined the foundation, eliminating any need for the
  current cumbersome vetting process

* it would even enable us (should we choose) to more easily expand
  the interpreted definition of ATC to include a variety of other
  types of verifiable contribution tied to a known E-mail address
  including commit authors and co-authors

.. _member directory API: http://git.openstack.org/cgit/openstack-infra/openstackid-resources/tree/app/Models/Foundation/Main/Member.php?id=5b6011d
.. _change owners script: http://git.openstack.org/cgit/openstack-infra/system-config/tree/tools/owners.py?id=53bb44d

Alternatives
------------

We could live with the terrible terribleness, continue to hold
easily disputed elections, scare away new contributors and run an
outdated Gerrit. Not much of an alternative if you ask me.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fungi

Gerrit Topic
------------

Use Gerrit topic "gerrit-contactstore-removal" for all patches
related to this spec.

.. code-block:: bash

    git-review -t gerrit-contactstore-removal

Work Items
----------

#. Update ``owners.py`` to use the new member directory API.

#. Notify election officials of the change in behavior.

#. Remove the contact store implementation from Gerrit configuration
   templates and manifests in ``puppet-gerrit`` and
   ``system-config`` repos.

#. Update the account setup steps documented in the ``infra-manual``
   repo to indicate that foundation membership is optional (but
   encouraged).

#. Notify the developer community at large by posting an
   announcement of the new contributor onboarding behavior
   change/simplification.

#. Make sure the Upstream Institute volunteers are aware so they can
   update their training materials accordingly.

Repositories
------------

No new git repositories need to be created.

Servers
-------

No new servers need to be created. The ``review.openstack.org`` and
``review-dev.openstack.org`` servers will have configuration changes
via Puppet and need ``gerrit`` service restarts for this to take
effect. The necessary outage will be brief, so a restart at a
reasonably convenient time for the community should not require
advance notification nor planning.

DNS Entries
-----------

No DNS entries need to be created or updated.

Documentation
-------------

As mentioned in the `Work Items`_ section, the Infra Manual will
require updates to reflect the new onboarding workflow.

Security
--------

This does not introduce any additional known security risks, and
there are no identified security-related considerations which need
discussing.

Testing
-------

Manual testing of the ``owners.py`` script change should be
performed against official contributor data, comparing output
between runs of the old and new versions for any unintended changes
in behavior.

Dependencies
============

There are no other specs, libraries or new Puppet modules on which
this specification depends.
