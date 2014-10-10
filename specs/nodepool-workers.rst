::

  Copyright (c) 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Nodepool image build and upload workers
=======================================

Story: https://storyboard.openstack.org/#!/story/317

Split the image build and upload features of nodepool into separate
workers for scalability and flexibility.

Problem Description
===================

In moving to using diskimage-builder (DIB) with nodepool we have
encountered some problems with building images on different platforms.
DIB should be able to cross-build for most GNU/Linux systems, but we
have seen some failures related to this.

Further, updates to the DIB mechanism in nodepool require a full
nodepool server restart, which can be somewhat disruptive.

Performing the image building on the same server as nodepool is not
currently a resource issue, however, parallel builds are constrained
by resources and distributing builds to multiple machines could
improve scalability.

Finally, there are some platforms we will never be able to cross-build
images on, such as Microsoft Windows.  This is not an immediate
concern for the OpenStack infrastructure program, but would be a
useful feature for downstream consumers of nodepool.

Proposed Change
===============

Create a new worker (independent process which may run on either the
main nodepool host or one or more new servers) which perform image
build related tasks called "nodepool-builder".

The nodepool-builder worker would read a configuration file to
determine which kinds of images it can produce (determined by the
operator according to criteria such as platform or operating system,
access to needed external resources, and scaleout plan).  It will then
register a function with gearman for each kind of image it can build,
with the job name of the form "image-build:<imagename>", e.g.,
"image-build:centos7".  A unique identifier for the image will be
provided as a parameter to the job.  It will then use DIB to build the
image and store it in a local repository of image files.

Additional image builders may be created to operate in another manner,
such as using the existing nodepool on-instance build methodology, or
other image building methods yet to be explored.  This way the image
building mechanism may be substituted more easily in the future.

The nodepool-builder worker will also register functions with gearman
with the form "image-upload:<imageid>" where the image id is the image
identifier provided in the image-build function.  The worker will
register such functions each time it completes an image build (it will
also register such functions on startup based on the images it has
available locally).  This is so that the build and upload steps may be
separated into separate functions, yet can be routed to the correct
worker (which has the image file on a local filesystem).  This
function will accept as an argument the name of a provider to which
the image should be uploaded.  It will perform the necessary format
conversions on the image file prior to upload.  It will return the
cloud image id with the WORK_COMPLETE packet.

The builder will also register functions with the form
"image-delete:<imageid>" (registered along with the image-upload
function as above) which will clean up local data related to the
image.  When an image is deleted, the corresponding image-delete and
image-upload functions will be de-registered from gearman.

The implementation of the worker will be performed in such a way that
it will be simple to create a subclass with a different method of
performing the image creation.  For example, there may be a base class
that performs all of the actions described except for the actual image
creation, and the actual worker may be a subclass of that which uses
DIB as the build implementation.

Alternatives
------------

The build and upload processes could be separate workers.  However,
the two processes are somewhat tightly coupled since they need to run
on the same host and generally in immediate succession.  Therefore the
current method where they share a process but use modular code to
enable different builders to share as much as possible is chosen.

Implementation
==============

Assignee(s)
-----------

Primary assignee: unknown

Work Items
----------

* Create base class for nodepool-builder that implements above
* Create nodepool-builder that implements dib-based image builds
* Alter nodepool to use gearman functions when using dib images only
  (continuing to use on-instance builds in the main server for other
  image types during the migration phase)
* Optional: create nodepool-builder that implements nodepool on-image
  builds
* Remove non-dib image builds from the main nodepool server
* Rename bits of the nodepool server and config that refer to dib
  specifics (since that is now a build worker implementation detail)

Repositories
------------

This affects the existing nodepool, system-config, and potentially
project-config repos.


Servers
-------

Affects the existing nodepool server and may necessitate the creation
of new servers for specific nodepool image builders (optional).

DNS Entries
-----------

If any image build servers are created, they will need DNS entries.

Documentation
-------------

The infra/system-config nodepool documentation should be updated to
describe the new system.

Security
--------

The gearman protocol is cleartext and unauthenticated.  IP based
access control is currently used, and certificate support along with
authentication is planned and work is in progress.  No sensitive
information will be sent over the wire (workers will read cloud
provider credentials from a local file).

Testing
-------

This should be testable privately and locally for most image types.
Any image types currently supplied by nodepool using dib will need to
switch over to the new system immediately.  Others may migrate on an
image-by-image basis.


Dependencies
============

N/A.
