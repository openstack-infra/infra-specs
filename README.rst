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
   ``openstack-infra/infra-specs`` project.
2. Propose a change to this repository and make sure ``Task:
   #<taskid>`` for the corresponding story's initial task is
   included as a footer in the commit message (see
   ``CONTRIBUTING.rst`` for relevant documentation links). This
   change should also add an entry for the proposed spec document
   in the ``Approved Design Specifications`` section of the
   ``doc/source/index.rst`` file.
3. Once proposed, members of the community provide feedback through
   code review, and the specification should be revised until there
   seems to be some reasonable consensus as to its fitness.
4. When ready for final approval, request addition of a call for
   votes to the weekly `infra meeting`_ agenda.
5. If agreed by the meeting attendees, the chair will announce an
   approval deadline before which members of the `Infrastructure
   Council`_ are asked to cast their roll call votes on the proposal
   under review.

Once a specification is approved...

1. Update the story, copying summary text of specification to there.
2. Leave a comment linking to the published URL of the specification
   on the `specs site`_.

Revisiting Specifications
=========================
We don't always get everything right the first time. If we realize we
need to revisit a specification because something changed, either we
now know more, or a new idea came in which we should embrace, we'll
manage this by proposing an update to the specification in question.

.. _Storyboard: https://storyboard.openstack.org/
.. _Gerrit: https://review.openstack.org/
.. _infra meeting:
     http://eavesdrop.openstack.org/#Project_Infrastructure_Team_Meeting
.. _Infrastructure Council:
     https://docs.openstack.org/infra/system-config/project.html#teams
.. _specs site: https://specs.openstack.org/openstack-infra/infra-specs/
