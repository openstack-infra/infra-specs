::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

================================
Puppet Module Functional Testing
================================

https://storyboard.openstack.org/#!/story/2000250

Perform functional testing of our puppet modules and the services they
deploy to assert that they work for both us and others.

Problem Description
===================

Today our puppet module and deployed service testing is very rudimentary.
We do linting of our puppet modules and noop runs to make sure that the
puppet parser is happy with them. This catches quite a few issues but does
nothing to assert that the puppet modules actually deploy a working service
when run in non noop mode.

We should be testing that changes to the puppet modules do not break
existing interfaces that we and others use. At a high level we can do this
by running puppet apply (not in noop mode) against a module then assert
things about the resulting state both for the running service and changes
puppet should be making.

Proposed Change
===============

The framework proposed to assist with functional testing is Beaker-rspec
<https://github.com/puppetlabs/beaker-rspec>. Beaker is an all-in-one testing
harness that spins up a virtual machine, provisions it, copies puppet manifests
onto it, runs puppet apply with detailed exit codes, and asserts state.
Beaker-rspec is a layer on top of Beaker that provides rspec (a ruby testing
DSL) syntax to describe behavior. Beaker-rspec is the puppet module testing
framework that is recommended by Puppet Labs and widely adopted in the puppet
community. The OpenStack Puppet Modules are currently using this framework, so
the implementation is largely already in place.

Because beaker is an all-in-one tool that controls the management and
provisioning of virtual machines, it has the advantage that it is easy for
developers to run tests locally using its built-in support for vagrant.
However, this monolithic design is also problematic in the openstack-infra CI
environment because it insists on managing several steps for which we already
have existing infrastructure. However, workarounds have been developed and the
OpenStack Puppet Modules are using these workarounds in their functional
testing jobs (currently non-voting).

Keep in mind that this does not aim to provide integration testing. For
example when testing the puppet-jenkins module we would not also have a
Zuul service talking to it, but may consider triggering a Jenkins job
via Gearman directly. Functional testing for many services may not be possible
without complicated mocks for dependency services. In cases like this we should
focus on asserting the configured state is correct. The functional testing step
will be followed by an integration testing step, so it is sufficient at this
stage to just test characteristics of some of the installations instead of the
running services themselves.

Alternatives
------------

* `Rspec-system <https://github.com/puppetlabs/rspec-system>`_.
  Rspec-system is the predecessor to beaker-rspec and is largely comparable to
  it. However, it is no longer maintained.

* `Envassert <https://pypi.python.org/pypi/envassert>`_.
  This is similar to serverspec, but it is written in python and depends
  on fabric. I would expect us to use envassert in a python unittest test
  runner so that we can do a simple `tox -efunc` after the puppet run
  completes and have that run our tests. Probably the only real issue
  here is that it relies on fabric which some may see as competing with
  puppet and ansible. To an extent this is probably true as you basically
  have to run fabric around envassert to get all the environment asserting
  goodness.

* Python unittest tests.
  We could just use a python test runner then write our own python tests
  using the unittest framework. This would be most like the existing tests
  that we run today, but would also likely require some lib building on our
  part. Anything that requires OS agnosticism will have to be built in
  somehow, though the majority of test cases can probably be checked with
  the os, subprocess, and requests modules.

* Collection of shell scripts
  Basically what it says, run a collections of shell scripts in some order
  (possibly via run parts or xargs), then check they all return 0. This is
  the simplest option and likely quickest to bootstrap, but also has almost
  no existing structure we can build around. It will likely lead to
  reinventing utility functions in each puppet module and tests that are not
  very similar to each other. Long term the cost for this is very high.

* We can perform rspec unittesting of the puppet modules instead of doing
  functional testing. This can have limited value in some cases, such as:

    * checking that everything actually parses & compiles
    * conditional logic testing
    * template output & logic testing

  Consensus seems to be that the value here is limited, and not worth the
  effort when compared to functional testing.

* Rely on integration testing instead. There is a spec up for integration testing.
  I think tempest has taught as an important lesson about integration testing.
  It is very useful for making sure that the interactions between services
  remain sane, but it is not so good at providing an easy to debug environment
  when something breaks at a functional level. We should learn from tempest
  and do both integration testing and functional testing. They complement
  each other and one is not a substitute for the others.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
        nibalizer

Other assignees:
        crinkle
        Other volunteers?

Gerrit Topic
------------

Use Gerrit topic "puppet-func-testing" for all patches related to this spec.

.. code-block:: bash

    git-review -t puppet-func-testing

Work Items
----------

#. Add Gemfiles to the puppet modules. Jenkins use the Gemfile to install the required gems, namely beaker-rspec.

#. Add nodepool and vagrant "nodesets" to the puppet modules. Beaker nodesets
   are YAML config files that tell Beaker how to manage and provision the node it
   is going to test. A nodepool nodeset contains the workarounds that allow us to
   tell beaker to manage and provision localhost instead of an external node. On
   an Ubuntu platform, this nodeset will look like this::

       HOSTS:
         ubuntu-14.04-amd64:
           roles:
             - master
           platform: ubuntu-14.04-amd64
           hypervisor : none
           ip: 127.0.0.1
       CONFIG:
         type: foss
         set_env: false

   Even though nodepool provisions the platform, beaker requires us to specify a
   valid platform so that it can do provisioning of its own. A centos 7 node will
   specify "el-7-x86_64" as its platform.

   "Hypervisor" is the term beaker uses for a virtual machine management API such
   as vagrant, OpenStack, Amazon, libvirt, etc. Setting it to "none" skips this part.

   Beaker is heavily reliant on creating SSH connections with its host under
   test, so we must specify an IP of 127.0.0.1 so that it does not try to assign
   an alternate IP address.

   The set_env option prevents beaker from modifying the sshd_config on the
   node, since the infra jobs already carefully manage this.

   The vagrant nodesets can be copied directly from one of the Puppet Labs
   modules with no modification.

#. Add spec/spec_helper_acceptance.rb to the puppet modules to control
   provisioning steps, such as installing puppet and other modules. This can be
   largely inspired by the Puppet Labs modules and the OpenStack Puppet
   Modules, but will be customized to work in the nodepool environment. We need
   to investigate the possibility of using zuul-cloner here to help with
   inter-dependent changes.

#. Add tests and puppet manifests. Tests are written in the Rspec DSL. The
   puppet manifests will be stored as fixtures, separate from the tests. This
   will be beneficial if we decide to replace beaker with an alternate testing
   framework, as the important part is in the fixtures.

#. Add jobs in JJB to run the tests, following the example of the jobs already
   in place for the OpenStack Puppet Modules.

#. In the future, it may be beneficial to create a new "hypervisor" for beaker
   to get better support for the hack that we're doing here.

#. Create and maintain a wiki page for issues we have with beaker and
   beaker-rspec, both for the purposes of helping the maintainers improve the
   tools as well as to help compare an alternate tool in the future.

Repositories
------------

No new repositories necessary. We will update project-config and the existing
puppet modules.

Servers
-------

The existing devstack-* test slaves should be perfect platforms to run
this testing on.

DNS Entries
-----------

None

Documentation
-------------

We will need to update the per module developer documentation to teach
developers how to run the tests locally and how to add new tests when
they make changes.

Security
--------

Since beaker will be run within a single nodepool node, this does not pose any
additional risk on top of what we already assume given that we run arbitrary
code on our nodes.

Testing
-------

These changes should be self tested by the new tests that we are adding.
So add the new tests and let them run.

Dependencies
============

The beaker-rspec gem and its dependencies will be installed via `bundle
install` using each module's Gemfile.
