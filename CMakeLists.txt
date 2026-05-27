cmake_minimum_required(VERSION 3.15...3.26)

project(libserialport
    VERSION 0.1.1
    DESCRIPTION "Clean CMake build for sigrok libserialport"
    HOMEPAGE_URL "https://github.com/sigrokproject/libserialport"
    LANGUAGES C
)

include(GNUInstallDirs)

# ---------------------------------------------------------------------------
# Build Options
# ---------------------------------------------------------------------------
option(BUILD_SHARED_LIBS "Build shared libraries instead of static" ON)
option(LIBSERIALPORT_BUILD_EXAMPLES "Build example programs" OFF)

# ---------------------------------------------------------------------------
# Source File & Dependency Discovery
# ---------------------------------------------------------------------------
set(LSP_SOURCES serialport.c timing.c)
set(LSP_LIBS "")

if(WIN32)
    list(APPEND LSP_SOURCES windows.c)
    list(APPEND LSP_LIBS setupapi cfgmgr32 ws2_32)
    add_compile_definitions(_WIN32_WINNT=0x0600)
elseif(APPLE)
    list(APPEND LSP_SOURCES macosx.c)
    find_library(IOKIT_FRAMEWORK IOKit REQUIRED)
    find_library(COREFOUNDATION_FRAMEWORK CoreFoundation REQUIRED)
    list(APPEND LSP_LIBS ${IOKIT_FRAMEWORK} ${COREFOUNDATION_FRAMEWORK})
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    list(APPEND LSP_SOURCES linux.c linux_termios.c)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(UDEV REQUIRED libudev)
    list(APPEND LSP_LIBS ${UDEV_LIBRARIES})
    include_directories(${UDEV_INCLUDE_DIRS})
elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD|OpenBSD|NetBSD|DragonFly")
    list(APPEND LSP_SOURCES freebsd.c)
endif()

# ---------------------------------------------------------------------------
# Target Definition
# ---------------------------------------------------------------------------
add_library(serialport ${LSP_SOURCES})
target_link_libraries(serialport PRIVATE ${LSP_LIBS})

target_compile_features(serialport PRIVATE c_std_99)
set_target_properties(serialport PROPERTIES C_EXTENSIONS OFF)

target_include_directories(serialport 
    PUBLIC 
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# ---------------------------------------------------------------------------
# Compiler Warnings & Symbol Visibility
# ---------------------------------------------------------------------------
target_compile_definitions(serialport PRIVATE LIBSERIALPORT_EXPORTING)

if(BUILD_SHARED_LIBS)
    if(MSVC)
        target_compile_definitions(serialport PRIVATE BUILDING_LIBSERIALPORT)
    else()
        # GCC/Clang: Use attributes for symbol management
        target_compile_definitions(serialport PRIVATE 
            "SP_API=__attribute__((visibility(\"default\")))"
            "SP_PRIV=__attribute__((visibility(\"hidden\")))"
        )
    endif()
else()
    # Static build: clear macros to avoid dllimport/dllexport logic
    target_compile_definitions(serialport PUBLIC "SP_API=" "SP_PRIV=")
endif()

# ---------------------------------------------------------------------------
# Library Properties
# ---------------------------------------------------------------------------
set_target_properties(serialport PROPERTIES
    OUTPUT_NAME "serialport"
    VERSION ${PROJECT_VERSION}
    SOVERSION 0
    PUBLIC_HEADER "libserialport.h"
)

install(TARGETS serialport
    EXPORT libserialport-targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# ---------------------------------------------------------------------------
# Examples Layer
# ---------------------------------------------------------------------------
if(LIBSERIALPORT_BUILD_EXAMPLES AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples")
    file(GLOB EXAMPLE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/examples/*.c")
    foreach(EXAMPLE_PATH ${EXAMPLE_FILES})
        get_filename_component(EXAMPLE_NAME ${EXAMPLE_PATH} NAME_WE)
        add_executable(${EXAMPLE_NAME} ${EXAMPLE_PATH})
        target_link_libraries(${EXAMPLE_NAME} PRIVATE serialport)
    endforeach()
endif()
