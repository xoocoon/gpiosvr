.. currentmodule:: gpiosvr.picossa

picossa module
==============

.. automodule:: gpiosvr.picossa

.. contents::
   :local:
   :depth: 2
   
Main classes
------------

.. autoclass:: EdgeCallback
   :members:
   :undoc-members:
   :exclude-members: WATCHDOG_BIT
   :special-members: __init__

.. autoclass:: ResponseFuture
   :members:
   :special-members: __init__
   :show-inheritance:

.. autoclass:: PicoModem
   :members:
   :special-members: __init__
   :member-order: bysource

.. autoclass:: PicoReceiverThread
   :members:
   :special-members: __init__
   :exclude-members: run
   :show-inheritance:

.. autoclass:: PicoSerialAdapter
   :members:
   :inherited-members:
   :special-members: __init__
   :show-inheritance:
   :member-order: bysource

picod constants
---------------

The following constants are used in C structs for the serial communication with
a Pico device via the `picod` daemon running on that device.

.. autoclass:: MessageId

.. autoclass:: RequestType

.. autoclass:: Expected

Constants and defaults
----------------------

.. autodata:: PICOD_CHANNEL_COUNT

.. autodata:: SERIAL_RESPONSE_TIMEOUT_DEFAULT_MS

.. autodata:: SERIAL_RESPONSE_CHECKING_INTERVAL_DEFAULT_μS

.. autodata:: SERIAL_READ_BUFFER_BYTES_DEFAULT

.. autodata:: CANCEL_CHECKING_INTERVAL_DEFAULT_MS

.. autodata:: PAUSE_BEFORE_READING_DEFAULT_MS

.. autodata:: READ_TIMEOUT_DEFAULT_MS

.. autodata:: READ_BUFFER_BYTES_DEFAULT

.. autodata:: READ_DELIMITER_DEFAULT

.. autodata:: READ_DELIMITER_MIN_COUNT_DEFAULT
