---
title: 'EBAZ4205 – “Recycle” a cheap crypto-miner (Part 4)'
permalink: /ebaz4205-recycle-cheap-crypto-miner-part-4/
categories:
    - 'FPGA Development'
    - SoC
    - Xilinx
    - Yocto
tags:
    - ebaz4205
    - SDK
    - SoC
    - 'Software Development Kit'
    - zeus
---

## Intro

In order to take full advantage of the device from a Software Development point of view, we still have some work to do. Up until now, it is not trivial to compile applications on a development host. The reason for this, obviously, is that the architectures do not match. But even if you fetch a cross compiler on your current Linux distribution, you also need to link against the libraries used on your embedded system. This can really be a pain in the \*\*\*, therefore, in order to make full use of the PS (Processing System) Yocto offers the capability to export a SDK (Software Development Kit) tailored exclusively to your distribution. So how does this work?

## The SDK

The good thing is, that a Software Development Kit (SDK) tailored for the EBAZ4205 can be generated by Bitbake. All we have to do, is to instruct Bitbake to prepare it for us. *(Note that at this point, you should have everything set up and ready as described in in [Part 3](https://embed-me.github.io/ebaz4205-recycle-cheap-crypto-miner-part-3/) of the series.)*

``` bash
bitbake ebaz4205-image-standard -c populate_sdk
```

Since this fetches, compiles and packs all the binaries that we need, it can take quite some time, but if you were following the series on how to “Recycle” a cheap crypto-miner, I am pretty sure you are already used to that 😉  
The generated tarball script can then be installed on any development host:

``` bash
./tmp/deploy/sdk/ebaz4205-dist-glibc-x86_64-ebaz4205-image-standard-cortexa9t2hf-neon-ebaz4205-zynq7-toolchain-1.0.sh
```

In order to activate it (by preparing the build environment), all we have to do, is source the installed environment-setup script.

``` bash
source /opt/ebaz4205-dist/1.0/environment-setup-cortexa9t2hf-neon-poky-linux-gnueabi
```

## The Application

Now, that everything is in place for cross-compilation, let’s stick to the most simple program one can imagine – classic “Hello World”.  
The following source code of our *main.c* file probably does not need any explanation.

``` c
#include <stdio.h>
#include <unistd.h>

int main (int argc, char *argv[])
{
        printf("Hello World from EBAZ4205!\n");
        return 0;
}
```

And in order to ease compilation, we will stick to [GNU make](https://www.gnu.org/software/make/manual/make.pdf). The following code section describes our *Makefile*.

``` makefile
CFLAGS=-c -Wall -Os
LDFLAGS=
SOURCES=main.c
OBJECTS=$(SOURCES:.c=.o)
EXECUTABLE=helloworld

all: $(SOURCES) $(EXECUTABLE)

$(EXECUTABLE): $(OBJECTS)
	$(CC) $^ -o $@ $(LDFLAGS)

%.o : %.c
	$(CC) $< -o $@ $(CFLAGS)

clean:
	rm -f $(EXECUTABLE)

cleanall:
	rm -f $(EXECUTABLE)
	rm -f $(OBJECTS)
```

The only thing that might be of interest here is that *$(CC)* will reference to our cross compiler that we previously set up. Therefore, we need to run *[make](https://linux.die.net/man/1/make)* in the same terminal we sourced the environment-setup script.

``` bash
make
```

## Verification

The output of the above make command is an executable. The *[file ](https://linux.die.net/man/1/file)*command will reveal, that indeed it is cross-compiled for an ARM architecture.

``` bash
$ file helloworld
helloworld: ELF 32-bit LSB shared object, ARM, version 1 (SYSV), dynamically linked (uses shared libs), BuildID[sha1]=63f4157307194669e84ad6b248374e1b1ea9c47c, for GNU/Linux 3.2.0, not stripped
```

After we used secure copy to place the binary on the EBAZ4205, we can verify it’s functionality.

``` bash
root@ebaz4205-zynq7:~# ./helloworld
Hello World from EBAZ4205!
```

Congratulations, you just cross-compiled an executable for your embedded target processor!

## What’s next

If you were following the most recent posts you might have figured it out. The Zynq is a never ending story. The capabilities are endless und there is always something to explore. With this short series of posts, we just scratched the surface and once you are comfortable with the Zynq, have a look at the MPSoC, it will blow your mind again 😉