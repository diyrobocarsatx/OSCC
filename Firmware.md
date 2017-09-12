# Firmware

There are four hardware modules, each with its own firmware:

* CAN Gateway (Arduino Uno + Two [[Seeed Studio CAN Shields|https://www.seeedstudio.com/CAN-BUS-Shield-V1.2-p-2256.html]])
* Brake (Arduino Mega + OSCC Actuator Control Board)
* Steering (Arduino Uno + OSCC Sensor Interface Board)
* Throttle (Arduino Uno + OSCC Sensor Interface Board)

## [Compatibility](#1-firmware_compatibility)
## [Building and Uploading Firmware](#1-firmware_building-and-uploading-firmware)
## [Monitoring Arduino Modules](#1-firmware_monitoring-arduino-modules)
## [Tests](#1-firmware_tests)
## [Easier CMake Configuration](#1-firmware_easier-cmake-configuration)

# Versions

New versions of the API and the firmware are released periodically as new features are added and bugs are
fixed. **It is of vital importance that you update whenever there is a new version so that you can be certain
you are not using a version with known safety issues.**

There are four versions to be aware of:

* **Sensor Interface Board (throttle and steering):** the version is printed on the front of the shield

* **Vehicle Control Module (EV brakes):** the version is printed on the front of the shield

* **Actuator Control Board (petrol brakes):** the version is printed on the front of the shield

* **API and Firmware:** a single version is shared by both and listed in the [Releases section](https://github.com/PolySync/oscc/releases) of the repository.

The following table can be used to ensure that you use the appropriate firmware version with your boards:

|                        | Board Version | Firmware Version |
| ---------------------- | ------------- | ---------------- |
| Sensor Interface       | >= 1.0.0      | >= 1.0.0         |
| Vehicle Control        | 0.1.0         | >= 1.1.0         |
| Actuator Control       | >= 1.2.0      | >= 1.0.0         |
|                        | < 1.2.0 __*__ | >= 1.0.1         |

__*__ *Later versions of the Actuator Control Board utilize new pins to perform additional safety checks on startup. To use the new firmware on older boards, please see additional instructions in the [build](#brake-startup-test) section.*

## Building and Uploading Firmware

The OSCC Project is built around a number of individual firmware modules that inter-operate to allow communication with your vehicle.
These modules are built from Arduinos and Arduino shields designed specifically for interfacing with various vehicle components.
Once these modules have been installed in the vehicle and flashed with the firmware, the API can be used to
receive reports from the car and send spoofed commands.

### Prerequisites

You must have Arduino Core and CMake (version 2.8 or greater) installed on
your machine.

```
sudo apt install arduino-core build-essential cmake
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

To generate Makefiles, tell `cmake` which platform to build firmware for. For example, if you want to build
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

<a name="brake-startup-test"></a>
**NOTE:**
> For older (< 1.2.0) versions of the actuator control board, you need to set an additional flag using `cmake .. -DKIA_SOUL=ON -DBRAKE_STARTUP_TEST=OFF` to disable startup tests that are not compatible with the previous board design.

This will generate the necessary files for building.

Now you can build all of the firmware with `make`:

```
make
```

If you'd like to build only a specific module, you can provide a target name to
`make` for whichever module you'd like to build:

```
make brake
make can-gateway
make steering
make throttle
```

### Uploading the Firmware

Once the firmware is successfully built, you can upload it. When you connect to
an Arduino with a USB cable, your machine assigns a serial device to it with the
path `/dev/ttyACM#` where `#` is a digit starting at 0 and increasing by one with
each additional Arduino connected.

You can upload firmware to a single module or to all modules. By default, `cmake`
is configured to expect each module to be `/dev/ttyACM0`, so if you connect a
single module to your machine, you can flash it without changing anything:

```
make throttle-upload
```

However, if you want to flash all modules, you need to change the ports in
`cmake` for each module to match what they are on your machine. The easiest way
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

## Monitoring Arduino Modules

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

You need to enable debug mode with `-DDEBUG=ON` and tell `cmake` what serial port
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


## Tests

There are two types of tests available: unit and property-based.

### Test Dependencies

The unit tests and property-based tests each have their own set of dependencies
required to run the tests.

For the unit tests you must have **Cucumber 2.0.0** and its dependency **Boost** installed:

```
sudo apt install ruby-dev libboost-dev
sudo gem install cucumber -v 2.0.0
```

For the property-based tests you must have **Rust**, its build manager **Cargo**, and **libclang**:

```
sudo apt install rustc cargo clang libclang-dev
```

### Running Tests

Building and running the tests is similar to the firmware itself, but you must instead tell
`cmake` to build the tests instead of the firmware with the `-DTESTS=ON` flag. We also pass
the `-DCMAKE_BUILD_TYPE=Release` flag so that `cmake` will disable debug symbols and enable
optimizations, good things to do when running tests to ensure nothing breaks with
optimizations. Lastly, you must tell the tests which vehicle header to use for
the tests (e.g., `-DKIA_SOUL=ON`).

```
cd firmware
mkdir build
cd build
cmake .. -DTESTS=ON -DCMAKE_BUILD_TYPE=Release -DKIA_SOUL=ON
```

### Unit Tests

Each module has a suite of unit tests that use **Cucumber** with **Cgreen**. There are pre-built
64-bit Linux versions in `firmware/common/testing/framework`.

Boost is required for Cucumber-CPP and has been statically linked into `libcucumber-cpp.a`.
If you need to build your own versions you can use the provided script `build_test_framework.sh`
which will install the Boost dependencies (needed for building), clone the needed
repositories with specific hashes, build the Cgreen and Cucumber-CPP libraries,
and place static Boost in the Cucumber-CPP library.

The built libraries will be placed in an `oscc_test_framework` directory in the
directory that you ran the script from. You can then copy `oscc_test_framework/cucumber-cpp`
and `oscc_test_framework/cgreen` to `firmware/common/testing/framework`.

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

### All Tests

Finally, as a convenience you can run all available tests:

```
make run-all-tests
```
