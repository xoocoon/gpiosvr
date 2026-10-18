# gpiosvr
An asyncio-based Python framework and server providing uniform access to the GPIOs of a Linux single-board computer (SBC) or a microcontroller.

Main features:

* Read/write access to the GPIOs of a device running Linux, allowing simultaneous access by multiple processes.
* Integration with Raspberry Pico boards and other RP2040- and RP2350-based microcontrollers connected to a Linux host over USB 2.0.
* Capture and decoding of input sequences from GPIO and UART lines, with the decoded input reported to the Linux kernel as key presses.

These features make *gpiosvr* a versatile framework for IoT and home automation projects, supporting various input sources and actuators.

Developed and tested for Raspberry Pi variants, namely:

* Pi 3B
* Pi 4B 
* Pico 1 (connected over USB 2.0)
* Pico 2 (connected over USB 2.0)

On the Pi 3B and Pi 4B, [pigpiod](https://abyz.me.uk/rpi/pigpio/pigpiod.html) is used as a backend for GPIO access. On the Pico, the [picossa](https://xoocoon.github.io/gpiosvr/html/picossa.html) module is used as a backend. It is a custom development based on the C daemon from [picod project](https://abyz.me.uk/picod/index.html). Support for *libgpiod* via [gpiod](https://pypi.org/project/gpiod/) is under development.

In a minimal setup, a single instance of the `gpio_server.py` executable accesses a built-in GPIO chip of the Linux host. To connect to the server, use the `pigs` executable. It can read and write GPIO levels, among other features.

Somewhat more advanced is the capture and decoding of input from peripheral hardware. For example, this might be an infrared receiver connected to a GPIO or an RFID reader sending data over UART. Once the corresponding technical parameters are defined, a GPIO server can report any expected input to the Linux kernel as key presses. For an introduction to the basic concepts, see the documentation of the [input](https://xoocoon.github.io/gpiosvr/html/input.html#core-concepts) module.

A mixed setup with a built-in GPIO chip and a GPIO extension in the form of a Raspberry Pico is also possible. In this case, one instance of `gpio_server.py` is connected to the built-in GPIO chip, and another one is connected to the Pico device over USB. The following example shows how to connect to both servers with the `pigs` utility:

```
# Connect to the default GPIO server:
pigs w 18 1                                 # set GPIO 18 to level 1 (high)
# Explicitly provide the target server's Unix socket:
pigs --socket=/var/run/pico/pico.sock r 23  # read the level of GPIO 23
```
See [Environment configuration](#environment-configuration) for the configuration of a default GPIO server.

Since *gpiosvr* is based on asyncio, and asyncio in turn is backed by a C implementation in CPython, it runs fast enough for most use cases. As an example, infrared signals from remote controls with pulse bursts in the range of microseconds can easily be decoded on a strong-enough Linux host.

For API documentation, see [https://xoocoon.github.io/gpiosvr](https://xoocoon.github.io/gpiosvr/html/).

## Server configuration

Various types of input can be captured and decoded with a GPIO server. The following is an examplary configuration for an analog door bell signal. Normally, an analog signal cannot be connected directly to a GPIO pin of a Raspberry Pi, so a rectifier and a Z diode might be put in series before. With the following JSON declarations, a `KEY_SOUND` key press is reported when the door bell is ringing:

```
{
  "evdevName": "pi-sig-injector",
  "signal": {
    "door-bell": {
      "basePulse_μs": 9000,
      "tolerancePercentage": 30,
      "basePulseCount": 5,
      "preamble_ms": 10,
      "postamble_ms": 12,
      "codeStartBit": 1,
      "debounce_μs": 600
    }
  },
  "iterators": {
    "door-bell": {
      "gpio": 25,
      "mayInject": false
    }
  },
  "keys": {
    "door-bell": {
      "0x1f": {
        "keyNames": "KEY_SOUND"
      }
    }
  }
}
```

The configuration reads as follows: Whenever at least 5 pulses with a length of 9000 microseconds each, are received on GPIO 25 with a timing tolerance of plus/minus 30%, a pair of a key down and a key up event for `KEY_SOUND` is exposed via the `/dev/input/event*` device logically named `pi-sig-injector`. 

To trigger an action upon a key press, *gpiosvr* provides the `key_monitor.py` executable. It may be run in one or more instances, each grabbing and processing an arbitrary subset of key presses. Without a key monitor, the effect is no different to really pressing a key on an attached keyboard. This opens up a versatile way for decoupling signal transmitters and receivers in an ecosystem of Linux daemons, optionally orchestrated by systemd.

Another input source might be an infrared remote control. In this case, no extra key monitor is required when there is already a media center processing the key presses. The following is an example for capturing the pulses of an MCE-compatible remote control.

```
{
  "evdevName": "pi-ir-injector",
  "ir": {
    "Media Center Edition": {
      "baseProtocol": "RC6_MCE",
      "address": "0x800f"
    }
  },
  "iterators": {
    "Media Center Edition": {
      "gpio": 19,
      "mayInject": true
    }
  },
  "keys": {
    "Media Center Edition": {
      "0x0411": {
        "keyName": "KEY_VOLUMEDOWN"
      },
      "0x0410": {
        "keyName": "KEY_VOLUMEUP"
      }
    }
  }
}
```

By contrast, this configuration relates to an infrared iterator instead of a generic signal iterator. This demonstrates that *gpiosvr* can be extended to support additional input types.

More complete configuration examples are included in the repo under `template/`:

- `pi_signal_config.json` includes an example for a light sensor, simply reporting any GPIO edge as a key press.
- `pico_signal_button_config.json` includes an example for reporting push button presses, including repeated key presses as long as a button is held down.
- `pi_signal_ir_config.json` includes an example for mapping RC6_MCE scan codes to key names.

The key definitions used by *gpiosvr* correspond to the constants used by the Linux kernel, as defined in [input-event-codes.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h). In the examples above, these are `KEY_SOUND` and `KEY_VOLUMEUP`, `KEY_VOLUMEDOWN`, respectively.

For a reference of configuration entries, see the documentation of the [input](https://xoocoon.github.io/gpiosvr/html/input.html#configuration-classes) and  [signal](https://xoocoon.github.io/gpiosvr/html/signal.html#configuration-classes) modules.


## Key handling configuration

A key monitor executes pre-defined commands on the Linux host whenever a specific key press occurs. The following is an example for commands to execute whenever the button F12 is pressed (`shortPressCommands`) or held down (`longPressCommands`).

```
"keys": {
  "KEY_F12": {
    "shortPressCommands": [
      [
        "/usr/bin/curl",
        "--netrc-file",
        "/mnt/encrypted/rest.netrc",
        "https://mediabox.local:6863/standby"
      ]
    ],
    "longPressCommands": [
      [
        "/usr/bin/curl",
        "--netrc-file",
        "/mnt/encrypted/rest.netrc",
        "https://mediabox.local:6863/shutdown"
      ]
    ],
    "ledColor": "orange"
  }
}
```

The example also shows that an indicator LED can be synchronized with key presses. If you come to the conclusion that a key monitor can be used independently of a GPIO server, you are right. Imagine a little keypad connected to your Linux SBC. As the keypad already produces genuine key events in the Linux kernel, you do not need a GPIO server in the first place. It is sufficient to set up a a key monitor along with a configuration file to execute pre-defined commands.

A more complete example is included in the repo under `template/`:

- `key_rc6-mce_config.json` includes an example for controlling a home theater installation via an infrared remote control.

For a reference of configuration entries, see the documentation of the [key](https://xoocoon.github.io/gpiosvr/html/key.html#configuration-classes) module.

# Installation

## Package dependencies

The following packages need to be installed:

- [ctlbase](https://pypi.org/project/ctlbase/), as a framework for CLI utilities and service daemons
- [evdev](https://pypi.org/project/evdev/), for handling key events
- [systemd-python](https://pypi.org/project/systemd-python/), for running `gpio_server.py` and/or `key_monitor.py` as systemd service units
- [pyserial](https://pypi.org/project/pyserial/), if the [picossa](https://xoocoon.github.io/gpiosvr/html/picossa.html) module shall be used as a GPIO backend for Raspberry Pico
- [pigpio](https://abyz.me.uk/rpi/pigpio/download.html), if *pigpiod* shall be used as a GPIO backend for Raspberry Pi up to 4B

If you install *gpiosvr* in a virtual Python environment, as recommended under [Package installation](#package-installation), dependencies will be installed automatically as needed.

On a Debian-based system, the dependencies can be installed as follows:

```
sudo apt update && sudo apt install -y pigpiod gpiod python3-libgpiod python3-evdev python3-serial python3-systemd
```

Note that the custom package `ctlbase` is not available as a Debian package but can be installed via [PyPI](https://pypi.org/project/ctlbase/).

## Linux prerequisites

The Linux kernel on the host needs to run the `uinput` kernel module. It is by default included in most Linux distributions. You can check its availability at runtime with the following commands.

```
lsmod | grep uinput  # must yield an output starting with 'uinput'
ls /dev/uinput       # must yield the output '/dev/uinput'
```

Note that if you prefer running a `gpio_server.py` instance with a user other than root, you must make sure the user is member of the `input` group. Otherwise it is not able to access the `/dev/uinput` device. If your host system assigns a group other than `input` to the `/dev/uinput` device, use that group instead.

## Package installation

For installing the *gpiosvr* package you have at least two choices:

* Install the latest release from PyPI via `pip`.
* Install a local clone of the GitHub repo.

In both cases, it is recommended to use a virtual Python environment to encapsulate all the required dependencies. See [https://docs.python.org/3/library/venv.html](https://docs.python.org/3/library/venv.html) for details on how to create one.

For installing from PyPI, use the `pip` command of your target environment as follows:

```
pip install gpiosvr
```

For installing a local clone of the GitHub repo, use the `pip` command of your target environment as follows:

```
pip install $GPIOSVR
```

... where `$GPIOSVR` resolves to the root directory of the repo containing the `pyproject.toml` file.

In both cases, if you want to deploy `gpio_server.py` and/or `key_monitor.py` as systemd service units, install the optional `systemd` dependency by appending `[systemd]` to the command. Likewise, if you want to use *pigpiod* as a GPIO backend, install the optional `pigpio` dependency by appending `[pigpio]` to the command. For installing both, append `[systemd,pigpiod]`.

Note that for building *systemd-python*, the systemd headers must be available on the system. On Debian and derivatives, you can install them via `sudo apt update && sudo apt install -y libsystemd-dev`.

> [!WARNING]
> On some Linux distributions, Python packages are managed by the system's package manager, especially Debian and derivatives. Installing *gpiosvr* globally could break these system-wide packages. On these systems, a virtual Python environment is strongly recommended.


```
sudo pip install $GPIOSVR_PATH --dry-run
```

## Executables installation

Installing the entire package, as described under [Package installation](#package-installation), also installs the executable scripts to a common `bin` directory. In a virtual Python environment, this normally is its `bin` directory.

If you want to make the scripts available system-wide, you can sym-link them to `/usr/local/bin` and add that directory to your `$PATH`:

```
for exe in gpio_server key_monitor leds pigs; do
  sudo ln -s $VENV_PATH/bin/$exe /usr/local/bin/
done

PATH="$PATH:/usr/local/bin"
```

... where `$VENV_PATH` resolves to the root directory of the virtual Python environment containing the `pyvenv.cfg` file.

To test the setup, simply call `pigs info`. It should print an info string from the connected GPIO server.

All executables support the `--help` argument which shows the available command line arguments.

They can also be looked up in the documentation:

* [gpio_server](https://xoocoon.github.io/gpiosvr/html/gpio_server.html)
* [key_monitor](https://xoocoon.github.io/gpiosvr/html/key_monitor.html)
* [pigs](https://xoocoon.github.io/gpiosvr/html/pigs.html)
* [leds](https://xoocoon.github.io/gpiosvr/html/leds.html)

## Environment configuration

Based on the [ctlbase.config](https://xoocoon.github.io/ctlbase/html/config.html) module, executables like `pigs` can auto-discover environment configuration files. These follow a basic Bash syntax for variable declarations. The following example sets the default for the GPIO server to use with the `pigs` command:

```
GPIO_SOCKET_PATH=/var/run/pico/pico.sock
```

For the `pigs` executable to find it, this line must be placed somewhere up the directory tree in a filenamed `etc/pigs` or `etc/default/pigs`. Alternatively, for a system-wide setting, it can be placed in `/etc/default`.

## systemd service setup

Under `template/`, you will find systemd templates for deploying `gpio_server.py` and `key_monitor.py` as systemd service units. If you do not want to use systemd as an init system, you can still use the `*.service` files as a reference for valid command lines.

The files `pi_server.service` and `pico_server.service` serve as examples for services connected to a built-in GPIO chip of a Raspberry Pi, and a Pico device, respectively. Please adjust the contents of the files according to your environment and deploy them to `/etc/systemd/system/`. Then have systemd recognize the new service(s) by executing `sudo systemctl daemon-reload`.

In the case of `pi_server.service`, have systemd start up the service automatically by executing `sudo systemctl enable pi_server.service`.

By contrast, it is **not** recommended to have `pico_server.service` start up automatically. In the system boot process, a race condition might occur between the startup of the service and the actual availability of the Pico device. Use an udev rule instead. A sample udev rule is included under `template/59-pico.rules`. It needs to be adjusted and deployed to `/etc/udev/rules.d/`. It will start up the service as soon as the specified Pico device becomes available.

`key_monitor.service` serves as an example for monitoring key presses, be it generated by an instance of `gpio_server.py` or not. Note that in the example, `After` and `Requires` dependencies are configured for `pi_server.service`. In the Pico case, change this to `pico_server.service`.

## Raspberry Pico setup

To make a Raspberry Pico device work with *gpiosvr*, the picod daemon must be deployed to it. It is recommended to use the [xoocoon version](https://github.com/xoocoon/picod) and follow its installation instructions.

Note that if you prefer running a `gpio_server.py` instance with a user other than root, you must make sure the user is member of the `dialout` group. Otherwise it is not able to access the `/dev/ttyACM*` device corresponding to the Pico device. If your host system assigns a group other than `dialout` to `/dev/ttyACM*` devices, use that group instead.

# Project state

The code for the functionality described above already exists and has matured over years. Hopefully, I will find the time to curate and document the code up to a state eligible for its sharing as OSS.

Please feel free to contact me if the functionality might be useful to you. I might share portions of the code beforehand on a bilateral basis.

Final note: This project is unrelated to https://github.com/projectweekend/gpiosvr/.
