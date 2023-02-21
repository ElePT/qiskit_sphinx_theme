Calculate expectation values in an algorithm
==============================================

The role of the ``Estimator`` primitive is two-fold: on one hand, it acts as an entry point to the quantum devices or
simulators, replacing ``backend.run()``.

On the other hand, it is an **algorithmic abstraction** for expectation value calculations, which removes the need
to perform operations to construct the final expectation circuit. This results in a considerable reduction of the code
complexity, and a more compact algorithm design.

.. note::

    The following example uses common tools from the ``qiskit.opflow`` module as a reference for the "legacy way of doing
    things", but we acknowledge that some of you might have used custom code for this task. In that case, you can decide
    between keeping your custom code and replacing ``backend.run()`` with a ``Sampler``, or replacing your custom code with
    the ``Estimator`` primitive.


Problem definition 
--------------------

We want to compute the expectation value of a quantum state (circuit) with respect to a certain operator.
Here we are using the H2 molecule and an arbitrary circuit as the quantum state:

.. code-block:: python


    from qiskit import QuantumCircuit
    from qiskit.quantum_info import SparsePauliOp

    # Step 1: Define operator
    op = SparsePauliOp.from_list(
        [
            ("II", -1.052373245772859),
            ("IZ", 0.39793742484318045),
            ("ZI", -0.39793742484318045),
            ("ZZ", -0.01128010425623538),
            ("XX", 0.18093119978423156),
        ]
    )

    # Step 2: Define quantum state
    state = QuantumCircuit(2)
    state.x(0)
    state.x(1)

.. _a-legacy-opflow:

[Legacy] Convert problem to ``opflow``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`Opflow <https://qiskit.org/documentation/apidoc/opflow.html>`__ provided its own classes to represent both
operators and quantum states, so the problem defined above would be wrapped as:

.. code-block:: python

    from qiskit.opflow import CircuitStateFn, PauliSumOp

    opflow_op = PauliSumOp(op)
    opflow_state = CircuitStateFn(state)

This step is no longer necessary using the primitives.

.. note::

    For more information on migrating from ``opflow``, see the `opflow migration guide <qisk.it/opflow_migration>`_ .

Calculate expectation values on real device or cloud simulator
---------------------------------------------------------------

[Legacy] Using ``opflow`` + ``backend.run()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can see the number of steps that legacy workflow involved to be able to compute an expectation
value:

.. note::

    You can replace ``ibmq_qasm_simulator`` with your device name to see the
    complete workflow for a real device.

.. code-block:: python

    from qiskit.opflow import StateFn, PauliExpectation, CircuitSampler
    from qiskit_ibm_provider import IBMProvider

    # Define the state to sample
    measurable_expression = StateFn(opflow_op, is_measurement=True).compose(opflow_state)

    # Convert to expectation value calculation object
    expectation = PauliExpectation().convert(measurable_expression)

    # Define provider and backend (formerly imported from IBMQ)
    provider = IBMProvider()
    backend = provider.get_backend("ibmq_qasm_simulator")

    # Inject backend into circuit sampler
    sampler = CircuitSampler(backend).convert(expectation)

    # Evaluate
    expectation_value = sampler.eval().real

    print("expectation: ", expectation_value)

[New] Using Runtime ``Estimator``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now, you can notice how the ``Estimator`` simplifies the user-side syntax, which makes it a more
convenient tool for algorithm design.

.. note::

    You can replace ``ibmq_qasm_simulator`` with your device name to see the
    complete workflow for a real device.

.. code-block:: python

    from qiskit_ibm_runtime import QiskitRuntimeService, Estimator

    service = QiskitRuntimeService(channel="ibm_quantum")
    backend = service.backend("ibmq_qasm_simulator")

    estimator = Estimator(session=backend)

    expectation_value = estimator.run(state, op).result().values

    print("expectation: ", expectation_value)



Other execution alternatives (non-Runtime)
------------------------------------------

In some cases, you might want to test your algorithm using local simulation. For this means, we
will should you two more migration paths using non-runtime primitives.

[Legacy] Using Qiskit Aer's Statevector simulator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    from qiskit.opflow import StateFn, PauliExpectation, CircuitSampler
    from qiskit_aer import AerSimulator

    # Define the state to sample
    measurable_expression = StateFn(opflow_op, is_measurement=True).compose(opflow_state)

    # Convert to expectation value calculation object
    expectation = PauliExpectation().convert(measurable_expression)

    # Define statevector simulator
    simulator = AerSimulator(mothod="statevector", shots=100)

    # Inject backend into circuit sampler
    sampler = CircuitSampler(simulator).convert(expectation)

    # Evaluate
    expectation_value = sampler.eval().real

    print("expectation: ", expectation_value)


[New] Using Reference ``Estimator`` or Aer ``Estimator``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Reference ``Estimator`` allows to perform either an exact or a shot-based noisy simulation based
on the ``Statevector`` class in the ``qiskit.quantuminfo`` module.

.. code-block:: python

    from qiskit.primitives import Estimator

    estimator = Estimator()

    result = estimator.run(state, op).result().values

    # for shot-based simulation:
    expectation_value = estimator.run(state, op, shots=100).result().values

    print("expectation: ", expectation_value)

You can still access the Aer Simulator through its dedicated
``Estimator``. This can come in handy for performing simulations with noise models. For more
information on using the Aer Primitives, you can check out this
`VQE tutorial <https://qiskit.org/documentation/tutorials/algorithms/03_vqe_simulation_with_noise.html>`_ .

.. code-block:: python

    from qiskit_aer.primitives import Estimator # all that changes is the import!!!

    estimator = Estimator()

    result = estimator.run(state, op).result().values

    # for shot-based simulation:
    expectation_value = estimator.run(state, op, shots=100).result().values

    print("expectation: ", expectation_value)
