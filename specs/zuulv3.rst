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

Nodepool should use ZooKeeper to fulfill node requests from Zuul.  A
request should be made using the Zookeeper `priority queue`_ construct
at the path::

  /nodepool/requests/500-123
    node_types: [list of node types]
    requestor: descriptive string of requestor (eg zuul)
    created_time: <unix timestamp>
    state: requested | pending | fulfilled | failed
    state_time: <unix timestamp>
    nodes: [list of node ids]
    declined_by: [list of launchers declining this request]

The name of the request node, "500-123", is composed of the priority
("500") followed by the sequence number ("123").  After creating the
request node, Zuul should read the request node back and set a watch
on it.  If the read associated with the watch set indicates that the
request has already been fulfilled, it should proceed to use the
nodes, otherwise, it should wait to be notified by the watch.  Note
special care will need to be taken to re-set watches if the connection
to ZooKeeper is reset.  The pattern of read to test whether request is
fulfilled and set watch if not can be repeated as many times as
necessary until the request is fulfilled.

This model is much more efficient for multi-node tests, where we will
no longer have to have special multinode labels.  Instead the
multinode configuration can be much more ad-hoc and vary per job.
Requests for nodes are in a FIFO queue and will be satisfied in the
order received according to node availability.  This should make
demand and allocation calculations much simpler.

A node type is simply a string such as 'trusty', that corresponds to
an entry in the nodepool config file.

The component of Nodepool which will process these requests is known
as a "launcher".  A Nodepool system may consiste of multiple launchers
(for instance, one launcher for each cloud provider).  Each launcher
will continuously scan the request queue (sorted by request id) and
attempt to process each request in sorted order.  A single launcher
may be engaged in satisfying multiple requests simultaneously.

When satisfying a request, Nodepool will first obtain a lock on the
request using the Zookeeper `lock construct`_ at the path::

  /nodepool/requests-lock/005-123

It will then attempt to satisfy the request from available nodes, and
failing that, cause new nodes to be created.  When multiple nodes are
requested together, nodepool will return nodes within the same AZ of
the same provider.

A simple algorithm which does not require that any launcher know about
any other launchers is:

#. Obtain next request
#. If image not available, decline
#. If request > quota, decline
#. If request < quota and request > available nodes (due to current
   usage), begin satisfying the request and do not process further
   requests until satisfied
#. If request < quota and request < available nodes, satisfy the
   request and continue processing further requests

Since Nodepool consists of multiple launchers, each of which is only
aware of its own configuration, there is no single component of the
system that can determine if a request is permanently unsatisfiable.
In order to avoid requests remaining in the queue indefinitely, each
launcher will register itself at the path::

  /nodepool/launchers/<hostname>-<pid>-<tid>

When a launcher is unable to satisfy a request, it will modify the
request node (while still holding the lock) and add its identifier to
the field `declined_by`.  It should then check the contents of this
field and compare it to the current contents of `/nodepool/launchers`.
If all of the currently on-line launchers are represented in
`declined_by` the request should be marked `failed` in the `state`
field.  The update of the request node will notify Zuul via the
previously set watch, however, it will check the state, and if the
request is not failed or fulfilled, will simply re-set the watch.  The
launcher will then release the lock and, if the request is not yet
failed, other launchers will be able to attempt to process the
request.  When processing the request queue, the launcher should avoid
obtaining the lock on any request it has already declined (though it
should always perform a check for whether the request should be marked
as failed in case the last launcher went off-line shortly after it
declined the request).

Requests should not be marked as failed for transient errors (if a
node destined for a request fails to boot, another node should take
its place).  Only in the case where it is impossible for Nodepool to
satisfy a request should it be marked as failed.  In that case, Zuul
may report job failure as a result.

If at any point Nodepool detects that the ephemeral request node has
been deleted, it should return any allocated nodes to the pool.

Each node should have a record in Zookeeper at the path::

  /nodepool/nodes/456
    type: ubuntu-trusty
    provider: rax
    region: ord
    az: None
    public_ipv4: <IPv4 address>
    private_ipv4: <IPv4 address>
    public_ipv6: <IPv6 address>
    allocated_to: <request id>
    state: building | testing | ready | in-use | used | hold | deleting
    created_time: <unix timestamp>
    updated_time: <unix timestamp>
    image_id: /nodepool/image/ubuntu-trusty/builds/123/provider/rax/images/456
    launcher: <hostname>-<pid>-<tid>

The node should start in the `building` state and if being created in
response to demand, set `allocated_to` to the id of the node request.
While building, Nodepool should hold a lock on the node at::

  /nodepool/nodes/456/lock

Once complete, the metadata should be updated, the state set to
`ready`, and the lock released.  Once all of the nodes in a request
are ready, Nodepool should update the state of the request to
`fulfilled` and release the lock.  Zuul, which will have been notified
of the change by the watch it set, should then obtain the lock on each
node in the request and update its state to 'in-use'.  It should then
delete the request node.

When Zuul is finished with the nodes, it should set their states to
`used` and release their locks.

Nodepool will then decide whether the nodes should be returned to the
pool, rebuilt, or deleted according to the type of node and current
demand.

If any Nodepool or Zuul component fails at any point in this process,
it should be possible to determine this and either recover or at least
avoid leaking nodes.  Nodepool should periodically examine all of the
nodes and look for the following conditions:

* A node allocated to a request that does not exist where the node is
  in the `ready` state for more than a short period of time (e.g., 300
  seconds).  This is a node that was either part of a fulfilled
  request and given to a requestor but the requestor has done nothing
  with it yet, or the request was canceled immediately after being
  fulfilled.

* A node in the `building` or `testing` states without a lock.  This
  means the Nodepool launcher handling that node died; it should be
  deleted.

* A node in the `in-use` state without a lock.  This means the Zuul
  launcher using the node died.

This should allow the main work of nodepool to be performed by
multiple independent launchers, each of which is capable of processing
the request queue and modifying the pool state as represented in
Zookeeper.

The initial implementation will assume only one launcher is running
for each provider in order to avoid complexities involving quota
spanning across launchers, rate limits, and how to prevent request
starvation in the case of multiple launchers for the same provider
where one is handling a very large request.  However, future work may
enable this with more coordination between launchers in zk.

Nodepool should also allow the specification of static inventory of
non-dynamic nodes.  These may be nodes that are running on real
hardware, for instance.

.. _lock construct:
   http://zookeeper.apache.org/doc/trunk/recipes.html#sc_recipes_Locks
.. _priority queue:
   https://zookeeper.apache.org/doc/trunk/recipes.html#sc_recipes_priorityQueues

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
        secrets:
          - logserver-credentials
      workspace: /opt/workspace  # Where to place git repositories
      post-run:
        - archive-logs

Jobs have inheritance, and the above definition provides a base level
of functionality for all jobs.  It sets a default timeout, requests a
single node (of type precise), and requests credentials to upload
logs.  For security, job credentials are not available to be inherited
unless the 'inherit' flag is set to true.  For example, a job to
publish a release may need credentials to upload to a distribution
site -- users should not be able to subclass that job and use its
credentials for another purpose.

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

Jobs may specify that they use other repos in the same tenant, and the
launcher will ensure all of the named repos are in place at the start
of the job::

  ### global_config.yaml (continued)
  - job:
      name: devstack
      parent: base
      repos:
        - openstack/nova
        - openstack/keystone
        - openstack/glance

Jobs may specify that they require more than one node::

  ### global_config.yaml (continued)
  - job:
      name: devstack-multinode
      parent: devstack
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
authorization requirements to a job.  The example above includes only
`secrets`, though other systems may be added in the future.

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
          - tarball
          - wheel
          - pypi-upload:
              dependencies:
               - tarball
               - wheel

Project templates are still supported, and can modify job parameters
in the same way described above.

Before Zuul executes a job, it finalizes the job content and
parameters by incorporating input from the multiple job definitions
which may apply.  The job that will ultimately be run is a job which
inherits from all of the matching job definitions in the order in
which they were encountered in the configuration.  This allows for
increasingly specific job definitions.  For example, a python unit
test job may be defined globally.  A variant of that job (with the
same name) may be specified with an alternate node definition for
"stable" branches.  Further, a project-local job specification may
indicate that job should only run when files in the "tests/" directory
are modified.  The result is that the job will only run when files in
"tests/" are modified, and, if the change is on a stable branch, the
alternate node definition will be used.

Currently unique job names are used to build shared change queues.
Since job names will no longer be unique, shared queues must be
manually constructed by assigning them a name.  Projects with the same
queue name for the same pipeline will have a shared queue.

Job dependencies were previously expressed in a YAML tree form, where
if, in the list of jobs for a project's pipeline, a job appeared as a
dictionary entry within another job, that indicated that the second
job would only run if the first completed successfully.  In Zuul v3,
job dependencies may be expressed as a directed acyclic graph.  That
is to say that a job may depend on more than one job completing
successfully, as long as those dependencies do not create a cycle.
Because a job may appear more than once within a project's pipeline,
it is impractical to express these dependencies in YAML tree from, so
the collection of jobs to run for a given project's pipeline is now a
simple list, however, each job in that list may express its
dependencies on other jobs using the `dependencies` keyword.

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

All of the content of Ansible playbooks is held in the git
repositories that Zuul operates on, and this is true for some of the
Ansible roles as well, though some playbooks will require roles that
are defined outside of this system.  Because the content of roles must
be already present on the host executing a playbook, Zuul will need to
be able to prepare these roles prior to executing a job.  To
facilitate this, job definitions may also specify role dependencies::

  ### global_config.yaml (continued)
  - job:
      name: ansible-nova
      parent: base
      roles:
        - zuul: openstack-infra/infra-roles
        - galaxy: openstack.nova
          name: nova

This would instruct zuul to prepare the execution context with roles
collected from the zuul-managed "infra-roles" repository, as well as
the "openstack.nova" role from Ansible Galaxy.  An optional "name"
attribute will cause the role will to be placed in a directory with
that name so that the role may be referenced by it.  When constructing
a job using inheritance, roles for the child job will extend the list
of roles from the parent job (this is intended to make it simple to
ensure that all jobs have a basic set of roles available).

If a job references a role in a Zuul-managed repo, the usual
dependency processing will apply (so that jobs can run with un-merged
changes in other repositories).

A Zuul repository might be a bare single-role repository (e.g.,
ansible-role-puppet), or it might be a repository which contains
multiple roles (e.g., infra-roles, or even project-config).  Zuul
should detect these cases and handle them accordingly.

* If a repository appears to be a bare role (has tasks/, vars/,
  etc. directories at the root of the repo), the directory containing
  the repo checkout (which should otherwise be empty) should be added
  to the roles_path Ansible configuration value.
* If a repository has a roles/ directory at the root, the roles/
  directory within the repo should be added to roles_path.
* Otherwise, the root of the repository should be added to the roles
  path (under the assumption that individual directories in the
  repository are roles).

In the future, Zuul may support reading Ansible requirements.yaml
files to determine roles needed for jobs.

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
