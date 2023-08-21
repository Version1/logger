#
# Copyright 2014-2018 Neueda Ltd.
#
# Include python
execute_process(
    COMMAND bash -c "${PYTHON_CONFIG} --includes"
    OUTPUT_VARIABLE PYTHON_INCLUDE_PATH
    RESULT_VARIABLE RET
    )
if(NOT ${RET} EQUAL "0")
    message(FATAL_ERROR "failed to run python-config --includes")
endif()

execute_process(
    COMMAND bash -c "${PYTHON_CONFIG} --ldflags"
    OUTPUT_VARIABLE PYTHON_LDFLAGS
    RESULT_VARIABLE RET
    )
if(NOT ${RET} EQUAL "0")
    message(FATAL_ERROR "failed to run python-config --ldflags")
endif()

execute_process(
    COMMAND bash -c "${PYTHON_CONFIG} --libs"
    OUTPUT_VARIABLE PYTHON_LIBS
    RESULT_VARIABLE RET
    )
if(NOT ${RET} EQUAL "0")
    message(FATAL_ERROR "failed to run python-config --libs")
endif()

string(STRIP ${PYTHON_LDFLAGS} PYTHON_LDFLAGS)
string(STRIP ${PYTHON_INCLUDE_PATH} PYTHON_INCLUDE_PATH)
string(STRIP ${PYTHON_LIBS} PYTHON_LIBS)

message(STATUS "Python includes: ${PYTHON_INCLUDE_PATH}")
message(STATUS "Python ldflags: ${PYTHON_LDFLAGS}")
message(STATUS "Python libs: ${PYTHON_LIBS}")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${PYTHON_INCLUDE_PATH}")
link_libraries(${PYTHON_LDFLAGS})

# set the flags for swig
set(CMAKE_SWIG_FLAGS -module logger -threads)
set_source_files_properties(../logger.py.i PROPERTIES CPLUSPLUS ON)

# Add swig module
swig_add_library(Logger
    LANGUAGE python
    SOURCES ../logger.py.i)
swig_link_libraries(Logger logger ${PYTHON_LIBS})
set_target_properties(${SWIG_MODULE_Logger_REAL_NAME} PROPERTIES OUTPUT_NAME "_logger")

# Files to install with Python
set(PYTHON_INSTALL_FILES
  # ${CMAKE_CURRENT_SOURCE_DIR}/example-log-handler.py
  ${CMAKE_CURRENT_BINARY_DIR}/logger.py)

# Declare install target for python
install(TARGETS ${SWIG_MODULE_Logger_REAL_NAME}
  DESTINATION lib/python)
install(FILES ${PYTHON_INSTALL_FILES}
  DESTINATION lib/python)

