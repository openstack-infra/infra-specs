::

  Copyright 2017 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

=========
Ethercalc
=========

Include the URL of your StoryBoard story:

https://storyboard.openstack.org/#!/story/2000839

The PTG will have unconference like scheduled rooms/activities. It has been
requested that we run an Ethercalc server to help facilitate the adhoc
scheduling of these rooms. The spreadsheet setup makes it easy to track
a matrix or rooms against times and Ethercalc's distributed editing feature
makes it a good choice for the PTG.

Problem Description
===================

Run an Ethercalc server to facilitate distributed scheduling of shared
(real world) resources.

Proposed Change
===============

We will deploy a new server, ethercalc01.openstack.org, using the puppet-nodejs
and puppet-redis modules to install nodejs+npm from which we can install
ethercalc. Ethercalc unlike etherpad does have a hard dependency on Redis so
puppet-redis will be used to deploy a colocated Redis server.

Alternatives
------------

We could use some on site physical scheduling medium. A common choice at
unconferences is post it notes on a grid. White boards would also work.
The trouble with these setups is that they are centralized to a physical
location making it difficult for people not on site or in different areas to
keep up with the current state.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  cboylan (clarkb)

Gerrit Topic
------------

Use Gerrit topic "ethercalc" for all patches related to this spec.

.. code-block:: bash

    git-review -t ethercalc

Work Items
----------

* Write puppet module for ethercalc
* Write operational documentation in system-config
* Deploy new server using new puppet module
* Update DNS when service is ready
* Announce service availability

Repositories
------------

Will need to create puppet-ethercalc. An alternative would be to tack this
on to the existing puppet-etherpad module as they are similar.

Servers
-------

Need to create ethercalc01.openstack.org.

DNS Entries
-----------

Need to create::

  ethercalc01.openstack.org A $IPFROMCLOUD
  ethercalc01.openstack.org AAAA $IPFROMCLOUD
  ethercalc.openstack.org CNAME ethercalc01.openstack.org

Documentation
-------------

Just docs related to the new service in the system-config docs.

Security
--------

This should be low risk. The service will be completely isolated from all
other services.

Testing
-------

We will reuse existing puppet testing templates to ensure the module works.

Dependencies
============

Will need new puppet-ethercalc repo. Note I have searched for an ethercalc
module on puppetforge and found none.
