# Intro to Vivado for Software Engineers

Apparently many software engineers hear about FPGA's, they get excited enough to buy a FPGA dev board. They install the tools and open it up and are immediately dazzled by all of the features and then give up and never touch their FPGA board again. This is meant to be a quick helpful tutorial to help someone with a software background get started with FPGA's. Rather than getting bogged down in nitty gritty details of VHDL and TCL syntax, I'm just going to go over it by looking through a basic example.

This specific tutorial uses Digilent's Arty board and VHDL.

This tutorial assumes you've downloaded, licensed, and installed Vivado. This also assumes that you've figured out the right switch settings to set on your board, and you got the USB driver working.

## Source Files
The first thing that you need to understand about in FPGA designs are your source files. You have two types of source files. Your HDL which defines the logic and your constraints.

The most important constraints that a simple FPGA design needs are pin locations. You obviously can't use whichever pins you want. If you want to blink an LED, you'll need to know which pin is connected to the LED. Usually if you buy a dev board, the board vendor will provide you with a constraint file that tells you where all of the pins are. Constraint files in Vivado have the extension XDC. The syntax of an XDC file is TCL (the scripting language). Vivado will essentially just execute your XDC file like a TCL script to assign pin locations to your signals. If the only thing that we want to do is to blink an LED then we probably only need a clock and an LED pin. We can delete the rest of the constraints.

To blink some lights on our arty board we need the following constraint file.

top.xdc
```
set_property -dict { PACKAGE_PIN E3    IOSTANDARD LVCMOS33 } [get_ports { clk100mhz }];
create_clock -add -name sys_clk_pin -period 10.00 -waveform {0 5} [get_ports { clk100mhz }];
set_property -dict { PACKAGE_PIN H5    IOSTANDARD LVCMOS33 } [get_ports { led }];
```

These are copied directly from the XDC file that was provided by Digilent. The biggest change is that I renamed led[0] to led. There's a lot of information in these constraints. The most important bits are the `PACKAGE_PIN X` and the `get_ports { Y }`. X is which pin of the FPGA we are using and Y is the port name we will use in our VHDL.

The other type of source file is the HDL. You get to choose VHDL or Verilog. While there are alternatives, I wouldn't say that any of them are mature enough to use them as a starting language yet. For this tutorial, I will use VHDL because it's what I'm more familiar with, but the two languages aren't amazingly different in their capabilities. HDLs describe logic and the connections between components. HDL files usually consist of expressions that assign values to signals. As an example, I'm going to give away the VHDL for blinken-lights so you can see what the basic structure of a VHDL file looks like.

top.vhd
```
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity top is
  port (
    clk100mhz : in std_logic;
    led : out std_logic
  );
end top;

architecture behavioral of top is
  signal counter : integer range 0 to 100_000_000 := 0;
begin

  led <= '1' when counter > 50_000_000 else '0';
  
  process(clk100mhz)
  begin
    if rising_edge(clk100mhz) then
      counter <= counter + 1;
    end if;
  end process;

end behavioral;
```

I'll break down each of the bits of this example:
```
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;
```
This is boring boiler plate you will need in nearly every component. These standard libraries define most of the types used in modern VHDL.

```
entity top is
  port (
    clk100mhz : in std_logic;
    led : out std_logic
  );
end top;
```
The two important pieces here are that we gave this component a name (`top`), and we declare our port names, their directions and their types. Both of the types of these ports are `std_logic`. There's a lot of interesting things that `std_logic` can do, but in this case, it's only interesting property is the fact that you can assign them the value `'1'` and the value `'0'`. They are useful for describing single-bit wires or pins.

```
architecture behavioral of top is
  signal counter : integer range 0 to 100_000_000 := 0;
begin
```
You can create multiple implementations of the same component. Honestly, this feature of VHDL isn't used very often. 99.9% of VHDL entities only have one architecture. But we still need to give it a name. The name in this example is `behavioral`. In between the `is` and the `begin` are where you put your declarations (signals, constants, functions, etc). In this example we declare one signal named `counter`. We also declared it's type (`integer range 0 to 100_000_000`) and it's initial value (`0`). In VHDL we can specify any range of integer that we would like. This specific integer goes from 0 to 100 million. VHDL integers wrap, so if I increment this particular signal when it's current value is 100 million, it's new value would be 0.

In VHDL you can construct combinational logic through assignment of expressions. Expressions can use operator primitives like +, -, and, or, not, etc. Here's an example of a VHDL combinational expression.

```
signal_name <= foo + bar; 
```

It's important to understand that this combinational statement doesn't execute at a particular moment in time, it is continuously executing. `signal_name` is wired to the output of the result of a adder connected to `foo` and `bar`. As `foo` and `bar` change over time, `signal_name` will short time afterward to the sum of `foo` and `bar`.

In our blinken-lights example we have a slightly more complicated expression:
```
led <= '1' when counter > 50_000_000 else '0';
```

This is the form for conditional assignment in combinational logic. It doesn't look too different than conditional assignments you might see in C or python.

One of the basic building blocks of an FPGA is a register. A register is a building block that we use to store the current circuit state. A register stores data every single clock cycle. Any signals that you want to store inside of a register must be able to travel through any combinational logic and routing delays and reach the register before the rising clock edge. If Vivado doesn't think that the signal will be able to get to the flop in time, it will fail your compile.

This is the basic pattern of what a register looks like in VHDL

```
process(clk100mhz)
begin
  if rising_edge(clk100mhz) then
    signal_name <= next_value;
  end if;
end process;
```
This process is assigning a new value to `signal_name` every rising edge of the clock. Any signal that is assigned inside of the rising edge of a clock will infer a register. So Vivado will recognize that `signal name` is a register. `next_value` doesn't need to be a signal. It can be an expression and Vivado will infer the necessary combinational logic that feeds into the register's next value.

This section in our blinken-lights example is the counter implementation.
```
process(clk100mhz)
begin
  if rising_edge(clk100mhz) then
    counter <= counter + 1;
  end if;
end process;
```
The counter's assignment in the clocked process means we want a register. The expression `counter + 1` means every clock cycle we would like this register to be incremented. The clock's frequency is 100MHz so we are incrementing this counter at 100,000,000 times per second. If you recall, our specific counter has a range from 0 to 100,000,000 so this counter will wrap around once per second.

While there are many other features of VHDL, most of it is just based around the concepts we have just used. Sequential assignment to create registers and combinational logic.

## Bitfile generation
Now that we understand our source files, let's try to compile. We compile our constraints and HDL files into bitfiles that we can transmit to the FPGA chip. Similar to how SW has steps like parsing, object code generation, and linking, FPGA designs also have discrete steps. I'm not going to talk too much about what they do, but it's important to understand the general order that they occur in.

* Synthesis
  * Elaboration
  * Technology Map
* Implementation
  * Place
  * Route
* Bitstream Generation

In terms of Vivado commands to invoke these steps they are
* `synth_design`
* `place_design`
* `route_design`
* `write_bitstream`

Now at this point you might open up Vivado in GUI mode, create a project, add your files and start pushing buttons. That would work, but honestly, I think a less intimidating path for a software engineer learning their way around this tool for the first time is to invoke Vivado in batch mode. Invoking Vivado in batch mode means that Vivado will execute a fixed set of steps from command line. These steps are specified in a TCL script.

Here's an example of a TCL script you might use:
build.tcl
```
read_vhdl top.vhd
read_xdc top.xdc
synth_design -top top -part xc7a35ticsg324-1L
place_design
route_design
write_bitstream blinky.bit
```
This isn't a whole lot more than the Vivado commands we just learned. Notice that `synth_design` has a few more arguments, we had to specify the top and the part number. Before we synthesized, we had to read the source files in and we gave the output file name to the `write_bitstream` command.

You can see how this is roughly analogous to a Makefile for a software project. We specified our source files and the commands to invoke for our compiler.

Assuming we have our source files in the right place with the right names and Vivado in our path, then we should be able to invoke Vivado from commandline.
```
>vivado -mode batch -source build.tcl
```
You should see a bunch of noise rain down from the console as it compiles the design but assuming you didn't mess anything up, then it should've spit out a bitfile named `blinky.bit`. Then we need to send the bitfile to the FPGA.

## Downloading the bitfile
For Vivado's hardware manager, I'd probably recommend doing this from GUI mode the first time. If you launch Vivado in GUI mode, there's a big friendly buton for "Open Hardware Manager". Give that a click. One of the neat things about Vivado is that it displays what TCL commands you are executing when you push a button. I like to follow along with what buttons I'm clicking to see what Vivado is doing. It's also nice because it makes it easier to copy these commands and execute them in TCL scripts. By clicking that button, you should've executed `open_hw`. At this point, we make sure our FPGA is connected to the computer, it's in JTAG mode, and powered on correctly, and the drivers installed correctly.

Now we want to auto-connect. There's a little icon with a green board and yellow down arrow. It's also an option if you click the Open target text up near the top. The Tcl prompt tells us that we executed `connect_hw_server`, `open_hw_target`, `current_hw_device [lindex [get_hw_devices] 0]` and `refresh_hw_device -update_hw_probes false [lindex [get_hw_devices] 0]`. All this is doing is finding out what devices you currently have connected to this machine and it sets the current device to what is probably your own Xilinx dev board you have connected to your computer.

We are doing a JTAG scan of our system. The JTAG scan is capable of doing some pretty neat poking around on our hardware (for instance, apparently my FPGA is apparently 36.0 degrees celsius), but the reason we are here is to download our bitfile.

I just right-click my FPGA (xc7a35t), and click program device, and browse to your bitfile. The Tcl console says we executed something like this
```
set_property PROBES.FILE {} [lindex [get_hw_devices] 0]
set_property PROGRAM.FILE {blinky.bit} [lindex [get_hw_devices] 0]
program_hw_devices [lindex [get_hw_devices] 0]
```

Neat. So now the LED should be blinking. We have blinken-lights!

