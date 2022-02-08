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

- Batch execution with BPS may "cluster" multiple quanta into a single "job" to e.g. avoid overheads in acquiring compute nodes or allow local scratch space to be used for some intermediate datasets.
  We currently have no way to store job-level provenance in a data repository, except by denormalizing it into per-quantum datasets.

Saving complete quantum provenance
==================================

Representing executed quanta in Registry
----------------------------------------

Proposed tables for storing quantum-based provenance in the registry database are shown in the diagram below [`source <https://dbdiagram.io/d/61fff3cc85022f4ee5479e62>`_]:

.. image:: /_static/tables.png
   :target: https://dbdiagram.io/d/61fff3cc85022f4ee5479e62
   :alt: registry provenance tables

This includes two new fields for the existing ``dataset`` table:

``quantum_id``
  The quantum that produced this dataset (may be null).
  Because a quantum can be produced by at most one quantum, we do not need a separate join table for this relationship.
  If provenance data is loaded into the database as proposed in :ref:`the next section <recording-provenance>`, this information will be available when the dataset is first inserted, so no separate update is needed.

``actually_produced``
  Whether this dataset was actually written to any datastore (defaults to `true`).
  A dataset for which ``actually_produced`` is ``false`` should never appear in *any* ``Datastore``, and the dataset record exists only because the dataset was predicted to exist when a ``QuantumGraph`` was generated.
  A value of ``true`` does not guarantee continued existence in any ``Datastore``, but this this flag is still expected to provide an important optimization in queries for datasets that must be present in ``Datastores``, especially since most other reasons for dataset nonexistence allow checks to be implemented more efficiently by policy shortcuts (e.g. "this ``Datastore`` cannot have this ``PVI`` because it never has any ``PVIs``").

The new ``task``, ``quantum``, and ``quantum_tags_*`` tables behave analogously to the ``dataset_type``, ``dataset``, and ``dataset_tag_*`` tables.
Like datasets, quanta may be associated with collections, are "owned" by exactly one RUN collection, and there may be only one quantum with a particular data ID and task in each collection.
Tasks are not associated with a particular collection, and are uniquely identified by their label.

.. note::

   It may make more sense to make task labels non-unique, except within a particular collection, in order to allow the label to have different meanings in different pipelines or change its definition more easily over time.
   This would analogous to the `RFC-804 <https://jira.lsstcorp.org/browse/RFC-804>`_ proposal for dataset type non-uniqueness, however, and as long as dataset type names *are* globally unique, and task labels are used to produce dataset type names (e.g. ``<label>_metadata`` or ``<label>_config``), there's relatively little to be gained from making label uniqueness apply only within a collection.
   The definition of those dataset types (which must be globally unique) would still effectively force global label uniqueness.

Links between quanta and their inputs datasets are stored in the ``quantum_inputs`` table.
This is a standard many-to-many join table with one extension: the ``actually_used`` flag.
This may be set to ``false`` by tasks to indicate that the input dataset was not used *at all*, i.e. running the quantum without the dataset would have no effect on the results.
Tasks that do not opt-in to this fine-grained reporting will be assumed to use all inputs given to them.
Note that there are actually three possible states for a quantum input dataset relationship, when the ``dataset.actually_produced`` flag is considered as well:

- a dataset is a "predicted" input if a ``quantum_input`` row exists at all;
- a dataset is also an "available" input if ``dataset.actually_produced`` is ``true``;
- a dataset is also an "actual" input if ``quantum_input.actually_used`` is ``true``.

A dataset may not have ``quantum_input.actually_used`` if ``dataset.actually_produced`` is ``false``.
Note that these states build on each other; we say e.g. "predicted *only*" when an input is not available (and by extension, not actual).

Open questions and variants
^^^^^^^^^^^^^^^^^^^^^^^^^^^

InitInput and InitOutput datasets
"""""""""""""""""""""""""""""""""

This schema does not provide a dedicated solution for associating tasks with the InitInput and InitOutput datasets they may consume and produce during construction.
Our preferred solution is to introduce a special "init" quantum for each task.
This quantum's inputs would be the InitInputs for the task, and its outputs would be the InitOutputs for the task.
It would have an empty data ID instead of the usual dimensions for the task.

This approach works best if it is reflected in the in-memory ``QuantumGraph`` data structure and the execution model; these special init quanta would be executed prior to the execution of any of their task's usual quanta, which would be handled naturally by considering all task InitInputs to also be regular inputs of the task's regular quanta.
This is actually more consistent with how BPS already treats the init job as just another node in its derived graphs, but with one init node per task, rather than one init node for the whole submission.
Having one init quantum per task hints at a solution to another problem: if we write a different software version dataset for each (per-task) init quantum, instead of one for the entire RUN, each can be handled as a regular, write-once dataset, instead of needing to simulate update-in-place behavior.

No predicted-only datasets
""""""""""""""""""""""""""

Instead of adding the ``dataset.actually_produced`` column to the ``dataset`` table, we could ignore predicted-only datasets entirely in provenance.
This simplifies the schema and avoids the problem how to query only for datasets that were actually produced.
These predictions will still appear in ``QuantumGraph`` objects prior to execution, so by dropping them we make it impossible to reconstruct from provenance the full graph that was e.g. submitted to a batch system - but reproducing ``QuantumGraph`` generation is already not a guarantee of the system, because it depends on inputs (e.g. collections) that change over time.
We would retain the ability to exactly re-execute any particular quantum and reproduce all output datasets however, because by definition these datasets are those that cannot have affected the results of the processing that did occur.

Reverse-lookups for special datasets
""""""""""""""""""""""""""""""""""""

The provenance tables in the original proposal do not contain any foreign-key columns for the datasets that store important provenance, such as software versions, task configuration, quantum metadata, or quantum logs.
Instead, these connections rely on dataset type conventions and RUN collection membership, e.g. "the configuration for the task ``<label>`` is in the ``<label>_config`` dataset in the same RUN collection," essentially the same as it is now.
This avoids circular dependencies between foreign keys (which require an ``UPDATE`` after each ``INSERT`` to fully populate), and it allows the set of provenance-relevant datasets to evolve without schema changes (provided some in-code scheme exists for remembering the historical conventions for provenance dataset types).

It also isn't straightforward to just add dataset foreign key columns to these tables in some cases: the ``task`` table as defined here has rows that are not tied to any particular RUN collection, and hence can't hold a reference to the config dataset.

.. _recording-provenance:

Recording provenance during execution
-------------------------------------

Avoiding per-dataset or per-quantum communication with a central SQL database is absolutely critical for at-scale execution with our middleware, so the provenance described above will need to be saved to files at first and loaded into the database later.

Most of the information we need to save is already included in the ``QuantumGraph`` produced prior to execution, especially if we include UUIDs for its predicted intermediate and output in the graph at construction.
We are already planning to do this for other reasons, as described in the `"Quantum-backed butler" proposal in DMTN-177 <https://dmtn-177.lsst.io/#limited-quantum-backed-butler>`_.
Always saving the graph to a managed location during any kind of execution (not just BPS) is thus a key piece of being able to load provenance into the database later.

The remaining information that is only available during/after execution of a quantum is

- timing and host fields for the ``quantum`` table;
- a record of which predicted outputs were ``actually_produced``;
- a record of which predicted inputs were ``actually_used``.

These can easily be saved to a file (e.g. JSON) written by the quantum-execution harness, and here the design ties again into the quantum-backed butler concept, which also needs to write per-quantum files in order to save datastore record data.
Just like the provenance we wish to save here, the eventual home of those datastore records is the shared ``Registry`` database, so it is extremely natural to save them both to the same files, and upload provenance when the datasets themselves are ingested in bulk after execution completes.

In fact, these per-quantum files may also help solve yet another problem; as described in `DMTN-213 <https://dmtn-213.lsst.io/>`_, our approach to multi-site processing and synchronization will probably involve metadata files that are transferred along with dataset files by Rucio, in order to ensure enough information for butler ingest is available from files alone.
These provenance files could easily play that role as well.

The quantum-backed butler design is a solution to a problem unique to at-scale batch processing, so writing the ``QuantumGraph`` and provenance files to BPS-managed locations (such as its "submit" directory) there is completely fine.
That's not true for provenance, which we want to work regardless of how execution is performed.
This is related to the long-running middleware goal of better integrating ``pipetask`` and BPS.
It will probably also involve carving out a third aspect of butler (a new sibling to ``Registry`` and ``Datastore``) for ``QuantumGraph`` and provenance files, because

- like ``Datastore``, this aspect would be backed by files in a shared filesystem or object store (but not necessarily the same filesystem or bucket as an associated ``Datastore``);
- like ``Registry``, the new aspect would provide descriptive and organizational metadata for datasets, rather than hold datasets themselves, and after execution its content would be completely loaded into the ``Registry``.

The full high-level design will be the subject of a future technote, but an early sketch can be found `in Confluence <https://confluence.lsstcorp.org/display/DM/Saving+per-Quantum+provenance+and+propagating+nothing-to-do+cases%2C+and+The+Future>`_.

Recording provenance only when using BPS (and relying on it to manage the ``QuantumGraph`` and provenance files) in the interim seems like a good first step.
Extending the design to include non-BPS processing may take time, but we do not anticipate it changing what happens at a low level or the appearance of persisted provenance information.

Interfaces for querying quantum provenance
------------------------------------------

Given the similarity between quanta and datasets in terms of table structure, a ``Registy.queryQuanta`` method analogous to ``queryDatasets`` provides a good starting point for provenance searches::

  def queryQuanta(
      self,
      label: Any,
      *,
      collections: Any = None,
      dimensions: Iterable[Dimension | str] | None = None,
      dataId: DataId | None = None,
      where: str | None = None,
      findFirst: bool = False,
      bind: Mapping[str, Any] | None = None,
      check: bool = True,
      with_inputs: Iterable[DatasetRef] | None = None,
      with_outputs: Iterable[DatasetRef] | None = None,
      **kwargs: Any,
  ) -> QuantumQueryResults:
      ...

Most arguments here are exactly the same as those to ``queryDatasets``,
with the dataset type argument replaced by a label expression identify the tasks, and two new arguments to constrain the query on particular input or output datasets.
Like ``queryDatasets``, the return type would be a lazy-evaluation iterable, with convenience methods for conversion to ``QuantumGraph`` instance; this type could also be returned by a new ``DataCoordinateQueryResults.findQuanta`` method to more directly find quanta from a data ID query (as the ``findDatasets`` does for datasets).

This interface does not provide enough functionality for most provenance queries, however - it just finds all quanta matching certain criteria, regardless of their relationships - so it is best considered way to obtain a starting point.
For those, we envision an operation that starts with a set of quanta and traverses the graph according to certain criteria, querying the database as necessary (perhaps once per task or dataset type) and returning matching quanta as it goes::

  def traverseQuanta(
      self,
      initial: Iterable[Quantum],
      forward_while: Optional[str | TraversalPredicate] = None,
      forward_until: Optional[str | TraversalPredicate] = None,
      backward_while: Optional[str | TraversalPredicate] = None,
      backward_until: Optional[str | TraversalPredicate] = None,
  ) -> Iterable[Quantum]:
      ...

Traversal could proceed forward (in the same direction as execution) or backward (from outputs back to inputs) or both.
The criteria for which quanta to traverse and return are encoded in the four predicate arguments, which are *conceptually* just boolean functions on a quantum::

  class TraversalPredicate(ABC):

      @abstractmethod
      def __call__(self, quantum: Quantum) -> bool:
          ...

Traversal would be terminated in a direction whenever a ``while`` predicate evaluates to `False` (without returning that quantum) or whenever the corresponding ``until`` predicate evaluates to `True` (which does return that quantum).

This simple conceptual definition of the predicate may not be possible in practice for performance reasons; traversal actually involves database queries, and while we can perform some post-query filtering in Python, we want most of the filtering to happen in the database.
In practice, then, we may need to define an enumerated library of ``TraversalPredicates``, and perhaps define logical operations to combine them, restricted to what we can translate to SQL queries.
Most common provenance queries could be satisfied by the following predicates and their negations (even if they cannot be combined):

- whether the quantum's task label is in a given set;
- whether any input or output dataset type is in given set;
- whether any input or output dataset UUID is in a given set.

It is worth noting here that the ``Quantum`` and ``QuantumGraph`` objects returned here are not necessarily the same types as those used prior to execution; execution adds more information that we want the provenance system to be able to return.
Whether to actually use different types involves a lot of classic software design tradeoffs involving inheritance and container classes, and resolving it is beyond the scope of this document.
If we do use different types, one of the most important operations on the "executed" forms will of course be transformation to a "predicted" form for reproduction.

Intentionally inexact reproduction
----------------------------------

TODO:

- Given existing QG, user wants to make some modifications and get a similar QG.
- Change datasets by searching a new collection search path, a new repo, or even the previous collections (since they may have changed), keeping data IDs and dataset types.
- Prune out inputs not actually used or outputs not actually produced (recursing to quanta).
- Change configuration, assuming or asserting that this does not change the connections.
- Change software versions, assuming or asserting that this does not change the connections.

Mapping to the IVOA provenance model
------------------------------------

Our quantum-dataset provenance model has a straightforward mapping to the  IVOA provenance model :cite:`2020ivoa.spec.0411S`, which is also based on directed acyclic graph concepts.
Our "dataset" corresponds to IVOA's "Entity", and our "quantum" corresponds to IVOA's "Activity".
The fields of these concepts in the IVOA have fairly obvious mappings to the fields of our schema (unique IDs, names of dataset types and tasks, execution timespans).
One important field that may be slightly problematic is the Entity's "location" field, which might *usually* map to a ``Datastore`` URI, but cannot in general, because our datasets may not have a URI, or may have more than one.

The IVOA terms are more general, and we may also want to map other concepts to them as well (e.g. a BPS job may be considered another kind of Activity, and a RUN collection could be another kind of Entity).
But none of the other potential mappings are as clear-cut or as useful as quantum-Activity and dataset-Entity.

There are two natural relationships defined by IVOA between an Activity and an Entity, which map directly to the kinds of edges in our ``QuantumGraph``:

- an Activity "Used" an Entity: a quantum has an "input" dataset;
- an Entity "WasGeneratedBy" an Activity: a dataset is an "output" of a quantum.

These relationships have a "role" field that is probably best populated with the string name used by a ``PipelineTask`` to refer to the ``Connection``.
Because this role will be the same for all relationships between a particular dataset type and task, it also makes sense to use IVOA's "UsedDescription" and "GeneratedDescription" classes to define these roles in a more formal, reusable way.
IVOA recommends certain predefined values be used for those descriptions when they apply (e.g. "Calibration" as a "UsedDescription"), which could be identified by configuration that depends on the pipeline definition.

IVOA also defines a "WasInformedBy" relationship between two Activities and a "WasDerivedFrom" relationship between two Entities.
These may be useful in collapsed views of the ``QuantumGraph`` in which datasets or quanta are elided, but in our case would always be computed from the Activity-Entity/quantum-dataset relationships, rather than meaningful in their own right.

IVOA has no direct counterpart to our "predicted" vs. "available" vs "actual" categorization of quantum-dataset relationships.
Because a relationship can have at most one associated "GeneratedDescription" or "UsedDescription", we cannot use one set of description types for the role-like information and another for this categorization.
It seems best to simply leave this out of the IVOA view of our provenance, and possibly limit the IVOA view to "actual" relationships, since it is not expected to play a role in actually reproducing our processing.

We currently have no concept that maps to IVOA's "Agent" or any of its relationships.

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

.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
