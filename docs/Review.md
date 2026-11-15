# Serial Review

## Table of Contents <!-- omit in toc -->

- [Scope and Method](#scope-and-method)
- [Verdict](#verdict)
- [The Calls the Rig Makes](#the-calls-the-rig-makes)
- [The Fork](#the-fork)
  - [Which Upstream Fixes Matter](#which-upstream-fixes-matter)
  - [Keeping the Fork, or Another Route](#keeping-the-fork-or-another-route)
- [The Build](#the-build)
- [Testability and Tests](#testability-and-tests)
- [Implementation Issues](#implementation-issues)
- [Suspicions the Probes Disproved](#suspicions-the-probes-disproved)
- [The Architecture Document](#the-architecture-document)
- [Recommendations](#recommendations)

Reviewed on 2026-11-15, at commit `d8d1606`, the tip of the branch `ros2`. It judges the library that [docs/ARCHITECTURE.md](ARCHITECTURE.md) describes, and does not repeat it.

## Scope and Method

The library is third-party code: [wjwwood/serial](https://github.com/wjwwood/serial) with another build. This is therefore not a design review of upstream, and the questions of structure and SOLID are not asked of it. The review answers three questions for StepIt Macro, the focus stacking rig that pins this commit in its `stepit.repos`: is the library safe under the motors and the Freezer board as the rig builds and calls it, is keeping the fork right, and what would a minimal test cover.

We read, whole, [`src/serial.cc`](../src/serial.cc), [`src/impl/unix.cc`](../src/impl/unix.cc), [`include/serial/serial.h`](../include/serial/serial.h), [`include/serial/impl/unix.h`](../include/serial/impl/unix.h), [`CMakeLists.txt`](../CMakeLists.txt), [`package.xml`](../package.xml) and the two test programs of [`tests`](../tests). We did not review `win.cc`, the three `list_ports` files, or the calls the rig never makes, beyond a line. The neighbours we read are the only caller, `DefaultSerial` and `DefaultFramedSerial` of [framed-serial](https://github.com/kineticsystem/framed-serial) at `d386c91`, and where the drivers of StepIt Motors and StepIt Freezer configure the port and catch its exceptions. For `flush()` we also read the kernel's side of `tcdrain()` in Linux 6.8: `tty_wait_until_sent()` of `drivers/tty/tty_ioctl.c`, `acm_tty_chars_in_buffer()` of `drivers/usb/class/cdc-acm.c`, and `drivers/tty/pty.c`.

Each suspicion that matters to the rig was put to a throwaway GoogleTest probe, 15 in all, compiled with `serial.cc` and `unix.cc` in a scratch folder inside the rig's image `stepit-macro:latest` (GCC 13.3, the host's kernel 7.0), then deleted. Each probe opens the port as `DefaultSerial` does (`setPort()`, `setBaudrate(9600)`, a `simpleTimeout` of 200 ms, the timeout of the motors' driver, `open()`) on a pseudo-terminal made with `openpty()`, and plays the device on the other end. A finding marked **verified** failed its probe; the others are **from reading**. No real device was opened: what a pseudo-terminal cannot show is said with each finding.

The fork was compared with a scratch clone of upstream at `69e0372`, and upstream's fixes were tried on a scratch clone of this repository.

The whole suite: the repository builds none of its tests, so there is nothing to run as it stands. Compiled by hand from copies, its two programs pass: `unix_serial_tests.cc`, 4 tests in 0.5 s, and `unix_timer_tests.cc`, 2 tests in 1.9 s.

## Verdict

**The library is fit to stay under the rig, with one call that must go and two small repairs.** The path the rig uses, about 700 of the 1066 lines of `unix.cc`, does what the architecture document says: the port is in raw mode and every byte value passes unchanged, a read and a write end at the timeout to the millisecond, and an unplugged device makes the next call throw at once. Six of the 15 probes failed as the rig builds the library, and a seventh with optimisation; none of them is a hazard for the motors, the camera or the lights.

**No finding is a safety issue for the rig in the sense of a motor that moves or does not stop because of this library.** The one finding that touches the safety nets is the first: `flush()` has no timeout, and framed-serial calls it after every byte, so the 0.2 s timeout of the motors' driver does not bound a cycle. It comes from reading the library and the kernel; a pseudo-terminal cannot reproduce it. The repair is in framed-serial, not here.

**Keeping the fork is right for now, but not frozen.** There is no Debian package of this library for ROS 2 Jazzy, upstream has no `ament_cmake` build, and the fork's build is sound. Two of upstream's nine later commits matter on Linux, both apply cleanly, and both are already in this repository, on its branch `main`. Upstream has made no change to the code since June 2021, so the fixes below that upstream does not have (findings 2 and 4) will only ever come from us: the rule "the fork changes the build only" has to give way for them, or the port moves into framed-serial as a class of our own.

## The Calls the Rig Makes

`grep -rn "serial_->\|serial::" ../framed-serial/src ../framed-serial/include` finds the constructor and 12 methods of `serial::Serial`, all called from `default_serial.cpp`. Each was judged on Linux:

| Call | Judgement | Evidence |
|---|---|---|
| `Serial()` without a port, `setPort()`, `getPort()`, `getBaudrate()`, `getTimeout()` | Sound | No system call on a closed port. |
| `setBaudrate()`, `setTimeout()` before `open()` | Sound | Stored, then applied by `reconfigurePort()`. Without `setTimeout()` a read does not wait at all: 0 ms in the probe. Every caller of the rig sets it. |
| `open()` | Sound when it succeeds, leaks when it fails late | Finding 2. Raw mode is complete: the probe sent the 256 byte values each way through a pseudo-terminal, which starts in cooked mode, and got them back unchanged. A missing device throws `IOException` with `errno` 2 and leaks nothing. |
| `isOpen()` | Sound, with one trap | It stays true after the device is unplugged, as the architecture document says, and after a `close()` that throws (finding 4). |
| `read(buffer, 1)` | Sound | 0 bytes after 199 ms on a silent port; `SerialException` after 0 ms once the other end is gone. A read of one byte never takes the `waitByteTimes()` path. |
| `write(buffer, 1)` | Sound, apart from a signal | 0 bytes after 199 ms on a port that takes no more; `SerialException` after 0 ms once the other end is gone. Finding 3. |
| `flush()` | **Unbounded** | Finding 1. |
| `close()` | Sound in practice | Finding 4 is what happens if `::close()` ever fails. |

The callers do their part: every `open()`, `read()` and `write()` of the two drivers is reached under a `catch (const std::exception&)`, which is the only way to catch the library's three exception classes together.

## The Fork

### Which Upstream Fixes Matter

Upstream's `main` is nine commits ahead of the fork point `683e12d`. We tried the five that touch what Linux builds on a scratch clone of `d8d1606`:

| Upstream commit | Matters to the rig | Applies to `ros2` |
|---|---|---|
| `57f7277` Add EINTR handling to `SerialImpl::write` | **Yes**: it is finding 3. With upstream's `unix.cc` the same probe wrote 300 000 bytes under 126 854 signals with no exception. | Cleanly |
| `9e331e7` fix invalid memory access when eol size >1 | Not today: the rig never calls `readline()` or `readlines()`. Worth taking with the other, so that the trap is gone before somebody does. | Cleanly |
| `a93fc84` custom baud rate after `tcsetattr()` | No: both boards run at 9600, a standard rate, and a USB serial device ignores the rate anyway. | Cleanly |
| `ed9f89c` Bluetooth ports in `list_ports()` | No: the rig never calls `list_ports()`. | Cleanly |
| `9fc9e81` Remove Boost dependency | For the build only, see [The Build](#the-build). | **Conflict** in `package.xml` |

The other four change Windows, the catkin install, the README and the licence file.

The two that matter need no upstream remote: this repository's own branch `main` is at `33e5a31`, upstream's `main` less its last commit, and holds them. A merge of it into `ros2` conflicts in `CMakeLists.txt` and `package.xml`; cherry-picking the two commits does not.

### Keeping the Fork, or Another Route

| Route | What it gives | What it costs |
|---|---|---|
| The fork as it is | Nothing to do. | Findings 2, 3 and 4 stay, with no test to catch a regression. |
| The fork, with `57f7277` and `9e331e7` cherry-picked and our own repair of `open()` | Findings 2 and 3 closed for fewer than 30 changed lines. | The first source line that differs from upstream: the design decision "the fork changes the build only" ends, and with it the promise that upstream's issues describe what we run. |
| Our own port class in framed-serial | `DefaultSerial` is the only caller and already hides the library behind `framed_serial::Serial`. A class of about 150 lines on `termios` and `poll()` would replace those 13 calls, and drop a repository, the static library and the code the rig never runs (Windows, macOS, `list_ports()`, the handshaking lines), and close findings 2 to 8 by construction. | Terminal code of our own to keep right. It needs the pseudo-terminal tests of [Testability and Tests](#testability-and-tests) from the first day. |
| A packaged library | `libserial-dev` 1.0.0 (Ubuntu's universe) and `ros-jazzy-serial-driver` 1.2.0 are both candidates in the image's package index. | Not evaluated here. Another interface and other timeouts: `serial-driver` is built on asio and its own threads. |

Our judgement: take the second route now, because it is one small pull request, and the third when the fork needs a second patch of our own. A fork that we must patch and that upstream no longer moves is our code in all but name; 150 lines that we test are easier to answer for than the 1831 that Linux builds and nothing tests.

## The Build

| Question | Verdict | Evidence |
|---|---|---|
| The export | Sound | `export_serialExport.cmake` defines `serial::serial` as a static imported target with `INTERFACE_INCLUDE_DIRECTORIES` and `INTERFACE_LINK_LIBRARIES "rt;pthread"`; framed-serial finds it with `find_package(serial REQUIRED)`. |
| A static library with `-fPIC` | Sound | `nm` finds the library's symbols in `libframed_serial.so` and in no other library of the rig's modules: one copy per process, so the "one owner per port" state cannot be doubled. |
| The flags | A weakness of the rig's build, not of the fork | The rig builds every module with `-DCMAKE_BUILD_TYPE=Debug`, and the package adds nothing: the library is compiled with `-g -fPIC`, no optimisation, no standard, no warning. Without optimisation GCC also leaves `_FORTIFY_SOURCE` off, which is the difference between an abort and silent corruption in finding 8. With the warnings of the rig's own packages, `unix.cc` gives 17 and `list_ports_linux.cc` 3. |
| The tests commented out | The real gap | See [Testability and Tests](#testability-and-tests). |
| `<test_depend>boost</test_depend>` | Stale | For one unused `#include <boost/bind.hpp>` of a test that is not built. `rosdep` resolves it to `libboost-all-dev` for anybody who builds the package with it; in the rig's image the camera's `web_video_server` needs that package anyway. |
| `package.xml` | Stale | Format 1, upstream's maintainer and upstream's URLs: a bug report goes to a project that does not have this build. |
| `cmake_minimum_required(VERSION 3.5)` | Works today | CMake 3.28 of Ubuntu 24.04 accepts it; newer releases warn below 3.10. |
| The catkin leftovers | Stale | `Makefile`, `.travis.yml`, `tests/CMakeLists.txt` and two sections of the README describe a build that no longer exists, as the architecture document says. |

## Testability and Tests

Nothing runs this code before it meets a real board: the repository builds no test, framed-serial mocks the port, and the two drivers mock framed-serial. A change of the pinned commit is checked by the motors moving.

The library is easy to test, and upstream showed how: a pseudo-terminal needs no hardware and no privilege. Our 15 probes ran in about 5 s. A minimal test program, built with `ament_add_gtest()`, would hold these, each on `openpty()`:

| Test | Today |
|---|---|
| Every byte value passes unchanged, each way | Passes |
| A read on a silent port returns 0 at the timeout, not before and not 50 ms after | Passes |
| A write to a port that takes no more returns a short count at the timeout | Passes |
| The other end closed: `read()` and `write()` throw at once, `close()` succeeds, `isOpen()` turns false | Passes |
| `open()` of a missing device throws and leaves no descriptor | Passes |
| `open()` of a path that is not a terminal throws and leaves no descriptor | **Fails**, finding 2 |
| `write()` under a timer signal whose handler has no `SA_RESTART` never throws | **Fails**, finding 3 |
| `open()` on an open port throws `SerialException` and the port stays open | Passes |

What a pseudo-terminal cannot cover, and stays a check by hand on the boards: `tcdrain()`, which returns at once on a pseudo-terminal because `pty.c` reports no pending output; the baud rate and the custom divisor; the handshaking lines; and a USB device pulled out, which the kernel reports as a hang-up and not, as here, as an input/output error. The library treats both the same way, from reading.

## Implementation Issues

Ordered by what they cost the rig. None risks a picture already taken; none is a security issue.

1. **`flush()` waits without a limit, and the rig calls it after every byte** (from reading; `SerialImpl::flush()`, line 797 of `unix.cc`, and `DefaultSerial::write()` of framed-serial). `tcdrain()` is `tty_wait_until_sent(tty, 0)`, and the kernel turns that 0 into "for ever"; only a signal or the removal of the device ends it. For a USB CDC device (`/dev/ttyACM0`, the Teensy of the motors, and a Nano R4) the kernel counts output as pending while any write buffer is still on its way, so a board whose USB answers while its program no longer reads holds `flush()` until it is unplugged. The driver's 0.2 s timeout does not apply: it bounds `write()` and `read()`, and both were measured at 199 ms. `Serial::flush()` also holds the read and the write lock while it waits. **For the rig**: the realtime loop of the motors would stop in `write()` without an exception, so `StepitHardware` never reports "Lost the StepIt controller" and `/motors/status` keeps its last value. It is not a motion hazard of its own, since a board that does not read cannot be told to stop either way, but it hides the one case the timeout exists for. A USB-to-UART bridge (`/dev/ttyUSB0`, a classic Nano) is not exposed: with no flow control it always drains. The probe could not show any of this: on a pseudo-terminal that takes no more bytes, `flush()` returned after 0 ms. The return value of `tcdrain()` is not checked either, so a flush cut short by a signal looks complete. StepIt Motors' architecture document already gives the price of the working case: a cycle of 1.6 ms, against 0.4 ms with one write per frame.

2. **A failed `open()` leaks its file descriptor** (verified; `SerialImpl::open()`, lines 139 to 156). When the device opens and `reconfigurePort()` then throws, `fd_` keeps the descriptor, `is_open_` stays false, `close()` therefore does nothing, and the next `open()` overwrites `fd_`. The probe opened `/dev/null` 50 times and counted 50 more descriptors; after 969 failures the whole process could open no file at all. **For the rig**: the Freezer reconnects every 2 s, so a `usb_port` that names something that is not a serial device, or a device that fails between `::open()` and `tcgetattr()`, exhausts the node in about 32 minutes. An unplugged device does not leak, since its path is gone: the common case is safe. The error does not help either: `IO Exception: ::tcgetattr, file ..., line 170`, with no `errno`. Not a safety issue.

3. **A `::write()` interrupted by a signal is reported as an unplugged device** (verified; `SerialImpl::write()`, lines 661 to 671). Any return below 1 throws "device reports readiness to write but returned no data (device disconnected?)", `EINTR` included. Under a timer signal every 50 µs, with a handler without `SA_RESTART`, 26 to 48 of 3000 one-byte writes threw, over three runs; with upstream's `57f7277`, none of 300 000. **For the rig**: its processes get a signal when they are stopped, and `rclcpp` installs its handlers without `SA_RESTART`, so the last command of a stop can, rarely, be the one that throws; the driver then logs a lost controller that is not lost. Not a safety issue: the signal has to land within the few microseconds of one `::write()`, on the thread that writes.

4. **A `::close()` that fails throws, also from the destructor** (verified by force; `SerialImpl::close()`, lines 458 to 472, and `~SerialImpl()`). We closed the descriptor behind the library's back to make `::close()` fail: `delete` of the `Serial` ended the process with `terminate called after throwing an instance of 'serial::IOException'`, and a direct `close()` left `isOpen()` true with the stale descriptor number still held. Linux gives a descriptor back even when `::close()` fails, so a second `close()` could close a file that another thread has just opened. **For the rig**: we found no way a terminal makes `::close()` fail, so this is a latent trap, not a present risk. Not a safety issue.

5. **The port is not exclusive** (verified; `open()`, line 139: no `TIOCEXCL`, no `flock()`). A second `Serial` opened the same port without error, and of 8 bytes sent the first read `ACEG` and the second `BDFH`. The architecture document says so. **For the rig**: a serial monitor, a firmware upload or the hand-run `get_info` of StepIt Motors, started while the rig runs, steals half of each reply; the driver sees CRC errors and timeouts. A nuisance to diagnose, not a safety issue.

6. **A write with a timeout of 0 can wait 49 days** (from reading; `write()`, lines 625 to 640). The first pass skips the check of the time left and casts it to `uint32_t` in `timespec_from_ms()`: if 1 ms has passed since the timer was built, -1 becomes 4 294 967 295 ms of `pselect()`. Not reachable in the rig, whose timeouts are 0.2 s and above.

7. **`tcsetattr()` is not checked** (from reading; `reconfigurePort()`, line 444). A port that refuses the settings opens as if it had taken them. Both boards are USB devices that ignore the line settings, so the rig does not depend on it.

8. **A descriptor of 1024 or more overruns the `fd_set`** (verified outside the rig's limits; `waitReadable()` and `write()`). With the limit raised to 4096 and the port on descriptor 2000, a build with `-O2` aborted with `bit out of range 0 - FD_SETSIZE on fd_set`; the rig's own flags, `-g`, read and wrote as if nothing had happened, over memory that is not the set's, and AddressSanitizer did not notice. Not reachable in the rig as it runs: the container's soft limit is 1024, so `open()` fails first.

9. **The locks cover one reader and one writer, nothing else** (from reading). `open()`, `close()`, `setTimeout()` and `isOpen()` take no lock, as the architecture document says. A caller that shares a port between threads serialises it itself, as the Freezer's `SynchronizedDriver` does; that is for the reviews of the drivers.

10. **Not called by the rig, one line each** (from reading). `readline()` and `readlines()` read before their buffer with an end of line longer than the bytes read (upstream `9e331e7`), and take `size` bytes of stack with `alloca()`. `open()` does not set `O_CLOEXEC`, so a child process inherits the port. `IOException::getErrorNumber()` is 0 for the three exceptions built from a text.

## Suspicions the Probes Disproved

- **An unplugged device might read as an endless timeout, or block.** No: with the other end closed, `read()` and `write()` threw `SerialException` after 0 ms, `flush()` returned, and `close()` worked.
- **Raw mode might be incomplete and change a byte**, e.g. `0x0D`, `0x11`, `0x13` or the frame's `0x7E`. No: all 256 values passed, each way.
- **The timeouts might drift.** No: 199 ms for 200 ms, for a read and for a write.
- **The fd leak might hit the common case, a board unplugged.** No: 50 opens of a missing device left no descriptor.
- **`FD_SETSIZE` might be reachable in the container.** No: its soft limit is 1024.
- **The library's own work per byte might be what slows a frame.** No: one byte written, flushed and read back over a pseudo-terminal took 18 µs, for about six system calls. What costs on a board is the wait inside `tcdrain()`.
- **`flush()` on a stalled pseudo-terminal** was meant to prove finding 1 and did not: it returned at once. That neither proves nor disproves the finding, which rests on reading the kernel.

## The Architecture Document

We found nothing untrue in [docs/ARCHITECTURE.md](ARCHITECTURE.md): the fork point, the ten commits, the nine upstream commits, the branches of the three other repositories, the line counts, the pins of the two modules and the behaviour of timeouts and exceptions all check out, most of them by a probe. Four places say less than the whole:

- **The Fork** says the default branch `main` is "upstream's code without the changes below". It is, and it is also newer than the fork point: at `33e5a31` it already holds the fixes of the table that follows.
- **How to Extend Serial** says an upstream fix applies by cherry-pick. Four of the five in its table do; `9fc9e81` conflicts in `package.xml`.
- **The Build** says the library compiles with the compiler's defaults. In the rig that means `-g` and no optimisation, because the rig asks for a Debug build.
- **Exceptions** says `getErrorNumber()` gives the `errno` of an `IOException`. It gives 0 for those thrown with a text, `::tcgetattr` among them.

## Recommendations

Ordered by value against effort; each fits one pull request.

1. **In framed-serial, write a frame with one `write()` and no `flush()`.** Closes finding 1 and takes the cycle of the motors from 1.6 ms towards the 0.4 ms already measured. A request and its reply are bounded by the timeouts; `tcdrain()` adds nothing to them. Trade-off: the change is in framed-serial and both drivers move their pin; it wants a check on both boards.
2. **On `ros2`, cherry-pick `57f7277` and `9e331e7`, and close the descriptor when `open()` fails after `::open()`.** Closes findings 2 and 3 and the first half of finding 10. Trade-off: one source change of our own, so the design decision "the fork changes the build only" is rewritten in the architecture document; the three `.repos` files move together.
3. **Build the tests.** Port `tests/CMakeLists.txt` to `ament_add_gtest()` as `f6bd881` did, drop the Boost include and `<test_depend>`, and add the rows of the table in [Testability and Tests](#testability-and-tests). Two of them fail until recommendation 2 is in, which is the order to do them in. Trade-off: the fork gains a test folder that differs from upstream's.
4. **Remove what describes the catkin build**: `Makefile`, `.travis.yml`, the two README sections, and name ourselves in `package.xml`. Trade-off: more files that differ from upstream, for no change in behaviour.
5. **When a second patch of our own is needed, move the port into framed-serial** as a class of its own on `termios` and `poll()`, with the tests of recommendation 3, and drop this repository from `stepit.repos`. Closes findings 4 to 8 by construction. Trade-off: we own the terminal code; it is the largest of the five.
