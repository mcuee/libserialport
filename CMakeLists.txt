cmake_minimum_required(VERSION 3.12)
project(libserialport C)

# --- 1. Source Files Configuration ---
set(SERIALPORT_SOURCES
    serialport.c
    timing.c
)

# Platform-specific backend source files
if(WIN32)
    list(APPEND SERIALPORT_SOURCES windows.c)
    set(WINDOWS_LIBS setupapi cfgmgr32)
elseif(APPLE)
    list(APPEND SERIALPORT_SOURCES darwin.c)
    find_library(FRAMEWORK_IOKIT IOKit)
    find_library(FRAMEWORK_COREFOUNDATION CoreFoundation)
    set(OS_LIBS ${FRAMEWORK_IOKIT} ${FRAMEWORK_COREFOUNDATION})
else()
    # Linux / BSD / POSIX backends
    list(APPEND SERIALPORT_SOURCES linux.c)
endif()

# --- 2. Core Library Target ---
add_library(serialport ${SERIALPORT_SOURCES})

# PUBLIC ensures that any executable linking against the library
# automatically inherits the include path to find 'libserialport.h'
target_include_directories(serialport PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

# Link platform-specific dependencies
if(WIN32)
    target_link_libraries(serialport PRIVATE ${WINDOWS_LIBS})
    # Safe fallback if definitions aren't explicitly passed via workflow command-line flags
    target_compile_definitions(serialport PRIVATE SP_PRIV=)
elseif(APPLE)
    target_link_libraries(serialport PRIVATE ${OS_LIBS})
endif()

# --- 3. Examples Build Configuration ---
if(IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/examples")
    add_executable(list_ports examples/list_ports.c)
    target_link_libraries(list_ports PRIVATE serialport)
endif()
