# Embedded Environment


Have you ever wondered what is going on behind the mystical IDE of your embedded choice?  

Many programming journeys start with learning user interface of a particular Development Environment -- a path I took myself. While IDEs can arguably simplify things, various knobs get defaulted to some magic values. Unless you know upfront, it can be tricky to lookup what these radio buttons actually do.

There is a plethora of bare-metal oriented [*Integrated Development Environments*](https://en.wikipedia.org/wiki/Integrated_development_environment) (IDEs) in the wild sharing a similar set of features. These include Keil, Atollic, IAR, mbed, a multitude of eclipse flavors, and well-known Arduino. 

This post/tutorial shows a method of developing embedded code without having a particular IDE in mind, a bare approach for bare metal development, underlining why IDE agnostic is elegant, efficient and simple.

## 1. Requirements
This post will focus on embedded ARM toolchain, particularly on the guidelines for the famous [*blue pill*](https://hackaday.com/2017/03/30/the-2-32-bit-arduino-with-debugging/) (aka [*stm32f103c8t6*](https://www.st.com/en/microcontrollers-microprocessors/stm32f103c8.html)). The process is almost identical for other stm32 microcontrollers and simillar for embeded arm devices from other vendors (Atmel, NXP, etc).

Further instructions are written for Unix platforms, that is Linux and MacOS.

Our dependencies are
 - [*gcc-arm-none-eabi*](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm) -- GNU ARM Embedded Toolchain.
 - [*git*](https://git-scm.com) -- Version Control.
 - [*CMake*](https://cmake.org) -- Build System.

### 1.1. gcc-arm-none-eabi
Compiler, linker, debugger, objdump, you name it. Install using a package manager or from the [*arm website*](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm).  

#### Ubuntu  
```shell
sudo apt install gcc-arm-none-eabi
```
#### MacOS  
```shell
brew install --cask gcc-arm-embedded
```

### 1.2. git
We will use git as a version control system and to pull some libraries into our project. Odds are you have git installed already, if unsure, try `git --version`. In case you don't have git, follow installation instructions [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git).

### 1.3. CMake
CMake will serve us as a build system, it will generate tedious [*make files*](https://www.gnu.org/software/make/manual/make.html) telling the compiler where to look for the source code and passing flags. CMake is the most widely spread build system, many IDEs rely on it. In fact, you don't have to be familiar with CMake to follow this tutorial.

Download CMake from [*here*](https://cmake.org/download/) or install with a package manager.

#### Ubuntu
```shell
sudo apt install cmake
```

#### MacOS
```shell
brew install cmake
```

## 2. Project Structure
It all begins with creating a project directory, which will further be referenced as *project root directory*.

```shell
mkdir blue-pill
cd blue-pill
```

Similarly to any software project, ours will have some code we write ourselves and some borrowed library code that will not change, except for when the libraries get updated. cmake directory will contain our build system.

```shell
mkdir sources
mkdir libraries
mkdir cmake
```

This is different from typical unix development, where your 3d party libraries are likely to live in `/usr/local`. Since we will be writing code for a different architecture we do not "install" libraries/headers into our system.

## 3. Libraries

### 3.1. CMSIS

First, we are going to add [*Cortex Microcontroller Software Interface*](https://arm-software.github.io/CMSIS_5/Core/html/index.html) or CMSIS. There are traits all Cortex-M microcontrollers share. CMSIS serves as a standard, developed by ARM itself, to interface things like *Nested Vector Interrupt Table* (NVIC), Exception Handlers (Reset, Hard Fault, SysTick, Supervisor Call), Memory protection, Floating Point Unit, etc.. [*Here*](https://interrupt.memfault.com/blog/arm-cortex-m-exceptions-and-nvic) is a good article about exceptions and NVIC.

[*Here*](https://github.com/ARM-software/CMSIS_5) is CMSIS github repository. In fact, all we will need in terms of the library support is available on git, therefore we can take use [*git submodule*](https://git-scm.com/book/en/v2/Git-Tools-Submodules) to link these repositories to our project.

```shell
# From the project root.
git init
git submodule add https://github.com/ARM-software/CMSIS_5 libraries/CMSIS_5
cd libraries/CMSIS_5
git checkout master # The default branch is develop.
cd ../.. # Back to the project root.
```

Though cloning the entire CMSIS repo might be bulky, we only need a few headers. Imagine somebody else wants to build your project, fishing for headers can be extremely unpleasant. Another solution -- some IDEs copy everything (cmsis and hal) into the project directory by default, which makes version control ugly, and does not allow to easily update libraries.  

We will also need the device specific CMSIS. This defines registers of the available peripherals.

```shell
# From the project root.
git submodule add https://github.com/STMicroelectronics/cmsis_device_f1.git libraries/cmsis_device_f1
```

### 3.2. HAL

The next step is Hardware Abstraction Layer of the microcontroller peripherals. The blue pill has STM32F103C8T6 microcontroller, which brings us to ST's [*github page*](https://github.com/STMicroelectronics) where we find [*stm32f1xx_hal_driver*](stm32f1xx_hal_driver).

```shell
git submodule add https://github.com/STMicroelectronics/stm32f1xx_hal_driver.git libraries/stm32f1xx_hal_driver
cd libraries/stm32f1xx_hal_driver
```

Let's be adventurous and screw a virtual com port to our project, because why not? The pill has usb support.

```shell
git submodule add https://github.com/STMicroelectronics/stm32_mw_usb_device.git libraries/stm32_mw_usb_device
cd libraries/stm32_mw_usb_device
```

[*Virtual Com Port*](https://en.wikipedia.org/wiki/Virtual_COM_port) can be accessed from your computer the same way a generic serial port would, eliminating the need of usb to TTL converter, that is when you plug in your blue pill into a unix computer you can talk through `/dev/tty.something`.

## 4. Sources

Usually microcontroller vendors provide some template code to start off. Here I will use [*STM32CubeMX*](https://www.st.com/en/development-tools/stm32cubemx.html) code generation tool. If you find using stm cube every time to generate initialization code painful, you need java to run it after all, make a template for a specific board and reuse it. For instance, the repository of this tutorial serves me as a template for the blue pill. Here is a checklist for stm cube.

1. Access mcu selector -> look for stm32f103c8.
2. Pinout & Configuration:
   - System Core -> RCC -> HSE = Crystal Resonator
   - System Core -> SYS -> Debug = Serial Wire # I am using swd, you could use jtag.
   - Connectivity -> USB -> Check Device
   - Middleware -> USB_DEVICE -> Class = Communication device class (virtual com port)
   - Optionally you could set PC13 to GPIO_Output to toggle the on-board led by clicking on the PC13 pin.
3. Clock Configuration. To me this is the most appreciated part of stm cube, you get to see and play with clock settings without analyzing tens of data sheet pages. Blue pill has 8 MHz external oscillator, make sure you check that
   - Input frequency = 8MHz
   - PLL source mux is set to HSE
   - System clock mux is PLL clock
   Play with different knobs, if you do something wrong cube will let you know. For instance usb frequency must always be set to 48MHz, if it's different cube will raise the red flag. Though the maximum system frequency you can get is 72MHz, you can later modify it in the code (by accident), there is nothing that will stop you. Be warned, overclocking might be deadly.
4. Project Manager.
   - Project -> Toolchain -> SW4STM32
   - Code Generator -> Check add necessary files as reference (this will prevent stm cube from copying all hal libraries into the project directory).
   - Project -> Project Name and Location: **do not** set our project root directory as the stm cube project location. When you open stm cube again in future, it can rewrite all the generated files, reorganizing and even deleting your changes. We should generate stm cube code to some temporary folder, then transfer the files manually. Often you would want to go back to stm cube and tweak the configuration.
5. Generate Code.

A side note on stm cube. The "firmware packages" that stm cube downloads to generate code contain lots of examples of peripherals initialization and usage. You can find these "firmware packages" at `~/STM32Cube/Repository/` or at [*ST's github*](https://github.com/STMicroelectronics). [*Here*](https://github.com/STMicroelectronics/STM32CubeF1) is one for STM32F1xx family.  

Navigate to the folder where stm cube generated initialization code and copy `Inc` and `Src` folders to our `sources` directory.

```shell
# Beware of your paths.
cp -r Inc ~/projects/blue-pill/sources/inc
cp -r Src ~/projects/blue-pill/sources/src
# ps: I prefer lower case inc and src
```

You might ask what about `startup/`? We already have a set of standard startup scripts in the `libraries/stm32f1xx_hal_driver`, which we will link later.

## 5. CMake

At this point we have summoned all sources in one place and are up to building a binary. Though we are building a hello world there is already a lot going on. The moment you supply power, processor exits reset state by setting the program counter to the predefined reset handler address -- and here your code begins, or not quite yours, for this tutorial we will use startup code provided by st, but you are in full control.

Back to cmake. Even though, cmake is much more human readable than make, we still have to deal with lots of tedious flags, which vary based on the microcontroller family. For instance, once I used to hardcode compiler flags into a line looking similar to this.

```cmake
# Not mentioning FPU flags..
set(COMMON_FLAGS "-mcpu=cortex-m3 ${FPU_FLAGS} -mthumb -mthumb-interwork -ffunction-sections -fdata-sections -g -fno-common -fmessage-length=0 -specs=nosys.specs -specs=nano.specs -Os")
```

Let's say one day you would like to switch to another microcontroller, now you have to update your build configuration, which as you see can be a bit tedious. Here I would like to introduce "yet another collection of cmake scripts" or [*yaccs*](https://github.com/nicocvn/yaccs), as inferred from its name yaccs contains some common build configurations and it supports embedded arm. 

```shell
# From project root.
git submodule add https://github.com/nicocvn/yaccs.git cmake/yaccs
```

Create a `CMakeLists.txt` in the project root with the following content. Credit goes to [*yaccs readme example*](https://github.com/nicocvn/yaccs/blob/master/docs/Example.md).
```cmake
# file: CMakeLists.txt

# This is required with modern version of CMake.
cmake_minimum_required(VERSION 3.10)

# Define the project.
project(blue-pill ASM C CXX)
add_executable(${PROJECT_NAME})

# Linker Script
set(LINKER_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/libraries/cmsis_device_f1/Source/Templates/gcc/linker/STM32F101XB_FLASH.ld)
set_target_properties(${PROJECT_NAME} 
                        PROPERTIES 
                        SUFFIX ".elf"
                        LINK_OPTIONS "-Wl,-gc-sections,--print-memory-usage,-Map=${PROJECT_NAME}.map,-T${LINKER_SCRIPT}")

# HAL libraries require to define target microcontroller.
# See stm32f1xx.h for more details.
target_compile_definitions(${PROJECT_NAME} PUBLIC -DSTM32F103xB -DUSE_HAL_DRIVER)

# Include yaccs main CMake file.
# This will automatically look and load the user-config file.
# This has to be done before calling the project() command!
include(cmake/yaccs/yaccs.cmake)

# This is optional but useful to get a sense of what is happening.
# This command will print various information about the build configuration.
yaccs_system_info()

# This is also optional but it helps by neatly organizing the build tree.
yaccs_init_build_tree()

# Add stuff from the sources directory.
add_subdirectory(sources/)
add_subdirectory(libraries/)
```

Create `yaccs-user-config.cmake` in the project root, this will tell yaccs compiler location and which configuration to use.
```cmake
# file: yaccs-user-config.cmake

# If the compiler is not in the PATH we need to tell yaccs where to find it.
set(yaccs_compiler_paths /Applications/ARM/bin/)

# Load the configuration.
include(cmake/yaccs/cortex-m_gcc-arm_m3_cxx14.cmake)
```

Tell cmake which sources to compile. With `add_subdirectory` statements, cmake looks for `CMakeLists.txt` in the specified directory, therefore we need to create one for `sources` and `libraries`.  

Create `sources/CMakeLists.txt` containing the following.
```cmake
# file: sources/CMakelists.txt

# Add inc/
target_include_directories(${PROJECT_NAME} PUBLIC inc/)

# Add src/
target_sources(${PROJECT_NAME} PUBLIC 
                ${CMAKE_CURRENT_SOURCE_DIR}/src/main.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/stm32f1xx_it.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/usb_device.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/usbd_cdc_if.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/usbd_conf.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/usbd_desc.c)
```
As you may notice we skipped `syscalls.c` generated by stmcube, this is because we are using a smaller version of the standard library, more on this [*here*](https://electronics.stackexchange.com/questions/459975/role-of-syscalls-c)

Create `libraries/CMakeLists.txt`.
```cmake
# file libraries/CMakelists.txt

# List header locations. 
target_include_directories(${PROJECT_NAME} PUBLIC
                            CMSIS_5/CMSIS/Core/Include
                            cmsis_device_f1/Include
                            stm32f1xx_hal_driver/Inc
                            stm32_mw_usb_device/Core/Inc
                            stm32_mw_usb_device/Class/CDC/Inc)

# Add startup code. For some reason there is no exact file that matches startup_stm32f103x8.s, very confusing, ST!
# A glimpse at the stm cube's generated code reveals startup_stm32f103xb.s file.
target_sources(${PROJECT_NAME} PUBLIC 
                ${CMAKE_CURRENT_SOURCE_DIR}/cmsis_device_f1/Source/Templates/system_stm32f1xx.c
                ${CMAKE_CURRENT_SOURCE_DIR}/cmsis_device_f1/Source/Templates/gcc/startup_stm32f103xb.s)

# Collect HAL Driver source files.
target_sources(${PROJECT_NAME} PUBLIC 
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal_cortex.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal_gpio.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal_rcc.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal_rcc_ex.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal_pcd.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_hal_pcd_ex.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32f1xx_hal_driver/Src/stm32f1xx_ll_usb.c)

# Collect USB CDC Driver.
target_sources(${PROJECT_NAME} PUBLIC 
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32_mw_usb_device/Core/Src/usbd_ioreq.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32_mw_usb_device/Core/Src/usbd_ctlreq.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32_mw_usb_device/Core/Src/usbd_core.c
                ${CMAKE_CURRENT_SOURCE_DIR}/stm32_mw_usb_device/Class/CDC/Src/usbd_cdc.c)
```

## 6. Build

We are almost there. In case stm fixed [*this*](https://github.com/STMicroelectronics/STM32CubeF1/issues/25) usb related bug, everything should compile. Make sure in function declarations [*USBD_LL_Transmit*](https://github.com/stansotn/blue-pill/blob/f37e6c5b6e390598f519f3b96e0d3aedd05a06a3/sources/src/usbd_conf.c#L533) and [*USBD_LL_PrepareReceive*](https://github.com/stansotn/blue-pill/blob/f37e6c5b6e390598f519f3b96e0d3aedd05a06a3/sources/src/usbd_conf.c#L553) the last function argument is `uint32_t size` and not `uint16_t size`.

Finally comes the time to build the project.
```shell
mkdir build
cd build
cmake ..
make
```

If you need to extract a binary blob (`.bin` or `.hex`), add following the `CMakeLists.txt` in the project root.
```cmake
# Extract binary blob
set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}> ${HEX_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}> ${BIN_FILE}
        COMMENT "Extracting ${HEX_FILE}\nExtracting ${BIN_FILE}")
```

## Afterword

The final blue-pill template created in this tutorial is available [*here*](https://github.com/stansotn/blue-pill).

A sequel on Embedded Debugging is available [*here*](https://stansotn.com/embedded_debugging/).

> **Help Me Improve**  
> I am learning to write meaningful documentation. I hope you enjoyed this post, please help me back by emailing some feedback!
> - Is information clear, correct and up to date?
> - How would you improve this post?
