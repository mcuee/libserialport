cmake_minimum_required(VERSION 3.12)
project(libserialport C)

# Gather source files
set(SERIALPORT_SOURCES
    serialport.c
    timing.c
    windows.c
)

# Define common system libraries for Windows API operations
set(WINDOWS_LIBS setupapi cfgmgr32)

# --- CONDITIONAL TARGET GENERATION BASED ON BUILD_SHARED_LIBS ---

if(BUILD_SHARED_LIBS)
    # ==========================================
    # SHARED LIBRARY CONFIGURATION
    # ==========================================
    add_library(serialport SHARED ${SERIALPORT_SOURCES})
    target_compile_definitions(serialport PRIVATE 
        LIBSERIALPORT_MSBUILD=1 
        SP_PRIV=
    )
    # PUBLIC allows dependent targets (like examples) to automatically inherit this directory
    target_include_directories(serialport PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(serialport PRIVATE ${WINDOWS_LIBS})

    # Build the example linked against the Shared Library
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/list_ports.c")
        add_executable(list_ports_shared examples/list_ports.c)
        target_link_libraries(list_ports_shared PRIVATE serialport)
    endif()

else()
    # ==========================================
    # STATIC LIBRARY CONFIGURATION
    # ==========================================
    add_library(serialport_static STATIC ${SERIALPORT_SOURCES})
    
    # Force the output file name to be 'serialport.lib' instead of 'serialport_static.lib'
    set_target_properties(serialport_static PROPERTIES OUTPUT_NAME "serialport")
    
    # Strip dllimport macros completely to avoid C2491 errors
    target_compile_definitions(serialport_static PRIVATE 
        LIBSERIALPORT_MSBUILD=1 
        SP_PRIV= 
        SP_API=  
    )
    target_include_directories(serialport_static PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(serialport_static PRIVATE ${WINDOWS_LIBS})

    # Build the example linked against the Static Library
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/examples/list_ports.c")
        add_executable(list_ports_static examples/list_ports.c)
        target_link_libraries(list_ports_static PRIVATE serialport_static)
        # Ensure the example preprocessor also knows SP_API is empty
        target_compile_definitions(list_ports_static PRIVATE SP_API=)
    endif()

endif()
