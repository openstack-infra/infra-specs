::

  Copyright 2018 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode


============================
StoryBoard Story Attachments
============================

Include the URL of your StoryBoard story:

https://storyboard.openstack.org/#!/story/2000679

LaunchPad supports bug attachments with launchpadlibrarian so that users can attach
logs, screenshots, etc to bug reports. There are several prospective StoryBoard users
blocked on migration until StoryBoard supports story attachments.

Problem Description
===================

When doing bug triage, lots of projects often ask for log files. These log files
can also become quite huge and complex enough that the average user may not know
which parts are relvant or not to be able to trim it down. Moreover, it can also
be a set of log files depending on the issue.

Another common use for attachments is for files (like patches) that contain information
about embargoed issues.

Bug reports can live for months and the files need to be accessible. This means that
linking to and relying on 3rd party providers (dropbox, paste.o.o) can potentially
mean the loss of debug information which can make a bug report incompletee.

Current users of LaunchPad rely on launchpadlibrarian for this feature.

Proposed Change
===============

We have donors willing to provide a variety of storage to our infrastucture team.
Luckily, the amount we require isn't large enough to require much door knocking.
The plan would be to implement the service in a pluggable manner so that we can use
whatever we can get- Ideally this will be some sort of object store, like Swift, but
ceph stores are also viable so long as they have swift compatible APIs. A pluggable
approach would be good for users outside of OpenStack as well because it will avoid
vendor lock in. This shouldn't require too much effort as there are already libraries that
exist that can provide the compatibility layer we need.

As for the actual implementation, we would want storyboard to privately maintain its
index of (persistent public) object urls, and would not want the object store to serve
an index of those. This way is easier on the StoryBoard API server as most of the
bandwith and processing is done by the object store. StoryBoard could obtain an
authroized upload URL and either proxy the upload through or orchestrate the
upload directly to Swift. Then it would supply the Swift URL for the attachment
object in the browser so that the user can retrieve it. The URLs that exist in the
index being served should also be created in a way that they can't be guessed
or generated- essentially something pseudorandom.

Files would be uploaded to the storyboard site and passed onto the database to be
saved in a new table. and, at the time of upload, the object url would be created and
saved in the database.

Alternatives
------------

Alternatives to having users interact directly with the object storage service.

# Undiscoverable URIs

The idea here is to have StoryBoard obtain a short-lived public URI from the storage
backend when a request for a Story containing attachments is made. This URI would
then be discoverable (via StoryBoard's API returning it) to the requester, but not
advertised to others. Other requests should obtain a different short-lived URI.

# Attachment API

This would entail an API endpoint for handling attachments, which would tie in with
private Story ACLs where necessary, and provide a permanent URI for each
attachment by opaquely handling storage and retrieval of objects from the configured
backend itself.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <None>


Gerrit Topic
------------

Use Gerrit topic "story_attachments" for all patches related to this spec.

.. code-block:: bash

    git-review -t story_attachments

Work Items
----------

1. Determine with infrastructure team if what is available will suit the design
2. Write attachment API and add database tables for attachments.
3. Make changes to storyboard-webclient for users to upload files through.
4. Write new migration script to handle the migration of bug attachments in LaunchPad.
5. Update API documentation and webclient manual to include new functionality.


Repositories
------------

None

Servers
-------

Only the existing production storyboard.openstack.org and storboard-dev.openstack.org servers
should be required for this implementation.

Note: While not a server, there is a infra cost for a swift container (or similar). 

DNS Entries
-----------

Will any other DNS entries need to be created or updated?

Documentation
-------------

The API docs and webclient docs will need to add new information about story attachements.

Security
--------

The files themselves will need to be non-discoverable, particularly where the stories are
marked as private. We also should be aware that there is potential for abuse. Storyboard
deployments already need someone to manage abuse risk so doing something like periodic
scans against the data.

As a follow up, we could configure the creation of the ungessable URL to have a timeout.
Shorter if the story with the associated attachment is private. This could all be provided by
TempURLs in Swift.

Testing
-------

Functional tests would definitely be a good thing to have; we can do functional testing
with devstack.

Dependencies
============

None.
