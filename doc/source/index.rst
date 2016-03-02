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
   specs/dib-nodepool
   specs/infra-cloud
   specs/logs-in-swift
   specs/maniphest
   specs/openstackci
   specs/zuulv3

Gerrit query for all changes related to priority efforts::

  status:open AND (topic:puppet-apply OR topic:dib-nodepool OR topic:infra-cloud OR topic:enable_swift OR topic:maniphest OR topic:downstream-puppet OR branch:feature/zuulv3)

https://review.openstack.org/#/q/status:open+AND+%28topic:puppet-apply+OR+topic:dib-nodepool+OR+topic:infra-cloud+OR+topic:enable_swift+OR+topic:maniphest+OR+topic:downstream-puppet+OR+branch:feature/zuulv3%29,n,z

Approved Design Specifications
==============================

These are specifications that have been approved; work may or may not
have started on these.  Reviewers will review related changes as time
permits.

.. toctree::
   :glob:
   :maxdepth: 1

   specs/artifact-signing
   specs/complete-reviewable-release-automation
   specs/deploy-ci-dashboard
   specs/deploy-stackviz
   specs/doc-publishing
   specs/jenkins-job-builder_2.0.0-api-changes
   specs/nodepool-launch-workers
   specs/nodepool-workers
   specs/public_hiera
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
   specs/translation_check_site
   specs/translation_setup
   specs/unified_mirrors
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
   specs/gerrit-2.11
   specs/migrate_askbot
   specs/migrate_to_zanata
   specs/puppet-modules
   specs/server_base_template_refactor
   specs/test-metrics-db
   specs/trystack-site


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
