::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Add elections.openstack.org
===========================

https://storyboard.openstack.org/#!/story/2000499

Last election cycle (Sept/Oct 2015) a new gerrit based workflow was used to
propose and validate candidates for the PTL and TC elections.  The publication
of validated candidates was somewhat manual in that a list of confirmed
candidates needed to be pasted in to a wiki page.  This list is trivially
generated with sphinx.  This spec covers the creation of
elections.openstack.org for hosting build artifacts openstack/election repo.
This will be closely modeled on the existing governance.openstack.org site and
workflow.

Problem Description
===================

See above.

Proposed Change
===============

The proposed change is to add a vhost, elections.openstack.org to
static.openstack.org and publish the results of a job run on the
openstack/election repo.  The scope for this spec is creating the host and
arranging for publishing the data.  Any additional work would be an additional
spec.  We're asking for minimal help with the changes form the infra team and
expect the infra team's primary role would be review and approval.

Alternatives
------------

#. As always we can do nothing and go with the status quo
#. We could integrate the election data into the existing governance vhost.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Tony Breeds (o-tony)

Additional assignee:
  Tristan Cacqueray (tristan-cacqueray)

Tony will primarily work on the {system,project}-config changes required and
Tristan will work on the job to build the required HTML.

Gerrit Topic
------------

We will use the add_elections as the gerrit topic

Work Items
----------

#. Define a vhost on static.openstack.org
#. Create a tox env and Jenkins job to generate candidate lists using the usual
   "python setup.py build_sphinx" entry point
#. Create a Jenkins job to publish the generated HTML

Repositories
------------

No.

Servers
-------

No new servers.  I propose that static.openstack.org be modified to handle
this role.

DNS Entries
-----------

Yes: election.openstack.org should be a CNAME for static.openstack.org

Documentation
-------------

I believe that this spec and changes to system-config and project-config repos
will be adequate documentation.

Security
--------

I propose to use governance.openstack.org as a template so I do not expect any
new security concerns

Testing
-------

I don't believe that this spec introduces any infra specific testing.  Of
course Tristan and I will endeavour to thoroughly test changes in the
openstack/election repo.


Dependencies
============

None outside of this spec.
