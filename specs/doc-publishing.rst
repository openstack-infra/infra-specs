::

  Copyright 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Docs Publishing via AFS
=======================

Story: https://storyboard.openstack.org/#!/story/168

We need to update the docs publishing pipeline to add features the doc
team needs as well as provide a process that does not rely on Jenkins
security and access principles since we have retired it.

Problem Description
===================

Juno summit session: https://etherpad.openstack.org/p/summit-b301-ci-doc-automation

The current doc publishing process uses Zuul to FTP the results of
doc builds to a Rackspace cloud site.  Multiple versions of docs (eg,
for releases or branches) are handled by publishing to a subdirectory
of the destination.  The publishing job for the nova developer
documentation is configured to publish everything that appears in
`doc/build/html` to `developer/nova`, so that content generated from
the master branch appears at
`http://docs.openstack.org/developer/nova/`.  However, the doc build
script detects if it is being run on a change that has merged to a
stable branch and moves the output into a subdirectory before
uploading.  Therefore, havana documentation ends up in
`doc/build/html/havana`, and the entire `doc/build/html` tree is still
uploaded to `developer/nova` but since no content is present above the
`havana` subdirectory, the documentation from the master branch is not
touched, while the new havana docs appear at
`http://docs.openstack.org/developer/nova/havana/`.  Similar
approaches may be used for other kinds of documentation builds.

Put another way, the docs jobs do not have a holistic view of the site
to which they are publishing, but instead operate only within the
context of one subdirectory and are expected not to interfere with
jobs publishing to other locations in the same site.

The major drawback for the docs team is that this mechanism does not
support automatically deleting a file when it has been removed from
the documentation.  Even if it is not linked internally, it may still
remain in search engine indexes and users may find it.

The current system is also not atomic or concurrency safe, as multiple
publishing jobs may be running at the same time, and using SCP or FTP
to upload simultaneously.  Nor is it easy to serve the content via
HTTPS.

Proposed Change
===============

By hosting docs using the `Andrew Filesystem
<http://docs.openstack.org/infra/system-config/afs.html>`_ (AFS), we
can bring the building and hosting infrastructure closer together.  We
will host the entire documentation site in AFS and perform
documentation builds on hosts which are AFS clients.  The build jobs
can then rsync their results into the correct location.

This approach may be used for any documentation publishing site.

AFS Access
----------

In order to sync the results into AFS, a part of the system will need
authenticated AFS access.

To accomplish this in our current (Zuul v2.5) configuration, we will
grant the Zuul launchers access to AFS and instruct them to copy the
results of doc build jobs into AFS.

We will create a Kerberos principal for the Zuul launchers, grant it
write access to the documentation tree in AFS, and then run the
launchers with credentials using k5start.  We will then add a new
publisher to zuul-launcher which will instruct it to rsync data from
the worker to the launcher host and from there into AFS.

The current content of docs sites will be copied into a separate
location in AFS for medium-term storage and access, but will not
become part of the new sites.  Instead, all content in the new docs
sites will be generated from jobs.  If we later find we need to copy
any content from it, we can do so, and eventually delete it once we
are satisfied.

This should allow us to begin using AFS for docs with a minimum of
changes.

As this adds a new feature to the set of actions supported by the
subset of the Jenkins Job Builder language understood by Zuul, this
requires a transition plan for Zuul v3.  The simplest way to handle
that would be to add the keytab used by the kerberos principal to the
Zuul v3 credential store, and allow that credential to be used by doc
build jobs.  That would allow the worker to obtain AFS credentials and
then rsync the job results into place in the same way as proposed for
Zuul v2.

Another option involves adding AFS/Kerberos support to Zuul directly
so that jobs can specify that they require certain AFS/Kerberos
credentials and have them supplied.  While this option may be more
desirable in the long run, there are still some open questions about
it so it needs some more design specification.

With at least one solid transition plan for Zuul v3, we should be able
to proceed with the enhancement to Zuul v2.5.


RSync Process
-------------

Before the job rsyncs the build into its final location, it must first
create a list of directories that should not be deleted.  This way if
an entire directory is removed from a document, it will still be
removed from the website, but directories which are themselves roots
of other documents (e.g., the havana branch) are not removed.  A
marker file at the root of each such directory will accomplish this;
therefore each build job should also ensure that it leaves such a
marker file at the root of its build.  The job should find each of
those in the destination hierarchy and add their containing
directories to a list of directories to exclude from rsyncing.  Then
an rsync command of the form::

  rsync -a --delete-after --exclude-from=/path/to/exclude-file /src/ /dest/

should safely update and delete only the relevant data.

The openstack-manuals jobs can operate in the same manner as developer
documentation jobs.

Apache Service
--------------

After the rsync is complete, the documents will be in a location that
does not necessarily map to the desired URL.  The apache process on
the docs server can be configured to rewrite URLs as necessary.  For
instance::

 docs.openstack.org/developer/nova/ -> /srv/docs/openstack/nova/
 docs.openstack.org/icehouse/install-guide -> /srv/docs/openstack/openstack-manuals/icehouse/install-guide

Could be achieved with rewrite rules like::

 /developer/(.*) -> /srv/docs/openstack/$1
 anything else: -> /srv/docs/openstack/openstack-manuals/

Finally, in the rare cases of major restructuring, or the need to
delete an entire "module" from the site, a member of infra-root can
log in and manually remove anything needed.

The developer.openstack.org, specs.openstack.org, and
governance.openstack.org sites are published using the same mechanisms
as docs.openstack.org currently.  Under the new system, we can create
apache virtual hosts for these sites that connect the appropriate URLs
with their publishing locations on disk.

Apache should honor redirects in .htaccess files so that the
documentation team can add redirects as they reorganize documentation
over time.

Alternatives
------------

Zuul Options
~~~~~~~~~~~~

There are several alternative ways of handling AFS access for Zuul:

1) Update Zuul to create and dispatch AFS credentials for jobs.  We
would configure it so that at the start of each doc publishing job it
would do the following:

  * Ensure the directory /afs/openstack.org/docs/<projectname> exists
  * Ensure that there is a PTS group with the name docs-<projectname>
  * Create a new Kerberos principal
  * Create a new PTS user for that principal
  * Add the PTS user to the PTS group
  * Provide a keytab for the principal to the job (must not appear in
    jenkins parameters)
  * Run the job
  * Delete the PTS user
  * Delete the Kerberos principal

This has the advantage of ensuring that each job has access to only
the area in AFS designated for it, and it also means that the keytab
used for authentication to AFS can not be reused later if exposed.

It has the disadvantage of needing a moderate amount of work in Zuul
(much of which would be easier in Zuulv3).  We also do not have
experience interacting with Kerberos and the AFS PTS database in an
automated fashion.  Finally, it may place a moderate load on those
services due to frequently creating and deleting entries.

2) Run the build jobs on special long-running workers that have a
keytab that grants write access to all of /afs/openstack.org/docs.

This means that any doc build job could conceivably write to any docs
location in AFS.  However it would not be easy to do so accidentally
-- such an event would need to be an explicitly malicious action
performed by someone authorized to merge code into an OpenStack
repository (or via some external dependency used during documentation
builds).  This seems unlikely, and if it does happen, we would be
likely to trace the problem to the source and easily correct the
situation by re-building existing documentation.  It is also possible
for a malicious job to expose the content of the keytab so that
someone could download it and have write access to AFS directly.

We have taken a similar approach with the wheel builders -- they run
on long-running workers which contain the credentials needed to write
to AFS.  We determined the risk was sufficiently low to permit that.

This could be implemented almost immediately (with no changes to Zuul
necessary).

When we stop using Jenkins in Zuul v3, we can either pass AFS
credentials to test node, or give the Zuul launcher access to AFS and
instruct it to sync the data with ansible.

3) Run the build jobs on special workers that have IP based access to
all of /afs/openstack.org/docs.

This is very similar to option 2 with the exception that there would
be no local keytab file used for AFS access, and so therefore it could
not be exposed.  Instead, the IP address of the long-running node
would be added to the AFS PTS database and given access.  Each time we
added a documentation builder, we would need to add its IP address as
well.

ReadTheDocs
~~~~~~~~~~~

While readthedocs does handle docs publishing, including being
version-aware, it is specific to python-based sphinx documentation and
would not be useful for openstack-manuals (or other artifacts).  It is
also considered quite complex to set up.

Swift
~~~~~

An earlier version of the proposal used Swift as a temporary storage
location for documentation builds, but due to the difficulty of
rsyncing directly into Swift, was deemed more complex to implement
than this.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jeblair

Additional volunteers:
  mordred
  pabelanger

Gerrit Topic
------------

Use Gerrit topic "afs-docs" for all patches related to this spec.

.. code-block:: bash

    git-review -t afs-docs

Work Items
----------

* Alter docs jobs to leave marker files at the root of each build.
* Create AFS volumes for old docs sites that we will copy from the FTP
  server.
* Create AFS volumes for new docs sites.
* Create PTS group for docs and grant access to AFS volumes.
* Create Kerberos principal for Zuul launchers and add to PTS group.
* Perform sync of FTP site to AFS archival location.
* Run zuul-launchers under k5start.
* Create manifest for files.openstack.org to serve data from AFS via
  apache.
* Create Apache virtual hosts on files.o.o for docs.o.o,
  developer.o.o, and specs.o.o.
* Add AFS publisher support to zuul-launcher.
* Add AFS publisher to all docs jobs.
* Run both publishers in parallel until sufficient content has been
  generated and placed in AFS.
* Update DNS to point to new sites.
* Re-perform sync of FTP site to AFS archival location.

Repositories
------------

N/A.

Servers
-------

files.openstack.org will be a new server which will be an AFS client
and Apache server.

DNS Entries
-----------

files.openstack.org will need to point to the new server.
docs.openstack.org, developer.openstack.org, and specs.openstack.org
will need their TTLs lowered in advance of the moves.  On moving, they
will become CNAME entries for files.openstack.org.

Documentation
-------------

Infra documentation will need to be written for the new server and
this process.

Security
--------

Zuul launchers will have full access to the docs area in AFS (similar
to current situation with full access to FTP site).

Testing
-------

This can operate in parallel with the current system without
disruption.

Dependencies
============

None.
