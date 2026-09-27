.. currentmodule:: gpiosvr.bin.gpio_server

gpio_server executable
======================

.. automodule:: gpiosvr.bin.gpio_server
   :no-undoc-members:

.. contents::
   :local:
   :depth: 2

Executable script operating a GPIO server as a daemon.

.. _gpio_server-features:

Features
--------

UART
^^^^

Sending and receiving data via UART is only supported for Pico devices. That is,
a GPIO backend must implement :class:`~.client_picod.PicodApi`.

Environment variables
---------------------

To use a running `pigpiod` daemon as a GPIO backend, its endpoint can be set in
the environment with the following variables:

* ``PIGPIO_ADDR`` – The host name or IP address of the `pigpiod` daemon to be
  used as a GPIO backend. The ``--pihost`` argument takes precedence, if present.
* ``PIGPIO_PORT`` – The TCP port of the `pigpiod` daemon to be used as a GPIO
  backend. The ``--piport`` argument takes precedence, if present.

Arguments
---------

To use a running `pigpiod` daemon as a GPIO backend, its endpoint can be set
with the following arguments:

* ``-n`` | ``--pihost`` (`str`) – The host name or IP address of the `pigpiod` 
  daemon to be used as a GPIO backend. 
* ``-p``| ``piport`` (`int` | `str`) – The TCP port of the `pigpiod` daemon to
  be used as a GPIO backend.

If using `picod` daemon and the :mod:`gpiosvr.picossa` module as a GPIO backend
on a Pico or Pico 2 device, the following arguments are valid:

* ``-u`` | ``--picouid`` (`int` | `str`) – The UID (`ID_SERIAL`) of the target 
  device, either as a 20-digit decimal number or a 16-digit hexadecimal number.
  It can be obtained by calling :meth:`~.PicoSerialAdapter.uid()` on the target
  device, or by executing ``udevadm info /dev/ttyACMx | grep ID_SERIAL`` on the
  command line, where ``/dev/ttyACMx`` represents the target device.
* ``--uartconfig`` (`str`) – Absolute path or filename (to be auto-detected)
  for the UART configuration, if one or two UART channels shall be opened.
* ``--conncheck`` (`int` | `str`) – The checking interval for the Pico
  connection in seconds. If, after the specified interval, the Pico target
  device is recognized as disconnected, the GPIO server is terminated. It
  defaults to 5 seconds.

The following arguments are independent from the GPIO backend:

* ``--signaldebug``, ``--buttondebug``, ``--irdebug`` (flags) – If present,
  debug output is written to the log file. The flags correspond to the activity
  of signal listeners (:class:`~.SignalListener`), button listeners
  (:class:`~.ButtonListener`) and IR listeners (:class:`~.IrListener`),
  respectively.
* ``--sockdebug`` (flag) – If present, debug output from the GPIO server's Unix
  socket is written to the log file. It mainly includes requests received via 
  the socket and the data written in response.
* ``--signalconfig``, ``--buttonconfig``, ``irconfig`` /`str`) – Absolute path
  to a configuration file or a filename for auto-discovery. The arguments
  correspond to a signal server configuration serving plain signals, button
  signals and IR signals, respectively.
* ``-g`` | ``--group`` (`str`) – Name of the group to which the GPIO server's
  Unix socket shall be accessible.
* Required positional argument – Absolute path to the Unix socket to be created
  by the GPIO server.

Additionally, the standard arguments from 
:external+ctl:meth:`ctlbase.shell.DaemonControlShell.addStandardArgs` are
accepted.

Main classes
------------

.. autoclass:: UnixSocketServer
   :members:
   :special-members: __init__


