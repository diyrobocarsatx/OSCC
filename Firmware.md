# Firmware

There are four hardware modules, each with its own firmware:

* CAN Gateway (Arduino Uno + Two [[Seeed Studio CAN Shields|https://www.seeedstudio.com/CAN-BUS-Shield-V1.2-p-2256.html]])
* Brake (Arduino Mega + OSCC Actuator Control Board)
* Steering (Arduino Uno + OSCC Sensor Interface Board)
* Throttle (Arduino Uno + OSCC Sensor Interface Board)

## Compatibility

Your hardware version is printed on the front of the OSCC shield.

### Kia Soul (2014-2017)

#### Steering (Sensor Interface Board)
| Board Version | Firmware Version |
| ------------- | ---------------- |
| >= 1.0.0      | >= 1.0.0         |

#### Throttle (Sensor Interface Board)
| Board Version | Firmware Version |
| ------------- | ---------------- |
| >= 1.0.0      | >= 1.0.0         |

#### Brake (Actuator Control Board)
| Board Version | Firmware Version |
| ------------- | ---------------- |
| 1.0.0 - 1.0.1 | >= 1.0.1 __*__       |
| >= 1.0.0      | >= 1.0.0         |

__*__ *Later versions of the actuator control board utilize new pins to perform additional safety checks on startup. To use the new firmware on older models, please see additional instructions in the [build](#startup-test) section.*

## Building and Uploading Firmware

The OSCC Project is built around a number of individual firmware modules that inter-operate to allow communication with your vehicle.
These modules are built from Arduinos and Arduino shields designed specifically for interfacing with various vehicle components.
Once these modules have been installed in the vehicle and flashed with the firmware, the API can be used to
receive reports from the car and send spoofed commands.

### Pre-requisites

You must have Arduino Core and CMake (version 2.8 or greater) installed on
your machine.

```
sudo apt install arduino-core cmake
```

OSCC uses CMake to avoid some of the limitations of the Arduino IDE. Using this method you can build
and upload the firmware from the command-line.

Check out [Arduino CMake](https://github.com/queezythegreat/arduino-cmake) for more information.

### Building the Firmware

Navigate to the `firmware` directory and create a build directory inside of it:

```
cd firmware
mkdir build
cd build
```

To generate Makefiles, tell CMake which platform to build firmware for. For example, if you want to build
firmware for the Kia Soul:

```
cmake .. -DKIA_SOUL=ON
```

By default, your firmware will have debug symbols which is good for debugging but increases
the size of the firmware significantly. To compile without debug symbols and optimizations
enabled, use the following instead:

```
cmake .. -DKIA_SOUL=ON -DCMAKE_BUILD_TYPE=Release
```

<a name="startup-test"></a>*For older (< 1.1.0) versions of the actuator control board, you need to set an additional flag using `cmake .. -DKIA_SOUL=ON -DBRAKE_STARTUP_TEST=OFF` to disable startup tests that are not compatible with the previous board design.*

This will generate the necessary files for building.

Now you can build all of the firmware with `make`:

```
make
```

If you'd like to build only a specific module, you can provide a target name to
`make` for whichever module you'd like to build:

```
make brake
make gateway
make steering
make throttle
```

### Uploading the Firmware

Once the firmware is successfully built, you can upload it. When you connect to
an Arduino with a USB cable, your machine assigns a serial device to it with the
path `/dev/ttyACM#` where `#` is a digit starting at 0 and increasing by one with
each additional Arduino connected.

You can upload firmware to a single module or to all modules. By default, CMake
is configured to expect each module to be `/dev/ttyACM0`, so if you connect a
single module to your machine, you can flash it without changing anything:

```
make throttle-upload
```

However, if you want to flash all modules, you need to change the ports in
CMake for each module to match what they are on your machine. The easiest way
is to connect each module in alphabetical order (brake, CAN gateway, steering,
throttle) so that they are assigned `/dev/ttyACM0` through `/dev/ttyACM3` in
a known order. You can then change the ports during the `cmake ..` step:

```
cmake .. -DKIA_SOUL=ON -DSERIAL_PORT_BRAKE=/dev/ttyACM0 -DSERIAL_PORT_CAN_GATEWAY=/dev/ttyACM1 -DSERIAL_PORT_STEERING=/dev/ttyACM2 -DSERIAL_PORT_THROTTLE=/dev/ttyACM3
```

Then you can flash all with one command:

```
make all-upload
```

Sometimes it takes a little while for the Arduino to initialize once connected, so if there is an
error thrown initially, wait a little bit and then retry the command.

### Monitoring Arduino Modules

It is sometimes useful to monitor individual Arduino modules to check for proper operation and to
debug. If the modules have been built with the flag `-DCMAKE_BUILD_TYPE=Debug`, their debug
printing functionality will be enabled and they will print status information to the serial interface.

The GNU utility `screen` is one option to communicate with the Arduino via serial over USB. It can
be used to both receive the output of any `Serial.print` statements in your Arduino code, and to
push commands over serial to the Arduino. If you don't already have it installed,
you can get it with the following command:

```
sudo apt install screen
```

You need to enable debug mode with `-DDEBUG=ON` and tell CMake what serial port
the module you want to monitor is connected to
(see [section on uploading](#uploading-the-firmware) for details on the default
ports for each module). The default baud rate is `115200` but you can change it:

```
cmake .. -DKIA_SOUL=ON -DDEBUG=ON -DSERIAL_PORT_THROTTLE=/dev/ttyACM0 -DSERIAL_BAUD_THROTTLE=19200
```

You can use a module's `monitor` target to automatically run `screen`, or a
module's `monitor-log` target to run `screen` and output to a file called
`screenlog.0` in the module's build directory:

```
make brake-monitor
make brake-monitor-log
```

You can exit `screen` with `C-a \`.

To do more in-depth debugging you can use any of a number of serial monitoring applications.
Processing can be used quite effectively to provide output plots of data incoming across a serial
connection.

Be aware that using serial printing can affect the timing of the firmware. You may experience
strange behavior while printing that does not occur otherwise.


### Tests

There are two types of tests available: unit and property-based.

Building and running the tests is similar to the firmware itself, but you must instead tell
CMake to build the tests instead of the firmware with the `-DTESTS=ON` flag. We also pass
the `-DCMAKE_BUILD_TYPE=Release` flag so that CMake will disable debug symbols and enable
optimizations, good things to do when running tests to ensure nothing breaks with
optimizations. Lastly, you must tell the tests which vehicle header to use for
the tests (e.g., `-DKIA_SOUL=ON`).

```
cd firmware
mkdir build
cd build
cmake .. -DTESTS=ON -DCMAKE_BUILD_TYPE=Release -DKIA_SOUL=ON
```

#### Unit Tests

Each module has a suite of unit tests that use **Cucumber** with **Cgreen**. There are prebuilt
64-bit Linux versions in `firmware/common/testing/framework`. Boost is required for Cucumber-CPP
and has been statically linked into `libcucumber-cpp.a`. If you need to build your own versions
you can use the provided script `build_test_framework.sh` which will install the Boost dependencies
(needed for building), clone the needed repositories with specific hashes, build the Cgreen and
Cucumber-CPP libraries, and place static Boost in the Cucumber-CPP library. The built will be placed
in an `oscc_test_framework` directory in the directory that you ran the script from. You can then copy
`oscc_test_framework/cucumber-cpp` and `oscc_test_framework/cgreen` to
`firmware/common/testing/framework`.

You must have **Cucumber** installed to run the tests:

```
sudo apt install ruby-dev
sudo gem install cucumber -v 2.0.0
```

We can run all of the unit tests available:

```
make run-unit-tests
```

Each module's test can also be run individually:

```
make run-brake-unit-tests
make run-can-gateway-unit-tests
make run-steering-unit-tests
make run-throttle-unit-tests
```

If everything works correctly you should see something like this:

```
# language: en
Feature: Receiving commands

  Commands received from a controller should be processed and acted upon.

  Scenario Outline: Enable throttle command sent from controller        # firmware/throttle/tests/features/receiving_commands.feature:8
    Given throttle control is disabled                                  # firmware/throttle/tests/features/receiving_commands.feature:9
    And the accelerator position sensors have a reading of <sensor_val> # firmware/throttle/tests/features/receiving_commands.feature:10
    When an enable throttle command is received                         # firmware/throttle/tests/features/receiving_commands.feature:12
    Then control should be enabled                                      # firmware/throttle/tests/features/receiving_commands.feature:14
    And the last command timestamp should be set                        # firmware/throttle/tests/features/receiving_commands.feature:15
    And <dac_a_val> should be written to DAC A                          # firmware/throttle/tests/features/receiving_commands.feature:16
    And <dac_b_val> should be written to DAC B                          # firmware/throttle/tests/features/receiving_commands.feature:17

    Examples:
      | sensor_val | dac_a_val | dac_b_val |
      | 0          | 0         | 0         |
      | 256        | 1024      | 1024      |
      | 512        | 2048      | 2048      |
      | 1024       | 4096      | 4096      |
```

### Property-Based Tests

The throttle, steering, and brake modules, along with the PID controller library, also contain a series of
property-based tests.

These tests use [QuickCheck for Rust](http://burntsushi.net/rustdoc/quickcheck/), so **Rust** and **Cargo**
need to be [installed](https://www.rust-lang.org/en-US/install.html) in order to run them locally.


We can run all of the property-based tests available:

```
make run-property-tests
```

Each module's test can also be run individually:

```
make run-brake-property-tests
make run-steering-property-tests
make run-throttle-property-tests
make run-pid-library-property-tests
```

Once the tests have completed, the output should look similar to the following:

```
running 7 tests
test check_integral_term ... ok
test check_derivative_term ... ok
test check_proportional_term ... ok
test check_reversed_inputs ... ok
test check_same_control_for_same_inputs ... ok
test check_zeroize ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests tests

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

#### Run All Tests

Finally, you can run all available tests:

```
make run-all-tests
```


### Easier CMake Configuration

If you have a lot of `-D` commands to pass to CMake (e.g., configuring the serial
port and baud rates of all of the modules), you can instead configure with a GUI
using `cmake-gui`:

```
sudo apt install cmake-gui
```

Then use `cmake-gui` where you would normally use `cmake`:

```
cd firmware
mkdir build
cd build
cmake-gui ..
```

The GUI will open and you can change all of the options you would normally need
to pass on the command line. First, press the `Configure` button and then press
`Finish` on the dialog that opens. In the main window you'll see a list of
options that you can change that would normally be configured on the command line
with `-D` commands. When you're done, click `Configure` again and then click
the `Generate` button. You can then close `cmake-gui` and run any `make` commands
like you normally would.


## [[Brake|Firmware-Brake]]
## [[Steering|Firmware-Steering]]
## [[Throttle|Firmware-Throttle]]
## [[OBD Messages|Firmware-OBD]]
