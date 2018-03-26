::

  Copyright 2018 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=====================
IRC Bot Consolidation
=====================

https://storyboard.openstack.org/#!/story/2001736

Reduce the number of IRC bots and ensure they can operate in all channels.

Problem Description
===================

We have a couple of IRC-related problems converging:

* Too many channels (more than limit of 120 per connection)
* Lots of individual bots, which all must be configured and all have that limit
* Bots are written for different frameworks
* Difficult to react to spam in one or more channels across so many

Proposed Change
===============

We use supybot for channel and meeting logs.  It is not under active
development, but it has a successor called Limnoria, which is in later
Ubuntu releases (after xenial).  It's python and is easy to install in
a venv in the interim.  Using it lets us continue to use our existing
meetbot system, which is the most difficult part of our irc services
to change.  Our other bots are mostly home-grown and could be ported
fairly easily.  With it, we can have the same bot instance participate
in all channels by using its ability to support multiple connections.
That will let us log all 186 channels, hold meetings in them, send
status updates, gerrit notifications, etc, with one config.

The following config example illustrates how to use the multiple
connection feature of limnoria to address the channel limit::

  supybot.networks: freenode1 freenode2
  supybot.networks.freenode1.nick: openstack1
  supybot.networks.freenode2.nick: openstack2
  supybot.networks.freenode1.servers: chat.freenode.net:6667
  supybot.networks.freenode2.servers: chat.freenode.net:6667
  supybot.networks.freenode1.channels: #openstack
  supybot.networks.freenode2.channels: #openstack-infra

Alternatives
------------

We could switch to errbot, though we would additionally need to
address the meetbot and logging functionality we currently use with
supybot, as well as develop a new method of updating configuration.

Implementation
==============

Assignee(s)
-----------

Your name here.

Gerrit Topic
------------

Use Gerrit topic "ircbots" for all patches related to this spec.

.. code-block:: bash

    git-review -t ircbots

Work Items
----------

* Install limnoria on eavesdrop, to replace supybot
* Configure freenode1 and freenode2 networks, using openstack1 and openstack2 nicks
* Add all channels to the config, splitting across freenode1 and freenode2
* Set all channels to extban based on #openstack so it's easy to
  update a global ban list.
* Add a plugin to watch for highlight spam, and automatically kick
  accounts and add a ban to #openstack
* Auto-generate the limnoria config based on a yaml file with channel descriptions
* Convert statusbot into a plugin and retire
* Convert gerritbot into a plugin and retire
* Convert recheckwatch into a plugin and retire

Repositories
------------

None.

Servers
-------

Changes to eavesdrop.o.o and other servers which currently run bots.

DNS Entries
-----------

None.

Documentation
-------------

Bot configuration docs should be updated in infra-manual.

Security
--------

This will probably put an unprivileged gerrit ssh key on eavesdrop,
but otherwise should not adversely affect security posture.  Ceasing
to run gerritbot on review.o.o is a likely security benefit.

Testing
-------

None.

Dependencies
============

None.
