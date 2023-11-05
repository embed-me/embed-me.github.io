---
title: 'Google 2020 CTF - Hardware Challenge'
date: '2020-11-06T19:12:13+00:00'
permalink: /google-2020-ctf-hardware-challenge/
categories:
    - CTF
    - 'Reverse Engineering'
tags:
    - capturetheflag
    - ctf
    - google
    - 'Reverse Engineering'
---

## Intro

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_hardware/google_ctf_banner.png)

When I heard that this years google CTF also contains a hardware challenge involving SystemVerilog I was quite stoked. As an Embedded Systems Engineer focused on FPGAs, it was clear instantly – I have to give it a try. For those of you who do not know SystemVerilog, it is a hardware description language used to design and implement electronic circuits within a chip (although it can also be used for simulation and verification).

## The Challenge

Let’s have a look at the google domain hosting the [CTF](https://capturetheflag.withgoogle.com/). By the way, you do not need to register in order to get access to the challenges. Simply use forceful browsing – <https://capturetheflag.withgoogle.com/challenges>

The basic hardware challenge is what we are looking for. First download the attachment and see what we have got.

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_hardware/google_ctf_overview.png)

The first file contains simple C++ Code which looks as follows.

``` cpp
#include "obj_dir/Vcheck.h"

#include <iostream>
#include <memory>

int main(int argc, char *argv[]) {
    Verilated::commandArgs(argc, argv);
    std::cout << "Enter password:" << std::endl;
    auto check = std::make_unique<Vcheck>();

    for (int i = 0; i < 100 && !check->open_safe; i++) {
        int c = fgetc(stdin);
        if (c == '\n' || c < 0) break;
        check->data = c & 0x7f;
        check->clk = false;
        check->eval();
        check->clk = true;
        check->eval();
    }
    if (check->open_safe) {
        std::cout << "CTF{real flag would be here}" << std::endl;
    } else {
        std::cout << "=(" << std::endl;
    }
    return 0;
}
```

Going through the code step by step, we can see that:

1. The applications generates a *Vcheck* instance
2. Enters a loop that allows you to provide up to 100 characters and exits when the *open\_safe* is true
3. Reads a character from stdin until a newline is read or the character is negative
4. Masks the character for the range 0..127 (7bits) and provides it to the *check* object
5. The next instructions seem to perform a rising edge to the *clk* input of the *check* object.
6. Finally, if the *open\_safe* variable is true it will print the flag

But where is the implementation of this mysterious class which seems to hold the secret how to “open the safe”? The file ending of the second file already looks familiar to me – it’s probably SystemVerilog. In order to understand the following code, it helps to think in terms of hardware, not software.

``` code
module check(
    input clk,

    input [6:0] data,
    output wire open_safe
);

reg [6:0] memory [7:0];
reg [2:0] idx = 0;

wire [55:0] magic = {
    {memory[0], memory[5]},
    {memory[6], memory[2]},
    {memory[4], memory[3]},
    {memory[7], memory[1]}
};

wire [55:0] kittens = { magic[9:0],  magic[41:22], magic[21:10], magic[55:42] };
assign open_safe = kittens == 56'd3008192072309708;

always_ff @(posedge clk) begin
    memory[idx] <= data;
    idx <= idx + 5;
end

endmodule
```

Once again, looking through the code step by step, we can see that:

1. The inputs/outputs of the module
2. Data is taken from the data input to the internal memory when a positive edge of the clock signal occurs
3. The index is incremented by 5, however, since the register is only 3 bits wide, the higher bits will be truncated, leading to the following sequence 0, 5, 2, 7… in which *data* is written to *memory*
4. The *memory* content registers are then connected to a wire *magic*
5. The wires from magic are connected to the wires called *kittens*
6. Finally *kittens* is compared to the decimal value *3008192072309708*

In order to get the safe to open, everything that is required is to do the process in reverse. For a more graphical representation of how the 56 bits are swapped, have a look at the following graph.

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_hardware/system_verilog_safe.png)

For example, the first char (7bit value) that is provided will be placed at index 0 of the register named *memory*. Wires than rout these bits to *magic\[55:49\]* and from there to *kittens*\[6:0\] where the final comparison occurs. In order to perform the bit swapping, I decided to write a python script.

## The Solution

There are a lot of possibilities to implement this, however, having a strong background in C and since the 56 bit fit into a long value, I came up with the following python code which makes heavy use of binary operators. Looking at it right now, it would have been a lot cleaner to convert *kittens* into a bit array…

``` python
#!/usr/bin/env python3

# Get this from the System Verilog File
kittens = 3008192072309708
print("kittens: {0:b}, 0x{0:x}".format(kittens))

# kittens segments
kittens_0 = ((kittens >> 0) & 0x3fff) 
kittens_1 = ((kittens >> 14) & 0xfff)
kittens_2 = ((kittens >> 26) & 0xfffff)
kittens_3 = ((kittens >> 46) & 0x3ff)

# Get magic vector
magic = (kittens_3) | (kittens_1 << 10) | (kittens_2 << 22) | (kittens_0 << 42)
print("magic:   {0:b}, 0x{0:x}".format(magic))

# Reorder memory
memory = []
memory.append((magic >> 7*7) & 0x7f)
memory.append((magic >> 0*7) & 0x7f)
memory.append((magic >> 4*7) & 0x7f)
memory.append((magic >> 2*7) & 0x7f)
memory.append((magic >> 3*7) & 0x7f)
memory.append((magic >> 6*7) & 0x7f)
memory.append((magic >> 5*7) & 0x7f)
memory.append((magic >> 1*7) & 0x7f)

pw = ""
index = 0
for i in range(8):
    pw += chr(memory[index])
    index = (index + 5) & 0x07

print("------------------- Password -------------------")    
print(pw)
```

So we generated a password that we think is correct, but as an FPGA or ASIC engineer we always need proof that the circuit performs as intended. We need a simulation in order to verify that the password is really correct SystemVerilog and Verilog in general is more common in America, while VHDL is more common in Europe. I am able to read and code both languages, however, I decided to use VHDL for the Testbench – yes, they are interchangeable, modules in one language can be integrated within a module of another language.

``` code
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;


entity check_tb is
end entity check_tb;


architecture arch of check_tb is

  signal clk_s       : std_logic := '0';
  signal data_s      : std_logic_vector(6 downto 0);
  signal open_safe_s : std_logic;

  procedure clk_p (
    signal clk_s : inout std_logic) is
  begin
    clk_s <= '0';
    wait for 10 ns;
    clk_s <= '1';
    wait for 10 ns;
  end procedure clk_p;
  
begin

  DUT : entity work.check
    port map (
      clk       => clk_s,
      data      => data_s,
      open_safe => open_safe_s);

  WaveGen_Proc : process
  begin

    wait for 10 ns;

    data_s <= b"0110111"; --Index: 0, Char: 7
    clk_p(clk_s);
    data_s <= b"1001100"; --Index: 5, Char: L
    clk_p(clk_s);
    data_s <= b"1101111"; --Index: 2, Char: o
    clk_p(clk_s);
    data_s <= b"1011000"; --Index: 7, Char: X
    clk_p(clk_s);
    data_s <= b"0100101"; --Index: 4, Char: %
    clk_p(clk_s);
    data_s <= b"0101010"; --Index: 1, Char: *
    clk_p(clk_s);
    data_s <= b"1011111"; --Index: 6, Char: _
    clk_p(clk_s);
    data_s <= b"1111000"; --Index: 3, Char: x
    clk_p(clk_s);

    assert false report "Simulation done!" severity failure;

  end process WaveGen_Proc;

end architecture arch;
```

The following code instantiates the module *check* (Design Under Test or DUT) and provides the password character per character on the rising edge. When done, the simulation is stopped by assertion.

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_hardware/modelsim6.png)

As can be seen above, the curser marks the point in time where the first sample is taken from the *data* input (0x37 = ‘7’) and is stored in the memory’s register offset 0. To fill the whole memory, this process repeats 8 times in total. Note that red signals (or ‘X’) indicate uninitialized states.

![_config.yml]({{ site.baseurl }}/images/google_2020_ctf_hardware/modelsim4.png)

In the end, the *open\_safe* signal turns high, which indicates that the combination of characters is correct and our thesis about the functionality of the design, as well as the password generation process is correct. Yay! A closer look at the *kittens* variable at the very bottom shows the hexadecimal value *0x0aafef4be2dbcc* which is decimal for *3008192072309708*.

Finally, the most rewarding step is to provide the flag to google for verification. Connecting with the use of netcat, we are prompted for the password which we happily supply.

``` batch
nc basics.2020.ctfcompetition.com 1337
```

…and we are rewarded with the flag

``` batch
CTF{W4s*************ck?}
```

Thank you google for this awesome hardware challenge. I might even try to grab some more flags if I find the time to.