# Copyright 2010, Thomas Moulard, LAAS-CNRS
#
# This file is part of roboptim-core.
# roboptim-core is free software: you can redistribute it and/or modify
# it under the terms of the GNU Lesser General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# roboptim-core is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Lesser Public License for more details.
# You should have received a copy of the GNU Lesser General Public License
# along with roboptim-core.  If not, see <http://www.gnu.org/licenses/>.

# Requires at least CMake 2.8 to configure the package.
CMAKE_MINIMUM_REQUIRED(VERSION 3.16)

SET(CMAKE_CXX_STANDARD 11)

SET(PROJECT_NAME roboptim-core-plugin-ipopt)
SET(PROJECT_DESCRIPTION "RobOptim core IPOPT plug-in")
SET(PROJECT_URL "https://github.com/roboptim/roboptim-core-plugin-ipopt")
set(PROJECT_USE_CMAKE_EXPORT TRUE)

INCLUDE(cmake/base.cmake)
INCLUDE(cmake/boost.cmake)
INCLUDE(cmake/cpack.cmake)

compute_project_args(PROJECT_ARGS LANGUAGES CXX C)
project(${PROJECT_NAME} ${PROJECT_ARGS})

# Use MathJax for Doxygen formulae
SET(DOXYGEN_USE_MATHJAX "YES")

SET(HEADERS
  ${CMAKE_SOURCE_DIR}/include/roboptim/core/plugin/ipopt/ipopt-common.hh
  ${CMAKE_SOURCE_DIR}/include/roboptim/core/plugin/ipopt/ipopt-parameters-updater.hh
  ${CMAKE_SOURCE_DIR}/include/roboptim/core/plugin/ipopt/ipopt-td.hh
  ${CMAKE_SOURCE_DIR}/include/roboptim/core/plugin/ipopt/ipopt.hh
  ${CMAKE_SOURCE_DIR}/include/roboptim/core/plugin/ipopt/ipopt-sparse.hh
  )

SET(PKG_CONFIG_ADDITIONAL_VARIABLES plugindir relplugindir ${PKG_CONFIG_ADDITIONAL_VARIABLES})


# If compiler support symbol visibility, enable it.
INCLUDE(CheckCCompilerFlag)
CHECK_C_COMPILER_FLAG(-fvisibility=hidden HAS_VISIBILITY)
IF(HAS_VISIBILITY)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")
ENDIF()

# Search for dependencies.
SET(BOOST_COMPONENTS
  date_time filesystem system thread unit_test_framework)
SEARCH_FOR_BOOST()

# Libtool dynamic loading
# This project does not use Libtool directly but still uses ltdl for
# plug-in loading.
SET(LIB_LTDL LIB_LTDL-NOTFOUND)
FIND_LIBRARY(LIB_LTDL libltdl${CMAKE_SHARED_LIBRARY_SUFFIX})
IF (NOT LIB_LTDL)
  MESSAGE(FATAL_ERROR
    "Failed to find Libtool ltdl library, check that Libtool ltdl is installed.")
ENDIF()
INCLUDE(CheckIncludeFileCXX)
CHECK_INCLUDE_FILE_CXX(ltdl.h LTDL_H_FOUND)
IF (NOT LTDL_H_FOUND)
  MESSAGE(FATAL_ERROR
    "Failed to find ltdl.h, check that Libtool ltdl is installed.")
ENDIF()

# Search for roboptim-core.
ADD_PROJECT_DEPENDENCY(roboptim-core)
add_project_dependency(Eigen3)
# Workaround Ipopt package issue.
# It does not define HAVE_STDDEF even when it is needed, so we will do
# it manually.
CHECK_INCLUDE_FILE_CXX(cstddef HAVE_CSTDDEF)
IF(HAVE_CSTDDEF)
  ADD_DEFINITIONS(-DHAVE_CSTDDEF)
  PKG_CONFIG_APPEND_CFLAGS(-DHAVE_CSTDDEF)
ENDIF()
CHECK_INCLUDE_FILE_CXX(stddef.h HAVE_STDDEF_H)
IF(HAVE_STDDEF_H)
  ADD_DEFINITIONS(-DHAVE_STDDEF_H)
  PKG_CONFIG_APPEND_CFLAGS(-DHAVE_STDDEF_H)
ENDIF()
ADD_REQUIRED_DEPENDENCY("ipopt")


# Which linear solver should Ipopt use (unless explicitly set)?
# Default: mumps (open-source linear solver)
SET(IPOPT_DEFAULT_LINEAR_SOLVER "mumps"
  CACHE STRING "Which linear solver should be used by Ipopt by default?")

# Make sure the MPI version of mumps is not used.
#
# This is a workaround for a bug in the pkg-config file provided by some
# Ubuntu releases where pkg-config always provides information about
# how to link with the MPI version of MUMPS.
IF(CMAKE_SYSTEM_NAME MATCHES "Linux")
  IF(IPOPT_DEFAULT_LINEAR_SOLVER STREQUAL "mumps")
    # Check that Ipopt was compiled with MUMPS support.
    IF(NOT IPOPT_LDFLAGS MATCHES "mumps")
      MESSAGE(FATAL_ERROR "Ipopt was not compiled with MUMPS support, yet it is "
        "set as the default linear solver.")
    ENDIF()

    # Fix for some older Ubuntu packages (14.04 and before).
    IF(EXISTS "/etc/debian_version"
       AND IPOPT_PREFIX MATCHES "/usr"
       AND NOT IPOPT_LDFLAGS MATCHES "-ldmumps_seq")
        STRING(REPLACE "-ldmumps" "-ldmumps_seq" IPOPT_LDFLAGS "${IPOPT_LDFLAGS}")
    ENDIF()
  ENDIF()
ENDIF()

# Enable SIGFPE signal to detect arithmetic errors in tests
OPTION(ENABLE_SIGFPE "Enable floating-point exceptions" OFF)
IF(ENABLE_SIGFPE)
  ADD_DEFINITIONS(-DENABLE_SIGFPE)
ENDIF()

# patch for ipopt: in the .pc file, the header path may be IpoptPath/include/coin
#  for clarity, we want to have IpoptPath/include/ instead
# This patch assumes that the initial header path has the following form :
#  "install-folder/include/coin"
SET(IPOPT_INCLUDE_DIRS_PATCHED ${IPOPT_PREFIX}/include)

#add the path to ipopt to the current include path
INCLUDE_DIRECTORIES(SYSTEM ${IPOPT_INCLUDE_DIRS_PATCHED})

#add this path to the .pc file
PKG_CONFIG_APPEND_CFLAGS(-I${IPOPT_INCLUDE_DIRS_PATCHED})

HEADER_INSTALL("${HEADERS}")

OPTION(DISABLE_TESTS "Disable test programs" OFF)

ADD_SUBDIRECTORY(src)

IF(NOT DISABLE_TESTS)
  ADD_SUBDIRECTORY(tests)
ELSE()
  MESSAGE(WARNING
    "Tests should only be disabled for speficic cases. Do it at your own risk.")
ENDIF()

