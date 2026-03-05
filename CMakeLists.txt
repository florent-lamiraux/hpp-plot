# BSD 2-Clause License

#
# Copyright (c) 2015, 2018, 2020 CNRS Authors: Joseph Mirabel, Guilhem Saurel
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

cmake_minimum_required(VERSION 3.22)

set(PROJECT_NAME "hpp-plot")
set(PROJECT_DESCRIPTION "Plotting tools for HPP")

set(PROJECT_USE_CMAKE_EXPORT TRUE)
set(PROJECT_USE_KEYWORD_LINK_LIBRARIES TRUE)
set(CXX_DISABLE_WERROR TRUE)

# Check if the submodule cmake have been initialized
set(JRL_CMAKE_MODULES "${CMAKE_CURRENT_LIST_DIR}/cmake")
if(EXISTS "${JRL_CMAKE_MODULES}/base.cmake")
  message(STATUS "JRL cmakemodules found in 'cmake/' git submodule")
else()
  find_package(jrl-cmakemodules QUIET CONFIG)
  if(jrl-cmakemodules_FOUND)
    get_property(
      JRL_CMAKE_MODULES
      TARGET jrl-cmakemodules::jrl-cmakemodules
      PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
    message(STATUS "JRL cmakemodules found on system at ${JRL_CMAKE_MODULES}")
  else()
    message(STATUS "JRL cmakemodules not found. Let's fetch it.")
    include(FetchContent)
    FetchContent_Declare(
      "jrl-cmakemodules"
      GIT_REPOSITORY "https://github.com/jrl-umi3218/jrl-cmakemodules.git")
    FetchContent_MakeAvailable("jrl-cmakemodules")
    FetchContent_GetProperties("jrl-cmakemodules" SOURCE_DIR JRL_CMAKE_MODULES)
  endif()
endif()

include("${JRL_CMAKE_MODULES}/hpp.cmake")
include("${JRL_CMAKE_MODULES}/python.cmake")
include("${JRL_CMAKE_MODULES}/boost.cmake")

# Tells pkg-config to read qtversion from pkg config file.
list(APPEND PKG_CONFIG_ADDITIONAL_VARIABLES qtversion cmake_plugin)

compute_project_args(PROJECT_ARGS LANGUAGES CXX)
project(${PROJECT_NAME} ${PROJECT_ARGS})

message(STATUS "Looking for Qt ${DESIRED_QT_VERSION_MAJOR}.")
set(CMAKE_INCLUDE_CURRENT_DIR ON)
find_package(
  Qt5 REQUIRED
  COMPONENTS Core
             Widgets
             Gui
             PrintSupport
             Concurrent
             OpenGL
             Network
             Xml)
foreach(
  component
  "Core"
  "Widgets"
  "Gui"
  "PrintSupport"
  "Concurrent"
  OpenGL
  Network
  Xml)
  list(APPEND QT_INCLUDE_DIRS ${Qt5${component}_INCLUDE_DIRS})
  list(APPEND QT_LIBRARIES ${Qt5${component}_LIBRARIES})
endforeach()
set(PKG_CONFIG_EXTRA "qtversion=${Qt5Core_VERSION}")

add_project_dependency("hpp-manipulation" REQUIRED)
add_project_dependency("qgv" REQUIRED)

# Python bindings for native graph viewer
set(PYTHON_COMPONENTS Interpreter Development NumPy)
findpython()
search_for_boost_python()

set(${PROJECT_NAME}_HEADERS
    include/hpp/plot/graph-widget.hh
    include/hpp/plot/hpp-native-graph.hh)
set(${PROJECT_NAME}_HEADERS_NOMOC)

set(${PROJECT_NAME}_FORMS)

set(${PROJECT_NAME}_RESOURCES)

qt5_wrap_cpp(${PROJECT_NAME}_HEADERS_MOC ${${PROJECT_NAME}_HEADERS})
qt5_wrap_ui(${PROJECT_NAME}_FORMS_HEADERS ${${PROJECT_NAME}_FORMS})
qt5_add_resources(${PROJECT_NAME}_RESOURCES_RCC ${${PROJECT_NAME}_RESOURCES})

add_definitions(${QT_DEFINITIONS})

set(${PROJECT_NAME}_SOURCES src/graph-widget.cc src/hpp-native-graph.cc)

add_library(
  ${PROJECT_NAME} SHARED
  ${${PROJECT_NAME}_SOURCES} ${${PROJECT_NAME}_HEADERS_NOMOC}
  ${${PROJECT_NAME}_HEADERS_MOC} ${${PROJECT_NAME}_FORMS_HEADERS}
  ${${PROJECT_NAME}_RESOURCES_RCC})

target_link_libraries(
  ${PROJECT_NAME}
  PUBLIC ${QT_LIBRARIES} hpp-manipulation::hpp-manipulation qgv::qgvcore)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${TARGETS_EXPORT_NAME}
  DESTINATION lib)

# Python module for native graph viewer
add_library(pyhpp_plot_graph_viewer MODULE src/pyhpp_plot/graph_viewer.cc)
target_link_boost_python(pyhpp_plot_graph_viewer PUBLIC)
target_link_libraries(
  pyhpp_plot_graph_viewer
  PUBLIC ${PROJECT_NAME} ${QT_LIBRARIES} hpp-manipulation::hpp-manipulation
         ${PYTHON_LIBRARIES})
target_include_directories(pyhpp_plot_graph_viewer
                           PRIVATE ${PYTHON_INCLUDE_DIRS})
set_target_properties(
  pyhpp_plot_graph_viewer
  PROPERTIES PREFIX ""
             OUTPUT_NAME "graph_viewer"
             LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/pyhpp_plot")

# pyhpp_plot_graph_viewer is a python module that spawn Qt windows. for this, at
# runtime, it need a QT_PLUGIN_PATH env var with a path where the shared lib for
# the current Qt "platform" are. Here we find a QT_PLUGIN_PATH that would work
# at config time, so that at runtime we can append that as a default fallback,
# if a end user has Qt installed in a non-FHS standard place and did not
# manually tune their $QT_PLUGIN_PATH env var for it

set(QT_PLUGIN_PATH_CMAKE)

# First, we might need to manually discover platforms plugins which are not
# installed in the same prefix as Qt5Gui. eg. Wayland.
find_package(
  Qt5 CONFIG
  COMPONENTS WaylandClient
  QUIET)
if(Qt5WaylandClient_FOUND)
  cmake_path(GET Qt5WaylandClient_DIR PARENT_PATH _p)
  file(GLOB pluginTargets "${_p}/Qt5Gui/Qt5Gui_*.cmake")
  if(pluginTargets)
    foreach(pluginTarget ${pluginTargets})
      include(${pluginTarget})
    endforeach()
  endif()
endif()

foreach(p IN LISTS Qt5Gui_PLUGINS)
  get_target_property(_l ${p} IMPORTED_LOCATION_RELEASE)
  cmake_path(GET _l PARENT_PATH _pp)
  cmake_path(GET _pp PARENT_PATH _ppp)
  if(NOT _ppp IN_LIST QT_PLUGIN_PATH_CMAKE)
    set(QT_PLUGIN_PATH_CMAKE ${QT_PLUGIN_PATH_CMAKE} ${_ppp})
  endif()
endforeach()

add_compile_definitions(
  "QT_PLUGIN_PATH_CMAKE=\"$<JOIN:${QT_PLUGIN_PATH_CMAKE},:>\"")

install(TARGETS pyhpp_plot_graph_viewer
        DESTINATION "${PYTHON_SITELIB}/pyhpp_plot")

# Install Python modules
install(FILES "${CMAKE_SOURCE_DIR}/src/pyhpp_plot/__init__.py"
              "${CMAKE_SOURCE_DIR}/src/pyhpp_plot/interactive_viewer.py"
        DESTINATION "${PYTHON_SITELIB}/pyhpp_plot")
