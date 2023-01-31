======================================
Migrating Operator Logic from Opflow
======================================

The ``qiskit.opflow`` module was introduced as a layer between circuits and algorithms that presented a series of
useful tools for quantum algorithms research and development. The core design of opflow was based on
the assumption that the only point of access to backends (real devices or
simulators, local or remote) was through a ``backend.run()`` type of method, which took in a circuit and returned the
measurement results; thus, all of the tasks related to building expectation value computations were left to the user
to manage. Opflow helped bridge that gap, it allowed to wrap circuits and observables into operator classes that
could be algebraically manipulated, so that the final result's expectation values could be easily computed following
different methods.

This basic opflow functionality was covered by  its core submodules: the ``operators`` submodule
(including operator globals, list ops, primitive ops, and state functions), the ``converters`` submodule, and
the ``expectations`` submodule.
Following this reference framework of ``operators``, ``converters`` and ``expectations``, opflow added more
algorithm-specific functionality, such as that provided by the ``evolutions`` submodule (specific for Hamiltonian
Simulation algorithms), as well as the ``gradients`` submodule (applied in multiple machine learning and optimization
use-cases). Some classes from the core modules mentioned above are also algorithm or application-specific,
for example the ``CVarMeasurement`` or the ``Z2Symmetries``.

The recent introduction of the ``qiskit.primitives`` challenged the assumptions upon which Opflow was designed. In particular,
the ``Estimator`` primitive provides the algorithmic abstraction to easily obtain expectation values from a series of
circuit-observable pairs, rendering the ``expectations`` submodule obsolete, and leaving most of the components in ``converters``
and ``operators`` without a clear purpose. The incorporation of primitives also motivated the development of a new gradient
framework that could leverage their interface, as well as new time evolution algorithms. The new code is leaner
and avoids certain performance bottlenecks that were introduced by the opflow design.

All of these reasons have encouraged us to move away from opflow, and find new paths of developing algorithms based on
the ``qiskit.primitives`` interface and the ``qiskit.quantum_info`` module, which is a powerful tool for representing
and manipulating quantum operators.

This guide traverses all of the opflow submodules and provides alternatives for each of them, be them a direct alternative
(i.e. using ``quantum_info``) or an explanation of the new way of doing things.

Opflow modules covered in this guide.

- Operator Base Class
- Operator Globals [Done]
- List Ops
- Primitive Ops
- State Functions

- Converters [Done-ish]
- Evolutions [Done-ish]
- Expectations

- Gradients [links]

Do we want to include utilities and mixins? they are documented in principle. We could have a link to the API ref.

**OPERATOR BASE CLASS**
-----------------------

.. list-table:: Migration of ``qiskit.opflow.operator_base``
   :header-rows: 1

   * - opflow
     - alternative
     - Notes
   * - ``opflow.OperatorBase``

     - ``alternative.BaseOperator``

     - Opflow side: StarAlgebraMixin, TensorMixin. QI side: GroupMixin

**OPERATOR GLOBALS**
--------------------

.. list-table:: Migration of ``qiskit.opflow.operator_globals (1/3)``
   :header-rows: 1

   * - opflow
     - alternative
     - Notes
   * - 1-Qubit Paulis: ``X``, ``Y``, ``Z``, ``I``

       Example:
        .. code-block:: python

            from qiskit.opflow import X
            operator = X ^ X

     - ``quantum_info.Pauli``

       Example:
        .. code-block:: python

            from qiskit.quantum_info import Pauli
            X = Pauli('X')
            op = X ^ X

     - For direct compatibility with classes in ``qiskit.algorithms``, wrap in ``quantum_info.SparsePauliOp``.

       Example:
        .. code-block:: python

            from qiskit.quantum_info import Pauli, SparsePauliOp
            op = Pauli('X') ^ Pauli('X') # equivalent to:
            op = SparsePauliOp('XX')

.. list-table:: Migration of ``qiskit.opflow.operator_globals (2/3)``
   :header-rows: 1

   * - opflow
     - alternative
     - Notes

   * - Common non-parametrized gates (Clifford): ``CX``, ``S``, ``H``, ``T``, ``CZ``, ``Swap``

       Example:
        .. code-block:: python

            from qiskit.opflow import H
            op = H ^ H

     - Append corresponding gate to QuantumCircuit + ``quantum_info.Clifford`` + ``to_operator()``

       Example:
        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.quantum_info import Clifford
            qc = QuantumCircuit(1)
            qc.h(0)
            qc.h(0)
            op = Clifford(qc).to_operator()

            # or
            qc = QuantumCircuit(1)
            qc.h(0)
            H = Clifford(qc).to_operator()
            op = H ^ H

     -

.. list-table:: Migration of ``qiskit.opflow.operator_globals (3/3)``
   :header-rows: 1

   * - opflow
     - alternative
     - Notes

   * - 1-Qubit States: ``Zero``, ``One``, ``Plus``, ``Minus``

       Example:
        .. code-block:: python

            from qiskit.opflow import Zero, One
            op = Zero ^ One

     - ``quantum_info.Statevector``

       Example:
        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.quantum_info import Statevector
            qc = QuantumCircuit(1)
            zero = Statevector(qc)
            qc.x(0)
            one = Statevector(qc)
            op = zero ^ one
     -

**PRIMITIVE OPS**
-----------------

- PrimitiveOp -> quantum_info Operator (Statevector??)
- CircuitOp -> no replacement / QuantumCircuit
- MatrixOp -> no replacement / quantum_info Operator
- PauliOp -> quantum_info Pauli
- PauliSumOp -> quantum_info SparsePauliOp
- TaperedPauliSumOp -> quantum_info SparsePauliOp. Functionality in nature?
- Z2Symmetries -> quantum_info/nature

**LIST OPS**
------------

No direct replacement for these. In opflow you could patch different types of operators together,
but in quantum info they are directly combined.

- ListOp
- ComposedOp
- SummedOp
- TensoredOp

**STATE FNs**
-------------

Generally replaced by ``quantum_info.QuantumState``, but they are structured differently:
there’s the Statevector (VectorStateFn) and StabilizerState (Clifford based vector).

- StateFn
- CircuitStateFn
- DictStateFn
- VectorStateFn
- SparseVectorStateFn
- OperatorStateFn
- CVaRMeasurement --> Used in :class:`~qiskit.opflow.CVaRExpectation`. Functionality replaced by DiagonalEstimator

**CONVERTERS**
--------------

manipulate operators within opflow. Most are no longer necessary when using primitives.
In this module you can find:

- CircuitSampler -> primitives
- AbelianGrouper -> no replacement
- DictToCircuitSum -> no replacement
- PauliBasisChange -> no replacement
- TwoQubitReduction -> quantum_info/nature

.. list-table:: Migration of ``qiskit.opflow.CircuitSampler``
   :header-rows: 1

   * - opflow
     - alternative
     - Notes

   * - ``CircuitSampler``

       Example:
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

     - ``qiskit.primitives.Estimator``

       Example:
        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.primitives import Estimator
            from qiskit.quantum_info import SparsePauliOp

            state = QuantumCircuit(1)
            state.h(0)
            hamiltonian = SparsePauliOp.from_list([('X', 1), ('Z',1)])

            estimator = Estimator()
            expectation_value = estimator.run(state, hamiltonian).result().values

     -  Provided with a backend/quantum instance and an operator expression, the job of the circuit sampler is
        to execute all circuits in the operator expression and replace them by the circuit result. This can now
        be done with an estimator primitive.

.. list-table:: Migration of ``qiskit.opflow.TwoQubitReduction``
   :header-rows: 1

   * - opflow
     - alternative?
     - Notes

   * - ``TwoQubitReduction``

     - ``???``

     -

**EVOLUTIONS**
--------------

The Evolutions are essentially implementations of Hamiltonian Simulation algorithms,
including various methods for Trotterization. These have been superseded by the new time evolvers module
using primitives (link).

In this module you can find:

**Evolutions:**

- EvolutionFactory -> no replacement
- EvolvedOp -> no replacement
- MatrixEvolution -> HamiltonianGate
- PauliTrotterEvolution -> PauliEvolutionGate

**Trotterizations:**

Trotterizations are replaced by the synthesis methods in qiskit.synthesis.evolutions (QDrift not ported yet).

- TrotterizationFactory
- Trotter
- Suziki
- QDrift

**EXPECTATIONS**
----------------
Expectations are converters which enable the computation of the expectation value of an observable with respect to some state function.
This functionality can now be found in the estimator primitive.

- ExpectationFactory: A factory class for convenient automatic selection of an Expectation based on the Operator to be converted and backend used to sample the expectation value.
- AerPauliExpectation: An Expectation converter for using Aer's operator snapshot to take expectations of quantum state circuits over Pauli observables.
- MatrixExpectation: An Expectation converter which converts Operator measurements to be matrix-based so they can be evaluated by matrix multiplication.
- PauliExpectation: An Expectation converter for Pauli-basis observables by changing Pauli measurements to a diagonal ({Z, I}^n) basis and appending circuit post-rotations to the measured state function.
- CVaRExpectation -> Replaced by DiagonalEstimator.

.. list-table:: Migration of ``qiskit.opflow.expectations.CVaRExpectation``
   :header-rows: 1

   * - opflow
     - alternative
     - Notes

   * - ``opflow.expectations.CVaRExpectation``

       Example:
        .. code-block:: python

            from qiskit.opflow import Z, Plus, StateFn, CVaRExpectation

            state = Plus
            observable = StateFn(Z)
            op = ~observable @ state
            cvar_expecation = CVaRExpectation(alpha=0.2)
            cvar = cvar_expecation.convert(op).eval()
     - ``algorithms.minimum_eigensolvers.diagonal_estimator._DiagonalEstimator``

       Example:
        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.primitives import Sampler
            from qiskit.algorithms.minimum_eigensolvers.diagonal_estimator import _DiagonalEstimator as CVaREstimator

            state = QuantumCircuit(1)
            state.h(0)
            state.measure_all() # add measurements
            observable = SparsePauliOp('Z')
            estimator = CVaREstimator(sampler=Sampler(), aggregation=0.2)
            cvar = estimator.run(state, observable).result().values
     -


**GRADIENTS**
--------------
Replaced by new gradients module (link) (link to new tutorial).

**UTILITY FUNCTIONS**
---------------------
- commutator
- anti_commutator
- double_commutator
