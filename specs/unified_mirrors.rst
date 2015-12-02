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
not restricted to) pypi-wheel, npm, bower, rubygems, and maven's nexus.

Proposed Change
===============

The existing pypi.<region>.openstack.org mirrors will be renamed to
mirror.<region>.openstack.org, and the current existing pypi mirror will be
moved into a subdirectory of the http root. Furthermore, each mirror will
also be named based on the hosting cloud provider. The eventual goal is URI
paths similar to the following:

* http://mirror.<region>.<provider>.openstack.org/pypi
* http://mirror.<region>.<provider>.openstack.org/wheel
* http://mirror.<region>.<provider>.openstack.org/npm
* http://mirror.<region>.<provider>.openstack.org/gem

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

* Michael Krotscheck (Mirror Rename, NPM Mirror)
* Greg Haynes (Wheel Mirror)
* Emilien Macchi (Ruby Mirror)

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
2.  Existing bandersnatch storage directories should be rsync'ed to the new
    host. This should be done while the bandersnatch services have been
    temporarily disabled.
3.  The bandersnatch cache directory on the new mirrors should be moved from
    /srv/static/mirror to /srv/static/pypi.
4.  The existing pypi_mirror.pp manifest should defer its vhost creation,
    and data directory location, to its including manifest.
5.  A new mirror.pp manifest should be created that provides the new data
    directory to pypi_mirror.pp, hosted at /pypi. This should be used by an
    entry in site.pp used by mirror.<region>.<cloud>.openstack.org.
6.  Once our new mirrors have had a successful puppet run, they should be
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
   global upper-constraint requirements.
3. A wheel distribution job should be created to rsync those wheels to our
   wheel mirror directories.
4. Our nodepool slaves should be instructed to use the new wheel mirror in
   addition to our pypi mirror.

The following work will need to be completed in order to create an npm mirror.
Goal: http://mirror.<region>.<cloud>.openstack.org/npm

1. A new npm_mirror.pp manifest should be added to mirror.pp, to provide an
   npm mirror replication service (using registry-static) hosted at /npm.
2. NodeJS and NPM will be added to our nodepool slaves. This is to simplify the
   next step.
3. The nodepool slaves should be instructed to use the new npm mirror, where
   necessary.

The following work will need to be completed in order to create a gem mirror.
Goal: http://mirror.<region>.<cloud>.openstack.org/gem

1. A new rubygems_mirror.pp manifest should be added to mirror.pp, to provide
   a rubygems replication service (using rubygems-mirror) hosted at /gem.
2. The nodepool slaves should be instructed to use the new gem mirror, where
   necessary.

Repositories
------------

No new repositories are required.

Servers
-------

* New hosts will be provisioned for each region, named
  mirror.<region>.<cloud>.openstack.org. These should all run trusty.
* 500GB of disk space will need to be provided for each mirror type
  (Estimates for existing mirrors range from 200GB to 300GB).

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

This adds a dependency to the registry-static project, an npm static mirroring
script. It also adds a dependency to the rubygems-mirror project, a static gem
mirroring service.
