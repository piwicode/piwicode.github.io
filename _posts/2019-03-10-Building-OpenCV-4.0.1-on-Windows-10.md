---
layout: post
title: Building OpenCV 4.0.1 on Windows 10
excerpt: Detailed procedure to build OpenCv with contrib for python and java.
---

The prebuilt OpenCV releases does not contain the tracker module, which lives in
the `contrib` code repository. Here is how to build OpenCV with contributions
and with python bindings and java bindings.

## Prerequisite

- Download and install [Visual Studio 2017 Community Edition](https://visualstudio.microsoft.com/en/downloads)
- Download and install [cmake](https://cmake.org/download/)
- Download and install [Git](https://git-scm.com/download/win)
- Download and install Python 3.7.2 - [Windows x86-64 executable installer](https://www.python.org/ftp/python/3.7.2/python-3.7.2-amd64.exe). Add Python it to the path when asked.

You can verify the success calling `python --version` from a command shell and
getting back `Python 3.7.2`.

## Common steps

From a `Git bash console`, go to a freshly created working directory and run the
following command to download OpenCV sources:

```
git clone -b 4.0.1 https://github.com/opencv/opencv.git
git clone -b 4.0.1 https://github.com/opencv/opencv_contrib.git
```

## Build with python binding

Go to a freshly created working directory and create a virtual python
environment where to install the python dependencies.

```
python -m venv venv
```

This should have created a subdirectory named `venv`, that contains packages
and scripts. Now activate the virtual environment with:

```
. venv/Scripts/activate
```

Download and install `numpy` in the environment:

```
pip install numpy
```

Now `numpy` lives in `./venv/Lib/site-packages/numpy`

It is time to generate Visual Studio project for the build definition. Create
a build directory:

```
mkdir build
cd build

cmake \
  -G "Visual Studio 15 2017 Win64" \
  -DBUILD_opencv_python3=ON \
  -DINSTALL_CREATE_DISTRIB=ON \
  -DPYTHON3_NUMPY_INCLUDE_DIRS=$PWD/../venv/lib/site-packages/numpy/core/include \
  -DOPENCV_EXTRA_MODULES_PATH=../opencv_contrib/modules ../opencv
```

 - `-G "Visual Studio 15 2017 Win64"` specifies which version of visual studio project to generate.
 - `-DBUILD_opencv_python3=ON` requires to generate python module files.
 - `-DPYTHON3_NUMPY_INCLUDE_DIRS=[...]` indicates where to find NumPy headers
 - `-DOPENCV_EXTRA_MODULES_PATH=[...]` specifies to build OpenCV with the contribution repository

Finally build the binaries:

```
cmake --build . --config Release -- -maxcpucount:2
cmake --build . --config Release --target install
```

Note that Python release does not contain debug binaries, and consequently it is
not possible to build OpenCV with debug symbols. Once the build is over
`ls python_loader/cv2` should return
`__init__.py  config.py  load_config_py2.py  load_config_py3.py`.

## Build java bindings

Clear the `build` directory, and install java jdk (e.g. jdk-11.0.2) from Oracle.
Select the jdk you intent to use for you project with the following command:

```
export JAVA_HOME='C:\Program Files\Java\jdk-11.0.2
```

Generate Visual Studio project files:

```
cmake \
  -G "Visual Studio 15 2017 Win64" \
  -DOPENCV_EXTRA_MODULES_PATH=../opencv_contrib/modules \
  -DBUILD_SHARED_LIBS=OFF \
  -DBUILD_FAT_JAVA_LIB=ON \
  -DBUILD_opencv_java=ON \
  -DANT_EXECUTABLE=$PWD/../apache-ant-1.10.5/bin/ant.bat \
  -DJNI_INCLUDE_DIRS='C:\Program Files\Java\jdk-11.0.2\include' \
  ../opencv
```

Verify that `java` and `java_bindings_generator` can be found on the line
` To be built: `. Finally run the build command:

```
cmake --build . --config Release -- -maxcpucount:2
cmake --build . --config Release --target install
```

## Build C++

Even though the Java version is convenient to build a GUI thanks to Swing, and
Python is very nice to transform lists thanks to comprehension, I had to C++
to debug OpenCV.  Indeed, I wasn't able to use my GPU with OpenCL and I needed
to step into OpenCV code & profile it. Here is how to build OpenCV for C++, with
TBB enabled:

Download TBB from https://www.threadingbuildingblocks.org/download and unzip in
you working directory. It creates a subdirectory with the release name (e.g.
`tbb2019_20190206oss`)

Set the environment variable calling the script:
```
tbb2019_20190206oss\bin\tbbvars.bat
```

Now generate the projects:

```
mkdir build
cd build
cmake \
  -G "Visual Studio 15 2017 Win64" \
  -DINSTALL_CREATE_DISTRIB=ON \
  -DBUILD_SHARED_LIBS=ON \
  -DOPENCV_EXTRA_MODULES_PATH=../opencv_contrib/modules \
  -DWITH_TBB=ON \
  ../opencv
```

Finally run the build command:

```
cmake --build . --config Release -- -maxcpucount:2
cmake --build . --config Release --target install
cmake --build . --config Debug -- -maxcpucount:2
cmake --build . --config Debug --target install
```

In Visual Studio is created a solution directory next to `tbb2019_20190206oss`,
and `build`. Edit the project properties to add the following:

- `Configuration Properties`
  - `Debugging`
    - `Environment`: `PATH=$(ProjectDir)\..\..\tbb2019_20190206oss\bin\intel64\vc14;$(ProjectDir)\..\..\build\bin\Debug;%PATH%`
  - `C/C++`
    - `Additional Include Directories`: `..\..\opencv_build_release\install\include`
  - `Linker`
    - `General`
      - `Additional Library Directories`: `..\..\opencv_build_release\lib\Debug` for debug and `..\..\opencv_build_release\lib\Release` for Release
    - `Input`
      - `AdditionalDependencies` : `opencv_world401d.lib` for Debug and `opencv_world401.lib` for Release
