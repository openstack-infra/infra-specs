::

  Copyright 2015, Hewlett Packard Enterprise Development LP

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=============================================
 Complete Reviewable Release Automation Work
=============================================

https://storyboard.openstack.org/#!/story/2000421

During Liberty we started building release tools to mostly automate a
process for reviewing release requests and then tagging releases (see
:doc:`centralize-release-tagging`). This spec describes the pieces
needed to complete that work so that when requests are merged the
release is tagged without human intervention.

Problem Description
===================

During Liberty we added tools to the openstack-infra/release-tools
repository to read the files in openstack/releases and publish
releases. Those tools only work correctly for libraries, and must be
run manually.

Proposed Change
===============

This cycle we need to expand support for all project types and
complete the automation so that the release tag is pushed to the
project repository by a job after the request is approved and merges
in the ``openstack/releases`` repository.

We also want to stop having the release tools update the bug and
blueprint status in Launchpad. Instead of updating the status of the
bug, we will leave a comment on the bug indicating when it was
released. Bug status should be updated to "Fix Released" when a patch
merges, instead of "Fix Committed". This will require updating the job
that runs when the release request merges in
``openstack/releases``.

A second job runs later to announce the new release, after the tarball
has been published. This job will run against the project repository,
so in order to ensure it has all of the information it needs, and to
ensure that there is a good record of the release activity, we will
store some data in the comment associated with the signed tag created
by the first job. We will need at least the series name so we can
determine the branch for the release history. We can assume zuul will
check out the project repository to the tagged commit, and determine
the current version from there.

Alternatives
------------

Continuing to tag releases by hand is causing delays in releases.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  - doug-hellmann
  - fungi

Gerrit Topic
------------

Use Gerrit topic "release-automation" for all patches related to this spec.

.. code-block:: bash

    git-review -t release-automation

Work Items
----------

#. Update jeepyb to set the bug status to "Fix Committed" instead of
   "Fix Released" (https://review.openstack.org/248922)

   * Need to communicate this change so that projects are aware of the
     process change
     (http://lists.openstack.org/pipermail/openstack-dev/2015-November/080288.html)

#. Update the options in openstack-infra/project-config to remove the
   redundant values setting jeepyb behavior to its new default
   (https://review.openstack.org/248923)

#. Create a new script in openstack-infra/release-tools to find the
   set of bugs mentioned as closed in the git commit messages for a
   release and add comments to those bugs giving the version number
   for when a fix for a bug was actually included in a
   release. (http://git.openstack.org/cgit/openstack-infra/release-tools/tree/release.sh#n84)

#. Create a new script in openstack-infra/release-tools to be run as
   part of the post-merge job for openstack/releases
   (http://git.openstack.org/cgit/openstack-infra/release-tools/tree/release_from_yaml.sh)

   That script needs to

   * identify the changed deliverable files in a given commit
   * determine the tags listed in the deliverable files that are not
     present in the git repositories
   * add the required signed tags to those repositories to trigger the
     existing release process, including any metadata needed for
     subsequent jobs (especially the series name)
   * invoke the bug update script created above after scanning each
     repository

   Other notes from the summit discussion

   * Requires support for GPG keys on the build machines, see
     :doc:`artifact-signing`.

   * Tag comments should include audit details like who requested the
     release (author & committer of the change to openstack/releases),
     a link back to that change-id, etc.

#. Set up a post-merge job for openstack/releases to run the script
   created in the previous step.

#. Create a new script in openstack-infra/release-tools to be run
   during the existing release build job (http://git.openstack.org/cgit/openstack-infra/release-tools/tree/announce.sh)

   That script needs to

   * generate and send release notes email based on the changes in
     each release

#. Update the release build job to call the script created in the
   previous step.

Repositories
------------

None

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

When the work is done we'll announce it on the mailing list.

Security
--------

This work builds on the key management work in :doc:`artifact-signing`.

Testing
-------

We tested ``release_from_yaml.sh``, ``release.sh`` and ``announce.sh``
during the Mitaka 1 milestone.

Dependencies
============

This work builds on the key management work in :doc:`artifact-signing`.
