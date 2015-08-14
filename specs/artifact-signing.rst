::

  Copyright 2015 OpenStack Foundation

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Artifact Signing Toolchain
==========================

https://storyboard.openstack.org/#!/story/2000336

The OpenStack Community will publish cryptographic signatures
accompanying release artifacts (tarballs, packages, et cetera) to
provide a verification of provenance.

Problem Description
===================

Unlike our Git repositories, where releases are represented by
cryptographic signatures of release managers embedded in tags, the
artifacts built in our infrastructure from those tagged repository
states come with no proof of their origin. Files uploaded to our
tarballs site or external services lack any attestation that they're
unmodified since the time of their original creation and
publication. This allows, among other risks, the possibility that a
published release can be altered either accidentally or by a
malicious actor.

Proposed Change
===============

Add a dedicated job worker responsible for performing artifact
signing between the generation and publication steps.

Alternatives
------------

* As always, we could do nothing, though the problem remains.
* We could engineer a system whereby individuals vet release
  artifacts and upload their own individual attestations, though
  that's not necessary precluded by the proposed solution anyway and
  could operate in parallel if desired.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fungi

Gerrit Topic
------------

Use Gerrit topic "artifact-signing" for all patches related to this
spec.

.. code-block:: bash

    git-review -t artifact-signing

Work Items
----------

1. Generate OpenPGP keypair with signing subkey, both set to expire
   at the beginning of the next development cycle; store the master
   private key and a revocation certificate with our other service
   credentials; add the subkey in hiera
2. Get all infra root admins to sign the artifact signing key and
   publish those signatures to the keyserver network
3. *openstack-infra/system-config*:

   1. Create a puppet class and node definition for
      signing.slave.openstack.org
   2. Add basic documentation of the infra root process for handling
      of the artifact signing key (infra root keysigning,
      publishing, extending the expiration date, and emergency
      revocation)

4. Launch the new signing.slave.openstack.org server
5. *openstack-infra/system-config*: Add signing.slave.openstack.org
   to cacti
6. Register the signing.slave.openstack.org in jenkins.openstack.org
   and add a ``signing`` label to it
7. *openstack-infra/project-config*:

   1. Develop a slave script for signing automation and integrate it
      into the openstack-infra/project-config repo (this could be an
      extension of or abstraction from some of the existing
      ``*-upload.sh`` scripts)
   2. Add a job to identify the artifact name (based on the tag and
      details within the repo, or just the tag and have multiple
      similar jobs), retrieve it, create a detached signature and
      then upload it back to an adjacent path
   3. Extend the release upload jobs/job-templates to retrieve,
      validate and incorporate detached signatures, at least for
      reuploads to external services which currently support it

8. *openstack/governance* and/or *openstack/ossa*: Add documentation
   of the existence of artifact signatures and instructions on
   validating them
9. Send an announcement to the
   openstack-announce@lists.openstack.org mailing list informing the
   community of the availability of signatures accompanying all
   upcoming releases

Repositories
------------

No new git repositories need to be created.

Servers
-------

A new signing.slave.openstack.org server needs to be created. No
existing servers will be affected.

DNS Entries
-----------

DNS entries (A/AAAA/PTR) for signing.slave.openstack.org need to be
created.

Documentation
-------------

We will need documentation for key handling/signing/rotation in the
openstack/system-config repo, outlining the process followed by
infra root admins. Documentation in either the openstack/governance
or openstack/ossa repos should probably also be created explaining
the existence of our artifact signatures and how they can be
validated. There is no anticipated impact to our current developer
workflow. An announcement is warranted once the implementation is in
place and confirmed working as intended.

Security
--------

There are associated security risks, but they are not regressions
from the current situation. Specifically, the artifact master and
signing subkeys (and revocation certificate too) need to be
safeguarded closely as they provide guarantees against post-release
tampering. Our existing secret management solutions should be
sufficient for this purpose. The signing subkey is the only one
exposed to a separate server (signing.slave.openstack.org), and the
Jenkins master to which this is registered is theoretically
vulnerable to attack from its other Jenkins slaves. Switching away
from Jenkins or upgrading to a release which supports `slave to
master access control
<https://wiki.jenkins-ci.org/display/JENKINS/Slave+To+Master+Access+Control>`_
may be warranted to mitigate this risk.

Note that this spec does not attempt to address trust challenges
earlier in the development, test and build toolchain. There are
strengthening opportunities throughout our infrastructure, but they
are out of scope and should be handled as separate specifications.

Testing
-------

One or more artifact signing jobs will need to be created to
generate and upload detached signatures to tarballs.openstack.org.
There are also opportunities here to add some additional validation
in our jobs which upload to external repositories, as they can
potentially validate the detached signatures and refuse to upload if
something looks wrong.

Dependencies
============

There are no dependencies for this specification.
