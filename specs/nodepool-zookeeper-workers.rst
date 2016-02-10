::

  Copyright (c) 2016 IBM Corp.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Nodepool: Use Zookeeper for Workers
===================================

Story: TBD.

Replace the use of Gearman in coordinating nodepool workers with
Zookeeper.

Problem Description
===================

In the `Nodepool Build Worker`_ spec, we created a separate worker
process to handle image building and uploading.  In order to schedule
and coordinate that activity, we used Gearman to trigger builds and
uploads, while keeping the authoritative information on image builds
in the database accessed only by the central Nodepool daemon.

This has worked well, but we have encountered some edge cases, and our
method of dealing with them is awkward and race-prone partly due to
limitations in the Gearman protocol, but also largely because there is
no shared state between the central Nodepool daemon and its workers.

In particular, if an error occurs or the worker is terminated while an
image build or upload is in progress, it is difficult for the daemon
to instruct the worker on what should be cleaned up -- the
delete-image function is only registered if the image actually exists
on disk, which is not something that the daemon can know without
asking.  It is similarly difficult fo the worker to know whether an
image should be deleted without having access to the authoritative
state -- it can see an image on disk but can not know whether or not
it is known to the daemon.

This could be solved by giving the worker access to the database used
by the daemon; then all of the state information would be known by
both parties and Gearman would be used merely to trigger events.
However, this proposal recommends utilizing a single technology -- the
distributed coordination service Zookeeper, to replace the functions
served by both the database and Gearman.

.. _Nodepool Build Worker: http://specs.openstack.org/openstack-infra/infra-specs/specs/nodepool-workers.html

Proposed Change
===============

All of the information in the `dib_image` and `snapshot_image` tables
will be stored in ZooKeeper instead of MySQL.

Zookeeper's data model is based on nodes in a hierarchy similar to a
filesystem.  If Nodepool is configured to build ubuntu-trusty DIB
images there would be a node such as::

  /nodepool/image/ubuntu-trusty

With a subnode to contain all of the ubuntu-trusty builds at::

  /nodepool/image/ubuntu-trusty/builds

And each build would appear as a child with a sequential ID and some
associated metadata::

  /nodepool/image/ubuntu-trusty/builds/123
    builder: build-hostname
    filename: ubuntu-trusty-123
    state: ready
    state_time: 1455137665
  /nodepool/image/ubuntu-trusty/builds/124
    builder: build-hostname
    filename: ubuntu-trusty-124
    state: ready
    state_time: 1455141272

Similarly, uploaded images may appear as::

  /nodepool/image/ubuntu-trusty/builds/123/provider/infra-cloud-west/images/456
    external_id: 406a9bd0-18b3-458a-8203-60642e759dc1
    state: ready
    state_time: 1455137665
  /nodepool/image/ubuntu-trusty/builds/124/provider/infra-cloud-west/images/457
    external_id: 9ed6ead0-98c6-4e9d-b912-f3bb369f916b
    state: ready
    state_time: 1455144872

This means that the worker and the daemon always share the same
information about both image builds and uploads at all times.  A
request to delete an image or build would take the form of setting the
state to `delete`.  This could be acted on by the worker in a periodic
job, or perhaps more quickly if the worker set a watch on each image
node for which it is responsible.

To support both scaling and building on multiple platforms, we expect
more than one image builder to be running.  To ensure that only one
worker builds a given image type, we can use a `lock construct`_ based
on Zookeeper primitives.  When a builder decides to build an image, it
would obtain the lock on::

  /nodepool/image/ubuntu-trusty/builds/lock

Once it obtained the lock, it would create::

  /nodepool/image/ubuntu-trusty/builds/125

To record the state of the ongoing build.  If the build is aborted,
due to the use of an ephemeral node as the lock, it will automatically
be released and another worker may decide to begin a new build.  When
the worker that failed restarts, it will see the incomplete build in
Zookeeper and delete it from disk (if needed) and then delete the
Zookeeper node.

When launching a node, the Nodepool daemon will simply list the
children of::

  /nodepool/image/ubuntu-trusty/builds/123/provider/infra-cloud-west/images/

And find the highest numbered ready image listed there.

Note that because the state is entirely shared, the logic to determine
when a build is necessary can be moved entirely within the builder.
This kind of logic distribution may ultimately free us from needing a
central daemon at all.

The builder will not only ensure that missing images are built, but
also will build updated replacement images according to a schedule in
the config file (as nodepool does now).  In order to avoid thundering
herd issues and determinism based on clock precision, the replacement
build schedule will be randomized slightly.

We will still want a way to request that an image be built
off-schedule.  For that, we could simply create a node as a flag,
e.g.::

  /nodepool/image/ubuntu-trusty/request-build

.. _lock construct:
   http://zookeeper.apache.org/doc/trunk/recipes.html#sc_recipes_Locks

Future Work
-----------

If this proves satisfactory, we could implement a similar mechanism
for node launching.  The `Nodepool Launch Worker`_ spec, describes
moving the node launch and delete actions into separate workers (one
per provider to maintain API rate limits).  It could be altered to use
the system described here to keep track of nodes.

Further, the `Zuul v3`_ spec describes a rather complex protocol for
Zuul and Nodepool to coordinate the use of nodes.  This is likely to
be subject to the same edge cases and race conditions as the Nodepool
image build workers have been.  Replacing that with a queue construct
built on Zookeeper will significantly simplify the protocol as well as
make it more robust.  At this point there would no longer be any need
for a central Nodepool daemon and Nodepool would be a fully
distributed and highly fault-tolerant application.

Later, after the rest of the Zuul v3 work is complete, we might
consider storing Zuul pipeline state in Zookeeper and developing
distributed Zuul queue workers to process it, thereby achieving the
same fully-distributed application design in Zuul.

.. _Nodepool Launch Worker:
   http://specs.openstack.org/openstack-infra/infra-specs/specs/nodepool-launch-workers.html

.. _Zuul v3:
   http://specs.openstack.org/openstack-infra/infra-specs/specs/zuulv3.html

Alternatives
------------

The status quo is sufficient, if complex.

The workers could be given access to the MySQL database to achieve
shared state, but still use gearman as the trigger.

The above, but a MySQL query could be used as the trigger instead of
gearman.

Implementation
==============

Assignee(s)
-----------

Primary assignee: TDB

Gerrit Topic
------------

Use Gerrit topic "nodepool-zk" for all patches related to this spec.

.. code-block:: bash

    git-review -t nodepool-zk

Work Items
----------

TBD

Repositories
------------

This affects the existing nodepool, system-config, potentially
project-config repos.


Servers
-------

Affects the existing nodepool server.  We will eventually want to run
multiple Zookeeper services.

DNS Entries
-----------

N/A

Documentation
-------------

The infra/system-config nodepool documentation should be updated to
describe the new system.

Security
--------

Zookeeper supports authentication, authorization, and SSL.

Testing
-------

This should be testable privately and locally.

Dependencies
============

N/A.
