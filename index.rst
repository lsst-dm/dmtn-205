:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   The butler stores datasets in collections along with software versions and pipeline configuration, but it does not store any information about the datasets that were used to generate a specific dataset. This document will discuss the plan for integrating provenance into Butler, sufficient to be able to regenerate any given dataset. It will refer to the requirements for provenance gathering specified in DMTN-185.


Existing concepts and mechanisms for provenance
===============================================

Processing data with LSST middleware involves executing a directed acyclic graph (DAG) called a ``QuantumGraph``, where each node is either a ``Quantum``, a single execution of a single ``PipelineTask``, or a dataset managed by the ``Butler``.
The datasets produced by executing a ``QuantumGraph`` are written to a single ``RUN`` collection in the data repository, with any other collections that hold input datasets associated with that ``RUN`` collection by defining a new ``CHAINED`` collection that includes both inputs and outs.
A ``RUN`` collection is constrained to have at most one dataset of a particular dataset type and data ID; the latter may be empty, implying only one dataset of a particular dataset type per ``RUN``.
This system already provides support for limited provenance recording:

- Dataset types with empty data IDs can be used to store arbitrary information describing a ``RUN``, as long as it is acceptable to obtain that information via ``Butler.get`` (i.e. it is not needed as a constraint on future ``Registry`` database queries).
  This is already used to record the configuration of all ``PipelineTasks`` being executed and list of software names and versions, and a hook allows concrete ``PipelineTasks`` to store and retrieve their own ``RUN``-level content, such as the schemas of any catalogs they produce or consume, prior to execution.
  Any kind of execution harness (including but not limited to BPS implementations) can also store their own per-``RUN`` information by defining their own dataset types, without any need to modify lower-level middleware software.

- Dataset types with data IDs that match those of a particular ``PipelineTask's`` quanta can be use to store arbitrary information describing a quantum.
  This is already used to store ``Task`` metadata and (in at least some configurations) logs.

- The ``CHAINED`` collection provides a record of the input collections that were used.

- The ``QuantumGraph`` itself is a complete description of the processing that can be used to reproduce a ``RUN`` exactly, on any system with access to the same datasets and sufficiently similar hardware and software (with the latter verifiable via the per-``RUN`` software versions dataset).

The primary limitation in the current system is that the ``QuantumGraph`` is at most saved to a file outside the data repository (at the user's discretion), and hence much of the provenance we produce is often discarded.
It also only captures *predicted* inputs and outputs, which are in general a superset of the *actual* inputs consumed and outputs produced by quanta; this is sufficient for reproduction when starting everything from scratch, but harder to make use of when reproducing subsets or starting from a partial run.
Saving a more complete, as-executed ``QuantumGraph`` in the data repository has always been part of the high-level middleware design, and a more detailed design is the subject of the next section.

Before moving on, it is worth pointing out a few smaller issues with the provenance that we do record:

- The software versions we record may not extend far enough into OS- or container-level software to guarantee reproducibility.

- Because we do not assume that the complete set of relevant software packages can be enumerated, and instead rely (at least partially) on what Python packages are imported, running additional processing that brings in new imports against an existing output ``RUN`` collection requires us to update the versions dataset in place - an operation we otherwise prohibit in ``Butler`` (datasets are considered atomic and immutable, at least for writes) and simulate by deleting the old one and writing a new one.
  This is fragile but hard to fix, unless we just prohibit that kind of ``RUN`` collection extension or do enumerate all relevant software somehow up front.

- The approach of using datasets to record arbitrary provenance for higher-level tooling is currently extensible only to the extent that those datasets can use existing storage classes.
  New storage classes must currently be registered by editing configuration in the data repository or ``daf_butler`` manually.
  This is a problem for much more than just provenance, however (it also limits pipeline extensibility), and is something we hope to resolve.

- Using ``CHAINED`` collections to link input collections to output collections is often assumed to relate input and output datasets that share the same data ID (or have related data IDs).
  This *usually* works, at least as far as relating predicted input to predicted output datasets, but only because our ``QuantumGraph`` generation algorithm is based on data ID commonality; some other ``QuantumGraph`` generation algorithm that is equally valid for execution may not have this property, or may make important exceptions.
  It also fails when a ``CHAINED`` collection includes a "merge" of two output ``RUN`` collections that have conflicting datasets (i.e. they have same dataset type and data ID); dataset lookup then uses the order in which those ``RUN`` collections appear in the ``CHAINED`` collection to resolve the conflict, but this may not correspond to what was actually used as an input, and in fact more than one of the conflictig datasets may have been used as inputs (to different quanta).

Saving complete quantum provenance
==================================

Representing executed quanta in Registry
----------------------------------------

TODO.

Sketch the schema, describe predicted/available/actual input and predicted/actual output distinctions.
Plan to use UUIDs, even for predicted-only datasets.
Need to figure out how to work empty-data-ID datasets in.

Recording provenance during execution
-------------------------------------

TODO:

- Write predicted QG to files or Redis then write new per-quantum output files or extend info in Redis; consider this storage to be in repo, but in a third sibling to Registry and Datastore rather than either.
- Bring home provenance to Registry when job completes and datasets are also brought home.
- Defer most of this description to another technote.

Interfaces for querying quantum provenance
------------------------------------------

TODO:

- Support getting QG given (all optional): input DatasetRef/DatasetType/Task/Quantum, output DatasetRef/DatasetType/Task/Quantum, collections all datasets or quanta must be in.
- Need to think about whether QG is well-defined for all argument combinations, and what arguments really mean; "input" and "outputs" are only well-defined for linear graphs, but there are probably definitions with that limiting behavior we can assume.
- Need to have options for minimal (actual inputs/outputs only) or original (predicted inputs/outputs).
- Everything else we need to do is convenience methods on that QG?
- Need to be able to fetch that QG without having Tasks importable.

Intentionally inexact reproduction
----------------------------------

TODO:

- Given existing QG, user wants to make some modifications and get a similar QG.
- Change datasets by searching a new collection search path, a new repo, or even the previous collections (since they may have changed), keeping data IDs and dataset types.
- Prune out inputs not actually used or outputs not actually produced (recursing to quanta).
- Change configuration, assuming or asserting that this does not change the connections.
- Change software versions, assuming or asserting that this does not change the connections.


Addressing provenance working group recommendations
===================================================

TODO: Try to cover every middleware-relevant recommendation from DMTN-185 somewhere in this section.

Recommendations relevant to quantum provenance
----------------------------------------------

TODO:

- List recommendations from DMTN-185 that earlier sections address, call out subtleties.
- Call out REQ-PTK-005 (URIs in PipelineTask provenance) as something we won't do, at least not directly, in that you can ask for a URI given a UUID, but it doesn't make sense to put the URI in the provenance tables or even demand that all Datastores use URIs at all.

EFD-Butler linkage
------------------

TODO:

- If it lands in the headers, it *could* land in the exposure table.
- Everything else from EFD that goes in butler should be written or ingested as a per-exposure dataset.
- Could use an opaque-table datastore if we wanted to (someday) make it possible to include these things in Registry queries.
- Make sure this is consistent with DMTN-185.
- Make sure this is consistent with LDM-556.

Metrics linkage
---------------

TODO:

- If metrics are measured by PipelineTasks, linkage to everything else is done.
- Use opaque-table Datastore if we want them in Registry queries.
- Also consider (write-only?) InfluxDB datastore.
- Defer to other technote.

Saving provenance in dataset files
----------------------------------

TODO:

- Sketch hook on Formatter that is given metadata to write if it can and discard if it can't.
- On `put`, pass metadata to Formatter with UUID of the dataset, and best (conservative) guess at UUIDs of actual inputs to this quantum.
- THIS MIGHT DIFFER FROM FINAL ACTUAL INPUTS, because the quantum isn't necessarily done yet (though it often will be).  Or should we record predicted/available inputs instead to avoid discrepancy?

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
