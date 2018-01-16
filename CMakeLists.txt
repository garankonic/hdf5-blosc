cmake_minimum_required(VERSION 2.8.10)
project(blosc_hdf5)
include(ExternalProject)

# options
option(BUILD_TESTS
    "Build test programs form the blosc filter" ON)

option(BUILD_PLUGIN
    "Build dynamically loadable plugin for HDF5 version > 1.8.11" ON)
if(BUILD_PLUGIN)
    set(PLUGIN_INSTALL_PATH "/usr/local/hdf5/lib/plugin" CACHE PATH 
      "Where to install the dynamic HDF5-plugin")
endif(BUILD_PLUGIN)

set(BLOSC_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/blosc")
set(BLOSC_INSTALL_DIR "${CMAKE_CURRENT_BINARY_DIR}/blosc")
set(BLOSC_CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${BLOSC_INSTALL_DIR})

message("BLOSC_PREFIX='${BLOSC_PREFIX}'")
message("BLOSC_INSTALL_DIR='${BLOSC_INSTALL_DIR}'")
message("BLOSC_CMAKE_ARGS='${BLOSC_CMAKE_ARGS}'")
message("GIT_EXECUTABLE='${GIT_EXECUTABLE}'")

ExternalProject_Add(project_blosc
  PREFIX ${BLOSC_PREFIX}
  GIT_REPOSITORY https://github.com/Blosc/c-blosc.git
  INSTALL_DIR ${BLOSC_INSTALL_DIR}
  CMAKE_ARGS ${BLOSC_CMAKE_ARGS}
)


# sources
set(SOURCES src/blosc_filter.c)
set(PLUGIN_SOURCES src/blosc_filter.c src/blosc_plugin.c )

# dependencies
if(MSVC)
    # FindHDF5.cmake does not find Windows installations. Try to
    # use an environment variable instead until the official "find"
    # file can be updated for Windows.
    #
    # Note that you have to set this environment variable by hand.
    file(TO_CMAKE_PATH "$ENV{HDF5_DIR}" HDF5_HINT)
    set(HDF5_DIR ${HDF5_HINT} CACHE STRING "Path to HDF5 CMake config directory.")
    find_package(HDF5 REQUIRED HINTS ${HDF5_DIR})
else(MSVC)
    #find_package(HDF5 REQUIRED)
	#set (LIB_TYPE STATIC) # or SHARED
	#string(TOLOWER ${LIB_TYPE} SEARCH_TYPE)
	set(HDF5_ROOT /home/maren/Soft/hdf5-1.10.1_install)
	set(HDF5_INCLUDE_DIRS /home/maren/Soft/hdf5-1.10.1_install/include)
	#find_package (HDF5 COMPONENTS CXX)
	#if(HDF5_FOUND)
		message("HDF5_ROOT='${HDF5_ROOT}'")
		message("HDF5_INCLUDE_DIRS='${HDF5_INCLUDE_DIRS}'")
		#set (HDF5_libraries ${HDF5_hdf5_LIBRARY} ${HDF5_hdf5_cpp_LIBRARY})
	  	set (HDF5_LIBRARIES "-L/home/maren/Soft/hdf5-1.10.1_install/lib/ /home/maren/Soft/hdf5-1.10.1_install/lib/libhdf5_cpp-static.a /home/maren/Soft/hdf5-1.10.1_install/lib/libhdf5-static.a -lz -lm -ldl -lsnappy")
		message("HDF5_LIBRARIES='${HDF5_LIBRARIES}'")
	#endif(HDF5_FOUND)
endif(MSVC)
include_directories(${HDF5_INCLUDE_DIRS})


# add blosc libraries
add_library(blosc_static STATIC IMPORTED)
set_property(TARGET blosc_static PROPERTY IMPORTED_LOCATION ${BLOSC_INSTALL_DIR}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}blosc${CMAKE_STATIC_LIBRARY_SUFFIX})
add_dependencies(blosc_static project_blosc)
include_directories(${BLOSC_INSTALL_DIR}/include)

add_library(blosc_filter_static STATIC ${SOURCES})
set_target_properties(
  blosc_filter_static PROPERTIES OUTPUT_NAME blosc_filter)
target_link_libraries(blosc_filter_static blosc_static ${HDF5_LIBRARIES})

if(BUILD_PLUGIN)
    add_library(blosc_plugin_static STATIC ${PLUGIN_SOURCES})
    set_target_properties(
      blosc_plugin_static PROPERTIES OUTPUT_NAME H5Zblosc)
    target_link_libraries(blosc_plugin_static blosc_static ${HDF5_LIBRARIES})

    install(TARGETS blosc_plugin_static DESTINATION ${PLUGIN_INSTALL_PATH} COMPONENT HDF5_FILTER_DEV)
endif(BUILD_PLUGIN)

# install
install(FILES src/blosc_filter.h DESTINATION include COMPONENT HDF5_FILTER_DEV)
install(TARGETS blosc_filter_static DESTINATION lib COMPONENT HDF5_FILTER_DEV)


# test
message("LINK LIBRARIES='blosc_filter_static ${HDF5_LIBRARIES}'")
if(BUILD_TESTS)
    enable_testing()
    set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
    find_package(Threads REQUIRED)
    set(LIBS ${LIBS} ${CMAKE_THREAD_LIBS_INIT})
    #add_executable(example src/example.c)
    #target_link_libraries(example blosc_filter_static ${HDF5_LIBRARIES} ${LIBS})
    #add_test(test_hdf5_filter example)
endif(BUILD_TESTS)
