project(dbus_python CXX C)
cmake_minimum_required(VERSION 2.8)
  
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

set(PACKAGE "dbus-python")
set(PACKAGE_BUGREPORT "http://bugs.freedesktop.org/enter_bug.cgi?product=dbus&component=python")
set(PACKAGE_NAME "dbus-python")
set(PACKAGE_URL "")
set(PACKAGE_VERSION "1.2.14")
set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set(PACKAGE_TARNAME "${PACKAGE_NAME}")

include (CheckTypeSize) 

#DBUS
find_package(DBUS REQUIRED)
set(CMAKE_REQUIRED_INCLUDES ${DBUS_INCLUDES})
set(CMAKE_EXTRA_INCLUDE_FILES dbus/dbus.h)
check_type_size(DBusBasicValue DBUSBASICVALUE)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

#GLIB
find_package(GLIB REQUIRED)
			       
# Find Python executable  
find_package(PythonInterp REQUIRED)
find_package(PythonLibs REQUIRED)

if(NOT PYTHONLIBS_FOUND OR NOT PYTHON_EXECUTABLE)
  message(SEND_ERROR "You need Python to build the dbus python bindings")
endif(NOT PYTHONLIBS_FOUND OR NOT PYTHON_EXECUTABLE)

# The code below prints the Python extension for the current system
set(get_python_suffix "import imp ; print(list(filter(lambda s : s[1] == 'rb' and s[0][0] == '.', imp.get_suffixes()))[0][0])")

#find_file(stdint stdint.h)
#if(NOT stdint)
#  message(SEND_ERROR "You need a C99 compliant stdint.h for windows, use e.g. http://msinttypes.googlecode.com/svn/trunk/stdint.h")
#endif(NOT stdint)

execute_process(
  COMMAND ${PYTHON_EXECUTABLE} -c "${get_python_suffix}"
  OUTPUT_VARIABLE _modsuffix
)
string(REPLACE "\n" "" _modsuffix ${_modsuffix}) 
message(STATUS "Python module suffix is: ${_modsuffix}")

# The code below returns the site-packages directory for the current system
set(get_site_package_dir "from distutils.sysconfig import get_python_lib; print(get_python_lib())")
execute_process(
  COMMAND ${PYTHON_EXECUTABLE} -c "${get_site_package_dir}"
  OUTPUT_VARIABLE site_pkg
)
string(REPLACE "\n" "" site_pkg ${site_pkg}) 
message(STATUS "Python module path is: ${site_pkg}")

include_directories(include/ ${PYTHON_INCLUDE_DIRS})
include_directories(dbus_bindings/)
include_directories(./)
file(GLOB dbus_binding_sources dbus_bindings/*.c)
add_library(dbus_bindings SHARED ${dbus_binding_sources})
target_link_libraries(dbus_bindings ${PYTHON_LIBRARIES} ${DBUS_LIBRARY})


include_directories(dbus_glib_bindings/ ${GLIB_INCLUDE_DIR} ${GLIB_CONFIG_INCLUDE_DIR} ${DBUS_INCLUDES})
file(GLOB dbus_glib_binding_sources dbus_glib_bindings/*.c)
file(GLOB dbus_gmain_sources dbus-gmain/*.c)
add_library(dbus_glib_bindings SHARED ${dbus_glib_binding_sources} ${dbus_gmain_sources})
target_link_libraries(dbus_glib_bindings ${PYTHON_LIBRARIES} ${DBUS_LIBRARY} ${DBUS_GLIB_LIBRARY} ${GLIB_LIBRARIES})

set_target_properties(dbus_bindings dbus_glib_bindings
 PROPERTIES
   PREFIX "" # There is no prefix even on UNIXes
   SUFFIX "${_modsuffix}" # The extension got from Python libraries
)

set(PYTHON ${PYTHON_EXECUTABLE})
set(abs_top_srcdir ${CMAKE_CURRENT_SOURCE_DIR})
set(abs_top_builddir ${CMAKE_CURRENT_BINARY_DIR})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/test/tmp-session-bus.conf.in  ${CMAKE_CURRENT_BINARY_DIR}/test/tmp-session-bus.conf)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/test/TestSuitePythonService.service.in  ${CMAKE_CURRENT_BINARY_DIR}/test/TestSuitePythonService.service)
