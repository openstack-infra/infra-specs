
::

  Copyright (c) 2015 Wayne Warren

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===============================
JJB 2.0.0 API Changes
===============================

https://storyboard.openstack.org/#!/story/2000258

The goal of this spec is to define a series of backwards-incompatible API
changes that will necessitate a 2.x release of Jenkins Job Builder. The
high-level goals of these changes are:

* to ensure that the various classes are useful outside of JJB itself where
  appropriate
* to clearly define public interfaces for various classes and document the
  purpose/role of these classes in the API
* to eliminate unuseful class relationships in favor of tiered delegation of
  program behavior
* to reduce the importance of the YamlParser as the only source of the data
  structures necessary for XML generation


Problem Description
===================

Jenkins Job Builder has history of organic, piecemeal development wherein
patches tend to be highly focused on improving one or another aspect of the
program. For the most part this is understandable, expected even--the modular
implementation of XML generation on a per-Jenkins-plugin basis lends itself well
to this style of development. However when it comes to changes to the core
abstractions, in particular the Builder and YamlParser clases, many ad hoc
design decisions have led to tightly bound implementations that make the overall
API difficult if not impossible to reuse as a library.

It's true that yes, we can import jenkins_jobs.Builder or
jenkins_jobs.parser.YamlParser to use them programmatically outside of JJB
proper, but it is difficult to imagine anyone actually doing this given the
current constructors for these classes as well as their relationships--Builder
objects actually own references to YamlParser objects when there are no clear
benefits to this organization save for expediency during JJB development.

The YamlParser itself actually does more than "parse" YAML, it is also
responsible for generating XmlJob objects from the yaml data it handles. This
means that all the lovely XML building code that makes JJB truly special cannot
reliably be accessed using JJB as a library since it depends on implementation
details of the YamlParser (specifically the xml_jobs attribute).

In addition to the tight binding between classes that make them practically
unusable, there is an almost impossibly tight binding between code in
jenkins_jobs.execute() and jenkins_jobs.Builder, which depends on code in
jenkins_jobs.execute() to munge various command line options and configuration
data, ie to obtain user, password, plugins_list, cache settings, etc. To top it
off, jenkins_jobs.Builder ends up taking the "config" object *anyway*, which
should make us question to what purpose jenkins_jobs.execute() is even handling
various pieces of configuration logic at all--arguably, this logic ought to live
in a separate "Config" class.

The problems described up to this point indicate a need to clean up the API, but
one issue that really deserves to be driven home is the essential character of
the YamlParser to everything that JJB does. Because of the tight bindings
between classes described above, understanding the tricky behavior of the
YamlParser is absolutely necessary to construct anything more simple than a
collection of "job" objects or even a simple set of "project"s containing lists
of "job-template"s.

This is primarily problematic because if JJB is going to offer a programmatic
API, it should be possible to use it to do something more interesting than the
exact same thing that is being done in jenkins_jobs.execute(). For example, it
should be possible to define one's own set of abstraction classes in Python that
can be used to generate the data used by jenkins_jobs.Builder to POST jobs to a
Jenkins server.

JJB plugin modules all take a "parser" argument, but rarely if ever is this
parser argument used in these modules. This parser is intended to be a reference
to a YamlParser instance but the benefit of this relationship does not really
outweigh its cost in terms of tight binding between plugin module and the means
used to obtain JJB data structures actually necessary for generating XML; its
primary use case most of the time is to provide access to the ModuleRegitry
object owned by the YamlParser. It would make more sense to replace this with an
explicit reference to the ModuleRegistry itself.

A Side Note on Macros
---------------------

A problematic JJB feature when it comes to cleaning up the API is the "macro".
The current implementation leads to an unnecessarily tight binding between
the YamlParser and the ModuleRegistry because the ModuleRegistry and XML
generation code require the YamlParser's xml_jobs attribute in order to
determine whether or not a component is a macro. This tight binding is
problematic because it means we need to mock the YamlParser's xml_jobs data
structure in order to use any other means of producing JJB data structures than
the YamlParser itself. As of right now, it is fine to mock YamlParser's xml_jobs
data structure with an empty dictionary but some detail of the way it is
accessed in the ModuleRegistry or XML generation code could easily change that
would cause unforeseen problems when attempting to avoid using the YamlParser.

Proposed Changes
================

Modularize Command Line Parsing
-------------------------------

Since so much of JJB's API depends on JJB's configuration file and command line
options, it makes sense to modularize these in such a way that they can easily
be used by third party scripts/tools. This will be done using a dedicated
"Config" class.

This will allow constructors for other JJB classes such as the Builder and
YamlParser to be simplified--rather than taking both a "config" object and a
number of other parameters they will instead take a single "config" parameter.

Redefine Programmatic APIs
--------------------------

The following classes will need their APIs redesigned to make them usable by
third parties:

* YamlParser
* ModuleRegistry
* Builder

These rewrites should be accompanied by docstrings that explicitly define the
purpose of these classes so that these purposes can be considered in the context
of future patches against JJB that attempt to make foundational changes to their
behavior.

YamlParser
''''''''''

The current YamlParser exposes all of its methods as "public" methods even when
there is no obvious or implicit value in doing so. The goal for rewriting its
API will be to provide explicit "public" methods that other classes can call to
get information from it.

ModuleRegistry
''''''''''''''

The new ModuleRegistry will serve a dual role; in addition to registering JJB
modules for use during XML generation it will also provide an API for modules to
use to access information other than the given JJB data. For example, it should
provide access to Jenkins plugin version information to allow modules to
implement plugin version dependent XML.


Add XmlBuilder class
--------------------

The XmlBuilder class should define a method that takes as input a dictionary
where:

* keys are job names, and
* values are all the key/value pairs necessary to produce an XmlJob for the
  named job

This method should produce as output a list of XmlJobs that can then be passed
to a Builder method either to produce test output or to update jobs on the
configured Jenkins instance.

Alternate names for this class might be JenkinsXmlBuilder,
JenkinsJobConfigBuilder.

Replace references to YamlParser with references to ModuleRegistry in modules
-----------------------------------------------------------------------------

This addresses the need to decouple plugin modules from the YamlParser while
also allowing us to access components of the ModuleRegistry when necessary (ie,
accessing Jenkins plugins_info to generate different XML for different versions
of plugins).

Rewrite JJB Macro Implementation
--------------------------------

The goal of this rewrite should be primarily to remove the need for referencing
the YamlParser's raw yaml data structure while generating XML. This way the
proposed XmlBuilder class does not have to be concerned with any particular
details of the YamlParser--reducing the risk that implementation of one of these
classes has unforeseen side effects on the other's behavior.

One method that has been proposed is to implement a secondary module registry
that specifically handles macros.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Wayne Warren, wayne+launchpad@sdf.org

Gerrit Topic
------------

Use Gerrit topic "jjb-2.0.0-api" for all patches related to this spec.

.. code-block:: bash

    git-review -t jjb-2.0.0-api

Work Items
----------

Development Process
'''''''''''''''''''

All work for this spec will target a feature branch of JJB called "2.0.x" which
will contain all backwards-incompatible changes targeted for this release,
including any work covered by other specs targeting the same release.

Once the work is complete, we will release Jenkins Job Builder 2.0.0.

Code Changes
''''''''''''

* Break command line parsing function into dynamically-loaded, modular
  components for consumption outside of the primary JJB command line tool.
* Create function that specifically handles pre-Builder processing
  of command line options and configuration settings; this will be necessary to
  make the Builder class usable as an API to be consumed externally.
  (Alternatively, consider ways to clean up the Builder constructor in such a
  way that doesn't need to take the full "config" object)
* Separate XML generation code from YamlParser, move into new class "XmlBuilder"
  that takes a ModuleRegistry object in its constructor and contains various
  methods that take one or more JJB job dictionaries and output a corresponding
  number of XmlJob objects.
* Separate YamlParser from YamlInterpreter:
  * YamlParser will be responsible for loading a given list of YAML files
  * YamlInterpreter will be responsible for applying JJB's DSL semantics to the
  loaded lists of dictionaries.
* Refactor Builder methods and their calling contexts to simplify the interface;
  the "update_jobs" method, for instance, should simply take a list of XmlJobs
  to update.

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

Documentation changes may be necessary as this work is primarily intended to
make existing JJB API usable outside of the jenkins_jobs.cmd module.

Security
--------

None

Testing
-------

Backwards Compatibility
'''''''''''''''''''''''

No existing test scenarios will be changed to support this spec; however, there
will need to be changes to test fixture base classes to support the new API
changes.

We will also monitor openstack-infra's JJB output to ensure that it does not
change because of this spec.

New Tests
'''''''''

Unit tests will explicitly validate the behavior of the following:

* the Builder class' "public" methods
* the YamlParser class' "public" methods
* the XmlBuilder class' "public" methods
* the Config class' "public" methods

Dependencies
============

Python Support
--------------

We intend to continue supporting the same Python versions supported by JJB 1.x:

* Python 2.7
* Python 3.4

