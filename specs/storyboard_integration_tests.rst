::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
    This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=============================
StoryBoard: Integration tests
=============================

StoryBoard needs to have integration tests to ensure that all components
are interacting together properly, and to confirm that changes done
in a component are not affecting the normal behaviour of the others.


Problem description
===================

Rationale
---------

StoryBoard is composed of several components that interact together.
At the moment we have storyboard project for the backend, and
storyboard-webclient for the frontend. We also have the python-storyboard
client and in the future more components can be added.

We have tests to ensure the quality of individual components, but we
don't have anything in place to ensure that changes done in one
project are not affecting the others. There is a subset of tests
created under StoryBoard webclient, but they only run when a change
is done in the frontend, not in the backend.

There is a strong need to create a set of tests to connect all these
components together. Changes done in StoryBoard backend need to be
tested against webclient and python-storyboard. Changes done either
in the webclient or on python-storyboard need to be tested against
the backend.


Proposed change
===============

Creation of integration tests for StoryBoard
--------------------------------------------

A new macro and related jobs using that need to be created
into the Infrastructure project. These tests need to be
called on check and gate on each affected project, and need to
run a complete suite of tests that ensure that changes done in
one component do not affect the others.

The related client components (at the moment
storyboard-webclient and python-storyboard client) need to
include integration tests on their repository. These tests will
run against the backend.

Workflow for an integration test execution
------------------------------------------

* A change is created on an StoryBoard component, that triggers
  the integration tests:

  - On a change to the backend, all client integration tests
    will run.

  - On a change to a client, that client's tests will be run.

* A setup script that installs the StoryBoard backend is
  executed. That script will install StoryBoard API server
  along with its dependencies, properly configured to run
  the tests. This script will belong to the scope of the
  StoryBoard backend project.

* Integration tests for the StoryBoard webclient and python
  client can be run now. The task to set the environment to run
  the tests will be in the scope of each client, because they are
  using different languages (javascript, python), so the tooling
  is totally different. We also ensure that if new projects are
  added into the future, can take care of their own tests.


Implementation
==============

Implementation of the integration tests needs to be different
depending on the client, and details are left to the person
implementing that, but following the exposed premises.

As described, a new builder needs to be created on
project-config, that takes care of the clone and backend
installation. Zuul-cloner will be used to set up the git
repositories for all components at the start of each test job.

Jobs will be runing in a one-per-one basis, it means:
* storyboard-webclient    => storyboard-backend
* python-storyboardclient => storyboard-backend

A change done in the backend will trigger both jobs, and if
new components are added we need to be sure that all
combinations are covered.


Assignee(s)
-----------
Primary Assignee:
    TBD

Work Items
----------
TBD

Repositories
------------
N/A

Servers
-------
This will involve testing on reusable devstack-trusty nodes.

DNS Entries
-----------
No new DNS entries.

Documentation
-------------
TBD

Security
--------
TBD

Testing
-------
Once tests are on place we need to run all possible combinations
to ensure that integration tests are on place and working.
