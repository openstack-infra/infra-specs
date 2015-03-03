::

  Copyright (c) 2014 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

==================================
Nodepool launch and delete workers
==================================

Story: https://storyboard.openstack.org/#!/story/2000075

Split the node launch and delete operations into separate workers for
scalability and flexibility.

Problem Description
===================

When nodepool launches or deletes a node, it creates a thread for the
operation.  As nodepool scales up the number of nodes it manages, it
may have a very large number of concurrent threads.  To launch 1,000
nodes would consume an additional 1,000 threads.  Much of this time is
spent waiting (sleeping or performing network I/O outside of the
global interpreter lock), so despite Python's threading limitations,
this is generally not a significant performance problem.

However, recently we have seen that seemingly small amounts of
additional computation can starve important threads in nodepool, such
as the main loop or the gearman I/O threads.  It would be better if we
could limit the impact of thread contention on critical paths of the
program while still preserving our ability to launch >1,000 nodes at
once.

Proposed Change
===============

Create a new worker (independent process which may run on either the
main nodepool host or one or more new servers) which performs node
launch and delete taks called 'nodepool-launcher'.  All of the
interaction with providers related to launching and deleting servers
(including ip allocation, initial ssh sessions, etc) will be done with
this worker.

The nodepool-launcher worker would read a configuration file with the
same syntax as the main nodepool server in order to obtain cloud
credentials.  The worker should be told which providers it should
handle via command-line arguments (the default should be all
providers).

It will register functions with gearman in the form
"node-launch:<provider>" and "node-delete:<provider>" for each of the
providers it handles.  Generally a single worker should expect to have
exclusive control of a given provider, as that allows the rate
limiting performed by the provider manager to be effective.  Though
there should be no technical limitation that enforces this, just a
recommendation to the operator to avoid having more than one
nodepool-launcher working with any given provider.

The worker will launch threads for each of the jobs in much the same
way that the current nodepool server does.  The worker may handle as
many simultaneous jobs as desired.  This may be unlimited as it is
currently, or it could be a configurable limit so that, say, it does
not have more than 100 simultaneous server launches running.  It is
not expected that the launcher would suffer the same starvation issues
that we have seen in the main nodepool server (due to its more limited
functionality), but if it does, this control could be used to mitigate
it.

The main nodepool server will then largely consist of the main loop
and associated actions.  Anywhere that it currently spawns a thread to
launch or delete a node should be converted into a gearman function
call to launch or delete.  The main loop will still create the
database entry for the initial node launch (so that its calulations
may proceed as they do now) and should simply pass the node id as an
argument to the launch gearman function.  Similarly, it should mark
nodes as deleted when the ZMQ message arrives, and then submit a
delete function call with the node id.

The main loop currently keeps track of ongoing delete threads so that
the periodic cleanup task does not launch more than one.  Similarly
with this change it should keep track of delete *jobs* and not launch
more than one simultaneously.  It should additionally keep track of
launch jobs, and if the launch is unsuccessful (or the worker
disconnects -- this also returns WORK_FAIL) it should mark the node
for deletion and launch a delete thread.  This will maintain the
current behavior where if nodepool is stopped (in this case, a
nodepool launch-worker is stopped), building nodes are deleted rather
than being orphaned.

Alternatives
------------

Nodepool could be made into a more single-threaded application,
however, we would need to devise a state machine for all of the points
at which we wait for something to complete during the launch cycle,
and they are quite numerous and changing all the time.  This would
seem to be very complex whereas threading is actually an ideal
paradigm.

Implementation
==============

Assignee(s)
-----------

Primary assignee: unknown

Work Items
----------

* Create nodepool-launcher class and command
* Change main server to launch gearman jobs instead of threads
* Stress test

Repositories
------------

This affects nodepool and system-config.

Servers
-------

No new servers are required, but are optional.  Initial implementation
should be colocated on the current nodepool server (it has
underutilized virtual CPUs).

DNS Entries
-----------

None.

Documentation
-------------

The infra/system-config nodepool documentation should be updated to
describe the new system.

Security
--------

The gearman protocol is cleartext and unauthenticated.  IP based
access control is currently used, and certificate support along with
authentication is planned and work is in progress.  No sensitive
information will be sent over the wire (workers will read cloud
provider credentials from a local file).

Testing
-------

This should be testable privately and locally before deployment in
production.

Dependencies
============

None.

Similar in spirit, but does not require https://review.openstack.org/#/c/127673/
