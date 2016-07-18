::

  Copyright 2016 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

========================
Newton testing on Xenial
========================

Include the URL of your StoryBoard story:

https://storyboard.openstack.org/#!/story/2000668

A new Ubuntu LTS release has happened and in order to keep up with the
TC's assertion that we support latest Ubuntu LTS and RHEL release we
need to shift our testing onto this new release. General plan is move the
default test image type from Ubuntu Trusty to Ubuntu Xenial for all testing
on branches >= Newton (currently Newton is called master).

Problem Description
===================

This poses a couple of issues. First we need a way to have Zuul instruct
Zuul launchers on using the appropriate image type for jobs based on the
branch. Second we need to do this in a way that avoids exploding the total
number of Gearman function registrations as there are scaling limits there.

Proposed Change
===============

Using JJB create jobs that are image type unique. This means instead of
having::

  gate-tempest-dsvm-full

we will have::

  gate-tempest-dsvm-full-ubuntu-trusty
  gate-tempest-dsvm-full-ubuntu-xenial

We can build these jobs from the same JJB job template ensuring the actual
content of the jobs is shared.

Going this route and using a default Zuul Launcher or Jenkins with Gearman
plugin will result in the following job registrations::

  gate-tempest-dsvm-full-ubuntu-trusty
  gate-tempest-dsvm-full-ubuntu-trusty:ubuntu-trusty
  gate-tempest-dsvm-full-ubuntu-xenial
  gate-tempest-dsvm-full-ubuntu-xenial:ubuntu-xenial

which is problematic because it will double our total number of Gearman
function registrations on an already stressed system. We can address that
by updating Zuul Launcher to optionally drop the label specific function
registrations. This has been implemented in
https://review.openstack.org/#/c/336311/

Alternatives
------------

An alternative solution would be to do what we did during the Precise to
Trusty transition. During that transition we used Zuul parameter functions
to introspect the branch under test then set the ZUUL_NODE parameter to
choose the correct test node type. The problem with this method was that
it hid away a lot of the node selection logic in the parameter function
where people did not expect to find it. On top of that any test not wanting
to use this default split (eg anything wanting to use centos) needed an
explicit override in that parameter function.

General consensus seems to be that we should try something different even
if we know there are other deficiencies with the other options as this
particular option was painful.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  clarkb

Gerrit Topic
------------

Use Gerrit topic "newton-xenial" for all patches related to this spec.

.. code-block:: bash

    git-review -t newton-xenial

Work Items
----------

Update Zuul Launcher code to optionally prevent label registrations.
Update Zuul Launcher installs and config to prevent label registrations.
Notify dev mailing list of switch.
Convert groups of jobs over to being run in a split setup. Likely we
will want to do this with a common set at a time. pep8, docs, unittests,
functional tests, tempest/integration tests. We can perform tests of
these jobs ahead of time as well to ensure that things work well.

Repositories
------------

No new repos required.

Servers
-------

This will make use of the existing ubuntu-xenial test images and instances.
We will also be taking advantage of the Xenial Ubuntu apt mirror and the
Xenial wheel mirror.

DNS Entries
-----------

No new DNS entries.

Documentation
-------------

The biggest impact here should just be in notifying the developers of the
switch. If we do this switchover like we have done it in the past it will
happen in a controlled manner after we have tested that things generally
work. It is possible that we will run into another major bug like the
Trusty python34 garbage collection bug so we should notify people that
this change is happening.

Security
--------

No new security issues as this only affects single use test instances that
already hand out root.

Testing
-------

For previous distro release transitions we have held test instances
and run unittests and integration tests on them to ensure that the new
platform works as expected.

Dependencies
============

- Just the new Zuul Launcher config being merged and deployed.
