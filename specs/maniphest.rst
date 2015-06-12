::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode


===================
maniphest migration
===================

https://storyboard.openstack.org/#!/story/2000281

storyboard does not have enough development resources, so migrating to
maniphest (part of phabricator) is the next best thing.

Problem Description
===================

OpenStack needs a bug, feature and work tracker that is not Launchpad. While
Launchpad has served us well, it is increasingly unable to handle our needs,
and is also tied to Ubuntu SSO and not openstackid.

Maniphest, from Phabricator, while not perfect, is a solid system with a
vibrant developer community.

Proposed Change
===============

Migrate Infra from storyboard to maniphest.

This spec only covers moving Infra to maniphest. If that goes well, a
subsequent spec, TC resolution and data migration will follow if it is deemed
to have gone well.

Conceptually, this is straight forward. We need to install a phabricator
server, configure it to turn off the services we don't want, migrate in the
data from storyboard and connect it to authentication.

Migrating the data from storyboard instead of from Launchpad is proposed for
three reasons:

#. Since this is Infra, there is data that is in storyboard and not in
   Launchpad, but there is no data in launchpad but not in storyboard.

#. Both maniphest and storyboard use MySQL and we have access to both - so the
   data migration can be done purely as a database level data transformation,
   which is very cheap to perform.

#. If we get to migrating the rest of OpenStack, we can reuse the same process.
   Since we already have a launchpad to storyboard migration written, we can
   simply migrate from Launchpad to storyboard, then export the data and
   migrate the data into maniphest.

The puppet-phabricator repo also has a database data migration script which
mostly works, but needs to be validated.

All of the current projects in storyboard, regardless of Infra or non-Infra
will be migrated at the same time. We will not run in a state where we have
three bug trackers.

The puppet-phabricator repo is already imported into infra that is based on
work from a POC. It needs to be fleshed out some. It does already contain
the config in `local.json.erb` to turn off the features OpenStack does not want.
Examples of features from phabricator that Infra does not want to run are the
Code Review tool Differential since we have gerrit, the Group Messaging tool
Conpherence, because we have IRC, the code hosting tool Diffusion, since
we already have the cgit farm, and the wiki tool, Phriction, since we have
mediawiki.

maniphest needs to be connected first to Launchpad and later to openstackid.
phabricator does not have openid support - but software-factory developed a
tool called `cauth` that knows how to do auth at the Apache layer and pass
through `REMOTE_USER` info. There is a phabricator plugin that consumes
`REMOTE_USER`, so we should use `cauth` for auth. It also does not have
OpenID support - but it is in python and storyboard has OpenID support, so
adding OpenID support to `cauth` should not be difficult.

As an aside - once we have that working, we may want to consider whether or not
to just use `cauth` and Apache for all of our sevices that need auth.

Work will need to be done to handle bug links and issue status changes. There
is a gerrit plugin for this,
https://gerrit.googlesource.com/plugins/its-phabricator which is being worked
by our friends at Wikimedia.

The data architecture in maniphest is slightly different, so we're going to
want to experiment with how its concepts of tags/projects work best for us.
With that in mind, to start we should have maniphest configured to allow for
more open permissions on those things so that we can try them out with real
data and real workloads. Once we're happy with that, we should lock those down.

Once Infra has migrated to maniphest, the current storyboard instance should be shut down to avoid confusion and redirects to the phabricator server should be
put in place.

Alternatives
------------

We could stay the course with storyboard, but we've so far been unable to
secure adequate development resources and it doesn't seem likely to change.

We could stay with Launchpad, but then we remain stuck with Ubuntu SSO.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mordred

Gerrit Topic
------------

Use Gerrit topic "<maniphest>" for all patches related to this spec.

.. code-block:: bash

    git-review -t <maniphest>

Work Items
----------

* Spin up a phabrictor server
* Install phabricator
* Write cauth openid support
* Validate and finish data migration script
* Perform data migration
* Turn off storyboard and install redirects

Repositories
------------

openstack-infra/puppet-phabricator

Servers
-------

phabricator.openstack.org

DNS Entries
-----------

phabricator.openstack.org

Documentation
-------------

All of the developer workflow around using storyboard will need to be
redocumented.

Security
--------

None

Testing
-------

We'll need functional testing for sure. We should probably consider a
staging server that we can use to test new config changes.

We also need to verify the data migration/import. There isn't a great way to
do this other than manual inspection. So we'll need to load the data into the
new server and have everyone find piles of information they find important.
Also - some of the data mapping choices in the migration script are arbitrary,
and we might discover we don't like them - so it's possible we might run a
migration, look at the import, decide we want a different mapping, change the
script, and run it again.

Dependencies
============

None
