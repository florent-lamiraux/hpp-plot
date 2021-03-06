# BSD 2-Clause License

#
# Copyright (c) 2015, 2018, 2020 CNRS
# Authors: Joseph Mirabel, Guilhem Saurel
# All rights reserved.

# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:

# * Redistributions of source code must retain the above copyright notice, this
#   list of conditions and the following disclaimer.

# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.

# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

CMAKE_MINIMUM_REQUIRED(VERSION 3.1)

SET(PROJECT_NAME "hpp-plot")
SET(PROJECT_DESCRIPTION "Plotting tools for HPP")

SET(PROJECT_USE_CMAKE_EXPORT TRUE)
SET(CXX_DISABLE_WERROR TRUE)

INCLUDE(cmake/hpp.cmake)

# Tells pkg-config to read qtversion from pkg config file.
LIST(APPEND PKG_CONFIG_ADDITIONAL_VARIABLES qtversion cmake_plugin)

COMPUTE_PROJECT_ARGS(PROJECT_ARGS LANGUAGES CXX)
PROJECT(${PROJECT_NAME} ${PROJECT_ARGS})

OPTION(HPP_MANIPULATION_CORBA_CLIENT_ONLY "Disable functionnalities requiring the corba servers of HPP" OFF)

ADD_PROJECT_DEPENDENCY("gepetto-viewer")
ADD_PROJECT_DEPENDENCY("gepetto-viewer-corba")

IF(gepetto-viewer-corba_FOUND)
  # Get desired Qt version
  string(REPLACE "." ";" DESIRED_QT_VERSION_LIST ${GEPETTO_VIEWER_CORBA_QTVERSION})
  list(GET DESIRED_QT_VERSION_LIST 0 DESIRED_QT_VERSION_MAJOR)

  IF(${DESIRED_QT_VERSION_MAJOR} EQUAL 4)
    SET(USE_QT4 True)
  ELSEIF(${DESIRED_QT_VERSION_MAJOR} EQUAL 5)
    SET(USE_QT4 False)
  ELSE()
    MESSAGE(FATAL_ERROR "This package is only compatible with Qt 4 and Qt 5")
  ENDIF()
ELSE(gepetto-viewer-corba_FOUND)
  SET(USE_QT4 FALSE CACHE BOOL "Use Qt4 instead of Qt5")
ENDIF(gepetto-viewer-corba_FOUND)

MESSAGE(STATUS "Looking for Qt ${DESIRED_QT_VERSION_MAJOR}.")
SET(CMAKE_INCLUDE_CURRENT_DIR ON)
IF(USE_QT4)
  FIND_PACKAGE(Qt4 REQUIRED QtCore QtGui)
  SET(PKG_CONFIG_EXTRA "qtversion: ${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}.${QT_VERSION_PATCH}")
ELSE(USE_QT4)
  FOREACH (component "Core" "Widgets" "Gui" "PrintSupport" "Concurrent")
    FIND_PACKAGE ("Qt5${component}" REQUIRED)
    LIST(APPEND QT_INCLUDE_DIRS ${Qt5${component}_INCLUDE_DIRS})
    LIST(APPEND QT_LIBRARIES    ${Qt5${component}_LIBRARIES}   )
  ENDFOREACH (component "Core" "Widgets" "Gui" "PrintSupport" "Concurrent")
  SET(PKG_CONFIG_EXTRA "qtversion=${Qt5Core_VERSION}")
ENDIF(USE_QT4)

ADD_PROJECT_DEPENDENCY("hpp-manipulation-corba" REQUIRED)
ADD_REQUIRED_DEPENDENCY("qgv")

SET (${PROJECT_NAME}_HEADERS
  include/hpp/plot/graph-widget.hh
  include/hpp/plot/hpp-manipulation-graph.hh
  )
SET (${PROJECT_NAME}_HEADERS_NOMOC
)

SET(${PROJECT_NAME}_FORMS
  )

SET(${PROJECT_NAME}_RESOURCES
  )

IF(USE_QT4)
  QT4_WRAP_CPP(${PROJECT_NAME}_HEADERS_MOC ${${PROJECT_NAME}_HEADERS})
  QT4_WRAP_UI(${PROJECT_NAME}_FORMS_HEADERS ${${PROJECT_NAME}_FORMS})
  QT4_ADD_RESOURCES(${PROJECT_NAME}_RESOURCES_RCC ${${PROJECT_NAME}_RESOURCES})
  INCLUDE(${QT_USE_FILE})
ELSE(USE_QT4)
  QT5_WRAP_CPP(${PROJECT_NAME}_HEADERS_MOC ${${PROJECT_NAME}_HEADERS})
  QT5_WRAP_UI(${PROJECT_NAME}_FORMS_HEADERS ${${PROJECT_NAME}_FORMS})
  QT5_ADD_RESOURCES(${PROJECT_NAME}_RESOURCES_RCC ${${PROJECT_NAME}_RESOURCES})
ENDIF(USE_QT4)
ADD_DEFINITIONS(${QT_DEFINITIONS})

SET (${PROJECT_NAME}_SOURCES
  src/graph-widget.cc
  src/hpp-manipulation-graph.cc
  )

ADD_LIBRARY (${PROJECT_NAME}
  SHARED
  ${${PROJECT_NAME}_SOURCES}
  ${${PROJECT_NAME}_HEADERS_NOMOC}
  ${${PROJECT_NAME}_HEADERS_MOC}
  ${${PROJECT_NAME}_FORMS_HEADERS}
  ${${PROJECT_NAME}_RESOURCES_RCC}
  )

TARGET_LINK_LIBRARIES(${PROJECT_NAME} ${QT_LIBRARIES} hpp-manipulation-corba::hpp-manipulation-corba)
PKG_CONFIG_USE_DEPENDENCY(${PROJECT_NAME} qgv)

INSTALL(TARGETS ${PROJECT_NAME} EXPORT ${TARGETS_EXPORT_NAME} DESTINATION lib)

ADD_SUBDIRECTORY (bin)
ADD_SUBDIRECTORY (plugins)
