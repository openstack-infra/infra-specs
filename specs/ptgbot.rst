::

  Copyright 2017 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

=======
PTG Bot
=======

https://storyboard.openstack.org/#!/story/2001067

We will host a production instance of the PTG event scheduling bot.

Problem Description
===================
Quoting from `Thierry Carrez's May 18 post to the OpenStack
development mailing list
<http://lists.openstack.org/pipermail/openstack-dev/2017-May/116974.html>`_:

    For the PTG events we have, by design, a pretty loose schedule.
    Each room is free to organize their agenda in whatever way they
    see fit, and take breaks whenever they need. This flexibility is
    key to keep our productivity at those events at a maximum. In
    Atlanta, most teams ended up dynamically building a loose agenda
    on a room etherpad.

    This approach is optimized for team meetups and people who
    strongly identify with one team in particular. In Atlanta during
    the first two days, where a lot of vertical team contributors
    did not really know which room to go to, it was very difficult
    to get a feel of what is currently being discussed and where
    they could go. Looking into 20 etherpads and trying to figure
    out what is currently being discussed is just not practical. In
    the feedback we received, the need to expose the schedule more
    visibly was the #1 request.

Proposed Change
===============

Add an instance of the ptgbot software running on and publishing to
a page served from eavesdrop.openstack.org.

Alternatives
------------

PTG attendees could continue using only the tools we currently
provide for coordination (a dedicated IRC channel, Ethercalc
spreadsheet and Etherpads), though this would not meet the requests
from the first PTG for a more streamlined overview of what's going
on at any given point in time.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fungi

Gerrit Topic
------------

Use Gerrit topic "ptgbot" for all patches related to this spec.

.. code-block:: bash

    git-review -t ptgbot

Work Items
----------

#. Create deployment automation repository
   `openstack-infra/puppet-ptgbot`.
#. Update `openstack-infra/system-config` repository to add the
   ``ptgbot`` module and instantiate it within the
   ``eavesdrop.openstack.org`` node resource definition; also
   include operational documentation for the service.

Repositories
------------

The `openstack-infra/puppet-ptgbot` Git repository will be created
as a new Infrastructure team deliverable.

Servers
-------

The existing ``eavesdrop.openstack.org`` server will be used to host
the daemon and Web content.

DNS Entries
-----------

No DNS entries need to be created or updated.

Documentation
-------------

Operational documentation will be added to the
`openstack-infra/system-config` repository.

Security
--------

The daemon shares much in design with the existing statusbot
implementation (particularly the ``#success ...`` feature but with
locally-generated Web content), and so has a risk profile which is
sort of a cross between statusbot and meetbot. Because of this, and
its expected lightweight resource consumption, we'll run it on the
same server where we run those.

Testing
-------

Our usual Puppet integration testing jobs will be applied.

Dependencies
============

This implementation will depend on creation of a new
`openstack-infra/puppet-ptgbot` module, which in turn utilizes the
existing `openstack/ptgbot` repository and software contained
therein.
