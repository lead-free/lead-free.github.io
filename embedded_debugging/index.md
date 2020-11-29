# Embedded Debugging


Continuing the saga on embedded development, this post goes over common ways to debug bare metal ARM, illustrating the technique on [*blue pill*](https://hackaday.com/2017/03/30/the-2-32-bit-arduino-with-debugging/) (aka [*stm32f103c8t6*](https://www.st.com/en/microcontrollers-microprocessors/stm32f103c8.html)) and most common debugger probes.

The debugger itself is a physical piece of hardware that lives inside the processor and can take control of the cpu and memories. When it comes to ARM, there are two protocols available to interface the on chip debugger -- [*JTAG*](https://en.wikipedia.org/wiki/JTAG) and [*Serial Wire Debug*](https://developer.arm.com/architectures/cpu-architecture/debug-visibility-and-trace/coresight-architecture/serial-wire-debug) or swd (more about the differences [*here*](https://electronics.stackexchange.com/questions/53571/jtag-vs-swd-debugging)). In this tutorial we will focus on swd, which our blue pill has a pinout for. Apart from the pin count the two interfaces are very similar.

## 1. Probes
To use the available swd (or jtag) interface you would need a debugger probe and a driver that understands how to talk both to the probe and to the [*gnu debugger*](https://en.wikipedia.org/wiki/GNU_Debugger) instance (further referred as gdb). Let's call this driver connector program a [*gdbserver*](https://en.wikipedia.org/wiki/Gdbserver). This tutorial shines some light on using [*Open On Chip Debugger*](https://github.com/ntfreak/openocd) or openocd as a gdbserver and its alternatives. 

To be clear gdbserver and gdb are different programs:
- gdbserver -- the driver program that employs the debugger probe, communicates to the outer world via tcp.
- gdb -- the client program that loads the executable and interacts with the user (i.e. set breakpoints).

### 1.1 Raspberry Pi
Any raspberry pi on its own can serve as a debugger probe thanks to the [*spi*](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface) driver for openocd developed by [*lupyuen*](https://github.com/lupyuen/openocd-spi). This is amazing because there is no need for external debugger probe, making embedded development more accessible ~~to broke students like myself~~. Chances are you have a raspberry pi collecting dust and you don't have a dedicated debugger hardware (like jlink or stlink). Important to notice, because gdbserver talks to the gdb instance via [*tcp*](https://en.wikipedia.org/wiki/Transmission_Control_Protocol), you can sit back and enjoy the development process (both building and debugging) from an external computer on the network, that is you don't have to develop code on the raspberry pi itself. 

### 1.2 stlink
An increasingly common probe. stlinks are being included with almost every st's official evaluation board, where besides being a debugger they are also used as a usb to serial converter. As you may have noticed, blue pill does not include an stlink, the on board micro usb receptacle is connected directly to the stm32f103's on chip usb peripheral.

An external [*stlink-v3 mini*](https://www.st.com/en/development-tools/stlink-v3mini.html) is a tiny, inexpensive and yet powerful debugger. In addition to swd and jtag, it is capable of usb to spi, i2c, can, and gpio interface via a public c++ api [*stlink-v3-bridge*](https://www.st.com/en/development-tools/stlink-v3-bridge.html).

The main disadvantage of stlink is that there is no straight forward way to run a compatible gdbserver. For some reason STMicroelectronics does not distribute a gdbserver for stlink as an executable binary as of today, even though gdbserver binaries are included with the [*STMCube IDE*](https://www.st.com/en/development-tools/stm32cubeide.html) and with little effort can be scrapped from the IDE package (see 2.2.4). From a conversation with an ST developer, turns out their gdbserver is a modified version of openocd publicly developed [*here*](https://github.com/STMicroelectronics/OpenOCD) and in future might be merged with the original openocd branch.

### 1.3 jlink
A royal plug and play probe with official gdbserver binaries available for a wide variety of architectures including raspberry pi. The best choice if you do not have time to grasp openocd and have funds to get the probe.

## 2. gdbserver

Pick your probe and see what are the options of getting a gdbserver running.

### 2.1. openocd + Raspberry Pi
Openocd must be built from source to enable swd over spi capability on any raspberry pi. A glance at history; openocd capabilities to make use of the raspberry pi's spi port were developed in [*this*](https://github.com/lupyuen/openocd-spi) fork, which to the day of writing this post had not been merged into the [*original opencod repo*](https://github.com/ntfreak/openocd). The author of the spi tweak published [*this*](https://medium.com/@ly.lee/openocd-on-raspberry-pi-better-with-swd-on-spi-7dea9caeb590) article on how to get the hack working with Rust and nrf52, beware this only works on a 32 bit raspbian. With appearance of 64 bit Ubuntu for raspberry pi, a [*pull request*](https://github.com/lupyuen/openocd-spi/pull/2) added aarch64 support. The latter is the particular version of openocd that I tested on raspberry pi, it works both on aarch32 raspbian and aarch64 ubuntu.

```shell
sudo apt install autoconf libtool libusb-1.0-0 libusb-1.0-0-dev
git clone --recursive https://github.com/jeliebig/openocd-spi
cd openocd-spi
./bootstrap
./configure --enable-bcm2835spi
make
```

Feel free to grab some coffee, the build process can take a while. Upon success you will find an openocd binary in the `src/` folder.

Raspbery pi's spi is disabled by default. Enable spi by writing `dtparam=spi=on` to `/boot/config.txt` in Raspbian or `/boot/firmware/usercfg.txt` in Ubuntu. Reboot. Then, enable read write access to spi port.
```shell
sudo chmod 606 /dev/spidev0.0
```

Create an openocd configuration file.
```
# file: swd-pi.ocd
# OpenOCD script for using Raspberry Pi as SWD Programmer for stm32f1x

# Select the Broadcom SPI interface for Raspberry Pi (SWD transport)
interface bcm2835spi

# Set the SPI speed in kHz
bcm2835spi_speed 31200  # 31.2 MHz

# Select stm32f1xx as target
source [find target/stm32f1x.cfg]

# Open gdbserver to the network.
bindto 0.0.0.0
```

Connect the blue pill.

| blue pill | raspi |
|:---:|:---:|
| swdio     | pin 19|
| swdclk    | pin 23|
| gnd       | gnd   |

Power up the blue pill. Despite being very convenient, I would not recommend powering the pill from pi's 3.3V line as if you short something on the pill, raspberry will die. If you supply the power through micro usb or 5V line you will likely to kill the pill's linear voltage regulator in case something gets shorted.

Run openocd. Argument `-s ../tcl/` is a path to the `tlc` folder where openocd looks for configuration files; argument `-f swd-pi.ocd` points to the session specific configuration we just created. 
```shell
# From openocd-spi/src
./openocd -s ~/projects/openocd-spi/tcl/ -f swd-pi.ocd
```

Upon success you shall see an output containing
```
Info : BCM2835 SPI SWD driver
Info : SWD only mode enabled
Info : clock speed 31200 kHz
Info : SWD DPIDR 0x1ba01477
Info : stm32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : Listening on port 3333 for gdb connections
```

In case you see
```
Info : SWD DPIDR 0x0000ffff
```
openocd failed to connect to the microcontroller, check connection.

In case you see the following being endlessly printed
```
spi_transmit failed: Bad file descriptor
```
Then either spi is disabled or openocd does not have permissions to `/dev/spidev0.0`. Odds are you can't ctrl+c yourself out of the error, then you need to find and kill the process from a different window.
```shell
pgrep openocd # Lookup process id.
kill -9 <process id> # Kill openocd process.
```

### 2.2. openocd + stlink

Getting stlink to cooperate is pretty much about getting openocd to work on your system one way or the other. Let's start with a configuration file for stlink and stm32fxx as it will be the same for any host.

```
# file: stlink-swd.ocd

# Select stlink probe
source [find interface/stlink.cfg]

# Select stm32f1xx as target
source [find target/stm32f1x.cfg]

# Select swd 
transport select hla_swd

# Open gdbserver to the network.
bindto 0.0.0.0
```

You may notice that openocd is available in a package manager `apt` or `brew`, but the version you will get is very outdated (`0.1` as of today) and will not support stlink-v3 even if you import a proper interface configuration. Also, depending on your system, `0.1` may crash if the probe is plugged into a usb3 port. Therefore I highly recommend building openocd from source.

#### 2.2.1. Debian aarch32/64

Follow the openocd installation guide for the raspberry pi above, you may skip the spi details if you are not using a raspberry pi or you don't want swd over spi support. Then start openocd with the configuration file for stlink.

#### 2.2.2. Ubuntu

Ubuntu is the most straightforward with building openocd.
```shell
sudo apt install autoconf libtool libusb-1.0-0 libusb-1.0-0-dev # Dependencies.
git clone --recursive https://github.com/ntfreak/openocd.git # Original openocd repo.
cd openocd
./bootstrap
./configure
make -j4
```

Upon success you will find an openocd binary in the `src/` folder.

Grab the configuration script `stlink-swd.ocd` presented above, connect your target to stlink and stlink to the host computer.
```shell
sudo ./openocd -s ../tcl -f stlink-swd.ocd
```
Argument `-s ../tcl` is a path to the `tlc` folder where openocd looks for configuration files; argument `-f stlink-swd.ocd` points to the session specific configuration we just created.

Upon success you shall see a similar output:
```
Info : clock speed 1000 kHz
Info : STLINK V3J7M2 (API v3)
Info : Target voltage: 3.3
...
Info : Listening on port 3333 for gdb connections
```

#### 2.2.3. MacOS
Unfortunately, openocd throws compilation errors on MacOS that I did not have motivation to resolve. The older version is available to install through `brew install openocd`, but it would not work with the newer stlink-v3. You are encouraged to troubleshoot the build and inform me on your findings.

```shell
brew install autoconf libtool libusb
git clone --recursive https://github.com/ntfreak/openocd.git # Original openocd repo.
cd openocd
./bootstrap
# Here you need to specify the paths to libusb.
./configure LIBUSB1_CFLAGS=-I/usr/local/Cellar/libusb/1.0.23/include/libusb-1.0 LIBUSB1_LIBS=-L/usr/local/Cellar/libusb/1.0.23/lib
make -j4
```

Yet it is possible to debug with the latest stlink hardware on MacOS, proceed to `2.2.4 Workaround`.

#### 2.2.4. Workaround
As a workaround you could scrape a pre built binary from [*STMCubeIDE*](https://www.st.com/en/development-tools/stm32cubeide.html). Tested on MacOS, should work on x86 Ubuntu and Windows. You'd have to download the STMCubeIDE package, but do not install it. Instead, extract and browse the package content, look for the following files.

- ST-LINK_gdbserver
- libSTLinkUSBDriver.dylib # .so on linux
- STM32_Programmer_CLI
- STLinkUpgrade.jar

Copy your findings to a separate folder. The gdbserver executable looks for its `libSTLinkUSBDriver` library in a certain subfolder. Make sure your files are organized as in the following example for macos.
```
.
├── ST-LINK_gdbserver
├── STLinkUpgrade.jar
├── STM32_Programmer_CLI
└── native
    └── mac_x64
        └── libSTLinkUSBDriver.dylib # .so on linux
```

Otherwise you will get the following error.
```
dyld: Library not loaded: @rpath/libSTLinkUSBDriver.dylib
```

Connect the hardware. Now you can run the gdbserver as
```
./ST-LINK_gdbserver -e -d -cp .
```

Uppon success you shall see
```
ST-LINK device initialization OK
Waiting for debugger connection...
Waiting for connection on port 61234...
```

Here `-cp .` is the path to `STM32_Programmer_CLI`, `-d` is for swd. If you look closer into the content of the STMCubeIDE package, you will find a `config.txt` in the folder with `ST-LINK_gdbserver`, which goes over all command line arguments the gdbserver can take. Once running, you might experience complaints about libusb, which can be installed with `brew install libusb` and outdated stlink firmware version, which cab be updated by running `STLinkUpgrade.jar` (has a gui).

I find `ST-LINK_gdbserver` easy to use as you do not have to provide any configuration files that openocd requires, it is rather hypocritical of STMicroelectronics not to freely distribute `ST-LINK_gdbserver`.

### 2.3. jlink

Jlink comes with all software ready to go, get it from [*here*](https://www.segger.com/products/debug-probes/j-link/tools/j-link-gdb-server/about-j-link-gdb-server/).

Look for `JLinkGUIServerExe` (warning gui), or `JLinkGDBServerCLExe` (command line). To avoid the gui prompts you need to provide enough arguments, described [*here*](https://wiki.segger.com/J-Link_GDB_Server).
```
JLinkGDBServerCLExe -if swd -device stm32f103c8
```

Warning: jlink edu is painful to run on a remote server as it requires you to confirm the license agreement in a gui prompt at least once a day. In case you need to work with a remote jlink edu, use openocd instead.

## 3. gdb

The GNU Debugger or [*gdb*](https://www.gnu.org/software/gdb/) is a standard tool for debugging ~~and hacking~~ software in various languages written for various architectures; gdb allows tracing program execution by setting breakpoints, watching memories and processor states. 

The example presented in this tutorial is just a tiny glimpse of what gdb is capable of. You can debugg along with [*this*](https://github.com/stansotn/blue-pill) empty initialization code for blue pill.

Start with loading the firmware containing debug symbols. Most flavors of linux including Ubuntu have gdb installed by default.
```shell
gdb blue-pill.elf
```

In case you are on macos, the default system debugger is lldb. I use `arm-none-eabi-gdb` from the embedded arm toolchain package available [*here*](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads).

Running gdb will welcome you into the debugger console, your command line output should be looking similar to the following.
```
Reading symbols from blue-pill.elf...
(gdb)
```

Next, make sure your gdbserver (openocd or jlink) is running and listening for connections. If you are familiar with gdb, you may know that you can attach to a process running on the same machine. Since in bare metal scenario, the program being debugged is running on a physically different machine and and likely a different architecture, the gdb connection will always be "remote" even if the gdbserver is running on the same computer.

The following command establishes connection with the gdbserver. Make sure your ip address and port matches the configuration. In case gdbserver is running on the same computer, you can put `localhost` instead of the ip address. The port depends on the gdbserver you are running, `3333` is default for openocd.
```
(gdb) target extended-remote 192.168.1.24:3333
```

Issuing `load` will flash the firmware!
```
(gdb) load
```

Loading firmware does not reset the mcu, that is the program counter has not been set to the reset handler address. Issue a reset and halt.
```
(gdb) monitor reset halt
```
`monitor <expression>` command sends the expression to gdbserver, gdb (client) itself does not know what `reset halt` does.

`step` will progress the program counter to the next location.
```
(gdb) step
```

Look, we are exiting reset handler! Your output should be similar to the following. 
```
Reset_Handler ()
    at /some_path/startup_stm32f103xb.s:66
```
Also, see how gdb knows `Reset_Handler()` is declared in startup_stm32f103xb.s line 66.

Let's set a breakpoint at the main function.
```
(gdb) break main
```
Referencing functions by name in gdb will only work with "global" functions. Let's set a breakpoint in the infinite loop by referring to the corresponding line in file `main.c`.
```
break main.c:96
```
To list all breakpoints type `info break`.  
Continue program execution.
```
(gdb) continue
```

This should take us to Breakpoint 1. 
```
Breakpoint 1, main () at /some_path/blue-pill/sources/src/main.c:74
```

`list` will display the proceeding lines of code following our current position. Here you could play with `step` and `next`, `step` will take you to the next program counter location, `next` -- to the next function call. If you are familiar with debugging in any IDE, `step` is equivalent of "step in", and `next` is "step out". `continue` to get to our second breakpoint. You can delete all breakpoints by issuing `delete` or delete a specific breakpoint `delete 1`, or you can also disable breakpoints with `disable`. If there is no active breakpoints, issuing `continue` will continue executions until you manually stop it with ctrl+c -- this halts execution of the debugee and does not exit gdb. There is so much to go on about gdb, we haven't started with watchpoints, processor states and memory.

Also, there is a python api that can help automate testing ~~and hacking~~ with gdb! 

### 3.1. gdb + gui

Many, if not all IDEs use gdb behind their visual interface. Now you can configure embedded debugging support in your favorite text editor. For instance if you use vscode, add a configuration file `.vscode/launch.json` in the project root. Make sure you have `ms-vscode.cpptools` extension. Here is my vscode debug configuration.

```json
// file: .vscode/launch.json
{ 
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "configurations": [
        {
            "name": "debug",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceRoot}/build/blue-pill.elf",
            "miDebuggerPath": "/Applications/ARM/bin/arm-none-eabi-gdb",
            // "debugServerPath" and "debugServerArgs" are commands to start the gdbserver
            //"debugServerPath": "/Applications/SEGGER/JLink_V662/JLinkGDBServerCLExe",
            //"debugServerArgs": "-device stm32l412cb -if swd",
            "cwd": "${workspaceRoot}",
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Connect to gdbserver",
                    "text": "target extended-remote 192.168.1.24:3333"
                },
                {
                    "description": "load executable",
                    "text": "file ${workspaceRoot}/build/blue-pill.elf"
                },
                {
                    "description": "Flash Firmware",
                    "text": "load"
                },
                {   
                    "description": "Reset target",
                    "text": "monitor reset halt"
                },
            ]
        }
    ]
}
```

## Afterword

> **Help Me Improve**  
> I am learning to write meaningful documentation. I hope you enjoyed this post, please help me back by emailing some feedback!
> - Is information clear, correct and up to date?
> - How would you improve this post?

