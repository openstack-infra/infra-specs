::

  Copyright 2016 Hewlett Packard Enterprise Development LP

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode


===========================
Pholio Service Installation
===========================

The `OpenStack UX team`_ have determined that the Pholio_ application in
the Phabricator_ suite best meets their needs and wish to have this service
installed.

.. _OpenStack UX team: https://wiki.openstack.org/wiki/UX
.. _Pholio: https://www.phacility.com/phabricator/pholio/
.. _Phabricator: https://www.phacility.com/phabricator/

Problem Description
===================

The OpenStack UX team are currently using Invision_ to provide feedback on
mocks.  The team have determined that Pholio best meets their needs and have
decided that they are planning to move to Pholio `in the near future`_.

.. _Invision: https://openstack.invisionapp.com/
.. _in the near future: https://wiki.openstack.org/wiki/UX#Getting_Involved


Proposed Change
===============

This spec only covers the deployment of the Pholio service. Data migration
from Invision to Pholio will have to be considered and handled separately.

Conceptually, this is straight forward. We need to install a phabricator
server and configure it to turn off the services we don't want.

The puppet-phabricator repo is already imported into infra that is based on
work from a POC. It needs to be fleshed out some. It does already contain
the config in `local.json.erb` to turn off the features OpenStack does not want.
Examples of features from phabricator that UX does not want to run are the
Code Review tool Differential since we have gerrit, the Group Messaging tool
Conpherence, because we have IRC, the code hosting tool Diffusion, since
we already have the cgit farm, and the wiki tool, Phriction, since we have
mediawiki.

For authentication, phabricator needs to be connected first to Launchpad.
phabricator does not have openid support - but we can use
`libapache2-mod-auth-openid` to provide OpenID authentication using Launchpad as
the provider via `REMOTE_USER` info. There is a phabricator plugin that
consumes `REMOTE_USER`, so we should use `libapache2-mod-auth-openid` for auth.

It is envisioned that authentication will be against OpenStackID_ although if
that remains incompatible with Phabricator, `Ubuntu Login`_ will be used instead.

.. _OpenStackID: https://openstackid.org/
.. _Ubuntu Login: https://login.ubuntu.com/

Alternatives
------------

I'm currently unaware of what alternatives the UX team considered. It would be
useful to have them listed here for future reference.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Craige_.

.. _Craige: https://www.openstack.org/community/members/profile/22079

Gerrit Topic
------------

Use Gerrit topic "pholio" for all patches related to this spec.

.. code-block:: bash

    git-review -t pholio

Work Items
----------

Write a puppet module that performs the following:

* Installs required packages
* Installs SSL certificates
* Pulls Phabricator_ from github
* Pulls Arcanist_ from github
* Pulls libphutil_ from github
* Pulls libphremoteuser_ from github
* Enables libphremoteuser_ in Phabricator_
* Creates / copies across required configuration files
* Initialises the database
* Performs the Phabricator storage upgrade
* Enables Apache2 modules (mod_rewrite, libapache2-mod-auth-openid)
* Configures Phabricator_ for the OpenStack environment / UX team needs
* Ensures services are running

.. _Phabricator: https://github.com/phacility/phabricator
.. _Arcanist: https://github.com/phacility/arcanist
.. _libphutil: https://github.com/phacility/libphutil
.. _libphremoteuser: https://github.com/psigen/libphremoteuser

Pre-installation, an infra-root will need to build pholio01.

Post installation, root will be required to make one user, I recommend the
PTL, a Phabricator administrator. The syntax on pholio01, assuming the PTL is id 1, would
be similar to this:

.. code-block:: sql

    mysql> update phabricator_user.user set isAdmin = 1 where id = 1;

Repositories
------------

openstack-infra/puppet-phabricator
openstack-infra/system-config

Servers
-------

pholio01.openstack.org

DNS Entries
-----------

pholio01.openstack.org (A record)
pholio.openstack.org (CNAME to pholio01)

Documentation
-------------

There is presently no documentation for this service, so it will need to be
written.

Security
--------

There are no specific security-related concerns for a deployment of the Pholio
service as we shouldn't expect to host sensitive information on the server.

Testing
-------

We'll need functional testing for sure. We should probably consider a
staging server that we can use to test new config changes.

Dependencies
============

None
