# gpiosvr
An asyncio-based server providing uniform access to the GPIOs of a Linux SBC.

Main features:

* Simultaneous access to the GPIOs by multiple processes on the same machine.
* Uniform access to additional GPIOs of a Raspberry Pico, connected via USB 2.0.
* Translation of pre-defined signals to native Linux key events exposed via a `/dev/input/event*` device.

In essence, *gpiosvr* is a versatile framework for IoT and home automation projects, decoding and propagating signals from various sources and for controlling actuators.

Developed and tested for Raspberry Pi variants, namely:

* Pi 3B
* Pi 4B 
* Pico 1 (connected via USB 2.0)

On the Pi 3B and Pi 4B, [pigpiod](https://abyz.me.uk/rpi/pigpio/pigpiod.html) is used as a backend for evaluating GPIO edges. On the Pico, the [picossa](https://xoocoon.github.io/gpiosvr/html/picossa.html) module is used as a backend. It is a custom development based on the C daemon from [picod project](https://abyz.me.uk/picod/index.html). 

For a minimal setup, one running instance of `gpio_server.py` is required on a Linux SBC with a built-in GPIO chip. It can be accessed with the `pigs` executable that reads and writes GPIO levels, among other features. More advanced is the definition of signals that may be received and translated into native Linux key events. A signal in this context is a specific pattern of GPIO level changes, with a specific tolerance. 

In a setup with an additional Raspberry Pico, one instance of `gpio_server.py` is attached to the built-in GPIO chip of the host Linux machine, and another one is attached to the GPIOs of the Pico device. The following example shows how to connect to both servers with the `pigs` utility:

```
# Connect to the default GPIO server:
pigs w 18 1                                 # set GPIO 18 to level 1 (high)
# Explicitly provide the target server's Unix socket:
pigs --socket=/var/run/pico/pico.sock r 23  # read the level of GPIO 23
```
See [Environment configuration](#environment-configuration) for the configuration a default GPIO server.

Since *gpiosvr* is based on asyncio, and asyncio in turn is backed by a C implementation in CPython, it runs fast enough for most use cases, even on Single Board Computers (SBCs). As an example, infrared signals from remote controls with pulse bursts in the range of microseconds can easily be decoded.

For apidocs, see [https://xoocoon.github.io/gpiosvr](https://xoocoon.github.io/gpiosvr/html/).

## Signal definition language

Various signal sources can be used with a `gpio_server.py` instance. For translating pulsed and other signals into key events, a GPIO server requires protocol definitions in one or more JSON files. The following is an example for capturing a pulsating door bell signal. On the hardware side, whenever the door bell rings, a rectifier and Z diode emits pulses to GPIO 25. With the following JSON declarations, a `KEY_SOUND` key press is emulated.

```
{
  "evdevName": "pi-sig-injector",
  "gpios": {
    "25": "door-bell"
  },
  "signalProtocols": {
    "door-bell": {
      "basePulse_μs": 9000,
      "tolerancePercentage": 30,
      "basePulseCount": 5,
      "preamble_ms": 10,
      "postamble_ms": 12,
      "codeStartBit": 1,
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
```

The configuration reads as follows: Whenever at least 5 pulses with a length of 9000 microseconds each, are received on GPIO 25 with a timing tolerance of plus/minus 30%, a pair of a key down and a key up event for `KEY_SOUND` shall be exposed via the `/dev/input/event*` device logically named `pi-sig-injector`. 

Another service on the same Linux machine must evaluate the key events to trigger a specific action. 
To this end, *gpiosvr* features an additional service `key_monitor.py`. It may be deployed in one or more instances, each grabbing and processing an arbitrary subset of key events. Unless such a key monitor is deployed, the effect is no different to really pressing a key on a keyboard. This opens up a versatile way for decoupling signal transmitters and receivers in an ecosystem of Linux deamons, optionally orchestrated by systemd.

Another signal source might be an IR remote control. In this case, no extra key monitoring service is required when there is already a media center software processing the key events. The following is an example for capturing the pulses of an MCE-compatible remote control.

```
{
  "evdevName": "pi-ir-injector",
  "gpios": {
    "19": "Media Center Edition"
  },
  "irProtocols": {
    "Media Center Edition": {
      "baseProtocol": "RC6_MCE",
      "address": "0x800f",
      "irListener": {
        "mayInject": true
      },
      "keys": {
        "0x0411": {
          "keyName": "KEY_VOLUMEDOWN"
        },
        "0x0410": {
          "keyName": "KEY_VOLUMEUP"
        }
      }
    }
  }
}
```

By contrast to the previous example, the configuration relates to a specific IR protocol handler instead of a generic signal handler. This demonstrates that *gpiosvr* can be extended by individual protocol handlers and corresponding JSON configurations.

More complete examples are included in the repo under `templates/`:

- `pi_signal_config.json` includes an example for a light sensor, simply translating any GPIO edge into a pair of key down and key up events.
- `pico_signal_button_config.json` includes an example for translating push button presses, including repeated key events as long as a button is held down.
- `pi_signal_ir_config.json` includes an example for mapping the hex codes produced by a RC6_MCE IR handler to key names.

For a reference of supported configuration keys, please see the documentation of the [signal](https://xoocoon.github.io/gpiosvr/html/signal.html#configuration-classes) module.

## Key definitions

The key definitions used by *gpiosvr* correspond to the constants used by the Linux kernel, as defined in [input-event-codes.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h).

In the examples above, these are `KEY_SOUND` and `KEY_VOLUMEUP`, `KEY_VOLUMEDOWN`, respectively.

## Key processing language

An instance of `key_monitor.py` executes pre-defined commands on the Linux machine whenever a specific key event occurs. The following is an example for commands to execute whenever the button F12 is pressed (`shortPressCommands`) or held down (`longPressCommands`).

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

The example also shows that an indicator LED can be aligned with key events. If you come to the conclusion that a key monitor can be used independently of a GPIO server, you are right. Imagine a little keypad connected to your single board computer. As the keypad already produces genuine key events in the Linux kernel, you do not need a GPIO server in the first place. It is sufficient to set up a a key monitor along with a configuration file like the one above to execute pre-defined commands.

A more complete example is included in the repo under `templates/`:

- `key_rc6-mce_config.json` includes an example for controlling a home theater installation via an IR remote control.

For a reference of supported configuration keys, please see the documentation of the [key](https://xoocoon.github.io/gpiosvr/html/key.html#configuration-classes) module.

# Installation

## Package dependencies

The following packages need to be installed:

- [ctlbase](https://github.com/xoocoon/ctlbase), for basing executable scripts
- [evdev](https://pypi.org/project/evdev), for handling key events
- [systemd-python](https://pypi.org/project/systemd-python/), for running `gpio_server.py` and/or `key_monitor.py` as systemd service units
- [pyserial](https://pypi.org/project/pyserial), if the [picossa](https://xoocoon.github.io/gpiosvr/html/picossa.html) module shall be used as a GPIO backend for Raspberry Pico
- [pigpio](https://abyz.me.uk/rpi/pigpio/download.html), if *pigpiod* shall be used as a GPIO backend for Raspberry Pi up to 4B
- [libgpiod](https://libgpiod.readthedocs.io/en/latest/python_api.html), if *libgpiod* shall be used as a GPIO backend – **Not yet integrated.**

If you install *gpiosvr* in a virtual Python environment, as recommended under [Package installation](#package-installation), dependecies will be installed automatically as needed.

For completeness, on a Debian-based system, the dependencies can be installed as follows:

```
sudo apt update && sudo apt install -y pigpiod gpiod python3-libgpiod python3-evdev python3-serial python3-systemd
```

Note that the custom package `ctlbase` is not available as a Debian package.

## Linux prerequisites

The Linux kernel on the host machine needs to run the `uinput` kernel module. It is by default included in most Linux distributions. You can check its availability at runtime with the following commands.

```
lsmod | grep uinput  # must yield an output starting with 'uinput'
ls /dev/uinput       # must yield the output '/dev/uinput'
```

## Package installation

For installing the *gpiosvr* package you have at least to choices:

* Install the latest release from PyPI via `pip`.
* Install directly from a local clone of the GitHub repo.

In both cases, it is recommended to use a virtual Python environment to encapsulate all the required dependencies. See [https://docs.python.org/3/library/venv.html](https://docs.python.org/3/library/venv.html) for details on how to create one.

For installing from PyPI, use the `pip` command of your Python environment as follows:

```
pip install gpiosvr
```

For installing from a local clone of the GitHub repo, use the `pip` command of your Python environment as follows:

```
pip install $GPIOSVR
```

... where `$GPIOSVR` resolves to the root directory of the repo containing the `pyproject.toml` file.

In both cases, if you want to deploy `gpio_server.py` and/or `key_monitor.py` as systemd service units, install the optional `systemd` dependency by appending `[systemd]` to the command. Likewise, if you want to use *pigpiod* as a GPIO backend, install the optional `pigpio` dependency by appending `[pigpio]` to the command. For installing both, append `[systemd,pigpiod]`.

Note that for building *systemd-python*, the systemd headers must be available on the system. On Debian and derivatives, you can install them via `sudo apt update && sudo apt install -y libsystemd-dev`.

> [!WARNING]
> On some Linux distributions, Python packages are managed by the system's package manager, especially Debian and derivatives. Installing *gpiosvr* globally could break these system-wide packages. If you want to take the risk, install the system-wide packages listed under [Package dependencies](#package-dependencies). Then "dry-run" the *gpiosvr* installation first, to see if these system-wide packages satisfy *gpiosvr*'s dependencies. If only `Requirement already satisfied` messages occur, you may proceed with the actual installation.

For "dry-running" a global installation, consider the following command:

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

Additionally, they can be looked up in the apidocs:

* [gpio_server](https://xoocoon.github.io/gpiosvr/html/gpio_server.html)
* [key_monitor](https://xoocoon.github.io/gpiosvr/html/key_monitor.html)
* [pigs](https://xoocoon.github.io/gpiosvr/html/pigs.html)
* [leds](https://xoocoon.github.io/gpiosvr/html/leds.html)

## Environment configuration

Based on the [ctlbase.config](https://xoocoon.github.io/ctlbase/html/config.html) module, executables like `pigs` can auto-discover environment configuration files. These follow a basic Bash syntax for variable declarations. The following example sets the default for the GPIO server to use with the `pigs` command:

```
GPIO_SOCKET_PATH=/var/run/pico/pico.sock
```

For the `pigs` executable to find it, this line must be placed somewhere up the directory tree in a file named `etc/pigs` or `etc/default/pigs`. Alternatively, for a system-wide setting, it can be placed in `/etc/default`.

## systemd service setup

Under `templates/`, you will find systemd templates for installing `gpio_server.py` and `key_monitor.py` as systemd service units. If you do not want to use systemd as an init system, you can still use the `*.service` files as a reference for valid command lines.

The files `pi_server.service` and `pico_server.service` serve as examples for services attached to the built-on GPIO chip of a Raspberry Pi, and a Raspberry Pico, respectively. Please adjust the contents of the files according to your environment and deploy them to `/etc/systemd/system/`. Then have systemd recognize the new service(s) by executing `sudo systemctl daemon-reload`.

In the case of `pi_server.service`, have systemd start up the service automatically by executing `sudo systemctl enable pi_server.service`.

By contrast, it is **not** recommended to have `pico_server.service` start up automatically. In the system boot process, a race condition might occur between the startup of the service and the actual availability of the Pico device. Use an udev rule instead. A sample udev rule is included under `templates/59-pico.rules`. It needs to be adjusted and deployed to `/etc/udev/rules.d/`. It will start up the service as soon as the specified Pico device becomes available.

`key_monitor.service` serves as an example for monitoring a `/dev/input/event*` device, be it generated by an instance of `gpio_server.py` or not. Note that in the example, `After` and `Requires` dependencies are configured for `pi_server.service`. In the Pico case, change this to `pico_server.service`.

## Raspberry Pico setup

To make a Raspberry Pico device work with *gpiosvr*, the microcode contained in `picod/picod.uf2` must be deployed to it. For that, hold down the "BOOT/SEL" button on the Pico, while connected it to a host machine via USB 2.0. In a file browser of your choice, copy the file to the `RPI-RP2` volume. After the automatic reboot, the Pico will be ready to communicate with an accordingly configured GPIO server.

The source code for the microcode was obtained from the [picod project](https://abyz.me.uk/picod/index.html).

# Project state

The code for the functionality described above already exists and has matured over years. Hopefully, I will find the time to curate and document the code up to a state eligible for its sharing as OSS.

Please feel free to contact me if the functionality might be useful to you. I might share portions of the code beforehand on a bilateral basis.

Final note: This project is unrelated to https://github.com/projectweekend/gpiosvr/.
