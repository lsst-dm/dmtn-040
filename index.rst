:tocdepth: 1

.. sectnum::

.. note::

   Currently, this is a living document. It serves as a central repository of
   information addressing operational concerns related to usage of Pegasus WMS
   for the LSST Batch Production Services.  Its content may be changed and/or
   expanded over time.


Overview
========

`Pegasus`_ is a workflow manager system (WMS) being developed and maintained at
USC Information Sciences Institute.  It was one of the "winners" of the survey
made in August, 2016 regarding existing workflow management systems and their
possible use in the LSST Batch Processing Service (see `DMTN-025`_).  Though
the survey  took many of their aspects into the consideration, many more
advanced operational concerns fell outside of its scope and were not addressed.
This document attempts to fill this gap.

Optimizing workflows
====================

Job clustering
--------------

There is an inevitable overhead when running jobs with help of Pegasus, usually
60 seconds (`ref`__).  For short jobs running for few seconds it can be
significantly longer than job's execution time.  Hence, Pegasus allows to
cluster such computational jobs into a single Pegasus job.

Pegasus supports three clustering methods:

- level based horizontal clustering,
- level based runtime,
- label based.

Regardless of clustering method, jobs belonging to a cluster are executed in a
single HTCondor job.

If the jobs in a cluster are not independent, they will be executed in their
topological order.

Jobs in a cluster can be executed either sequentially or in parallel on a
single node. Large, parallel multi-node jobs can also be handled by MPI based
task management tool **pegasus-mpi-cluster**.

By default, Pegasus tries to get as much work done as possible. It means that
it does not stop execution even if one of the clustered fails.  This behaviour
can be changed by setting property

.. code::

   pegasus.clusterer.job.aggregator.seqexec.firstjobfail=false

It will makes Pegasus stop on the first failing job.

Out of the box, Pegasus does not monitor individual jobs within a job cluster.
Though its documentation claims it is possible to activate it by setting
property

.. code::

   pegasus.clusterer.job.aggregator.seqexec.log=true

it turned out this functionality is subject to certain restrictions (see
`this`__ thread on Pegasus mailing list.

.. note::

   More descriptions of other properties controlling job clustering can be
   found `here`__.
   

.. __: https://pegasus.isi.edu/documentation/job_clustering.php
.. __: http://mailman.isi.edu/pipermail/pegasus-users/2018-April/000713.html
.. __: https://pegasus.isi.edu/documentation/properties.php#job_clustering_props

Horizontal clustering
^^^^^^^^^^^^^^^^^^^^^

From Pegasus manual: 

    In case of horizontal clustering, each job in the workflow is associated
    with a level. The levels of the workflow are determined by doing a
    modified Breadth First Traversal of the workflow starting from the root
    nodes. The level associated with a node, is the furthest distance of it
    from the root node instead of it being the shortest distance as in normal
    BFS.

The granularity of the horizontal clustering may be controlled by two Pegasus
profiles:

**clusters.size**
    Denotes how many jobs need to be merged into a single clustered job.

**clusters.num**
    Denotes how many clustered jobs does the user want to see per level per
    site.

In the case, where both the properties are set, the **clusters.num** value
supersedes the **clusters.size** value.

Runtime clustering
^^^^^^^^^^^^^^^^^^

In case of runtime clustering, jobs in the workflow are associated with a
level. The levels of the workflow are determined in the same manner as in
horizontal clustering. However jobs are grouped together such that all
clusters have similar runtime.

Runtime clustering supports two modes of operation:

1. Clusters jobs together such that the clustered job's runtime does not exceed
   a user specified values set with **clusters.maxruntime** profile.
2. Clusters all jobs into a fixed number of clusters, specified by the profile
   **clusters.num**, such that the runtimes of the clustered jobs are similar.

If both of **clusters.maxruntime** and **clusters.num** are specified,
**clusters.num** profile will be ignored by the clustering engine.
   
.. note::

   To use runtime clustering, the Pegasus profile **runtime**, specifying
   expected runtime (in seconds), must be set for each job.

Label based clustering
^^^^^^^^^^^^^^^^^^^^^^

Again, from Pegasus manual:

    In label based clustering, the user labels the workflow. All jobs having
    the same label value are clustered into a single clustered job. This allows
    the user to create clusters or use a clustering technique that is specific
    to his workflows. If there is no label associated with the job, the job is
    not clustered and is executed as is.

The labels for the jobs in the workflow are specified by associated Pegasus
profile keys with the jobs during the DAX generation process.  The user can
choose which profile key to use for labeling the workflow. By default, it is
assumed that the user is using the Pegasus profile key **label** to associate
the labels.  Other key may be used by setting **pegasusu.clusterer.label.key**
to a desired value.

FAQ
---

Can we cluster jobs automatically?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes, jobs can be clustered automatically. Pegasus provides two clustering
methods: horizontal and label based clustering.

Horizontal clustering operates in one of two modes: job count and runtime
based. Regardless of the mode it is only able to cluster jobs

#. of the same type (i.e. referring to the same logical transformation), and
#. at the same level of the workflow.

In label based clustering jobs with the same label are put in the same
clustered job. This method allows to aggregate jobs across levels of the
workflow.


What information needs to be provided to Pegasus?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Specific information required by Pegasus to cluster jobs varies depending on
the method of choice.  In general, an operator needs to set relevant Pegasus
properties (options affecting the whole system) and profiles (options
controlling behavior of individual jobs).  The properties and profiles relevant
to each clustering method are described in detail `here`__.  An
operator needs also to specify clustering method during workflow planning (see
**--cluster** option of `pegasus-plan`__).  

.. __: https://pegasus.isi.edu/documentation/job_clustering.php
.. __: https://pegasus.isi.edu/documentation/cli-pegasus-plan.php

Can clustering be manually controlled?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes, operators can use label based clustering to group jobs in a workflow in an
arbitrary manner.

Can jobs in a cluster be executed in parallel?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes, constituent jobs in a clustered jobs can be executer in parallel either on
a single, multi CPU/core node using **pegasus-cluster** or across multiple
nodes using MPI based management tool **pegasus-mpi-cluster**.


Can we restart jobs in a clustered job?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Probably no. A clustered job is a single HTConodor job and that is maximal
"resolution" DAGMan and hence Pegasus operates on.

Can we monitor constituent jobs in a clustered job?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Pegasus documentations claims it is possible to monitor individual jobs within
a clustered job.  However, this functionality is not often used and requires a
shared filesystem on the compute site (see `this`__ thread on
``pegasus-users@isi.edu``).

.. __: http://mailman.isi.edu/pipermail/pegasus-users/2018-April/000713.html

Data management
===============

Data transfer
-------------

Pegasus does data management for the executable workflow. Based on entries in
the Replica Catalog it discovers locations of the input datasets and adds data
movement and registration nodes in the workflow to 

#. stage-in input data to the staging sites,
#. stage-out output data generated be the workflow to the final storage,
#. stage-in intermediate data between compute sites if required,
#. register the output data into the Replica Catalog.

If input data for a job already exists on a compute site, then it is possible
for Pegasus to symlink against that data.

It will also transfer executables to the compute sites if they are not
installed there (i.e. are marked as **STAGEABLE** in the Transformation
Catalog).

That approach allows it to run workflows in the following configurations:

* shared file system
* non-shared file system
* condor pool without shared file system.

Controlling data transfer
^^^^^^^^^^^^^^^^^^^^^^^^^

By default, Pegasus adds transfer jobs and cleanup jobs based on the number of
jobs at a particular level of the workflow.  For every 10 compute jobs on a
level of a workflow, one data transfer job is created.  Cleanup jobs are
similarly constructed with an internal ratio of 5.  However, a user may specify
the desired number of transfer jobs by relevant Pegasus profiles (see `this`__
page for details).

Pegasus uses a **transfer refiner** to decide how to distribute input and
output files among the existing transfer jobs.  It supports three different
transfer refiners:

#. **basic**: adds one stage-in and stage-out job per compute job of the
   workflow,
#. **balanced cluster**: does round robin distribution of files amongst
   stage-in and stage-out jobs per level of the workflow (default),
#. **cluster**: similar to **balanced cluster** but differs in the way how
   distribution of files happen across stage-in and stage-out jobs per level of
   the workflow, all the input files for a job get associated with a single
   transfer job.

Which transfer refiner to use is controlled by property
**pegasus.transfer.refiner**.

.. __: https://pegasus.isi.edu/documentation/transfer.php#data_movement_nodes

Data cleanup
------------

Pegasus planner adds data cleanup jobs to the executable workflow which are
responsible for removing files and directories during the workflow execution.
Number of the added cleanup jobs depends on the selected cleanup strategy:

- **none**: Disables cleanup altogether.
- **leaf**: The planner adds a leaf cleanup node per staging site that removes
  the directory created by the *create_dir* job in the workflow.
- **inplace**: The planner adds cleanup nodes per level of the workflow in
  addition to leaf cleanup nodes. Those nodes remove files no longer required
  during the execution.
- **constraint**: The planner adds cleanup nodes to constraint the amount of
  storage space used by the workflow, in addition to leaf cleanup nodes.  The
  nodes remove files no longer required during workflow execution enforcing
  imposed limits on disk usage. File sizes are read form the **size** flat in
  the DAX, or from a CSV file.

FAQ
---

What/when do files need to be listed in Pegasus replica catalog?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

During planning, Pegasus uses replica catalog to map logical filenames present
in the abstract workflow to physical filenames, i.e. actual resources.  Thus it
should contain all input files for the particular workflow.  Other files can be
registered in it later on during the execution of the workflow. It can be used
to limit data transfers in case of larger, hierarchical workflows.

When cleanup is done?
^^^^^^^^^^^^^^^^^^^^^

It depends on the selected cleanup strategy, see above. Be default, Pegasus
uses **inplace** cleanup strategy. It adds cleanup nodes per level of the
workflow in addition to leaf cleanup nodes removing the directory created by
Pegasus to run the workflow.

Will intermediate files be deleted to make room if not used for other nodes?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes, providing the either **inplace** or **constraint** cleanup strategy was
selected.

Dealing with failures
=====================

Executing workflows in a distributed environment inevitably lead to failures
either due to hardware issues, configuration errors, or faulty software.  In
general

A wrapper program **pegasus-kickstart** manages and monitors the execution of
jobs (`ref`__).

To determine a state of a finished job Pegasus uses a utility
**pegasus-exitcode** which it runs as the DAGMan postscript.  The utility
performs several checks (some optional) to find out whether a job failed or
not.  These checks are described on **pegasus-exitcode** man `page`__.

.. warning::

   If a job fails, materialized data are *left* in the staging site associated
   with a given compute site.

.. __: https://pegasus.isi.edu/documentation/cli-pegasus-kickstart.php
.. __: https://pegasus.isi.edu/documentation/cli-pegasus-exitcode.php

Retries
-------

Transient infrastructure failures such as a node being temporarily down can be
mitigated with **retries**, i.e. Pegasus will automatically try to run a job
again (once, by default). 

The desired number of retries can be set using a dedicated Pegasus profile.
For example, to set the number of retries for all jobs to 3 one need to set

.. code::

   dagman.retry 3

in Pegasus properties file.

Specifying number of retries per transformation, site, or job is also possible.

FAQ
---

What does Pegasus see as a failure?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Pegasus consider a job to be failed if the executable returned a non-zero exit
status, it did not produced expected output files, or exceeded specified
timeout.

Does output file get brought home?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

No, all materialized data are staged out to the staging area associated with a
compute site where they were created.

Can intermediate files be brought home on failure for debugging purposes?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes, materialized data are left in the staging site associated with a given compute site even when jobs fails.

Can we set the maximal number of retries?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes. DAGMan property **dagman.retry** can be used to tell Pegasus (or more
specifically the DAGMan) how many times it should attempt to rerun a given job. 

Can we retry to run a job on certain failures but not others?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

No.

Does Pegasus do the cleanup before a retry?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Pegasus wrapper **pegasus-kickstart** which manages and monitors the execution
of jobs on remote resources allows for the optional execution of *setup-*,
*pre-*, *post-*, and *cleanup-* subjobs (`ref`__). If cleanup-job is defined,
Pegasus will attempt to run for all failed jobs regardless of the exit status
of any other jobs and as such may be used to perform a cleanup before a retry.
On non-shared filesystems all compute jobs are executed using lightweight job
wrapper *PegasusLite* performs cleanup automatically.

.. __: https://pegasus.isi.edu/documentation/cli-pegasus-kickstart.php#SUBJOBS

Can Pegasus override a failure if we need to?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

No.

.. _DMTN-025: https://dmtn-025.lsst.io/
.. _Pegasus: https://pegasus.isi.edu/
