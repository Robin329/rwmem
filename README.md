[![Build Status](https://travis-ci.org/tomba/rwmem.svg?branch=master)](https://travis-ci.org/tomba/rwmem)

# rwmem - A small tool to read & write device registers

Copyright 2013-2016 Tomi Valkeinen

## Intro

rwmem is a small tool for reading and writing device registers. rwmem supports
two modes: mmap mode and i2c mode.

In mmap mode rwmem accesses a file by memory mapping it. Using /dev/mem as the
memory mapped file makes rwmem access memory and can thus be used to access
devices which have memory mapped registers.

In i2c mode rwmem accesses an i2c peripheral by sending i2c messages to it.

rwmem features:

* addressing with 8/16/32/64 bit addresses
* accessing 8/16/32/64 bit memory locations
* little and big endian addressess and accesses
* bitfields
* address ranges
* register description database

## Build Dependencies

You need `cmake` for creating the makefiles.

You need python3 development files if python bindings are enabled (RWMEM_ENABLE_PYTHON, enabled by default). On debian based systems you need to install `python3-dev` package.

## Build Instructions

```
git clone https://github.com/tomba/rwmem.git
cd rwmem
git submodule update --init
mkdir build
cd build
cmake ..
make -j4
```

## Cross Compiling Instructions:

**Directions for cross compiling depend on your environment.**

These are for mine with buildroot:

```
$ mkdir build
$ cd build
$ cmake -DCMAKE_TOOLCHAIN_FILE=<buildrootpath>/output/host/usr/share/buildroot/toolchainfile.cmake ..
$ make -j4
```

Your environment may provide similar toolchainfile. If not, you can create a toolchainfile of your own, something along these lines:

```
SET(CMAKE_SYSTEM_NAME Linux)

SET(BROOT "<buildroot>/output/")

# specify the cross compiler
SET(CMAKE_C_COMPILER   ${BROOT}/host/usr/bin/arm-buildroot-linux-gnueabihf-gcc)
SET(CMAKE_CXX_COMPILER ${BROOT}/host/usr/bin/arm-buildroot-linux-gnueabihf-g++)

# where is the target environment
SET(CMAKE_FIND_ROOT_PATH ${BROOT}/target ${BROOT}/host)

SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

## Build options

You can use the following cmake flags to control the build. Use `-DFLAG=VALUE` to set them.

Option name              | Values        | Default  | Notes
------------------------ | ------------- | -------- | --------
CMAKE_BUILD_TYPE         | Release/Debug | Release  |
BUILD_SHARED_LIBS        | ON/OFF        | OFF      | librwmem.so instead of librwmem.a
RWMEM_ENABLE_PYTHON      | ON/OFF        | ON       | Enable python bindings
BUILD_STATIC_EXES        | ON/OFF        | OFF      | Build with -static
TREAT_WARNINGS_AS_ERRORS | ON/OFF        | OFF      |

## Examples without register file

Show what's in memory location 0x58001000

        $ rwmem 0x58001000

Show what's in memory locations between 0x58001000 to 0x58001040

        $ rwmem 0x58001000-0x58001040

Show what's in memory locations between 0x58001000 to 0x58001040

        $ rwmem 0x58001000+0x40

Show what's in memory location 0x58001000's bit 7 (i.e. 8th bit)

        $ rwmem 0x58001000:7

Show what's in memory location 0x58001000's bits 7-4 (i.e. bits 4, 5, 6, 7)

        $ rwmem 0x58001000:7:4

Write 0x12345678 to memory location 0x58001000

        $ rwmem 0x58001000=0x12345678

Modify memory location 0x58001000's bits 7-4 to 0xf

        $ rwmem 0x58001000:7:4=0xf

Show the byte at location 0x10 in file edid.bin

        $ rwmem --mmap edid.bin -s 8 0x10

Set /dev/fb0 to red

        $ rwmem -p q --mmap /dev/fb0 0x0..$((800*4*480))=0xff0000

Read a byte from i2c device 0x50 on bus 4, address 0x20

        $ rwmem -s 8 --i2c=4:0x50 0x20

Read a 32 bit big endian value from 16 bit big endian address 0x800 from i2c device 0x50 on bus 4

        $ rwmem -s 32be -S 16be --i2c=4:0x50 0x800

## Examples with register file

Show the whole DISPC address space

        $ rwmem DISPC

Show the known registers in DISPC address space

        $ rwmem DISPC.*

Show SYSCONFIG register in DISPC

        $ rwmem DISPC.SYSCONFIG

Modify MIDLEMODE field in DISPC's SYSCONFIG register to 0x1

        $ rwmem DISPC.SYSCONFIG:MIDLEMODE=0x1

List all registers in the register file

        $ rwmem --list

List registers in DISPC

        $ rwmem --list DISPC

List registers in DISPC

        $ rwmem --list DISPC.*

List registers in DISPC starting with VID

        $ rwmem --list DISPC.VID*

List fields in DISPC SYSCONFIG

        $ rwmem --list DISPC.SYSCONFIG:*

Read binary dump of DISPC to dispc.bin file

        $ rwmem --raw DISPC > dispc.bin

Show SYSCONFIG register, as defined in dispc.regs, in file dispc.bin

        $ rwmem --mmap dispc.bin --regs dispc.regs --ignore-base DISPC.SYSCONFIG

## Write mode

The write mode parameter affects how rwmem handles writing.

Write mode 'w' means write-only. In this mode rwmem never reads from the
address. This means that if you modify only certain bits with the bitfield
operator, the rest of the bits will be set to 0.

Write mode 'rw' means read-write. In this mode rwmem reads from the address,
modifies the value, and writes it back.

Write mode 'rwr' means read-write-read. This is the same as 'rw' except rwmem
reads from the address again after writing for the purpose of showing the new
value. This is the default mode.

## Print mode

The print mode parameter affects what rwmem will output.

'q'  - quiet
'r'  - print only register value, not individual fields
'rf' - print register and fields (when available).

## Raw output mode

In raw output mode rwmem will copy the values it reads to stdout without any
formatting. This can be used to get binary dumps of memory areas.

## Size and Endianness

You can set the size and endianness for data and for address with -s and -S
options. The size and endianness for address is used only for i2c. The size is
in number of bits, and endianness is "be", "le", "bes" or "les".

In addition to the normal big (be) and little endian (le), rwmem supports
"swapped" modification for endianness (bes and les). In swapped endianness, 16
bit words of a 32 bit value are swapped, and similarly 32 bit words of a 64
bit value are swapped.

## Register description file

A register description file is a binary register database. See the code
(regfiledata.h) and the included python scripts to see the details of the
format.

It is easy to generate register description files using the regfile_writer.py
python script. Two additional python scripts can be used to parse IPXACT files
(ipxact_parse.py) and csv files from rwmem v1 (csv_parse.py).

## Bash completion

examples/bash_completion/rwmem is an example bash completion script for rwmem.

## rwmem.ini file format

rwmem will look for configuration options from ~/.rwmem/rwmem.ini file.
examples/rwmem.ini has an example rwmem.ini file.

main.detect entry can be set to point to a script which should echo the name
of the platform rwmem is running on. The name of the platform is then used to
look for "platform" entries in the rwmem.ini, which can be used to define
platform specific rwmem configuration (mainline regfile for the time being).
