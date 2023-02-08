## Brief

If you need a "blinking LED" example for "Blue Pill" board based on STM32F103C8T6, you are welcome to download [this](https://github.com/ooova/stm32f103c8t6-cmake-blink-example) github repository.<br>
To do so you can execute<br>
`git clone --recurse-submodules https://github.com/ooova/stm32f103c8t6-cmake-blink-example`<br>
command to clone whole repository including all the submodules<br>
**or:**
1. download repository as *.zip from https://github.com/ooova/stm32f103c8t6-cmake-blink-example<br>
2. uncompressed the archive
3. go to uncompressed repo directory
4. execute `git submodule update --init --recursive`

If you need a some more abstract "example"/"step-by-step" guide for any Cortex-M MCU, you are welcome to dive into this post.

>**Attention!<br>All the text placed between `<>` braces must be replaced according to your needs/requirements or you cat leave it "as is" if it fits you. If some variables or commands (it would be whole strings) placed between `[]` braces - it is optional but may be necessary for some platforms. I made it intentionally in the source code to force you carefully trace through all the instructions to avoid possible mismatch with your environment.<br>To get more information about specific CMake function refer to the [CMake documentation](https://cmake.org/documentation/)**

### 1 Prepare all the necessary tools

Download an archive with the arm-none-eabi toolchain [form the ARM website](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) and extract it to any directory. On this webpage you can find 4 versions of the toolchain (at list at the moment of this post writing), but a recommend to use 11.2 - the oldest one because the 12.2 have some problems with newlibc that I do not know how to fix at this moment. For example, I will create for that a separate directory in 'home':
```console
user@machine:~$ mkdir ~/.bin
user@machine:~$ cd ~/.bin
user@machine:~/.bin$ tar -xf /<path>/<to>/<downloaded>/<toolchain>/<archive>.tar.xz
```
Add to the `~/.bashrc` script the `export` command:
```bash
# some other stuff in your .bashrc file
# ...
export PATH=${PATH}:/<path>/<to>/<extracted>/<toolchain>/bin
```
Furthermore, in freshly installed system the `make` and `cmake` tools could be missed.<br>
Try to execute the following commands to check it:
```console
user@machine:~$ make --version
user@machine:~$ cmake --version
```
If you get messages like
```console
Command `make` not found, but can be installed with:
sudo apt install make
```
or even
```console
Command `cmake` not found, but can be installed with:
sudo apt install cmake
```
install them on your computer immediately before something terrible happens, like you can't build your amazing project!
Finally I highly recommend to use a git for version control. If you do not want to use this tool for some reasons, just skip this section, otherwise check if git is install in your machine
```console
user@machine:~$ git --version
```
and if it's not, install it:
```console
user@machine:~$ sudo apt install git
```
To flash the firmware I'm using the [Open On-Chip Debugger (OpenOCD)](https://openocd.org/) tool which is
```console
user@machine:~$ sudo apt install openocd
```

### 2 Getting libraries

In this example I'm using an STM32 microcontroller, so I'm going to download [this](https://github.com/STMicroelectronics/stm32f1xx_hal_driver) HAL library, [this](https://github.com/STMicroelectronics/cmsis_core.git) CMSIS core library, and [this](https://github.com/STMicroelectronics/cmsis_device_f1.git) vendor-specific "extension" for CMSIS library from official ST repo on the github. If you have different MCU, please, find an appropriate library. Often you can find them on vendor's official website.

I'm going go keep all the libraries in the project's directory and to do so I will use git submodule.<br>
Let's create a project directory
```console
user@machine:~$ mkdir -p ~/projects/example
user@machine:~$ cd ~/projects/example
```
If you going to use git (do not forget to config git user.name and user.email), initialize a repo in the project dir, download the libraries as a git submodule, add a simple .gitignore file, and make an initial commit
```console
user@machine:~/projects/example$ git init
user@machine:~/projects/example$ git submodule add https://github.com/STMicroelectronics/stm32f1xx_hal_driver.git
user@machine:~/projects/example$ git submodule add https://github.com/STMicroelectronics/cmsis_device_f1.git
user@machine:~/projects/example$ git submodule add https://github.com/STMicroelectronics/cmsis_core.git
user@machine:~/projects/example$ echo build/ > .gitignore
user@machine:~/projects/example$ git add .gitignore
user@machine:~/projects/example$ git commit -m "<some commit message, for example: Initial commit. Add HAL library as git submodule>"
```
and that's it.
If you don't want to use git, just skip the above step and clone the repos
```console
user@machine:~/projects/example$ git clone https://github.com/STMicroelectronics/stm32f1xx_hal_driver.git
user@machine:~/projects/example$ git clone https://github.com/STMicroelectronics/cmsis_device_f1.git
user@machine:~/projects/example$ git clone https://github.com/STMicroelectronics/cmsis_core.git
```
or simply download them from the web pages mentioned above.
If you are going to use a MCU different from the STM32F1XX series, please find and download appropriate libraries yourself.

### 3 Create a CMake build setup

If you need to set actual firmware version in CMakeLists.txt and access to it from the source code, you may consider using of the following function that will create a "config.h" that you could include in the project' sources.<br>
The "config.h" file will be created using corresponding "config.h.in" file with following content:
```cpp
#define <YOUR_PROJECT_NAME>_VERSION_MAJOR ${<YOUR_PROJECT_NAME>_VERSION_MAJOR}
#define <YOUR_PROJECT_NAME>_VERSION_MINOR ${<YOUR_PROJECT_NAME>_VERSION_MINOR}
```
The CMakeLists.txt for Cortex-M MCU could look like this:
``` cmake
cmake_minimum_required(VERSION <3>.<20>)

set(CMAKE_TOOLCHAIN_FILE ${CMAKE_SOURCE_DIR}/<your-toolchain-file-name>.cmake)

project(<BLINK> VERSION <1>.<0>)
[configure_file(<config-file-name>.h.in ${CMAKE_SOURCE_DIR}/<config-file-name>.h)]

set(CMAKE_C_STANDARD <11>)
# I'm not using the 20th standard here because it deprecates compound assignment
# with 'volatile'-qualified left operand (like `REG |= BIT`), while a lot of HAL's macros use it
set(CMAKE_CXX_STANDARD <17>)

[set(DEV_FAMILY STM32F103xB)]
# Please pay special attention for following two variables if you accidentally miss to change them
# or change to the wrong value it would be extremely hard to realize whi your firmware doesn't work
set(CPU cortex-<m3>)
set(ARCH armv<7><-m>)
[set(HSE_VALUE 8000000UL)]

# PATHS =====================================================
set(LIBS_PATH
  <${CMAKE_SOURCE_DIR}>
  )
set(HAL_PATH
  <${LIBS_PATH}/stm32f1xx_hal_driver>
  )
set(CMSIS_CORE_PATH
  <${LIBS_PATH}/cmsis_core>
  )
set(CMSIS_DEV_PATH
  <${LIBS_PATH}/cmsis_device_f1>
  )
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

# C/C++ SOURCES =============================================
# Here you have two options:
# 1st - use a file() function and collect all the sources of the vendor's library automatically
<file(GLOB VENDOR_LIBS_SRC CONFIGURE_DEPENDS>
<  ${VENDOR_LIBS_PATH}/Src/*.c>
<  )>
# 2nd - use a set() function and enumerate all the needed source files
<set(HAL_LIBS_SRC>
<  ${HAL_PATH}/Src/stm32f1xx_hal.c>
<  ${HAL_PATH}/Src/stm32f1xx_hal_adc.c>
<  ${HAL_PATH}/Src/stm32f1xx_hal_adc_e>x.c>
<  ${HAL_PATH}/Src/stm32f1xx_hal_can.c>
#  ...
<  )>
file(GLOB PRJ_SRC CONFIGURE_DEPENDS
  ${CMAKE_SOURCE_DIR}/*.c
  ${CMAKE_SOURCE_DIR}/*.cc
  ${CMAKE_SOURCE_DIR}/src/*.c
  ${CMAKE_SOURCE_DIR}/src/*.cc
  )
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

# INCLUDE DIRs ==============================================
set(HAL_LIBS_INC
  <${HAL_PATH}/Inc>
  )
set(CMSIS_CORE_LIBS_INC
  <${CMSIS_CORE_PATH}/Include>
  )
set(CMSIS_DEV_LIBS_INC
  <${CMSIS_DEV_PATH}/Include>
  )
set(PRJ_INC
  <${CMAKE_SOURCE_DIR}/>
  <${CMAKE_SOURCE_DIR}/src/>
  )
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

# LINKER SCRIPT =============================================
set(LINKER_SCRIPT
  ${CMAKE_SOURCE_DIR}/<GD32L233Rx>.ld
  )
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

add_definitions(
  <-DHSE_VALUE=${HSE_VALUE}>
  <-D${DEV_FAMILY}>
  )

add_executable(${PROJECT_NAME}.elf
  ${HAL_LIBS_SRC}
  # This file provides two functions SystemInit() and SystemCoreClockUpdate().
  ${CMSIS_DEV_PATH}/<Source>/<Templates>/<system_stm32f1xx>.c
  ${PRJ_SRC}
  )

include_directories(
  ${HAL_LIBS_INC}
  ${CMSIS_CORE_LIBS_INC}
  ${CMSIS_DEV_LIBS_INC}
  ${PRJ_INC}
  )

target_compile_options(${PROJECT_NAME}.elf PRIVATE
  -Wno-psabi
  -fdata-sections
  -ffunction-sections
  -Wl,--gc-sections
  -ffreestanding
  -Wall
  -Wpedantic
  -mcpu=${CPU}
  -march=${ARCH}
  -mlittle-endian
  -mthumb
  -masm-syntax-unified
  -fno-exceptions
  -fno-unwind-tables
  )

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
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

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

set(CMAKE_C_FLAGS_DEBUG     "-O0 -g" CACHE INTERNAL "")
set(CMAKE_C_FLAGS_RELEASE   "-Os")
set(CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_C_FLAGS_DEBUG}" CACHE INTERNAL "")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -fno-threadsafe-statics -fno-rtti" CACHE INTERNAL "")

set(CMAKE_OBJCOPY ${ARM_TOOLCHAIN_DIR}/${TOOLCHAIN_PREFIX}objcopy CACHE INTERNAL "objcopy tool")
set(CMAKE_OBJDUMP ${ARM_TOOLCHAIN_DIR}/${TOOLCHAIN_PREFIX}objdump CACHE INTERNAL "objcopy tool")
set(CMAKE_SIZE_UTIL ${ARM_TOOLCHAIN_DIR}/${TOOLCHAIN_PREFIX}size CACHE INTERNAL "size tool")

set(CMAKE_FIND_ROOT_PATH ${BINUTILS_PATH})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```
The above toolchain configuration example is quite universal and I suppose it can be used without changes on variety of systems.

### 4 Compose a startup code and a linker script

As I already mentioned in [this post](../cortex-m-tool-set-for-linux.md), we have to compose a startup code and a linker script to "explain" to the linker where to save the code and data and where to load it from.<br>
To be fare, in some cases we wouldn't develop thous files by ourself because some of vendors provide them with its CMSIS library (for the STM32F1xx it contained in `cmsis_device_f1/Source/Templates/gcc`). But we will do this to learn the technique.

### 4.1 Linker script (*.ld)

Usually the content of the *.ld files (linker scripts) essentially same for any Cortex-M MCUs except a few first lines
```
/* Entry Point must match to the entry point of startup file */
ENTRY(Reset_Handler)

/* Generate a link error if heap and stack don't fit into RAM */
_Min_Heap_Size = <0>;      /* required amount of heap  */
_Min_Stack_Size = <0x400>; /* required amount of stack */

/* Specify the memory areas */
/* Use this const for MCU's where INTERRUPT_VECTOR_TABLE is located in the RAM
/* INTERRUPT_VECTOR_TABLE_SIZE = 0x150; */
MEMORY
{
  FLASH (rx) : ORIGIN = <0x08000000>, LENGTH = <128K>
  /* use this RAM config equesion for MCU's where INTERRUPT_VECTOR_TABLE is located in the RAM
  /* RAM (rwx)  : ORIGIN = <0x20000000> + INTERRUPT_VECTOR_TABLE_SIZE, LENGTH = <20K> - INTERRUPT_VECTOR_TABLE_SIZE */
  RAM (xrw) : ORIGIN = <0x20000000>, LENGTH = <20K>
}

/* Highest address of the user mode stack */
_estack = ORIGIN(RAM) + LENGTH(RAM);    /* end of RAM */
```
The code in section above depends on the memory map of the specific device and must be edited according to it.<br>
Usually the memory map is provided in a MCU datasheet.

### 4.2 Startup code

The startup file could be written using any language that can be compiled and linked together with the rest of the code. It could be assembler, C or even C++. I prefer to use C++ for this purpose.<br>
If you going to use C++ for making a startup code like in this example, the main concept you need to understand is that startup code purpose is:
1. place all the data (including the interrupt vector handler pointers) in the proper sections of the memory;
2. define the interrupt handler names that will be used in your code and make give to them default "weak" implementations;
3. initialize all the data with default values or by calling the constructors for user-defined types;
4. call the function for hardware initialization (usually that function defines clock sources, its dividers, multipliers and stuff like that) and finally call the function "main" as an entry point of your project. By the way, exactly at this point you could essentially define the name of the entry point and call it for example "hallelujah" indeed of boring "main" :)

To place the code to the specific memory section you have to use `__attribute__` keyword like<br>
`__attribute__((section(".isr_vector")))`<br>
The above line instructs compiler to place the specific variable in the ".isr_vector" section of the object file to give a chance to the linker to place that variable in a proper place if the final binary file.<br>
In the below code snippets I'm going to show most meaningful parts of startup code
```C++
#include <algorithm>
#include <cstdint>

#include "system_stm32f1xx.h"

#define DEFINE_DEFAULT_ISR(name) \
  extern "C" \
  __attribute__((interrupt)) \
  __attribute__((weak)) \
  __attribute__((noreturn)) \
  void name() { \
      while(true); \
  }

/* dummy Exception Handlers */
DEFINE_DEFAULT_ISR(<NMI_Handler>)
DEFINE_DEFAULT_ISR(<HardFault_Handler>)
// ...
// complete the definition list with
// exception handlers specific for your particular MCU

/* external interrupt handlers */
DEFINE_DEFAULT_ISR(WWDG_IRQHandler)
DEFINE_DEFAULT_ISR(PVD_IRQHandler)
DEFINE_DEFAULT_ISR(TAMPER_IRQHandler)
// ...
DEFINE_DEFAULT_ISR(USART1_IRQHandler)
// ...
// complete the definition list with
// interrupt handlers specific for your particular MCU

extern std::uint32_t _estack;//__StackTop;
extern "C" void Reset_Handler();

// settle the interrupt handlers in the specific section of the output binary file
const volatile std::uintptr_t g_pfnVectors[]
__attribute__((section(".isr_vector"))) {
  // Stack Ptr initialization
  reinterpret_cast<std::uintptr_t>(&_estack),
  // Entry point
  reinterpret_cast<std::uintptr_t>(Reset_Handler),
  // Exceptions
  reinterpret_cast<std::uintptr_t>(<NMI_Handler>),              /* Non maskable interrupt. */
  reinterpret_cast<std::uintptr_t>(<HardFault_Handler>),        /* HardFault_Handler */
  // .. if there is no handler by the next address, add a NULL entry
  reinterpret_cast<std::uintptr_t>(<nullptr>),                    /* "Reserved" according to datasheet */
  // ...
  // complete the exception handlers placement

  // External Interrupts
  reinterpret_cast<std::uintptr_t>(<WWDG_IRQHandler>),
  reinterpret_cast<std::uintptr_t>(<PVD_IRQHandler>),
  reinterpret_cast<std::uintptr_t>(<TAMPER_IRQHandler>),
  // ...
  // complete the interrupt handlers placement
};

// This function for STM32 is located in "./cmsis_device_f<x>/Source/Templates/system_stm32f1xx.c"
// It should be different for other MCUs
extern void SystemInit(void);
// This is the firmware entry point
extern int main();

extern "C"
void Reset_Handler() {
  // Initialize data section
  extern std::uint8_t _sdata;
  extern std::uint8_t _edata;
  extern std::uint8_t _sidata;
  std::size_t size = static_cast<std::size_t>(&_edata - &_sdata);
  std::copy(&_sidata, &_sidata + size, &_sdata);

  // Initialize bss section
  extern std::uint8_t __bss_start__;
  extern std::uint8_t __bss_end__;
  std::fill(&__bss_start__, &__bss_end__, UINT8_C(0x00));

  // Initialize static objects by calling their constructors
  typedef void (*function_t)();
  extern function_t __init_array_start;
  extern function_t __init_array_end;
  std::for_each(&__init_array_start, &__init_array_end, [](const function_t pfn) {
      pfn();
  });

  // Do not forget to initialize the clock circuits!!!! ! ! ! !  !   !    !     !
  SystemInit();

  main();
  while(1) {
  }
}
```

## 5 Build the project

Saying specifically about STM32, to configure the HAL library you have to take the stm32f<1>xx_hal_conf_template.h file, rename it to stm32f<1>xx_hal_conf.h and change its content according to your requirements. For other hardware platforms some other additional actions could be required.<br>

The build process is pretty simple:
1. create a `build` directory
2. go to the `build` directory
3. call `cmake` with needed arguments
4. call `make`
```console
user@machine:~/projects/example$ mkdir build
user@machine:~/projects/example$ cd build
user@machine:~/projects/example/build$ cmake [-DCMAKE_BUILD_TUPE=<Debug>|<Release>] ..
user@machine:~/projects/example/build$ make
```

## 6 Flash firmware and debug

To get an original comprehensive quid visit [this beautiful post](https://cycling-touring.net/2018/12/flashing-and-debugging-stm32-microcontrollers-under-linux).<br>
In short terms, all that you need are [OpenOCD](https://openocd.org/) as connection server and `GDB` as a debug server for your target device.<br>
The `GDB` ('arm-none-eabi-gdb') should be already installed if you follow the 1-st paragraph of this post.<br>
How to deal with OpenOCD:
1. Install OpenOCD:<br>
`sudo apt install openocd`
2. Start the OpenOCD with an appropriate interface (basically the hardware programmer) config file as a first parameter and a target config file as a second parameter:<br>
`openocd -f </usr/local/share/openocd/scripts/interface/stlink.cfg> -f </usr/local/share/openocd/scripts/target/stm32f1x.cfg>`
Now you can start `telnet` connection to `openocd` in another console session to be able to upload your firmware into it:<br>
```console
user@machine:~$ telnet localhost 4444
user@machine:~$ reset init
user@machine:~$ halt
user@machine:~$ flash write_image erase </full/path/to/firmware/file.elf>
user@machine:~$ exit
```
To start and connect a `GDB` to OpenOCD do the follow:
```console
user@machine:~$ arm-none-eabi-gdb <~/path/to/firmware/file.elf>
(gdb) target remote localhost:3333
...
(gdb) monitor reset halt
...
(gdb) load
...
(gdb)
```

