::
  Copyright 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode



==================================================
Refactor openstack_project::{server,base,template}
==================================================

Problem Description
===================

Right now there are three 'main' classes in puppet. They are

openstack_project::server
openstack_project::base
openstack_project::template

All three of these control resources that matter to most systems.  Server
contains template, template contains base. Then usually an application
has a class that wraps server.

The problem with this is that if you want to make a change to a resource
that is controlled in base, parameters must be added to allow you pass in
changes to the application class, then to the server class, then to the
template class, finally picking it up again in the base class.


Storyboard Link
===============

https://storyboard.openstack.org/#!/story/2000172


Proposed Change
===============

I propose we flatten server, base, and template to a single class called
server. All the parameters can be flattened as well, and if statements added
to allow someone or something who really wanted 'template' to get just those
resoruces. (nodepool prepare node jumps directly into template, its the only
place I've found so far that base or template are called directly)

Additionally I propose we pull the openstack_project::server invocation
out of the application classes and put it in the node definition. Example:

.. code-block:: puppet

    node 'review.openstack.org' {
      class { 'openstack_project::server':
        iptables_tcp_public_ports => [80, 443, 29418],
        sysadmins                 => hiera('sysadmins', 'default'),
      }
      class {'openstack_project::review':
        # ... other params ..
      }
    }


When every node in the environment takes the same openstack_project::server
class, that means that the differences between two nodes can be thought of
as the array of parameters passed into those servers. It makes servers more
like each other and easier to reason about. It also frees the application
class to do what it does best, which is set up the application. o_p::server
handles all the base configuration and o_p::application handles the application.

Alternatives
------------

We could remain the same. We could flatten server, base, and template without
pulling the openstack_project::server invocation out of the application class.

Implementation
==============

Server class would be expanded to include the parameters and resources
of base class and template class. New parameters would be added so that
there is a way to get just the resources of template and base.

Application classes would be modified to not need server class, site.pp
would be modified.


Assignee(s)
-----------

Primary assignee:
  nibalizer

Gerrit Topic
------------

Use Gerrit topic "downstream-puppet" for all patches related to this spec.

.. code-block:: bash

    git-review -t downstream-puppet

Work Items
----------

* base.pp flatten
* template.pp flatten
* refactor of places where those classes are used
* refactor of server.pp
* application specific refactors + site.pp changes


Repositories
------------

No new git repos

Servers
-------

Puppet change only

DNS Entries
-----------

Puppet change only

Documentation
-------------

Largely should be transparent to anyone who isn't mucking around in
infra internals

Security
--------

No security implications

Testing
-------

apply-test will provide acceptance testing

Dependencies
============

No known dependencies





