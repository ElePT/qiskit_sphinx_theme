Guide on algorithm tuning options
=================================

One of the advantage of the primitives is that they abstract away the circuit execution setup, so that algorithm developers
can focus on the pure algorithmic components. However, sometimes, to get the best out of an algorithm, you might want
to tune certain primitive options. This section will walk you through some of the common settings you might need.

.. attention::

    This section focuses on Qiskit Runtime Primitive :class:`.Options` (imported from ``qiskit_ibm_runtime``). While
    most of the primitives interface is common across implementations, most :class:`.Options` are not. Please consult the
    corresponding API references for further information on the ``qiskit`` and ``qiskit_aer`` primitive options.

1. Shots
~~~~~~~~

For some algorithms, setting a specific number of shots is a core part of their routines. Before the primitives,
the shots could be set during the call to ``backend.run(shots=1024)``. Now, the setting is part of the execution
options ("second level option"). This can be done during the primitive setup:

.. code-block:: python

    from qiskit_ibm_runtime import Estimator, Options

    options = Options()
    options.execution.shots = 1024

    estimator = Estimator(session=session, options=options)


Or, in cases where you need to modify the number of shots set between iterations/primitive calls, you can set the
shorts directly in the ``run()`` method. This overwrites the initial``shots`` setting.

.. code-block:: python

    from qiskit_ibm_runtime import Estimator

    estimator = Estimator(session=session)

    estimator.run(circuits=circuits, observables=observables, shots=50)

    # other logic

    estimator.run(circuits=circuits, observables=observables, shots=100)

For more information on the primitive options, you can check out the API
`reference <https://qiskit.org/documentation/partners/qiskit_ibm_runtime/stubs/qiskit_ibm_runtime.options.Options.html#qiskit_ibm_runtime.options.Options>`_.
for the ``Options`` class.


2. Transpilation
~~~~~~~~~~~~~~~~

By default, the Qiskit Runtime Primitives perform circuit transpilation under the hood. There are several optimization
levels you can choose from, these levels affect the transpilation strategy and might include additional error
suppression mechanisms. Level 0 only involves basic transpilation.
For more information on the meaning of each optimization level, check out this
`how-to <https://qiskit.org/documentation/partners/qiskit_ibm_runtime/locale/es_UN/how_to/error-suppression.html#setting-the-optimization-level>`_.

The optimization level option is a "first level option", and can be set as shown below:

.. code-block:: python

    from qiskit_ibm_runtime import Estimator, Options

    options = Options(optimization_level=2)

    # or..
    options = Options()
    options.optimization_level = 2

    estimator = Estimator(session=session, options=options)


You might want to configure your transpilation strategy further, and for this means, there are advanced transpilation
options you can set up. These are "second level options", and can be set as follows:

.. code-block:: python

    from qiskit_ibm_runtime import Estimator, Options

    options = Options()
    options.transpilation.initial_layout = ...
    options.transpilation.routing_method = ...

    estimator = Estimator(session=session, options=options)

For more information, and a complete list of advanced transpilation options, check out the following
`how-to <https://qiskit.org/documentation/partners/qiskit_ibm_runtime/locale/es_UN/how_to/error-suppression.html#advanced-transpilation-options>`_.

Finally, you might want to specify further settings that are not available through the primitives interface,
or use custom transpiler passes. In these cases, you can set ``skip_transpilation=True`` to submit
user-transpiled circuits. For more information on how this is done, you can read the following
`tutorial <https://qiskit.org/documentation/partners/qiskit_ibm_runtime/tutorials/user-transpiled-circuits.html>`_.

The ``skip_transpilation`` option is an advanced transpilation option, set as follows:

.. code-block:: python

    from qiskit_ibm_runtime import Estimator, Options

    options = Options()
    options.transpilation.skip_transpilation = True

    estimator = Estimator(session=session, options=options)


3. Error Mitigation
~~~~~~~~~~~~~~~~~~~

Finally, you may want to leverage different error mitigation methods and see how these affect the performance of your
algorithm. These can also be set throguh the ``resilience_level`` option, and the method selected for each level is
different for ``Sampler`` and ``Estimator``. You can find more information on the following
`how-to <https://qiskit.org/documentation/partners/qiskit_ibm_runtime/how_to/error-mitigation.html>`_.

The configuration is similar to the rest of the options:

The ``skip_transpilation`` option is an advanced transpilation option, set as follows:

.. code-block:: python

    from qiskit_ibm_runtime import Estimator, Options

    options = Options(resilience_level = 2)

    # or...

    options = Options()
    options.resilience_level = 2

    estimator = Estimator(session=session, options=options)
