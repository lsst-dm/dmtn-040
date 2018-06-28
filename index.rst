:tocdepth: 1

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Currently, this is a living document. It serves as a central repository of
   information addressing operational concerns related to usage of Pegasus WMS
   for the LSST Batch Production Services.  Its content may be changed and/or
   expanded over time.


Overview
========

`Pegasus`_ is a workflow manager system (WMS) being developed and mainatined at
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

The labels for the jobs in the workflow are specified by associated pegasus
profile keys with the jobs during the DAX generation process.  The user can
choose which profile key to use for labeling the workflow. By default, it is
assumed that the user is using the Pegasus profile key **label** to associate
the labels.  Other key may be used by setting **pegasusu.clusterer.label.key**
to a desired value.

FAQ
^^^

Can we cluster jobs automatically?
""""""""""""""""""""""""""""""""""

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
"""""""""""""""""""""""""""""""""""""""""""""""""

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
""""""""""""""""""""""""""""""""""""""

Yes, operators can use label based clustering to group jobs in a workflow in an
arbitrary manner.

Can jobs in a cluster be executed in parallel?
""""""""""""""""""""""""""""""""""""""""""""""

Yes, constituent jobs in a clustered jobs can be executer in parallel either on
a single, multi CPU/core node using **pegasus-cluster** or across multiple
nodes using MPI based management tool **pegasus-mpi-cluster**.


Can we restart jobs in a clustered job?
"""""""""""""""""""""""""""""""""""""""

Probably no. A clustered job is a single HTConodor job and that is maximal
"resolution" DAGMan and hence Pegasus operates on.

Can we monitor constituent jobs in a clustered job?
"""""""""""""""""""""""""""""""""""""""""""""""""""

Pegasus documentations claims it is possible to monitor individual jobs within
a clustered job.  However, this functionality is not often used and requires a
shared filesystem on the compute site (see `this`__ thread on
``pegasus-users@isi.edu``).

.. __: http://mailman.isi.edu/pipermail/pegasus-users/2018-April/000713.html


.. _DMTN-025: https://dmtn-025.lsst.io/
.. _Pegasus: https://pegasus.isi.edu/
