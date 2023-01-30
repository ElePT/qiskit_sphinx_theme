======================================
Migrating Operator Logic from Opflow
======================================

qiskit.opflow -> qiskit.quantum_info
====================================

Provide context for opflow operator logic.

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

Below is a table of example access patterns in :obj:`~qiskit.opflow` and the alternative
with :obj:`~qiskit.quantum_info`:

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
            op = Clifford(qc).to_operator()
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

     - Construct corresponding state in QuantumCircuit + ``quantum_info.Clifford`` + ``to_operator()``

       Example:
        .. code-block:: python

            from qiskit import QuantumCircuit
            from qiskit.quantum_info import Clifford
            qc = QuantumCircuit(1)
            zero = Clifford(qc).to_operator()
            qc.x(0)
            one = Clifford(qc).to_operator()
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

- ListOp --> quantum info PauliList
- ComposedOp
- SummedOp -> quantum_info SparsePauliOp
- TensoredOp -> ?

**STATE FNs**
-------------

- StateFn
- CircuitStateFn
- DictStateFn
- VectorStateFn
- SparseVectorStateFn
- OperatorStateFn
- CVaRMeasurement

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
- MatrixEvolution -> no replacement
- PauliTrotterEvolution -> time evolvers Trotter?

**Trotterizations:**

- TrotterizationFactory
- Trotter --> time evolvers Trotter
- Suziki --> Do we have replacement?
- QDrift --> Do we have replacement?

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
