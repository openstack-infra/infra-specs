::

  Copyright 2018 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

======================
OpenDev Gerrit Hosting
======================

Storyboard: https://storyboard.openstack.org/#!/story/2004627

Migrate review.openstack.org to review.opendev.org.

Problem Description
===================

OpenDev is intended to be the project infrastructure used by open
infrastructure projects, no longer limited to OpenStack.  To convey
that, we need to change the name of the Gerrit server, and accommodate
new repository organizational structures.

Proposed Change
===============

Because of the complexity of operating a Gerrit server, OpenDev will
provide a single Gerrit server for all supported projects.  Whitelabel
branding of the service will not be available.

Since Zuul, golang, and other tools operate best with a single
canonical name for git repositories, we will also deprecate whitelabel
branding of git repository hosting.

The existing review.openstack.org server will be renamed to
review.opendev.org.

Plain git hosting will be served from opendev.org.  The repository
directory structure will be org/project.

For example the canonical name for the nova repository will be
"opendev.org/openstack/nova".  Zuul will be "opendev.org/zuul/zuul".

The URL https://opendev.org/openstack/nova will serve a browseable
version of the repository, and can also be used to clone and fetch
directly with git.

We will use Gitea for repository browsing.  It is a fully-featured
GitHub close which nonetheless allows disabling features we don't use
(issues, wiki, pull requests) as well as customizing web pages (so we
can configure a custom header, etc.  It can support inline RST
rendering via pandoc.  It is written in Go, and an official container
image is provided.  We will need to use its API to create repositories
in its internal database as part of our new repo creation process.  It
supports cloning from the browse URL.  It has a landing page which we
can customize to our needs, as well as a repository exploration
feature.  It also supports code search across all projects.  Once it
is up and running, we can retire codesearch.openstack.org (and
redirect that URL to opendev.org).

We will not support authentication or any features other than browsing
and searching to start, however, we may want to use other features in
the future (such as wiki), so we should plan to eventually configure
the system to support those.

Example site: https://try.gitea.io/gitea/gitea

Only the HTTP/HTTPS git protocol will be supported, not the native git
protocol.  HTTP will redirect to HTTPS.

Redirects from git.openstack.org, git.zuul-ci.org, git.starlingx.io
will be established and maintained for at least a year.

As part of the initial migration, we will move all Zuul repositories
to use the "zuul/" prefix, along with any other similar moves which
may be ready.  We will offer all projects the option to rename their
repositories in the future, as we expect that it will take some time
for them to establish policies around how they would like to use their
prefixes.  For any future project renames, we will establish redirects
on opendev.org.

When we perform the move, we will force-merge (bypassing code review
and testing) changes to the .gitreview file of every project in the
system.


Implementation
==============

Assignee(s)
-----------

* corvus

Gerrit Topic
------------

Use Gerrit topic "opendev-gerrit" for all patches related to this spec.

.. code-block:: bash

    git-review -t opendev-gerrit

Work Items
----------

* Remove all git:// protocol references (eg in devstack)

* Select gitea or gitlist

* Create gitea/gitlist backend servers running software in containers

* Create new opendev.org frontend load balancers

* Configure Gerrit to replicate to opendev servers

* Configure redirects on backend servers

* Add review.opendev.org to DNS pointing at review.openstack.org IP addrs

* During outage:

  * Rename Gerrit server to review.opendev.org

  * Replace openstack logo with opendev logo

  * Update Gerrit apache config to redirect review.openstack.org to
    review.opendev.org

  * Update DNS to point git.openstack.org, git.starlingx.io,
    git.zuul-ci.org to opendev.org

  * Rename any projects ready to be renamed as part of the move

  * Force-merge changes

Repositories
------------

We may need new repositories related to operating gitea/gitlist.

Servers
-------

We will briefly have two git farms, but can retire the current git
farm at the completion of the migration.  We can operate with one or
two backends during setup and scale out to a full cluster size during
the outage.

DNS Entries
-----------

Many DNS changes will happen as described in the work items.  Some
will be manual changes to the openstack.org zone, most will be in
opendev.org which is managed via git.  Most DNS entries can be made in
advance of the cutover.

Documentation
-------------

The new code browsing system should be documented.  Instructions for
configuring redirects on project renames should be added.

Security
--------

This should not alter the security posture of any of the affected
services.

Testing
-------

New services can be tested with testinfra in system-config.

Dependencies
============

No dependencies.
