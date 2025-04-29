# USB-PD Power Manager

This is a USB-PD-capable Battery Charger and Power Path Module based on the [HUSB238 USB-PD Sink Controller](https://www.hynetek.com/uploadfiles/site/219/news/aabbbbdb-48c9-4a44-a6dc-2c15f53282e6.pdf) and [BQ25672 Battery Charger](https://www.ti.com/lit/ds/symlink/bq25672.pdf).

> This is **NOT** a finished project!


## Objectives

The main objective with this module is, quite literally, to make the "perfect Power Manager module/solution for any project." Obviously, "perfect" here is what I consider perfect myself, so here goes a laundry list of things I want from it:

1. Actually negotiate power when powered through USB, whatever the means.
    - A lot of projects just depend on the common 5.1K resistor on CC1/CC2 to get 5V/3A, but that's not a guarantee unless your use-case is very restricted.
    - Also, pulling 500mA or more on USB without USB connection/negotiation, USB-PD or USB-BC is out of spec. Not that it matters in all projects, but a well behaved USB-powered solution avoids many surprises.
    - 5V is pretty ubiquitous, but some projects may necessitate more voltage to drive higher-powered devices (e.g. fans), or to charge batteries more efficiently. Higher voltages are also necessary in order for USB-C to deliver more power in general.
2. Capable of negotiating power through **USB PD**, up to the highest power profile available (100W, 20V @ 5A).
    - In the future, I'd like this to be expanded to support USB-PPS as well, but chip availability is a bit low for enabling this functionality, so USB-PD will do for the moment.
3. Capable of falling back to **USB BC**, if not connecting to USB Type-C end-to-end.
    - There's plenty of power available through USB BC 1.2 or similar charger protocols, and it's still very common to use previous USB-A or USB-B (i.e. non Type-C) ports in many places.
4. Battery-powered, with a battery charger built-in.
    - Necessitates Power Path management.
    - Also necessitates battery protection measures, e.g. overcurrent protection, overvoltage protection, reverse polarity protection, etc.
5. Have the smallest possible form-factor. Be friendly to breadboarding, while still being useful as a standalone module.
    - Size would be similar to a Raspberry Pi Pico in width, but no specific restriction on length.
    - Ahem... ✨ [Castellations!](https://jlcpcb.com/blog/castellated-pcbs-introduction-and-design-requirements) ✨ (What a word, ain't it?)
6. Guaranteed 5V and 3.3V outputs, regardless of the "mains" voltage.
    - 5V with a switched dc/dc regulator, for powering sensors, LEDs, etc. Optionally adjustable to 3.3V via standalone (pin) configuration.
    - 3.3V with a LDO for powering microcontrollers, also adjustable to 5V if needed (e.g. for less noise).
    - Both can be disabled at any time, to reduce power consumption.
7. Optional external power input, as a backup for USB power.
8. Battery fuel gauge built-in to the module.
    - Helps with power consumption prediction, etc.
9. I2C as the main control option, but standalone pin or jumper based configuration also possible.
    - Standalone controls enable the board to be used in scenarios without a microcontroller.
    - I2C enables the board to be used with higher precision (e.g. negotiate USB-PD power after first negotiation, configure current/charger limits past the default standalone ones, monitor power consumption, etc.)

I also wanted to put a soft-reset button. I like the idea of the module having power and reset buttons that are useable to a microcontroller, with e.g. long presses meaning turn-off/reset, etc. Perhaps that may be included in a future revision.


## Implementation

### Component selection

Focus on ICs from known brands. Easier to predict availability and quality.

Basic test: go to JLCPCB and see if there's a good amount of stock available. Discard any option with zero stock. Price also important, but not essential (as this will be a personal project).

- USB-PD Sink Controller (**VUSB**): [**HUSB238_002DD**](https://www.hynetek.com/uploadfiles/site/219/news/aabbbbdb-48c9-4a44-a6dc-2c15f53282e6.pdf)
    - 001DD or 004DD would be the best option, but the 002DD has a whole lot more availability for some reason.
    - VUSB is not supposed to be consumed directly, but instead it must be passed on to the Power Path controller.
    - VUSB is only switched on when USB-PD negotiation is performed (either with success or failure).
- Battery Charger and Power Path (**VSYS**): [**BQ25672**](https://www.ti.com/lit/ds/symlink/bq25672.pdf)
    - Selected mostly because of the 24V input voltage tolerance.
    - Having I2C is also important, but not essential.
    - Could be the BQ25792 as well, it's pin-compatible and offers more power, but the BQ25672 has better availability
- Switch-mode DC/DC converter (**VSW**): [**TPS63070**](https://www.ti.com/lit/ds/symlink/tps63070.pdf)
    - Being able to step-up is important for scenarios with 1-cell batteries, or when the input voltage is lower than the output voltage in general. Even more important is to be able to step-down the voltage, considering the upwards of 24V inputs. Hence, it must be a buck-boost.
    - Input comes from VSYS (BQ25672), which reduces the required VIN to 19V.
    - Unfortunately, it has a 16V input voltage limit, but it's the highest limit amongst the moderately available buck-boosts out there. I found no other buck-boost as available or as cheap as this one. If anyone knows of a better option PLEASE let me know, I researched quite a lot but couldn't find a better one!
    - Because of the low VIN input, the module can only be used with up to 3-cell configurations, which limits VSYS to around 13V max. Going above that risks damaging the buck-boost.
- Low-dropout voltage regulator (**VLDO**): [**TPS7A2601**](https://www.ti.com/lit/ds/symlink/tps7a26.pdf)
    - I quite like this LDO!
    - Current limit is a bit low at 500mA, but it's enough for most projects. Higher power can be better served by VBUS or VSW
    - Target output voltages: 3.3V and 5V.
    - Has a very high VIN, at 20V. Safe to connect to VBUS/VSYS, but not exactly safe to use at higher voltages (e.g. regulate from 20V VIN but with much lower output currents in order not to blow it up!)
- MOSFETs: focus on using dual mosfet ICs, in order to reduce board space usage whenever possible.
    - Dual N-Channel: [**Si7224DN**](https://www.mouser.com/datasheet/2/427/si7224dn-1764912.pdf)
        - One of the channels has a lower Vgs, kinda weird. Could have chosen a better one.
    - Dual P-Channel: [**Si7223DN**](https://www.vishay.com/docs/75609/si7223dn.pdf)
    - N-channel and P-channel: [**DMC3025LDV**](https://www.diodes.com/datasheet/download/DMC3025LDV.pdf)
    - Focus on packages compatible with DFN-3x3. Small enough for higher power, whilst being easier to find replacements later on (e.g. chinese options like the HSBB3313, HSBB3909, HSBB6254, HSBB3202, etc.)
    - There were probably better options out there, but I got tired of looking at some point...
- Battery fuel gauge: [**MAX17205G**](https://www.analog.com/media/en/technical-documentation/data-sheets/max17201-max17215.pdf)
    - No particular reason for choosing this one other than the fact that it supports at least 3-cell batteries.
    - Has good performance, low power consumption and enough availability.
    - Useful that it can balance cells as well!


### Module topology

The KiCad schematic focuses on showing the whole module as a single page. I like the idea of being able to see the whole circuit at a glance, obviously separated by smaller logical parts, but this may not be optimal in order to show the constituent parts of the schematic as a whole.

So, follows below a diagram making more evident the constituent parts of the module, I hope it makes sense:

![Topology](/docs/img/usb-pd-power-manager-topology.drawio.svg)

### Schematics

![Schematics](/docs/kicad/schematics/toy-usb-pd-power-manager.svg)

[Schematics PDF here.](/docs/kicad/schematics/toy-usb-pd-power-manager.pdf)


### PCB layout

I stopped working on the PCB layout in order to re-evaluate the project and make a review in general, but all the constituent parts are laid out and routed (with some exceptions). For example, the switch-mode DC/DC converters are laid out according to datasheet suggestions, but are not connected to the overall board.
