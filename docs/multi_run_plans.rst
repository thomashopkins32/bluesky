Multi-Run Plans
===============

Introduction
------------

This section is a brief tutorial on multi-run plans (introduced in Bluesky v1.6.0).
A traditional single-run plan contains a set of instructions for performing only one run,
which is assigned a scan ID and a UID. When a multi-run plan is executed by the Run Engine, multiple
runs can be performed as part of a single plan. Data from each run can be independently
displayed and saved to the database via Databroker. Prior versions of Bluesky supported
only sequential execution of multiple runs within a plan: building larger plans by creating
a sequence of smaller plans and preassembled plans shipped with Bluesky is a standard
practice. In Bluesky v1.6.0 a number of features were introduced to allow plans
with nested runs. Two runs are considered nested if one 'outer' run is interrupted, another
'inner' run is executed, and then the first run is resumed and completed. The number of levels
of nesting is not limited by Bluesky. Interruptions can be initiated by the plan itself
(simply by opening another run before closing currently executed run) or externally (e.g.
by triggering a suspender and causing execution of pre- or post-plan). This tutorial includes
a brief explanation of the new Bluesky features for supporting multi-run plans and several
examples that demonstrate the implementation of plans that contain sequential, nested and recursive
runs.

Definition of a 'Run'
---------------------

From the point of view of Bluesky, a run is a sequence of instructions (messages) for controlling
the instrumental equipment that starts with `open_run` and ends with `close_run` message.
We may also apply the term 'run' to a block of code which generates such a sequence of messages.
Data from each run is bundled together via an assigned distinct Scan ID and UID. The set of documents
is also generated for each run, including mandatory 'start' and 'stop' documents. The documents
can be processed by callbacks (such as BestEffortCallback) and saved to the database via Databroker.

In the plan, the run may be defined by explicitely enclosing the code in `bps.open_run()` and
`bps.close_run()` stubs:

.. code-block:: python

    # Using 'bps.open_run()' and 'bps.close_run()' stubs to define a run

    import bluesky.plan_stubs as bps
    from bluesky import RunEngine

    RE = RunEngine({})

    def sample_plan():
        ...
        yield from bps.open_run(md={})  # 'md' - metadata to be added to the 'start' document
        ...
        < code that controls execution of the scan >
        ...
        yield from bps.close_run()

    RE(sample_plan())

or using `@bpp.run_decorator`, which inserts `open_run` and `close_run` control messages
before and after the sequence generated by the enclosed code:

.. code-block:: python

    # Using 'bpp.run_decorator' to define a run

    import bluesky.preprocessors as bpp
    from bluesky import RunEngine

    RE = RunEngine({})

    @bpp.run_decorator(md={})  # 'md' - metadata to be added to the 'start' document
    def sample_plan():
        ...
        < code that controls execution of the scan >
        ...

    RE(sample_plan())

The rules for basic Bluesky plans require that the currently running scan is closed before
the next scan is opened, therefore the following code works:

.. code-block:: python

    # This code works, since the first run is closed before the second one is opened

    import bluesky.plan_stubs as bps
    from bluesky import RunEngine

    RE = RunEngine({})

    def sample_plan():
        yield from bps.open_run(md={})
        < code that controls execution of the scan >
        yield from bps.close_run()  # Closing the first run (scan)
        yield from bps.open_run(md={})  # Opening the second run (scan)
        < code that controls execution of the scan >
        yield from bps.close_run()

    RE(sample_plan())

but the following code fails:

.. code-block:: python

    # This code fails, since the second run is opened before the first run is closed

    import bluesky.plan_stubs as bps
    from bluesky import RunEngine

    RE = RunEngine({})

    def sample_plan():
        yield from bps.open_run(md={})  # Opening the first run
        < code that controls execution of the scan >
        yield from bps.open_run(md={})  # Opening the second run before the first one is closed
        < code that controls execution of the scan >
        yield from bps.close_run()
        yield from bps.close_run()

    RE(sample_plan())


Note, that the preassembled plans, such as `bluesky.plans.count` or `bluesky.plans.list_scan`,
are complete single-run plans, enclosed in `open_run` and `close_run` messages, therefore
the following code fails as well:

.. code-block:: python

    # This code fails while attempting to start a preassembled plan from an open run

    import bluesky.plan_stubs as bps
    from bluesky.plans import count
    from bluesky import RunEngine

    RE = RunEngine({})

    def sample_plan():
        yield from bps.open_run(md={})  # Starting the first run
        < code that controls execution of the scan >
        yield from bpp.count(<some arguments>)  # Attempting to run a preassembled plan from an open run
        yield from bps.close_run()

    RE(sample_plan())

An example of the situation when a preassembled plan is called from another open run is
when a preassembled plan is included in a suspender pre- or post-plan. When the suspender is
triggered, the current run is interrupted (not closed) and the pre- or post-plan attempts to open
another run (the mechanism is the same as in the case of nested runs, see below). As a result,
Run Engine fails for the same reason as in the two previous code examples. The new multi-run plan
Bluesky features allow to implement nested plans, as well as include full-featured scans
in pre- and post-plans.

Bluesky Features for Support of Multi-run Plans
-----------------------------------------------

In order to handle simultaneously open runs within a plan, Run Engine is looking at the run key attribute
of each control message to decide which scan is currently being executed. The default value for the run key
is `None`, but it could be manually set in the plan for any block of code which define the run. A run key
value may be of any type, but it is **strongly** recommended that manually assigned run keys are
human-readable informative strings.

The new 'inner' run can be opened from within the 'outer' run only if the run keys of the 'inner' and
'outer' scans are different. Otherwise the plan execution fails.

The run key is used by Run Engine

* to maintain the state of each run independently from other open runs;

* to include run metadata, such as scan ID and UID, into the emitted documents. (Metadata is then used
  to route the documents to the appropriate callbacks. If documents are saved using Databroker, the metadata
  allows to associate documents with runs and retrieve run data from the database.)

Run key is assigned to a block of code using `bpp.set_run_key_wrapper` or `@bpp.set_run_key_decorator`:

.. code-block:: python

    import bluesky.preprocessors as bpp
    from bluesky import RunEngine

    # Using decorator
    @bpp.set_run_key_decorator("run_key_example_1")
    @bpp.run_decorator(md={})
    def sample_plan():
        ...
        < code that controls execution of the run >
        ...

    RE(sample_plan())

    from bluesky.plans import scan
    from ophyd.sim import hw
    det, motor = hw().det, hw().motor

    # Using wrapper
    s = scan([det], motor, -1, 1, 10)
    s_wrapped = bpp.set_run_key_wrapper(s, "run_key_example_2")
    RE(s_wrapped)

The implementation of `@bpp.set_run_key_decorator` and `bpp.set_run_key_wrapper` is
replacing the default value `None` of the attribute `run` in each message generated within
the enclosed block with the user-defined run key.

The `@bpp.set_run_key_decorator` and `bpp.set_run_key_wrapper` are primarily intended
to be applied to a function that contains a run implementation, but may be also used
with any block of plan code. For example, one may write a plan that simultaneously
opens multiple runs and executes them in parallel by generating groups of messages
with run ids of the open scans. This is currently not recommended and should be attempted
only at the developer's own risk.

Plans with Sequential Runs
---------------------------

Sequential calling of multiple runs is supported by older versions of Bluesky. There is no need
to use multi-run plan features if runs are not overlapping (the next run is opened only after
the previous run is closed), but run keys still can be assigned to all or some runs if needed.

In the following example, two preassembled plans are called in sequence. Run Engine is subscribed to
a single instance of BestEffortCallback, which is set up to display data specific for each run
when the run opened.

.. literalinclude:: /examples/multi_run_plans_sequential.py

.. ipython:: python
    :suppress:

    %run -m multi_run_plans_sequential

.. ipython:: python

    RE(plan_sequential_runs(10))

Plans with Nested Runs
----------------------

The following example illustrates the use of `@bpp.set_run_key_decorator` to implement two nested runs:
the 'outer' run interrupts measurements, calls the 'inner' run and then completes the measurements.
The 'outer' and 'inner' runs are assigned different run ids ('run_1' and 'run_2'). Note that
the `@bpp.set_run_key_decorator` for the 'outer' run does not overwrite the run id of the 'inner' scan,
despite the fact that it is generated inside the enclosed code, since the decorator is designed to replace
the run id attribute of the message only if it has the default value of `None`, i.e. the run id of
a message can be replaced by the decorator only the first time it is processed by the decorator.

If multiple runs are to be opened simultaneously, each run needs to be subscribed to its own instance
of callback. Standard RunEngine subscription mechanism does not provide this capability. Instead,
subscription should be performed via `RunRouter`. The code in the following example demonstrates how
to use `BestEffortCallback` to monitor data from multiple nested runs.

.. literalinclude:: /examples/multi_run_plans_nested.py

The output of the plan contains data from two runs with each run assigned its own ID and UID. The tables
for the runs are printed by two separate instances of `BestEffortCallback`. The data from two tables
is printed in the order of acquisition: the table for the 'inner' run is printed in the gap of
the table for the 'outer' run.

.. ipython:: python
    :suppress:

    %run -m multi_run_plans_nested

.. ipython:: python

    RE(sim_plan_outer(10))

The wrapper `bpp.set_run_key_wrapper` can be used instead of the decorator. For example
the run `sim_plan_inner` from the previous example can be rewritten as follows:

.. code-block:: python

    def sim_plan_inner(npts):
        def f():
            for j in range(npts):
                yield from bps.mov(hw.motor1, j * 0.1 + 1, hw.motor2, j * 0.2 - 2)
                yield from bps.trigger_and_read([hw.motor1, hw.motor2, hw.det2])
        f = bpp.run_wrapper(f(), md={})
        return bpp.set_run_key_wrapper(f, "run_2")

Subscription to callbacks via RunRouter provides flexibility to subscribe each run
to its own set of callbacks. In the following example `run_key` is added to the start
document metadata and used to distinguish between two runs in the function factory that
performs callback subscriptions.

.. literalinclude:: /examples/multi_run_plans_select_cb.py

.. ipython:: python
    :suppress:

    %run -m multi_run_plans_select_cb

.. ipython:: python

    RE(sim_plan_outer(10))

In some cases it may be necessary to implement a run that could be interrupted
and a new instance of the same run started. For example, the suspender pre- or post-plan
may contain a run, which takes substantial time to execute. Such run may be interrupted
if the suspender is repeatedly triggered. This will cause another instance of the pre-
or post-plan to be started while the first one is still in the open state. This process
is similar to recursive calling of the run (run which includes instructions to call
itself). Recursive calls are possible if unique run key is assigned to a run each
time it is started.

The following example illustrates dynamic generation of run keys. The plan may have no practical purpose
besides demonstration of the principle. The plan is calling itself recursively multiple times until
the global counter `n_calls` reaches the maximum value of `n_calls_max`. The unique run key is generated
before at each call.

.. literalinclude:: /examples/multi_run_plans_recursive.py

.. ipython:: python
    :suppress:

    %run -m multi_run_plans_recursive

.. ipython:: python

    RE(sim_plan_recursive(4))

The identical result can be achieved by using `bpp.set_run_key_wrapper()`:

.. code-block:: python

    # Call counter and the maximum number calls
    n_calls, n_calls_max = 0, 3

    def sim_plan_recursive(npts):
        global n_calls, n_calls_max

        n_calls += 1  # Increment counter
        if n_calls <= n_calls_max:
            # Generate unique key for each run. The key generation algorithm
            #   must only guarantee that execution of the runs that are assigned
            #   the same key will never overlap in time.
            run_key = f"run_key_{n_calls}"

            @bpp.run_decorator(md={})
            def plan(npts):

                for j in range(int(npts/2)):
                    yield from bps.mov(hw.motor1, j * 0.2)
                    yield from bps.trigger_and_read([hw.motor1, hw.det1])

                # Different parameter values may be passed to the recursively called plans
                yield from sim_plan_recursive(npts + 2)

                for j in range(int(npts/2), npts):
                    yield from bps.mov(hw.motor1, j * 0.2)
                    yield from bps.trigger_and_read([hw.motor1, hw.det1])

            yield from bpp.set_run_key_wrapper(plan(npts), run_key)
