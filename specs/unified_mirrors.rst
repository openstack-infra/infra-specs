::

  Copyright 2015 Hewlett Packard Enterprise Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

===============
Unified Mirrors
===============

This spec captures the work necessary to provide multiple mirror types to
the openstack build infrastructure, without requiring the creation of a new node
for each mirror.

Problem Description
===================

The package mirrors available in our infra regions are python specific, both
due to their names (pypi.<region>.openstack.org), and because the mirror is
hosted at the root directory of the webserver. This prevents us from reusing
the host for different package mirrors.

At the same time, there is new demand for different mirrors, including (but
not restricted to) pypi-wheel, EPEL, UCA...

Proposed Change
===============

The existing pypi.<region>.openstack.org mirrors will be renamed to
mirror.<region>.openstack.org, and the current existing pypi mirror will be
moved into a subdirectory of the http root. Furthermore, each mirror will
also be named based on the hosting cloud provider. The eventual goal is URI
paths similar to the following:

* http://mirror.<region>.<provider>.openstack.org/pypi
* http://mirror.<region>.<provider>.openstack.org/wheel
* http://mirror.<region>.<provider>.openstack.org/centos
* http://mirror.<region>.<provider>.openstack.org/ubuntu

In order to reduce the complexity and storage requirements of the
mirror hosts, the mirror content will be written into AFS.  When
mirror updates are performed, the AFS volume hosting the mirror will
be "released" (that is, replicated into fault-toleranet read-only
volumes).  Each of the mirror hosts will serve the same content from
those read-only volumes, but will be configured with a substantial
local cache so that ultimately most content will be served directly
from local disk.

Alternative 1: Per-language mirror hosts
----------------------------------------

Rather than using one host for all mirrors, we could create one host per
mirror. This is possible, however it increases the maintenance overhead,
consumes unnecessary resources, and introduces additional points of failure.

Alternative 2: In-flight updates of mirrors
-------------------------------------------

Rather than provisioning new hosts, we could live-modify the existing hosts.
Building new hosts is more labor intensive, however it results in zero
downtime, and also permits us to upgrade the operating system for those
instances still running Ubuntu precise.

Alternative 3: The status quo
-----------------------------

Rather than creating more mirrors, we can simply maintain the status quo. This
discards potential speed improvements from having our own wheel mirrors, and
makes our builds vulnerable to downtimes in upstream repositories.

Implementation
==============

Assignee(s)
-----------

* Michael Krotscheck (Mirror Rename)
* Greg Haynes (Wheel Mirror)

Support from infra root will be required to provision servers, rsync
packages, and update DNS records.

Gerrit Topic
------------

Use the 'unified_mirror' gerrit topics for all patches related to this spec:

.. code-block:: bash

    git-review -t unified_mirror

Work Items
----------

The following work items will need to be completed in order to facilitate the
rename. Goal: http://mirror.<region>.<cloud>.openstack.org/pypi

1.  New hosts will be provisioned for each region, and DNS records created,
    named mirror.<region>.<cloud>.openstack.org. These should all run ubuntu
    trusty.
2.  A new host, mirror-update.openstack.org will be provisioned to run
    the bandersnatch process.  It will write into AFS, and upon the
    completion of each successful run, it will release the volume to
    read-only replicas.
3.  Each mirror server will be configured to serve files out of AFS
    via the read-only replica path.
4.  The existing pypi_mirror.pp manifest should defer its vhost creation,
    and data directory location, to its including manifest.
5.  A new mirror.pp manifest should be created that provides the new data
    directory to pypi_mirror.pp, hosted at /pypi. This should be used by an
    entry in site.pp used by mirror.<region>.<cloud>.openstack.org.
6.  Once our new mirror have had a successful puppet run, they should be
    manually tested to ensure they function correctly.
7.  Nodepool slaves should be instructed to use the new mirror urls.
8.  Builds should be manually checked to ensure that they are using the new
    mirrors.
9.  The old pypi.<region>.openstack.org hosts should be terminated, as well
    as their DNS entries.
10. All old mirror references and manifests, should be deleted from
    system-config.

The following work items will need to be completed in order to create a wheel
mirror. Goal: http://mirror.<region>.<cloud>.openstack.org/wheel

1. A new wheel_mirror.pp manifest should be added to mirror.pp to provide room
   for our built wheels, hosted at /wheel.
2. A wheel build job should be created, to build our wheels from the
   global upper-constraint requirements and write the output into AFS.
3. A volume release job should be created to release the data to
   read-write AFS volumes if the wheel update is successful.
4. Our nodepool slaves should be instructed to use the new wheel mirror in
   addition to our pypi mirror.

Repositories
------------

No new repositories are required.

Servers
-------

* New hosts will be provisioned for each region, named
  mirror.<region>.<cloud>.openstack.org. These should all run trusty.
* 100-200GB of disk space will need to be provided for an AFS cache.
  The AFS cache size will be set at 50GB.  For mirrors where Cinder is
  available, a 100GB volume should be provisioned to start with.
  Where Cinder is not available, a flavor with 200GB of local storage
  should be used.

DNS Entries
-----------

New DNS entries will be required for mirror.<region>.<cloud>.openstack.org.
Old DNS entries for pypi.<region>.openstack.org will need to be removed.

Documentation
-------------

Existing documentation in the infra manual should be updated to indicate new
mirror locations.

Security
--------

No security concerns anticipated other than those already addressed.

Testing
-------

Manual testing of the new mirrors should be performed before they are used.

Dependencies
============

There are no dependencies.
