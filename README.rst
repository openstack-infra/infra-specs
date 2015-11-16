======================
Infra Specs Repository
======================

This is a git repository for doing design review on enhancements to
the OpenStack Project Infrastructure.  This provides an ability to
ensure that everyone has signed off on the approach to solving a
problem early on.

Expected Work Flow
==================

1. Create a story in StoryBoard_ with a task affecting the
   ``infra-specs`` project.
2. Propose a change to infra-specs repository (ensure Story:<story
   number> is in the commit message).
3. Leave a comment on the story with the Gerrit URL of the
   specification.
4. Review happens on proposal by infra-core members and others.
5. When ready for final approval, bring forward the proposed item to
   the infra meeting.

Once a specification is approved...

1. Update story, copy summary text of specification to there.
2. Leave a comment to the git address of the specification.

Revisiting Specifications
=========================
We don't always get everything right the first time. If we realize we
need to revisit a specification because something changed, either we
now know more, or a new idea came in which we should embrace, we'll
manage this by proposing an update to the specification in question.

.. _Storyboard: https://storyboard.openstack.org
