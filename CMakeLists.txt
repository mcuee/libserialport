cmake_minimum_required(VERSION 3.12)
project(libserialport C)

# Gather all implementation source files
set(SERIALPORT_SOURCES
    serialport.c
    timing.c
    windows.c
)

# Mandatory Windows OS API libraries
set(WINDOWS_LIBS setupapi cfgmgr32)

# --- 1. SHARED TARGET & EXAMPLE ---
if(NOT DEFINED BUILD_STATIC_ONLY)
    add_library(serialport SHARED ${SERIALPORT_SOURCES})
    target_compile_definitions(serialport PRIVATE 
        LIBSERIALPORT_MSBUILD=1 
        SP_PRIV=
    )
    # Using absolute paths ensures headers are found regardless of sub-directory layouts
    target_include_directories(serialport PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(serialport PRIVATE ${WINDOWS_LIBS})

    # Build shared example
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/list_ports.c")
        add_executable(list_ports_shared examples/list_ports.c)
        target_link_libraries(list_ports_shared PRIVATE serialport)
        target_include_directories(list_ports_shared PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
    endif()
endif()

# --- 2. STATIC TARGET & EXAMPLE ---
if(NOT DEFINED BUILD_SHARED_ONLY)
    add_library(serialport_static STATIC ${SERIALPORT_SOURCES})
    
    # Keeps output binary names uniform ('serialport.lib')
    set_target_properties(serialport_static PROPERTIES OUTPUT_NAME "serialport")
    
    # Strip dllimport macros entirely to destroy C2491 errors on definitions
    target_compile_definitions(serialport_static PRIVATE 
        LIBSERIALPORT_MSBUILD=1 
        SP_PRIV= 
        SP_API=  
    )
    target_include_directories(serialport_static PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(serialport_static PRIVATE ${WINDOWS_LIBS})

    # Build static example
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/list_ports.c")
        add_executable(list_ports_static examples/list_ports.c)
        target_link_libraries(list_ports_static PRIVATE serialport_static)
        target_include_directories(list_ports_static PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
        # Ensure the example compilation unit strips out dllimport tracking as well
        target_compile_definitions(list_ports_static PRIVATE SP_API=)
    endif()
endif()
