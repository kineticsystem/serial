# Serial Communication Library

### This Fork

This repository is a fork of [wjwwood/serial](https://github.com/wjwwood/serial). Its branch `ros2` builds with `ament_cmake`, for ROS 2, and changes no source file in `src`. What the fork changes, how it is built and what a caller may rely on is in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

Check the fork out from this repository, on its branch `ros2`, in the `src` folder of a colcon workspace. The default branch, `main`, is upstream's code and has no `ament_cmake` build:

```
git clone --branch ros2 https://github.com/kineticsystem/serial.git
```

The fork is an `ament_cmake` package. Build it with colcon, from the root of the workspace, with ROS 2 sourced:

```
colcon build --packages-select serial
```

The tests are not built today: [`CMakeLists.txt`](CMakeLists.txt) has them commented out, and [`tests/CMakeLists.txt`](tests/CMakeLists.txt) still calls catkin.

The rest of this README is upstream's, and describes upstream and its catkin build, not this fork: the two build badges, the sections Dependencies and Install, and the targets `make`, `make test` and `make install`. Only `make doc` does not depend on the build: it runs Doxygen on [`doc/Doxyfile`](doc/Doxyfile).

### Upstream

Build status of upstream, [wjwwood/serial](https://github.com/wjwwood/serial): [![Build Status](https://travis-ci.org/wjwwood/serial.svg?branch=master)](https://travis-ci.org/wjwwood/serial)*(Linux and OS X)* [![Build Status](https://ci.appveyor.com/api/projects/status/github/wjwwood/serial)](https://ci.appveyor.com/project/wjwwood/serial)*(Windows)*

This is a cross-platform library for interfacing with rs-232 serial like ports written in C++. It provides a modern C++ interface with a workflow designed to look and feel like PySerial, but with the speed and control provided by C++. 

This library is in use in several robotics related projects and can be built and installed to the OS like most unix libraries with make and then sudo make install, but because it is a catkin project it can also be built along side other catkin projects in a catkin workspace.

Serial is a class that provides the basic interface common to serial libraries (open, close, read, write, etc..) and requires no extra dependencies. It also provides tight control over timeouts and control over handshaking lines. 

### Documentation

Website: http://wjwwood.github.io/serial/

API Documentation: http://wjwwood.github.io/serial/doc/1.1.0/index.html

### Dependencies

Required:
* [catkin](http://www.ros.org/wiki/catkin) - cmake and Python based buildsystem
* [cmake](http://www.cmake.org) - buildsystem
* [Python](http://www.python.org) - scripting language
  * [empy](http://www.alcyone.com/pyos/empy/) - Python templating library
  * [catkin_pkg](http://pypi.python.org/pypi/catkin_pkg/) - Runtime Python library for catkin

Optional (for tests): 
* [Boost](http://www.boost.org/) - Boost C++ libraries

Optional (for documentation):
* [Doxygen](http://www.doxygen.org/) - Documentation generation tool
* [graphviz](http://www.graphviz.org/) - Graph visualization software

### Install

Get the code:

    git clone https://github.com/wjwwood/serial.git

Build:

    make

Build and run the tests:

    make test

Build the documentation:

    make doc

Install:

    make install

### License

The MIT License

Copyright (c) 2012 William Woodall, John Harrison

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

### Authors

William Woodall <wjwwood@gmail.com>
John Harrison <ash.gti@gmail.com>

### Contact

William Woodall <william@osrfoundation.org>
