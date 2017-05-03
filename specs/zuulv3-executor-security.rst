::

  Copyright (c) 2017 IBM

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Zuul v3 Executor Security
=========================

Storyboard: https://storyboard.openstack.org/#!/story/2000910

Playbooks provided in project repos are already run with a set of
Ansible plugins to protect the executor from compromise or information
leaks. While this belt is keeping our security pants on, we definitely
don't want them to fall down if the belt fails, so we need suspenders
in the form of OS level containment.

The goals of this effort as as follows:


* Define simple, automated ways for Zuul to protect its own executor.
* Provide operators with guidance on Executor security measures.
* Keep zuul simple.

Note that we will not discuss any methods to mitigate resource exhaustion
outside the executor, such as filling up Swift with artifacts, using
nodes for purposes outside the ToS agreed upon the by Zuul operator, etc.

Problem Description
===================

If a bug in Ansible or our Ansible plugins allows users to break out of
the insecure context, the executor will currently be vulnerable to several
known attack vectors.

Local Privilege Escalation (LPE)
--------------------------------

The executor runs as an unprivileged daemon user. It will run
`ansible-playbook` with those same privileges. While administrators
should lock this user down to the minimum amount of access required to
launch jobs, `Linux` and other operating systems which might run Zuul
are not immune from privilege escalation vulnerabilities.

Critical Information Leaks (CIL)
--------------------------------

Systems which are not generally secured against local users may provide
helpful information to malicious actors. This includes simple things
like operating system kernel versions, networking configuration, and more
critical information like files containing secrets that are accidentally
exposed by incorrect local file permissions.

Denial of Service (DoS)
-----------------------

A bad actor that breaks out of Ansible protections may be able to do
some very small things to consume all of the resources of the executor.

Proposed Change
===============

Execution Flow
--------------

Currently the executor functions like this, with untrusted context being
secured by Ansible plugins. Any of the playbooks may be run in a trusted
or untrusted context depending on whether or not they were defined in
a project repo or in a config repo.

 1. Make a writable job dir
 2. Copy merged git repos to job dir
 3. Run pre playbooks
 4. Run in-repo playbooks
 5. Run post playbooks
 6. Nuke job dir

Two possible revisions to this are "Secure Execution on Executor" and
"Secure Execution on Node":

Secure Execution on Executor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can rework the untrusted context by wrapping it in containment methods,
outlined below. If we choose containment methods that require an image
we'll add two steps before step (1) above:

 -1. Build image for chroot periodically.
 0. Copy image to job dir
 1. Make a writable job dir
 2. Copy merged git repos to job dir
 3. Run pre playbooks
 4. Run in-repo playbooks
 5. Run post playbooks
 6. Nuke job dir

Step (-1) above could be done out of band with the executor using
`diskimage-builder`. It's possible `nodepool-builder` could be used here,
but for an initial implementation, I believe this can simply be done on
a cron once a day and configured as a path to a template image directory
or tarball. Any CoW system is an implementation/optimization detail to
speed up the copy step and reduce disk footprint.

A more practical plan is to skip (-1) and (0) and just bind mount /usr
into the working directory, which is already the default method used by
Bubblewrap. This means that a contained attacker has access to all the
tools from the executor host, but in terms of real security, depriving
them of gcc, while we allow running python and contacting the internet,
is only a very minor hurdle for an attacker to get over. We may want
to advise deployers to keep the executor host software footprint to a
minimum as a result.

Diskspace monitoring
--------------------

Because playbooks will need to transfer artifacts around, we will
need to monitor artifact space usage by playbooks. While usage of an
object storage service like Swift is also an option, there will always
be some percentage of space needed on the executor for playbooks to use
as scratch space, and we don't want to require object storage services
for effective use of Zuul.

A simple method will be to have a single process/thread which walks
playbook artifact storage with `du` periodically. Any job that has
exceeded its space allocation will be terminated immediately and have
its artifact space emptied. A very fast consumer of space will be able
to fill a disk before this can be done, so the limit should be relatively
low in comparison to the size of the storage area.

A config option will be created to define the per-job disk space limit
for all jobs. This should simplify the initial implementation, but later
on it may be necessary to define per-job space limitations.

Evaluation of methods of containment will assume that this change precedes
or accompanies any implementation.

Access Credentials
------------------

We need to grant `ansible-playbook` the ability to access test nodes.
Since our only allowed Ansible connection method is SSH, we can
narrow this to SSH key access. Ideally we can allow the untrusted
`ansible-playbook` to use an SSH key to access test nodes without exposing
key material.

SSH Agent
~~~~~~~~~

The executor already is configured for a path to an SSH private key file.
This file could be added into the contained chroot, but that would expose
the private key material to the untrusted playbook, which would allow
said malicious actor to log the key and use it to access other test
nodes as long as that SSH key is used.

Instead we can use `ssh-agent` and expose the socket to the contained
`ansible-playbook`. Because `ssh-agent` only signs challenges, it will
mean that a malicious user will have to be able to do more than just log
the private key to make use of it, and their access to the key will end
when their access to `ssh-agent` ends.

This will require making sure the socket is visible inside containment,
and passing in the environment necessary to help `ssh` find it.

Available Containment Methods
-----------------------------

There are a number of different options available to address executor
security.

Some known methods are listed below with general background information,
including a list of pros and cons for each.

Many of these can be combined, some cannot. It seems likely that the end
solution will have us adopting at least 2. We may also need to add in a
layer of abstraction to Zuul to allow users to write their own security
integrations based on their knowledge and abilities, but that is beyond
the scope of this document.

ulimit
~~~~~~

This limits what resources a user-space process can consume.

LPE
***

No coverage.

CIL
***

No coverage.

DoS
***

 * Can prevent exhaustion of user-space memory

 * Can prevent direct exhaustion of process space

 * Still vulnerable to exhaustion of kernel structures and I/O

Pros
****

 * Simple implementation

 * No filesystem changes needed

 * Built-in to all operating systems.

 * No performance overhead

Cons
****

 * Only covers a few DoS vectors and nothing else

Chroot
~~~~~~

This would involve building a directory with only the binaries needed
to run playbooks, source trees bind mounted or copied in, and writable
space for artifacts.

Special care would be taken to ensure the binary paths were readonly
and any writable paths are mounted noexec.

LPE
***

 * Mitigates due to removal of most binaries [binaries]_

 * Mitigates due to removal of access to directories outside chroot.

 * Vulnerable to kernel problems which allow chroot breakout or
   privilege escalation via Python.

CIL
***

 * Mitigates due to removal of most binaries [binaries]_

 * Mitigates due to removal of access to directories outside chroot.

 * Still vulnerable to any kernel<->user space interaction which Python
   can do natively.

 .. [binaries] This mitigation is complicated by the fact that an attacker
     could build binaries on a test node and transfer it back as an
     artifact. Getting permissions and noexec parts right would
     be key.

DoS
***

 * No significant improvement.

Pros
****

 * Simple, built-in to most operating systems

 * Well understood, can be fully achieved by unprivileged user.

Cons
****

 * Incomplete coverage

 * Known attack vectors

 * Requires building chroot filesystem carefully.

Cgroups
~~~~~~~

Cgroups allow one to limit a set of processes' access to various kernel
subsystems, and to identify them as a group.

Various helpers exist for them, and those will be evaluated separately to
the fundamental cgroup capability.

The implementation would be to create a cgroup for each ansible-playbook execution,
with the administrator being able to decide the template for that cgroup.

LPE
***

 * Mitigates somewhat by restricting access to some kernel subsystems.

CIL
***

 * Mitigates somewhat by restricting access to some kernel subsystems.

DoS
***

 * Significant mitigation due to limitations on all kernel subsystems.

 * Provides convenient way to integrate with `du` process as any detected
   overrun of disk space can have its cgroup 'frozen' stopping all
   processes in the cgroup.

 * Controls "noisy neighbor" by guaranteeing even consumption of CPU and IO.

Pros
****

 * Relatively simple to create and modify cgroups

Cons
****

 * Direct cgroup manipulation requires root privileges or setuid helper

Seccomp
~~~~~~~

Seccomp is a system by which a process may restrict what syscalls it,
and any of its children, may make. It is a relatively straightforward
process to consider what syscalls Ansible would need to make, since its
primary functions are local file CRUD, and network operations.

LPE
***

 * Reduces attack surface of the kernel by limiting to the needed syscalls.

 * Reduces ability of python to do real damage beyond what the needed syscalls
   can do.

CIL
***

 * Should reduce surface area again by limiting access to syscalls which leak
   information.

DoS
***

 * Same mitigations as LPE.

Pros
****

 * Well understood, universally available Linux security technology.

 * The syscall-oriented nature means it's likely the set of syscalls
   needed will remain relatively static, reducing maintenance load as new
   versions of Ansible are released.

Cons
****

 * Tooling is a bit obtuse and user-unfriendly.

LXC
~~~

An LXC container is effectively a combination of chroot, cgroup, and
Linux kernel namespaces.

A potential implementation would be to build a chroot filesystem using
diskimage-builder and then launch an LXC container with that as the root
filesystem, and bind mounts for readonly data (git trees) and writable
space (artifacts).

LPE
***

 * Mitigates a bit more than Cgroup+Chroot by preventing crossing user
   namespace boundaries.

CIL
***

 * Mitigates a few more leaks by further partitioning processes access to data
   in the kernel that may belong to other processes.

DoS
***

 * No better than cgroups + chroot.

Pros
****

 * Simpler implementation than Docker

 * Well understood and mature set of technologies

Cons
****

 * Less popular than Docker, risk it being abandoned

 * Single-vendor open source project (Canonical) makes this problematic
   for Zuul deployers on not-Ubuntu/Debian.

 * Still requires careful filesystem and mount crafting.

Docker
~~~~~~

Docker started life as a daemon to control LXC, just like LXC 2.0 is
now. It has grown quite a bit from there and provides all of the same
LPE/CIL/DoS protections as LXC.

In addition to the LXC capabilities, it features a rich set of image
build tools, and a daemon for storing and retrieving those called 'docker
hub'. There is also a centralized internet Docker Hub where users share
their container images.

Pros
****

 * Industry wide attention means support and adoption will be less
   controversial.

 * Includes container storage limits as a feature, possibly mitigating
   the need for the `du` storage monitoring thread, or at least providing
   extra protection against the race condition.

Cons
****

 * A mountain of features which we don't need means it is far more
   complex than needed. The net effect of downtime and confusion for
   operators of Zuul may not be worth the security mitigations.

rkt
~~~

Rkt is aimed at those who do feel that Docker is overkill for containing
things. It mostly sits as an abstraction for containment of things, with
systemd-nspawn and kvm available. It provides all the same LPE/CIO/DoS
protections as LXC.

Pros
****

 * Well thought out design that tries only to do one thing well

Cons
****

 * Single-vendor

 * Unknown how well tested it is

Bubblewrap
~~~~~~~~~~

https://github.com/projectatomic/bubblewrap

Bubblewrap is similar to Docker or LXC, except that it may not require
root privleges to sandbox an application.  It is also aimed specifically
at sandboxing rather than providing image based isolation like LXC and
Docker. It would be used similar to LXC or Docker, and provide around
the same level of mitigation for LPE/CIL/DoS.

Pros
****

 * Small simple command line utility with no privileged daemons necessary.

 * Specifically built for sandboxing partially trusted apps only.

 * Supports Seccomp

Cons
****

 * User space is not included in Ubuntu 16.04 (Backporting is trivial).

 * Kernel on Ubuntu 16.04 is limited, Yakkety backport is required to
   get full set of USER_NS features.

 * The kernel side is relatively new and untested, and has already had
   a few local root exploits found in it.

systemd-nspawn
~~~~~~~~~~~~~~

Similar to bubblewrap, but coming from the systemd project. It does have
some unprivileged capabilities, but I believe for our use case we would
need it to be setuid or run as root.

Its containment capabilities are comparable to Bubblewrap.

Pros
****

 * It can take advantage of Btrfs or LVM for CoW
   snapshots automatically, which is nice for scaling to lots of
   concurrent jobs.

Cons
****

 * Confusing relationship with systemd and machined.

 * Seems focused on running a whole OS rather than an app.

AppArmor
~~~~~~~~

AppArmor is a relatively straight forward kernel security module that
allows defining the behavior of individual binaries. Combined with chroot,
this could be enough to mitigate most vulnerabilities.

LPE
***

 * Mitigates further by reducing surface area in the kernel and userspace

CIL
***

 * Mitigates further by reducing surface area in the kernel and userspace

DoS
***

 * No significant improvement.

Pros
****

 * Extremely Simple profile language adds value without confusing admins
   too much.

Cons
****

 * Not supported on CentOS/Fedora/RHEL

 * Having AppArmor enforcing can complicate things if packages have
   defined AppArmor profiles that do not agree with how the executor
   wants to use those packages.


SELinux
~~~~~~~

SELinux is similar to AppArmor, but can offer more fine-grained control
and thus more complete protection, at the cost of more complexity and
thus a more difficult implementation. It has more or less the same LPE/CIL/DoS
profile as AppArmor.

Pros
****

 * Extremely powerful tools allow extremely fine-grained control

 * Specifically limits chroot and/or container breakouts with the
   combination of process contexts and MCS (Multi-Category-Security)

Cons
****

 * Having SELinux enforcing means the whole executor system must have its SELinux
   configuration fully defined.

Recommendation
--------------

Based on the surface level evaluations, I believe Bubblewrap has the
highest value for the lowest complexity. We can use it with the /usr
from the executor bind mounted into the chroot, which is slightly less
secure than managing our own overlays and images since we may end up with
dangerous setuid binaries accessible to users. We are already building
working directories for jobs so putting a chroot in there doesn't seem
like too far of a departure.

Bubblewrap can be used via setuid on Ubuntu 16.04 (via backports)
without upgrading to a Yakkety kernel. It allows us to get a ton of
containment without sacrificing much in the way of complexity. We can
combine it with cgroups later to increase DoS protection once we have
it containing the process. We can also add SELinux support fairly easily
once this is known to work. Finally we can layer on seccomp and reduce
surface area even further.

Building images for the chroot with minimal binaries would reduce surface
area further, but this can be deferred until we have full container/COE
support for testing nodes. This way we can keep image building where it
is now, in Nodepool.

Alternatives
------------

Secure Execution on a Test Node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Alternatively, we could rely on Ansible in the node, and keep the flow
as-is, but make the untrusted context mean "inside a node". In order to
do that we would need to make one of the nodes an "untrusted executor"
(simplest answer on which one to use is the first one in the node set).
This would involve the following changes:

 * Build custom inventory

   * An inventory would need to have the untrusted executor setup
     specially so that it uses ansible_connection=local, or it would
     need to be able to SSH to itself.

 * Create and distribute creds

   * The untrusted executor would need an ephemeral private SSH key,
     and all other nodes in the nodeset would need this key installed.

 * Network Access

   * Currently we verify that nodepool -> nodes works, and assume executor
     -> nodes is equivalent.  But this would require that we be able to
     SSH from node to node, which may not always be possible. We also
     likely will want to make sure inventories have the private IP.

 * Ansible setup on untrusted executor

   * We currently don't put any restrictions on nodes other than the
     ability to SSH into them. We'd need to install ansible somehow,
     possibly in a chroot to keep it isolated from the user's test
     execution and dependencies. Isolating Ansible in this way should
     be quite a bit simpler than isolating Ansible in a security context
     though.

Pros
****

 * Same containment for executor as tests mean we could probably
   just drop the Ansible plugins.

 * Executor scales with test nodes

Cons
****

 * Ansible must be injected or present in all test nodes.

   * Injection is brittle, requiring extra download and build steps that
     add failure risk to test runs, potentially wasting resources.

   * Requiring Ansible to be present is a burden for those who want to
     take advantage of the fact that Zuul and nodepool allow custom images.

   * Ansible's requirements are non-trivial, so if we can't spare more
     test nodes for an executor-specific Ansible, at the very least
     we would need to inject a virtualenv or chroot to run Ansible in,
     contaminating the test nodes' environment.

 * Resources normally allocated to running tests will be consumed by
   executor, or nodes will need to be allocated to running playbooks only.

Ultimately, this method is rejected for both of the Cons above. The
Ansible plugin should provide medium level security, and a healthy dose
of namespaces, cgroups, and chroot should keep any breakouts contained.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * SpamapS


Work Items
----------

* Request backport of bubblewrap userspace from latest Ubuntu stable to
  xenial-backports.
* Create ansible minimal chroot image.
* Add chroot-copy into job dir before insecure contexts.
* Add code to call ansible-playbook via `bwrap` in the insecure context.

Repositories
------------

openstack-infra/zuul (feature/zuulv3)

Servers
-------

N/A

DNS Entries
-----------

N/A

Documentation
-------------

We will need to write heavy documentation outlining not only how to setup
a executor, but what risks are still present.

Security
--------

This spec is entirely focused on enhancing the process for securing Zuul v3.

Testing
-------

Integration tests will need to be configured with the mitigation technologies
we implement.

Dependencies
============

zuulv3
