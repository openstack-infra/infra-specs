::

  Copyright 2015, Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

============================
 Centralize Release Tagging
============================

Include the URL of your StoryBoard story:

https://storyboard.openstack.org/#!/story/2000292

We would like to manage release tags through reviews, just as we do
with other changes. Unfortunately, gerrit does not yet support
reviewing tags directly. This spec describes an alternate solution
using a separate git repository, with some manual tools for short-term
use and some ideas about automation for later.

Problem Description
===================

Until now we have encouraged project teams to prepare their own
library releases as new versions of projects were needed. We've
started running into a couple of problems with that, with releases
not coming often enough, or at a bad time in the release cycle, or
with version numbering not being applied consistently, or without
proper announcements. To address these issues, the release management
team is proposing to create a small team of library release managers
to handle the process around all library releases (clients,
non-application projects, middleware, Oslo libraries, etc.). This
will give us a chance to collaborate and review the version numbers
for new releases, as well as stay on top of "stale" libraries with
fixes or features that sit unreleased for a period of time. It will
also be the first step to creating an automated review process for
release tags.

Proposed Change
===============

Create a new git repository with a name like
``openstack/releases``.

For each set of deliverables, we will use one YAML file in
``openstack/releases`` per release series to hold all of the metadata
for all releases of that deliverable. For each release, we need to
track:

* the launchpad project name (such as ``oslo.config``)
* the series (Kilo, Liberty, etc.)
* for each repository

  * the name (such as ``openstack/oslo.config``)
  * the hash of the commit to be tagged

* the version number to use
* highlights for the release notes email (optional)

We will track this metadata for the history of all releases of the
deliverable, so we can use the same data to render a set of release
history documentation.

The release tools need the series name to update launchpad, and they
also need the branch name to check the history of tags visible from
that branch to do some validation.  Since git tags apply to a commit,
and are not branch-aware, we can use the series name in the filename
and have the tagging script either assume some defaults (if it does
not find a stable branch of that name use master) or recognize which
series is currently on master.

The file will be named based on the deliverable to be tagged, so
releases for ``liberty`` from the ``openstack/oslo.config`` repository
will have a file in ``openstack/releases`` called
``liberty/oslo.config.yaml``. Releases of the same deliverable from
the ``stable/kilo`` branch will be described by
``kilo/oslo.config.yaml``.

For example, one version of
``liberty/oslo.config.yaml`` might contain::

   ---
   launchpad: oslo.config
   releases:
     - version: 1.12.0
       projects:
         - repo: openstack/oslo.config
           hash: 02a86d2eefeda5144ea8c39657aed24b8b0c9a39

and then for the subsequent release it would be updated to contain::

   ---
   launchpad: oslo.config
   releases:
     - version: 1.12.1
       projects:
         - repo: openstack/oslo.config
           hash: 0c9113f68285f7b55ca01f0bbb5ce6cddada5023
       highlights: >
          This release includes the change to stop importing
          from the 'oslo' namespace package.

For deliverables with multiple repositories, the list of projects
would contain all of them. For example, the Neutron deliverable might
be described by ``liberty/neutron.yaml`` containing:

::

   ---
   launchpad: neutron
   releases:
     - version: 7.0.0
       projects:
         - repo: openstack/neutron
           hash: somethingunique
         - repo: openstack/neutron-fwaas
           hash: somethingunique
         - repo: openstack/neutron-lbaas
           hash: somethingunique
         - repo: openstack/neutron-vpnaas
           hash: somethingunique

For Phase I, we won't have much true automation using these files, and
someone from the release team will need to run a tool to read the file
and apply the appropriate tags. That tool should be a straightforward
modification to the existing ``release_postversion.sh`` script.

For future phases we can investigate having enough signed data in the
YAML file to let an automated job apply the tag when the request is
approved.

Alternatives
------------

Allow Project Owners to Continue Tagging Releases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We will continue to have issues with incorrect semver use and poorly
timed releases.

Have Project Owners Request Releases via Email/IRC
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rather than going through the bureaucracy of managing the requests via
git we could just use email and IRC as we have been doing. However,
that would not bring us closer to automating releases after the
requests are reviewed, which is our ultimate goal.

Update Gerrit to Support Reviewing Tags
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Apparently the Gerrit project team is interested in the feature, but
it isn't a high priority. We could consider this a Phase III for the
project if someone from our community becomes available to work on
it. On the other hand, we would need to find another way to track
releases after-the-fact, and tags in one repository do not handle
multi-repo deliverables such as neutron.

Use Branches
~~~~~~~~~~~~

Earlier drafts of this proposal suggested using different branches of
the ``openstack/releases`` repository to manage releases from
different branches of the upstream projects. That forces us to create
all the same branches in the new repository that are needed in any
repository for which releases are being managed. Since not all of them
will use the same branching structure, this is not optimal.

One File Per Repository
~~~~~~~~~~~~~~~~~~~~~~~

Use a file named based on the git repository to be tagged, so
releases from the ``master`` branch of the ``openstack/oslo.config``
repository would have a file in ``openstack/releases`` called
``openstack/oslo.config/master/releases.yaml``. Releases for the same
repository from the ``stable/kilo`` branch will be described by
``openstack/oslo.config/stable/kilo/releases.yaml``

For example, one version of
``openstack/oslo.config/master/releases.yaml`` might contain::

   --
   series: liberty
   hash: 02a86d2eefeda5144ea8c39657aed24b8b0c9a39
   version: 1.12.0

and then for the subsequent release it would be updated to contain::

   --
   series: liberty
   hash: 0c9113f68285f7b55ca01f0bbb5ce6cddada5023
   version: 1.12.1
   highlights: >
      This release includes the change to stop importing
      from the 'oslo' namespace package.

Multi-repo deliverables such as Neutron could use separate files,
submitted together.

This scheme does not allow us to easily produce web pages showing the
release histories.

Single File With All Branches
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rather than maintaining a separate file for each branch, we could use
a single file and list all branches in it. This makes it a little more
complicated to detect new changes, though, and has the same problem as
appending all releases to a single file -- the tool that applies the
tags needs to check all of them, and the list will only grow over
time.

Branch After Project in the Path
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We could use file names like ``oslo.config/kilo.yaml`` instead of
``kilo/oslo.config.yaml``. That would place all of the files from the
same deliverable in a directory together. However, it is more likely
that we will focus on the contents of a series rather than historical
releases of an individual project.

Record Launchpad Names in the Governance Repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There is a separate list of projects in the governance repository, and
we could list some of the data about projects that doesn't change
there. That would require the tool download the relevant files,
though, and would not help us with scripting releases for projects not
under TC governance.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  doug-hellmann

I'm comfortable setting up a new repository and building in-tree
tools. I may need help with some of the validation job work.

Gerrit Topic
------------

Use Gerrit topic "centralize-release-tagging" for all patches related to this spec.

.. code-block:: bash

    git-review -t centralize-release-tagging

Work Items
----------

1. Create the ``openstack/releases`` repository, with a README and
   template YAML file.
2. Create a new tool (or update an existing script) in
   ``openstack-infra/release-tools`` to read the YAML files from
   ``openstack/releases`` and run the interactive release script we
   use now.
3. Create a basic validation tool to read the YAML files and provide a
   check job. We can't do a lot to validate the requested tag, beyond
   noticing that it already exists, but we can make sure all of the
   needed parts are there and can be parsed properly, and we can run a
   report showing the unreleased changes, what pbr thinks the version
   should be, and whether the version means a major jump in the series
   to help the reviewer (these are all things I do by hand right now).
4. Make ``release_postversion.sh`` smarter about figuring out the
   branch for validating the proposed version number.

Repositories
------------

``openstack/releases`` will hold the release request files.

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

We will document the process in the README in ``openstack/releases``
to start, and then in the Project Driver's Guide portion of
infra-manual.

Security
--------

During Phase I releases will still be tagged by people with
established trust rings. For future phases where the tagging is
handled by a post-merge job we will want to do some validation of
signed data in the request file.

Testing
-------

We have fairly robust release tools now, but we will want to test some
of the new tools for working with the YAML files.

Dependencies
============

- https://review.openstack.org/189856 -- Creating a library-release
  team with Gerrit ACLs to push tags to repositories containing
  libraries.

- http://lists.openstack.org/pipermail/openstack-dev/2015-June/066346.html --
  Mailing list thread initiating the discussion.
