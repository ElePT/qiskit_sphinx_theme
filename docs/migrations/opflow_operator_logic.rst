======================================
Migrating Operator Logic from Opflow
======================================

The opflow module was introduced as a layer between circuits and algorithms that provided
the language and computational primitives for Quantum Algorithms and Applications research and
development using Qiskit.

It contained two main submodules: Operators and Converters. The first one provided the tools


Opflow modules covered in this guide

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
     - quantum_info
     - Notes
   * - ``opflow.OperatorBase``

     - ``quantum_info.BaseOperator``

     - Opflow side: StarAlgebraMixin, TensorMixin. QI side: GroupMixin

**OPERATOR GLOBALS**
--------------------

.. list-table:: Migration of ``qiskit.opflow.operator_globals (1/3)``
   :header-rows: 1

   * - opflow
     - quantum_info
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
     - quantum_info
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
     - quantum_info
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
- CVaRMeasurement --> Functionality replaced by DiagonalEstimator

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
     - primitives
     - Notes

   * - ``CircuitSampler``

       Example:
        .. code-block:: python

            from qiskit import Aer, QuantumCircuit
            from qiskit.opflow import X, Z, StateFn, CircuitSampler
            state = QuantumCircuit(1)
            state.h(0)
            hamiltonian = X + Z
            expr = StateFn(hamiltonian, is_measurement=True).compose(state)
            backend = Aer.get_backend('statevector_simulator')
            sampler = CircuitSampler(backend)
            sampled = sampler.convert(expr)

     - ``qiskit.primitives.Sampler``

       Example:
        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.primitives import Sampler
            from qiskit.quantum_info import SparsePauliOp
            state = QuantumCircuit(1)
            state.h(0)
            hamiltonian = SparsePauliOp.from_list([('X', 1), ('Z',1)])
            sampler = Sampler()
            sampled = sampler.run(state, hamiltonian).result().quasi_dists

     -  Provided with a backend/quantum instance and an operator expression, the job of the circuit sampler is
        to execute all circuits in the operator expression and replace them by the circuit result. This can now
        be done through a sampler primitive. Please note that the sampler returns quasi-dists (link to docs).

.. list-table:: Migration of ``qiskit.opflow.TwoQubitReduction``
   :header-rows: 1

   * - opflow
     - quantum_info?
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
Replaced by estimator primitive, also quantum_info.Statevector???

In this module you can find:

- ExpectationFactory
- AerPauliExpectation
- MatrixExpectation
- PauliExpectation
- CVaRExpectation

**GRADIENTS**
--------------
Replaced by new gradients module (link) (link to new tutorial).

**UTILITY FUNCTIONS**
---------------------
- commutator
- anti_commutator
- double_commutator
