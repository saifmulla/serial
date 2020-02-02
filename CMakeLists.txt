cmake_minimum_required(VERSION 2.8.3)
cmake_policy(SET CMP0048 NEW)
project(serial VERSION "1.2.2" LANGUAGES CXX)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE ON)
include(GNUInstallDirs)
# include(GenerateExportHeader)

# just to make visual studio or xcode happy
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")

SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_CXX_STANDARD 11)
set(THREADS_PREFER_PTHREAD_FLAG ON)

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

option(USE_CATKIN "by default catkin search is turn Off, if required set USE_CATKIN ON" ON)
option(BUILD_SHARED_LIBS "default build shared libs, if needed turn this off" ON)
option(BUILD_TESTING "Testing is disabled unless choosen specifically" OFF)

if(CMAKE_BUILD_TYPE STREQUAL Debug)
    message (STATUS "compiling with debug mode on")
    SET(BUILD_TESTING ON)
    set(CMAKE_BUILD_TYPE Debug)
elseif(CMAKE_BUILD_TYPE STREQUAL Release)
    message (STATUS "compiling with release mode on")
    set(CMAKE_BUILD_TYPE Release)
endif()

find_package(Threads REQUIRED)
if (UNIX AND NOT APPLE)
    find_library(rt_lib NAMES rt)
    if (rt_lib-NOTFOUND)
        message(FATAL_ERROR "libRt not found on system.")
    endif()
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
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

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
# generate_export_header(${PROJECT_NAME})

# set_target_properties(${PROJECT_NAME} PROPERTIES
# if (CMAKE_SYSTEM_NAME STREQUAL Darwin)
#     FRAMEWORK TRUE
#     FRAMEWORK_VERSION ${PROJECT_VERSION}
#     MACOSX_FRAMEWORK_IDENTIFIER org.cmake.${LIBRARY_TARGET_NAME}
#     SOVERSION ${PROJECT_VERSION}
# endif()
# )
set_property(TARGET ${PROJECT_NAME} PROPERTY VERSION ${PROJECT_VERSION})
set_property(TARGET ${PROJECT_NAME} PROPERTY SOVERSION ${PROJECT_VERSION})
# set_property(TARGET ${PROJECT_VERSION} PROPERTY
#   INTERFACE_serial_MAJOR_VERSION 1)
set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY
  COMPATIBLE_INTERFACE_STRING ${PROJECT_VERSION}
)


add_subdirectory(examples)
## Tests
if(CATKIN_ENABLE_TESTING OR BUILD_TESTING)
    add_subdirectory(tests)
endif()

# export(TARGETS ${PROJECT_NAME} FILE SerialExports.cmake)
## Install executable
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION serial
    # $<$<PLATFORM_ID:Darwin>:FRAMEWORK "${CMAKE_INSTALL_LIBDIR}">
    # FRAMEWORK DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# Install headers
install(FILES include/serial/serial.h include/serial/v8stdint.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/serial)
install(EXPORT ${PROJECT_NAME}Targets DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/serialConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

export(EXPORT serialTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/serialTargets.cmake")

configure_file(cmake/serialConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/serialConfig.cmake"
  COPYONLY
)

install(EXPORT serialTargets 
    FILE 
        serialTargets.cmake
    DESTINATION 
        ${CMAKE_INSTALL_LIBDIR}/cmake/)

install(
    FILES 
        ${CMAKE_CURRENT_BINARY_DIR}/serialConfig.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/serialConfigVersion.cmake
    DESTINATION
        ${CMAKE_INSTALL_LIBDIR}/cmake/
    )

include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/serialCPack.cmake")