cmake_minimum_required(VERSION 3.14)

########################################################################
#
# Set variables for AMReX versioning
#
########################################################################
find_package (Git QUIET)

set( _tmp "" )

# Try to inquire software version from git
if ( EXISTS ${CMAKE_CURRENT_LIST_DIR}/.git AND ${GIT_FOUND} )
   execute_process ( COMMAND git describe --abbrev=12 --dirty --always --tags
      WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
      OUTPUT_VARIABLE _tmp )
   string( STRIP ${_tmp} _tmp )
   # filter invalid descriptions in shallow git clones
   if (NOT _tmp MATCHES "^([0-9]+)\\.([0-9]+)(\\.([0-9]+))*(-.*)*$")
       set( _tmp "")
   endif ()
endif()

# Grep first line from file CHANGES if cannot find version from Git
if (NOT _tmp)
   file(STRINGS ${CMAKE_CURRENT_LIST_DIR}/CHANGES ALL_VERSIONS REGEX "#")
   list(GET ALL_VERSIONS 0 _tmp)
   string(REPLACE "#" "" _tmp "${_tmp}")
   string(STRIP "${_tmp}" _tmp )
   set(_tmp "${_tmp}.0")
endif ()

set( AMREX_GIT_VERSION "${_tmp}" CACHE INTERNAL "" )
unset(_tmp)

# Package version is a modified form of AMREX_GIT_VERSION
if (AMREX_GIT_VERSION)
    string(FIND "${AMREX_GIT_VERSION}" "-" _idx REVERSE)
    string(SUBSTRING "${AMREX_GIT_VERSION}" 0 "${_idx}" _pkg_version )
    string(FIND "${_pkg_version}" "-" _idx REVERSE)
    string(SUBSTRING "${_pkg_version}" 0 "${_idx}" _pkg_version )
    string(REPLACE "-" "." _pkg_version "${_pkg_version}")
endif ()

set( AMREX_PKG_VERSION "${_pkg_version}" CACHE INTERNAL "" )
unset(_pkg_version)

#
# For the time being, set this option here.
# We must do this because we need to provide a default for CMAKE_CXX_COMPILER
#
option( ENABLE_DPCPP  "Enable DPCPP support" OFF )
if (ENABLE_DPCPP)
   if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
      # Set default for CMAKE_CXX_COMPILER only if amrex
      # is not a subproject
      set(CMAKE_CXX_COMPILER dpcpp)
   endif ()
   message(STATUS "Enabling experimental support for dpc++")
endif ()


########################################################################
#
# AMReX project
#
########################################################################
project( AMReX
    DESCRIPTION "A software framework for massively parallel, block-structured adaptive mesh refinement (AMR) applications"
    VERSION  ${AMREX_PKG_VERSION}
    HOMEPAGE_URL "https://amrex-codes.github.io/amrex/"
    LANGUAGES C CXX
    )

message(STATUS "CMake version: ${CMAKE_VERSION}")

#
# Load required modules
#
set( AMREX_CMAKE_MODULES_PATH "${CMAKE_CURRENT_LIST_DIR}/Tools/CMake" CACHE INTERNAL "" )
set( CMAKE_MODULE_PATH ${AMREX_CMAKE_MODULES_PATH} )

#
# Provide a default install directory
#
if ( CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT )
    set ( CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}/installdir"
          CACHE PATH "AMReX installation directory" FORCE)
endif ()

message(STATUS "AMReX installation directory: ${CMAKE_INSTALL_PREFIX}")

#
# Check if CMAKE_BUILD_TYPE is given. If not, use default
#
if ( NOT CMAKE_BUILD_TYPE )
   set(CMAKE_CONFIGURATION_TYPES "Release;Debug;MinSizeRel;RelWithDebInfo")
   set(CMAKE_BUILD_TYPE Release
       CACHE STRING
       "Choose the build type, e.g. Release, Debug, or RelWithDebInfo." FORCE)
else ()
   message(STATUS "Build type set by user to '${CMAKE_BUILD_TYPE}'.")
endif()

#
# Include options, utilities and other stuff we need
#
include( AMReX_Utils )
include( AMReX_Options )

#
# Enable Fortran if requested
#
if(ENABLE_FORTRAN)
   enable_language(Fortran)
endif ()

#
# Enable CUDA if requested
#
if (ENABLE_CUDA)
    enable_language(CUDA)
    include(AMReX_SetupCUDA)
endif ()

#
# Check compiler version
#
set_mininum_cxx_compiler_version(GNU 4.8)
set_mininum_cxx_compiler_version(MSVC 19.23)

#
# Set CMAKE_<LANG>_FLAGS_<CONFIG> if not already defined
#
set_default_config_flags ()

#
# Source files for all binaries and libraries found under src
#
add_subdirectory(Src)

#
# Tutorials and "test_install" target
#
option(ENABLE_TUTORIALS "Enable Tutorials" NO)

if (ENABLE_TUTORIALS)
   add_subdirectory(Tutorials)
endif ()

#
# Plotfile tools
#
option(ENABLE_PLOTFILE_TOOLS "Enable Plotfile tools" NO)

if (ENABLE_PLOTFILE_TOOLS)
   # If this get executed, it cannot be EXCLUDED_FROM_ALL
   # because it needs to get installed
   add_subdirectory(Tools/Plotfile)
endif ()
