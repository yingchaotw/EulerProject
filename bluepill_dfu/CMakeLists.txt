# The most thoroughly commented embedded CMakeLists

# This call is required at the top of every CMakeLists file, it sets the minimum CMake version our project
# can be built with, and this decides the features that can be used.
# Generally it is advisable to set it as low as possible until features from the newer versions are required.
# Another good approach is to check https://repology.org/project/cmake/versions for the CMake versions shipped
# in most used Linux distributions.
cmake_minimum_required(VERSION 3.13)


# This sets the languages the project is going to be using, if C++ support is desired add CXX to the list.
enable_language(C ASM)

# This sets a custom microcontroller specific compiler flags variable.
# First, the cpu variant is set and then the compiler is told to emit thumb instructions.
# Resources: https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html
set(MCU_OPTIONS
    -mcpu=cortex-m3
    -mthumb
)

# This sets up a variable containing extra functionality compiler flags.
# The flags added here ensure that all objects are placed in separate linker sections. The importance of
# this will be clear further down.
set(EXTRA_OPTIONS
    -fdata-sections
    -ffunction-sections
)

# Set up the compiler optimization options custom variable
set(OPTIMIZATION_OPTIONS
    # Conditionally set compiler optimization level to -Og when building with the Debug configuration.
    # As most microcontrollers are rather limited in terms of flash size and cpu speed, the default
    # optimization level of -O0 is not suitable. -Og is an optimization level created for the best tradeoff
    # between size, speed and debugging viability.
    $<$<CONFIG:Debug>:-Og>
    # Other configurations use their default compiler optimization levels, -O2 for the Release configuration
    # and -Os for the MinSizeRel configuration.
    # Resources:
    # https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#build-configurations
    # https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html
)

# Preprocessor flags for generating dependency information
# Resource: https://gcc.gnu.org/onlinedocs/gcc/Preprocessor-Options.html
set(DEPENDENCY_INFO_OPTIONS
    # Tell the preprocessor to generate dependency files for Make-compatible build systems instead of full
    # preprocessor output, while removing mentions to system header files.
    # If we need the preprocessor output ourselves we can pass the -E argument to the compiler instead.
    -MMD
    # This option instructs the preprocessor to add a phony target for each dependency other than the main
    # file to work around errors the build system gives if you remove header files without updating it.
    -MP
    # This specifies that we want the dependency files to be generated with the same name as the corresponding
    # object file.
    -MF "$(@:%.o=%.d)"
)

# Create a custom variable containing compiler flags for generating debug information
# Resource: https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html
set(DEBUG_INFO_OPTIONS
    # The -g flag along with a number tells the compiler to produce the debugging output and the level of detail
    -g3
    # This configures the debug output format and version, it's best to use dwarf version 2 for best debugger
    # compatibility
    -gdwarf-2
)

# Add the compile options from the previously created variables globally
# It is possible to override these options per-target by using `target_compile_options()`, for instance to apply
# a higher optimization level to third party libraries we are not going to be debugging.
add_compile_options(
    ${MCU_OPTIONS}
    ${EXTRA_OPTIONS}
    ${DEBUG_INFO_OPTIONS}
    ${DEPENDENCY_INFO_OPTIONS}
    ${OPTIMIZATION_OPTIONS}
)

# Global linker options
add_link_options(
    # Pass the MCU options to the linker as well
    ${MCU_OPTIONS}

    # This tells the linker to include the nano variant of the newlib standard library which is optimized
    # for minimal binary size and RAM use. The regular newlib variant is used simply by not passing this.
    --specs=nano.specs
    --specs=nosys.specs

    # Here the linkerscript of the chip is passed. The linkerscript tells the linker where to store
    # the objects in memory. Resources:
    # https://blog.thea.codes/the-most-thoroughly-commented-linker-script/
    # https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_chapter/ld_3.html
    -T${CMAKE_CURRENT_SOURCE_DIR}/STM32F103C8Tx_FLASH.ld

    # This directive instructs the linker to generate a mapfile. Mapfiles contain information about the final
    # layout of the firmware binary and are an invaluable resource for development.
    # Recommended reading: https://interrupt.memfault.com/blog/get-the-most-out-of-the-linker-map-file
    -Wl,-Map=${PROJECT_NAME}.map,--cref

    # This is the linker flag for removing the unused sections from the final binary, it works in conjunction
    # with -fdata-sections and -ffunction-sections compiler flags to remove all unused objects from the final binary.
    -Wl,--gc-sections

    # This is the linker flags this week that are helpful for monitoring a system’s memory usage, it works in conjunction
    # with --print-memory-usage compiler flags to see the actual size breakdown of each of these regions when the binary is linked.
    # Example:
    #  Memory region         Used Size  Region Size  %age Used
    #  RAM:                     7672 B        20 KB     37.46%
    #  FLASH:                  14264 B        64 KB     21.77%
    -Wl,--print-memory-usage

    # It is also possible to use specific linker command line options to turn off one error message whilst leaving the other enabled.
    # The has a LOAD segment with RWX permissions and has a TLS segment with execute permission errors can be turned back into warning messages by 
    # using the --no-error-rwx-segments option or removed entirely from the linker's output by using the --no-warn-rwx-segments option. 
    -Wl,--no-warn-rwx-segments
)

# This call tells the linker to link the standard library components to all the libraries and executables
# added after the call.
# The order of the standard library calls is important as the linker evaluates arguments one by one.
# -lc is the libc containing most standard library features
# -lm is the libm containing math functionality
# -lnosys provides stubs for the syscalls, essentially placeholders for what would be operating system calls
link_libraries("-lc -lm -lnosys")

# Add library subdirectories
# When a subdirectory is added, the CMakeLists file contained in that subdirectory is evaluated, this is usually
# chained for composability. In this case the Drivers CMakeLists creates a static library called Drivers.
add_subdirectory(Drivers)
add_subdirectory(USB_DEVICE)
add_subdirectory(Middlewares)


# In this case we need to include the Core/Inc for the drivers library, as it depends on the definitions inside
# the Core headers.
target_include_directories(Drivers  PUBLIC
    Core/Inc
)


# Create a custom executable filename variable by appending the ELF extension to the project name
set(EXECUTABLE ${PROJECT_NAME}.elf)

# Create the executable target and adds top-level sources to it.
# An executable is the final product of the build process, in this case our firmware.
# Note the startup file gets added along with the C sources.
add_executable(${EXECUTABLE}
    Core/Src/main.c
    Core/Src/gpio.c
    Core/Src/stm32f1xx_hal_msp.c
    Core/Src/stm32f1xx_it.c
    Core/Src/system_stm32f1xx.c
    startup_stm32f103xb.s
)


# Here we link the libraries to the executable. Same principle applies to the reasoning behind PRIVATE.
target_link_libraries(${EXECUTABLE}  PRIVATE
    Drivers
    USB_DEVICE
    Middlewares
)


# Add the include directories for the executable. Since we don't need the includes to propagate up as our
# executable is the final product of the build process, we are including as PRIVATE.
target_include_directories(${EXECUTABLE}  PRIVATE
    Core/Inc
)

# Add additional warning options for the executable target, enabling them for our sources, but not third
# party libraries.
# All of these provide much more safety if not ignored.
# Resources: https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html
target_compile_options(${EXECUTABLE}    PRIVATE
    -Wall
    -Wextra
    -Wshadow
    -Wconversion
    -Wdouble-promotion
)


# This creates a custom command that prints out the firmware binary size information.
# Text is the code, data stores variables that have a non-zero initial value and have to be stored in flash,
# bss stores zero initial values that only take up ram.
# dec and hex are just the cumulative size in decimal and hexadecimal notation respectively.
# Example:
#     text	   data	    bss	    dec	    hex	filename
#    14036	    228	   7448	  21712	   54d0	STM32-DFU.elf
# Resources: https://mcuoneclipse.com/2013/04/14/text-data-and-bss-code-and-data-size-explained/
add_custom_command(TARGET ${EXECUTABLE}
    POST_BUILD
    COMMAND ${CMAKE_SIZE_UTIL} ${EXECUTABLE}
)

# Create a custom command to generate binary and hex files.
# These can be used depending on which method of loading the firmware to the MCU is used.
add_custom_command(TARGET ${EXECUTABLE}
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex ${EXECUTABLE} ${PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary ${EXECUTABLE} ${PROJECT_NAME}.bin
)