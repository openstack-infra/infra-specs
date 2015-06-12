::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

========================================
shade: A library that understands clouds
========================================

Infra uses multiple clouds and as a result has learned a lot about what needs
to be done to do that. In the interest of being good citizens, instead of that
knowledge being inside of nodepool, it should be in a reusable library.

Problem Description
===================

As much as OpenStack promises a utopian future where an application can be
written once and target multiple clouds that run OpenStack, the reality is
that deployer choice leaks through the abstractions to the point where the
end user must know about it. This causes logic to require a-priori knowledge
about clouds, as well as complex logic even on discoverable differences.

The current user interface libraries, `python-*client`, are particularly
user unfriendly as they were primarily written with server-to-server
communication in mind. They were also each designed completely differently
so that an application which uses more than one OpenStack feature becomes
quickly confusing to write.

In addition to Infra, `ansible` has a set of modules that focus on creating
and managing cloud resources. As part of using `ansible` to orchestrate
`puppet`, it only makes sense for Infra to use `ansible` to manage its
resources, which means that the logic Infra has learned about how that works
should be applicable. Specifics on using `ansible` for that purpose are
out of scope of this spec, but `ansible` upstream as a consumer is an
important design consideration.

Proposed Change
===============

The `shade` library will handle all of this. It will contain the logic learned
from `nodepool`, or moving forward, it will contain any new complex cloud
manipulation logic that `nodepool` needs. It should be considered that
`nodepool` is `shade's` primary user.

To that end, `shade` must support constructs like application based API rate
limiting and caching appropriate for long-lived connections.

A consumer of `shade` should never need to put in logic such as "if my cloud
supports X, then do Y, else Z". There are two situations in which such logic
might arise.

Firstly, there are two or more ways of doing the same logical action.
An example is getting a floating IP, which could be the purview of
`neutron` or of `nova`. `shade` should present a general `create_floating_ip`
to the user and hide all details about where it came from.

Secondly, there is functionality that simply does not exist on a cloud.
For example, some clouds are deployed without trove. In that case, the user
will receive an error message stating that the selected cloud does not support
managing trove resources.

The `python-*client` libraries are not written with end users in mind. They
have, as their primary use case, the enabling of server to server
communication. As such, they make a set of assumptions that is not in keeping
with a consumer point of view. Their use should be replaced by
`python-openstacksdk` once it is ready. However, it is not, so in the mean
time the `python-*client` libraries need to be used. As the future plan is to
replace them, all objects and exceptions they return should be expressly
hidden, even though masking exceptions is considered poor form.

A future state could be imagined where `shade` and `python-openstacksdk` merge,
but it does not seem to be the primary concern of either library at the moment.
If it did happen, it would likely be as a "simple" API or something on top of
or to the side of the rest of the SDK. The reasons for this largely is that
`python-openstacksdk` is more concerned with providing an SDK to program the
OpenStack APIs with - and `shade` is more concerned with hiding the ways in
which deployers have chosen to do things that leak through the API. It is
likely that a future state where `shade` is depreciated is one in which the
issues it deals with are bundled into the server APIs. In this instance, a
layer of business logic is not needed.

Passthrough access to the underlying `Client` objects is useful for phased
adoption of `shade`. Before 1.0 is released, removal should be considered, or
hidden behind a disableable warning. This is to ensure a user has to explicity
opt-in knowing that they are not part of the API.

`ansible` is the second user of `shade`. The main addition this brings is the
need for idempotent operations. The `ansible` modules must have enough in the
API to be able to provide that without large amounts of repeated logic in the
modules themselves. In fact, most of the `ansible` modules should actually
contain very little code that is not related to `ansible` argument processing
or interpretation of results into a suitable format.

Finally, it is not `shade's` purpose in life to express what is or is not
OpenStack, nor to be involved in such categorizations. Its job is to improve
the end user's experience. For that reason, `shade` should take a maximal
approach to including support for things. If someone wants to add support
for `designate` or `magnum` or `manila` or whatever, that's awesome.

It is a conscious and active decision to not use a plugin interface for this.
Because again `shade` exists to reduce the cognitive burden on the user, the
user should not have to know to install plugins to be able to use their cloud.
The two main reasons for pluggable clients in the past is:

* Strict policies on what is 'Integrated'
* To enable proprietary extensions

The first is no longer a problem for OpenStack broadly, and even if it was
it's still not a practical issue for an Infra project.

The second is the thing that will ultimately cause OpenStack to die if it is
allowed to continue. While the right of people to choose to destroy all the
goodness in the world is an important right for them to have, there is no need
for Infra to involve itself such a tragedy.

Anything that's in `shade` needs to be testable by running `shade` functional
tests against a devstack in the Infra gates.

There is currently one exception to the testable in Infra gates, which is that
the Rackspace Task API for Glance does not work in devstack, so we cannot test
it. We have an exception for this because at the moment, `nodepool` must use
that API, and it is an API that exists in glance, even if the backing code
is broken. However, the general rule stands, and any violations of that rule
need to be carefully considered exceptions - and probably accompanied by a
large amount of complaining.

Alternatives
------------

We could ignore writing a library and write all of our logic directly in
nodepool. This is problematic because it causes a lot of really useful code
and logic to not be easily reusable by the community at large.

We could write all of the logic directly in the ansible modules upstream and
then have nodepool turn into an engine which consumes the ansible modules. This
is more tempting, but ansible does not support long-lived objects, which means
that we'd be execing ansible on every operation which seems rather extreme. It
also means that people not using ansible would be unable to benefit from the
logic.

We could improve the client libraries or `python-openstacksdk`. We've tried to
include richer logic in the client libraries and have been told it's not what
they are for. The `python-openstacksdk` is still young and we've been told it's
not ready for production use yet. We need some of the logic for `shade` now,
so the timescale for getting it done in `python-openstacksdk` isn't very
workable.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mordred

Additional assignee(s):
  Shrews
  greghaynes
  dguerri
  TheJulia
  Spamaps

Gerrit Topic
------------

`shade` is a library itself, so there is no dedicated gerrit topic.

Work Items
----------

* Implement Image uploading for nodepool
* Get to feature parity with nodepool on floating-ips and server creation
* Implement ansible modules for every function in shade

Repositories
------------

openstack-infra/shade

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

`shade` needs developer documentation of its API

Security
--------

None

Testing
-------

`shade` should have both unit tests and functional tests. The functional
tests should run against devstack VMs. If a developer chooses to, they should
be able to manually run functional tests against live clouds, since the purpose
of shade is to enable use of myriad clouds, not to support or expose
theoretical APIs.

Dependencies
============

None
