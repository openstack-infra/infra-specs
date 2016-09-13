::

  Copyright 2016, Red Hat, Inc.

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
 Automate Creating Branches
============================

Include the URL of your StoryBoard story:

https://storyboard.openstack.org/?#!/story/2000722

The release management team would like to manage branches through a
reviewable process, like we do with tags.

Problem Description
===================

The process for creating stable and feature branches is manual and
ill-defined, both of which make it error-prone. We have partially
automated the process of creating the stable branches, but a release
manager has to remember to run the script after tagging an RC1 release
for a project. We would like to manage the branches using deliverable
files in openstack/releases, like we do now with tags.

Proposed Change
===============

Extend the schema of the deliverable files in openstack/releases to
include information about branches as well as releases. Add a new
"branches" key at the top level, parallel to "releases". The contents
of the "branches" section are a list of branches with "name" and
"location" values.

The "name" is a string value that will become the name of the new
branch without any modifications made to it (no prefix added
automatically, no lower-casing, etc.).

The "location" value depends on the name. If a branch name starts with
``stable/`` then the location must be either an existing version tag
or the most recently added version number under the "releases" list
(allowing a tag and branch to be submitted together).

If a branch name starts with ``feature/`` then the "location" must be
a mapping between the target repository name and the SHA of a commit
already in the target repository.

The names of branches will be reviewed by human reviewers.

Even after implemented we would retain the ability for humans to
create branches manually, but this process should make *most* branch
creation simpler and more reliable.

Alternatives
------------

1. Continue to create branches by hand, using the informal request
   system (email or IRC) and RC1 tagging.

   This approach remains error prone.

2. Add a "branch" key to the releases section of the deliverable file.

   This approach would not support feature branches.

Implementation
==============

Assignee(s)
-----------

Primary assignee: doughellmann

Gerrit Topic
------------

Use Gerrit topic "branch-automation" for all patches related to this
spec.

.. code-block:: bash

    git-review -t branch-automation

Work Items
----------

1. Extend the validation of deliverable files to cover branches, using
   the rules described above.

2. Extend the validation of deliverable files to ensure that RC1
   versions also have stable branches added at the same time.

3. Extend the validation of deliverable files to ensure that if a
   branch exists, it starts at the point specified in the
   location. (This ensures that we import old data correctly.)

4. Either extend the tag-releases job (or add a new job) to look at
   the branch list in the deliverable file and create the last one in
   the list if it does not exist. This approach is consistent with our
   treatment of adding tags one at a time.

   Some relevant code for working with branches already exists in
   openstack-infra/project-config/jenkins/scripts/release-tools/make_stable_branch.sh,
   .../branch_from_yaml.sh, and
   openstack-infra/release-tools/make_feature_branch.sh. Those scripts
   will be reused or obsoleted by this work.

   Some projects, like devstack and grenade, may require extra work
   when a branch is created. We could use a hook script from inside of
   the repository being branched to make additional changes in the
   repository after it is branched, or we could embed that logic in
   the branching script itself using repository-specific logic. The
   latter is more secure, but the former is more flexible.

Repositories
------------

No new repositories needed.

Servers
-------

The branching will be done by the same signing node that currently
applies tags.

DNS Entries
-----------

No new entries.

Documentation
-------------

1. Update the README.rst in openstack/releases to describe the schema.
2. Update the project team guide to cover the new procedure for
   requesting a branch.

Security
--------

The signing node should not need any additional credentials, though
the associated gerrit account may need more permissions than it has.

Testing
-------

We can automate the testing of the validation code and manually test
creating the branches using the openstack/release-test repository.

Dependencies
============

None
