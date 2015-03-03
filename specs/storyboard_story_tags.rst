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

======================
Storyboard: Story Tags
======================

https://storyboard.openstack.org/#!/story/98

StoryBoard needs to support tagging to allow for free-form classification
of stories by its users. It will also allow to port current tag-driven
workflows from Launchpad to StoryBoard, facilitating the transition.

Problem description
===================

Definition
----------
In information systems, a tag is a non-hierarchical keyword or term assigned
to a piece of information. This kind of metadata helps describe an item and
allows it to be found again by browsing or searching.

It also lets teams implement basic team-specific workflows, where the presence
or absence of a tag is used to derive state (rather than endlessly overloading
the data model with team-specific fields).

Historical usage
----------------
In OpenStack, Launchpad bug tags are currently used to:
- classify the affected submodule or domain of expertise ("xen")
- identify bugs that could be fixed by new contributors ("low-hanging-fruit")
- trigger notifications for subset of bugs ("security")
- define sets of bugs used as part of the release process ("juno-rc-potential")

Convergence in the use of standard tags is encouraged in Launchpad by the
ability to define "official tags" that the UI suggests in type-ahead.

See the current "official tags" used by OpenStack  at:
https://wiki.openstack.org/wiki/BugTags

Launchpad tags can be specified in search forms and subscription rules.
This lets users easily designate and retrieve subset of bugs in a reusable way.

Note that Launchpad blueprints notoriously do NOT support tags: this is seen
as a major feature gap (along with lack of comment, search...) supporting
the move to a new tool.

Proposed change
===============

We propose to let users associate tags to StoryBoard stories. Tags should be
supported as a search parameter (to retrieve stories associated with a given
tag). Users should be able to subscribe to tags (and get notified when tagged
stories are modified).

Tags should only contain lowercase alphanumeric characters, plus dash (-) and
underscore (_) characters. To encourage convergence, uppercase characters
should be automatically converted to lowercase on input. Their length is
limited to 30 characters.

In order to enable various use cases for tags, we propose three types of tags:

Free-form tags
--------------
Free-form tags are keywords that can be freely set (and unset) by any
authenticated user of StoryBoard. They may use any name they want, as long
as it respects the abovementioned syntax rules.

System tags
-----------
System tags are predefined keywords that we want to recommend usage of.

It is a feature needed to increase convergence in tag usage, and reduce
tag consolidation manual tasks. As an example, take a system tag called
"documentation", that would be used to signal a potential documentation impact.
The tag input field could use type-ahead to strongly suggest the correct
keyword when you start typing "doc..", hopefully reducing variants like "doc",
"docs", "documentations", etc.

System tags should come with a description of their intended usage, which
should be accessible as part of the StoryBoard interface.

System tags can be freely set (and unset) by any authenticated user of
StoryBoard. They shall be defined by superusers using a specific admin UI.

Protected tags
--------------
Protected tags are keywords that would be reserved for use by a specific group.
Members of the protected tag owner group should be the only ones to be able to
set or unset the tag. For example, the release managers team could use a
protected "juno-ffe" tag to track that a Juno feature freeze exception was
granted to the corresponding story, and easily build lists of open feature
freeze exceptions.

Since tags are applied at story-level, any project-specific protected tag
should be project-specific itself. For example if Nova drivers want to track
that the spec for a given story has been approved, they could use a
"nova-approved" protected tag to that effect. Since the owner groups are
distinct, an equivalent tag for Cinder drivers would have to be distinct (and
for example be called "cinder-approved").

Protected tags should come with a description of their intended usage and owner
group, and you should be able to access that information as part of the
StoryBoard interface. Protected tags shall be defined by superusers using a
specific admin UI.

NB: Protected tags are a new feature: they are not needed for Launchpad feature
parity. However, they are a convenient way to implement a number of workflows
in StoryBoard that would otherwise need to be hardcoded. Given the very
dynamic nature of OpenStack development, it's easier to give users concepts
that they can build experimental workflows with than to require code changes
or configuration changes to let them implement them.

Alternatives
------------
The "tag" concept could be extended to other citizens of StoryBoard, like tasks
or projects.

I think allowing tags to be applied to tasks would confuse the UI a lot and
result in non-intuitive behavior (do new tasks inherit other tasks
tags ?). The experience from Launchpad usage (where the tag is applied to the
bug rather than on the task) shows that tags apply to most tasks anyway, and
humans can read that information in a smart way. The value of this additional
granularity in information is limited, while the cost in extra data entry and
UI overcrowding is high.

Project tags could make sense to designate informal groups of projects.
However we already implement projectgroups, which should ideally be cheap to
create and therefore cover that use case.

Implementation
==============

Implementation details should be left to the person that would do the
implementation work. What follows is just a suggestion.

Each type of tag should be quickly identifiable. The proposed implementation
would represent them with different background colors:

* Free-form tags would be represented with a neutral background (grey color)
* System tags would be represented with a cold color (blue/green color)
* Protected tags would be represented with a hot color (red/yellow color)

Assignee(s)
-----------
Primary Assignee:
    TBD

Work Items
----------
* Create an API to define system tags and protected tags
* Create an API to associate tags to stories
* Add tag data in story details API responses
* Teach the storyboard-webclient to use those new APIs

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
Free-form and system tags may be set and unset by any authenticated user.
They may therefore be used for spamming (set), or info destruction (unset).
However this is not different from any other potentially more lucrative fields
(like title or description). The benefit of letting anyone authenticated
edit data in a task/bug tracker generally outweighs those drawbacks, which are
not specific to tags.

Testing
-------
TBD

Dependencies
============

Protected tags will need "StoryBoard Teams API and management spec" to be
approved and implemented first (specs/storyboard_teams.rst).
