::

  Copyright (c) 2017 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

================
Nodepool Drivers
================

Storyboard: https://storyboard.openstack.org/#!/story/2001044

Support multiple provider drivers in Nodepool, including static
hosts.

Problem Description
===================

As part of the Zuul v3 effort, it was envisioned that Nodepool would
be expanded to support supplying static nodes in addition to the
current support for OpenStack instances, as well as potentially nodes
from other sources.  The work to move the data structures and queue
processing to ZooKeeper was expected to facilitate this.  This
specification relates both efforts, envisioning supplying static nodes
as an implementation of the first alternative driver for Nodepool.

Proposed Change
===============

There are many internal classes which will need to be changed to
accomodate the additional level of abstraction necessary to support
multiple drivers.  This specification is intentionally vague as to
exactly which should change, but instead lays out a high-level
overview of what should be shared and where drivers should diverge in
order to help guide implementation.

Nodepool's current internal architecture is well suited to an
extension to support multiple provider drivers.  Because the queue
processing, communication, and data storage all occur via ZooKeeper,
it's possible to create a component which fulfills Nodepool requests
that is completely external to the current Nodepool codebase.  That
may prove useful in the future in the case of more esoteric systems.
However, it would be useful for a wide range of Nodepool users to have
built in support for not only OpenStack, but other cloud systems as
well as static nodes.  The following describes a method of extending
the internal processing structure of Nodepool to share as much code
between multiple drivers as possible (to reduce the maintenance cost
of multiple drivers as well as the operational cost for users).
Operators may choose to run multiple providers in a single process for
ease of deployment, or they can split providers across multiple
processes or hosts as needed for scaling or locality needs.

The nodepool-launcher daemon is internally structured as a number of
threads, each dedicated to a particular task.  The main thread,
implemented by the ``NodePool`` class starts a ``PoolWorker`` for each
provider-pool entry in the config file.  That ``PoolWorker`` is
responsible for accepting and fulfilling requests, though the
specifics of actually fulfilling those requests are handled by other
classes such as ``NodeRequestHandler``.

We should extend the concept of a ``provider`` in Nodepool to also
include a driver.  Every provider should have a driver and also a
``pools`` section, but the rest of the provider configuration (clouds,
images, etc.) should be specific to a given driver.  Nodepool should
start an instance of the ``PoolWorker`` class for every provider-pool
combination in every driver.  However, the OpenStack-specific
behavior currently encoded in the classes utilized by ``PoolWorker``
should be abstracted so that a ``PoolWorker`` can be given a different
driver as an argument and use that driver to supply nodes.

When nodes are returned to nodepool (their locks having been
released), the ``CleanupWorker`` currently deletes those nodes.  It
similarly should be extended to recognize the driver which supplied
the node, and perform an appropriate action on return (in the case of
a static driver, the appropriate action may be to do nothing other
than reset the node state to ``ready``).

The configuration syntax will need some minor changes::

  providers:
    - name: openstack-public-cloud
      driver: openstack
      cloud: some-cloud-name
      diskimages:
        - name: fedora25
      pools:
        - name: main
          max-servers: 10
          labels:
            - name: fedora25-small
              min-ram: 1024
              diskimage: fedora25
    - name: static
      driver: static
      pools:
        - name: main
          nodes:
          - name: static01.example.com
            host-key: <SSH host key>
            labels: static
          - name: static02.example.com
            host-key: <SSH host key>
            labels: static

Alternatives
------------

We could require that any further drivers be implemented as separate
processes, however, due to the careful attention paid to the Zookeeper
and Nodepool protocol interactions when implementing the current
fulfillment algorithm, prudence suggests that we at least provide some
level of shared implementation code to avoid rewriting the otherwise
boilerplate node request algorithm handling.  As long as we're doing
that, it is only a small stretch to also facilitate multiple drivers
within a single Nodepool launcher process so that running Nodepool
does not become unecessarily complicated for an operator who wants to
use a cloud and a handful of static servers.

Implementation
==============

Assignee(s)
-----------

* smyers
* tristanC

Gerrit Branch
-------------

Nodepool and Zuul are both branched for development related to this
spec.  The "master" branches will continue to receive patches related
to maintaining the current versions, and the "feature/zuulv3" branches
will receive patches related to this spec.  The .gitreview files have
been be updated to submit to the correct branches by default.

Work Items
----------

* Abstract Nodepool request handling code to support multiple drivers
* Abstract Nodepool provider management code to support multiple drivers
* Collect current request handling implementation in an OpenStack driver
* Extend Nodepool configuration syntax to support multiple drivers
* Implement a static driver for Nodepool

Repositories
------------

N/A

Servers
-------

N/A

DNS Entries
-----------

N/A

Documentation
-------------

The Nodepool documentation should be reorganized by driver.

Security
--------

There is no access control to restrict under what conditions static
nodes can be requested.  It is unlikely that Nodepool is the right
place for that kind of restriction, so Zuul may need to be updated to
allow such specifications before it is safe to add sensitive static
hosts to Nodepool.  However, for the common case of supplying specific
real hardware in a known test environment, no access control is
required, so the feature is useful without it.

Testing
-------

This should be unit tested in the way typical for Nodepool.

Dependencies
============

This is related to the ongoing `Zuul v3`_ work and builds on the
completed `Zookeeper Workers`_ work in Nodepool.

.. _Zuul v3:
   http://specs.openstack.org/openstack-infra/infra-specs/specs/zuulv3.html
.. _Zookeeper Workers:
   http://specs.openstack.org/openstack-infra/infra-specs/specs/nodepool-zookeeper-workers.html
