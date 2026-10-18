.. currentmodule:: gpiosvr.input

input module
============

.. automodule:: gpiosvr.input

.. contents::
   :local:
   :depth: 2

.. _input-concepts:

Core concepts
-------------

.. _input-type:

Input type
^^^^^^^^^^

An input type specifies the input expected from a particular hardware source.
For a pulse generator such as an IR remote control, the input type corresponds
to the signal’s technical parameters, such as pulse durations and counts. For a 
UART channel, it corresponds to the connection parameters and the valid byte
sequences that can be received.

This is important for hardware initialization and for the decoding of actual
input chunks.

.. _input-code:

Scan code
^^^^^^^^^

When input chunks are received based on a specific input type, they are decoded 
to corresponding scan codes. This concept is known from keyboards and remote
controls, where a key or button triggers some electronic signals that appear as
a scan code in the kernel. Such a code is mapped to a logical key or character
like "A" or "2". For instance, the scan code of the "A" key on a US-American IBM
keyboard is ``0x1c``. In the scope of this module, an incoming byte sequence,
e.g. from a UART channel, is also called scan code.

An individual scan code may take one of the following forms:

  * A `str` starting with ``0x``, in which case it is interpreted as an
    unsigned integer in hexadecimal form.
  * Any other `str` in which case it is interpreted as a sequence of ASCII
    characters.
  * An `int`, in which case it is interpreted as an unsigned integer in decimal
    form.

The receiving and decoding of input chunks is done by so-called input iterators.

Input family
^^^^^^^^^^^^

Input types with similar technical parameters shape one input family. As a
defining characteristic, all the input types in a family can be handled by the
same input iterator class based on :class:`~.InputIterator`. As an example,
the RC6 infrared protocols might shape an input family ``rc6``, with input types
such as ``rc6-0``, ``rc6-6a-32`` or ``rc6-6a-mce``.

.. _input-alternate:

Alternate mode
^^^^^^^^^^^^^^

The code-to-keys mapping for a scan code may specify one or more alternate key
names via :attr:`~.CodeToKeysMapping.altKeyNames`. They are used instead of the
regular key names when the alternate mode is on.

The alternate mode is toggled via :meth:`~.InputServer.toggleAltMode`. Toggling
is also possible via a code-to-keys mapping with an
:attr:`~.CodeToKeysMapping.alias` value set to one of the members or member
names of :class:`~.AltAlias`.

The state of the alternate mode can be checked via
:attr:`~.InputServer.isAltModeOn`.

.. _input-extension:

Extension mechanism
-------------------

To create a custom extension for an input family, an input iterator class needs
to be created. For auto-discoverability, the following naming conventions apply:

* All classes must be contained in a module named after the input family. That
  is, for a input family named ``rc6``, the corresponding module must be named
  ``gpiosvr.rc6``. Optionally, the module name can have a custom prefix, e.g.
  ``signal_``, resulting in the module name ``gpiosvr.signal_rc6``.
* The input iterator class must be named after the input family and have the
  postfix ``EventIterator``. That is, for a input family named ``rc6``, the class
  name must be ``Rc6EventIterator``.

One instance of an input iterator class relates to one specific input type. To
capture the technical parameters of an input type, a class based on
:class:`~.InputParameters` must be included in a custom extension. For
auto-discoverability, the following naming convention applies:

* The class for technical parameters must be named after the input family and
  have the postfix ``Parameters``. That is, for a input family named ``rc6``,
  the class name must be ``Rc6Parameters``.

To provide an individual configuration to an input iterator, a class based on 
:class:`~.InputIteratorConfig` is needed. Such a class may be part of a custom
extension. For auto-discoverability, the following naming convention applies:

* The class for an input iterator configuration, if included, must be named
  after the input family and  have the postfix ``EventIteratorConfig``. That is, for 
  a input family named ``rc6``, the class name must be ``Rc6EventIteratorConfig``.

What is more, a custom extension is not limited to input iterators. It may also
include an input server class that is able to publish input events to the Linux
kernel. It must be based on :class:`~.InputServer`. To provide an individual
configuration to an input server, a class based on :class:`~.InputServerConfig`
is needed. The following naming conventions apply for auto-discoverability:

* The input server class, if included, must be named after the input family and
  have the postfix ``Server``. That is, for a input family named ``rc6``, the
  class name must be ``Rc6Server``.
* The class for an input server configuration, if included, must be named after 
  the input family and have the postfix ``ServerConfig``. That is, for a input
  family named ``rc6``, the class name must be ``Rc6ServerConfig``.

A predefined input family is ``signal``, embodied by the :mod:`~.signal`
module and its derivatives.

To bootstrap an input server from a custom extension, instances of the
corresponding configuration classes need to be created first. The most
convenient way is to load a server configuration from a file into a `dict` and 
provide it to :meth:`~.ExtensionHelper.createInputServers`. This yields a 
sequence of one or more input servers unless any error occurs. It is also
possibly to instantiate the :ref:`input-configuration` and the input iterator 
and/or input server classes programatically. See the `fromDict` methods of the
configuration classes as a starting point.

.. _input-configuration:

Configuration classes
---------------------

The configuration classes help to set up one or more input iterators and servers
via `dict` instances, e.g. parsed from JSON files, or programatically.

The following is the JSON representation of an exemplary input server 
configuration for the predefined input family ``signal``:

.. code-block:: none

  {
    "evdevName": "pi-sig-injector",
    "signal": {               // <- technical parameters for input family "signal"
      "door-bell": {          // <- technical parameters for input type "door-bell"
        "basePulse_μs": 9000,
        "basePulseCount": 5,
        "preamble_ms": 10,
        "postamble_ms": 12,
      },
      "light-sensor": {       // <- technical parameters for input type "light-sensor"
        "minEdgeCount": 1,
        "preamble_ms": 75,
        "postamble_ms": 75,
        "codeStartBit": "GPIO_LEVEL"
      }
    },
    "iterators": {            // <- input iterator configurations
      "door-bell": {          // <- iterator configuration for input type "door-bell"
        "gpio": 25,
        "mayInject": false
      }
    },
    "keys": {                 // <- code-to-keys mappings
      "door-bell": {          // <- code-to-keys mappings for input type "door-bell"
        "0x1f": {             // <- code-to-keys mapping for scan code "0x1f"
          "keyNames": "KEY_VOLUMEDOWN"
          "altKeyNames": "KEY_MUTE"
        }
      }
    }
  }

Auto-discovering and parsing JSON files is in the scope of the
:mod:`ctlbase.config` module.

.. autoclass:: InputParameters
   :members:
   :special-members: __init__

.. autoclass:: CodeToKeysMapping
   :members:
   :special-members: __init__

.. autoclass:: CodeToKeysMappings
   :members:
   :special-members: __init__

.. autoclass:: InputIteratorConfig
   :members:
   :special-members: __init__

.. autoclass:: InputServerConfig
   :members:
   :special-members: __init__

Main classes
------------

.. autoclass:: InputEvent
   :members:
   :special-members: __init__

.. autoclass:: InputIterator
   :members:
   :special-members: __init__, __anext__
   :show-inheritance:

.. autoclass:: InputServer
   :members:
   :special-members: __init__

Helper classes
--------------

.. autoclass:: ExtensionHelper
   :members:

Constants and defaults
----------------------

.. autoclass:: AltAlias
   :members:
