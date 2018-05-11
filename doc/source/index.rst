========================================================
 OpenStack Project Infrastructure Design Specifications
========================================================

Priority Efforts
================

These are the efforts we focus our review attention on first.  They
are a great way to get involved collaboratively with other
infrastructure developers.

.. toctree::
   :maxdepth: 1

   specs/task-tracker

Gerrit query for all changes related to priority efforts::

  status:open AND topic:storyboard-migration

https://review.openstack.org/#/q/status:open+AND+topic:storyboard-migration,n,z

Approved Design Specifications
==============================

These are specifications that have been approved; work may or may not
have started on these.  Reviewers will review related changes as time
permits.

.. toctree::
   :glob:
   :maxdepth: 1

   specs/deploy-ci-dashboard
   specs/jenkins-job-builder_2.0.0-api-changes
   specs/nodepool-drivers
   specs/puppet-module-functional-testing
   specs/refstack_dot_org
   specs/stackalytics
   specs/storyboard_worklists_boards
   specs/survey
   specs/translation_check_site
   specs/wiki_modernization
   specs/project-hosting

Help Wanted
===========

These are unassigned specifications: they are approved in concept
but have yet to attract any volunteers or have lost their volunteers
prior to completion. They may also be missing specific details like
a Story link, work items, impact, dependencies... Anyone proposing
changes implementing one of these is *strongly* encouraged to amend
the associated spec adding themself as an assignee (and fleshing out
additional details if necessary) while moving it into the approved
section of this index.

.. toctree::
   :glob:
   :maxdepth: 1

   specs/irc
   specs/puppet_4_prelim_testing
   specs/storyboard_integration_tests
   specs/storyboard_story_tags
   specs/storyboard_subscription_pub_sub
   specs/storyboard_task_branches

Implemented Design Specifications
=================================

These specifications have already been implemented and are listed here
for historical purposes.

.. toctree::
   :maxdepth: 1

   specs/ansible_puppet_apply
   specs/apps-site
   specs/artifact-signing
   specs/branch-automation
   specs/centralize-release-tagging
   specs/code-search
   specs/complete-reviewable-release-automation
   specs/config-repo-split
   specs/deploy-stackviz
   specs/dib-nodepool
   specs/doc-publishing
   specs/ethercalc
   specs/firehose
   specs/gerrit-2.11
   specs/gerrit-2.13
   specs/gerrit-contactstore-removal
   specs/infra-cloud
   specs/migrate_askbot
   specs/migrate_to_zanata
   specs/neutral-governance-website
   specs/newton-on-xenial
   specs/nodepool-workers
   specs/nodepool-zookeeper-workers
   specs/openstackci
   specs/ptgbot
   specs/public_hiera
   specs/publish-election-repo
   specs/puppet-modules
   specs/releases-openstack-org
   specs/server_base_template_refactor
   specs/shade
   specs/test-metrics-db
   specs/translation_setup
   specs/trystack-site
   specs/unified_mirrors
   specs/zuul_split
   specs/zuulv3
   specs/zuulv3-executor-security

Abandoned Design Specifications
===============================

These specifications had been approved previously but have not been
implemented, they have been abandoned.

.. toctree::
   :maxdepth: 1

   specs/logs-in-swift
   specs/maniphest
   specs/nodepool-launch-workers
   specs/pholio

Specifications Repository Information
=====================================

.. toctree::
   :maxdepth: 2

   README <readme>
   contributing
