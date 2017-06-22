::

  Copyright 2015 Łukasz Jernaś
  Copyright 2017 Frank Kloeker

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

================================================
Provide a translation check site for translators
================================================

Story: https://storyboard.openstack.org/#!/story/2000267

Provide a means for translators for easily checking the translations imported
from the translation system in a production like environment.

Problem Description
===================

Translators and translation reviewers need a way to verify that translation
behaves correctly when applied to the OpenStack Dashboard and its plug-ins.
This also helps when context information is missing from a string, for example
if "Search" is a label or a button, because in some cases the translations
have to be different. Now a translator has to run their own DevStack instance,
fetch the updated translation, run the build, etc - which is cumbersome,
requires skills outside the translation ones and duplicates a lot of work
between every translation group.

Proposed Change
===============

A sample OpenStack instance should be provided to translators, which
runs the current working branch and regularly fetches updated translations.
The instance should run Horizon and every module supported by the stock
dashboard.
This could be achieved with openstack-ansible AIO.
AIO means "all in one". OpenStack components are installed in separate container
(i.e. Horizon) on a single VM. Advantages for containers are: easy to manage
(i.e. snapshot backup and recovery in case of failed updates of the horizon container).
openstack-ansible itself is reboot-save. It's not necessary to re-install
from scratch like DevStack. In the Horizon container a script is installed
to fetch the translation file periodically from the translation server via cron,
compile it and serves the new content to the dashboard. The dashboard contents
the core functionality plus core plugins. The instance should be able to spawn
pseudo VMs that are using fakevirt driver, create networks and all other
capabilities provided by Horizon, but should be firewalled off from the rest
of the world with a periodic cleanup of all resources.
There is no need for SSO integration as the only required accounts are a shared
admin and a user account, with the credentials known to the translation team.
The sync cycle of the translation files is configurable. Update mechanism for
different branches or releases of the dashboard are also configurable and done
by ansible. The installed AIO should upgraded 4 times each cycle: soft/hard
string freeze, release, stable.

Alternatives
------------
As an alternative a prebuilt VM image could be created for the translators
to run on their own workstations or cloud with a manual run of ansible
playbooks for update Horizon or sync the translation files.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
 * Frank Kloeker <f.kloeker@telekom.de>

Also:
 * Clark Boylan <cboylan@sapwetik.org> (Infrastructure side)
 * Rob Cresswell <robert.cresswell@outlook.com> (PTL Horizon)
 * Andy McCrae <andy.mccrae@gmail.com> (PTL OpenStackAnsible)
 * Ian Y. Choi <ianyrchoi@gmail.com> (PTL I18n)
 * KATO Tomoyuki <kato.tomoyuki@jp.fujitsu.com> (I18n Core Team)

Gerrit Topic
------------

Use Gerrit topic "i18n-checksite" for all patches related to this spec.

.. code-block:: bash

    git-review -t i18n-checksite

Work Items
----------

* Create puppet module for creating and spawning instance
* Create firewall and networking rules, only HTTPS for the dashboard

Repositories
------------

None

Servers
-------

A new server will have to be created with sufficient resources to run all
the required components and at least a few cirrus virtual machines.

DNS Entries
-----------

There should be a new DNS entry created for easy access by the translation
teams.

Documentation
-------------

Documentation related to configuration and potential AIO debugging
for the infrastructure team in the system-config repository.

Documentation for translators who will be using this.

Security
--------

Security issues should be taken into consideration, as the dashboard
allows the creation of new networks, floating IP addresses, load balancers
and instances. However, as long as fake virt driver is used, these security
concerns disappear.

Testing
-------

A basic set of functional tests and monitoring should be set up, for example
if the environment recreation completed successfully and every service runs
correctly.


Dependencies
============

This will require extension of openstack-ansible-os_horizon

