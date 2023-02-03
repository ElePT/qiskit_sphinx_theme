=======================
Opflow Migration Guide
=======================

*Jump to* `TL;DR`_.

Background
----------

The ``qiskit.opflow`` module was originally introduced as a layer between circuits and algorithms, a series of building blocks
for quantum algorithms research and development. The core design of opflow was based on the assumption that the
point of access to backends (be they real devices or simulators) was a ``backend.run()``
type of method: a method that takes in a circuit and returns its measurement results.
Under this assumption, all the tasks related to operator handling and building expectation value
computations were left to the user to manage. Opflow helped bridge that gap, it allowed to wrap circuits and
observables into operator classes that could be algebraically manipulated, so that the final result's expectation
values could be easily computed following different methods.

This basic opflow functionality is covered by  its core submodules: the ``operators`` submodule
(including operator globals, list ops, primitive ops, and state functions), the ``converters`` submodule, and
the ``expectations`` submodule.
Following this reference framework of ``operators``, ``converters`` and ``expectations``, opflow includes more
algorithm-specific functionality, which can be found in the ``evolutions`` submodule (specific for hamiltonian
simulation algorithms), as well as the ``gradients`` submodule (applied in multiple machine learning and optimization
use-cases). Some classes from the core modules mentioned above are also algorithm or application-specific,
for example the ``CVarMeasurement`` or the ``Z2Symmetries``.

..  With the introduction of the primitives we have a new mechanism that allows.... efficient... error mitigation...

The recent introduction of the ``qiskit.primitives`` provided a new interface for interacting with backends. Now, instead of
preparing a circuit to execute with a ``backend.run()`` type of method, the algorithms can leverage the ``Sampler`` and
``Estimator`` primitives, send parametrized circuits and observables, and directly receive quasi-probability distributions or
expectation values (respectively). This workflow simplifies considerably the pre-processing and post-processing steps
that previously relied on opflow. For example, the ``Estimator`` primitive returns expectation values from a series of
circuit-observable pairs, superseding most of the functionality of the ``expectations`` submodule. Without the need of
building opflow expectations, most of the components in ``operators`` also became redundant, as they commonly wrapped
elements from ``qiskit.quantum_info``.

Higher-level opflow sub-modules, such as the ``gradients`` sub-module, were refactored to take full advantage
of the primitives interface. They can now be accessed as part of the ``qiskit.algorithms`` module,
together with other primitive-based subroutines. Similarly, the ``evolutions`` sub-module got refactored, and now
can be easily integrated into a primitives-based workflow (as seen in the new ``time_evolvers`` algorithms).

All of these reasons have encouraged us to move away from opflow, and find new paths of developing algorithms based on
the ``qiskit.primitives`` interface and the ``qiskit.quantum_info`` module, which is a powerful tool for representing
and manipulating quantum operators.

This guide traverses the opflow submodules and provides either a direct alternative
(i.e. using ``quantum_info``), or an explanation of how to replace their functionality in algorithms.

TL;DR
-----
The new ``qiskit.primitives`` have superseded most of the ``qiskit.opflow`` functionality. Thus, the latter is being deprecated.

Index
-----
This guide covers the migration from these opflow sub-modules:

**Operators**

- `Operator Base Class`_
- `Operator Globals`_
- `Primitive and List Ops`_
- `State Functions`_

**Converters**

- `Converters`_
- `Evolutions`_
- `Expectations`_

**Gradients**

- `Gradients`_


Operator Base Class
-------------------

The ``opflow.OperatorBase`` abstract class can generally be replaced with ``quantum_info.BaseOperator``, keeping in
mind that ``quantum_info.BaseOperator`` is more generic than its opflow counterpart. In particular, you should consider that:

1. ``opflow.OperatorBase`` implements a broader algebra mixin. Some operator overloads are not available in
``quantum_info.BaseOperator``.

2. ``opflow.OperatorBase`` also implements methods such as ``to_matrix()`` or ``to_spmatrix()``, which are only found
in some of the ``quantum_info.BaseOperator`` subclasses.

.. list-table:: Migration of ``qiskit.opflow.operator_base``
   :header-rows: 1

   * - opflow
     - alternative
     - notes
   * - ``opflow.OperatorBase``
     - ``quantum_info.BaseOperator``
     - For more information, check the ``quantum_info.BaseOperator`` source code.

Operator Globals
----------------

1-Qubit Paulis
~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.operator_globals (1/3)``
   :header-rows: 1

   * - opflow
     - alternative
     - notes
   * - ``opflow.X``, ``opflow.Y``, ``opflow.Z``, ``opflow.I``
     - ``quantum_info.Pauli``
     - For direct compatibility with classes in ``qiskit.algorithms``, wrap in ``quantum_info.SparsePauliOp``.
   * -

        .. code-block:: python

            from qiskit.opflow import X
            operator = X ^ X

     -

        .. code-block:: python

            from qiskit.quantum_info import Pauli
            X = Pauli('X')
            op = X ^ X

     -

        .. code-block:: python

            from qiskit.quantum_info import Pauli, SparsePauliOp
            op = Pauli('X') ^ Pauli('X') # equivalent to:
            op = SparsePauliOp('XX')

Common non-parametrized gates (Clifford)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.. list-table:: Migration of ``qiskit.opflow.operator_globals (2/3)``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.CX``, ``opflow.S``, ``opflow.H``, ``opflow.T``, ``opflow.CZ``, ``opflow.Swap``
     - Append corresponding gate to ``QuantumCircuit`` + ``quantum_info.Clifford`` + ``.to_operator()``
     -

   * -

        .. code-block:: python

            from qiskit.opflow import H
            op = H ^ H

     -

        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.quantum_info import Clifford
            qc = QuantumCircuit(2)
            qc.h(0)
            qc.h(1)
            op = Clifford(qc).to_operator()

            # or... would this work?
            qc = QuantumCircuit(1)
            qc.h(0)
            H = Clifford(qc).to_operator()
            op = H ^ H

     -

1-Qubit States
~~~~~~~~~~~~~~
.. list-table:: Migration of ``qiskit.opflow.operator_globals (3/3)``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.Zero``, ``opflow.One``, ``opflow.Plus``, ``opflow.Minus``
     - ``quantum_info.Statevector``
     -

   * -

        .. code-block:: python

            from qiskit.opflow import Zero, One
            op = Zero ^ One

     -

        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.quantum_info import Statevector
            qc = QuantumCircuit(1)
            zero = Statevector(qc)
            qc.x(0)
            one = Statevector(qc)
            op = zero ^ one
     -


Primitive and List Ops
----------------------
Most of the workflows that previously relied in components from ``opflow.primitive_ops`` and ``opflow.list_ops`` can now
leverage ``quantum_info.operators`` elements instead. Some of these classes don't require a 1-1 replacement because
they were created to interface with other opflow components.

PrimitiveOps
~~~~~~~~~~~~~~
TODO: Add examples!!!

.. list-table:: Migration of ``qiskit.opflow.primitive_ops``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.PrimitiveOp``
     - No replacement needed
     - Can directly use ``quantum_info.Operator``
   * - ``opflow.CircuitOp``
     - No replacement needed
     - Can directly use ``QuantumCircuit``
   * - ``opflow.MatrixOp``
     - ``quantum_info.Operator``
     -
   * - ``opflow.PauliOp``
     - ``quantum_info.Pauli``
     - For direct compatibility with classes in ``qiskit.algorithms``, wrap in ``quantum_info.SparsePauliOp``
   * - ``opflow.PauliSumOp``
     - ``quantum_info.SparsePauliOp``
     -
   * - ``opflow.TaperedPauliSumOp``
     - This class combines the operator with its identified symmetries in one object, and with the refactoring of ``Z2Symmetries`` is no longer necessary
     -
   * - ``opflow.Z2Symmetries``
     - ``quantum_info.Z2Symmetries``
     - This class was refactored to also replace ``TaperedPauliSumOp``

PrimitiveOps Examples
~~~~~~~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.Z2Symmetries``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * -

        .. code-block:: python

            from qiskit.opflow import PuliSumOp, Z2Symmetries, TaperedPauliSumOp

            qubit_op = PauliSumOp.from_list(
                [
                ("II", -1.0537076071291125),
                ("IZ", 0.393983679438514),
                ("ZI", -0.39398367943851387),
                ("ZZ", -0.01123658523318205),
                ("XX", 0.1812888082114961),
                ]
            )
            z2_symmetries = Z2Symmetries.find_Z2_symmetries(qubit_op)
            primitive = SparsePauliOp.from_list(
                [
                ("I", -1.0424710218959303),
                ("Z", -0.7879673588770277),
                ("X", -0.18128880821149604),
                ]
            )
            tapered_op = TaperedPauliSumOp(primitive, z2_symmetries)
     -

        .. code-block:: python

            from qiskit.quantum_info import SparsePauliOp, Z2Symmetries

            qubit_op = SparsePauliOp.from_list(
                [
                    ("II", -1.0537076071291125),
                    ("IZ", 0.393983679438514),
                    ("ZI", -0.39398367943851387),
                    ("ZZ", -0.01123658523318205),
                    ("XX", 0.1812888082114961),
                ]
            )
            z2_symmetries = Z2Symmetries.find_z2_symmetries(qubit_op)
            tapered_op = z2_symmetries.taper(qubit_op)[1]
     -

ListOps
~~~~~~~
.. list-table:: Migration of ``qiskit.opflow.list_ops``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.ListOp``
     - No replacement needed. This classed was used internally within opflow.
     -

   * - ``opflow.ComposedOp``
     - No replacement needed. This classed was used internally within opflow.
     -

   * - ``opflow.SummedOp``
     - No replacement needed. This classed was used internally within opflow.
     -

   * - ``opflow.TensoredOp``
     - No replacement needed. This classed was used internally within opflow.
     -

State Functions
---------------

This module can be generally replaced by ``quantum_info.QuantumState``, with some differences to keep in mind:

1. The primitives-based workflow does not rely on constructing state functions as opflow did
2. The equivalence is, once again, not 1-1.
3. Algorithm-specific functionality has been migrated to the respective algorithm's module


.. list-table:: Migration of ``qiskit.opflow.state_fns``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.StateFn``
     - No replacement needed. This classed was used internally within opflow.
     -

   * - ``opflow.CircuitStateFn``
     - No replacement needed. This classed was used internally within opflow.
     -

   * - ``opflow.DictStateFn``
     - No replacement needed. This classed was used internally within opflow.
     -

   * - ``opflow.VectorStateFn``
     - This classed was used internally within opflow, but there exists a ``quantum_info`` replacement. There's the ``quantum_info.Statevector`` class and the ``quantum_info.StabilizerState`` (Clifford based vector).
     -

   * - ``opflow.SparseVectorStateFn``
     - No replacement needed. This classed was used internally within opflow.
     - See ``opflow.VectorStateFn``

   * - ``opflow.OperatorStateFn``
     - No replacement needed. This classed was used internally within opflow.
     -
   * - ``opflow.CVaRMeasurement``
     - Used in :class:`~qiskit.opflow.CVaRExpectation`. Functionality now covered by ``SamplingEstimator``. See example in expectations.
     -

Converters
----------

They manipulate operators within opflow. Most are no longer necessary when using primitives.

Circuit Sampler
~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.CircuitSampler``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.CircuitSampler``
     - ``primitives.Estimator``
     -

   * -

        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.opflow import X, Z, StateFn, CircuitStateFn, CircuitSampler
            from qiskit.providers.aer import AerSimulator

            qc = QuantumCircuit(1)
            qc.h(0)
            state = CircuitStateFn(qc)
            hamiltonian = X + Z

            expr = StateFn(hamiltonian, is_measurement=True).compose(state)
            backend = AerSimulator()
            sampler = CircuitSampler(backend)
            expectation = sampler.convert(expr)
            expectation_value = expectation.eval().real

     -

        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.primitives import Estimator
            from qiskit.quantum_info import SparsePauliOp

            state = QuantumCircuit(1)
            state.h(0)
            hamiltonian = SparsePauliOp.from_list([('X', 1), ('Z',1)])

            estimator = Estimator()
            expectation_value = estimator.run(state, hamiltonian).result().values

     -

Two Qubit Reduction
~~~~~~~~~~~~~~~~~~~~
.. list-table:: Migration of ``qiskit.opflow.TwoQubitReduction``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``TwoQubitReduction``

     - ``???``

     -

Other Converters
~~~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.converters``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.AbelianGrouper``
     - No replacement needed. This classed was used internally within opflow.
     -
   * - ``opflow.DictToCircuitSum``
     - No replacement needed. This classed was used internally within opflow.
     -
   * - ``opflow.PauliBasisChange``
     - No replacement needed. This classed was used internally within opflow.
     -

Evolutions
----------

The ``evolutions`` sub-module was created to provide building blocks for hamiltonian simulation algorithms,
including various methods for trotterization. This module is divided
The ``opflow.PauliTrotterEvolution`` class computes evolutions for exponentiated sums of Paulis by changing them each to the
Z basis, rotating with an RZ, changing back, and trotterizing following the desired scheme. Within its ``.convert`` method,
the class follows a recursive strategy that involves creating ``opflow.EvolvedOp`` placeholders for the operators,
constructing ``PauliEvolutionGate``\s out of the operator primitives and supplying one of the desired synthesis methods to
perform the trotterization (either via a ``string``\, which is then inputted into a ``opflow.TrotterizationFactory``,
or by supplying a method instance of ``opflow.Trotter()``, ``opflow.Suzuki()`` or ``opflow.QDrift()``).

The different trotterization methods that extend ``opflow.TrotterizationBase`` were migrated (motivation?) to ``qiskit.synthesis``,
and now extend the ``synthesis.evolution.ProductFormula`` base class. They no longer contain a ``.convert()`` method for standalone use,
but now are designed to be plugged into the ``synthesis.PauliEvolutionGate`` and called via ``.synthesize()``.
In this context, the job of the ``opflow.PauliTrotterEvolution`` class can now be handled directly by the algorithms
(for example, ``algorithms.time_evolvers.TrotterQRTE``\), by constructing the evolution gate and synthesizing it (?),
as shown in the following example:

.. list-table:: Migration of ``qiskit.opflow.evolutions (1/2)``
   :header-rows: 1

   * - opflow
     - alternative

   * -

        .. code-block:: python

            from qiskit.opflow import Trotter, PauliTrotterEvolution, PauliSumOp

            hamiltonian = PauliSumOp.from_list([('X', 1), ('Z',1)])
            evolution = PauliTrotterEvolution(trotter_mode=Trotter(), reps=1)
            evol_result = evolution.convert(hamiltonian.exp_i())
            evolved_state = evol_result.to_circuit()
     -

        .. code-block:: python

            from qiskit.quantum_info import SparsePauliOp
            from qiskit.synthesis import SuzukiTrotter
            from qiskit.circuit.library import PauliEvolutionGate
            from qiskit import QuantumCircuit

            hamiltonian = SparsePauliOp.from_list([('X', 1), ('Z',1)])
            evol_gate = PauliEvolutionGate(hamiltonian, 1, synthesis=SuzukiTrotter())
            evolved_state = QuantumCircuit(1)
            evolved_state.append(evol_gate, [0])

In a similar manner, the ``opflow.MatrixEvolution`` class performs evolution by classical matrix exponentiation,
constructing a circuit with ``UnitaryGate``\s or ``HamiltonianGate``\s containing the exponentiation of the operator.
This class is no longer necessary, as the ``HamiltonianGate``\s can be directly handled by the algorithms.

.. list-table:: Migration of ``qiskit.opflow.evolutions (2/2)``
   :header-rows: 1

   * - opflow
     - alternative

   * -

        .. code-block:: python

            from qiskit.opflow import MatrixEvolution, MatrixOp

            hamiltonian = MatrixOp([[0, 1], [1, 0]])
            evolution = MatrixEvolution()
            evol_result = evolution.convert(hamiltonian.exp_i())
            evolved_state = evol_result.to_circuit()
     -

        .. code-block:: python

            from qiskit.quantum_info import SparsePauliOp
            from qiskit.extensions import HamiltonianGate
            from qiskit import QuantumCircuit

            evol_gate = HamiltonianGate([[0, 1], [1, 0]], 1)
            evolved_state = QuantumCircuit(1)
            evolved_state.append(evol_gate, [0])

To summarize:

.. list-table:: Migration of ``qiskit.opflow.evolutions.trotterizations``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.TrotterizationFactory``
     - This class is no longer necessary.
     -
   * - ``opflow.Trotter``
     - ``synthesis.SuzukiTrotter``
     - This class implemented the Trotter-Suzuki product formula, but the ``synthesis`` module also offers a ``synthesis.LieTrotter`` class
   * - ``opflow.Suzuki``
     - ``synthesis.SuzukiTrotter(reps=1)``
     -
   * - ``opflow.QDrift``
     - ``synthesis.QDrift``
     -

.. list-table:: Migration of ``qiskit.opflow.evolutions.evolutions``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.EvolutionFactory``
     - This class is no longer necessary.
     -
   * - ``opflow.EvolvedOp``
     - ``synthesis.SuzukiTrotter``
     - This class is no longer necessary.
   * - ``opflow.MatrixEvolution``
     - ``HamiltonianGate``
     -
   * - ``opflow.PauliTrotterEvolution``
     - ``PauliEvolutionGate``
     -

Expectations
------------
Expectations are converters which enable the computation of the expectation value of an observable with respect to some state function.
This functionality can now be found in the estimator primitive.

Algorithm-Agnostic Expectations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.expectations``
   :header-rows: 1

   * - opflow
     - alternative
     - notes
   * - ``opflow.ExpectationFactory``
     - No replacement needed.
     - A factory class for automatic selection of an Expectation based on the Operator to be converted and backend used to sample the expectation value.
   * - ``opflow.AerPauliExpectation``
     - Use ``Estimator`` primitive from ``qiskit_aer`` instead.
     - An Expectation converter for using Aer's operator snapshot to take expectations of quantum state circuits over Pauli observables.
   * - ``opflow.MatrixExpectation``
     - Use ``Estimator`` primitive from ``qiskit`` instead (uses Statevector).
     - An Expectation converter which converts Operator measurements to be matrix-based so they can be evaluated by matrix multiplication.
   * - ``opflow.PauliExpectation``
     - Use any ``Estimator`` primitive.
     - An Expectation converter for Pauli-basis observables by changing Pauli measurements to a diagonal ({Z, I}^n) basis and appending circuit post-rotations to the measured state function.

CVarExpectation
~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.expectations.CVaRExpectation``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``opflow.expectations.CVaRExpectation``
     - Functionality absorbed into corresponding VQE algorithm: ``qiskit.algorithms.minimum_eigensolvers.SamplingVQE``
     -
   * -

        .. code-block:: python

            from qiskit.opflow import CVaRExpectation, PauliSumOp

            from qiskit.algorithms import VQE
            from qiskit.algorithms.optimizers import SLSQP
            from qiskit.circuit.library import TwoLocal
            from qiskit_aer import AerSimulator
            backend = AerSimulator()
            ansatz = TwoLocal(2, 'ry', 'cz')
            op = PauliSumOp.from_list([('ZZ',1), ('IZ',1), ('II',1)])
            alpha=0.2
            cvar_expectation = CVaRExpectation(alpha=alpha)
            opt = SLSQP(maxiter=1000)
            vqe = VQE(ansatz, expectation=cvar_expectation, optimizer=opt, quantum_instance=backend)
            result = vqe.compute_minimum_eigenvalue(op)

     -

        .. code-block:: python

            from qiskit.quantum_info import SparsePauliOp

            from qiskit.algorithms.minimum_eigensolvers import SamplingVQE
            from qiskit.algorithms.optimizers import SLSQP
            from qiskit.circuit.library import TwoLocal
            from qiskit.primitives import Sampler
            ansatz = TwoLocal(2, 'ry', 'cz')
            op = SparsePauliOp.from_list([('ZZ',1), ('IZ',1), ('II',1)])
            opt = SLSQP(maxiter=1000)
            alpha=0.2
            vqe = SamplingVQE(Sampler(), ansatz, optm, aggregation=alpha)
            result = vqe.compute_minimum_eigenvalue(op)
     -

**Gradients**
-------------
Replaced by new gradients module (link) (link to new tutorial).

