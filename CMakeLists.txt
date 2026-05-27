cmake_minimum_required(VERSION 3.12)
project(libserialport C)

# Gather source files
set(SERIALPORT_SOURCES
    serialport.c
    timing.c
    windows.c
)

# Define common system libraries for Windows
set(WINDOWS_LIBS setupapi cfgmgr32)

# --- 1. Shared Library Target ---
add_library(serialport SHARED ${SERIALPORT_SOURCES})
target_compile_definitions(serialport PRIVATE 
    LIBSERIALPORT_MSBUILD=1 
    SP_PRIV=
)
target_include_directories(serialport PUBLIC include)
target_link_libraries(serialport PRIVATE ${WINDOWS_LIBS})

# --- 2. Static Library Target ---
add_library(serialport_static STATIC ${SERIALPORT_SOURCES})
# Ensure both produce 'serialport.lib' but in different folders/targets
set_target_properties(serialport_static PROPERTIES OUTPUT_NAME "serialport")
target_compile_definitions(serialport_static PRIVATE 
    LIBSERIALPORT_MSBUILD=1 
    SP_PRIV= 
    SP_API=  # Force empty to prevent dllimport decoration
)
target_include_directories(serialport_static PUBLIC include)
target_link_libraries(serialport_static PRIVATE ${WINDOWS_LIBS})

# --- 3. Example Build Function ---
function(add_libserialport_example EX_NAME SRC)
    # Build example linked against Shared
    add_executable(${EX_NAME}_shared ${SRC})
    target_link_libraries(${EX_NAME}_shared PRIVATE serialport)
    target_include_directories(${EX_NAME}_shared PRIVATE include)
    
    # Build example linked against Static
    add_executable(${EX_NAME}_static ${SRC})
    target_link_libraries(${EX_NAME}_static PRIVATE serialport_static)
    target_include_directories(${EX_NAME}_static PRIVATE include)
endfunction()

# Add examples
# Adjust the path to where your examples reside
add_libserialport_example(list_ports examples/list_ports.c)
