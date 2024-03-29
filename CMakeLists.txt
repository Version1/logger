#
# Copyright 2014-2018 Neueda Ltd.
#
cmake_minimum_required(VERSION 2.8.0)
project(logger CXX)

set(SUBMODULE_FLAGS -DTESTS=off)

if(SWIG_EXECUTABLE)
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DSWIG_EXECUTABLE=${SWIG_EXECUTABLE})
endif()

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set (CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "default install path" FORCE)
endif()

if(WIN32)
  set(CMAKE_SHARED_LINKER_FLAGS "/FORCE:UNRESOLVED")
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
  set(BUILD_SHARED_LIBS TRUE)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MD /DNOMINMAX /DYY_NO_UNISTD_H")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MDd /DNOMINMAX /DYY_NO_UNISTD_H")
endif()

if(APPLE)
  set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")
  set(CMAKE_XCODE_ATTRIBUTE_GCC_VERSION "com.apple.compilers.llvm.clang.1_0")
  set(CMAKE_SHARED_LINKER_FLAGS "-Wl,-undefined,dynamic_lookup")
  set(CMAKE_MACOSX_RPATH 1)
endif()

# options
option(DEBUG "Enable debug build" OFF)
option(JAVA "Enable Java bindings" OFF)
option(PYTHON "Enable Python bindings" OFF)
option(CSHARP "Enable C# bindings" OFF)
option(COVERAGE "Enable gcov coverage" OFF)
option(EXAMPLES "Include example apps in install" ON)
set(PYTHON_CONFIG "python2-config" CACHE STRING "python-config for build env config")

# set version info
set(LOGGER_MAJOR_VERSION 0)
set(LOGGER_MINOR_VERSION 3)
set(LOGGER_PATCH_VERSION 2)
set(LOGGER_VERSION ${LOGGER_MAJOR_VERSION}.${LOGGER_MINOR_VERSION}.${LOGGER_PATCH_VERSION})

if (UNIX)
  macro (add_library _name)
    _add_library(${ARGV})
    if (NOT APPLE AND TARGET ${_name})
      SET_TARGET_PROPERTIES(${_name}
        PROPERTIES
            VERSION ${LOGGER_VERSION}
            # SOVERSION 0.0.0
        )
    endif()
  endmacro()
endif (UNIX)

add_compile_options(-Wall -c -g)
if (DEBUG)
  message(STATUS "logger DEBUG: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DDEBUG=on)
else ()
  message(STATUS "logger DEBUG: OFF")
  add_compile_options(-O3)
endif (DEBUG)

if (JAVA)
  message(STATUS "JAVA: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DJAVA=ON)
else ()
  message(STATUS "JAVA: OFF")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DJAVA=OFF)
endif()
if (PYTHON)
  message(STATUS "PYTHON: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON=ON -DPYTHON_CONFIG=${PYTHON_CONFIG})
else ()
  message(STATUS "PYTHON: OFF")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON=OFF)
endif()
if (CSHARP)
  message(STATUS "CSHARP: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DCSHARP=ON)
else ()
  message(STATUS "CSHARP: OFF")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DCSHARP=OFF)
endif()

# need flex
find_package(FLEX REQUIRED)

# allow external projects
include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)

# extra links
link_directories(${CMAKE_INSTALL_PREFIX}/lib)

# coverage
if (COVERAGE)
    set(COVERAGE_COMPILE_FLAGS "-g -O0 -coverage -fprofile-arcs -ftest-coverage")
    set(COVERAGE_LINK_FLAGS    "-coverage -lgcov")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${COVERAGE_COMPILE_FLAGS}" )
    set(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} ${COVERAGE_LINK_FLAGS}" )
endif(COVERAGE)

# deps
if(NOT SUBMODULE_ROOT_DIR)
    set(SUBMODULE_ROOT_DIR ${PROJECT_SOURCE_DIR})
endif()
if(NOT SUBMODULE_PREFIX_DIR)
    set(SUBMODULE_PREFIX_DIR ${CMAKE_BINARY_DIR})
endif()

find_library(EVENT_LIB
  NAMES event
  HINTS "${CMAKE_INSTALL_PREFIX}/lib"
  )
find_library(EVENT_PTHREAD_LIB
  NAMES event_pthreads
  HINTS "${CMAKE_INSTALL_PREFIX}/lib"
  )
find_path(EVENT_INCLUDE_DIR event2/event.h
  PATHS "${CMAKE_INSTALL_PREFIX}/include")
if (EVENT_LIB)
  message(STATUS "event-found: " ${EVENT_LIB})
  message(STATUS "event-pthread-found: " ${EVENT_PTHREAD_LIB})
  message(STATUS "event-headers-found: " ${EVENT_INCLUDE_DIR})
else()
  set(EVENT_INCLUDE_DIR ${CMAKE_INSTALL_PREFIX}/include/event2)
  set(LIBEVENT_CMAKE ${SUBMODULE_ROOT_DIR}/ext/libevent CACHE PATH "Location of libevent-cmake" FORCE)
  message(STATUS "libevent-location: " ${LIBEVENT_CMAKE})
  ExternalProject_Add(
    libevent
    PREFIX ${SUBMODULE_PREFIX_DIR}
    SOURCE_DIR ${LIBEVENT_CMAKE}
    CMAKE_ARGS -DEVENT__DISABLE_TESTS=ON -DEVENT__DISABLE_REGRESS=ON -DEVENT__DISABLE_OPENSSL=ON -DEVENT__DISABLE_SAMPLES=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
    )
  set(EVENT_LIB event)
endif()

include_directories(
  ${PROJECT_SOURCE_DIR}/src
  ${CMAKE_INSTALL_PREFIX}/include/
  ${CMAKE_INSTALL_PREFIX}/include/sbf
  ${CMAKE_INSTALL_PREFIX}/include/utils
  ${EVENT_INCLUDE_DIR}
  ${CMAKE_INSTALL_PREFIX}/include/properties)

# sbf
set(SBF_CMAKE ${SUBMODULE_ROOT_DIR}/ext/sbf CACHE PATH "Location of sbf-cmake" FORCE)
message(STATUS "sbf-location: " ${SBF_CMAKE})
ExternalProject_Add(
    SBF_PROJECT
    PREFIX ${SUBMODULE_PREFIX_DIR}
    SOURCE_DIR ${SBF_CMAKE}
    CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
    )
ExternalProject_Get_Property(SBF_PROJECT binary_dir)
set(SBF_PROJECT_DIR ${binary_dir}/lib/cmake/sbf CACHE PATH "location of sbf cmake files" FORCE)
if (NOT EVENT_LIB)
    add_dependencies(SBF_PROJECT libevent)
endif()
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};sbf;ALL;/")

# utils
set(UTILS_CMAKE ${SUBMODULE_ROOT_DIR}/ext/utils CACHE PATH "Location of utils-cmake" FORCE)
message(STATUS "utils-location: " ${UTILS_CMAKE})
ExternalProject_Add(
    UTILS_PROJECT
    PREFIX ${SUBMODULE_PREFIX_DIR}
    SOURCE_DIR ${UTILS_CMAKE}
    CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
    )
ExternalProject_Get_Property(UTILS_PROJECT binary_dir)
set(UTILS_PROJECT_DIR ${binary_dir}/lib/cmake/utils CACHE PATH "location of utils cmake files" FORCE)
add_dependencies(UTILS_PROJECT SBF_PROJECT)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};utils;ALL;/")

# properties
set(PROPERTIES_CMAKE ${SUBMODULE_ROOT_DIR}/ext/properties CACHE PATH "Location of properties-cmake" FORCE)
message(STATUS "properties-location: " ${PROPERTIES_CMAKE})
ExternalProject_Add(
    PROPERTIES_PROJECT
    PREFIX ${SUBMODULE_PREFIX_DIR}
    SOURCE_DIR ${PROPERTIES_CMAKE}
    CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
    )
ExternalProject_Get_Property(PROPERTIES_PROJECT binary_dir)
set(PROPERTIES_PROJECT_DIR ${binary_dir}/lib/cmake/properties CACHE PATH "location of properties cmake files" FORCE)
add_dependencies(PROPERTIES_PROJECT SBF_PROJECT UTILS_PROJECT)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};properties;ALL;/")

# unit-tests
option(TESTS "Enable unit-tests" OFF)
if(TESTS)
  if(WIN32)
    ExternalProject_Add(
      googletest
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/googletest
      CMAKE_ARGS -Dgtest_force_shared_crt=ON -DCMAKE_CXX_FLAGS="/w" -DBUILD_GMOCK=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest -DCMAKE_INSTALL_LIBDIR=${CMAKE_BINARY_DIR}/googletest/lib
      )
  else()
    ExternalProject_Add(
      googletest
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/googletest
      CMAKE_ARGS -DBUILD_GMOCK=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest -DCMAKE_INSTALL_LIBDIR=${CMAKE_BINARY_DIR}/googletest/lib
      )
  endif()

  include(CTest)
  enable_testing()

  add_subdirectory(test)
endif()

# add source
add_subdirectory(src)

# add examples
if(EXAMPLES)
  add_subdirectory(examples)
endif()

set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${CMAKE_BINARY_DIR};logger;ALL;/")
set(CPACK_GENERATOR "TGZ")
set(CPACK_PACKAGE_VERSION_MAJOR "${LOGGER_MAJOR_VERSION}")
set(CPACK_PACKAGE_VERSION_MINOR "${LOGGER_MINOR_VERSION}")
set(CPACK_PACKAGE_VERSION_PATCH "${LOGGER_PATCH_VERSION}")
set(CPACK_PACKAGE_NAME "logger")
set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
if(DEBUG)
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}_dbg-${CPACK_PACKAGE_VERSION}")
else()
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}")
endif()

INCLUDE(CPack)
