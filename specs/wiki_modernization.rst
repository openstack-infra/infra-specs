::

  Copyright 2016 Hewlett-Packard Enterprise

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==================
Wiki Modernization
==================

https://storyboard.openstack.org/#!/story/2000641

The OpenStack wiki at wiki.openstack.org needs to be fully Puppetized,
upgraded to the current stable version of MediaWiki and upgraded to a
newer Ubuntu LTS version.

Problem Description
===================

Vandalism to the wiki this year has highlighted the fact that we need to
upgrade the wiki in order to apply modern security fixes and begin using
modern spam fighting tools. In the short term, we have disabled new
account creation but this is restricting the value of the wiki and
unfairly disadvantaging new contributors.

It is current running on Ubuntu 12.04 and dependencies dictate that we
upgrade to 14.04 in order to get the most recent version of MediaWiki.

The installation is also not fully Puppetized, so some parts of our
maintenance and upgrade process are manual and not fully documented.

Proposed Change
===============

This modernization effort has several components.

The full process of installation has to be Puppetized. Investigation
needs to be done into the existing puppet-mediawiki modules to see if we
should use and contribute to those, or continue building our in house
module. The mediawiki package for Ubuntu will continue to not be used,
since it's been dropped from Debian and the packaging and mediawiki
community has indicated a preference for source-based installation.

Create new instance running Ubuntu 14.04 in order to move the wiki off
of 12.04. Backups will need to be made of the current state of the wiki
and data moved as needed.

Using our new Puppetized installation process, migrate the wiki content
to the new version of mediawiki our module is using.

Optional: Evaluate continued use of the Ubuntu SSO mechanism for the
wiki as we have no control over the compromised spam accounts coming in
from that service..

Alternatives
------------

Work with the community to move away from using a wiki at all and
evaluate tooling to replace it.

Continue to use a wiki, but evaluate alternatives to MediaWiki and do a
migration to that new platform.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fungi

Gerrit Topic
------------

Use Gerrit topic "wiki-upgrade" for all patches related to this spec.

.. code-block:: bash

    git-review -t wiki-upgrade

Work Items
----------

* Evaluate state of PuppetForge puppet-mediawiki modules compared to
  ours.
* Work to modernize selected puppet-mediawiki module so it fully meets
  our needs.
* Bring up new wiki-dev.o.o instance running Ubuntu 14.04 to begin
  testing the new wiki, and use in the longer term to coordinate
  upgrades and new features.
* Bring up new wiki.o.o instance running Ubuntu 14.04.
* Come up with migration and rollback plan for data to new server and
  new version of MediaWiki.
* Notification to the community around changes with new version and any
  changes to logging in to or using the wiki.
* Complete final cutover to new server
* Evaluate and try the popular, modern spam fighting plugins and
  techniques on the new server.

Repositories
------------

Our existing puppet-mediawiki repository may be improved upon if we
don't find a better third party module. No new repositories should be
required.

Servers
-------

A new wiki-dev.o.o server will be created.

The current wiki.o.o server will be replaced with a new one running
Ubuntu 14.04.


DNS Entries
-----------

New wiki-dev.openstack.org A/AAAA records will need to be created.

Documentation
-------------

Updates to our system-config documentation for the wiki will need to be
completed.

Security
--------

The goal is for security to be increased by this modernization work.

Testing
-------

The introduction of the wiki-dev server will help us be more effective
with changes made to the wiki server overall. Standard testing of Puppet
modules will continue with the puppet-mediawiki module as before.

Automated tooling that updates the wiki will need to be tested and
confirmed working.

Dependencies
============

N/A
