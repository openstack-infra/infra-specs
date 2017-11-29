::

  Copyright 2017 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Top-Level Project Hosting
=========================

https://storyboard.openstack.org/#!/story/2001382

The OpenStack Foundation has embarked on an effort to foster new
"`open infrastructure`_" projects.  To facilitate that, provide
infrastructure support for projects using non "OpenStack" branding.

.. _open infrastructure:
   http://lists.openstack.org/pipermail/foundation/2017-November/002532.html

Problem Description
===================

All of our existing project hosting services are OpenStack branded.
We'd like to support other projects, but gain an economy of scale from
our existing services where possible.

Each of the services we operate can be evaluated on two axes: how
desirable it is to operate that service under an individual project's
brand, and how technically feasible it is to do so.  For example, it
is both very desirable and feasible to operate a web site for a
project under its own brand.  It is somewhat desirable and feasible to
do the same for mailing lists.  It may be desirable, but under our
current constraints, less feasible to do the same for Gerrit.  And due
to the cross-project integration nature of our CI system, Zuul, it may
be feasible but less desirable to operate the CI system under more
than one brand.

This specification focuses only on those services where using an
individual project's brand is both desirable and feasible.

To address those services for which this is either not desirable or
feasible, we are discussing moving those services to a neutral brand.
This specification does not address that, other than it is intended to
be compatible with it should we do so in the future.

Proposed Change
===============

This describes how we can support multiple top-level projects for
several (but not all) of our existing services.  We may want to extend
this to cover other services later, but for now, the focus is on the
most critical for new projects.

DNS Servers
-----------

To facilitate the hosting of new domains, we will set up at least two
authoritative DNS servers, ns1.openstack.org and ns2.openstack.org,
and host them in different cloud regions.

We will deploy bind and/or NSD on these servers (implementors choice).

They will simply serve zone files directly from git repos which will
undergo code review in the normal manner.

Eventually we will probably want to change the domain names of these
dns servers if we rebrand the project infrastructure service itself to
something other than "OpenStack".

Note, this describes a service for new domains, and does not alter the
way we manage the openstack.org zone.

Mailing Lists
-------------

Mailman supports multi-domain hosting, as long as the mailing list
names are unique across domains.  In order to allow for such
conflicts, as well as per-domain themes, we will need to use multiple
installations of Mailman.

To accomplish this, alter the mailman puppet module to create a new
local mailman installation for every domain we host.  Then allow
creation of lists within that domain as usual.

The exim config will need to be updated to support multiple domains as
well.

Git Servers
-----------

Alter jeepyb and the projects.yaml file it reads to output a cgitrc
file for each domain.  The apache virtualhosts on the git farm can set
the CGIT_CONFIG env variable as appropriate, so that each domain
displays only the relevant projects.

Some new top-level projects may have code in OpenStack's Gerrit
already.  To facilitate these cases, create symlinks on the git
servers so that every project can be cloned without it's prefix.  We
could just do this as necessary for specific projects, however, this
is a likely step in the process of flattening the OpenStack git
namespace anyway, so we may as well solve the problem once globally.

This should work because we do, as a matter of policy, require unique
project names regardless of the prefix.

Websites
--------

Projects can add new virtualhosts to files.openstack.org to serve
static content from AFS.  This will let them take advantage of the
existing docs publishing jobs (or write their own jobs that use static
generators) to serve websites in the same way.  Each new project site
would get its own AFS volume.

Alternatives
------------

One alternative would be to ignore the issue and let other people
bring up their own infrastructure for other top-level projects.  But
we'd rather collaborate with them on all of the projects that the
OpenStack Foundation supports, to increase the diversity of
contributions all around.  Many services will benefit from an economy
of scale as well.

With collaboration, an alternative implementation would be to simply
spin up new severs for every project.  This ignores the economy of
scale we can obtain in some cases.  Our infrastructure is still not
*completely* automated and in many cases, that may increase the amount
of work compared to colocation.

Another alternative applicable to some services would be to alter our
infrastructure to be more automated.  Particularly with the use of
containers, this could achivee a good balance between economies of
scale and service complexity.  However, that will require more design
before implementation, and there are projects that are ready to begin
using these services now.  Instead, we should implement what we can
with our current infrastructure quickly, with an eye to moving to
automated container-based systems in the future.

An alternative to running our own authoritative DNS servers would be
to use an existing cloud service.  However, a strongly desired feature
of the proposed system is to have the option of managing the contents
of DNS in git, in standard zone files.  We may, in the future, write
tooling to translate zone files to API calls, but the proposal is the
simplest method to start.

Implementation
==============

Assignee(s)
-----------

* corvus

Gerrit Topic
------------

Use Gerrit topic "project-hosting" for all patches related to this spec.

.. code-block:: bash

    git-review -t project-hosting

Work Items
----------

* Create DNS puppet modules

* Create DNS servers

* Update Mailman puppet modules to support multiple sites

* Update existing Mailman config to use new site paradigm

* Symlink git repos on git farm

* Add support to jeepyb for multiple sites

* Add support to git farm puppet modules for multiple cgit sites

* Update projects.yaml to specify jeepyb site

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.

Repositories
------------

We may need new repositories for DNS related puppet modules.

When new top-level projects are added, we will likely need new git
repositories to host DNS and web content for each.

Servers
-------

We will add new authoritative DNS servers.  Otherwise, all services
will use existing servers.


DNS Entries
-----------

We will need to manually add the entries for the new ADNS servers.
Beyond that, new projects should be able to use these servers for
their own DNS hosting.

Documentation
-------------

System-config documentation for these services should be updated to
match.  Eventually we may want to add an overview page for services
offered to top-level projects, but that is not necessary as a first
step.

Security
--------

This should not alter the security posture of any of the affected
services.

A new service, ADNS servers, is proposed.  The software will be
supplied by the OS with automatic security updates.  We will need to
operate that service according to current best practices, for
instance, disabling or restricting AXFR to avoid being used in a
reflection attack.

Testing
-------

Puppet modules will be tested on throwaway dev servers before use.

Dependencies
============

No dependencies.
