::

  Copyright 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

==================================
Migrate to Zanata for translations
==================================

https://storyboard.openstack.org/#!/story/328

Since Transifex no longer maintains an external, open source version of the
cloud-based service the translations team uses, we need to move to an open
source alternative. The infrastructure and translations teams have decided to
use the open source (LGPL) Zanata project hosted by the OpenStack
Infrastructure team.

Problem Description
===================

We currently do not host any translations tools, so a translations server will
need to be deployed for use with Zanata. All of our tooling is built around
Transifex so these scripts will also need to be adjusted and then the
translations imported into the new tool.

Proposed Change
===============

Switch from Transifex to Zanata for translations.

Alternatives
------------

Decision to use Zanata was based on what translators believe is a superior UI
and a workflow similar to that of Transifex. However, if Zanata does not work
for some reason, we can look into Pootle again.

Implementation
==============

Assignee(s)
-----------

Primary assignee(s):

* lyz
* jaegerandi

Work Items
----------

* Package Zanata client so that we can install it as part of the syncing jobs.

* Figure out how to setup the initial Zanata config file and whether we can do
  this dynamically or whether we have to add it to the repositories.

* Find or develop Puppet modules for Wildfly and Zanata, the Zanata team will be
  sending us Ansible playbooks for Zanata which we can convert to Puppet.

* Use these Puppet modules to bring up both a production and a development
  server for Zanata.

* Modify translations scripts that sync translations with Transifex to work with
  Zanata. In the first iteration, the scripts should continue to support both
  Transifex and Zanata for syncing and as they evolve to do import only from
  Transifex to Gerrit.

* Since the Zanata API does not allow to download only files that are at least
  75 per cent translated (or any other number), enhance our scripts that we
  download the files for all languages and filter locally out the not
  sufficiently translated files.

* Begin running modified translations scripts on the development server and work
  with the Translations team to test with some dummy translations against a
  sandbox.

* Work with Translations team to document the new translations workflow.

* Work with Zanata team to enhance Zanata so that we get notified whenever new
  languages are available for translation. Note: Currently the list of languages
  for a repository is stored in the local Zanata config file and thus we need to
  adjust it - or find a way to ignore the local config file - to download
  translated files for all languages.

* Export all translations (not only the 75% translated ones) from Transifex and
  import all the files into Zanata. Also import old versions of projects with
  translations (for example horizon/icehouse, horizon/havana) as reference and
  seed for translation memory.

* Notify translations team that the export has been completed and disable
  translations capability in Transifex now that everything is exported and
  we're ready to turn on the new service.

* Point new scripts at new Zanata production server and go live, announce to
  the Translations team.

* Notify broader development community about the switch and identify any way
  it impacts them.

* Document new infrastructure in OpenStack Infrastructure Documentation.

Repositories
------------

A new puppet-zanata repository will need to be created, along with updates to
system-config to consume this module and one for Wildfly, which can probably be
found upstream.

Servers
-------

A translate.openstack.org server will need to be created.

We will also need to re-install or replace the translate-dev server to be our
development platform using the Wildfly and Zanata Puppet modules (it currently
has an old demo of Pootle).

DNS Entries
-----------

DNS entry for translate.openstack.org will need to be created.

If we create a new server, DNS for translate-dev will need to be updated to
point at the new development sterver.

Documentation
-------------

Documentation for our Zanata server setup will need to be added to the
ci.openstack.org documentation.

Translator-facing documentation will need to be updated for the new tool.

Security
--------

These new services run effectively on Ubuntu, so we don't anticipate security
problems on the core operating system.

A couple of lifecycle notes about Wildfly and Zanata:

* Wildfly is "aiming to deliver new final major versions in around a nine
  month cycle" so we will need to stay on top of upgrades for that. More at:
  http://wildfly.org/governance/

* Zanata has a major release we will want to upgrade to every 3 months.

Testing
-------

A development server with the prepared automated scripts pointed at a sandbox
will first be used to confirm that translations are being processed properly.

Dependencies
============

We may be pulling in an external Puppet module to support Wildfly.

Otherwise this should be an independent change in our infrastructure aside from
the previously mentioned updating of automated scripts.
