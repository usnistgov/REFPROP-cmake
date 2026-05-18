# REFPROP-cmake

Small repo with CMake build system for building REFPROP shared library for REFPROP versions 9.1 and newer

Why you should use this build system:

* The windows-style mixed-case symbols are exported on all platforms (for instance there is **ALWAYS** a ``SETUPdll`` symbol, which means you can write a clean cross-platform interface).  This magic is achieved with export aliases.
* You can easily point the repo at a different version of the REFPROP sources, allowing for building/testing several versions of REFPROP in parallel

## Getting help

Open an issue: https://github.com/usnistgov/REFPROP-cmake/issues/new

## License

Public domain, (though REFPROP itself is not public domain)

## Pre-Requisites

* Python (See below about disabling the use of Python)
    * ``six``  (Often packaged automatically (e.g., with Anaconda), or you can pip install it at the command prompt: ``pip install six``)
    * ``numpy`` (make sure that this prints something reasonable at the command prompt: ``python -c "import numpy; print(numpy.__version__)"``).  See below about disabling the use of numpy
* CMake
* VS Build Tools (on Windows)
* A Fortran compiler
    * Intel oneAPI (ifx) (just the compiler is fine)
    * GNU Fortran (GFortran)
        * Follow the MINGW build instructions below
* Git

## Instructions

1. Do a recursive(!) clone of this repository, e.g.,  
   ``git clone --recursive https://github.com/usnistgov/REFPROP-cmake.git``
2. Copy the FORTRAN directory from your REFPROP installation into the root of your checked out code (or see below about using the path directly)
3. Add the path to your Fortran compiler to your system PATH variable. On Windows, add to the Path environment variable, which is usually:
    * Intel oneAPI: ``C:\Program Files (x86)\Intel\oneAPI\compiler\latest\bin``
4. Open a console in the root of the cloned repository. On Windows, open as admin the "Intel oneAPI command prompt for Intel 64 and Visual Studio 2022"
5. Make a working directory:   
  ``mkdir build``
6. Move into that directory:   
  ``cd build``
7. Configure the build system:
   1. Linux:  
   ``cmake .. -DCMAKE_BUILD_TYPE=Release``  

   2. Windows:  
      (substitute the path to your ifx.exe):  
   ``cmake .. -G "Visual Studio 17 2022" -DCMAKE_BUILD_TYPE=Release -DCMAKE_Fortran_COMPILER="C:/Program Files (x86)/Intel/oneAPI/compiler/latest/bin/ifx.exe"``
8. Build the DLL:   
  ``cmake --build . --config Release``

Once the shared library has been build, you will need to place it somewhere that your operating system knows where to find it.  On Windows, that would be on the ``PATH`` environment variable.  On OSX, that would be one of the default shared library locations (see [apple docs](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/UsingDynamicLibraries.html) ).

## OSX/MacOS Notes

* If you have an M1(+) chip, check whether your gcc/gfortran is set to use `x86_64` or `arm64`. Two options have been tested and are summarized here.

1. Use the gfortran from homebrew and build for x86_64 and use Rosetta2 emulation. The flags you want are something like:

    ``cmake .. -DCMAKE_FORTRAN_COMPILER=/path/to/gfortran -DREFPROP_X8664=ON``

    * If you want to force a 32-bit build (I'm looking at you Excel 2016 on Mac), you can do:

    ``cmake .. -DREFPROP_32BIT=ON``

    * You can obtain homebrew versions of gcc and gfortran with ``brew install gcc`` once homebrew is installed

    * On OSX, ``cmake --build .`` with homebrewed python and vanilla system python both installed fails because ``find_package(PythonInterp)``, called by ``cmake .. -DCMAKE_BUILD_TYPE=Release``, picks up the system python rather than the brewed python, as evident from an examination of ``CMakeCache.txt`` and ``build.make``.

    The solution is to force cmake to use the brewed python:

    ```
    cmake .. -DCMAKE_BUILD_TYPE=Release -DPYTHON_EXECUTABLE:FILEPATH=/usr/local/bin/python3
    ```

2. Use gcc/gfortran from Macports. Install Macports gcc and select it by `sudo port install gccXX` and `sudo port select gcc mp-gccXX` where XX is the version of gcc, e.g., `12`. Then you can use the same non-Mac-specific install instructions as described above in the "Instructions" section.

* If you want statically-linked system libraries when compiling on OSX, to improve, but not guarantee, that a binary built on one machine will run on another, you can define:

  ```
  cmake .. -DREFPROP_OSX_STATIC_LINK=ON
  ```

## General Notes

* Platforms other than Windows (and sort of OSX) are CASE-SENSITIVE!  The folder ``fortran`` is not the same as ``FORTRAN``

* If you don't want to copy the FORTRAN directory to the root of the checked out code, you can alternatively pass the cmake flag ``REFPROP_FORTRAN_PATH`` as in something like:
    
    ``cmake .. -DREFPROP_FORTRAN_PATH=/path/to/refprop/fortran``

  If the path has spaces in it, you need to quote-escape the path

* If you do not want to, or cannot, get a numpy version (especially on Red Hat based Linux distros) that will allow you to do this at the command line:

    ``python -m numpy.f2py -c "import numpy"``

  then you can disable all use of numpy by passing the command line flag ``REFPROP_NO_NUMPY``

    ``cmake .. -DREFPROP_NO_NUMPY=ON``

  which will result in the C/C++ header not being generated.  It is highly recommended to find a working numpy version.

 * If you absolutely cannot get access to Python, you can also define ``REFPROP_NO_PYTHON``, which will disable all use of Python, but this will disable the alias generation, so the only symbols that will be exported will be the default symbols for your compiler, which could be a problem for non-Intel compilers.  

* If you want to make the Intel runtime dynamically linked into the shared library (this is necessary in order to load hundreds of copies of REFPROP in memory with the REFPROP-manager (see https://github.com/usnistgov/REFPROP-manager)), define the CMake flag ``-DREFPROP_DYNAMIC_RUNTIME=ON``.  The default is to statically link the runtime, which is the right answers for most users and use cases.


## Instructions for MINGW builds on Windows

It is possible to use a fully open-source build system on windows to compile REFPROP.  This is enabled by the use of the MINGW compiler system.

To get started from a clean windows installation, you will need:
* [CMake](https://cmake.org/download/): When you install, it is recommended to add the install directory to the ``PATH`` system variable
* [MINGW](https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win32/Personal%20Builds/mingw-builds/installer/mingw-w64-install.exe): You may want to run the installer twice, the first time selecting the ``i686`` architecture (for 32-bit compilation), and the second time, selecting the ``x86_64`` architecture (for 64-bit compilation).

  After installation:
    * Add the installation directory to your system PATH variable  
      (e.g. ``C:\msys64\mingw64\bin`` for the 64-bit installation)
    * Open the MSYS2 MINGW64 terminal and install the following packages via:
      ```
      pacman -S mingw-w64-x86_64-cmake
      pacman -S mingw-w64-x86_64-make
      pacman -S mingw-w64-x86_64-gcc
      pacman -S mingw-w64-x86_64-gcc-fortran
      ```
* [Miniconda](https://conda.io/miniconda.html):  This installs a minimal python setup, along with with the ``conda`` package manager (use the 64-bit python 3.6 one).  You probably want to add conda and python to the system PATH variable when asked in the installer. Once it is installed, install numpy with : ``conda install numpy`` at the command line.  If you require administrative rights to install to the default Anaconda installation location, open an administrative shell by typing ``cmd`` in the windows start menu search, right-clicking on cmd.exe, and selecting "Run as Administrator"
* [Git](https://git-scm.com/download/win)

1. Open the MSYS2 MINGW64 terminal

2. Check out the git sources with:
    ```
    git clone --recursive https://github.com/usnistgov/REFPROP-cmake
    ```
3. Move into that directory:
    ```
    cd REFPROP-cmake
    ```
4. Make a working directory:
    ```
    mkdir build_gfortran
    ```
5. Move into that directory:
    ```
    cd build_gfortran
    ```
6. Configure the build system (substituting the actual paths):
    ```
    cmake .. -G "MinGW Makefiles" \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_C_COMPILER="C:/msys64/mingw64/bin/gcc.exe" \
      -DCMAKE_CXX_COMPILER="C:/msys64/mingw64/bin/g++.exe" \
      -DCMAKE_Fortran_COMPILER="C:/msys64/mingw64/bin/gfortran.exe" \
      -DCMAKE_MAKE_PROGRAM="C:/msys64/mingw64/bin/mingw32-make.exe" \
      -DREFPROP_MINGW_STATIC_LINK=ON \
      -DREFPROP_ADDITIONAL_LINK_FLAGS="C:/msys64/mingw64/lib/libquadmath.a -Wl,--whole-archive C:/msys64/mingw64/lib/libwinpthread.a -Wl,--no-whole-archive"
    ```
7. Build the DLL:
    ```
    cmake --build . --config Release
    ```
8. Verify there are no dependencies besides KERNEL32.dll and msvcrt.dll by running:
    ```
    objdump -p REFPRP64.DLL | grep "DLL Name"
    ```