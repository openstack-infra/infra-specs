::

  Copyright 2016 Hewlett Packard Enterprise Development LP

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=============
Survey Server
=============

Include the URL of your StoryBoard story:

https://storyboard.openstack.org/#!/story/2000691

Offer an open source survey service for OpenStack projects.

Problem Description
===================

We don't provide a solution in infra for creating and sharing surveys
with their communities. As a result, many use third party and/or
proprietary services.

Proposed Change
===============

Create a new server for surveys running LimeSurvey_.

There is no Ubuntu package in the repositories for this service, so
we'll need to create a Puppet module that handles installation and then
keep up with security updates.

Implement webserver-based authentication with OpenID using Apache
mod_auth_openid and LimeSurvey's auth_webserver plugin.

.. _LimeSurvey: https://www.limesurvey.org/

Alternatives
------------

Maintained open source survey options are very limited. There is an
older Ruby on Rails application called Journey_ and a new, alpha release
one called Encuestame_ Java application. Given the framework
requirements, these both feel a bit overkill for our simple needs.

One option for simple surveys may be to use our new Ethercalc_ server
as it can do forms with responses that fill counters in sheet fields.
I have a hunch though that this will be too simple though as it would
be easily gamified.

.. _Journey: http://welcome.journeysurveys.com/
.. _Encuestame: http://www.encuestame.org/
.. _Ethercalc: https://ethercalc.openstack.org

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Anita Kuno (anteaya)

Gerrit Topic
------------

Use Gerrit topic "survey" for all patches related to this spec.

.. code-block:: bash

    git-review -t survey

Work Items
----------

 * Investigate availability of existing LimeSurvey Puppet modules.
 * If none are available, develop a puppet class in a system-config
   patch to do the work with our current implementation of Puppet 3.
   Revisit this direction once our Puppet is upgraded.
 * Configure LimeSurvey to use the mod_auth_openid module.
 * Launch the new server with a survey service and MySQL database.

Repositories
------------

None at this time

Servers
-------

Create survey.openstack.org

DNS Entries
-----------

survey.openstack.org

Documentation
-------------

We will need to write system-config documentation for administration of
the survey software.

User-facing software may also be needed, so contributors know about and
how to interact with the survey software.

Security
--------

We will need to pay attention to security updates of the software from
upstream since there is no distro package.

Testing
-------

N/A

Dependencies
============

N/A
