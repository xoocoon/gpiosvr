.. currentmodule:: gpiosvr.picossa_pigs

picossa_pigs module
===================

.. automodule:: gpiosvr.picossa_pigs

With an instance of :class:`~.PigsExecutor`, it is possible to execute commands
and scripts like::

   m 12 w pud 12 o w 12 1

This sets GPIO 12 as an output GPIO, disables the internal resistor and sets it 
to level `1`.

.. contents::
   :local:
   :depth: 2

Classes
-------

.. autoclass:: PigsScript
   :members:
   :special-members: __init__
   
.. autoclass:: PigsExecutor
   :members:
   :special-members: __init__
   
Constants and defaults
----------------------

.. autodata:: CANCEL_CHECKING_INTERVAL_DEFAULT_μS

