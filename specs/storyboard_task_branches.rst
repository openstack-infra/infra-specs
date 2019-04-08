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

===============================
StoryBoard support for branches
===============================

StoryBoard needs to support branch-specific tasks in order to be usable for
OpenStack bug tracking.


Problem description
===================

Rationale
---------

StoryBoard is a task tracker, and its main use case is the ability
to track resolution of a specific issue (story) across multiple teams.

Most OpenStack projects maintain stable/* branches. A few of them use feature
branches. As we near release, projects make use of temporary proposed/*
branches. All bugs need to be fixed in the master branch, but some of them
will also need to be fixed in other branches, and that involves the work of a
lot of separate subteams. So we need the ability to track tasks across
branches (bugfix or vulnerability fix backports) as much as we need to
track tasks across projects.


Historical usage
----------------

Launchpad supports series-specific tasks, which can be used to emulate branch
support. A series in Launchpad is like a release cycle ("juno", "kilo") which
contains multiple release "milestones". By default in Launchpad a task applies
to the series under development, but you can create series-specific tasks
to target past series. An example of that can be seen at:

https://bugs.launchpad.net/keystone/+bug/1354208

That said, Launchpad implementation becomes very confusing because you can
also explicitly create a task targeting the current series, in which case
the main task is disabled with a confusing "status tracked in..." message.


Proposed change
===============

Project names and git repository links
--------------------------------------

Currently the project names are used for two different things. They represent
the project name in StoryBoard views and dropdowns, and they specify the name
of the git repository that is associated with that project.

This dual meaning results in confusion when StoryBoard is used to support
projects that are not backed by a code repository, in a general waste of
UI space (due to the mention of the GitHub organization in the project name),
and in the inability to have projects in different git servers (all projects
are assumed to come from git.openstack.org).

Since for branch support we'll need to be able to query git repositories for
their branches, this sounds like the right moment to specify the git
repository location separately from the project name.

Task branches
-------------

Projects in StoryBoard may define multiple task branches. Each project has
at least one task branch (called "master"). If a project has only one task
branch, the UI should optimize and ignore that concept altogether, to
simplify the views.

If a project is linked to a git repository, task branches may be created
automatically from branches defined in that git repository (using for example
a daily cron job). Each project may opt to have task branches autocreated
from its code repository branches. This should not prevent them from manually
creating an additional task branch.

Task branches should be able to expire: they would still show in past tasks,
but you would not be able to select them for new tasks. Deleted branches
would automatically result in expired task branches.

Note that the task branch concept degrades gracefully for non-code projects,
which can opt to manually define several areas of work as task branches, or
ignore the concept altogether and use the default branch.


Implementation
==============

Implementation details should be left to the person that would do the
implementation work. What follows is just a suggestion.

The Project table would have two new columns:

* 'code_repo', which would hold a URL describing the code repository,
  like https://git.openstack.org/openstack/nova

* 'autocreate_branches', which would tell StoryBoard to try to create task
  branches automatically from the branches declared on that code repository

A new Branch table would be created, with the following columns:

* 'project_id', mapping to the owner project

* 'branch_name', with the name of the task branch

* 'creation_date', date of creation

* 'expired', a binary flag that marks branches that should no longer be
  selectable in tasks

* 'expiration_date', last date the expired flag was switched to True

* 'autocreated', a flag that marks autocreated entries, so that they can
  be auto-expired when the corresponding branch is deleted in the git repo

By default, all projects would start with a single "master" branch.

The Task table would have one new column:

* 'task_branch', which would point to the affected branch in the project.
  Defaults to the 'master' branch for that project.

The UI would show the 'branch' only if multiple different branches are
defined in the project, saving valuable horizontal space for projects that
don't have a use for this concept.

Assignee(s)
-----------
Primary Assignee:
    TBD

Work Items
----------
TBD

Repositories
------------
No new repositories.

Servers
-------
No new servers.

DNS Entries
-----------
No new DNS entries.

Documentation
-------------
TBD

Security
--------
Manual branch creation should be restricted to an admin set of users, to
avoid accidental or spurious creation of project branches.

Testing
-------
TBD


Dependencies
============

This proposal is not dependent on any other, but it is a prerequisite for
task milestone or story type support.
