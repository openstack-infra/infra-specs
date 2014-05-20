======================
Infra Specs Repository
======================

This is a git repository for doing design review on enhancements to
the OpenStack Project Infrastructure.  This provides an ability to
ensure that everyone has signed off on the approach to solving a
problem early on.

Repository Structure
====================
The expected structure of the respository is as follows::

  specs/
      implemented/


Expected Work Flow
==================

1. Create a story in StoryBoard with a task affecting the
   ``infra-specs`` project.
2. Propose review to infra-specs repository (ensure Story:<story number> is
   in the commit message).
3. Leave a comment with the Gerrit address of the specification.
4. Bring forward the proposed item to the infra meeting for summary.
5. Review happens on proposal by infra-core members and others.
6. Iterate until review is Approved or Rejected.

Once a specification is Approved...

1. Update story, copy summary text of specification to there.
2. Leave a comment to the git address of the specification.


Revisiting Specifications
=========================
We don't always get everything right the first time. If we realize we
need to revisit a specification because something changed, either we
now know more, or a new idea came in which we should embrace, we'll
manage this by proposing an update to the specification in question.

Learn As We Go
==============
This is a new way of attempting things, so we're going to be low in
process to begin with to figure out where we go from here. Expect some
early flexibility in evolving this effort over time.
