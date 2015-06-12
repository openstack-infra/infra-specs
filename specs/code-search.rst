::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Host a code search service
==========================

https://storyboard.openstack.org/#!/story/2000280

It is often convenient, for instance when researching certain
classes of widespread bugs or looking for implementation examples,
to be able to search through all of the OpenStack projects' source
code. As a service to our contributors and downstream communities,
we should provide a convenient mechanism to be able to do this.

Problem Description
===================

To host a code search service, we need to decide on the software
we'll use, implement a series of configuration management
changes/additions, write accompanying management documentation, and
build the resources needed to run it.

Proposed Change
===============

Create or reuse an existing Puppet module to deploy the Hound_ code
search engine and serve it via Apache from a dedicated virtual
machine under care of the OpenStack Infrastructure Project. A
proof-of-concept deployment is running at
http://hound.openstack.org/ but can be taken down once this work is
underway.

.. _Hound: https://github.com/etsy/Hound

Alternatives
------------

Some alternatives were explored:

1. Keep expecting people to clone every repo and loop through them
   with git grep? It works, but it's pretty painful.
2. A number of hosted code search services exist running on the
   public Internet, but for improved performance and customizability
   it's simpler to run one ourselves. Often those third party
   services get confused by our development workflow, are missing
   new repos, end up with stale repos indexed after renames and so
   on.
3. There are quite a few code search engines available, published as
   free software, besides Hound_. The alternative we tried most
   seriously (via a proof-of-concept demo) is
   `Livegrep <https://github.com/livegrep/livegrep>`_, but its
   indexes were static and had to be replaced via a fairly costly
   process considering the volume of source code we maintain. Others
   were ruled out for various similar reasons.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  taron

Infra root shepherds:
  fungi pleia2

Gerrit Topic
------------

Use Gerrit topic "code-search" for all patches related to this spec.

.. code-block:: bash

    git-review -t code-search

Work Items
----------

1. Initiate a change in ``openstack-infra/project-config`` to create
   an empty ``openstack-infra/puppet-hound`` Git repository as
   described in the `Repository Creator's
   Guide <http://docs.openstack.org/infra/manual/creators.html>`_.
   This should be accompanied by a related change to
   ``openstack/governance`` adding the new repo as part of the
   Infrastructure project.
2. Create a new ``hound`` Puppet module in
   ``openstack-infra/puppet-hound`` which enables an opinionated
   and working but not OpenStack-specific deployment of the service
   suitable for publication to the Puppet Forge. Existing
   ``openstack-infra/puppet-.*`` should be examined for inspiration
   and consistency. A rough change at
   https://review.openstack.org/178488 covers most of the features
   needed but is unnecessarily entangled with the
   ``openstack_project`` module. Feel free to copy that code and
   credit the original author with "Co-Authored-By: ..." in the
   commit message.
3. Upstream improvement to Hound_ is needed to support noticing
   config file changes without shutting down and restarting, filed
   as https://github.com/etsy/Hound/issues/119 and seems to have the
   general approval of its developer team.
4. Upstream improvement to Hound_ is needed to change the repo list,
   as it's a select box and doesn't support any namespacing making
   it particularly unwiedly for us. Typeahead completion would be
   great.
5. A change will be proposed to the
   ``openstack-infra/system-config`` Git repo lightly documenting
   deployment and management of the service and adding that document
   to the ``doc/source/index.rst`` file. Examples in the
   ``doc/source`` directory can be used for inspiration and
   consistency of style. This same change can also add a
   ``codesearch.openstack.org`` node to the global
   ``manifests/site.pp`` file along with (if needed) a
   ``modules/openstack_project/manifests/codesearch.pp`` class file
   and any accompanying custom config files or templates.
6. Deployment of the above changes will be performed by a root admin
   onto a new ``codesearch.openstack.org`` server, and related A,
   AAAA and PTR resource records will be created in DNS for it.
7. If necessary, use of the service can be documented in the
   ``openstack-infra/infra-manual`` repo as well.
8. Once the service appears stable and working as intended, announce
   it to the
   `openstack-dev <http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-dev>`_,
   `openstack-infra <http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-infra>`_
   and
   `openstack-operators <http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-operators>`_
   mailing lists.

Repositories
------------

An ``openstack-infra/puppet-hound`` Git repo will be created as part
of this plan.

Servers
-------

A ``codesearch.openstack.org`` server will be added, and the
existing ``hound.openstack.org`` proof-of-concept demo will be
deleted.

DNS Entries
-----------

A, AAAA and PTR resource records will be created for the
``codesearch.openstack.org`` server.

Documentation
-------------

The service's deployment and management will be documented in the
``openstack-infra/system-config`` repo, published to the
http://docs.openstack.org/infra/system-config/ site. Optionally, use
of the service can be documented in the `Infrastructure
Manual <http://docs.openstack.org/infra/manual>`_.

Security
--------

This is not a trusted service, needs no authentication for normal
use, and it runs on its own dedicated virtual machine. HTTPS should
not be necessary for this service, so no X.509 certificate will be
ordered.

Testing
-------

The configuration management for this service will be tested via
existing apply/syntax CI jobs.

Dependencies
============

- None identified.
