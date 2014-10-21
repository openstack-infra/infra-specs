::
  Copyright 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Separate puppet-module projects
===============================

Rather than having all puppet modules in a single project, each should be in
its own project.

Problem Description
===================

The openstack-infra/config project is a monolithic ecosystem of puppet modules.
This makes it hard to pick-and-choose specific modules to use downstream such
as asterisk or graphite when the larger ecosystem is not wanted, and also makes
it easier to accidentally write changes in the wrong puppet module. Some
modules, such as jenkins or gerrit, should be generic and reusable across
deployments while others (the openstack_project module and possibly others in
the future) are very site-specific. Making the site-specific modules a
downstream consumer of the generic modules will more clearly delineate where
site-specific changes should be made and make the generic modules more easily
re-consumable.

Proposed Change
===============

Put each (or at least most) puppet modules in their own project.

Alternatives
------------

* Leave openstack-infra/config as is (monolithic)
* Create several projects, each with a logical group of modules
  (e.g. logstash + log_processor + elasticseacrh + kibana)

Implementation
==============

The openstack_project module and manifests/site.pp will remain in config.
The devstack_host, remove_nginx, and salt modules will be deleted.
All other modules will be put in their own project.

Assignee(s)
-----------

Primary assignee(s):
  - jesusaurus
  - nibalizer

Additional assignee(s):
  - bodepd

Work Items
----------

We will need to create integration tests to ensure that api compatibility is
maintained between puppet modules. This will be difficult to create without
having multiple modules, but is important to have before actively developing
the new modules. After breaking out the first module we should create an
integration test to make sure that changes to it do not break
openstack-infra/config and then we can continue with the rest of the modules.

The following process must be done for each module separately:

#. Freeze changes to a specific module within config

#. Isolate the module's history with git-subtree:

    .. code-block:: bash

        git subtree split --prefix=modules/$MODULE --branch=$MODULE

   * git-subtree will do the right thing by re-rooting the resulting code at
     modules/$MODULE but will require git >= 1.8.3.2 (and even then may be in
     a separate contrib package)

#. Create a temporary project (e.g. personal github repo) to host the new base repo

    * Example url: https://github.com/$YOU/puppet-$MODULE.git

#. Push the new branch to the temporary project created above

    .. code-block:: bash

        git remote add github https://github.com/$YOU/puppet-$MODULE.git
        git push github $MODULE:master

#. Add a new gerrit project for the module in project-config (using the temporary project as upstream)

    * Follow example here: https://review.openstack.org/#/c/130619/

#. Modify system-config/modules.env to install the module from the new gerrit project
   and add the new project to the puppet integration tests. Remove the old module
   from openstack_infra/config with rm.

   * We should continuously deploy the master branch
   * See example here: https://review.openstack.org/#/c/130634/

#. Propose a review to add some of the files that are needed by the module:

   * Rakefile ::

       require 'rubygems'
       require 'puppetlabs_spec_helper/rake_tasks'
       require 'puppet-lint/tasks/puppet-lint'
       PuppetLint.configuration.fail_on_warnings = true
       PuppetLint.configuration.send('disable_80chars')
       PuppetLint.configuration.send('disable_autoloader_layout')
       PuppetLint.configuration.send('disable_class_inherits_from_params_class')
       PuppetLint.configuration.send('disable_class_parameter_defaults')


   * Modulefile ::

       name 'openstack-pip'
       version '0.0.1'
       source 'git://git.openstack.org/openstack-infra/puppet-pip.git'
       author 'openstackci'
       license 'Apache 2.0'
       summary 'Puppet module for the pip package manager'
       description 'This module provides providers to help pip install well. Only tested against 2.7.'
       project_page 'https://github.com/openstack-ci/puppet-pip'


   * README.md ::

       # OpenStack Pip Module

       Adds a pip3 provider to handle pip installs in the Python3 ecosystem


       ## Example

       ```puppet

       package {'flask':
         ensure   => latest,
         provider => 'pip3'
       }
       ```

   * Metadata.json ::

       {
         "name": "openstackci-pip",
         "version": "0.0.1",
         "author": "Openstack CI",
         "summary": "Pip provider",
         "license": "ASL 2.0",
         "source": "git://git.openstack.org/openstack-infra/puppet-pip.git",
         "project_page": "https://github.com/openstack-ci/puppet-pip",
         "issues_url": "https://github.com/openstack-ci/puppet-pip",
         "dependencies": []
       }

    # Note that determining dependencies may not be immediately obvious,
    we must count on the code review process to ensure that we've done
    this right.


#. Lather, rinse, and repeat


Repositories
------------

* openstack-infra/puppet-accessbot
* openstack-infra/puppet-asterisk
* openstack-infra/puppet-bugdaystats
* openstack-infra/puppet-bup
* openstack-infra/puppet-cgit
* openstack-infra/puppet-drupal
* openstack-infra/puppet-elastic_recheck
* openstack-infra/puppet-elasticsearch
* openstack-infra/puppet-exim
* openstack-infra/puppet-gerrit
* openstack-infra/puppet-gerritbot
* openstack-infra/puppet-github
* openstack-infra/puppet-graphite
* openstack-infra/puppet-iptables
* openstack-infra/puppet-jeepyb
* openstack-infra/puppet-jenkins
* openstack-infra/puppet-kibana
* openstack-infra/puppet-lodgeit
* openstack-infra/puppet-log_processor
* openstack-infra/puppet-logrotate
* openstack-infra/puppet-logstash
* openstack-infra/puppet-mailman
* openstack-infra/puppet-mediawiki
* openstack-infra/puppet-meetbot
* openstack-infra/puppet-mysql_backup
* openstack-infra/puppet-nodepool
* openstack-infra/puppet-openssl
* openstack-infra/puppet-openstackid
* openstack-infra/puppet-packagekit
* openstack-infra/puppet-pip
* openstack-infra/puppet-planet
* openstack-infra/puppet-puppetboot
* openstack-infra/puppet-recheckwatch
* openstack-infra/puppet-redis
* openstack-infra/puppet-releasestatus
* openstack-infra/puppet-remove_nginx
* openstack-infra/puppet-reviewday
* openstack-infra/puppet-salt
* openstack-infra/puppet-snmpd
* openstack-infra/puppet-ssh
* openstack-infra/puppet-ssl_cert_check
* openstack-infra/puppet-statusbot
* openstack-infra/puppet-storyboard
* openstack-infra/puppet-subversion
* openstack-infra/puppet-sudoers
* openstack-infra/puppet-tmpreaper
* openstack-infra/puppet-ulimit
* openstack-infra/puppet-unattended_upgrades
* openstack-infra/puppet-unbound
* openstack-infra/puppet-user
* openstack-infra/puppet-zuul

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

Each new module will have its own documentation.

Security
--------

None

Testing
-------

* Unit tests:
  We currently only lint and syntax-check the modules in config. They should
  also have rspec-beaker and server-spec tests written for them (even if we
  don't move them to their own project).

* Integration tests:
  We need to test that changes to the new projects do not break config (such as
  with changes to a class's parameter list).

Developer Impact
================

By migrating from a single project to many projects, developers will no longer
be able to atomically change multiple modules at the same time. This means that
changes that touch multiple modules will have to be made in a backwards-compatible
way with soft dependencies between changes (such as two changes mentioning each
other in their commit messages). Requiring backwards-compatible changes will
also make it easier for downstream consumers to use the modules.

Dependencies
============

None
