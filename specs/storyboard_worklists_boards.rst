::

  Copyright 2015 Codethink Limited.

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
StoryBoard Worklists and Boards
===============================

https://storyboard.openstack.org/#!/story/2000322

StoryBoard's current model for priority has each task have a discrete
priority value. However, that isn't ideal as everyone has different
priorities for things and there is no way to represent that in the
current model. Instead we want to allow users to create worklists,
which they can then use to create prioritised lists relevant to
themselves.

Problem Description
===================

A developer, a PTL, and the community as a whole may all have different
opinions on the priority of tasks or stories in StoryBoard. Currently, a
task can only have a single discrete priority, which could lead to issues
if people's priorities come into conflict.

Using "worklists" - lists of tasks and/or stories - allows for complex
prioritisation of work and also allows each user to have their own view
or prioritisation. There is even the possibility of each user having
multiple worklists to represent prioritisation from the various viewpoints
they may need to consider.

Worklists could also be used to create a kanban-like interface. These
boards would allow for a linear kanban workflow for managing the flow of
work in a team for example.

Proposed Change
===============

Boards
------

A board is a set of worklists, with the added constraint that no task or
story may be in more than one of the worklists. If the user tries to add
a story or task to a worklist in the board which is already in a different
worklist, the attempt should be unsuccessful. In a board, the worklists
represent the lanes in a kanban board. Boards can be public or private to
the people with appropriate permissions to see it. Boards can optionally
be related to projects.

Worklists
---------

A worklist is an arbitrary list of tasks and stories for which order may or
may not have meaning. Worklists can be public or private to the people with
appropriate permissions to see the worklist. Each story or task can be in
any number of worklists. Worklists can optionally be related to projects.

Automatic Worklists
-------------------

Some people may find it useful to have a worklist containing only items which
meet some criteria, for example a worklist containing all tasks related to a
project with status "review". Keeping such a worklist manually updated would
be difficult and time consuming, so it should be possible to create a worklist
which automatically updates its contents.

Implementation
==============

Database
--------

A new table called ``worklists`` would be created with the following columns:

* title
* creator_id
* project_id
* created_at
* updated_at
* private (boolean, default to False)
* archived (boolean, default to False)
* automatic (boolean, default to False)
* permission_id

A new table called ``worklist_criteria`` would be created. This table would be
used to implement automatic worklists, and would have the following columns:

* list_id
* item_type (either ``story`` or ``task``)
* field
* value
* title

A new table called ``worklist_items`` would be created with the following
columns:

* item_id
* item_type (either ``story`` or ``task``)
* list_id
* list_position

A new table called ``boards`` would be created with the following columns:

* title
* description
* creator_id
* project_id
* created_at
* updated_at
* private (boolean, default to False)
* archived (boolean, default to False)
* permission_id

A new table called ``board_worklists`` would be created with the following
columns:

* board_id
* list_id
* position

UI Behaviour
------------

When the user creates a worklist, they will be able to choose whether it
is public or private to people with permissions. If it is public, it will
still only be editable by people with adequate permissions. The user will
also be able to define a group of people who have edit permissions on the
list. Tasks and stories will also be able to be added to the list both at
the time of creation and at any point in the future. An existing list will
have the option of being "archived", as opposed to being deleted.

Whether or not a worklist is "automatic" is also selected when creating
the list. Criteria for the contents of an automatic list are chosen here.
If an existing list is converted to an automatic list, the existing
contents are removed from the list. Automatic lists cannot be re-ordered.

When the user creates a board, they will be able to select whether it is
public or private in a similar way to worklists. The user will also be
able to create worklists to be the lanes of the board. These worklists
will be given the same ACL as the board. Tasks and stories can be added
to these lists at this time, but a task or story cannot appear more than
once in the board. After creation, the worklists function like normal
worklists, with the ability to drag tasks or stories between them. If
the whole board gets archived, all the worklists that make it up are
archived too.

The user cannot add existing worklists to a board, to avoid unexpected
changes happening to a worklist used elsewhere due to the board changing,
or unexpected changes occurring in the board due to a worklist used
elsewhere changing. It should be possible to set up an automatic worklist
to "mirror" an existing worklist if that behaviour is desired.

API
---

Two new API endpoints will be created, ``/v1/boards`` and ``/v1/worklists``.
These new endpoints will support basic CRUD operations to allow management
of worklists and boards.

Assignee(s)
-----------

Primary assignee:
  adam-coldrick (Adam Coldrick)

Gerrit Topic
------------

Use Gerrit topic "worklists" for all patches related to this spec.

.. code-block:: bash

    git-review -t worklists

Work Items
----------

* Add new tables to the database model, and write a migration.
    * boards, worklists and some mapping tables
* Implement REST API for boards and worklists.
* Implement board and worklist functionality in the webclient.


Repositories
------------

No new repositories.

Servers
-------

No new servers.

DNS Entries
-----------

No new DNS entries

Documentation
-------------

Storyboard's API documentation will need updating with the new
endpoints.

Documentation to aid in the usage of worklists and boards should
be created.

Security
--------

It is important to ensure that 'private' worklists and boards can
only be accessed by the people in the ACL. It is also important to
ensure that worklists which are used in boards have the same ACL
as the board. Care will also need to be taken to ensure that
information about the contents of private worklists and boards
doesn't leak onto any other pages.

Testing
-------

The existing unit tests for StoryBoard will need to be extended to
cover the new API endpoints and database tables.

Dependencies
============

This work doesn't depend on any other work.
