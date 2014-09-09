::

  Copyright (c) 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Split config into project-config and system-config
==================================================

Story: https://storyboard.openstack.org/#!/story/167

This describes a further refactor of the openstack-infra/config repo
to split system administration from project configuration.

Problem Description
===================

The config repo, particularly the openstack_project module, contains
both information on how our particular systems are operated (what we
might call system administration information) as well as the
configuration of those systems specific to their use hosting the
OpenStack project.

Much of the latter might be considered more like configuration data.
There are a number of people within the OpenStack project competent to
review changes to project and CI system configuration, but they are
overwhelmed by the system administration related changes in the same
repo.

Likewise, those who would like to help refactor the system
administration in the config repo to be more useful to other projects
and downstream users are not particularly interested in reviewing new
project changes.  Further, colocating the two kinds of information
makes it potentially harder for downstream users of the config repo.
Future refactoring of the system administration portions of the
openstack_project module are anticipated by this specification, but
are not described here.

Proposed Change
===============

The following parts of the config repo will be extracted (preserving
history where possible) into a new git repo called
``openstack-infra/project-config``::

  modules/openstack_project/files/jenkins_job_builder/config/*
  modules/openstack_project/files/zuul/openstack_functions.py
  modules/openstack_project/files/zuul/layout.yaml
  modules/openstack_project/files/zuul/layout-dev.yaml
  modules/openstack_project/files/accessbot/channels.yaml
  modules/openstack_project/files/gerrit/acls/*
  modules/openstack_project/files/gerrit/notify_impact.yaml
  modules/openstack_project/files/nodepool/scripts/*
  modules/openstack_project/files/review-dev.projects.yaml
  modules/openstack_project/files/review.projects.yaml
  modules/openstack_project/files/slave_scripts/*
  modules/openstack_project/files/specs/index.html
  modules/gerritbot/files/gerritbot_channel_config.yaml

Other files may move into the repo in the future as well as it becomes
clear how to separate them from system administration concerns
(ongoing work with multiple hiera data files may prove useful here).

Puppet configuration will then be updated to maintain a vcsrepo
checkout of the project-config repo (whose URL will be configurable to
support downstream use) and either reference or install files as
needed from it.  Puppet templates (.erb files) will not be supported.
Some puppet actions are triggered by changes to these files, and that
will still need to be accomodated.

The files which have been added to ``project-config`` will be removed
from the config repo, and the config repo itself will be renamed to
``system-config``.

Alternatives
------------

N/A.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  corvus (If you want to work on this, contact me!)

Work Items
----------

* Git filter-branch the listed files into a project-config repo
* Propose a change to adjust puppet config to install and reference
  the project-config repo on related hosts, and remove related files.
* Freeze changes to related files.
* Update project-config with latest data from config.
* Land the above change.
* Unfreeze.
* Rename config to system-config, and update the puppet master to use
  the new repo name.

Repositories
------------

Create openstack-infra/project-config from a git filter-branch of config.
Rename openstack-infra/config to openstack-infra/system-config

Servers
-------

N/A.

DNS Entries
-----------

N/A.

Documentation
-------------

The infra/config docs will need to be updated to reference the new
repo and locations of files.  A new sphinx macro will need to be made
to support linking to the new repo within docs.  An announcement
should be made to both the infra and dev lists.

Security
--------

None of the listed files have passwords in them and no template
parsing is immediately anticipated.  If templates (to support, eg,
passwords in included files) are to be added to the project-config
repo and puppet is instructed to parse them, it could be instructed by
a template in the project-config repo to insert a password into a
wrong location and thereby either accidentally or intentially exposing
it.  If templating is used, then reviewers of the project-config repo
should be selected with security-related trust in mind and reminded of
the potential for exposure.

Testing
-------

Private environment testing after the creation of the initial
project-config repo is likely to be the best way to test the change.

Dependencies
============

Related to this specification to split out puppet modules, but does
not depend on it: https://review.openstack.org/#/c/99990/
