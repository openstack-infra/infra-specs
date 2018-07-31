.. code-block:: text

  Copyright 2019 Red Hat, Inc.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===================================
Use letsencrypt for infra SSL needs
===================================

Incorporate https://letsencrypt.org/ into our configuration
management.

Problem Description
===================

Currently SSL certificates for infrastructure services are manually
purchased at some cost, renewed by hand and provisioned into
configuration management.  The process is costly, work-intensive and
does not completely match our usual infrastructure-as-code workflows.
The overheads are a barrier to getting encryption rolled out across
all services.

We would like to replace manual provision of purchased certificates
with those provided by https://letsencrypt.org/.  The solution should
make certificate provision and renewal completely automated and
integrate with the existing configuration management environment.

What is the existing situation?
-------------------------------

* Certificates manually renewed.

* Certificate key data is manually inserted into Ansible variables
  on bridge.o.o in the key storage area

* Key material is then deployed as part of usual periodic
  ansible/puppet run on host; usually by puppet as a hiera variable
  (e.g. ``ssl_cert => hiera('foo_ssl_cert')``) but possibly by ansible
  directly.

* Self-signed certificates are created for a number of less
  "important" services, such as dev servers, or nodepool status pages.
  Primarily due to the cost an inconvenience of creating certificates.

What can we use letsencrypt for?
---------------------------------

Domain validation certificates.  This means you have proved you
control the domain to letsencrypt.  They do not offer higher level
verification such as `Extended Validation
<https://en.wikipedia.org/wiki/Extended_Validation_Certificate>`__
which prove the identity of the owner.

Our needs are currently exclusively domain validation certificates.

What about non-http services?
-----------------------------

Although not a initial target, non-https services like zookeeper and
gearman should be possible.

Zookeeper for example uses `JKS
<https://cwiki-test.apache.org/confluence/display/ZOOKEEPER/ZooKeeper+SSL+User+Guide>`__
which it should be possible to `import
<https://community.letsencrypt.org/t/tutorial-java-keystores-jks-with-lets-encrypt/34754>`__

SMTP can also integrate with letsencrypt for TLS; see
https://starttls-everywhere.org/ and
https://www.jwz.org/blog/2018/06/starttls-everywhere/

Is letsencrypt a long term solution?
------------------------------------

The `Internet Security Research Group
<https://letsencrypt.org/isrg/>`__ and by extension the letsencrypt
project appears to have gained a critical mass of support, making it
the defacto solution for domain validation certificates.  We can
benefit and contribute to open tooling around letsencrypt.

How do you validate you own a domain for renewal?
-------------------------------------------------

* https://letsencrypt.org/how-it-works/

You run an agent that communicates to letsencrypt, which challenges it
to either

 * put a http resource on the domain
 * add a dns entry

Once done, it will accept the public key associated with the challenge
and issue certificates.

Are there rate limit concerns?
------------------------------

* https://letsencrypt.org/docs/rate-limits/

50 new certificates per registered domain per week.  It is possible to
have up to 100 names per certificate.  Requests that count as renewals
count towards quota, but are also allowed if you are over quota.

The last renewal was 17 certificates, with one having 8 names and the
rest only 1 name (per clarkb).  Thus we are well within rate limits.

Any interactions with other TLS infrastructure?
------------------------------------------------

There already exists a ``*.openstack.org`` certificate in use for the
caching provider for ``www.openstack.org`` and is thus outside infra
control.  There is also a certificate for
``openstackid-resources.openstack.org`` which was not infra issued.

Can we use the HTTP challenge model?
------------------------------------

This model has a chicken-egg problem of needing a webserver to respond
before being able to get a certificate for the webserver you are
trying to secure.  There are several facets to this.

It would be possible for any host to configure it's Apache/webserver
to statically serve the ACME challenge, and have an agent run
periodically on the host to generate the keys.  Some common roles may
be useful in the base job repos, but essentially each service would
set this up in a besopke manner for it's environment.

This probably works fine for completely new services.  However, in a
server-replacement scenario, we can not transparently generate new
keys for the server before switching the A/AAAA DNS entries.  For
example, if the ``foo.opendev.org`` service is handled by
``foo01.opendev.org`` and we wish to make ``foo02.opendev.org``
replace it, we can not generate a certificate on ``foo02.opendev.org``
that covers ``foo.opendev.org,foo2.opendev.org`` before
``foo.opendev.org`` is redirected.  Thus we either need longer
downtime, or to start copying keys, etc.  This *can* be done with DNS
validation, because all we need to do is prove that we can update
``foo.opendev.org``; not actually switch it's A/AAAA records.

Incorporating HTTP challenge response into our existing deployment is
problematic.  Updating Apache configuration, installing ACME agents
and other setup crosses into a lot of existing puppet which would need
to be heavily reworked.

Other approaches can could be considered.  Certificates could be
initially issued centrally and the remote HTTP server would need to be
either given the token respond to the challenge, or otherwise somehow
forward the challenge back to the issuing host.  This is possible with
redirects and such; but again requires much more engineering than a
DNS based approach for no benefits.  It helps the extant puppet
situation of distributing keys from central storage, but makes things
more difficult for new deployments where we wish to keep keys on host
only.

We would also loose some flexibility; port 80/443 doesn't *have* to be
bound to a "standard" webserver, and thus it may require bespoke
solutions to ensure the HTTP challenge URL can be responded to.  With
DNS validation we never have to worry about making the host respond
correctly.

The conclusion of this spec is that HTTP authentication is not
suitable for our requirements.

DNS Validation Overview
-----------------------

DNS based validation seems a more reasonable path forward.  We have
two DNS environments to consider.

Firstly, the extant ``openstack.org`` domain is hosted by Rackspace's
Cloud DNS offering.  We do not expect to start new services homed
within the ``openstack.org`` domain, but we would ideally like to
support existing services having valid letsencrypt certificates.

Secondly, infra runs its own authoritative nameservers as part of the
OpenDev initiative.  This is the primary focus.

Are certificates precious?
--------------------------

Moving forward, we do not want to continue distributing certificates
from centralised secrets; they should be considered ephemeral and
generated and kept on the host as much as possible.  i.e. certificates
should not be precious.

However, there will be a period of transition as service deployment
moves from extant puppet modules which are taking their key data from
centralised secrets.

Ideally we can handle both.

What about load balancing?
--------------------------

With DNS based validation, every host can have a certificate that is
valid for itself and the balanced domain.  For example,
``git01.openstack.org`` will request a certificate to cover
``git.openstack.org,git01.openstack.org``; upon request the two
relevant TXT authentication records (one for each domain) are placed
into DNS and the authentication is complete.

What ACME agent/tool should we use?
------------------------------------

The underlying protocol for getting certificates is called `ACME
<https://letsencrypt.org/how-it-works/>`__.  There are many clients;
see `<https://letsencrypt.org/docs/client-options/>`__.

Many of these tools are quite extensive, as they perform a range of
operations like automatically deploying and updating
apache/ngnix/other configuration.  As we manage configuration via our
own config management, we do not need any of these features.

The `acme.sh <https://github.com/Neilpang/acme.sh>` agent stands out
as being particularly suitable.

* It is implemented in ``sh`` (specifically *not* ``bash``) with very
  standard UNIX tools (curl, openssl, etc).  In contrast, tools like
  `certbot <https://github.com/certbot/certbot>`__ need a python
  environment. i.e. it basically runs anywhere, including tiny
  containers.
* It is currently maintained and an active project, with CI
* We have some experience with large shell-based projects (devstack,
  d-g).  The code looks good.
* It supports authentication via the two methods of interest to us; a
  "manual" mode where putting the TXT records is left up to the user,
  and it can operate directly on Rackspace DNS API.
* For future flexibility can operate on many other hosted DNS
  environments (like 50+ different ones).
* While it does have certificate deployment options to update
  apache/ngnix/exim/sshd/and so on, they are all opt-in and not
  necessary.
* It has an "I (heart) Docker" sticker on the page

For these reasons, ``acme.sh`` is used below.

Proposed Change
===============

The proposal is in two parts: the first covers the legacy
``openstack.org`` environment, whilst the second uses the self-hosted
nameservers behind ``opendev.org``.

openstack.org
-------------

Note: this proposal is prototyped at https://review.openstack.org/637456

An ansible-playbook can be written to iterate each ``openstack.org``
host of interest and generate certificate material with ``acme.sh``
using Rackspace DNS for letsencrypt authentication.  This playbook
runs periodically on ``bridge.o.o`` (once a day is sufficient) and
will store any generated certificate material in a private certifcate
storage area.

Minor post-processing after issuing a certificate combines the
resulting file contents of the key material into a single YAML file
for each host, with pre-defined and unchanging variable names.

As part of the base run, this generated YAML file with certificate
data is copied into the relevant host's hiera directory (and hiera is
configured to load from this file).  At this point, extant puppet has
access to the certificates as normal hiera variables and can deploy
them with very little change.

OpenDev
-------

Note: this proposal is prototyped at https://review.openstack.org/636759

For projects under the OpenDev umbrella we have two goals:

* Key material ephemeral and stored on server
* Using infra hosted DNS servers

We create an ansible role that will generate the certifiate data into
files on a given host.  We will consider actually *using* the
certificate data as implementation specific; playbooks for ensuring
service restart, mounting into containers, etc will need to be
incorporated on top of this proposal.

The proposal is as follows:

* We utilise a separate signing domain, hereon referred to as
  ``acme-opendev.org`` (see note below).  This is preconfigured, and
  only used for ACME authentication.
* Hosts that want a certificate preconfigure a CNAME record
  ``_acme-challenge.hostname.opendev.org`` pointing to
  ``_acme-challenge.acme-opendev.org``.  This is committed via the
  normal review process to the main DNS configuration.
* Each host defines the certificates it wants and the domains that
  should be covered by that certificate in a known configuration
  variable.
* Provisioning happens via ansible roles running on ``bridge.o.o``
  against the remote host that wants certificates as part of the
  standard ``base.yaml`` playbook.  The roles:

  0. Ensure a valid install of ``acme.sh`` on the remote host
  1. For each certificate required on the host, run ``acme.sh`` in
     "DNS manual mode" on the remote host with the required domains to
     be authenticated.
  2. This generates an authentication token(s) output that needs to be
     put into a TXT record on the signing domain.  This is stored in a
     pre-known host variable for the next step.
  3. Another role then reads the authentication tokens from each host
     via the variables, which are collated and put into a TXT records
     created on ``adns1.opendev.org`` as
     ``_acme-challenge.acme-opendev.org``.  Any necessary reloads
     etc. are done to make the records visible (see note below on
     lifespan).
  4. When records are available, the acme.sh renewal process is run on
     the remote host.  The remote letsencrypt servers authenticate the
     TXT records, and certificate material (cert, private key, chain
     file) is generated and put in a well-known location on the host.

The host now has valid keys and further roles/playbooks can configure
services to use them.

Notes:

 * we *could* have the signing domain as a sub-domain of
   ``opendev.org``.  We choose a separate domain for several reasons.
   Firstly, a large benefit of the signing domain is that it is
   completely separate to your production domain; it is one less thing
   to potentially mess up.  Secondly, having a separate domain gives
   us excellent forward flexibility -- for example; in the future we
   could host this domain somewhere that has an API for updating TXT
   records (designate instance, say) and use that directly for ACME
   authentication.  This would be transparent to ``opendev.org`` or
   any other hosts.
 * letsencyrpt `walks all the TXT records
   <https://github.com/letsencrypt/boulder/blob/1fe8aa8128078f7d822b3e3f3bcc979003896cc0/va/va.go#L759:L767>`__,
   and stops when it finds a match.  So we don't have any problem with
   multiple concurrent updates.
 * The TXT records are not explicitly removed.  They remain until the
   next time a certificate is issued or renewed, when the entire
   signing zone file is overwritten.  They are not in any way
   sensitive data.  Their useful life, however, is only within the
   single ansible/puppet pulse they were created in.
 * The exact renewal process will be defined when we can issue
   certificates.  The plan is that if the host has all valid
   certificates, step 1 above will return an empty list an nothing
   further will be done.  If any certificate has less than 30 days
   remaining, acme.sh should automatically renew it, which is
   essentially the same as issuing a new certificate.
 * For organisations, letsencrypt has group accounts and such.  These
   will be implemented once basic issuance and renewal is working.

Alternatives
------------

* Continue with manual renewal
* We could only implement the ``opendev.org`` portion, leaving the RAX
  DNS based authentication for ``openstack.org`` alone, assuming it
  will not be required after the renewal period of the existing
  certificates.

Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  ianw

Can optionally list additional ids if they intend on doing substantial
implementation work on this blueprint.

Gerrit Topic
------------

Use Gerrit topic "letsencrypt-infra" for all patches related to this spec.

.. code-block:: bash

    git-review -t letsencrypt-infra

Work Items
----------

* The broad strokes of the proposals are layed out in the prototypes.
  Getting those various roles production ready, tested and documented
  are the major work items.


Repositories
------------

Unlikely to require new repositories.  Jobs will be in existing repos.

Servers
-------

No new servers required.

DNS Entries
-----------

``acme-opendev.org`` or whatever separate domain is used for CNAME
redirection will need to be provisioned.


Documentation
-------------

Standard infra documentation (system-config, job documentation, etc)
will need updates, but no external documentation should require
updates.

Security
--------

We are dealing with more private halves of keys.  However we are
moving to a model where they are considered ephemeral and can be
rotated at any time.


Testing
-------

* We can stage implementation of keys to hosts, meaning no "flag" day
  of switching everything, and we can test in isolation.

* We do not have a good way to update DNS entries for gate CI testing.
  Initially we can generate self-signed certificates.  For future
  work, there are docker environments that run a LE environment.  We
  could stand-up one of these either on a per-test basis or as a
  service (might be useful if we have a set CI signging key we could
  configure hosts under test to trust).

* Initial production testing can be done with the letsencrypt staging
  environment, which doesn't give valid keys but also doesn't have
  restrictive limits if things are failing.

* Ongoing validation with certcheck


Dependencies
============

This work needs to integrate with ongoing work in the `update config
management spec`_.

.. _`update config management spec`: https://specs.openstack.org/openstack-infra/infra-specs/specs/update-config-management.html
