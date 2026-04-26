signal module
=============

.. automodule:: gpiosvr.signal

.. contents::
   :local:
   :depth: 2

.. _configuration:

Configuration classes
---------------------

The classes in this section evaluate and hold configurations for signal 
protocols, signal listeners and GPIO mappings. They can be easily initialized 
with `dict` instances, e.g. read from JSON files. 

Auto-discovering and parsing JSON files are out of scope for
:mod:`gpiosvr.signal`. See the :mod:`ctlbase` package instead. 
The following is a sample JSON representation of a signal configuration::

   {
     "evdevName": "pi-sig-injector",
     "gpios": {
       "25": "door-bell"
     },
     "signalProtocols": {
       "door-bell": {
         "basePulse_μs": 9000,
         "basePulseCount": 5,
         "codeStartBit": 1,
         "preamble_ms": 10,
         "postamble_ms": 12,
         "tolerancePercentage": 30,
         "debounce_μs": 600,
         "signalListener": {
           "mayInject": false
         },
         "keys": {
           "0x1f": {
             "keyName": "KEY_SOUND"
           }
         }
       }
     }
   }


.. autoclass:: SignalConfig
   :members:
   :special-members: __init__

.. autoclass:: SignalProtocolDescription
   :members:
   :special-members: __init__

.. autoclass:: SignalListenerConfig
   :members:
   :member-order: bysource
   :special-members: __init__


Pulse event classes
-------------------

.. autoclass:: PulseEvent
   :members:

.. autoclass:: PulsesInterrupted
   :members:
   :show-inheritance:
   
.. autoclass:: PulseEventIterator
   :members:
   :special-members: __init__

.. autoclass:: SignalListener
   :members:
   :member-order: bysource
   :special-members: __init__

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
