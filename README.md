# REFPROP-cmake

Small repo with CMake build system for building REFPROP shared library

Why you should use this build system:

* The windows-style mixed-case symbols are exported on all platforms (for instance there is **ALWAYS** a ``SETUPdll`` symbol, which means you can write a clean cross-platform interface).  This magic is achieved with export aliases.
* You can easily point the repo at a different version of the REFPROP sources, allowing for building/testing several versions of REFPROP in parallel

Brought to you by Ian Bell, NIST, ian.bell@nist.gov

## License

Public domain, (though REFPROP itself is not public domain)

## Pre-Requisites

* Python (See below about disabling the use of Python)
    * ``six``  (Often packaged automatically (e.g., with Anaconda), or you can pip install it at the command prompt: ``pip install six``)
    * ``numpy`` (make sure that this prints something reasonable at the command prompt: ``python -c "import numpy; print(numpy.__version__)"``).  See below about disabling the use of numpy
* A fortran compiler
* Cmake
* git

## Instructions

1. Do a recursive(!) clone of this repository (e.g. ``git clone --recursive https://github.com/usnistgov/REFPROP-cmake.git``)
2. Copy the FORTRAN directory from your REFPROP installation into the root of your checked out code (or see below about using the path directly)
3. Open a console in the root of the cloned repository
4. ``mkdir build``
5. ``cd build``
6. ``cmake .. -DCMAKE_BUILD_TYPE=Release``
7. ``cmake --build .``

Once the shared library has been build, you will need to place it somewhere that your operating system knows where to find it.  On windows, that would be on the ``PATH`` environment variable.  On OSX, that would be one of the default shared library locations (see [apple docs](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/UsingDynamicLibraries.html) ).

## OSX Notes

* If you want to force a 32-bit build (I'm looking at you Excel 2016 on Mac), you can do:

    ``cmake .. -DREFPROP_32BIT=ON``

* On OSX, it seems you need to use the ``homebrew`` version of ``gcc`` and ``gfortran``.  You can obtain homebrew versions of gcc and gfortran with ``brew install gcc`` once homebrew is installed

* On OSX, ``cmake --build .`` with homebrewed python and vanilla system python both installed fails because ``find_package(PythonInterp)``, called by ``cmake .. -DCMAKE_BUILD_TYPE=Release``, picks up the system python rather than the brewed python, as evident from an examination of ``CMakeCache.txt`` and ``build.make``.

The solution is to force cmake to use the brewed python:

```
cmake .. -DCMAKE_BUILD_TYPE=Release -DPYTHON_EXECUTABLE:FILEPATH=/usr/local/bin/python3
```

## Linux Notes

**Linux Prerequisites:**

* Wine 
    * ``mono``  
    * ``wine-gecko`` (If you intend to run the Window's gui front end, you will need gecko for certain document displays.  If you just want the FORTRAN, FLUID, and MIXTURE folders, gecko is optional.
* A fortran compiler
* Cmake
* git
* python as you likely want the header file.

**Linux Additional Instructions:**

1. Purchase RefProp and download the .exe file.
2. Set up wine to force 32-bit mode.
Add: ``"export WINEARCH=win32"``to .bashrc (after line 10).  
   See also: [How to force wine into acting like 32-bit windows](https://askubuntu.com/questions/136714/how-to-force-wine-into-acting-like-32-bit-windows-on-64-bit-ubuntu)   
3. Install the NISTxxxx.exe using wine.
``"wine NISTxxxx.exe"`` (currently NIST2310.exe).
4. Install REFPROP-cmake per instructions using git clone.
5. Find the FORTRAN folder in your REFPROP wine install.  Usually located in ~/.wine/drive-c/Program fFiles(x64)/REFPROP/.  Note that .wine is a hidden directory and you must enable hidden directories for graphical file managers to find it. Also this is the location of the MIXTURES and FLUIDS directories.
6. Copy directory FORTRAN over to your REFPROP-cmake root directory.  
7. Follow above directions.  Tip: pay attention to the "." and ".." in the cmake commands.

## General Notes

* Platforms other than windows (and sort of OSX) are CASE-SENSITIVE!  The folder ``fortran`` is not the same as ``FORTRAN``

* If you don't want to copy the FORTRAN directory to the root of the checked out code, you can alternatively pass the cmake flag ``REFPROP_FORTRAN_PATH`` as in something like:
    
    ``cmake .. -DREFPROP_FORTRAN_PATH=/path/to/refprop/fortran``

  If the path has spaces in it, you need to quote-escape the path

* If you do not want to, or cannot, get a numpy version (especially on Red Hat based linux distros) that will allow you to do this at the command line:

    ``python -m numpy.f2py -c "import numpy"``

  then you can disable all use of numpy by passing the command line flag ``REFPROP_NO_NUMPY``

    ``cmake .. -DREFPROP_NO_NUMPY=ON``

  which will result in the C/C++ header not being generated.  It is highly recommended to find a working numpy version.

  If you absolutely cannot get access to Python, you can also define ``REFPROP_NO_PYTHON``, which will disable all use of Python, but this will disable the alias generation, so the only symbols that will be exported will be the default symbols for your compiler, which could be a problem for non-Intel compilers.  

* On windows, if you want to use Intel Fortran + Visual Studio, you can change the generator, with something like:

    ``cmake .. -G "Visual Studio 11 2012 Win64"``

  Run ``cmake --help`` to see a complete list of supported generators (not all of which will support FORTRAN)

  When building with a Visual Studio generator, you will want to ensure that you get a Release build, which is ensured by passing the flag ``--config Release`` to the build command, something like:

    ``cmake --build . --config Release``
* If you want to make the Intel runtime dynamically linked into the shared library (this is necessary in order to load hundreds of copies of REFPROP in memory with the REFPROP-manager (see https://github.com/usnistgov/REFPROP-manager)), define the CMake flag ``-DREFPROP_DYNAMIC_RUNTIME=ON``.  The default is to statically link the runtime, which is the right answers for most users and use cases.

## Instructions for MINGW builds on windows

It is possible to use a fully open-source build system on windows to compile REFPROP.  This is enabled by the use of the MINGW compiler system.

To get started from a clean windows installation, you will need:
* [cmake](https://cmake.org/download/): When you install, it is recommended to add the install directory to the ``PATH`` system variable
* [MINGW](https://sourceforge.net/projects/mingw-w64/files/latest/download): You may want to run the installer twice, the first time selecting the ``i686`` architecture (for 32-bit compilation), and the second time, selecting the ``x86_64`` architecture (for 64-bit compilation)
* [miniconda](https://conda.io/miniconda.html):  This installs a minimal python setup, along with with the ``conda`` package manager (use the 64-bit python 3.6 one).  You probably want to add conda and python to the system PATH variable when asked in the installer. Once it is installed, install numpy with : ``conda install numpy`` at the command line.  If you require administrative rights to install to the default Anaconda installation location, open an administrative shell by typing ``cmd`` in the windows start menu search, right-clicking on cmd.exe, and selecting "Run as Administrator"
* [git](https://git-scm.com/download/win)

Then to set up your shell, at the command prompt do:
```
set PATH=D:\Software\mingw\bin;%PATH%
```
or put the path to wherever the MINGW compiler has been installed.

Check out the git sources with:
```
git clone --recursive https://github.com/usnistgov/REFPROP-cmake
```
Move into that directory:
```
cd REFPROP-cmake
```
Make a working directory
```
mkdir build
```
Move into that directory
```
cd build
```
Configure the build system
```
cmake .. -DREFPROP_FORTRAN_PATH=R:/FORTRAN -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release
```
and build the DLL
```
cmake --build .
```
That's it!

Or, all in a tidy batch file that clones the repo, does the build, and cleans up after itself... 

You will want to save these contents in a ``.bat`` file and run it from the command prompt, passing it ``32`` for a 32-bit build generating REFPROP.DLL, or ``64`` for a 64-bit build, generating the ``REFPRP64.DLL`` file.

Here is the contents of ``build_dll.bat``
```
@echo off

REM Call this script like: build_dll.bat 32
REM for a 32-bit build, or build_dll.bat 64 for a 64-bit build

REM --- THESE ARE THE PATHS YOU MAY NEED MODIFY ---
set PATH_32BIT=D:\Software\mingw-w64\i686-7.2.0-posix-dwarf-rt_v5-rev0\mingw32\bin
set PATH_64BIT=D:\Software\mingw-w64\x86_64-7.2.0-posix-seh-rt_v5-rev0\mingw64\bin
set FORTRAN_PATH=R:/FORTRAN

REM --------- You should not need to touch anything below this line ------------

@setlocal enabledelayedexpansion

if "%1" == "64" (
    set MINGW_PATH=%PATH_64BIT%
    set BITNESS=64
) 
if "%1" == "32" (
    set MINGW_PATH=%PATH_32BIT%
    set BITNESS=32
) 
if "%MINGW_PATH%" == "" (
    echo An invalid bitness was selected, valid values are "64" or "32"
    pause
    exit /b
)
if not exist "%MINGW_PATH%" (
    echo The path to MINGW bin folder is invalid
    pause
    exit /b
)
set "PATH=%MINGW_PATH%;%PATH%"

git clone --recursive https://github.com/usnistgov/REFPROP-cmake
cd REFPROP-cmake
mkdir build
cd build
cmake .. -DREFPROP_FORTRAN_PATH=%FORTRAN_PATH% -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release
cmake --build .
cd ../..
if "%BITNESS%" == "32" (
    copy REFPROP-cmake\build\REFPROP.DLL REFPROP.DLL
)
if "%BITNESS%" == "64" (
    copy REFPROP-cmake\build\REFPRP64.DLL REFPRP64.DLL
)
rmdir /Q /S REFPROP-cmake
```
