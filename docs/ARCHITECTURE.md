# Serial Architecture

## Table of Contents <!-- omit in toc -->

- [Introduction](#introduction)
- [The Big Picture](#the-big-picture)
- [The Layers](#the-layers)
- [What a Caller May Rely On](#what-a-caller-may-rely-on)
  - [Timeouts](#timeouts)
  - [Exceptions](#exceptions)
  - [Threads](#threads)
- [A Read on Linux](#a-read-on-linux)
- [The Fork](#the-fork)
- [The Build](#the-build)
- [Who Uses It](#who-uses-it)
- [Tests](#tests)
- [How to Extend Serial](#how-to-extend-serial)
- [Design Decisions and Trade-offs](#design-decisions-and-trade-offs)

## Introduction

This document explains how this library is built, what this fork changes, and what a program that uses it may rely on. It describes the code at the commit `d8d1606`, the tip of the branch `ros2`. It assumes we have read the [README](../README.md).

The library is [wjwwood/serial](https://github.com/wjwwood/serial), a C++ serial port library for Linux, macOS and Windows, by William Woodall and John Harrison. This repository is a fork that changes how it is built, and nothing of what it does: see [The Fork](#the-fork). We did not write the code, so this document does not explain it line by line; it says what matters to a caller.

It follows one idea: **one class, `serial::Serial`, hides the platform, and every read and write ends within a timeout the caller sets**. A read that times out is not an error: it returns the bytes it got. An exception means the port itself failed.

## The Big Picture

The whole interface is one header, [`include/serial/serial.h`](../include/serial/serial.h), in the namespace `serial`:

| Name | Type | What it does |
|---|---|---|
| `Serial` | class | A serial port: `open()`, `close()`, `read()`, `write()`, the settings of the line, and the handshaking lines. It cannot be copied. |
| `Timeout` | struct | The five numbers, in milliseconds, that bound a read and a write. `Timeout::simpleTimeout(ms)` builds the usual one. |
| `SerialException`, `IOException`, `PortNotOpenedException` | classes | What the library throws, see [Exceptions](#exceptions). |
| `list_ports()` | function | The serial devices of the computer, each as a `PortInfo`: its path, a description and a hardware id. |
| `bytesize_t`, `parity_t`, `stopbits_t`, `flowcontrol_t` | enums | The settings of the line. The defaults are 9600 baud, 8 bits, no parity, 1 stop bit, no flow control. |

A `Serial` built with a port name opens it in its constructor, and throws if it cannot. Built without one, it stays closed until `setPort()` and `open()`. Changing a setting of an open port, e.g. `setBaudrate()`, applies it at once.

## The Layers

`Serial` holds a pointer to a `Serial::SerialImpl`, and forwards every call to it. There is one `SerialImpl` per platform, with the same methods, and the build compiles exactly one of them:

| File | Lines | Responsibility | Built on Linux |
|---|---|---|---|
| [`include/serial/serial.h`](../include/serial/serial.h) | 775 | The public interface. It names no platform type: `SerialImpl` is only declared. | yes |
| [`src/serial.cc`](../src/serial.cc) | 430 | `Serial`: the locks, the overloads of `read()` and `write()` for strings and vectors, `readline()` and `readlines()`. Everything else forwards. | yes |
| [`src/impl/unix.cc`](../src/impl/unix.cc), [`include/serial/impl/unix.h`](../include/serial/impl/unix.h) | 1066, 221 | `SerialImpl` for Linux and macOS, on `termios`, `pselect()` and `ioctl()`, and `MillisecondTimer`, the deadline of a read or a write, on `CLOCK_MONOTONIC`. | yes |
| [`src/impl/list_ports/list_ports_linux.cc`](../src/impl/list_ports/list_ports_linux.cc) | 335 | `list_ports()` for Linux: it globs `/dev/ttyACM*`, `/dev/ttyS*`, `/dev/ttyUSB*`, `/dev/tty.*` and `/dev/cu.*`, and reads the name and the USB ids of each from `/sys`. | yes |
| [`src/impl/win.cc`](../src/impl/win.cc), [`include/serial/impl/win.h`](../include/serial/impl/win.h), [`list_ports_win.cc`](../src/impl/list_ports/list_ports_win.cc) | 646, 207, 152 | The same for Windows. | no |
| [`list_ports_osx.cc`](../src/impl/list_ports/list_ports_osx.cc) | 286 | `list_ports()` for macOS, on IOKit. | no |

The platform is chosen twice, and both must agree: `CMakeLists.txt` picks the source files (`APPLE`, `UNIX`, `WIN32`), and `serial.cc` picks the header of `SerialImpl` (`_WIN32`). On Linux the library is the three source files marked above, 1831 lines, and links `rt` and `pthread`.

The headers of `include/serial/impl` are installed with the public one, but they are not an interface: a caller includes `serial/serial.h` only.

## What a Caller May Rely On

This section describes the Linux implementation, the only one we build.

### Timeouts

A `Timeout` gives a read its total time, `read_timeout_constant + read_timeout_multiplier × bytes asked`, and the longest silence between two bytes, `inter_byte_timeout`. A write has its own constant and multiplier.

- **A read returns when it has all the bytes asked, or when its time is over, whichever comes first.** It returns how many bytes it read, and never throws for a timeout. A caller that needs all of them compares the count.
- **A write returns how many bytes it wrote**, fewer than asked if the port did not take them in time.
- **`Timeout::simpleTimeout(ms)` is the one to use**: the total time for a read and for a write, and no limit between bytes.
- **A timeout of 0 does not block.** A `Serial` has this timeout until `setTimeout()` is called: a read returns the bytes already received, none if there are none.
- **`Timeout::max()` turns the inter-byte timeout off, 0 does not.** With an `inter_byte_timeout` of 0 and a total time above 0, a read that waits does not sleep: it polls until the total time is over. The comment of `setTimeout()` in the header says that 0 turns it off; the code and the comment of `Timeout` say `Timeout::max()`.
- **`flush()` waits, and has no timeout.** It calls `tcdrain()`, which returns when the bytes written have left the computer; it drops nothing. `flushInput()` and `flushOutput()` are the ones that drop bytes.
- **A read of several bytes may sleep on purpose.** With the inter-byte timeout off, a read of more than one byte waits the time the missing bytes take at the baud rate set, to fetch them in one call. A device that ignores the baud rate, e.g. a USB serial port, still pays this wait. A read of one byte never does.

### Exceptions

The three exception classes derive from `std::exception`, each on its own: there is no common `serial` base class, so a caller that wants them all catches `std::exception`.

| Exception | When |
|---|---|
| `PortNotOpenedException` | `read()`, `write()`, a flush, or a handshaking line on a port that is not open. |
| `IOException` | The operating system refused: `open()` (no such device, no permission), `pselect()`, `ioctl()`, `close()`. `getErrorNumber()` gives the `errno`. |
| `SerialException` | `open()` on a port already open; a device that says it is ready and then gives or takes no byte, which is how an unplugged device shows; a handshaking line that cannot be read or set. |
| `std::invalid_argument` | `open()` with an empty port name; a setting that is not one of the values of its enum. |

The `\throw` lines of the header are not complete: `read()` lists two exceptions, and can also throw `IOException`.

**An unplugged device** makes the next `read()` or `write()` throw; the port still reads as open, and the caller closes it. The library never reopens a port.

**A baud rate outside the standard list**, e.g. 250000, is set as a custom divisor with the `TIOCSSERIAL` ioctl. A device driver that does not have this ioctl makes `open()` throw an `IOException`.

**The library takes no lock on the device.** `open()` does not ask for exclusive access, so a second program can open the same port, and each then reads bytes meant for the other.

### Threads

The library starts no thread and calls nothing back: every call runs in its caller's thread, and blocks it for at most the timeout, `flush()` apart.

`SerialImpl` has two mutexes, one for reading and one for writing:

| Calls | Lock |
|---|---|
| `read()`, `readline()`, `readlines()`, `flushInput()` | read |
| `write()`, `flushOutput()` | write |
| `setPort()`, `flush()` | both |
| everything else: `open()`, `close()`, `isOpen()`, `available()`, `waitReadable()`, `setTimeout()`, the settings and the handshaking lines | none |

So one thread may read while another writes, and two threads that read get whole reads, one after the other. Nothing more is promised: closing a port, or changing its timeout, while another thread reads is a race. A program that shares a `Serial` between threads for more than one reader and one writer needs a lock of its own.

## A Read on Linux

`open()` opens the device non-blocking and puts it in raw mode, with `VMIN` and `VTIME` at 0, so that `::read()` never waits: every wait is a `pselect()` with a deadline. `Serial::read(buffer, size)` then goes as follows:

1. `Serial` takes the read lock and calls `SerialImpl::read()`, which throws `PortNotOpenedException` on a closed port.
2. It starts a `MillisecondTimer` with the total time of the read.
3. It reads what the kernel already holds, with one `::read()`. With a timeout of 0, that is the whole read.
4. While bytes are missing and time is left, it waits in `pselect()` for the smaller of the time left and the inter-byte timeout.
5. When the port is readable it reads again. No byte at that point means the device is gone: `SerialException`.
6. It returns the number of bytes read, which is `size` or, after a timeout, less.

A write is the same loop with `pselect()` on writability, and always tries once, even with a timeout of 0.

## The Fork

Upstream is not a remote of this repository. We compared it with a separate clone of upstream, and with the branches of the repositories it came through, on 2026-11-15:

| Repository | Branch | Tip |
|---|---|---|
| [wjwwood/serial](https://github.com/wjwwood/serial), upstream | `main` | `69e0372` |
| [RoverRobotics-forks/serial-ros2](https://github.com/RoverRobotics-forks/serial-ros2) | `master` | `ae46504` |
| [tylerjw/serial](https://github.com/tylerjw/serial) | `ros2` | `d8d1606` |
| this repository | `ros2` | `d8d1606` |

The history leaves upstream at **`683e12d`**, "Fix memory leak when exception is thrown by impl classes", of 2019-03-26: it is the last commit that is also in upstream's `main`. Ten commits follow, by Dan Rose and Scott K Logan, then by Tyler Weaver. **None is ours**: this repository holds the same branches, at the same commits, as tylerjw/serial. Its default branch, `main`, is upstream's code without the changes below; the code described here is the branch `ros2`.

| Commit | Date | What it changes |
|---|---|---|
| `7430d52` Migrated to ROS2 | 2019-04-05 | `CMakeLists.txt` and `package.xml` go from catkin, the build system of ROS 1, to `ament_cmake`, the one of ROS 2. |
| `8900d46` Add setupapi to ament_export_libraries | 2019-04-10 | Windows only: programs that link the library also link `setupapi`. |
| `5174182` Fix the build and cleanup CMakeLists | 2019-04-22 | One `add_library()` with the sources of the platform added by `target_sources()`. It comments out the example program and the tests. |
| `f6bd881` Convert package to ament_cmake | 2019-07-03 | A second migration, on a branch: it also ports `tests/CMakeLists.txt` to `ament_add_gtest()`. |
| `8e0effc` Merge branch 'ament_cmake' | 2020-09-03 | Merges it. |
| `7fe3be8`, `2df800d` | 2020-09-08 | Remove the timer test, then put it back: no change. |
| `ae46504` Revert "Merge branch 'ament_cmake'" | 2020-09-08 | Undoes the merge, and with it the port of the tests. |
| `7141160` fix library export | 2022-02-12 | The include folder becomes a public property of the target, the whole `include` folder is installed, and the target is exported: a program can link `serial::serial`. Before, the package exported only the variables of `ament_export_libraries()`. |
| `d8d1606` position independent code | 2022-02-13 | `POSITION_INDEPENDENT_CODE ON`: the static library can be linked into a shared one. It also changes one word of a comment of `serial.h`, to "sagainst", a typo. |

What is left of the ten, `git diff 683e12d d8d1606`, is three files: `CMakeLists.txt`, six lines of `package.xml`, and that one word of a comment. **No source file differs from upstream's at `683e12d`**, and `tests/CMakeLists.txt` does not either: it still calls catkin.

Upstream has gone on since. Its `main` has nine commits this fork does not have. Those that touch what we build:

| Upstream commit | What it fixes |
|---|---|
| `9e331e7` fix invalid memory access when eol size >1 | `readline()` and `readlines()` compare the end of line with bytes before the start of their buffer, while fewer bytes than the end of line have been read. |
| `57f7277` Add EINTR handling to SerialImpl::write | A `::write()` interrupted by a signal is tried again. Here it throws the `SerialException` of an unplugged device. |
| `a93fc84` custom baud rate | A custom baud rate is applied after `tcsetattr()`, not before. |
| `ed9f89c` serial bluetooth ports | `list_ports()` also lists `/dev/rfcomm*`. |
| `9fc9e81` Remove Boost dependency | The test no longer includes Boost, and `package.xml` no longer asks for it. |

## The Build

The package is an `ament_cmake` package: [`package.xml`](../package.xml) names `ament_cmake` as its build tool and its build type, and [`CMakeLists.txt`](../CMakeLists.txt) calls `find_package(ament_cmake REQUIRED)` and `ament_package()`. It is no longer a catkin package, and not plain CMake either: it configures only where `ament_cmake` is found, i.e. with ROS 2 sourced. `colcon` builds it like any package of a workspace:

```
colcon build --packages-select serial
```

- **A static library.** `add_library()` names no type and the package does not set `BUILD_SHARED_LIBS`, so the build makes `libserial.a`, compiled as position independent code.
- **What it exports.** The target `serial::serial`, which carries the include folder and the link to `rt` and `pthread`, and the older variables `serial_INCLUDE_DIRS` and `serial_LIBRARIES`. A package uses it with `find_package(serial REQUIRED)`, and `<depend>serial</depend>` in its `package.xml`.
- **No compiler options.** The package sets no C++ standard and no warning flags: it compiles with the compiler's defaults.
- **No tests and no example.** Both are commented out of `CMakeLists.txt`, see [Tests](#tests).
- **One dependency it does not need.** `package.xml` still has `<test_depend>boost</test_depend>`, for a test that is not built.

Three files of the repository still describe the catkin build, and no longer work as written: the [`Makefile`](../Makefile), whose `make test` builds the target `run_tests` of catkin, [`.travis.yml`](../.travis.yml), and the Dependencies and Install sections of the README. The folder [`visual_studio`](../visual_studio) and the Doxygen files of [`doc`](../doc) are upstream's, unchanged.

## Who Uses It

The library is the lowest layer of the serial stack of StepIt Macro, a focus stacking rig. One class uses it, `DefaultSerial` of the library [framed-serial](https://github.com/kineticsystem/framed-serial), which adds frames and a CRC on top; the drivers of StepIt Motors and StepIt Freezer talk to their microcontrollers through framed-serial. They do not name `serial::` themselves; only two hand-run hardware checks of StepIt Motors do, to catch `serial::IOException`.

`DefaultSerial` builds a `Serial` without a port, then calls `setPort()`, `setBaudrate()`, `setTimeout()` with a `simpleTimeout`, `open()`, `isOpen()` and `close()`, `read()` of one byte at a time, and `write()` followed by `flush()`. It uses nothing else: no `readline()`, no `list_ports()`, no handshaking line. It catches nothing: the exceptions of this library reach its callers as they are.

The rig builds on Linux only, amd64 and arm64, with `colcon`, from the copy its `stepit.repos` pins at `d8d1606`, with `-DBUILD_TESTING=OFF`; the two modules pin the same commit to build on their own. The static library ends inside `libframed_serial.so`, which is why it must be position independent code.

## Tests

The repository has two test programs, both upstream's and both for Unix:

| Test | Cases | What it covers |
|---|---|---|
| [`tests/unix_serial_tests.cc`](../tests/unix_serial_tests.cc) | 4 | A read, a write, a read that times out, and a read that gets fewer bytes than asked, on a pseudo-terminal made with `openpty()`: no hardware needed. |
| [`tests/unit/unix_timer_tests.cc`](../tests/unit/unix_timer_tests.cc) | 2 | `MillisecondTimer` against `usleep()`, within 1 ms on short waits and 5 ms on long ones. |

**Nothing builds or runs them.** `CMakeLists.txt` has `add_subdirectory(tests)` commented out, and [`tests/CMakeLists.txt`](../tests/CMakeLists.txt) calls `catkin_add_gtest()`, which `ament_cmake` does not have; the repository has no CI of its own besides the `.travis.yml` of the catkin build. The files of `tests/proof_of_concepts` are experiments, not tests.

The code that uses the library does not test it either: the tests of framed-serial replace the port with a mock. The library therefore runs only against a real device. To check it by hand, [`examples/serial_example.cc`](../examples/serial_example.cc) lists the ports, or writes a string to one and reads it back; it needs a device that echoes, e.g. the Arduino sketch at the top of `unix_serial_tests.cc`, and it is commented out of the build too.

## How to Extend Serial

| To... | Change... |
|---|---|
| Take a fix from upstream | Cherry-pick its commit onto the branch `ros2`: the sources are upstream's at `683e12d`, so it applies without the build files. Then move the commit pinned by every program that uses the library, together. |
| Build the tests | In `tests/CMakeLists.txt`, replace `catkin_add_gtest()` with `ament_add_gtest()` after `find_package(ament_cmake_gtest REQUIRED)`, as `f6bd881` did; remove the include of Boost from `unix_serial_tests.cc`; uncomment `add_subdirectory(tests)` in `CMakeLists.txt`. |
| Add a call to `Serial` | Declare it in `serial.h`, forward it in `serial.cc`, taking the read or the write lock if it touches the stream, and implement it in **both** `unix.cc` and `win.cc`: a platform we do not build still has to keep the same `SerialImpl`. |
| Fix the behaviour on Linux | `src/impl/unix.cc` only. `serial.h` stays free of platform types. |

## Design Decisions and Trade-offs

**A fork, not upstream and not a Debian package.** Upstream is a catkin package and has no ROS 2 build on any branch: its `ros2` branch stops at `683e12d`, with no change. The index of ROS 2 Jazzy, [`jazzy/distribution.yaml`](https://github.com/ros/rosdistro/blob/master/jazzy/distribution.yaml), has no `serial` package either, so there is no Debian package to install: the library is built from source, from a port to `ament_cmake` with an exported target. A program pins this repository at a commit, and the repository is ours: the code does not change or disappear with somebody else's. The price: the fork is frozen at upstream's code of March 2019, without the fixes listed in [The Fork](#the-fork), and nobody but us will bring them over.

**The fork changes the build only.** No line of code differs from upstream, so upstream's documentation and upstream's issues still describe what we run, and a fix from upstream applies cleanly. The price is that we also keep upstream's defects as they were.

**A private implementation per platform.** `serial.h` shows no `termios` and no Windows handle, so a caller compiles the same on every platform, and an implementation can change without recompiling its callers' interface. The price is a second class to keep in step by hand for a platform we never build.

**Timeouts are return values, failures are exceptions.** A silent device is normal on a serial line, so a read that times out returns a short count; only a port that fails throws. The price: a caller must check every count, and the exceptions have no common base besides `std::exception`.

**A static library.** The code is small and has no state outside a `Serial` object, so each program that links it can carry its own copy. It has to be position independent code to go into a shared library, which is the one change `d8d1606` makes.

**Two locks, for the stream only.** A reader and a writer do not wait for each other. The settings, `open()` and `close()` are left unguarded: the library assumes one owner per port.

**The tests are left unbuilt.** The fork's authors commented them out rather than port them, and reverted the one commit that did. We have not changed that: the library is third-party code that we pin and do not develop, and its users test their own code against a mock. The price is that a change of commit is checked only on the real devices.
