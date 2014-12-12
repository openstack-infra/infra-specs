::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Test Metrics Database
=====================


https://storyboard.openstack.org/#!/story/156

Using subunit2sql store data regarding gating test runs in a SQL database to
enable longer term analytics on testing in the gate. Also, construct a
dashboard for the stored data to visualize the gating job trends over the period
of time for which there is data in the DB.

Problem Description
===================

Right now we capture test run artifacts and archive them for roughly 1 release
cycle on logs.o.o. In addition logstash provides 10 days of API queryable
information about the test runs. This has proved invaluable for both
ascertaining the current status of the gate and debugging failures. However,
due to the lack of long term data being available there is an inability to
perform analysis to categorize the long term trends in our test suite.

This specifically becomes an issue when we want to optimize how we're using
tempest in the gate. For example, if we wanted to trim down the tests we're
gating on to maintain a timing quota would be very difficult because we don't
have an API interface to use which we can figure out which tests never fail, or
which tests on average take the longest to run.


Proposed Change
===============

Using the subunit2sql library, which was recently added to openstack-infra, a
new post processing job (similar to how logs are processed for logstash) is
added to read the subunit file from run, and push that to a SQL DB setup for
storing the data. The SQL DB should be put on a separate server, possibly using
Trove to provision it. By having the data available in DB allows for API access
to enable interesting analytics to be performed. However, to enable an
environment conducive to developing tooling around performing this analysis,
public read-only access to the DB will be needed; mysql-proxy will be used to
enable a public DB endpoint.

On top of the DB a new dashboard will be created that visualises the data
stored in the DB. It's basic functionality will be to show an analysis
of the performance and stability of individual tests over time, as well as
showing the broad long term trends on the jobs over the whole stored history.
(ie, graphs of test success and failures counts, total run time for each run
type over the whole stored history)

Additionally, storing the test timing data in a database will allow us to
eventually use the timing data with testr to perform scheduler optimizations.
Pre-seeding the timing data into testr is something which has been discussed
previously, but there was not a good method of doing this before. However, this
is dependent on adding a new SQL repository to testrepository based on
subunit2sql.

Alternatives
------------

An alternative approach for the data collection would be to build a log
scraping tool that would scrape the archived logs in order to collect the
required information from the logs instead of the subunit file directly. However
that seems like an error prone and less efficient approach.

As alternative to using a SQL DB a different data storage mechanism could be
used. For example hadoop, a nosql db, and graphite have all been considered as
alternatives. However, because the parts of the subunit data we need are
already highly structured using SQL makes sense for the storage. Using SQL also
enables us to exploit other tooling to make the task simpler.

Implementation
==============

Assignee(s)
-----------

Matthew Treinish <mtreinish@kortar.org>
Clark Boylan <cboylan@sapwetik.org>

Work Items
----------

* Add server for SQL DB and maybe the dashboard (possibly using trove)
* Add post-processing workers (similar to how logstash does it) to parse the
  subunit output from the run and push it to the DB using subunit2sql
* Write a dashboard script to parse the database to visualize the metrics
  we want from the database
* Start running the dashboard on top of the DB
* Add a SQL repository type using subunit2sql to testrepository
* Switch the dsvm jobs to use the SQL repository in testr to access the
  historical test data

Repositories
------------

The only required repository is subnunit2sql which has already been created.
However the dashboard script will need to be stored somewhere, it will be too
specific to the CI environment that it probably doesn't belong in the
subunit2sql repo. Depending on the size we can keep it directly in
openstack-infra/config. However, if it ends up be sufficiently large we might
need to create separate repository to store it.

Servers
-------

So to enable this a new SQL database is required. While a new server is not
strictly required to do this, based on the estimated load caused by running
subnunit2sql for each test run in the gate it probably makes the most sense.
It makes sense to use trove here to spin up a database to use. Another
consideration is to enable public read access to the database which will be
need to allow development on top of the data, as well as incorporating the data
into the testr scheduler on gate slaves.

DNS Entries
-----------

A DNS entry will need to be created the new DB server so that we can have the
post-processing job send the results to it. Additionally, the dashboard view
will need to be given an address, however a separate DNS entry probably isn't
required for that since a new path on an existing webserver is probably
sufficient. (ie status.o.o/test-stats) There is also going to be a mysql-proxy
setup needed to enable public read access to the database, which also will need
to be addressable.


Documentation
-------------

subunit2sql is lacking substantial documentation right now. This documentation
will include both operational instructions for subunit2sql as well as being a
DB API and schema guide. Additionally, a new doc for setting up the test metric
workflow shoud also be added.

Security
--------

There shouldn't be any additional security implications besides the risks
associated with running a new server and SQL DB. Care also needs to be taken
when Storing the credentials for access to the DB server for the post
processing jobs. Setting up a mysql-proxy to allow public read access to the
DB does open up a new potential attack entry-point.

Testing
-------

There shouldn't be any additional testing requires. Both subunit2sql and
the dashboard should have there own unit testing.

Dependencies
============

- This BP will be the first use of subunit2sql in the infra workflow
- mysql-proxy will also need to be installed and configured
