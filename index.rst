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

.. _saving-complete-quantum-provenance:

Saving complete quantum provenance
==================================

.. _provenance-schema:

Representing executed quanta in Registry
----------------------------------------

Proposed tables for storing quantum-based provenance in the registry database are shown in the diagram below [`source <https://dbdiagram.io/d/61fff3cc85022f4ee5479e62>`__]:

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
   This would analogous to the `RFC-804 <https://jira.lsstcorp.org/browse/RFC-804>`__ proposal for dataset type non-uniqueness, however, and as long as dataset type names *are* globally unique, and task labels are used to produce dataset type names (e.g. ``<label>_metadata`` or ``<label>_config``), there's relatively little to be gained from making label uniqueness apply only within a collection.
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
We are already planning to do this for other reasons, as described in the `"Quantum-backed butler" proposal in DMTN-177 <https://dmtn-177.lsst.io/#limited-quantum-backed-butler>`__.
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

.. _querying-provenance:

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

Quantum-based provenance excels at exact reproduction of previous processing runs, but it can also be used - with some limitations - to re-run processing with intentional changes.

The most straightforward way to reprocess with changes is to create a completely new ``QuantumGraph``.
The software versions and task configuration stored in per-``RUN`` datasets can be combined with the special "init" quanta for each task to reconstruct the pipelines used in a ``RUN`` exactly, and of course other versions or configurations may be used instead as desired.
The input collections and data ID expression also typically provided as input to the ``QuantumGraph`` generation algorithm are not directly saved in our provenance schema, however, because after execution we intentionally do not draw any distinctions between quanta that may have originated in different graphs as long as their outputs land in the same ``RUN`` (and hence have the same software versions and configuration).
Higher level tools such as BPS or a campaign managements system may nevertheless save this information in their own datasets, along with any other graph-building or per-submission information relevant for those tools.

Changing software versions and configuration while keeping the input datasets and data IDs the same can be more directly accomplished using quantum-based provenance.
The former is what happens naturally when a different version of the codebase is used to fetch and run a stored ``QuantumGraph``, while the latter can easily be expressed as mutator methods on the ``QuantumGraph`` object (or perhaps arguments to the code that transforms an already-executed provenance graph into a ready-to-run predicted graph).
There is one large caveat: different versions and configuration can lead to different predictions for inputs and outputs for a quantum, and when applied to a full graph, this can result in some quanta being pruned either because they are no longer needed to produce desired target datasets or because it can be known in advance that they will have no work to do.
In some cases, it should logically expand the graph instead - but if we are starting from provenance quanta and are not re-running the ``QuantumGraph`` generation algorithm in full, we cannot in general add fundamentally new quanta, though we may be able to identify ways to do so for specific use cases in the future.
This depends on how our algorithm for ``QuantumGraph`` generation evolves; our current algorithm has essentially no way to incorporate existing quanta, but a long-planned (but long-delayed) new algorithm should be much more flexible in this regard.

Changing the input collections before re-executing a ``QuantumGraph`` obtained from provenance would work in much the same way, and would have very similar limitations in the sense that pruning the graph is straightforward but expanding it is not.
Unlike version and configuration changes, updating the graph to reflect new input collections involves querying the ``Registry``, and doing this efficiently (in particular, avoiding per-quantum queries) will make this more difficult to implement.

Changing the input data ID expression while starting from quantum provenance does not make sense in the same way; the right way to think of this is that the data ID expression is instead used in performing the provenance query to fetch those quanta.
It is worth noting that these queries are not affected by the original boundaries of the ``QuantumGraph`` objects used for production - a ``QuantumGraph`` obtained from provenance can include quanta from multiple ``RUN`` collections as well as multiple ``QuantumGraph``-generation submissions within a single ``RUN`` collection.

.. note::

   These multi-``RUN`` provenance ``QuantumGraphs`` cannot be translated one-to-one into runnable predicted ``QuantumGraphs``, as long as our execution model expects all writes to go into a single output ``RUN`` collection with consistent configuration and versions for all quanta within it.
   The most straightforward way to address this would be to make the process that translates provenance graphs into predicted graphs a fundamentally one-to-many operation, requiring users to run each per-``RUN`` predicted graph on its own.
   For unrelated reasons (e.g. RSP service outputs, user-defined processing, unusual processing for commissioning), we are considering expanding the data ID / dimensions system to allow custom data ID keys or relax the dataset type + data ID unique constraints, which would also open up new ways of saving configuration (e.g. multiple init quanta per ``RUN``, each with its own config datasets), and that may in enable execution of ``QuantumGraph`` objects with heterogeneous configuration.

.. _ivoa-mapping:

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

Italized bullets in this section are specific recommendations quoted from `DMTN-185 <http://dmtn-185.lsst.io>`__.
Middleware responses are in regular text below.

Recommendations relevant to quantum provenance
----------------------------------------------

- *[REC-SW-3] Software provenance support should include mechanisms for capturing the versions of underlying non-Rubin software, including the operating system, standard libraries, and other tools which are needed “below” the Rubin software configuration management system. The use of community-standard mechanisms for this is strongly encouraged.*
- *[REQ-WFL-005] Both the OS and the OS version must be recorded. This requirement may be met within the pipeline task provenance, but it is an upscope since currently, only the OS type is recorded.*

The existing software-version recording logic used in ``PipelineTask`` execution (implemented in `lsst.utils.packages`) does extend to non-Rubin software, and it does use community-standard mechanisms when possible.
But it also relies heavily on bespoke methods for obtaining versions for certain packages, and it is unclear to what extent this is historical (i.e. predating our use of ``conda`` for the vast majority of our third-party dependencies), as well as for deciding when to use various different community-standard mechanisms.
The code should at least be carefully reviewed for possible simplification and generalization.

- *[REQ-PTK-001] As planned, complete the recording of as-executed configuration for provenance.*

This is already implemented.

- *[REQ-PTK-002] As planned, complete the storage of the quantum graph for each executed Pipeline in the Butler repository.*

This will be satisfied by implementing the design described in [:ref:`provenance-schema`] and [:ref:`recording-provenance`].

- *[REQ-PTK-003] Code and command-line support for recomputing a specified previous data product based on stored provenance information should be provided.*

This will be satisfied by implementing the design described in
[:ref:`querying-provenance`].

- *[REQ-PTK-004] A study should be made on whether W3/VO provenance ontologies are a suitable data model either for persistence or service of provenance to users.*

This is discussed extensively in [:ref:`ivoa-mapping`].
To answer the question posed here directly, mapping to the IVOA data model is entirely suitable as one way to serve provenance to users, but it is slightly lossy and should neither be our way of storing quantum provenance internally nor the only way we serve this provenance to users.

- *[REQ-PTK-005] URIs (as well as DataIDs) should be recorded in Butler data collections.*

**This recommendation requires clarification before we can comment on its implementation in middleware.**

The butler ``Registry`` associates each dataset with a data ID and at least one collection, but collections do not store data IDs directly.
The butler's ``Datastore`` component **may** associate a dataset with one **or more** URIs, but this is not guaranteed in general, and even when present these URIs are not always sufficient information to be able to reconstruct an in-memory dataset.
These URIs may also use an internal form that is not usable by science users.

So, while there may well be (and will generally be) URIs involved in dataset storage within butler data repositories, they do not play an important role in provenance, and while some interpretations of this recommendation are trivially satisfied by the current middleware design, these interpretations are not consistent with a recommendation relevant to provenance, and it seems more likely that the intent was both more related to provenance and is probably not satisfiable by the middleware.

- *[REQ-WFL-001] Logs from running each quantum must be captured and made available from systems outside the batch processing system.*
- *[REQ-WFL-002] Any workflow level configuration and logs must be persisted and made available from systems outside the batch processing system. This information should be associatable with specific processing runs.*

The ``PipelineTask`` execution system already includes support for saving logs to butler datasets.
That satisfies these requirements in a minimal sense, but we expect higher-level workflow tools to do a better job of saving logs (in addition or instead) using third-party tools better suited for log analysis.
This is already the case at the IDF.

- *[REQ-WFL-003] Failed quanta must be reported including where in the batch processing system the quantum was running at the time of failure.*

This will be satisfied by implementing the design described in [:ref:`provenance-schema`] and [:ref:`recording-provenance`], provided the ``host`` field in the ``quantum`` table is consistent with the "where" question here.

- *[REQ-WFL-004] Though no requirement exists, it should be possible to inspect, post-facto, the resource usage (CPU, memory, I/O etc.) for individual workers.*

This is already implemented for provenance queries that start with the data ID of the quantum to be inspected, because these values are stored in the task metadata dataset (at a coarse level by the execution system, and optionally with more fine-grained information by the task itself).
The rest of the design described in this document should allow this information to be connected with worker nodes.

Observatory-Butler linkage
--------------------------

Possibly relevant recommendations:

- *[REC-EXP-1] As planned, program details known to the scheduler (such as science programme and campaign name) should be captured by the Butler.*

As long as these details are included in the FITS headers of the raw files, they can be configured to be capture by the butler and stored in the ``exposure`` dimension table.
This is already the case for the fields used as examples in this recommendation.

- *[REC-EXP-2] As planned, OCS queue submissions that result in meaningfully grouped observations should be identified as such in the Butler.*

The middleware does currently read a group ID field from the raw headers and store it in the ``exposure`` table.
We also have a ``visit``-definition scheme that interprets the group ID as a way to relate the snaps that correspond to a single visit, for on-sky science observations only (in the butler dimensions model, a ``visit`` may only be defined for on-sky data).

While that makes this recommendation is formally satisfied, this approach has two (related) problems:

- Using group ID to connect snaps to visits leaves us without a way to create more flexible groups of science exposures.
- The association of group ID with visits also discourages using group ID for more flexible groups of non-science exposures, such as focus sweeps or flat sequences, or causes confusion about whether visits (or some new visit-like dimension) should be defined and used for these.

It would be much better for a new snap-specific header key to be introduced and interpreted by the middleware, freeing up the group ID to have more flexible and ad-hoc definitions.
We believe this change is already ticketed in the data acquisition systems, but the work is not yet complete.

Metrics linkage
---------------

- *[REC-MET-001] For metrics that can be associated with a Butler dataId, the metrics should be persisted using the Data Butler as the source of truth. The dataId associated with the metric should use the full granularity.*
- *[REC-MET-002] Any system that uses Butler data to derive metrics should persist them in the Butler provided that the metrics are associable with a Data ID.*

This is already implemented and in regular use in the ``faro`` package in particular, and is currently the only way that metrics derived from butler data are initially stored.
It is arguable whether the butler datasets are considered the source of truth after upload to SQuaSH.

- *[REC-MET-003] When lsst.verify.Job objects are exported, the exported object should include the needed information (run collection and dataId) to associate with the source of truth metric persisted with the Data Butler.*

It is at least unusual for butler datasets to store their own data ID and especially their own ``RUN`` collection internally.
It might make more sense for metric values persisted to butler data repositories to be saved as a dataset that does not have this state, and for the system that exports it to SQuaSH to combine the dataset content with the data ID and ``RUN`` collection from the ``Registry``.

- *[REC-MET-005] Even if effort for implementation is not available in construction, we should develop a conceptual design for structured, semantically rich storage of metrics in the Butler.*

We currently save metric measurements as individual JSON files, which is convenient for upload to SQuaSH but inconvenient for querying metric values via the butler.
It also precludes using thresholds on metric values at criteria in ``QuantumGraph`` generation.
A custom ``Datastore`` backed by by either the ``Registry`` "opaque table" system or SQuaSH itself (along with ``Registry`` query system extensions) would make butler queries against metrics much more convenient and efficient.
`DMTN-203 <https://dmtn-203.lsst.io/>`__ will provide more detail on this subject.
This will be easier if we can normalize the content in metric datasets with what is in the ``Registry`` and generally make them smaller and more consistent, in essence unifying the ``lsst.verify`` data model with the butler one:

- Each ``lsst.verify.Metric`` can be mapped directly to a butler dataset type, so there should be no need for a metric measurement to store its ``Metric`` internally.
- The opaque blobs associated with an ``lsst.verify.Measurement`` should probably be factored out into separate butler datasets with different dataset types when measurements are stored in the butler.
- An ``lsst.verify.Job`` is a container for a group of measurements, and is probably best not mapped directly to anything stored by the butler, but a higher-level factory for ``Job`` instances that uses the butler to query for and fetch measurements and blob data may be useful, especially as a way to upload to SQuaSH.

This mapping is very much preliminary, and is based on a fairly superficial understanding of the ``lsst.verify`` data model.
A more detailed design should be included in `DMTN-203 <https://dmtn-203.lsst.io/>`__.

Saving provenance in dataset files
----------------------------------

- *[REC-FIL-1] Serialised exported data products (FITS files in the requirements) should include file metadata (e.g. FITS header) that allows someone in possession of the file to come to our services and query for additional provenance information for that artifact (e.g. pipeline-task level provenance).*

Low-level I/O for files written by the butler goes through the ``lsst.daf.butler.Formatter`` interface, which could be easily extended to include external ``dict``-like metadata that should be recorded in the file.
That metadata should be passed through a new optional keyword argument to ``Butler.put``, and can be prepared by the execution system to include provenance information.
As implied by the recommendation text, this could only be implemented in formatters that use data formats that can store flexible metadata, but this should be true in practice for any data format used for public data products (including FITS, Parquet, and JSON).

To satisfy *[REC-FIL-1]*, all we need to store is the dataset's UUID, which (after implementing the design described in [:ref:`recording-provenance`]) will be available to the execution system for insertion into the provenance metadata when the dataset is first saved to disk.
It would be easy and possibly useful to also save the UUID of the producing quantum, and it may be worth also recording the UUIDs of all datasets input to the quantum (grouped by dataset type) as well, to possibly avoid the need for full provenance queries in the simplest cases.
This may be less useful than it seems or perhaps even slightly confusing in some cases, however, because:

- as of the time a dataset is written, we can only reliably know the "available" inputs to the quantum; the task may *later* declare that only some of these inputs were actually used;
- in a few cases (large gather-scatter sequence points in the graph), the number of inputs to the quantum may be large (in the case of FGCM, a full-survey sequence point, it will be enormous);
- there is no guarantee that the datasets directly input (as opposed to transitively used as inputs via some predecessor quantum's outputs) to a quantum constitute scientifically interesting provenance.

At this time, it seems prudent to save only the dataset and quantum UUIDs, absent a clear use case for saving more.

- *[REC-FIL-2] A study should be made of the possibility of embedding a DataLink or other service pointer in the FITS header in lieu of representing the provenance graph in the file.*

The formatter hook described in the previous subsection would clearly be capable of embedding such a link in the file when it is first written, but doing so effectively declares that the service pointer can

- be calculated from some combination of information (dataset UUID, data ID, dataset type, ``RUN`` collection, URI(s)) known to the execution system or formatter;
- be assumed to remain static over the lifetime of the file.

These seem like dangerous assumptions, in that satisfying them probably either creates unwanted dependencies between software components (the execution system has to be configured to know about RSP service endpoints) or puts undesirable constraints on others.

Injecting this metadata into files when they are retrieved seems much cleaner conceptually, but it may rule out simple and/or efficient approaches to data access that would otherwise be on the table.
We would not consider this kind of implementation of this recommendation to be a middleware responsibility.

Provenance recommendations not directly relevant to middleware
--------------------------------------------------------------

To the extent that these recommendations describe best practices or conventions, we believe middleware provenance systems will be consistent with them, but they do not directly map to any current or planned functionality.

- *[REC-EXP-3] Any system (eg. LOVE, OLE/OWL) allowing the entering or modification of exposure-level ancillary data should collect provenance information on that data (who, what, why).*
- *[REQ-TEL-001] Investigate ways to expose all information in the Camera Control System Database to the EFD.*
- *[REQ-TEL-002] The MMSs should ideally have an API and at the very least a machine-readable export of data that would allow its data to be retrieved by other systems.*
- *[REQ-TEL-003] Any new CSCs (and wherever possible any current CSCs that lack them) should have requirements on what provenance information they should make available to SAL so it can be associated with their telemetry.*
- *[REC-SW-1] There are a number of extant versioning mechanisms in DM and T&S software environments. Care should be to not proliferate those unreasonably but to share software versioning and packaging infrastructure where possible. As these systems are hard to get right, the more teams use them, the more robust they tend to be.*
- *[REC-SW-2] All systems should have individual explicit requirements addressing what, if any, demands there are to be able to recover a prior system state. When such requirements are needed, the systems should have to capture and publish in a machine-readable form, version information that is necessary to fulfil those requirements. Such requirements should cover the need for data model provenance, eg. whether it is necessary to know when a particular schema was applied to a running system.*
- *[REC-SW-4] Containerization offers significant and tangible advantages in software reproducibility for a modest investment in build/deploy infrastructure; it should be preferred wherever possible for new systems, and systems that predate the move to containerization should be audited to examine whether there is a reasonable path to integrate them to current deployment practices.*
- *[REC-FIL-3] Irrespective of ongoing design discussions, every attempt should be made to capture information that could later be used to populate a provenance service.*
- *[REC-SRC-001] Perform a census of produced and planned flags to ensure that 64 bits for sources and 128 bits for objects are sufficient within a generous margin of error. This activity should also be carried out for DIASources and DIAObjects source IDs.*
- *[REC-SRC-002] We are concerned that merely encoding a 4-bit data release provenance in a source does not scale to commissioning needs and the project should decide whether it is acceptable for additional information beyond the source ID to be required to fully associate a source with a specific image.*
- *[REC-SRC-003] More generally, a study should be conducted on whether 64 bit source IDs are sufficient.*
- *[REC-SRC-004] Although not provenance-related, we recommend that the DPDD be updated to clearly state whether footprints and heavy footprints are to be provided.*
- *[REC-MET-004] A plan should be developed for persisting metrics that are not directly associated with Butler-persisted data.*
- *[REC-LOG-1] Since time is the primary provenance element for a log entry, systems are to produce (or make searchable) in UTC.*
- *[REC-LOG-2] Each site (summit, IDF, USDF, UKDF, FRDF) should provide a log management solution or dispatch to another site’s log management service to aid log discoverability.*
- *[REC-LOG-3] Individual systems should make clear log retention requirements.*


.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
