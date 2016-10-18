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

   specs/ansible_puppet_apply
   specs/nodepool-zookeeper-workers
   specs/task-tracker
   specs/zuulv3
   specs/newton-on-xenial
   specs/doc-publishing

Gerrit query for all changes related to priority efforts::

  status:open AND (topic:puppet-apply OR topic:nodepool-zk OR topic:storyboard-migration OR branch:feature/zuulv3 OR topic:newton-xenial OR topic:afs-docs OR topic:gerrit-upgrade)

https://review.openstack.org/#/q/status:open+AND+%28topic:puppet-apply+OR+OR+topic:nodepool-zk+OR+topic:storyboard-migration+OR+branch:feature/zuulv3+OR+topic:newton-xenial+OR+topic:afs-docs+OR+topic:gerrit-upgrade%29,n,z

Approved Design Specifications
==============================

These are specifications that have been approved; work may or may not
have started on these.  Reviewers will review related changes as time
permits.

.. toctree::
   :glob:
   :maxdepth: 1

   specs/artifact-signing
   specs/branch-automation
   specs/complete-reviewable-release-automation
   specs/deploy-ci-dashboard
   specs/deploy-stackviz
   specs/doc-publishing
   specs/gerrit-2.13
   specs/jenkins-job-builder_2.0.0-api-changes
   specs/neutral-governance-website
   specs/nodepool-launch-workers
   specs/nodepool-workers
   specs/nodepool-zookeeper-workers
   specs/pholio
   specs/publish-election-repo
   specs/puppet_4_prelim_testing
   specs/puppet-module-functional-testing
   specs/refstack_dot_org
   specs/releases-openstack-org
   specs/shade
   specs/stackalytics
   specs/storyboard_integration_tests
   specs/storyboard_story_tags
   specs/storyboard_subscription_pub_sub
   specs/storyboard_task_branches
   specs/storyboard_worklists_boards
   specs/task-tracker
   specs/translation_check_site
   specs/unified_mirrors
   specs/wiki_modernization
   specs/zuul_split

Implemented Design Specifications
=================================

These specifications have already been implemented and are listed here
for historical purposes.

.. toctree::
   :maxdepth: 1

   specs/apps-site
   specs/centralize-release-tagging
   specs/code-search
   specs/config-repo-split
   specs/dib-nodepool
   specs/firehose
   specs/gerrit-2.11
   specs/infra-cloud
   specs/migrate_askbot
   specs/migrate_to_zanata
   specs/openstackci
   specs/public_hiera
   specs/puppet-modules
   specs/server_base_template_refactor
   specs/test-metrics-db
   specs/translation_setup
   specs/trystack-site

Abandoned Design Specifications
===============================

These specifications had been approved previously but have not been
implemented, they have been abandoned.

.. toctree::
   :maxdepth: 1

   specs/logs-in-swift
   specs/maniphest

Specifications Repository Information
=====================================

.. toctree::
   :maxdepth: 2

   README <readme>
   contributing


Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
