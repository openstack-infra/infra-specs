::

  Copyright (c) 2014 Hewlett-Packard Development Company, L.P.
  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

======================
Public Hiera directory
======================

This is a specification for creating a new public directory embeded in the
openstack-infra/config project to hold public hiera data.

Problem description
===================

Puppet code in openstack-infra/config, referred to hereafter as config,
currently uses the hiera data backend. The advantage to using hiera in a puppet
environment is to separate the presentation of data from the function of code.
An obvious consequence of this is that the secret data can be put into hiera
and hidden, making the puppet code sanitized for public viewing. This is
exactly what has happened with config.

The disadvantage to the current system is that non-secret data is currently
very awkwardly managed. There are some instances, where non-secret data is
pushed into the secret hiera. We also have instances, such as the long list
of jenkins plugins, where the data is embedded into the Puppet code. We also
have variables set at top scope, such as elasticsearch_nodes, that can be
moved into hiera. This also makes use of hiera limited to core contributors.

Proposed change
===============

The proposed solution is to create a second hiera directory, embedded into
the config git repository, for public data to live in. Hiera will have to be
reconfigured to work with both directories, preferring the secure
directory. This should make consuming openstackci as a downstream easier.
Since variables set in this hiera directory can be used or overridden as
necessary.

Alternatives
------------

Status quo.
Create a 1:1 copy of the secret hiera git repository containing only fake
data.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Spencer Krum (nibalizer)


Work Items
----------

* Move the puppet working dir out of /opt into /etc

  This is required because the hiera yaml backend doesn't really
  support multiple data dirs. We can hack it by setting the
  hiera data dir to be a common superior directory of both dirs.
  We don't want to use / so we will use /etc/puppet and move
  the puppet working dir out of /opt into /etc/puppet.

* Configure a second data directory in the infra/config git repo
* Set hiera.yaml appropriately to source both dirs in order

  Puppet will do this by managing the config file.

* Symlink /etc/hiera.yaml to /etc/puppet/hiera.yaml

  This ensures that the hiera lookups performed by Puppet and
  the hiera lookups performed by the 'hiera' command line tool
  use the same configuration.

* Seed with data


Repositories
------------

No new repositories

Servers
-------

No new servers

DNS Entries
-----------

No DNS changes

Dependencies
============

None
