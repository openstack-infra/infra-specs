::

  Copyright (c) 2014 Hewlett-Packard Development Company, L.P.
  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

======================
Zuul layout.yaml split
======================

This is a specification for split zuul layout.yaml config
in different yaml files.

Problem description
===================

Zuul is using a single layout.yaml to store all the config.
It ends up in a very long and not readable file, with projects,
pipelines and jobs merged together. That is complicated
to understand and it also makes difficult for non-openstack
projects using this system to properly split changes between
openstack and their own projects.

Proposed change
===============

A really good sample is the way jenkins-job-builder is doing
it. It's splitting all the config file between directories
and files, each one with their own purpose, that allows to
properly isolate projects and makes easier to understand it.
Ideally Zuul should behave the same way, by recursively
digging into a directory, lookig for all the yaml config files,
and merging that together to produce the resulting
configuration.

Zuul layout file is composed of several main sections: includes,
pipelines, project-templates, jobs and projects. Ideally zuul
needs to be recursively iterating over directory provided and
read all yaml files. It will be adding entries to each section
according to the content of the files. So each pipeline,
projects, etc... will be incrementally adding entries as we
progress iterating over directorise and files.
Problem is when some duplicate is found. What about if several
files provide two entries for same pipeline, or same project?
I think that simply making zuul fail and alert about the
problem will be enough, so we do not allow conflicting
entries. Another option could be having a preferences system,
based on the level/name of files, so contents on a file
01-layout.yaml take precedence over files on 99-layout.yaml.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

Work Items
----------

* Allow zuul to grab a directory instead of a single file to
  be used as the configuration source.
* Allow zuul to recurse this directory to grab all the
  relevant yaml files.
* Make zuul check conflicts in provided yaml files / Merge
  depending on preferences system.
* With all the bits of configuration, produce a final one
  that will be used by Zuul to operate.

