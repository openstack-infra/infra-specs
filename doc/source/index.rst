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

   specs/logs-in-swift
   specs/dib-nodepool
   specs/openstackci
   specs/migrate_to_zanata
   specs/maniphest

Gerrit query for all changes related to priority efforts::

  status:open AND (topic:enable_swift OR topic:dib-nodepool OR topic:zanata OR topic:downstream-puppet OR topic:maniphest)

https://review.openstack.org/#/q/status:open+AND+%28topic:enable_swift+OR+topic:dib-nodepool+OR+topic:zanata+OR+topic:downstream-puppet+OR+topic:maniphest%29,n,z

Approved Design Specifications
==============================

These are specifications that have been approved; work may or may not
have started on these.  Reviewers will review related changes as time
permits.

.. toctree::
   :glob:
   :maxdepth: 1

   specs/ansible_puppet_apply
   specs/centralize-release-tagging
   specs/code-search
   specs/doc-publishing
   specs/infra-cloud
   specs/nodepool-launch-workers
   specs/nodepool-workers
   specs/public_hiera
   specs/puppet_4_prelim_testing
   specs/puppet-module-functional-testing
   specs/refstack_dot_org
   specs/shade
   specs/storyboard_integration_tests
   specs/storyboard_story_tags
   specs/storyboard_subscription_pub_sub
   specs/storyboard_task_branches
   specs/zuul_split
   specs/zuulv3

Implemented Design Specifications
=================================

These specifications have already been implemented and are listed here
for historical purposes.

.. toctree::
   :maxdepth: 1

   specs/apps-site
   specs/config-repo-split
   specs/migrate_askbot
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
