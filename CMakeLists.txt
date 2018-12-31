cmake_minimum_required(VERSION 2.8.3)
cmake_policy(SET CMP0048 NEW)
project(serial VERSION "1.2.2" LANGUAGES CXX)
include(GNUInstallDirs)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_CXX_STANDARD 11)

if(WIN32)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
    if(MSVC)
        set(CMAKE_DEBUG_POSTFIX "d")
    endif()
endif()
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
if(APPLE)
    set(CMAKE_MACOSX_RPATH ON)
endif()

option(USE_CATKIN "by default catkin search is turn Off, if required set USE_CATKIN ON" OFF)
option(BUILD_SHARED_LIBS "default build shared libs, if needed turn this off" ON)
option(BUILD_TESTING "Testing is disabled unless choosen specifically" OFF)

set(THREADS_PREFER_PTHREAD_FLAG ON)
if(CMAKE_BUILD_TYPE STREQUAL Debug)
    message (STATUS "compiling with debug mode on")
    SET(BUILD_TESTING OFF)
    set(CMAKE_BUILD_TYPE Debug)
elseif(CMAKE_BUILD_TYPE STREQUAL Release)
    message (STATUS "compiling with release mode on")
endif()


find_package(Threads REQUIRED)
if (UNIX AND NOT APPLE)
    find_library(rt)
endif()
    # Find catkin
if (USE_CATKIN)
    find_package(catkin REQUIRED)
endif()

if(APPLE)
    find_library(IOKIT_LIBRARY IOKit)
    find_library(FOUNDATION_LIBRARY Foundation)
endif()

if(UNIX AND NOT APPLE)
    if (USE_CATKIN)
        catkin_package(
            LIBRARIES ${PROJECT_NAME}
            INCLUDE_DIRS include
            DEPENDS rt pthread
        )
    endif()
else()
    # Otherwise normal call
    if (USE_CATKIN)
        catkin_package(
            LIBRARIES ${PROJECT_NAME}
            INCLUDE_DIRS include
        )
    endif()
endif()

## Sources
set(serial_SRCS
    src/serial.cc
    include/serial/serial.h
    include/serial/v8stdint.h
)
if(APPLE)
    # If OSX
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_osx.cc)
elseif(UNIX)
    # If unix
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_linux.cc)
else()
    # If windows
    list(APPEND serial_SRCS src/impl/win.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_win.cc)
endif()

## Add serial library
add_library(${PROJECT_NAME} ${serial_SRCS})
if(APPLE)
    target_link_libraries(${PROJECT_NAME} ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY})
elseif(UNIX)
    target_link_libraries(${PROJECT_NAME} rt ${CMAKE_THREAD_LIBS_INIT})
else()
    target_link_libraries(${PROJECT_NAME} setupapi)
endif()
target_include_directories(${PROJECT_NAME} PUBLIC 
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include/serial>
)
target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_11)
set_target_properties(${PROJECT_NAME} PROPERTIES
    IMPORTED_LOCATION ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
    $<$<PLATFORM_ID:Darwin>:FRAMEWORK TRUE>
    $<$<PLATFORM_ID:Darwin>:FRAMEWORK_VERSION ${PROJECT_VERSION}>
    $<$<PLATFORM_ID:Darwin>:MACOSX_FRAMEWORK_IDENTIFIER org.cmake.${LIBRARY_TARGET_NAME}>
)

add_subdirectory(examples)

## Install executable
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Install headers
install(FILES include/serial/serial.h include/serial/v8stdint.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/serial)

## Tests
if(CATKIN_ENABLE_TESTING OR BUILD_TESTING)
    add_subdirectory(tests)
endif()
