::

  Copyright 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Docs Publishing via Swift
=========================

Story: https://storyboard.openstack.org/#!/story/168

We need to update the docs publishing pipeline to add features the doc
team needs as well as provide a process that does not rely on Jenkins
since we intend to retire it.

Problem Description
===================

Juno summit session: https://etherpad.openstack.org/p/summit-b301-ci-doc-automation

The current doc publishing process uses Jenkins to FTP the results of
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

The infrastructure team wants to remove Jenkins, and both the current
FTP and SCP based publishers depend on Jenkins unique security
arrangements to prevent exposure of the credentials used.  Our
intended replacement has no such feature and we expect workers to be
completely untrusted.

The current system is also not atomic or concurrency safe, as multiple
publishing jobs may be running at the same time, and using SCP or FTP
to upload simultaneously.

This process is also very similar to log and artifact publishing, and
keeping alignment with those process is desirable.

Proposed Change
===============

The new system will use a dedicated virtual server running Apache and
using LVM cinder volumes to serve content.  The existing build+publish
Jenkins job will be replaced with two jobs: the build job (which will
only build the documents and upload them to an intermediate storage
location) and the publish job (which will run locally on the docs
server and retrieve the contents and publish them to their final
locations).

As part of the effort to migrate away from Jenkins, we have developed
a script that uploads logs from test runs to an OpenStack Swift
container.  Each job is granted limited permission to upload logs with
a specific prefix to their ids (like a subdirectory) within the
container.  We will use the same script to upload the results of the
build job to Swift.  For jobs that run in the check queue that
currently publish to docs-draft, this will be the end of the process
(Zuul can link directly to the swift URL). [Alternative: use
os-loganalize to proxy from swift at a more friendly url.]

For final publishing, further action is needed.  The general approach
will be to have a dedicated Zuul worker on the publishing server that
will download files from swift and rsync them to the correct location.

The publish job needs to know what files to download from swift (as it
can not rely on index pages).  It should also do this without using
any Swift credentials (mostly for simplicity; there is not much of a
security concern with the publish worker accessing the Swift API if it
needs to).  In order to retrieve the full set of files from swift, the
build job should create a manifest file called ``.manifest.txt`` at
the root of the output directory with a recursive directory listing of
all files to be published.  The publish job can then retrieve that
file (by constructing the URL from the ZUUL_* environment variables)
and subsequently all of the contents that it lists. [Alternative: have
the build job upload a tarball, but that means we can't use exactly
the same process for draft and publishing.]

The publish job also needs to determine the target directory to rsync.
To do this, it will take the contents of the ZUUL_PROJECT environment
variable as a base, and to this it will append the contents of the
``.target.txt`` file that it will expect to find in the contents it
downloaded from swift.  The file should be empty for rsyncing to the
root ("openstack/nova"), or contain, e.g., the string "havana" to
rsync to "nova/havana".  This lets the build jobs specify the final
publishing location but only within the context of the project (eg,
nova can't accidentally overwrite keystone's documentation).

Before the publish job rsyncs the downloaded data into its final
location, it must first create a list of directories that should not
be deleted.  This way if an entire directory is removed from a
document, it will still be removed from the website, but directories
which are themselves roots of other documents (e.g., the havana
branch) are not removed.  A marker file at the root of each such
directory will accomplish this, and in fact, simply leaving either the
.target.txt or .manifest.txt files in place after copying to the
destination will suffice.  The publishing job should find each of
those in the destination hierarchy and add their containing
directories to a list of directories to exclude from rsyncing.  Then
an rsync command of the form::

  rsync -a --delete-after --exclude-from=/path/to/exclude-file /src/ /dest/

should safely update and delete only the relevant data.

The openstack-manuals repo builds multiple manuals, in separate
subdirectories, from a single repository.  Using an overlay method
similar to what is used for the developer documentation, the current
build/publishing job updates only the changed documents and places
them in appropriate target directories.  The approach described above
assumes the build job will output the full contents of a single
"module" each time; having missing directories in that output risks
the publisher removing them in the rsync step.  One of the following
approaches will need to be chosen to deal with this:

1) Reconfigure the docs build job to build all of the manuals for
   publication (the optimization to only build changed manuals could
   be retained for efficiency in gating).  This wastes some CPU time
   on build hosts but perhaps not that much (and should be
   quantified).

2) Alter the above approach to handle multiple rsync source and
   destination path outputs for a single job.  This makes the
   build/publishing process more complex.

3) Allow the build job to provide an initial exclusion list for the
   rsync command (so that it can add directories that it knows are
   under its control but are not being updated in this run).

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

The developer.openstack.org and specs.openstack.org sites are
published using the same mechanisms as docs.openstack.org currently.
Under the new system, we can create apache virtual hosts for these
sites that connect the appropriate URLs with their publishing
locations on disk.

Alternatives
------------

The above implementation has several minor alternative changes noted
within.  In addition to this approach, we also considered the
following:

ReadTheDocs
~~~~~~~~~~~

While readthedocs does handle docs publishing, including being
version-aware, it is specific to python-based sphinx documentation and
would not be useful for openstack-manuals (or other artifacts).  It is
also considered quite complex to set up.

AFS
~~~

The Andrew File System is a global distributed filesystem that would
work quite well in this instance.  Workers could be granted limited
ACLs to publish to specific locations, so we could use the current
combined build+publish job approach, but the worker could rsync
directly to the final publishing location in AFS, and volume
replication could be used to make atomic updates to the entire site.
A static web server would then serve files out of AFS; more web
servers can be added as needed to scale.

This approach requires some investment in creating and maintaining an
AFS cell for OpenStack, as well as some enhancement work to Nodepool
and Zuul to deal with Kerberos credentials.  This is all work that we
would like to do for other reasons (including mirrors), but is more
substantial than what would be needed for the selected approach.
Moreover, it should not be difficult to move from the selected
approach to use AFS later should a cell materialize.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  TBD

Work Items
----------

* Create new publish.openstack.org server (this will be a server name
  that is not publicized, instead we will use apache virtual hosts for
  the public hostnames which will be CNAME DNS entries).
* Create apache vhosts for docs.openstack.org,
  developer.openstack.org, and specs.openstack.org on
  publish.openstack.org
* Create new doc build job that publishes to swift; start running this
  in addition to current publishing jobs on at least one project and
  openstack-manuals
* Enhance the doc publishing jobs to create .target.txt files
* Enhance the swift-upload tool to create .manifest.txt files
* Write and install the Zuul worker that will run on the docs server
* After testing, add the new jobs to all projects
* Copy data from the FTP site
* Change DNS to point to the new server
* Remove old build jobs
* Remove Rackspace cloud sites instances

Repositories
------------

N/A.

Servers
-------

publish.openstack.org will be a new server with LVM managed cinder
volumes.  Perhaps using SSD.

DNS Entries
-----------

publish.openstack.org will need to point to the new server.
docs.openstack.org, developer.openstack.org, and specs.openstack.org
will need their TTLs lowered in advance of the moves.  On moving, they
will become CNAME entries for publish.openstack.org.

Documentation
-------------

Infra documentation will need to be written for the new server and
this process.

Security
--------

The build jobs will have no special access and will only be able to
put content in swift.  The publishing job will run locally on the docs
server, but will run no user-supplied code, and will constrain the
publishing of content to a project-specific area.

Testing
-------

This can operate in parallel with the current system without
disruption.

Dependencies
============

We should finalize the log publishing system first (this is nearly
done at the time of writing).
