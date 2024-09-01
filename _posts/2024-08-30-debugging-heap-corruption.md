---
title: 'Debug a KNX Stack Heap Currupton'
permalink: /debugging-heap-corruption/
categories:
    - 'Development'
tags:
    - development
    - C++
    - KNX
    - 'Home Automation'
---

## Intro

This KNX stack available on [GitHub](https://github.com/thelsing/knx) is an implementation of the KNX protocol, designed for building custom (home) automation systems. It enables developers to integrate and control devices such as lighting, heating, and security systems using the KNX standard, providing a framework for creating smart home and building solutions. It can be used with Arduino and an RP2040 PCB connected to the KNX bus. Due to my limited time recently, I thought this would be perfect for building a simple buzzer to notify me of certain events in the house (e.g., when the alarm is armed upon leaving) and at the same time learn something new. However, what I initially thought would be a done in an hour turned out to be more time consuming than expected. Here's a summary of how I uncovered the root cause behind my application's unexpected behavior, which turned out to be heap corruption in the KNX stack.


## The Situation

[Kaenx-Creator](https://github.com/OpenKNX/Kaenx-Creator) can be used to create ETS product files which can then be imported to ETS in order to parameterize your own KNX application. The details are not relevant for this post, however, what is relevant is that it generates a header file (knxprod.h) based on your configuration. This is how it looks like:

``` c
#define APP_Frequency_1_Parameter		0x0000
#define APP_Frequency_1_Parameter_Shift		1
#define APP_Frequency_1_Parameter_Mask		0x7FFF
// Offset: 0, Size: 15 Bit, Text: Frequenz
#define ParamAPP_Frequency_1_Parameter		((uint)((knx.paramWord(APP_Frequency_1_Parameter) >> APP_Frequency_1_Parameter_Shift) & APP_Frequency_1_Parameter_Mask))
#define APP_Duration_1_Parameter		0x0000
// Offset: 0, Size: 16 Bit (2 Byte), Text: Zeitdauer
#define ParamAPP_Duration_1_Parameter		((uint)((knx.paramWord(APP_Duration_1_Parameter))))
//!< Number: 0, Text: Trigger Melodie 1, Function: Play Melodie 1
#define APP_KoPlay_Melody     	      	 	0
#define KoAPP_Play_Melody			knx.getGroupObject(APP_KoPlay_Melody)
```

The generated file is tightly coupled to the KNX Stack meantioned above, already making use of some exposed methods of the stack.

Below shows an excerpt from my trivial application making use of above defines after configuration.

``` c
#include <Arduino.h>
#include <knx.h>      //this includes the knx stack and you can use the knx object!
#include <knxprod.h>  //include file above

void playMelody(GroupObject &go) {
    // play some nice melody/pattern
}

void setup() {
    // ...

    // read adress table, association table, groupobject table and parameters from eeprom
    Serial.println("Read Memory");
    knx.readMemory();

    // ...more setup

    // Attach a callback function to the group object
    KoAPP_Play_Melody.callback(playMelody);

    knx.start();
}

void loop() {
    knx.loop();
}
```

Let's program the device in ETS:

![](/assets/posts/debugging_heap_corruption/ets_download_all.png)

In the GUI it fails after some time:

![](/assets/posts/debugging_heap_corruption/ets_download_all_timeout.png)

The application freezes completeley with the following serial output:

``` bash
readMemory
restored Tableobjects
KNX - Individual Adress is set!
KNX - Is Configured
TP is connected
progmode on
Basic restart requested
save saveRestores 2
save tableobjs   <------ stuck here, only reset helps
```

What puzzled me even more was that the stack worked fine for others, and given the number of followers on the repository, I wondered why this issue seemed to affect only me. What was I doing wrong?

It was time to attach a debugger! I discovered that the issue occurs within the KNX stack when handling "MemoryBlocks." After reviewing the code in the repository, I noticed that these blocks are placed into a linked list and, during the program or configuration process, they are freed one by one starting with the first. Despite this, I couldn't identify any problems in the code, aside from the extensive use of raw pointers and the heap, which isn't ideal for C++ and embedded devices. There should be a warning if a block is freed that wasn't previously allocated, but I didn't see any such warning in the serial output above. So, what exactly is going on here?

``` c
void Memory::freeMemory(uint8_t* ptr)
{
    MemoryBlock* block = _usedList;
    MemoryBlock* found = nullptr;

    while (block)
    {
        if (block->address == ptr)
        {
            found = block;
            break;
        }

        block = block->next;
    }

    if (!found)
    {
        println("freeMemory for not used pointer called");
        _platform.fatalError();
    }

    removeFromUsedList(block);
    addToFreeList(block);
}
```

After spending more time on it, I began to connect the dots. The hard fault was caused by an invalid next pointer in the memory blocks linked list. This pointer was misaligned, and because the address wasn't 4-byte aligned, it triggered the hard fault.

Serial output of 'block->address' which corresponds to the address of the next block.

``` bash
...
Memory::freeMemory
block: 0x2000DC30
block->address: 0x100036F9 <---- HERE!?
Memory::freeMemory
block: 0x100036F9 <---- STUCK
```

Excerpt from the ARMv6-M Architecture Reference Manual which is the architecture the RP2040 Cortex-M0+ cores are based on:


> A3.2 Alignment support
> ARMv6-M always generates a fault when an unaligned access occurs


So this is the reason why we were not able to see "freeMemory for not used pointer called" on the console!

Now comes the tricky part - why is the next pointer incorrect? Surely some part of the code must have added a corrupt memory block, right? I spent hours debugging, but the corrupt block seemed to appear out of nowhere. So, how could I pinpoint which part of the thousands of lines of code was corrupting the stack?

Then I noticed something interesting: according to the RP2040 datasheet, the SRAM is located between 0x20000000 and 0x20042000. However, the next pointer was pointing to the FLASH (XIP) section, which starts at 0x10000000 and ends at 0x11000000 on my setup, as I have a 16MB external flash connected.

Maybe we are lucky and the address allows us to find the rough location where the corruption is happening.

``` bash
objdump firmware.elf 
...
100036f8 <_ZNSt17_Function_handlerIFvR11GroupObjectEPS2_E10_M_managerERSt9_Any_dataRKS5_St18_Manager_operation>:
100036f8:       b590            push    {r4, r7, lr}
100036fa:       b085            sub     sp, #20
100036fc:       af00            add     r7, sp, #0
100036fe:       60f8            str     r0, [r7, #12]
10003700:       60b9            str     r1, [r7, #8]
10003702:       1dfb            adds    r3, r7, #7
10003704:       701a            strb    r2, [r3, #0]
```

Although a bit cryptic, the name suggests that this function is likely the handler for Group Objects, which aligns with the method call in our application. By combining details from the auto-generated header and our source code:

``` c
// from knxprod.h
#define APP_KoPlay_Melody 0
#define KoAPP_Play_Melody knx.getGroupObject(APP_KoPlay_Melody)

// within our application
KoAPP_Play_Melody.callback(playMelody);
```

So the actual call looks like this:

``` c
knx.getGroupObject(0).callback(playMelody);
```

Let's jump into the knx stack source code to find out what is done when we register the callback:

``` c
GroupObject& getGroupObject(uint16_t goNr)
{
    return _bau.groupObjectTable().get(goNr);
}
```

The groupObjectTable() method simply returns the object itself:

``` c
GroupObjectTableObject& BauSystemBDevice::groupObjectTable()
{
    return _groupObjTable;
}
```

Ah, now it all makes sense. We're passing an argument of 0, so (uint16_t)0-1 results in -1, which equals 0xffff, or 65535. That can't be good ;-)

``` c
GroupObject& GroupObjectTableObject::get(uint16_t asap)
{
    return _groupObjects[asap - 1]; 
}
```

And because group objects are raw pointers...

``` c
GroupObject* _groupObjects = 0;
```

...and they leave on the heap...

``` c
bool GroupObjectTableObject::initGroupObjects()
{
    //...
    _groupObjects = new GroupObject[goCount];
    //...
}
```

...they will corrupt any memory address they point to once we register our callback!

## Summary

What happens is that a faulty object pointer is returned. When we perform actions on this object, it leads to heap corruption. During storage or freeing, one of the next pointers in the linked list pointed to an address that was NOT 32-bit aligned (a requirement for the RP2040 platform), causing an exception that locked up the processor. It turns out that, according to the KNX standard, only Group Object IDs starting from 1 are permitted, but due to no error handling in the library, this part was missed.

## Lessions Learned

Avoid using raw pointers and C-style arrays in C++; instead, use available container classes. Improper error handling and lack of bounds checking when accessing arrays can result in serious, hard-to-debug issues. Maintainers were notified and added proper error handling.