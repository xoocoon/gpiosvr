.. currentmodule:: gpiosvr.uart

uart module
===========

.. automodule:: gpiosvr.uart

.. contents::
   :local:
   :depth: 2

.. _uart-configuration:

Configuration classes
---------------------

The classes in this section evaluate and hold configurations for UART 
transmission (TX) and receiving (RX). They can be initialized with `dict`
instances, e.g. read from JSON files. 

Auto-discovering and parsing JSON files are out of scope for
:mod:`gpiosvr.uart`. See the :external:mod:`ctlbase.config` module instead. 
The following is the JSON representation of an exemplary UART configuration:

.. code-block:: none

    "uart0": {
      "baud": 9600,
      "isCloseOnExit": false,
      "pauseBeforeReading_ms": 200,
      "readBufferBytes": 64,
      "readTimeout_ms": 2000,
      "readDelimiter": "0x0A",
      "readDelimiterMinCount": 1,
      "keys": {
        "BELL": {
          "keyName": "KEY_SOUND" 
        }
      }
    }

.. autoclass:: UartParameters
   :members:
   :special-members: __init__

.. autoclass:: UartConfig
   :members:
   :special-members: __init__

Main classes
------------

.. autoclass:: UartContext
   :members:
   :special-members: __init__

