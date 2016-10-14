::

  Copyright (c) 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=======
Zuul v3
=======

Storyboard: https://storyboard.openstack.org/#!/story/2000305

As part of an effort to streamline Zuul and Nodepool into an
easier-to-use system that scales better and is more flexible, some
significant changes are proposed to both.  The overall goals are:

* Make zuul scale to thousands of projects.
* Make Zuul more multi-tenant friendly.
* Make it easier to express complex scenarios in layout.
* Make nodepool more useful for non virtual nodes.
* Make nodepool more efficient for multi-node tests.
* Remove need for long-running slaves.
* Make it easier to use Zuul for continuous deployment.
* Support private installations using external test resources.
* Keep zuul simple.

Problem Description
===================

Nodepool
--------

Currently Nodepool is designed to supply single-use nodes to jobs.  We
have extended it to support supplying multiple nodes to a single job
for multi-node tests, however the implementation of that is very
inefficient and will not scale with heavy use.  The current system
uses a special multi-node label to indicate that a job requires the
number of nodes provided by that label.  It means that pairs (or
triplets, or larger sets) of servers need to be created together,
which may cause delays while specific servers are created, and servers
may sit idle because they are destined only for use in multi-node jobs
and can not be used for jobs which require fewer (or more) nodes.

Nodepool also currently has no ability to supply inventory of nodes
which are not created and destroyed.  It would be nice to allow
nodepool to mediate access to real hardware, for instance.

Zuul
----

Zuul is currently fundamentally a single-tenant application.  Some
folks want to use it in a multi-tenant environment.  Even within
OpenStack, we have use for multitenancy.  OpenStack might be one
tenant, and each stackforge project might be another.  Even without
the OpenStack/stackforge divide, we may still want the kind of
separation multi-tenancy can provide.  Multi-tenancy should allow for
multiple tenants to have the same job and project names, but also to
share configuration if desired.  Tenants should be able to define
their own pipelines, and optionally control some or all of their own
configuration.

OpenStack's Zuul configuration currently uses Jenkins and Jenkins Job
Builder (JJB) to define jobs.  We use very few features of Jenkins and
Zuul was designed to facilitate our move away from Jenkins.  The JJB
job definitions are complex, and some of the complexity comes from
specific Jenkins behaviors that we currently need to support.
Additionally, there is no support for orchestrating actions across
multiple test nodes.

Proposed Change
===============

Nodepool
--------

Nodepool should be made to support explicit node requests and
releases.  That is to say, it should act more like its name -- a node
pool.  It should support the existing model of single-use nodes as
well as long-term nodes that need mediated access.

Nodepool should implement the following gearman function to get one or
more nodes::

  request-nodes:
    input: {node-types: [list of node types],
            request: a unique id for the request,
            requestor: unique gearman worker id (eg zuul)}

When multiple nodes are requested together, nodepool will return nodes
within the same AZ of the same provider.

Requests for nodes will go into a FIFO queue and be satisfied in the
order received according to node availability.  This should make
demand and allocation calculations much simpler.

A node type is simply a string such as 'trusty', that corresponds to
an entry in the nodepool config file.

The requestor is used to identify the system that is requesting the
node.  To handle the case where the requesting system (eg Zuul) exits
abruptly and fails to return a node to the pool, Nodepool will reverse
the direction of gearman function invocation when supplying a set of
nodes.  When completing allocation of a node, nodepool invokes the
following gearman function::

  accept-nodes:<requestor>:
    input: {nodes: [list of node records],
            request: the unique id from the request}
    output: {used: boolean}

If `zuul` was the requestor supplied with request-nodes, then the
actual function invoked would be `accept-nodes:zuul`.

A node record is a dictionary with the following records: id,
public-ipv4, public-ipv6, private-ipv4, private-ipv6, hostname,
node-type.  The list should be in the same order as the types
specified in the request.

When the job is complete it will return a WORK_COMPLETE packet with
`used` set to true if any nodes were used.  `used` will be set to
false if all nodes were unused (for instance, if Zuul no longer needs
the requested nodes).  In this case, the nodes may be reassigned to
another request.  If a WORK_FAIL packet is received, including due to
disconnection, the nodes will be treated as used.

Nodepool will then decide whether the nodes should be returned to the
pool, rebuilt, or deleted according to the type of node and current
demand.

This model is much more efficient for multi-node tests, where we will
no longer have to have special multinode labels.  Instead the
multinode configuration can be much more ad-hoc and vary per job.

Nodepool should also allow the specification of static inventory of
non-dynamic nodes.  These may be nodes that are running on real
hardware, for instance.

Zuul
----

Tenants
~~~~~~~

Zuul's main configuration should define tenants, and tenants should
specify config files to include.  These include files should define
pipelines, jobs, and projects, all of which are namespaced to the
tenant (so different tenants may have different jobs with the same
names)::

  ### main.yaml
  - tenant:
      name: openstack
      include:
        - global_config.yaml
        - openstack.yaml

Files may be included by more than one tenant, so common items can be
placed in a common file and referenced globally.  This means that for,
eg, OpenStack, we can define pipelines and our base job definitions
(with logging info, etc) once, and include them in all of our tenants::

  ### main.yaml (continued)
  - tenant:
      name: openstack-infra
      include:
        - global_config.yaml
        - infra.yaml

A tenant may optionally specify repos from which it may derive its
configuration.  In this manner, a repo may keep its Zuul configuration
within its own repo.  This would only happen if the main configuration
file specified that it is permitted::

  ### main.yaml (continued)
  - tenant:
      name: random-stackforge-project
      include:
        - global_config.yaml
      source:
        my-gerrit:
          repos:
          - stackforge/random  # Specific project config is in-repo

Nodesets
~~~~~~~~

A significant focus of Zuul v3 is a close interaction with Nodepool to
both make running multi-node jobs simpler, as well as facilitate
running jobs on static resources.  To that end, the node configuration
for a job is introduced as a first-class resource.  This allows both
simple and complex node configurations to be independently defined and
then referenced by name in jobs::

  ### global_config.yaml
  - nodeset:
      name: precise
      nodes:
        - name: controller
          image: ubuntu-precise
  - nodeset:
      name: trusty
      nodes:
        - name: controller
         image: ubuntu-trusty
  - nodeset:
      name: multinode
      nodes:
        - name: controller
          image: ubuntu-xenial
        - name: compute
         image: ubuntu-xenial

Jobs may either specify their own node configuration in-line, or refer
to a previously defined nodeset by name.

Jobs
~~~~

Jobs defined in-repo may not have access to the full feature set
(including some authorization features).  They also may not override
existing jobs.

Job definitions continue to have the features in the current Zuul
layout, but they also take on some of the responsibilities currently
handled by the Jenkins (or other worker) definition::

  ### global_config.yaml
  # Every tenant in the system has access to these jobs (because their
  # tenant definition includes it).
  - job:
      name: base
      timeout: 30m
      nodes: precise
      auth:
        inherit: true  # Child jobs may inherit these credentials
        swift:         # Swift usage may only be defined in config repo
          - container: logs
      workspace: /opt/workspace  # Where to place git repositories
      post-run:
        - archive-logs

Jobs have inheritance, and the above definition provides a base level
of functionality for all jobs.  It sets a default timeout, requests a
single node (of type precise), and requests swift credentials to
upload logs.  For security, job credentials are not available to be
inherited unless the 'inherit' flag is set to true.  For example, a
job to publish a release may need credentials to upload to a
distribution site -- users should not be able to subclass that job and
use its credentials for another purpose.

Further jobs may extend and override the remaining parameters::

  ### global_config.yaml (continued)
  # The python 2.7 unit test job
  - job:
      name: python27
      parent: base
      nodes: trusty

Our use of job names specific to projects is a holdover from when we
wanted long-lived slaves on Jenkins to efficiently re-use workspaces.
This hasn't been necessary for a while, though we have used this to
our advantage when collecting stats and reports.  However, job
configuration can be simplified greatly if we simply have a job that
runs the python 2.7 unit tests which can be used for any project.  To
the degree that we want to know how often this job failed on nova, we
can add that information back in when reporting statistics.  Jobs may
have multiple aspects to accomodate differences among branches, etc.::

  ### global_config.yaml (continued)
  # Version that is run for changes on stable/diablo
  - job:
      name: python27
      parent: base
      branches: stable/diablo
      nodes:
        - name: controller
          image: ubuntu-lucid

  # Version that is run for changes on stable/juno
  - job:
      name: python27
      parent: base
      branches: stable/juno  # Could be combined into previous with regex
      nodes: precise         # if concept of "best match" is defined

Jobs may specify that they require more than one node::

  ### global_config.yaml (continued)
  - job:
      name: devstack-multinode
      parent: base
      nodes: multinode

Jobs may specify auth info::

  ### global_config.yaml (continued)
  - job:
      name: pypi-upload
      parent: base
      auth:
        secrets:
          - pypi-credentials
          # This looks up the secrets bundle named 'pypi-credentials'
          # and adds it into variables for the job

Jobs may indicate that they may only be used by certain projects::

  ### shade.yaml (continued)
  - job:
      name: shade-api-test
      parent: base
      allowed-projects:
        - openstack-infra/shade
      auth:
        secrets:
          - shade-cloud-credentials

Note that this job may not be inherited from because of the auth
information.

Secrets
~~~~~~~

The `auth` attribute of a job provides way to add authentication or
authorization requirements to a job.  Examples above include `swift`
and `secrets`, though other systems may be added.

A `secret` is a collection of key/value pairs and is defined as a
top-level configuration object::

   ### global_config.yaml (continued)
   - secret:
     name: pypi-credentials
     data:
       username: !encrypted/pkcs1 o+7OscBFYWJh26rlLWpBIg==
       password: !encrypted/pkcs1 o+7OscBF8GHW26rlLWpBIg==

PKCS1 with RSAES-OAEP (implemented by the Python `cryptography`
library) will be used so that the data are effectively padded.  Since
the encryption scheme is specified by a YAML tag (`encrypted/pkcs1` in
this case), this can be extended later.

Zuul will maintain a private/public keypair for each repository
(config or project) specified in its configuration.  It will look for
the keypair in `/var/lib/zuul/keys/<source name>/<repo name>.pem`.  If
a keypair is needed but not available, Zuul will generate one.  Zuul
will serve the public keys using its web server so that users can
download them for use in creating the encrypted secrets.  It should be
easy for an end user to encrypt a secret, whether that is with an
existing tool such as OpenSSL or a new Zuul CLI.

There is a keypair for each repository so that users can not copy a
ciphertext from a given repo into a different repo that they control
in order to coerce Zuul into decrypting it for them (since the private
keys are different, decryption will fail).

It would still be possible for a user to copy a previously (or even
currently) used secret in that same repo.  Depending on how expansive
and diverse the content of that repo is, that may be undesirable.
However, this system allows for management of secrets to be pushed
into repos where they are used and can be reviewed by people most
knowledgable about their use.  By facilitating management of secrets
by repo specialists rather than forcing secrets for unrelated projects
to be centrally managed, this risk should be minimized.

Further, a secret may only be used by a job that is defined in the
same repo as that secret.  This prevents users from defining a job
which requests unrelated secrets and exposes them.

In many cases, jobs which use secrets will be safe to use by any
repository in the system (for example, a Pypi upload job can be
applied to any repo because it does not execute untrusted code from
that repo).  However, in some cases, jobs that use secrets will be too
dangerous to allow other repositories to use them (especially when
those repositories may be able to influence the job and cause it to
expose secrets).  We should add a flag to jobs which indicate that
they may only be used by certain projects (typically only the repo in
which they are defined).

Pipelines may be configured to either allow or disallow the use of
secrets with a new boolean attribute, 'allow-secrets'.  This is
intended to avoid the exposure of secrets by a job which was subject
to dynamic reconfiguration in a check pipeline.  We would disable the
use of secrets in our check pipelines so that no jobs with secrets
could be configured to run in it.  However, jobs which use secrets for
pre-merge testing (for example, to perform live API testing on a
public cloud) could still be run in the gate pipeline (which would
only happen after human review verified they were safe), or an access
restricted on-demand pipeline.

Projects
~~~~~~~~

Pipeline definitions are similar to the current syntax, except that it
supports specifying additional information for jobs in the context of
a given project and pipeline.  For instance, rather than specifying
that a job is globally non-voting, you may specify that it is
non-voting for a given project in a given pipeline::

  ### openstack.yaml
  - project:
      name: openstack/nova
      gate:
        queue: integrated  # Shared queues are manually built
        jobs:
          - python27  # Runs version of job appropriate to branch
          - pep8:
              nodes: trusty  # override the node type for this project
          - devstack
          - devstack-deprecated-feature:
              branches: stable/juno  # Only run on stable/juno changes
              voting: false  # Non-voting
      post:
        jobs:
          - tarball:
              jobs:
                - pypi-upload

Templates are still supported.  If a project lists a job that is
defined in a template that is also applied to that project, the
project-local specification of the job will modify that supplied by
the template.

Currently unique job names are used to build shared change queues.
Since job names will no longer be unique, shared queues must be
manually constructed by assigning them a name.  Projects with the same
queue name for the same pipeline will have a shared queue.

A subset of functionality is available to projects that are permitted
to use in-repo configuration::

  ### stackforge/random/.zuul.yaml
  - job:
      name: random-job
      parent: base      # From global config; gets us logs
      nodes: precise

  - project:
      name: stackforge/random
      gate:
        jobs:
          - python27    # From global config
          - random-job  # Flom local config

Ansible
~~~~~~~

The actual execution of jobs will continue to be distributed to
workers over Gearman.  Therefore the actual implementation of how jobs
are executed will remain pluggable, however, the zuul-gearman protocol
will need to change.  Because the system needs to perform coordinated
tasks on one or more remote systems, the initial implementation of the
workers will use Ansible, which is particularly suited to that job.

The executable content of jobs should be defined as ansible playbooks.
Playbooks can be fairly simple and might consist of little more than
"run this shell script" for those who are not otherwise interested in
ansible::

  ### stackforge/random/playbooks/random-job.yaml
  ---
  hosts: controller
  tasks:
    - shell: run_some_tests.sh

Global jobs may define ansible roles for common functions::

  ### openstack-infra/zuul-playbooks/python27.yaml
  ---
  hosts: controller
  roles:
    - tox:
        env: py27

Because ansible has well-articulated multi-node orchestration
features, this permits very expressive job definitions for multi-node
tests.  A playbook can specify different roles to apply to the
different nodes that the job requested::

  ### openstack-infra/zuul-playbooks/devstack-multinode.yaml
  ---
  hosts: controller
  roles:
    - devstack
  ---
  hosts: compute
  roles:
    - devstack-compute

Additionally, if a project is already defining ansible roles for its
deployment, then those roles may be easily applied in testing, making
CI even closer to CD.

The pre- and post-run entries in the job definition might also apply
to ansible playbooks and can be used to simplify job setup and
cleanup::

  ### openstack-infra/zuul-playbooks/archive-logs.yaml
  ---
  hosts: all
  roles:
    - archive-logs: "/opt/workspace/logs"

Execution
~~~~~~~~~

A new Zuul component would be created to execute jobs.  Rather than
running a worker process on each node (which requires installing
software on the test node, and establishing and maintaining network
connectivity back to Zuul, and the ability to coordinate actions
across nodes for multi-node tests), this new component will pick up
accept jobs from Zuul, and for each one, write an ansible inventory
file with the node and variable information, and then execute the
ansible playbook for that job.  This means that the new Zuul component
will maintain ssh connections to all hosts currently running a job.
This could become a bottleneck, but ansible and ssh have been known to
scale to a large number of simultaneous hosts, and this component may
be scaled horizontally.  It should be simple enough that it could even
be automatically scaled if needed.  In turn, however, this does make
node configuration simpler (test nodes need only have an ssh public
key installed) and makes tests behave more like deployment.

To support the use case where the Zuul control plane should not be
accessible by the workers (for instance, because the control plane is
on a private network while the workers are in a public cloud), the
direction of transfer of changes under test to the workers will be
reversed.

Instead of workers fetching from zuul-mergers, the new zuul-launcher
will take on the task of calculating merges as well as running
ansible.


Continuous Deployment
~~~~~~~~~~~~~~~~~~~~~

Special consideration is needed in order to use Zuul to drive
continuous deployment of development or production systems.  Rather
than specifying that Zuul should obtain a node from nodepool in order
to run a job, it may be configured to simply execute an ansible task
on a specified host::

  - job:
      name: run-puppet-apply
      parent: base
      host: review.openstack.org
      fingerprint: 4a:28:cb:03:6a:d6:79:0b:cc:dc:60:ae:6a:62:cf:5b

Because any configuration of the host and credential information is
potentially accessible to anyone able to read the Zuul configuration
(which is everyone for OpenStack's configuration) and therefore could
be copied to their own section of Zuul's configuration, users must add
one of two public keys to the server in order for the job to function.
Zuul will generate an SSH keypair for every tenant as well as every
project.  If a user trusts anyone able to make configuration changes
to their tenant, then they may use Zuul's public key for their tenant.
If they are only able to trust their own project configuration in
Zuul, they may add Zuul's public key for that specific project.  Zuul
will make all public keys available at known HTTP addresses so that
users may retrieve them.  When executing such a job, Zuul will try the
project and tenant SSH keys in order.

Tenant Isolation
~~~~~~~~~~~~~~~~

In order to prevent users of one Zuul tenant from accessing the git
repositories of other tenants, Zuul will no longer consider the git
repositories it manages to be public.  This could be solved by passing
credentials to the workers for them to use when fetching changes,
however, an additional consideration is the desire to have workers
fully network isolated from the Zuul control plane.

Instead of workers fetching from zuul-mergers, the new zuul-launcher
will take on the task of calculating merges as well as running
ansible.  The launcher will then be responsible for placing prepared
versions of requested repositories onto the worker.

Status reporting will also be tenant isolated, however without
HTTP-level access controls, additional measures may be needed to
prevent tenants from accessing the status of other tenants.
Eventually, Zuul may support an authenticated REST API that will solve
this problem natively.

Alternatives
------------

Continuing with the status quo is an alternative, as well as
continuing the process of switching to Turbo Hipster to replace
Jenkins.  However, this addresses only some of the goals stated at the
top.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * corvus

Also:
  * jhesketh
  * mordred

Gerrit Branch
-------------

Nodepool and Zuul will both be branched for development related to
this spec.  The "master" branches will continue to receive patches
related to maintaining the current versions, and the "feature/zuulv3"
branches will receive patches related to this spec.  The .gitreview
files will be updated to submit to the correct branches by default.

Work Items
----------

* Modify nodepool to support new allocation and distribution (mordred)
* Modify zuul to support new syntax and isolation (corvus)
* Create zuul launcher (jhesketh)
* Prepare basic infra ansible roles
* Translate OpenStack JJB config to ansible

Repositories
------------

We may create new repositories for ansible roles, or they may live in
project-config.

Servers
-------

We may create more combined zuul-launcher/mergers.

DNS Entries
-----------

No changes other than needed for additional servers.

Documentation
-------------

This will require changes to Nodepool and Zuul's documentation, as
well as infra-manual.

Security
--------

No substantial changes to security around the Zuul server; use of Zuul
private keys for access to remote hosts by Zuul has security
implications but will not be immediately used by OpenStack
Infrastructure.

Testing
-------

Existing nodepool and Zuul tests will need to be adapted.
Configuration will be different, however, much functionality should be
the same, so many functional tests should have direct equivalencies.

Dependencies
============

None.
