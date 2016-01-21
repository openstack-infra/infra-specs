::

  Copyright 2016, SUSE LLC

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Improve Translation Setup
=========================

https://storyboard.openstack.org/#!/story/2000452

Projects that want to setup translations for their repositories run
into a few common problems and we should address these to make the
setup for them easier. The problems they run into come from
limitations of the current infra scripts, so we need to rework the
infra scripts.

We should also have a simple and consistent setup for translations -
with a sensible consistency. Right now it's consistent for python
projects but not sensible in many cases.

Note that manuals are also translated but are not part of this spec.

Problem Description
===================

Initially only the core projects, like horizon, keystone, neutron,
nova and swift, have been translated. With the growth of OpenStack
more and more projects want to have translations. So, now we have
translations for libraries like oslo.i18n, for python clients like
python-novaclient or python-openstackclient, for dashboard plug-ins
like designate-dashboard, or for networking plug-ins like
networking-l2gw or networking-ovn.

Setting up translations for new projects currently has the following
challenges:

#. For "python" repositories, we expect that the locale file is
   located at ``$repo/locale/$repo.pot`` - without any change possible.
   This leads to python-novaclient/locale/python-novaclient.pot and
   oslo.log/locale/oslo.log.pot - in both cases the python module has
   a different name, novaclient and oslo_log.

   Currently everybody makes it wrong and we have to help them use
   the proper file names.

   There's also an inconsistency on how the gettext ``domainname`` is
   set, file ``setup.cfg`` contains an entry and also oslo.i18n's
   initialization. For some projects these are not in sync.

#. For dashboard repos using django, there's no common entry point yet
   and some are even setup like "python" repositories (like
   ``openstack/designate-dashboard``). Each one is done differently,
   so needs special casing in the infra scripts.

#. There's also discussion going on to add to the various networking plug-in
   repositories also dashboard content so that they contain both
   python and Django directories. The vitrage-dashboard repository is
   an existing one that is set up this way but not yet translated. We
   should define how to translate these.

Proposed Change
===============

The major changes are a common set up for the Django repositories as
well as a new way to determine the location of the POT files:

#. We will identify the ``modulename`` of each repository in the
   following way: The file ``setup.cfg`` is parsed and the
   ``packages`` entry in the ``[files]`` section is used as name for
   the module. If there is more than one entry, all matching the regex
   ``^tests.*`` are removed and the shortest entry is used. Note that
   the entries have to be valid python object paths, the scripts will
   not handle repositories with invalid ones like entries containing a
   "/".

   For some repositories, we might need to special case this and add
   an exact modulename - but this should be a rare exception.

#. Instead of using ``$repo/locale/$repo.pot``, the CI scripts will use
   ``$modulename/locale/$modulename.pot`` as location for POT files of
   python projects.

#. Repositories using the django framework use two domains, one called
   ``django.pot`` for strings from Python files and one called
   ``djangojs.pot`` for strings from Javascript files.

   Since these jobs run on a slave that has access to passwords, we
   will not call scripts inside the repositories but let the CI
   scripts do the work.

   We will update the CI scripts to call pybabel directly like the
   following:

   .. code-block:: bash

      KEYWORDS="-k gettext_noop -k gettext_lazy -k ngettext_lazy:1,2"
      KEYWORDS+=" -k ugettext_noop -k ugettext_lazy -k ungettext_lazy:1,2"
      KEYWORDS+=" -k npgettext:1c,2,3 -k pgettext_lazy:1c,2 -k npgettext_lazy:1c,2,3"
      pybabel extract -F babel-${DOMAIN}.cfg \
        -o ${modulename}/locale/${DOMAIN}.pot $KEYWORDS ${modulename}

   For this, the files ``babel-django.cfg`` and
   ``babel-djangojs.cfg`` need to be setup in the repositories as
   follows:

   File ``babel-django.cfg``:

   .. code-block:: ini

      [extractors]
      django = django_babel.extract:extract_django

      [python: **.py]
      [django: templates/**.html]
      [django: **/templates/**.csv]

   File ``babel-djangojs.cfg``:

   .. code-block:: ini

      [extractors]
      # We use a custom extractor to find translatable strings in AngularJS
      # templates. The extractor is included in horizon.utils for now.
      # See http://babel.pocoo.org/docs/messages/#referencing-extraction-methods for
      # details on how this works.
      angular = horizon.utils.babel_extract_angular:extract_angular

      [javascript: **.js]

      # We need to look into all static folders for HTML files.
      # The **/static ensures that we also search within
      # /openstack_dashboard/dashboards/XYZ/static which will ensure
      # that plugins are also translated.
      [angular: **/static/**.html]

   This uses the ``Babel``, ``django-babel``, and ``horizon``
   packages, these will be installed by the CI scripts in a virtual
   environment for calling ``pybabel``.

   Teams do not have to set up any tox environment for their
   repositories but if they add a tox environment it should be called
   ``extractmessages`` and it should use pybabel like the CI scripts
   do.

#. All repositories ending in ``-dashboard``, ``-horizon``, or ``-ui``
   will get the treatment from the previous step. Projects that do not
   have special requirements will continue to use ``python setup.py
   extract_messages``.

   Note that the ``horizon`` repository has to be treated specially
   since it has two modules, all other repositories have only one
   module.

   This Django setup will be used for the following repositories that
   are currently setup (or planned to be setup) in ``project-config``
   repository for translations: horizon, django_openstack_auth,
   designate-dashboard, horizon-cisco-ui, magnum-ui, murano-dashboard,
   trove-dashboard, zaqar-ui.

#. Repositories like the networking plug-in ones that will have
   both Python and Django code will all be treated the same way. The
   CI scripts will run both Python and Django translations extraction.
   The exact way to configure the names of the Python module and the
   Django directory will be evaluated and documented.

   One idea to evaluate is supporting an optional section in file
   ``setup.cfg`` like the following:

   .. code-block:: ini

      [openstack_translations]
      django_modules = module1
      regular_modules = module2 module3

#. As a review policy, changes to enable translations in the
   ``project-config`` repository should only be done after a project
   is set up for translations, otherwise jobs are triggered that
   always fail.

Alternatives
------------

We can keep the status quo with leads to individual solutions for each
repository and education of teams on how to properly set up repositories.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* jaegerandi

Also:

* amotoki

Gerrit Topic
------------

Use Gerrit topic "translation_setup" for all patches related to this spec.

.. code-block:: bash

    git-review -t translation_setup

Work Items
----------

* Freeze adding of new translation setup for repositories until the
  spec is finished.

* Update the Infra scripts. Implement the new setup for one or two
  repositories and test everything, then add further repositories. As
  part of this we need to rename modules in Zanata and in
  repositories.

  There might be a freeze for translations sync while the renaming
  happens.

* Update documentation:

  - for script usage, the system description might get updated, it
    lives at
    http://docs.openstack.org/infra/system-config/translate.html .
  - for projects on how to enable translations, we should add a new
    section to the Creator's Guide of the `Infra Manual
    <http://docs.openstack.org/infra/manual/creators.html>`__.
  - The horizon devref documentation needs to be reviewed and updated
    accordingly.

* Review cookiecutter projects and see whether those need to be
  updated for these changes.

Repositories
------------

This will be done in existing repositories. Work will touch the
project-config repository for the infra scripts and might touch
repositories that are translated.

Servers
-------

No new servers are needed.

DNS Entries
-----------

No update for DNS entries is needed.

Documentation
-------------

The page http://docs.openstack.org/infra/system-config/translate.html
as well as the Infra Manual should get updated.

Security
--------

No security impact.

Testing
-------

Manual tests need to be done to check that the scripts work.

Dependencies
============

No dependencies exist.
