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
set(LSP_SOURCES 
    serialport.c 
    timing.c
)
set(LSP_LIBS "")

if(WIN32)
    list(APPEND LSP_SOURCES windows.c)
    list(APPEND LSP_LIBS setupapi cfgmgr32 ws2_32)
    
    # Target Windows Vista or newer for modern serial APIs
    add_compile_definitions(_WIN32_WINNT=0x0600)

elseif(APPLE)
    list(APPEND LSP_SOURCES macosx.c)
    find_library(IOKIT_FRAMEWORK IOKit REQUIRED)
    find_library(COREFOUNDATION_FRAMEWORK CoreFoundation REQUIRED)
    list(APPEND LSP_LIBS ${IOKIT_FRAMEWORK} ${COREFOUNDATION_FRAMEWORK})

elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    list(APPEND LSP_SOURCES linux.c linux_termios.c)
    
    # Modern Linux requires libudev for robust hardware enumeration
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(UDEV REQUIRED libudev)
    list(APPEND LSP_LIBS ${UDEV_LIBRARIES})
    include_directories(${UDEV_INCLUDE_DIRS})

elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD|OpenBSD|NetBSD|DragonFly")
    list(APPEND LSP_SOURCES freebsd.c)
else()
    message(FATAL_ERROR "Unsupported OS platform: ${CMAKE_SYSTEM_NAME}")
endif()

# ---------------------------------------------------------------------------
# Target Definition
# ---------------------------------------------------------------------------
# Target name is 'serialport' so output file matches 'libserialport.so/dll'
add_library(serialport ${LSP_SOURCES})
target_link_libraries(serialport PRIVATE ${LSP_LIBS})

# Require strictly standard C99 compliance
target_compile_features(serialport PRIVATE c_std_99)
set_target_properties(serialport PROPERTIES C_EXTENSIONS OFF)

# Handle Target Include Paths (Safe for both building and upstream installation)
target_include_directories(serialport 
    PUBLIC 
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# ---------------------------------------------------------------------------
# Compiler Warnings & Symbol Visibility (The Clean Way)
# ---------------------------------------------------------------------------
# Enforce symbol hiding by default; libraries should explicitly export APIs
set_target_properties(serialport PROPERTIES
    C_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN ON
)

if(MSVC)
    target_compile_options(serialport PRIVATE /W4)
    target_compile_definitions(serialport PRIVATE "SP_API=__declspec(dllexport)" "SP_PRIV=")
else()
    target_compile_options(serialport PRIVATE -Wall -Wextra -Wno-unused-parameter)
    target_compile_definitions(serialport PRIVATE 
        "SP_API=__attribute__((visibility(\"default\")))"
        "SP_PRIV=__attribute__((visibility(\"hidden\")))"
    )
endif()

# ---------------------------------------------------------------------------
# Library Properties & Artifact Naming
# ---------------------------------------------------------------------------
set_target_properties(serialport PROPERTIES
    OUTPUT_NAME "serialport"
    VERSION ${PROJECT_VERSION}
    SOVERSION 0
    PUBLIC_HEADER "libserialport.h"
)

# ---------------------------------------------------------------------------
# Installation & Packaging
# ---------------------------------------------------------------------------
install(TARGETS serialport
    EXPORT libserialport-targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Generate high-quality downstream discovery config files (avoids FindXXXX.cmake)
install(EXPORT libserialport-targets
    FILE libserialportTargets.cmake
    NAMESPACE libserialport::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libserialport
)

# ---------------------------------------------------------------------------
# Examples Layer (Optional)
# ---------------------------------------------------------------------------
if(LIBSERIALPORT_BUILD_EXAMPLES AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples")
    file(GLOB EXAMPLE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/examples/*.c")
    foreach(EXAMPLE_PATH ${EXAMPLE_FILES})
        get_filename_component(EXAMPLE_NAME ${EXAMPLE_PATH} NAME_WE)
        add_executable(${EXAMPLE_NAME} ${EXAMPLE_PATH})
        target_link_libraries(${EXAMPLE_NAME} PRIVATE serialport)
    endforeach()
endif()
