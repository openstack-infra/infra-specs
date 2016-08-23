::

  Copyright 2016 Thierry Carrez

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

============================
A Task Tracker for OpenStack
============================

https://storyboard.openstack.org/#!/story/2000610

A task tracker is an essential tool in any community toolbelt, especially
global/virtual ones like OpenStack. It allows to create work items, distribute
them across teams and specific assignees, and track their completion without
using inferior synchronization mechanisms like email or IRC pings. OpenStack
currently suffers from not having a standard solution for that.

Problem Description
===================

Some definitions
----------------

Bug trackers (or issue trackers) are used in communities where there is
a strong disconnect between the end user population and the developer
population. End users report issues using the software by describing
symptoms they experience. Bug triagers look into the reported issues,
try to reproduce them, then turn them into a series of tasks (or work
items) assigned to various teams for resolution. A bug tracker is mostly
meant to communicate with users and facilitate bug reporting. Bugs have
status like "Unconfirmed", "Confirmed", "Fix in progress", "Fix available".
Bug trackers are also sometimes used as a way to communicate desired new
features, although separate tools are sometimes used to that effect
(wishlist trackers).

Task trackers are used to organize work within a set of teams. They let you
create work items, group them in various ways in order to visualize the work
to be done, and track the completion of work items and groups of related work
items ("stories"). Task trackers have statuses like "Unassigned", "Assigned",
"Under review", "Merged/Completed". Bug trackers often double as task
trackers: a bug is a story that first needs to be triaged into tasks. Doing
so usually results in a suboptimal experience for the tool users. The key
difference is that end users report symptoms, which do not relate to any
particular section of code. They need to be turned into tasks attached to
specific code areas and teams. Triaging being a tedious task, end user bug
reports feel ignored while developers file their own tasks to track their
work and those see fast progress.

Release publishing tools are used to catalog available releases and point to
release notes.

Change tracking tools keep track of what each version actually contains in
terms of features, bugfixes and vulnerability fixes.

The past
--------

Historically OpenStack has been using Launchpad for task tracking, bug
tracking, release publication and change tracking. It was a convenient
one-stop shop for all our tracking needs. However as OpenStack grew we
ran into a number of Launchpad limitations, in particular:

* No support for tracking features that span multiple projects

* Performance/usability issues in stories with lots of tasks

* Limited release publication features (in particular, no way to show
  a complete OpenStack release)

* Release publication was linked to using Launchpad for change tracking,
  which created inaccuracies as Launchpad's view differed from git contents

* No support for our own OpenID provider, requiring OpenStack users to create
  multiple user/passwords

* Partial API feature coverage making it hard to workaround limitations

* Launchpad is difficult to run as a standalone instance, so we can't really
  fork it if we need some specific feature added/removed

We explored creating our own tool, StoryBoard, to better align with our
specific workflows. The infra team served as guinea pig in dogfooding
StoryBoard. But without clear commitment that this was chosen as the way
forward, StoryBoard development stalled, leaving us to explore other
tools (Maniphest) to cover our task tracking needs.

Today
-----

Today the OpenStack community is left with a mix of tools. Some use Launchpad,
some use StoryBoard, some use some homegrown solution, some use proprietary
tools, and a lot of people just abandoned the idea of task tracking and abuse
email and IRC synchronization to compensate. We need to provide our community
a solution as soon as possible.

It is also important that we standardize on one solution. You can't track any
cross-project work if the different teams involved use different tracking
solutions. At best you'll track work in a place that the other team is
ignoring. More likely you'll just abandon the idea of tracking cross-project
work (or the idea of doing cross-project work).

There are a number of recent changes that factor in the choice of a solution:

* The release team developed its own tooling for release publication (the
  releases.openstack.org website) so we no longer need to have that featureset
  in the future tool

* We doubled down on using git for change tracking, by producing release notes
  in-tree (using reno), so the chosen tool should definitely not include
  invasive change tracking features

* StoryBoard development is now more active as the tool is being used outside
  of OpenStack

* Canonical recently pushed new resources to Launchpad development (which was
  mostly stalled 2 years ago), although arguably those resources are dedicated
  to improving build tooling for the Ubuntu platform as it moves to using new
  tools (snap format, git repositories...)

Options
-------

1. The first option is to abandon the idea of changing, and re-standardize on
   Launchpad. Benefits: solution maturity, familiarity, limited work to do.
   Drawbacks: All the reasons that pushed us away from it a few years ago
   still stand. Additionally, Launchpad's all-in-one approach is becoming a
   new problem as the release team decoupled release publication and change
   tracking tools. Combined with our inability to fully customize our own
   Launchpad deployment that makes for a pretty unlikely option.

2. The second option is to migrate everyone to StoryBoard. Benefits:
   purpose-built tool aligned with OpenStack needs, API-first and static
   webclient design, coded in Python in the OpenStack Way. Drawbacks: still
   not mature, inferior UX (compared to other available solutions), missing
   key features (like proper ACLs to handle vulnerability work), requires a
   lot of maintenance/development investment from the OpenStack community.

3. The third option is to migrate everyone to Maniphest (Phabricator's
   task tracking component). Benefits: excellent UX, lots of features, active
   external maintenance. Drawbacks: partial API coverage (API-last design),
   model mismatch (no concept of stories affecting multiple projects) requiring
   us to jump through some hoops to emulate it using subtasks and supertasks,
   all-in-one approach bleeding into our wish to only use a single component,
   opinionated design upstream which makes it difficult to upstream changes
   to support our specific workflows, usage of PHP making it costly to maintain
   a fork.

Proposed Change
===============

There is no easy choice here. If there was we would have made a final decision
on this a long time ago. One way to make the decision is to think about where
we ultimately want to be, and pick the solution that can lead us there.

We ultimately want to be using a tool that can meet our needs of today and
tomorrow. We want a tool with full API coverage so that we can easily extend
or supplement it with user-specific tooling.

The proposed change is therefore to migrate everyone to StoryBoard.

Issues with proposed solution
-----------------------------

While StoryBoard is the only solution on the path to our ideal solution, it
is not very far along that path. That means we'll have to live with its
limitations, including missing features and various UX glitches.

This may trigger hate and pain rather than the expected surge in participation
to its development.

In order to reduce that, a number of critical issues are listed in this spec.
Those need to be resolved before we launch the migration operation.


Implementation
==============

Phase 1: Identify and fix remaining gaps
----------------------------------------

During this phase, we'd identify all the features that need to be implemented
and all the bugs that need to be fixed (including UX glitches) before we can
migrate everyone to StoryBoard. The feature gap analysis should continue no
longer than the remainder of the Newton development cycle in OpenStack, with
followup discussion/finalization of the list at the Ocata summit if needed (but
completing it earlier is encouraged).

This list should be built by engaging with various key users and making sure
they can migrate from using Launchpad to using StoryBoard with minimal
disruption. It should be conservative: must-have rather than should-have.

This list should then be turned into a milestone-based implementation plan,
with an estimation of when general migration could happen. As a mostly
autonomous development team with its own non-OpenStack constituency, the
StoryBoard team will have control over prioritization and final say on whether
some requested features are not suitable to implement at all. Whether the plan
is sufficient for a migration of the OpenStack community should be determined
by the OpenStack Technical Committee.

While phase 1 is going, we should actively on-board new volunteer teams
(beyond Infra) which feel ready to use StoryBoard in its current state. This
will hopefully bring more people to contribute to StoryBoard as users scratch
their own itches, creating a virtuous circle.

Phase 2: General migration
--------------------------

Once phase 1 is completed, we should migrate all remaining Launchpad users in
one shot. Migration will be scheduled not less than 1 month following the
technical committee's agreement to migrate. 

Assignee(s)
-----------

During phase 1 we need a volunteer to facilitate the discussion between the
StoryBoard team and the rest of the users, and help prioritize the must-have
items. The StoryBoard team would work on implementing the missing features and
fixing the blocker issues.

Phase 2 is mostly led by the infrastructure team and will be implemented as a
separate, followup spec.

Gerrit Topic
------------

Use Gerrit topic "storyboard-migration" for all patches related to this spec.

.. code-block:: bash

    git-review -t storyboard-migration

Work Items
----------

Phase 1:

1. Facilitator identifies feature stakeholders from the OpenStack community

2. Feature stakeholders identify remaining needed features

3. StoryBoard team and facilitator prioritize feature requests

4. Technical Committee are asked to confirm the proposed plan sufficiently
   addresses blockers for migration

5. StoryBoard team (hopefully with new developer assistance from the extended
   OpenStack community) implement identified features

Repositories
------------

No new git repositories should need to be created.

Servers
-------

No new servers should need to be created.

DNS Entries
-----------

No new DNS entries should need to be created.

Documentation
-------------

This will require updating a lot of documentation. One of the phase 1 items
would be to identify and update the developer and project creator docs where
needed.

Security
--------

Beyond due diligence on the security design of the API server, we need
specific features to support the Vulnerability Management Team workflow in
a way that ensures confidentiality of the submitted vulnerabilities. Special
care and tests should be applied to reduce the likelihood of leaks in that
sensitive area.

Testing
-------

No specific tests (beyond normal testing of added features) should be
necessary in support of this spec.

Dependencies
============

Before migrating new users, StoryBoard should be plugged into an OpenStack
identity provider (OpenStackID, ipsilon...) and therefore this work should
be completed first.
