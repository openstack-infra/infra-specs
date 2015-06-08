::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode


============================
Puppet 4 Preliminary Testing
============================

https://storyboard.openstack.org/#!/story/2000243

Puppet 4 is coming and there is nothing we can do to stop it. We can, however,
prepare and be ready for the onslaught.

Problem Description
===================

Because puppet 4 is a semver major change, it means we can expect that breaking
changes will be introduced that will cause us to have to audit and possibly
change things in many places in our rather vast armada of puppet code. It would
be satisfying to just complain about the burden such a choice places on the
user, but it's the world we live in and we will ultimately have to deal with
it whether we want to or not.

Proposed Change
===============

There are puppet lint modules that can provide warnings about puppet 4
issues. It's also possible to install puppet 4 into testing environments
before we install it globally. The puppet-openstack team has already added
puppet4 puppet lint checks to their code, so we should start aligning with
some of their process so we can share effort.

First, move to using Bundler for our dependencies and rake tasks. This means
adding a Gemfile to each puppet repo. The puppet lint jobs already know how
to deal with a Gemfile enabled. This will allow us to specify additional
puppet lint plugins that we need to check the things that are gotchas.

Once all of the infra puppet repos have Gemfiles, add the `puppet-syntax-future`
template to all of them, which currently assumes a Gemfile structure, but is
already set up to run the future parser check.

As a follow on to that, we can stop installing the puppet lint gems globally
on all of our test nodes, because we'll be installing those gems into a local
dir using bundler. It's not necessary for this, but those installs will be
cruft at this point, so we should clean them up.  We should also just make a
general "bundler-prep" macro and rework the many places in the jjb source for
both infra and puppet-openstack where bundler prep steps are copied.

We should then update the apply test to be able to be run in a bundler context.
Then we can add some non-voting puppet apply jobs that run the apply tests with
puppet 4 instead of puppet 3. The Gemfiles from puppet-openstack already
support looking for a puppet version in the env var `PUPPET_GEM_VERSION`,
so we can use that a key as to whether we want to use system puppet or bundle
puppet.

Alternatives
------------

The bundler part is not necessary. We could also write a jenkins job macro
that overrides the apt pins, installs puppet 4 and installs a set of gems.

puppet 4 could be avoided and we could stay with puppet 3 until the end of time,
but at some point puppet 3 is going to be out of security updates.

We could also ditch puppet and move 100% to ansible. At the moment though, we
have a lot of puppet folks and a lot of puppet code, so a wholesale move seems
like a thing that would not be short in planning, and puppet 4 is imminent. So
even if we were to decide to do such a thing, it would likely still not be
short-term enough to be able to avoid puppet 4.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  unknown

mordred has some initial patches up for review, but probably will not be the
one leading this charge in an ongoing basis, unless he just gets ragey one
Friday night.

Gerrit Topic
------------

Use Gerrit topic "puppet-4" for all patches related to this spec.

.. code-block:: bash

    git-review -t puppet-4

Work Items
----------

* Add Gemfiles to all of the puppet repos
* Fix puppet4 related problems
* Update puppet-apply test to handle puppet-4 as an option
* Add non-voting puppet4-apply tests
* Fix puppet4-apply issues
* Make puppet4-apply voting

Repositories
------------

No new repositories.

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

Local testing will need to learn about `bundle exec rake lint`. A helper
script will be provided.

Security
--------

None

Testing
-------

A large portion of this is about constructing tests.

Dependencies
============

This is related strongly to the ansible-puppet-apply task, but they are
not strongly dependent - except to say that we're not going to upgrade to
puppet 4 in production until the puppet-apply task is also done.
