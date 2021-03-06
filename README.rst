===============
elastic-recheck
===============

"Use ElasticSearch to classify OpenStack gate failures"

* Open Source Software: Apache license

Idea
----

Identifying the specific bug that is causing a transient error in the gate is
difficult. Just identifying which tempest test failed is not enough because a
single tempest test can fail due to any number of underlying bugs. If we can
find a fingerprint for a specific bug using logs, then we can use ElasticSearch
to automatically detect any occurrences of the bug.

Using these fingerprints elastic-recheck can:

* Search ElasticSearch for all occurrences of a bug.
* Identify bug trends such as: when it started, is the bug fixed, is it getting
  worse, etc.
* Classify bug failures in real time and report back to gerrit if we find a
  match, so a patch author knows why the test failed.

queries/
--------

All queries are stored in separate yaml files in a queries directory at the top
of the elastic-recheck code base. The format of these files is ######.yaml
(where ###### is the launchpad bug number), the yaml should have a ``query``
keyword which is the query text for elastic search.

Guidelines for good queries:

- Queries should get as close as possible to fingerprinting the root cause. A
  screen log query (e.g. ``tags:"screen-n-net.txt"``) is typically better than
  a console one (``tags:"console"``), as that's matching a deep failure versus
  a surface symptom.

- Queries should not return any hits for successful jobs, this is a sign the
  query isn't specific enough. A rule of thumb is > 10% success hits probably
  means this isn't good enough.

- If it's impossible to build a query to target a bug, consider patching the
  upstream program to be explicit when it fails in a particular way.

- Use the 'tags' field rather than the 'filename' field for filtering. This is
  primarily because of grenade jobs where the same log file shows up in the
  'old' and 'new' side of the grenade job. For example,
  ``tags:"screen-n-cpu.txt"`` will query in ``logs/old/screen-n-cpu.txt`` and
  ``logs/new/screen-n-cpu.txt``. The ``tags:"console"`` filter is also used to
  query in ``console.html`` as well as tempest and devstack logs.

- Avoid the use of wildcards in queries since they can put an undue burden on
  the query engine. A common case where wildcards are used and shouldn't be are
  in querying against a specific set of ``build_name`` fields, e.g.
  ``gate-nova-python26`` and ``gate-nova-python27``. Rather than use
  ``build_name:gate-nova-python*``, list the jobs with an ``OR``. For example::

   (build_name:"gate-nova-python26" OR build_name:"gate-nova-python27")

When adding queries you can optionally suppress the creation of graphs
and notifications by adding ``suppress-graph: true`` or
``suppress-notifcation: true`` to the yaml file.  These can be used to make
sure expected failures don't show up on the unclassified page.

In order to support rapidly added queries, it's considered socially acceptable
to approve changes that only add 1 new bug query, and to even self approve
those changes by core reviewers.

Note that old queries which are no longer hitting in logstash and are
associated with fixed or incomplete bugs are routinely deleted. This is to keep
the load on the elastic-search engine as low as possible when checking a job
failure. If a bug marked as Incomplete does show up again, the bug should be
re-opened with a link to the failure and the e-r query should be restored.

Adding Bug Signatures
---------------------

Most transient bugs seen in gate are not bugs in tempest associated with a
specific tempest test failure, but rather some sort of issue further down the
stack that can cause many tempest tests to fail.

#. Given a transient bug that is seen during the gate, go through `the logs
   <http://logs.openstack.org/>`_ and try to find a log that is associated with
   the failure. The closer to the root cause the better.

   Note that queries can only be written against INFO level and higher log
   messages. This is by design to not overwhelm the search cluster.

#. Go to `logstash.openstack.org <http://logstash.openstack.org/>`_ and create
   an elastic search query to find the log message from step 1. To see the
   possible fields to search on click on an entry. Lucene query syntax is
   available at `lucene.apache.org
   <http://lucene.apache.org/core/4_0_0/queryparser/org/apache/lucene/queryparser/classic/package-summary.html#package_description>`_.

#. Tag your commit with a ``Related-Bug`` tag in the footer, or add a comment
   to the bug with the query you identified and a link to the logstash URL for
   that query search.

   Putting the logstash query link in the bug report is also valuable in the
   case of rare failures that fall outside the window of how far back log
   results are stored. In such cases the bug might be marked as Incomplete
   and the e-r query could be removed, only for the failure to re-surface
   later. If a link to the query is in the bug report someone can easily
   track when it started showing up again.

#. Add the query to ``elastic-recheck/queries/BUGNUMBER.yaml``
   (All queries can be found on `git.openstack.org
   <https://git.openstack.org/cgit/openstack-infra/elastic-recheck/tree/queries>`_)
   and push the patch up for review.

You can also help classify `Unclassified failed jobs
<http://status.openstack.org/elastic-recheck/data/uncategorized.html>`_, which is
an aggregation of all failed gate jobs that don't currently have elastic-recheck
fingerprints.

Future Work
------------

- Move config files into a separate directory
- Make unit tests robust
- Add debug mode flag
- Expand gating testing
- Cleanup and document code better
- Add ability to check if any resolved bugs return
- Move away from polling ElasticSearch to discover if its ready or not
- Add nightly job to propose a patch to remove bug queries that return
  no hits -- Bug hasn't been seen in 2 weeks and must be closed
