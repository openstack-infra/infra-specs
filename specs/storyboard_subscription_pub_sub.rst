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

========================
Subscriptions and Events
========================

https://storyboard.openstack.org/#!/story/96

StoryBoard needs to notify a user when changes occur to a resource which
they have decided to be notified about.

A key feature needed by all ticket tracking systems is the ability to
notify a user when topics which they care about have changed. A common way
to describe this is "Subscription", where a single user will ask to be
notified about certain types of changes for certain resources. More
complicated implementations include filtering, notification alert levels,
summaries of events that all impact the same resource,
or automatic notification based on inferred or calculated relevance.

Problem description
===================

In its simplest form, we would like each user to be able to indicate which
resources they are interested in, and be able to retrieve a date-sorted
list of which of those resources have changed recently. This will then be
shown in the UI as a list of events that occurred that are relevant to the
user. More advanced features would be the ability to filter these events,
or to receive notification of new events in "real time".

Requirements are as follows:
* A user should be able to manage their subscriptions
* Subscriptions should be as up-to-date as possible with as little data
loss as possible.
* A user should be able to subscribe to tasks, stories, projects,
and project groups.
* Subscriptions should have a minimal impact on API performance.
* When a subscription is added, all changes from that point forward should
be reported. Historical changes do not need to be generated.
* When a subscription is removed, a user's subscription list does not have
to be recalculated to extract no-longer-relevant events.
* StoryBoard should be able to run on a single server with no crazy
additional infrastructure required.
* Oslo libraries should be taken into account.

Subscriptions are the classic, personalized, pub-sub problem, where a user's
list of subscriptions can be large, and the matrix of events that could
cause a subscription to be notified can be complex and processor intensive.
Classically, there are four ways to handle this problem: Push, Pull,
Async, or JIT.

Push
----
The push approach assumes that you will notify all subscribers when the
event occurs. In the case of our API, this means that during a
PUT/POST/DELETE, all subscribers to that resource are checked to see
whether they need to be notified. In most small-scale system this works
fine, however as subscriptions increase this approach is generally not
stable, as the time required to process the request (and the number of
errors that could possibly be raised) will start to impact the client
making the API request. Things that would raise concerns are timeout,
as well as the number of points of failure, and as a result of that this is
not an appropriate approach.

Pull
----
The pull approach believes in restricting the processing load described
above to only those users who care enough to ask for subscription events.
In this case, a list is generated fresh every time GET /subscriptions (or
similar) is called. This approach is appropriate when usage is expected to
be low, similar to the generation of a report. Given that a subscription
list is a fairly frequently polled resource, this is not an appropriate
approach.

Async
-----
An asynchronous approach makes use of deferred, distributed processing to
"eventually" update a user's list of subscription events. Worker management
systems such as Gearman or Rabbit are notified whenever a resource changes
(likely via Pecan API hooks), and they 'eventually' ask a worker process to
go figure out which subscriptions need to be updated. Advantages of this
approach is that we can avoid the Python Global Interpreter Lock by having
separate worker processes, and any errors encountered during subscription
processing will be isolated and thus not impact the actual API request. The
challenge with this is that most queueing/worker management systems are
resource intensive (kafka), do not guarantee delivery (gearman),
or have known issues with split-brain clustering (rabbit). Any approach
will have to accommodate the chosen system.

Streaming
---------
A streaming approach begins by emitting an event whenever a resource
changes, and to notify all subscribers that are currently connected via a
socket. Persistence of events may be handled by creating
individual processes that listen to the stream and persist the received
data much like a subscribed client might update its UI. This approach
solves the real-time problem with a hot sexy technology, however coordinating
listeners and ensuring that persistence is handled properly raises the same
problems as the Async or Push/Pull problem. As a result this approach is
unnecessarily more complex than it needs to be.


Proposed change
===============

Summary
-------
StoryBoard will emit events whenever a resource changes. Since most
resources map directly to database changes, the majority of these changes can
be handled via Pecan post-request hooks.

Events, when emitted, will be written to a deferred processing queue. If
the queue is unavailable or misconfigured, a warning should be written to
the log, however the system should complete the original request normally
and discard the change event altogether.

The queuing system to be used is RabbitMQ, because it guarantees delivery
and recovers after a crash. This decision is based on the assumption that
the number of events dispatched by the database will not be sufficient to
require a full RabbitMQ cluster, which means we don't have to worry about
split-brain problems.

The StoryBoard server will spin up a series of processes that listen to the
event queue and perform actions based on the type of event received. In the
case of subscriptions, a process would read the event,
load the impacted resource and its change, search for any subscriptions to
the impacted resource, update each subscription's owners' subscription feed,
and then notify rabbit that the message has been received and processed. If
updating one subscription fails, the process should still attempt to
complete the other ones.

The number of events that are retained per user should be configurable by
age, with a default of 1 month.

A user's event feed should be retained in the database in its own table.

API: Subscriptions
------------------
StoryBoard will expose a new endpoint at /v1/users/ID/subscriptions. This
endpoint will support basic CRUD operations by which a user can manage
their subscriptions.

API: Feed
---------
StoryBoard will expose a new endpoint at /v1/users/ID/feed. This endpoint
will provide a list of events that have been emitted by a user's
subscriptions, sorted by date in descending order.

Alternatives
------------
Alternative approaches have been listed in the problem description.
Alternative queueing systems include ZeroMQ & gearman, which was disqualified
because it does not guarantee delivery, Kafka which was disqualified
because (anecdotally) it requires a cluster to perform properly.

Implementation
==============

Assignee(s)
-----------

Primary Assignee:
    TBD

Work Items
----------
* Create an API to add subscriptions for projects, project groups, stories,
  and tasks.
* Teach the storyboard-webclient to allow subscription on projects, project
  groups, stories, and tasks.
* Install RabbitMQ on StoryBoard Server.
* Use Oslo.messaging to create an SQLAlchemy hook that broadcasts change
  events for project groups, projects, stories, and tasks.
* Add configuration to StoryBoard for the AMQP connection string and
  optionally an enabling flag for the whole feature.
* Create a storyboard-worker process that connects to AMQP and receives
  messages for processing.
* Create a way for the storyboard-worker process to process lots of
  different kinds of events (event hooks of some sort? processor factory?)
* Build a subscription event handler which is run by storyboard-worker and
  updates a subscriber's feed.
* Create an API endpoint that exposes the feed.
* Teach the storyboard-webclient to display the feed.

Repositories
------------
No new repositories.

Servers
-------
No new servers. storyboard.openstack.org will need to have a running
RabbitMQ instance.

DNS Entries
-----------
No new DNS entries.

Dependencies
============
See above. Puppet module for storyboard will need to be updated. Additional
dependencies are on oslo.messaging, rabbitmq-server, upstart, etc.
