signal module
=============

.. automodule:: gpiosvr.signal

.. contents::
   :local:
   :depth: 2

.. _signal-extension:

Extension mechanism
-------------------

Custom signal listeners can be created based on the extension mechanism
described for the :ref:`input <input-extension>` module.

Existing extensions are:

    * :mod:`~.signal_button` for physical buttons
    * :mod:`~.signal_ir` for infrared protocols (currently only RC6 MCE)

The default implementation of :class:`~.SignalListener` supports signal types
with fixed or variable pulse lengths or single edges.

.. signal-configuration:

Configuration classes
---------------------

The signal configuration classes are based on the corresponding classes from
the :ref:`input <input-configuration>` module.

.. autoclass:: SignalParameters
   :members:
   :special-members: __init__
   :show-inheritance:

.. autoclass:: SignalListenerConfig
   :members:
   :special-members: __init__
   :show-inheritance:

Main classes
------------

.. autoclass:: PulseEvent
   :members:

.. autoclass:: PulsesInterrupted
   :members:
   :show-inheritance:
   
.. autoclass:: PulseEventIterator
   :members:
   :special-members: __init__
   :show-inheritance:

.. autoclass:: SignalListener
   :members:
   :special-members: __init__
   :show-inheritance:

.. autoclass:: SignalServer
   :members:
   :special-members: __init__
   :show-inheritance:

Constants and defaults
----------------------

.. autodata:: TOLERANCE_PERCENTAGE_DEFAULT

.. autodata:: DEBOUNCE_μS_DEFAULT

.. autoclass:: BitOrder
   :members:

.. autodata:: BIT_ORDER_DEFAULT

.. autodata:: CODE_START_BIT_DEFAULT

.. autodata:: MAX_BASE_PULSE_COUNT

Helper classes
--------------

.. autoclass:: PulseHelper
   :members:
