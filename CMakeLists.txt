cmake_minimum_required (VERSION 3.5)
project (REFPROP)
enable_language(Fortran)

if (NOT REFPROP_NO_PYTHON)
  find_package (PythonInterp)
endif()

#######################################
#           BUILD OPTIONS             #
#-------------------------------------#
# These options are available to be   #
# modified in the build process.      #
# packages may want to modify these   #
# to suit, or just leave as defaults. #
#######################################

if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/fortran")
    set (FORTRAN_PATH "fortran")
else()
    set (FORTRAN_PATH "FORTRAN")
endif()

SET (REFPROP_FORTRAN_PATH
     "${CMAKE_CURRENT_SOURCE_DIR}/${FORTRAN_PATH}"
     CACHE STRING
     "The path to the directory of the FORTRAN source files")

SET (REFPROP_GCOV
     FALSE
     CACHE BOOL
     "Add the flags to support gcov code coverage")

SET (REFPROP_ADDITIONAL_LINK_FLAGS
  ""
  CACHE STRING
  "Additional link flags for linking the library")

SET (REFPROP_SOURCE_AUTOFIX
  TRUE
  CACHE BOOL
  "Automatically fix REFPROP 9.1.0 case sensitivity filenames")

SET (REFPROP_NO_NUMPY
  FALSE
  CACHE BOOL
  "Disable use of numpy to generate C/C++ header")

SET (REFPROP_NO_PYTHON
  FALSE
  CACHE BOOL
  "Disable all use of Python")

SET (REFPROP_OSX_STATIC_LINK
  FALSE
  CACHE BOOL
  "Statically link the standard libraries for OSX")

SET (REFPROP_MINGW_STATIC_LINK
  FALSE
  CACHE BOOL
  "Statically link the standard libraries for MINGW builds")


get_filename_component(REFPROP_FORTRAN_ABS_PATH "${REFPROP_FORTRAN_PATH}"
                       REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")

# Determine the name of the shared library we are going to produce
# On windows, the name depends on the bitness, on linux/OSX it is always refprop
if (UNIX)
  set(NAME "refprop")
else ()
  if (CMAKE_SIZEOF_VOID_P MATCHES "8")
    SET(NAME "REFPRP64")
  else()
    SET(NAME "REFPROP")
  endif()
  # Handle the bitness forcing for gfortran
  if ("${ARCH_FLAG}" MATCHES "-m32")
    SET(NAME "REFPROP")
  endif()
  if ("${ARCH_FLAG}" MATCHES "-m64")
    SET(NAME "REFPRP64")
  endif()
endif()

# make sure that the default is a RELEASE
if (NOT CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE RELEASE CACHE STRING
      "Choose the type of build, options are: None Debug Release."
      FORCE)
endif ()
 
# FFLAGS depend on the compiler
get_filename_component (Fortran_COMPILER_NAME ${CMAKE_Fortran_COMPILER} NAME)

# By default, no symbol aliases are used
set (ALIASES_STRING "")
# By default, no additional link flags
set (REFPROP_LINK_FLAGS "")
# By default, no DEF file is generated/used
set (DEF_FILE "")

if (Fortran_COMPILER_NAME MATCHES "gfortran.*")

  set(ARCH_FLAG "")
  if (REFPROP_32BIT)
    message(STATUS "Forcing 32-bit build and passing -m32 to compiler...")
    set(ARCH_FLAG "-m32")
  else()
    if (REFPROP_64BIT)
      message(STATUS "Forcing 64-bit build and passing -m64 to compiler...")
      set(ARCH_FLAG "-m64")
    else()
      if (REFPROP_X8664)
      message(STATUS "Forcing x86_64 build and passing -march=x86-64 to compiler...")
      set(ARCH_FLAG "-march=x86-64")
      endif()
    endif()
  endif()

  # gfortran
  set (CMAKE_Fortran_FLAGS_RELEASE " -Wuninitialized -Wall -funroll-all-loops -fno-f2c -O3 ${ARCH_FLAG}")
  set (CMAKE_Fortran_FLAGS_DEBUG   "-fno-f2c -Wuninitialized -Wall -O0 -g ${ARCH_FLAG}")

  # if(MINGW)
  #   set(CMAKE_Fortran_FLAGS_RELEASE "${CMAKE_Fortran_FLAGS_RELEASE} -mrtd")
  #   set(CMAKE_Fortran_FLAGS_DEBUG "${CMAKE_Fortran_FLAGS_DEBUG} -mrtd")
  # endif()

  if (NOT REFPROP_NO_PYTHON)
    if (NOT EXISTS ${PYTHON_EXECUTABLE})
      message(FATAL_ERROR "Specified Python executable does not exist: ${PYTHON_EXECUTABLE}")
    endif()

    # Now we need to determine whether gfortran uses --defsym or -alias to
    # define symbol aliases for the linker
    execute_process(COMMAND gfortran -Wl,--defsym
                    RESULT_VARIABLE DEFSYM_CHECK_RESULT
                    OUTPUT_VARIABLE DEFSYM_CHECK_OUTPUT
                    ERROR_VARIABLE DEFSYM_CHECK_OUTPUT
                    )
    # If there is a note in the output about defsym syntax error,
    # defsym is the alias keyword, but it didn't understand the argument.
    # That's good, it means we should use --defsym to define aliases
    string(FIND "${DEFSYM_CHECK_OUTPUT}" "syntax error" DEFSYM_STR_INDEX)
    if (${DEFSYM_STR_INDEX} GREATER 0)
      set(DEFSYM_FLAG "--using-defsym")
    endif()
    message(STATUS "DEFSYM_FLAG: ${DEFSYM_FLAG}")

    # The base command to be used to extract the aliases in a form that the compiler can consume
    set(BUILD_COMMAND ${PYTHON_EXECUTABLE} -u "${CMAKE_CURRENT_SOURCE_DIR}/generate_aliases.py" --mangling setupdll_ -O "${CMAKE_CURRENT_BINARY_DIR}/aliases_string.txt" --FORTRAN-path "${REFPROP_FORTRAN_ABS_PATH}" ${DEFSYM_FLAG})

    # If mingw, we need to add the DEF file to make sure that the mixed-case symbols are exported to the DLL
    if (MINGW)
      set(BUILD_COMMAND ${BUILD_COMMAND} --DEF-file REFPROP.DEF)
      execute_process(COMMAND ${BUILD_COMMAND}
                      RESULT_VARIABLE GEN_ALIASES_RESULT
                      OUTPUT_VARIABLE GEN_ALIASES_OUTPUT
                      ERROR_VARIABLE GEN_ALIASES_OUTPUT
                      )
      set(DEF_FILE "${CMAKE_CURRENT_BINARY_DIR}/REFPROP.DEF")
    else()
      execute_process(COMMAND ${BUILD_COMMAND}
                      RESULT_VARIABLE GEN_ALIASES_RESULT
                      OUTPUT_VARIABLE GEN_ALIASES_OUTPUT
                      ERROR_VARIABLE GEN_ALIASES_ERROR
                      )
      if(GEN_ALIASES_RESULT EQUAL "1")
        message(STATUS "Alias generation output: ${GEN_ALIASES_OUTPUT}")
        message(STATUS "Alias generation error: ${GEN_ALIASES_ERROR}")
      endif()
      file(READ "${CMAKE_CURRENT_BINARY_DIR}/aliases_string.txt" ALIASES_STRING)
    endif()

  endif()

  # Start off with the aliases
  set (REFPROP_LINK_FLAGS "${ALIASES_STRING}")

  # On OSX, we want to statically link the libquadmath library, but gfortran is VERY
  # insistent that it wants a dynamically linked libquadmath.  See also this open bug
  # for gfortran: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=46539
  # For Mac systems
  if ("${CMAKE_SYSTEM_NAME}" MATCHES "Darwin" OR MINGW)
    execute_process(COMMAND gfortran ${ARCH_FLAG} -print-file-name=libquadmath.a
                    RESULT_VARIABLE LIBQUADMATH_CHECK_RESULT
                    OUTPUT_VARIABLE LIBQUADMATH_CHECK_OUTPUT
                    ERROR_VARIABLE LIBQUADMATH_CHECK_OUTPUT
                    )
    string(STRIP ${LIBQUADMATH_CHECK_OUTPUT} LIBQUADMATH_PATH)
    message(STATUS "Path to libquadmath.a:" "${LIBQUADMATH_PATH}")
    file(COPY ${LIBQUADMATH_PATH}
         DESTINATION .)
    message(STATUS "Copied libquadmath.a here.")

    if (REFPROP_OSX_STATIC_LINK OR REFPROP_MINGW_STATIC_FLAG)

      # Now we construct the flags to be passed to linker
      set(REFPROP_LINK_FLAGS "${REFPROP_LINK_FLAGS} -static -static-libgfortran -static-libgcc ")

      # This, in concert with the libquadmath logic above, ensures that libquadmath is statically
      # linked rather than being dynamically linked.  We want a static link so that the shared
      # library can be distributed to users without gfortran
      set(REFPROP_LINK_FLAGS "${REFPROP_LINK_FLAGS} -L. ")
    endif()

    # Do not set RPATH for OSX
    SET(CMAKE_MACOSX_RPATH OFF)
  endif()

  if (REFPROP_GCOV)
    message(STATUS "Adding debug flags to enable gcov code coverage...")
    set (CMAKE_Fortran_FLAGS_DEBUG "${CMAKE_Fortran_FLAGS_DEBUG} -fprofile-arcs -ftest-coverage -O0 -pg")
    set (CMAKE_Fortran_FLAGS_RELWITHDEBINFO "${CMAKE_Fortran_FLAGS_RELWITHDEBINFO} -fprofile-arcs -ftest-coverage -O0 -pg")
  endif()

elseif (Fortran_COMPILER_NAME MATCHES "ifort.*")
  if (REFPROP_DYNAMIC_RUNTIME)
    if ("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
	  set (CMAKE_Fortran_FLAGS_RELEASE "-O3 -qopenmp -qopenmp-link=dynamic")
	  set (CMAKE_Fortran_FLAGS_DEBUG   "-g -00 -qopenmp -qopenmp-link=dynamic")
	else()
      set (CMAKE_Fortran_FLAGS_RELEASE "-f77rtl -O3 /threads")
      set (CMAKE_Fortran_FLAGS_DEBUG   "-f77rtl -debug:full -Od")
	endif()
  else()
    if ("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
	  set (CMAKE_Fortran_FLAGS_RELEASE "-O3 -static-intel -qopenmp -qopenmp-link=static")
	  set (CMAKE_Fortran_FLAGS_DEBUG   "-g -00 -static-intel -qopenmp -qopenmp-link=static")
	else()
      # Force visual studio to statically link the c runtime to avoid dependency on MSVCRXXX.dll
      set (CMAKE_Fortran_FLAGS_RELEASE "-f77rtl -O3 /threads /libs:static /MT")
      set (CMAKE_Fortran_FLAGS_DEBUG   "-f77rtl -debug:full -Od /libs:static /MT")
	endif()
  endif()
else ()
  message ("CMAKE_Fortran_COMPILER full path: " ${CMAKE_Fortran_COMPILER})
  message ("Fortran compiler: " ${Fortran_COMPILER_NAME})
  message ("No optimized Fortran compiler flags are known, we just try -O2...")
  set (CMAKE_Fortran_FLAGS_RELEASE "-O2")
  set (CMAKE_Fortran_FLAGS_DEBUG   "-O0 -g")
endif()

# Collect the source files from the specifed location
file(GLOB APP_SOURCES "${REFPROP_FORTRAN_ABS_PATH}/*.FOR")
# If the PASS_FTN.FOR is in the DLLFILES subfolder, add it too
set(PASS_FTN_PATH "${REFPROP_FORTRAN_ABS_PATH}/DLLFILES/PASS_FTN.FOR")
if (EXISTS "${PASS_FTN_PATH}")
  if (EXISTS "${REFPROP_FORTRAN_ABS_PATH}/PASS_FTN.FOR")
    message(FATAL_ERROR "PASS_FTN.FOR was found in both the root folder of sources as well as in the DLLFILES subfolder. Please delete one of them and try again.")
  endif()
  get_filename_component(PASS_FTN_ABS_PATH "${PASS_FTN_PATH}"
                       REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
  list(APPEND APP_SOURCES "${PASS_FTN_ABS_PATH}")
endif()

if(${REFPROP_SOURCE_AUTOFIX})

  # Check if any of the source files include the statement "include 'commons.for'";
  # this indicates the use of REFPROP 9.1 or earlier, which has a case-sensitivity problem
  # on non-windows systems with the includes that needs to be repaired
  set(FIX_91 OFF)
  foreach (source ${APP_SOURCES})
    FILE(READ ${source} CONTENTS)
    string(FIND "${CONTENTS}" "include 'commons.for'" found_INDEX)
    if (${found_INDEX} GREATER 0)
      set(FIX_91 ON)
      break()
    endif()
  endforeach()

  if (FIX_91 AND (NOT FIX_91_COMPLETE))

    message(STATUS "REFPROP version <= 9.1 identified by presence of include \"'commons.for'\"; copying locally and fixing include statement")

    # Copy each of the files into a temporary directory within the build directory
    foreach (source ${APP_SOURCES})
      FILE(COPY ${source} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/FORTRAN_temp")
    endforeach()

    # Collect the source files from the containing directory
    file(GLOB APP_SOURCES "${CMAKE_CURRENT_BINARY_DIR}/FORTRAN_temp/*.FOR")

    # Do some munging of the FORTRAN files for version 9.1.0 to fix case sensitivity
    # of some of the include statements
    foreach (source ${APP_SOURCES})
      FILE(READ ${source} CONTENTS)
      string(REPLACE "include 'commons.for'" "include 'COMMONS.FOR'" CONTENTS ${CONTENTS})
      string(REPLACE "include 'comtrn.for'" "include 'COMTRN.FOR'" CONTENTS ${CONTENTS})
      FILE(WRITE ${source} ${CONTENTS})
    endforeach()

    message(STATUS "Fixing of FORTRAN files complete. Compilation ready.")

    set(REFPROP_FORTRAN_PATH "${CMAKE_CURRENT_BINARY_DIR}/FORTRAN_temp")
    get_filename_component(REFPROP_FORTRAN_ABS_PATH "${REFPROP_FORTRAN_PATH}"
                       REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
    message(STATUS "Set REFPROP_FORTRAN_ABS_PATH to ${CMAKE_CURRENT_BINARY_DIR}/FORTRAN_temp")

    set(FIX_91_COMPLETE ON CACHE BOOL "Done fixing fortran files")
  endif()

endif(${REFPROP_SOURCE_AUTOFIX})

# Remove the files COMMONS.FOR and COMTRN.FOR from the list of files to be compiled for REFPROP < 9.1.1
foreach (source ${APP_SOURCES})
  string(FIND "${source}" "COMMONS.FOR" found_INDEX)
  if (${found_INDEX} GREATER 0)
    list(REMOVE_ITEM APP_SOURCES ${source})
  endif()
  string(FIND "${source}" "COMTRN.FOR" found_INDEX)
  if (${found_INDEX} GREATER 0)
    list(REMOVE_ITEM APP_SOURCES ${source})
  endif()
endforeach()

# Add the .DEF file if needed
list(APPEND APP_SOURCES "${DEF_FILE}")

add_library(${NAME} SHARED ${APP_SOURCES} )

if (WIN32)
  # Don't add a 'lib' prefix to the shared library if windows
  set_target_properties(${NAME} PROPERTIES PREFIX "")
endif()

set_target_properties(${NAME} PROPERTIES APPEND_STRING PROPERTY LINK_FLAGS "${REFPROP_LINK_FLAGS} ${REFPROP_ADDITIONAL_LINK_FLAGS}")
target_include_directories(${NAME} PUBLIC "${REFPROP_FORTRAN_ABS_PATH}")

if (NOT REFPROP_NO_NUMPY AND (NOT REFPROP_NO_PYTHON))
  if (NOT EXISTS ${PYTHON_EXECUTABLE})
      message(FATAL_ERROR "Specified Python executable does not exist: ${PYTHON_EXECUTABLE}")
    endif()
  # Build the REFPROP.h header file to go with this shared library
  # See for instance https://samthursfield.wordpress.com/2015/11/21/cmake-dependencies-between-targets-and-files-and-custom-commands/\
  # or other places on the internet...
  set(HEADER_FORTRAN_PATH "${REFPROP_FORTRAN_ABS_PATH}")
  if (EXISTS "${REFPROP_FORTRAN_ABS_PATH}/DLLFILES/PASS_FTN.FOR")
    set(HEADER_FORTRAN_PATH "${REFPROP_FORTRAN_ABS_PATH}/DLLFILES")
  endif()
  set(HEADER_COMMAND ${PYTHON_EXECUTABLE} -u "${CMAKE_CURRENT_SOURCE_DIR}/externals/REFPROP-headers/generate_header.py" --FORTRAN-path "${HEADER_FORTRAN_PATH}" --python-exe "${PYTHON_EXECUTABLE}")
  add_custom_command(
    OUTPUT
      REFPROP.h.stamp
    DEPENDS
      "${HEADER_FORTRAN_PATH}/PASS_FTN.FOR"
    COMMENT
      "About to build the REFPROP.h header file w/ ${HEADER_COMMAND}"
    COMMAND
      ${HEADER_COMMAND}
    COMMAND
      cmake -E touch REFPROP.h.stamp
    WORKING_DIRECTORY
      "${CMAKE_CURRENT_BINARY_DIR}"
    )

  add_custom_target(REFPROP_H
                    ALL
                    DEPENDS
                      REFPROP.h.stamp
                    )
endif()
