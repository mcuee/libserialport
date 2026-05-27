cmake_minimum_required(VERSION 3.12)
project(libserialport C)

# Shared Library Target
add_library(serialport SHARED serialport.c timing.c windows.c)
target_compile_definitions(serialport PRIVATE LIBSERIALPORT_MSBUILD=1 SP_PRIV=)
target_include_directories(serialport PUBLIC include)

# Static Library Target
add_library(serialport_static STATIC serialport.c timing.c windows.c)
set_target_properties(serialport_static PROPERTIES OUTPUT_NAME "serialport")
target_compile_definitions(serialport_static PRIVATE LIBSERIALPORT_MSBUILD=1 SP_PRIV= SP_API=)
target_include_directories(serialport_static PUBLIC include)

# Example Build Logic
# We can define a function to create example targets easily
function(add_libserialport_example EX_NAME SRC)
    # Build example linked against Shared
    add_executable(${EX_NAME}_shared ${SRC})
    target_link_libraries(${EX_NAME}_shared PRIVATE serialport)
    
    # Build example linked against Static
    add_executable(${EX_NAME}_static ${SRC})
    target_link_libraries(${EX_NAME}_static PRIVATE serialport_static)
endfunction()

# Add your examples here
add_libserialport_example(list_ports examples/list_ports.c)
# add_libserialport_example(another_example examples/another.c)
