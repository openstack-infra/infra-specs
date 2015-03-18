::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Migrate ask.openstack.org to infra
==================================

https://storyboard.openstack.org/#!/story/2000158

The OpenStack Q&A site located at ask.openstack.org currently
running in an instance deployed by a third-party. The main
goal of this migration to make the entire deployment process
repeatable, and move the project to proper infrastructure
puppet repositories.

Problem Description
===================

The ask.openstack.org site was deployed manually and contains
the minimal application stack required for running the site.
Currently the site is missing regular backups, security
updates and deployment documentation.

Proposed Change
===============

Create the missing puppet modules, prepare the changes in
openstack-infra/system-config repo.

Alternatives
------------

Leave ask.openstack.org as is.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  marton-kiss

Gerrit Topic
------------

Use Gerrit topic "askbot-site" for all patches related to this spec.

.. code-block:: bash

    git review -t askbot-site

Work Items
----------

* Lower ask.openstack.org DNS TTL to 300
* Create puppet-askbot split-out module
* Add vamsee-solr module and puppet-askbot to modules.env
* Make the system-config changes, add ask.pp
* Add SSL certificates and passwords to hiera
* Launch new ask.openstack.org server
* Restore database and static files from original ask.openstack.org site
* Silent testing with /etc/hosts override
* Backup / restore of original ask.openstack.org data
* Update DNS entry of ask.openstack.org with the new server address
* Redirect html traffic using nginx to new ask.openstack.org to avoid db sync issues
* Restore ask.openstack.org DNS TTL to 3600

Repositories
------------

A new puppet-askbot repository will need to be created, along with updates to
system-config to consume this module.

Servers
-------

An ask.openstack.org will need to be created.

DNS Entries
-----------

The ask.openstack.org zone must be point to the newly created server as the
last step of this migration process.

Documentation
-------------

Askbot documentation need to be added to ci.openstack.org documentation.

Security
--------

The services will run on Ubuntu, so core operating system not requires any
special attention.

The application stack have some elements that must be deployed from
tar.gz or pypy instead of OS packages:

* Apache Solr (4.7.2)
* askbot

Testing
-------

Askbot don't have integration tests implemented. After instance creation
and initial data migration, I suggest to do a 1-2 week long silent test
of the UI and address upcoming bugs during that period.

Backup / recovery steps
=======================

Backup assets
-------------

Askbot holds all of the site data in a postgresql database, and all static files
in the filesystem. In case of the existing deployment it is available at
/srv/ask/dumps directory. The dumps are created by the shell script
srv/ask/node2/cron/backup.sh executed daily by cron.

Invoke management commands
--------------------------

The management cli tool is available under /srv/askbot-sites/slot0/config, so
to invoke management commands you need to enter this directory first.

.. code-block:: bash

    $ cd /srv/askbot-sites/slot0/config
    $ python manage.py <command> <args>

Stop celeryd and apache, jetty services
---------------------------------------

.. code-block:: bash

    $ sudo service askbot-celeryd stop
    $ sudo service apache2 stop
    $ sudo service jetty stop

Restore database dump
---------------------

Recreate an empty database:

.. code-block:: bash

    $ sudo su - postgres -c 'dropdb askbotdb'
    $ sudo su - postgres -c 'createdb askbotdb --owner=ask'

Restore from backup:

.. code-block:: bash

    $ psql -d askbotdb -h localhost -U ask -W -f /path/to/last-dump.sql

`Notice` you will be prompted for the ask_db_password from hiera.

Sync db and migrate:

.. code-block:: bash

    $ cd /srv/askbot-sites/slot0/config
    $ sudo python manage.py syncdb
    $ sudo python manage.py migrate

`Notice` this step is required to apply schema changes between askbot versions.

Start celeryd
-------------

.. code-block:: bash

    $ sudo service askbot-celeryd start
    $ sudo service apache2 start

Rebuild solr indexes
--------------------

.. code-block:: bash

    $ sudo service jetty start
    $ cd /srv/askbot-sites/slot0/config
    $ sudo python manage.py askbot_rebuild_index -l en
    $ sudo python manage.py askbot_rebuild_index -l zh

Test the solr deployment, query string "sahara":

.. code-block:: bash

    $ curl "http://127.0.0.1:8983/solr/core-en/select/?fq=django_ct%3A%28askbot.thread%29&rows=10&q=%28sahara%29&start=0&wt=json&fl=%2A+score"

`Notice` this query must return a non-empty resultset.

Restart celeryd
---------------

.. code-block:: bash

    $ sudo service askbot-celeryd restart
    $ sudo service apache2 restart

Restore static files
--------------------

Static files must be extracted into /srv/askbot-sites/slot0/upfiles directory.
It mostly holds profile pictures, and site logo, so if pictures not showing up
in the site those files are missing, or have a wrong file permission.

.. code-block:: bash

    $ cd /srv/askbot-sites/slot0
    $ sudo rm -rf upfiles
    $ sudo tar xf /path/to/last-upfiles.tar --strip-components=2

Dependencies
============

We are using vamsee-solr module 0.0.7 from puppetforge, and it is forcing
Us to use solr 4.7.2 because 4.10.x requires some extra patches to work and
this upgrade also means a schema change.
