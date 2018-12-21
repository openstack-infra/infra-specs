::

  Copyright 2015 Hewlett-Packard Development Company, L.P.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode


====================
Ansible Puppet Apply
====================

https://storyboard.openstack.org/#!/story/2000207

Our current Config Management system consists of using ansible to orchestrate
running puppet agent on each of our hosts, which then have to connect back
to a puppetmaster to learn what they should apply. It's a highly coupled system
that ultimately doesn't need to be that complex.

Problem Description
===================

While the puppetmaster has served us well for quite some time, with the
introduction of ansible for sequencing and orchestration, it's no longer
needed. Having the puppetmaster introduces a few complexities, such as the
need for certificate exchange and the certificate re-use dance when we're
replacing servers. It's also a bootstrapping issue when spinning up a new
infra, since you need the puppetmaster in place to get the puppetmaster in
place.

Finally, our friends from puppet sometimes like to release new versions of
puppet that are backwards incompatible in ways that touch the wire protocol.
If we're using puppet apply on each of the nodes, then rolling out a per-node
upgrade is simple.

Proposed Change
===============

Add a playbook that handles cloning system-config on all of the nodes and
potentially running install_modules.sh on them. This should run before puppet
is run. As an optimization, we could fetch on the master and rsync to all of
the nodes, or do clarkb's trick of fetching the ref on the master and passing
that to the remote git clone/update commands.

Split the current hiera common.yaml into a set of files, a
common.yaml file that contains things that should be available everywhere. A
set of $group.yaml files, which map to a group of servers where every server
in the group should have access to the values. And a set of $fqdn.yaml files
that contain values that should only be accessed by one server.

Group membership from the puppet perspective will be managed by a group variable
set at the node level in site.pp. There should be an ansible group for every
defined puppet group. For the purposes of this, that grouping will be managed
manually, but as a follow on, we should find a better way to manage this
mapping. Also, for the purposes of this, each node will only have one group
from puppet/hiera perspective. Ansible supports multiple groups, as does the
ansible openstack inventory module. For now, we'll avoid making use of that.

In the ansible-puppet role, copy over the master's hieradata that is appropriate
for that node to have. That is, the common.yaml file, any $group.yaml files
for the node in question, and the $fqdn.yaml file. Then, have ansible run puppet
apply pointing at the manifests/site.pp file on the host.

We will need to populate the hiera files as part of node bootstrap. This will
be simple once we have replaced launch_node.py with an ansible playbook using
the openstack modules. However, that work actually wants this work first to
simplify the exchange, so for now, we will want a patch to launch_node.py that
calls the ansible puppet role instead of doing the ssh to the host to run
puppet itself.

To handle disabled nodes, we'll mark all of our ansible playbooks in run_all
with ;!disabled - referencing an ansible node group called disabled that we
can add and remove hosts from on the master.

Since we will not have a puppetmaster, we'll need to coordinate puppet apply
reporting to puppetdb. For this, we need to create a CA, a cert that the
puppetdb server will have, and a cert that we will distribute to every host
we run. The certs for the cattle and the puppetdb server can be installed via
hiera secrets, with the cattle cert being an excellent target for common.yaml.

Alternatives
------------

We could just keep using puppet agent.

We could ditch puppet altogether and move completely to ansible.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
 - mordred
 - nibalizer

Gerrit Topic
------------

Use Gerrit topic "puppet-apply" for all patches related to this spec.

.. code-block:: bash

    git-review -t puppet-apply

Work Items
----------

The description is kinda already written like work items.

Repositories
------------

None

Servers
-------

None

DNS Entries
-----------

None

Documentation
-------------

None

Security
--------

Each server will be contacting puppetdb directly, rather than funneling
through the puppetmaster.

Secrets related to a particular server will be stored on that server. Of course,
those secrets are all aready on that server in some form anyway.

Testing
-------

We already have puppet apply tests, so that's awesome.

Dependencies
============

None
