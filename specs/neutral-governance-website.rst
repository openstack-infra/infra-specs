::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Neutral governance website
==========================

https://storyboard.openstack.org/#!/story/2000738

The governance.openstack.org website was initially created to publish
Technical Committee governance documents. However since then it is also
used to publish User committee documents (under /uc) and election
details (under /election). Those are not very discoverable and the layout
makes the UC look like a second-class citizen. This change proposes to put
the Technical Committee documents under /tc, to mimic what is done with
the other sections. The index page for governance.openstack.org would become
a neutral page generally explaining governance and pointing to the various
subsites. Proper redirects would be put in place to avoid breaking existing
links.

Problem Description
===================

See above.

Proposed Change
===============

Like for all things OpenStack, the proposed solution is to create a new
repository (openstack/governance-website) which would only be used to hold
the neutral top page. openstack/governance publication jobs would be altered
to publish under /tc, and redirects would be put in place to avoid breaking
links.

Alternatives
------------

We could put the neutral index page directly in the openstack/governance
repository, and move all the Technical Committee content under a tc/
subdirectory within it.

The benefits would be that we'd avoid creating a repository for a single
index page. The drawbacks are:

#. this would introduce a disrupting change to the governance repository
   directory structure, which we would have to propagate to documentation
#. the overall Sphinx title ("OpenStack technical Committee") would appear
   on the "neutral" index page, making it look not that neutral
#. it would make one repository more special than the others


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Thierry Carrez (ttx)

Gerrit Topic
------------

We will use the neutral_governance as the gerrit topic.

Work Items
----------

#. Create a openstack/governance-website repository
#. Push initial structure and proposed neutral page to the new repository
#. Switch publishing of openstack/governance to /srv/static/tc
   (in jenkins/jobs/projects.yaml) and wait for/trigger one refresh
#. Temporarily set governance.openstack.org/ docroot to /srv/static/tc
   (in modules/openstack_project/manifests/static.pp)
#. Set up a redirect from /tc/ to /srv/static/tc, while still using it as
   docroot (in modules/openstack_project/manifests/static.pp)
#. Publish openstack/governance-website content under /srv/static/governance
   (modify jenkins/jobs/projects.yaml and zuul/layout.yaml)
#. Alter ./modules/openstack_project/templates/static-governance.vhost.erb
   so that it supports a list of local redirects
#. Set up such redirects for /reference/ -> /tc/reference/,
   /resolutions/ -> /tc/resolutions/, and /goals/ -> /tc/goals in
   modules/openstack_project/manifests/static.pp
#. Set /srv/static/governance back as governance.openstack.org docroot
   in modules/openstack_project/manifests/static.pp

Repositories
------------

openstack/governance-website

Servers
-------

No new servers, this leverages static.openstack.org.

DNS Entries
-----------

No new entry, this leverages governance.openstack.org.

Documentation
-------------

I believe that this spec and changes to system-config and project-config repos
will be enough documentation.

Security
--------

I do not expect any new security concerns.

Testing
-------

I don't believe that this spec introduces any infra specific testing.


Dependencies
============

None outside of this spec.
