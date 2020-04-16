.. _sec_api:

=================
API Documentation
=================

This is the API documentation for ``msprime``, and provides detailed information
on the Python programming interface. See the :ref:`sec_tutorial` for an
introduction to using this API to run simulations.
See the `tskit documentation <https://tskit.readthedocs.io/en/stable>`_ for
information on how to use the
`tskit Python API <https://tskit.readthedocs.io/en/stable/python-api.html>`_
to analyse simulation results.

****************
Simulation model
****************

The simulation model in ``msprime`` closely follows the classical ``ms``
program. Unlike ``ms``, however, time is measured in generations rather than
in units of :math:`4 N_e` generations, i.e., "coalescent units".
This means that when simulating a population with diploid effective size :math:`N_e`,
the mean time to coalescence between two samples
in an ``msprime`` simulation will be around :math:`2 N_e`,
while in an ``ms`` simulation, the mean time will be around :math:`0.5`.
Internally, ``msprime`` uses the same algorithm as ``ms``,
and so the ``Ne`` parameter to the :func:`.simulate` function
still acts as a time scaling, and can be set to ``0.5`` to match many theoretical results,
or to ``0.25`` to match ``ms``. Population sizes for each
subpopulation and for past demographic events are also defined as absolute values, **not**
scaled by ``Ne``. All migration rates and growth rates are also per generation.

When running simulations we define the length :math:`L` of the sequence in
question using the ``length`` parameter. This defines the coordinate space
within which trees and mutations are defined. :math:`L` is a continuous value,
so units are arbitrary, and coordinates can take any continuous value from :math:`0` up to
(but not including) :math:`L`. (So, although we recommend setting the units of length to be
analogous to "bases", events can occur at fractional positions.)
Mutations occur in an infinite sites process along this sequence,
and mutation rates are specified per generation, per unit of sequence length.
Thus, given the per-generation mutation rate :math:`\mu`, the rate of mutation
over the entire sequence in coalescent time units is :math:`\theta = 4 N_e \mu
L`. It is important to remember these scaling factors when comparing with
analytical results!

Similarly, recombination rates are per unit of sequence length and per
generation in ``msprime``. Thus, given the per generation crossover rate
:math:`r`, the overall rate of recombination between the ends of the sequence
in coalescent time units is :math:`\rho = 4 N_e r L`. Although breakpoints do
not necessarily occur at integer locations, the underlying recombination model
is finite, and the behaviour of a small number of loci can be modelled using
the :class:`.RecombinationMap` class. However, this is considered an advanced
feature and the majority of cases should be well served with the default
recombination model and number of loci.

For gene conversion there are two parameters. The gene conversion rate determines the initiation
and is again per unit of sequence length and per generation in ``msprime``.
Thus, given the per generation gene conversion rate :math:`g`, the overall rate of
gene conversion initiation between the ends of the sequence is :math:`\rho = 4 N_e g L` in
coalescent time units. The second parameter :math:`track\_len` is the expected track length
of a gene conversion. At each gene conversion initiation site the track of the conversion
extends to the right and the length of the track is geometric distributed with parameter
:math:`1/track\_len`. Currently recombination maps for gene conversion are not supported.
However, recombination (with or without recombination maps) and a constant gene conversion
rate along the genome can be combined in ``msprime``.

Population structure is modelled by specifying a fixed number of subpopulations
:math:`d`, and a :math:`d \times d` matrix :math:`M` of per generation
migration rates. Each element of the matrix :math:`M_{j,k}` defines
the fraction of population :math:`j` that consists of migrants from
population :math:`k` in each generation.
Each subpopulation has an initial absolute population size :math:`s`
and a per generation exponential growth rate :math:`\alpha`. The size of a
given population at time :math:`t` in the past (measured in generations) is
therefore given by :math:`s e^{-\alpha t}`. Demographic events that occur in
the history of the simulated population alter some aspect of this population
configuration at a particular time in the past.

.. warning:: This parameterisation of recombination, mutation and
    migration rates is different to :program:`ms`, which states these
    rates over the entire region and in coalescent time units. The
    motivation for this is to allow the user change the size of the simulated
    region without having to rescale the recombination, gene conversion, and mutation rates,
    and to also allow users directly state times and rates in units of
    generations. However, the ``mspms`` command line application is
    fully :program:`ms` compatible.
    If recombination and gene conversion are combined the gene conversion
    rate in :program:`ms` is determined by the ratio :math:`f`, which corresponds to
    setting :math:`g = f r`. In ``msprime`` the gene conversion rate :math:`g` is
    set independently and does not depend on the recombination rate. However,
    ``mspms`` mimics the :program:`ms` behaviour.

*******************
Running simulations
*******************

The :func:`.simulate` function provides the primary interface to running
coalescent simulations in msprime.

.. autofunction:: msprime.simulate()

********************
Population structure
********************

Population structure is modelled in ``msprime`` by specifying a fixed number of
subpopulations, with the migration rates between those subpopulations defined by a migration
matrix. Each subpopulation has an ``initial_size`` that defines its absolute diploid size at
time zero and a per-generation ``growth_rate`` which specifies the exponential
growth rate of the sub-population. We must also define the number of genomes to
sample from each subpopulation. The number of populations and their initial
configuration is defined using the ``population_configurations`` parameter to
:func:`.simulate`, which takes a list of :class:`.PopulationConfiguration`
instances. Population IDs are zero indexed, and correspond to their position in
the list.

Samples are drawn sequentially from populations in increasing order of
population ID. For example, if we specified an overall sample size of 6, and
specify that 2 samples are drawn from population 0 and 4 from population 1,
then samples 0 and 1 will be initially located in population 0, and
samples 2, 3, 4, and 5 will be drawn from population 2.

Given :math:`N` populations, migration matrices are specified using an :math:`N
\times N` matrix of between-subpopulation migration rates. See the
documentation for :func:`.simulate` and the `Simulation model`_ section for
more details on the migration rates.

.. autoclass:: msprime.PopulationConfiguration

.. _sec_api_demographic_events:

******************
Demographic Events
******************

Demographic events change some aspect of the population configuration
at some time in the past, and are specified using the ``demographic_events``
parameter to :func:`.simulate`. Each element of this list must be an
instance of one of the following demographic events
that are currently supported. Note that all times are measured in
generations, all sizes are absolute (i.e., *not* relative to :math:`N_e`),
and all rates are per-generation.

Model change events are also considered to be demographic events;
see the :ref:`sec_api_simulation_models_change_events` section for details.

.. autoclass:: msprime.PopulationParametersChange
.. autoclass:: msprime.MigrationRateChange
.. autoclass:: msprime.MassMigration
.. autoclass:: msprime.CensusEvent

+++++++++++++++++++++
Parsing species trees
+++++++++++++++++++++

Species trees hold information about the sequence and the times at which species
diverged from each other. Viewed backwards in time, divergence events are equivalent
to mass migration events in which all lineages from one population move to another
population. The history of a set of populations can thus be modelled according to
a given species tree. To faciliate the specification of the model, 
:func:`.parse_species_tree` parses a species tree and returns the mass migration
events corresponding to all species divergence events in the tree, together with
population configurations that specify population sizes and names.

When species trees are estimated with a program like `StarBEAST
<https://academic.oup.com/mbe/article/34/8/2101/3738283>`_ they can further
contain estimates on the population sizes of extant and ancestral species.
:func:`.parse_starbeast` parses species trees estimated with StarBEAST and uses
these estimates to define the population configurations.

Note that when the species tree has branch lengths not in units of generations but
in units of years or millions of years (which is common), a generation time in years
is required for parsing.

.. autofunction:: msprime.parse_species_tree()
.. autofunction:: msprime.parse_starbeast()

++++++++++++++++++++++++++++
Debugging demographic models
++++++++++++++++++++++++++++

.. warning:: The ``DemographyDebugger`` class is preliminary, and the API
    is likely to change in the future.

.. autoclass:: msprime.DemographyDebugger
    :members:

****************************
Variable recombination rates
****************************

.. autoclass:: msprime.RecombinationMap
    :members:

.. _sec_api_simulation_models:

*****************
Simulation models
*****************

The default simulation model in ``msprime`` is the standard coalescent with recombination
model. We also support a number of different models, which are documented in this section.

Simulations models are specified using the ``model`` parameter to
:func:`.simulate`. This parameter can either take the form of a
string describing the model (e.g. ``model="dtwf"``) or an instance of a
model definition class (e.g ``model=msprime.DiscreteTimeWrightFisher(1000)``).
The available models are documented in the following subsections.

A key element of simulation models in ``msprime`` is the concept
of a "reference population size". The ``Ne`` argument to
:func:`simulate` can also be used to define this parameter
when combined with a string shorthand for a model.

.. code-block:: python

    msprime.simulate(10, Ne=1000, model="dtwf")

and

.. code-block:: python

    msprime.simulate(10, model=msprime.DiscreteTimeWrightFisher(1000))

define the same simulation.

.. note:: When ``Ne`` and an instance of :class:`.SimulationModel` with
    ``reference_size`` set are both provided as arguments to :func:`.simulate`,
    the ``Ne`` parameter is ignored.


.. todo:: Add a discussion of population sizes here, describing
    what Ne/model.reference_size really means, and how it interacts
    with the individual population sizes.

.. JK: Commented this discussion out as it seemed likely to confuse. We
.. definitely need to give a thorough description of what's actually
.. going on somewhere though.

.. Simulation models such as the coalescent are defined
.. in terms of "scaled time". Time in the standard diploid coalescent
.. is measured in units of :math:`N_e` generations, and
.. one of the ways in which msprime tries to make life easier for
.. users is to automatically convert times and rates to and
.. from units of generations. Thus, a key responsibility for a simulation model is to
.. convert times specified by users in generations into "model time"
.. (in which the simulation is performed) and to tranlate the
.. simulated model times back into generations. This is what the
.. reference population size associated with a model is used for
.. and fundamentally what it means.

.. The situation is somewhat confused by the sizes associated
.. with specific populations using, e.g., the
.. :class:`.PopulationConfiguration` class. In coalescent models, these
.. sizes are used to compute rates of coalescence, and have no direct
.. relationship to the model's reference population size. Thus, we may
.. have several populations, all of which have different sizes and none
.. equal to the model's reference population size.
.. The model's reference population size is used to scale all times
.. and rates, irrespective of the sizes of the various individual
.. populations.

.. The situation for the :class:`.DiscreteTimeWrightFisher` model
.. is different. In this case, there is no concept of rescaling time
.. as time is always measured in generations. Therefore, in this case the
.. reference population size has no function and is just used as a
.. shortcut for specifying individual population sizes.

We are often interested in simulating mixtures of models: for example,
using the :class:`.DiscreteTimeWrightFisher` model to simulate the
recent past and then using the standard coalescent to complete the
simulation of the ancient past. This can be achieved using the
:class:`.SimulationModelChange` event. See the
:ref:`sec_tutorial_hybrid_simulations` for an example of this
approach.

+++++++++++++++++++++++++++++
Coalescent and approximations
+++++++++++++++++++++++++++++

.. autoclass:: msprime.StandardCoalescent

.. autoclass:: msprime.SmcApproxCoalescent

.. autoclass:: msprime.SmcPrimeApproxCoalescent

+++++++++++++++++++++++++++
Discrete time Wright-Fisher
+++++++++++++++++++++++++++

Msprime provides the option to perform discrete-time Wright-Fisher simulations
for scenarios when the coalescent model is not appropriate, including large
sample sizes, multiple chromosomes, or recent migration.

To use this option, set the flag ``model="dtwf"`` as in the following example::

    >>> tree_sequence = msprime.simulate(
    ...     sample_size=6, Ne=1000, length=1e4, recombination_rate=2e-8,
    ...     model="dtwf")


All other parameters can be set as usual.

.. autoclass:: msprime.DiscreteTimeWrightFisher


.. _sec_api_simulation_models_selective_sweeps:

++++++++++++++++
Selective sweeps
++++++++++++++++

.. todo:: Document the selective sweep models.


.. _sec_api_simulation_models_change_events:

++++++++++++++++++++++++
Simulation model changes
++++++++++++++++++++++++

.. todo:: Describe the subtleties of how simulation model changes work
    along with how times are computed.

.. autoclass:: msprime.SimulationModelChange


.. _sec_api_simulate_from:

*********************************************
Initialising simulations from a tree sequence
*********************************************

By default ``msprime`` simulations are initialised by specifying a set of samples,
using the ``sample_size`` or  ``samples`` parameters to :func:`.simulate`. This
initialises the simulation with segments of ancestral material covering the
whole sequence. Simulation then proceeds backwards in time until a most recent
common ancestor has been found at all points along this sequence. We can
also start simulations from different initial conditions by using the
``from_ts`` argument to :func:`.simulate`. Informally, we take an 'unfinished'
tree sequence as a parameter to simulate, initialise the simulation
from the state of this tree sequence and then run the simulation until
coalescence. The returned tree sequence is then the result of taking the
input tree sequence and completing the trees using the coalescent.

This is useful for forwards-time simulators such as
`SLiM <https://messerlab.org/slim/>`_ that can output tree sequences. By running
forward-time simulation for a certain number of generations we obtain a
tree sequence, but these trees may not have had sufficient time to
reach a most recent common ancestor. By using the ``from_ts`` argument
to :func:`.simulate` we can combine the best of both forwards- and
backwards-time simulators. The recent past can be simulated forwards
in time and the ancient past by the coalescent. The coalescent
simulation is initialised by the root segments of the
input tree sequence, ensuring that the minimal amount of ancestral
material possible is simulated.

Please see the :ref:`tutorial <sec_tutorial_simulate_from>` for an example of how to use this
feature with a simple forwards-time Wright-Fisher simulator

++++++++++++++++++
Input requirements
++++++++++++++++++

Any tree sequence can be provided as input to this process, but there is a
specific topological requirement that must be met for the simulations to be
statistically correct. To ensure that ancestral segments are correctly associated within chromosomes
when constructing the initial conditions for the coalescent simulation,
forward-time simulators **must** retain the nodes corresponding to the
initial generation. Furthermore, for every sample in the final generation
(i.e. the extant population at the present time) there must be a path
to one of the founder population nodes. (Please see the :ref:`tutorial <sec_tutorial_simulate_from>`
for further explanation of this point and an example.)

+++++++++++++++++++++++++++++
Recombination map limitations
+++++++++++++++++++++++++++++

Because of the way that ``msprime`` handles recombination internally, care must
be taken when specifying recombination when using the ``from_ts`` argument.
If recombination positions are generated in the same way in both the initial
tree sequence and the coalescent simulation, then everything should work.
However, the fine scale details of the underlying recombination model matter,
so matching nonuniform recombination maps between simulators may not be
possible at present. (To make it work, we must ensure that every recombination
breakpoint in ``from_ts`` matches exactly to a possible recombination
breakpoint in msprime's recombination map, which is not guaranteed because of
msprime's discrete recombination model.)

One case in which it is guaranteed to work is if ``from_ts`` has integer
coordinates, and we want to simulate a coalescent with a uniform recombination
rate. In this case, to have a uniform recombination rate ``r`` use::

    L = int(from_ts.sequence_length)
    recomb_map = msprime.RecombinationMap.uniform_map(L, r, L)
    final_ts = mpsrime.simulate(from_ts=from_ts, recomb_map=recomb_map)


.. _sec_api_node_flags:

**********
Node flags
**********

For standard coalescent simulations, all samples are marked with the
:data:`tskit.NODE_IS_SAMPLE` flag; internal nodes all have a flags value of 0.
When using the ``record_full_arg`` argument to :func:`.simulate`, the following
flags values are defined:

.. data:: msprime.NODE_IS_RE_EVENT

    The node is an ARG recombination event. Each recombination event is marked
    with two nodes, one identifying the individual providing the genetic
    material to the left of the breakpoint and the other providing the genetic
    material the right.

.. data:: msprime.NODE_IS_CA_EVENT

    The node is an ARG common ancestor event that did not result in
    marginal coalescence.

.. data:: msprime.NODE_IS_MIG_EVENT

    The node is an ARG migration event identifying the individual that migrated.
    Can be used in combination with the ``record_migrations`` argument to
    :func:`.simulate`.

.. data:: msprime.NODE_IS_CEN_EVENT

    The node was created by a :class:`msprime.CensusEvent`.


********************
Simulating mutations
********************

When running coalescent simulations it's usually most convenient to use the
``mutation_rate`` argument to the :func:`.simulate` function to throw neutral
mutations down on the trees. However, sometimes we wish to throw mutations
down on an existing tree sequence: for example, if we want to see the outcome
of different random mutational processes on top of a single simulated topology,
or if we have obtained the tree sequence from another program and wish to
overlay neutral mutations on this tree sequence.

.. autoclass:: msprime.InfiniteSites

.. data:: msprime.BINARY == 0

    The binary mutation alphabet where ancestral states are always "0" and
    derived states "1".

.. data:: msprime.NUCLEOTIDES == 1

    The nucleotides mutation alphabet in which ancestral and derived states are
    chosen from the characters "A", "C", "G" and "T".

.. autofunction:: msprime.mutate

*********************************
Evaluating sampling probabilities
*********************************

``msprime`` provides the capability to evaluate the sampling probabilities:
that of a stored tree sequence for a given diploid effective population size
:math:`N_e` and per-link, per-generation recombination probability :math:`r`
under the standard ancestral recombination graph; and that of a pattern of
mutations given a tree sequence and per-site, per-generation mutation 
probability :math:`\mu` under the infinite sites model.

Specifically, the methods assume that each pair of lineages merges at rate
:math:`1 / (2 N_e)`, and each link which separates ancestral material splits
at rate :math:`r`. The number of mutations has a conditional Poisson
distribution with mean :math:`T \mu`, where :math:`T` is the total branch
length in the tree sequence, and each mutation further contributes a factor of
:math:`l / T`, where :math:`l` is the branch length on which the mutation could
have happened such that it appears on all of the lineages on which it is
observed. Branch lengths must be stored in generations.

For both methods, the full history of the sample must be encoded into the tree
sequence. If the tree sequence is a realisation from the :func:`.simulate`
function, then the ``record_full_arg`` setting must have been used to store
recombination events, as well as coancestor events without marginal
coalescences. Sampling probabilities of tree sequences from other sources can
also be evaluated provided they contain the same information in the same
format.

.. autofunction:: msprime.log_arg_likelihood

.. autofunction:: msprime.unnormalised_log_mutation_likelihood

.. warning:: As the name suggests, the
    :func:`.unnormalised_log_mutation_likelihood` function returns the
    probability of the pattern of mutations given the underlying tree sequence
    and per-site, per-generation mutation probability only up to a
    combinatorial factor which depends on the configuration of mutations, but
    not on the tree sequence or other parameters. As such, it is only
    appropriate for use as a relative likelihood function. In contrast, the
    :func:`.log_arg_likelihood` function returns the exact probability of the
    tree sequence given the effective population size and per-link,
    per-generation recombination probabilty.

A recombination probability of zero is a permitted argument, and will return
a finite log likelihood if there are no break points in the tree sequence, and
a numerican representation of negative infinity, `-float("inf")`, if there are
one or more breakpoints. Zero mutation probabilities are also permitted, and
handled similarly.
