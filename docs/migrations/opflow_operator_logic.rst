=======================
Opflow Migration Guide
=======================

*Jump to* `TL;DR`_.

Background
----------

The ``qiskit.opflow`` module was originally introduced as a layer between circuits and algorithms, a series of building blocks
for quantum algorithms research and development. The core design of opflow was based on the assumption that the
point of access to backends (be they real devices or simulators, local or remote) was through a ``backend.run()``
type of method: a method that takes in a circuit and returns its measurement results.
Under this assumption, all the tasks related to building expectation value
computations were left to the user to manage. Opflow helped bridge that gap, it allowed to wrap circuits and
observables into operator classes that could be algebraically manipulated, so that the final result's expectation
values could be easily computed following different methods.

This basic opflow functionality was covered by  its core submodules: the ``operators`` submodule
(including operator globals, list ops, primitive ops, and state functions), the ``converters`` submodule, and
the ``expectations`` submodule.
Following this reference framework of ``operators``, ``converters`` and ``expectations``, opflow added more
algorithm-specific functionality, which can be found in the ``evolutions`` submodule (specific for hamiltonian
simulation algorithms), as well as the ``gradients`` submodule (applied in multiple machine learning and optimization
use-cases). Some classes from the core modules mentioned above are also algorithm or application-specific,
for example the ``CVarMeasurement`` or the ``Z2Symmetries``.

..  With the introduction of the primitives we have a new mechanism that allows.... efficient... error mitigation...

The recent introduction of the ``qiskit.primitives`` challenged the assumptions upon which opflow was designed. In particular,
the ``Estimator`` primitive provides the algorithmic abstraction to easily obtain expectation values from a series of
circuit-observable pairs, superseding most of the functionality of the ``expectations`` submodule. Without the need of
building opflow expectations, most of the components in ``operators`` also became redundant, as they
commonly wrapped elements from ``qiskit.quantum_info``.

In addition to this, the introduction of the qiskit primitives motivated the development of a new gradient framework that
could leverage their interface, as well as new time evolution algorithms. The new code is leaner
and avoids certain performance bottlenecks that were introduced by the opflow design.

All of these reasons have encouraged us to move away from opflow, and find new paths of developing algorithms based on
the ``qiskit.primitives`` interface and the ``qiskit.quantum_info`` module, which is a powerful tool for representing
and manipulating quantum operators.

This guide traverses the opflow submodules and provides either a direct alternative
(i.e. using ``quantum_info``), or an explanation of the new way of doing things.

TL;DR
-----
The assumptions based on which ``qiskit.opflow`` was written are no longer up-to-date. Thus, it is being deprecated.

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

The ``opflow.OperatorBase`` abstract class can generally be replaced with ``quantum_info.BaseOperator``, but this
is not an exact 1-1 relation. For type-hinting/type-checking purposes, you should keep in mind that ``quantum_info.BaseOperator``
is more generic than its opflow counterpart. In particular, you should consider that:

1. ``opflow.OperatorBase`` implements a broader algebra mixin, it overloads operators such as ``~`` (adjoint/inverse)
or the ``tensorpower`` operation ``^``, where  ``operator^int`` tensors ``operator`` with itself ``int`` times.


2. All ``opflow.OperatorBase`` subclasses contain methods such as ``to_matrix()`` or ``to_spmatrix()``, which are only
implemented in some of the ``quantum_info.BaseOperator`` subclasses.

.. list-table:: Migration of ``qiskit.opflow.operator_base``
   :header-rows: 1

   * - opflow
     - alternative
     - notes
   * - ``opflow.OperatorBase``

     - ``quantum_info.BaseOperator``

     - Opflow side: StarAlgebraMixin, TensorMixin. QI side: GroupMixin

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
Most of the workflows that previously relied in components from `opflow.primitive_ops` and `opflow.list_ops` can now
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
     - No replacement needed. Can directly use ``quantum_info.Operator``
     -
   * - ``opflow.CircuitOp``
     - No replacement needed. Can directly use ``QuantumCircuit``
     -
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
     - This functionality was designed for Nature-specific use cases, and is now taken care of within ``qiskit-nature``
     -
   * - ``opflow.Z2Symmetries``
     - This functionality was migrated to ``quantum_info.Z2Symmetries``
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

Algorithm-agnostic State Functions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
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

CVaRMeasurement
~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.CVaRMeasurement``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``qiskit.opflow.CVaRMeasurement``
     - Functionality replaced by ``_DiagonalEstimator`` in ``minimum_eigensolvers``.
     - Used in :class:`~qiskit.opflow.CVaRExpectation`. See example in expectations.

   * -

        .. code-block:: python

            from qiskit.opflow import CVaRMeasurement
            # TODO
     -

        .. code-block:: python

            from qiskit import QuantumCircuit
            # TODO

     -


Converters
----------

manipulate operators within opflow. Most are no longer necessary when using primitives.

Circuit Sampler
~~~~~~~~~~~~~~~

.. list-table:: Migration of ``qiskit.opflow.CircuitSampler``
   :header-rows: 1

   * - opflow
     - alternative
     - notes

   * - ``CircuitSampler``
     - ``qiskit.primitives.Estimator``
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

The Evolutions are building blocks for hamiltonian simulation algorithms, including various methods for trotterization.

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

Expectations
------------
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
            cvar_expectation = CVaRExpectation(alpha=0.2)
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
            vqe = SamplingVQE(Sampler(), ansatz, opt)
            result = vqe.compute_minimum_eigenvalue(op)
     -

**Gradients**
-------------
Replaced by new gradients module (link) (link to new tutorial).

