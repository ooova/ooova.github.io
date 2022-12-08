# Tool set for cortex-m dev on Linux from the ground up: how to and what for.

## 1 Step by step quick start guide

First of all, if you are not familiar with how programs are built from source, what the preprocessor, compiler, and linker do, I highly recommend you read [this post](https://allthingsembedded.com/2018/12/29/cross-compiling-for-embedded-devices/).

### 1.1 GNU toolchain

Use instruction from [this pretty good guide](https://askubuntu.com/a/1243405) from the *__askubuntu__* website if it still avainable.

If the above link is not available, here is the short one:

Remove arm-none-eabi toolchain if it was installed before:

`sudo apt remove gcc-arm-none-eabi`

Download latest version of an [arm gnu toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) from the ARM oficial website.

Put the downloaded archive to the dirrectory you want.

Unpack the archive. For example, you can use

`tar -xf <name-of-the-file>`

Create links so that binaries are accessible system-wide:

`sudo ln -s </path/to/dir/with/toolchain/bins/>arm-none-eabi-<tool> /usr/bin/arm-none-eabi-<tool>`

where `<tool>` - one of the gnu utils: `gcc`, `g++`, `gdb`, `size`, `objcopy`.

Check if it works:

`arm-none-eabi-<tool> --version`

If something not works, try to install dependencies. ARM's "full installation instructions" listed in `readme.txt` won't tell you what dependencies are - you have to figure it out by trial and errors.

### 1.2 Build system

I prefer to use GNU CMake and will relay on it further. If you have other preferences, please adapt the further instructions to your favorite build system.

Create a directory for your new awesome project:

`mkdir -p ~/<my-awesome-project>`

Add to you project directory a file named `CMakeLists.txt` and fill it with the following content:

**Attention!<br>All the text placed between `<>` braces you must replace according to your needs/requirements or you cat leave it "as is" if it fits you. If some variables or commands (it would be whole strings) placed between `[]` braces - it is optional but may be necessary for some platforms. I made it intentionally in the source code to force you carefully trace through all the instructions to avoid possible mismatch with your environment.<br>To get more information about specific CMake function refer to the [CMake documentation](https://cmake.org/documentation/)**

``` cmake
cmake_minimum_required(VERSION <3.20>)

set(CMAKE_TOOLCHAIN_FILE ${CMAKE_SOURCE_DIR}/<your-toolchain-cmake-file>.cmake)

project(<your_awesome_project_name> VERSION <1>.<0>)

# If you need set actual firmware version and access to it from
# the source code, you may consider using of the following function
[configure_file(config.h.in ${CMAKE_SOURCE_DIR}/config.h)]

set(CMAKE_C_STANDARD <11>)
set(CMAKE_C_STANDARD_REQUIRED True)
set(CMAKE_CXX_STANDARD <20>)
set(CMAKE_CXX_STANDARD_REQUIRED True)

[set(DEV_FAMILY <GD32L23x>)]
set(CPU <cortex-m23>)
set(ARCH <armv8-m.base>)
[set(HXTAL_VALUE <8000000UL>)]

# Path to the vendor's hardware-specific libraries
set(VENDOR_LIBS
  <${CMAKE_SOURCE_DIR}/GD32L23x_Firmware_Library>
)

# PATHS =====================================================
# CMSIS path ------------------------------------------------
set(CMSIS_PATH
  <${VENDOR_LIBS}/CMSIS>
)
# GD32 SPL path --------------------------------------------
set(SPL_PATH
  <${VENDOR_LIBS}/GD32L23x_standard_peripheral>
)

# LINKER SCRIPT =============================================
set(LINKER_SCRIPT
  <${CMAKE_SOURCE_DIR}/GD32L233Rx.ld>
)
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

# C/C++ SOURCES =============================================
# CMSIS core sources ----------------------------------------
file(GLOB CMSIS_CORE_SRC CONFIGURE_DEPENDS
  <${CMSIS_PATH}/GD/GD32L23x/Source/*.c>
)
# GD32 SPL sources -----------------------------------------
file(GLOB SPL_SRC CONFIGURE_DEPENDS
  <${SPL_PATH}/Source/*.c>
)
# project sources -------------------------------------------
file(GLOB PRJ_SRC CONFIGURE_DEPENDS
  <${CMAKE_SOURCE_DIR}/*.c>
  <${CMAKE_SOURCE_DIR}/*.cc>
  <${CMAKE_SOURCE_DIR}/src/*.c>
  <${CMAKE_SOURCE_DIR}/src/*.cc>
  )
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

# INCLUDES ==================================================
# CMSIS core includes ---------------------------------------
set(CMSIS_CORE_INC
  <${CMSIS_PATH}/Include/>
  )
# CMSIS device includes -------------------------------------
set(CMSIS_DEV_INC
  <${CMSIS_PATH}/GD/GD32L23x/Include/>
  )
# GD32 SPL includes ----------------------------------------
set(SPL_INC
  <${SPL_PATH}/Include/>
  )
# project includes ------------------------------------------
set(PRJ_INC
  <${CMAKE_SOURCE_DIR}/>
  <${CMAKE_SOURCE_DIR}/src/>
  )
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

add_definitions(
  [-D${DEV_FAMILY}]
  [-DHXTAL_VALUE=${HXTAL_VALUE}]
  )

add_executable(${PROJECT_NAME}.elf
  ${CMSIS_CORE_SRC}
  ${SPL_SRC}
  ${PRJ_SRC}
  )

include_directories(
  ${CMSIS_CORE_INC}
  ${CMSIS_DEV_INC}
  ${SPL_INC}
  ${PRJ_INC}
)

set(CMAKE_C_FLAGS_DEBUG     "-O0 -g" CACHE INTERNAL "")
set(CMAKE_C_FLAGS_RELEASE   "-Os")
set(CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_C_FLAGS_DEBUG}" CACHE INTERNAL "")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")

target_compile_options(${PROJECT_NAME}.elf PRIVATE
  -Wno-psabi
  -fdata-sections
  -ffunction-sections
  -Wl,--gc-sections
  -ffreestanding
  [-Wall]
  [-Wextra]
  [-Wpedantic]
  -mcpu=${CPU}
  -march=${ARCH}
  -mlittle-endian
  -mthumb
  -masm-syntax-unified
  -fno-exceptions
  -fno-unwind-tables
)

set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -fno-threadsafe-statics -fno-rtti" CACHE INTERNAL "")

target_link_options(${PROJECT_NAME}.elf PRIVATE
  -T${LINKER_SCRIPT}
  -Wl,-Map=${PROJECT_NAME}.map,--cref
  -mthumb
  -mcpu=${CPU}
  -march=${ARCH}
  -specs=nosys.specs
  -specs=nano.specs
  -lc
  -lm
  -static
  -lnosys
  -Wl,--gc-sections
  -Wl,--print-memory-usage
)

add_custom_command(TARGET ${PROJECT_NAME}.elf
  POST_BUILD
  COMMAND ${CMAKE_OBJCOPY} -O ihex ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
  COMMAND ${CMAKE_OBJCOPY} -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin
  COMMAND ${CMAKE_OBJDUMP} -S ${PROJECT_NAME}.elf > ${PROJECT_NAME}.lss
  COMMAND ${CMAKE_SIZE_UTIL} -B ${PROJECT_NAME}.elf
  COMMENT "Generating ${PROJECT_NAME}.hex, ${PROJECT_NAME}.bin"
)

```
The instruction
``` cmake
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_SOURCE_DIR}/<your-toolchain-cmake-file>.cmake)
```
in the code above imply the using of a cmake toolchain file. That file might looks as follow:
``` cmake
set(CMAKE_SYSTEM_NAME               Generic)
set(CMAKE_SYSTEM_PROCESSOR          arm)

if(MINGW OR CYGWIN OR WIN32)
  set(UTIL_SEARCH_CMD where)
elseif(UNIX OR APPLE)
  set(UTIL_SEARCH_CMD which)
endif()

set(TOOLCHAIN_PREFIX arm-none-eabi-)

execute_process(
  COMMAND ${UTIL_SEARCH_CMD} ${TOOLCHAIN_PREFIX}gcc
  OUTPUT_VARIABLE BINUTILS_PATH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

get_filename_component(ARM_TOOLCHAIN_DIR ${BINUTILS_PATH} DIRECTORY)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_COMPILER ${TOOLCHAIN_PREFIX}gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
set(CMAKE_ASM_COMPILER ${CMAKE_CXX_COMPILER})

set(CMAKE_OBJCOPY ${ARM_TOOLCHAIN_DIR}/${TOOLCHAIN_PREFIX}objcopy CACHE INTERNAL "objcopy tool")
set(CMAKE_OBJDUMP ${ARM_TOOLCHAIN_DIR}/${TOOLCHAIN_PREFIX}objdump CACHE INTERNAL "objcopy tool")
set(CMAKE_SIZE_UTIL ${ARM_TOOLCHAIN_DIR}/${TOOLCHAIN_PREFIX}size CACHE INTERNAL "size tool")

set(CMAKE_FIND_ROOT_PATH ${BINUTILS_PATH})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```
The above toolchain configuration example is quite universal and I suppose it can be used without changes on variety of systems.

### 1.3 Sources

To build your firmware you need three type of source files:
  1. hardware-specific libraries
  2. project source files
  3. startup source file and linker script

Let's inspect them in turn.

#### Hardware-specific libraries

These libraries usually (or almost ever) distributed in sources and includes following
  1. **CMSIS** (Microcontroller Software Interface Standard) library provided by ARM company and contains all the functions, types and definitions related to the core of the MCU
  2. **Vendor-specific** (MCU-specific) libraries, that contains the abstractions that specific for a particular MCU. It could be initialization functions for interfaces (UART, I2C etc.) and other peripherals (timers, ADC etc.), some pre-defined functions to config clocking, and so on.

Usually your can find them on the vendor's website. For example, 'ST microelectronics' provide the HAL and LL libraries (previously they provided the SPL); GigaDevice provide their own 'Firmware Library' for an each chip family.<br>
But sometimes (particularly in the case of the ST) it is not always obvious how to get the library, because the vendor may distribute it only as a part of the IDE.

#### Project source files

I suppose, there is no needs to comment this paragraph a lot. :)

#### Startup source file and linker script

Some people (including me for recent) has think about the startup file and linker script as a dark magic and won't to deal with it trying to find some "ready to use" examples on github or web.<br>
But [this beautiful post about startup code](https://allthingsembedded.com/post/2019-01-03-arm-cortex-m-startup-code-for-c-and-c/) and [this one about the GNU Linker script](https://allthingsembedded.com/post/2020-04-11-mastering-the-gnu-linker-script/) have given me a good understanding of how they work and made me change the way I develop firmware. So I highly recommend you to read them (if they still available).

The main idea is as follow:<br>
When the source code compiles to the object file, that file is complemented with information about the symbols required and contained within the code. During the compilation, the different types of data have been placed in specific sections of the resulting object files.<br>
Usually the following sections are common in a C program:
- .isr_vector section contains the addresses of every Interrupt Service Routine
- .text section contains the code (the machine language instructions that will be executed by the processor)
- .rodata section contains any data that is marked as read only
- .data section contains initialized global and static variables
- .bss section contains all uninitialized global and static variables

Than the linker resolve missing symbols, perform optimizations such as removing unused code and data. It basically merges all object files into a single executable. The linker can also link other code contained in libraries (static or shared).

To clearly understand how to compose the startup file and linker script that will fit your requirements, please read the posts given above.

### 1.4 Firmware flashing tools and debug

On the linux you have several options to flash your firmware to target and debug it.<br>
1. find appropriate tools on the MCU vendor's website, or on the on-circuit programmer vendor's website (if any are provided).
2. use open source and/or free universal tools.
From my perspective, the second option is more preferable due to it's more flexible, free to use, and usually available for variety of platforms.

[Here is a good post](https://cycling-touring.net/2018/12/flashing-and-debugging-stm32-microcontrollers-under-linux) about how to use the OpenOCD and GDB got flash and debug the firmware on microcontrollers.

The great benefit of using OpenOCD is the possibility of using one software tool for different target platforms from different vendors and different in-circuit programmers.

One possible difficulty you may face is that not all of the MCUs or evaluation boards are supported. But fortunately, you can add the missing configuration for the particular target by yourself or try to use the configuration for a similar device. For example, I use the "gd32e23x.cfg" file with the GD32L23x target, and it works just fine.
