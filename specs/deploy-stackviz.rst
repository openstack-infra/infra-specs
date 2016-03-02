::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Stackviz Deployment
===============================

Story: https://storyboard.openstack.org/story/2000498

Stackviz is a tool for visualizing Tempest gate runs. Presently, the project is
in a stable state under openstack/stackviz. This spec details the process of
getting Stackviz integrated on the build images to run with live data. For the
most up to date information on this process, see:

https://etherpad.openstack.org/p/BKgWlKIjgQ

Problem Description
===================

Stackviz currently exists under openstack/stackviz, and can be downloaded,
built and run against local test run data. The tool provides a timeline,
detailed test view, and traceback logs to help developers easily debug runs
that went wrong. However, this does not provide much utility to OS developers
about their current patches in the gate. For Stackviz to be a truly useful
debugging tool, it must be available on-demand, with little to no config work
on the part of the developer.

Proposed Change
===============

To solve this, we propose that Stackviz be integrated upstream with the
following two-pronged approach:

First, download a copy of Stackviz onto the Nodepool images. The required
npm dependencies will be downloaded and installed, and the static site will
be built. To accomplish this, a new Nodepool DIB element will be made for
the creation of the repository and building of the static site.

Second, copy and configure the static site on the logs server. Additional data
processing at this step will include installing the python processing module
(stackviz-export) and parsing subunit and dstat.csv logs. A configuration file
will be generated for each patch, which will allow the developer to browse a
Stackviz site on the logs server using their patch's data.

Alternatives
------------

A puppet deployment is another option for consideration (puppet-stackviz).
Traditionally, infra has deployed software via puppet modules and not DIB
elements. Stackviz would be breaking the pattern in this case.

A third-party Stackviz deployer is currently up for testing purposes at:
https://stackviz.timothyb89.org/

This demonstrates another possibility: creating a standalone server that pulls
logs from Gerrit at the request of a user to generate a Stackviz on-the-fly.
More details for this project can be viewed at:
https://etherpad.openstack.org/p/stackviz-deployment

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  austin81

Secondary assignee:
  timothyb89

Gerrit Topic
------------

Use Gerrit topic "deploy-stackviz" for all patches related to this spec.

.. code-block:: bash

    git-review -t deploy-stackviz

Work Items
----------

Currently, there are WIP patches for both parts of the process outlined above:
Nodepool element: https://review.openstack.org/279317/
Devstack-gate patch: https://review.openstack.org/212207/

See https://etherpad.openstack.org/p/BKgWlKIjgQ for more details

Repositories
------------

None

Servers
-------

The Nodepool images will need an additional DIB element to do pre-processing
work for Stackviz. The devstack-gate images will also require an additional
script for creating config files and hosting the sites.

DNS Entries
-----------

None

Documentation
-------------

If implemented correctly, Stackviz should fundamentally change the OS
developer workflow process. Checking the Stackviz site on the logs server
should become the first thing that the developers do once their patch goes
through the gate tests. http://docs.openstack.org/infra/manual/developers.html
will need to be updated.

Security
--------

None

Testing
-------

None

Dependencies
============

https://git.openstack.org/openstack/stackviz
