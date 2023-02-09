# EasyLight_PCB

A simple and reusable PCB that can power and controls lights both from USB and an internal battery. This will be integrated into multiple other projects of mine. 

The main features are:
1. Uninterrupted power supply for lights that can hot-swap between USB power and battery backup
2. Integrated charging, protection and load sharing/power path selection of a Li-Ion battery
3. Control of WS2812B, SK6812 and similar digital RGB(W) lights
4. Easy inputs via external buttons
5. Slider to turn off the lamp without disconnecting battery charge

Open like regular in KiCad 6. Look into the exports under [Actions/KiCad Exports](https://github.com/ModischFabrications/EasyLight_PCB/actions/workflows/kicad-exports.yml) to see schematics, renderings, exports and more. 

It's designed to be mostly assembled by JLC-PCB's SMT Assembly service, which should make replicating everything pretty easy. The final price should be around 50€ for 10 boards with all parts included, you won't get much cheaper if you assemble yourself. Ordering fewer boards is rarely economical: 5 boards cost 42€, because the a part of the cost is "extended part" reel switching for 3$ each. Feel free to deselect parts you don't need from PCBA, connectors are a good choice. You might get away ordering fewer if you heavily reuse very few components or limit yourself to basic parts. 

This project took around 40 hours of work to finish, from part research to finished PCB. Take some time to work through all of it if you want to replicate anything, it's not for the faint-hearted.

## Decisions

### Programming & Debugging
It's not easy to program microcontrollers without a huge markup in size and/or cost. 

The easiest solution is to have a socket on the PCB to program the IC externally and plug it in afterwards. There are no electrical problems with the board itself, it's easy to exchange once killed and you can use basic components. 
There are problems though:
- Stacking PCB - Socket - IC takes at least 10mm in height
- Additional socket to place
- Only THT DIP chips supported

The default option for better boards is to use an "In System Programmer", or ISP in short, that connects directly to the chip. There are some considerations on connected IO, especially on pins usually meant as an input, but it's independent of the actual location and package of the IC. Do note that they can be connected in reverse, which will usually kill everything on your board. 

The traditional AVR-ISP is a 6 pin connector with 2.54mm pitch on the board. This is compatible to generic hookup wires used for every arduino ever made, but needs a bit of space because of the large pitch and THT parts. 

It's possible to use finer connectors with a bit more features, e.g. JST-SH-6, but that will lock you out of the generic programmers and poses additional costs. It will however make programming more reliable and enjoyable.

More elegant solutions depend on "pogo pins" and exposed copper test pads. It's compact and doesn't need any additional parts on the board, but has it's problems with alignment. Hint: Don't try to hold solid pins onto these pads and expect programming to work, you need a bit of spring tension to keep 2x3 pins connected. Pogo pins with large cone heads can actually be used on plated holes meant to solder in AVR-ISPs as well, but are more difficult to find. Another trick is to use vias in the pads itself to align everything, but make sure not to tent/plate them. Make sure to pierce contaminants for boards with cheap plating. 

There is some [great research](http://www.auelectronics.com/forum/index.php?topic=42.0) on these, especially on the [tips used](https://electronics.stackexchange.com/questions/360050/implementing-a-good-connection-for-pogo-pins-in-eagle), [and plating](https://electronics.stackexchange.com/questions/393531/what-pcb-plating-do-i-need-to-use-exposed-copper-pads-with-pogo-pins). Some even [sell premade ones](https://www.tindie.com/products/madworm/tiny-avr-isp-pogo-pin-programming-adapter/), take a look at them for inspiration. 

See [Tag-Connect](https://www.tag-connect.com/) for a commercial solution that's solid and prevents reverse connections. It [seems to have some problems](https://ludzinc.blogspot.com/2013/03/missed-it-by-that-much.html) though and is a very fine 1.27mm pitch. If you buy one, take the solution without mounting holes, the other one is huge.

TLDR: 
- Use the right pogo:
  - cone (1.4mm) for plated holes/vias, can use pads
  - crown for pads with solder
  - round for gold plated pads with repeated contacts
- use pads with 1.27 or 2.54mm pitch
- add via/plated hole to pad to make alignment easier
- use gold plating or apply solder to pads that will be contacted often
- add alignment hole if you feel fancy
-> generic 2.54mm THT pin header holes (ID=1mm, OD=1.7mm) are fine

### Power planning
These boards are planned for two use cases:
1. Docked: Permanent wall power with battery as backup for short durations
2. Mobile: Usually operated by battery, charged whenever needed

The whole system is planned to stay well below 1.5A@5V with either high intensity battery runtimes of <8h or standby battery runtime of 20+ days. 
Battery thresholds are conservative, health is more important than maximized capacities. Buy a bigger battery if you want more runtime. 

Wall Power input is assumed to be larger than power demand by the load + battery, reduce charging current if that is not the case. This usually means >=500mA, depending on use case. 

### Battery Level/Charge Detection
ATtiny85 can read VCC via internal ADC, which makes battery voltage readings easy due to the unregulated output.
There is a slight drop from the power path and other peripherals, but it's probably smaller than the variance of the ADC itself. 

Most chargers also offer a CRG/STAT output, see https://electronics.stackexchange.com/a/624987/217104 on how to read that. 

This can be used to detect 4 states:  
1. Battery empty
2. Battery charging
2. Battery discharging
2. Battery full

### Load sharing/Power Path
The usual solution to charge the battery of a device is to cut off the output, either mechanically with a switch (BAT || SUB) or with a P-FET. This is easy, but is incompatible for UPSs and similar "always-on" use cases.

[Load sharing](https://www.microtype.io/lithium-ion-battery-charger-circuit-load-sharing/) is needed to power the device while charging the battery. Yes, the ubiquitous TP4056 boards [do it wrong](https://www.best-microcontroller-projects.com/tp4056.html). 

Cheap solutions use two diodes, most a P-FET in the battery path, better ones one for each. Add doubled up FETs to prevent body diode conduction and the BOMs balloons up. This application can live with the default solution and just one FET, because input is always higher voltage (/2) and unlimited (/2). Do note the "backwards" P-FET, it's to save the second transistor while blocking reverse current. 

Ideal diodes or PowerMUX would be ideal, but are more expensive. Have a look at [this explanation from TI](https://www.ti.com/lit/an/slvae57b/slvae57b.pdf), especially Figure 8-1/9-3. 

### Button inputs
Using 1 input per Pin is easy to program, but wasteful for MCUs with few IOs. Chaining button to an analog input is much more efficient and can support up to 32 buttons per pin at very detectable 32mV increments. Detecting sequential presses is easy, parallel is more difficult. Smarter chaining could also permit multiple buttons at once with decades (1K, 2K, 4K, ...). See [this explanation](http://www.ignorantofthings.com/2018/07/the-perfect-multi-button-input-resistor.html) or [this one](https://github.com/bxparks/AceButton/blob/develop/docs/resistor_ladder/README.md). 

[This simulation](https://www.falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgBnEFFG73-nxR48UcCABmAQwA2HBmwButGqOwIUK0WEJU9tKkn0wEbAE7gMhEOs1gr3EWJ5xz4YTY3vRw0VTDwbjx8toKOfrQYrlz21sGWcU7+krLynN5hYB6+YhDScgoAxjZ4IV7YpdwYmnqw8BAwcA117BYVQgLt4LpiuIEA7iV8OlQ0qt16bADmWp6aY2qEEZBsgwtzQxsrg-GhsVurCRtZal7bGfH78edXAvHXh3hUl9kCNx6hTwdcmJr31eFcikCkEAaFfoD-PAVm00BtsHCcs80NEbIjOoikuJ8mlBgjkUIAQ8dmDynDQud8WEusTNqEqRTDhDwZ0zky7nciW9HlR6ZpGXi-gIvg8uCLhc8sXlUgoLF9PpKIjQoukEdS-lLgWlYfCNREXIddD4BIQEHFuYNTdZQkaDoMrKdNFEyjVDs6wg6wucrRt3QKQO74j7RQHIB0+IHNTjZaGXbHIfxAhYfTazQmAoEuMGTWmkdiZY81aFPYyZngi15y-ylmJzp6RiAqxMoG5PfF6z0oa5iiWvG2AbVoc0mlAWm5bTb8Anla5lE3PmqG-oaIZa6PTFmp0Gt1GCwB7bogQh8UaQUjURrwMhWlBGbiH7BsA-YcjHsQr89GOpwa9m2-iflyEfIA) is also somewhat useful if you want to play around with values. 
Advice: Keep Pullup equal to the smallest resistor and keep their resistances somewhat close together, so that the pressing the largest in parallel to the smallest is still detectable. The internal pullup is fine for one-offs, but varies too much for dependable values, better use an external one. 

Search vor *DAC*, *R-2R* or *resistor ladder* for more references. 

3 Buttons are integrated into the PCB by default, but there are more possibilities by soldering wires into the empty connector next to the buttons to use external ones. 

### Test Points & Extensions
I would like to use this schematic for as much as possible, so there are a few "generics" that might not be always needed. 

### LiPo instead of LiFePo4
LiFePo4 is a great chemistry because it is 3.0V to 3.6V by default, which makes voltage regulators for the ESP32 (also 3.0 to 3.6V) obsolete. 
Sadly, they aren't readily available in small pouches, the smallest ones have AA format, which is too big for our button. 

LiPo has a wide variety of form factors and fits our geometry better, but is 4.2V to 2.7V. 
Discharging to 3V only uses roughly 80% of power. Keeping the minimum voltage high is also healthier for the battery. 
A boost-converter could discharge lower, but it would burn more power when idle. 

This means we need a highly efficient voltage regulator or operate off of a wide range of voltages. 
It needs low I_Q, low V_DO while offering V_OUT of ~3V (to allow for more dropout) at <= 500mA. 
I chose unregulated power because all parts can handle it and it reduces complexity while increasing efficiency. 

Side note: Disposable vapes are a cancer to the environment, but have "free" LiPos in them, which are great to use here. 

### Safety
Short-circuit-resistance of the power leads is archieved using a polyfuse. 
Battery safety is handled by a dedicated chip on the PCB (there is an additional one in most batteries). 

Input ESD & Overvoltage should be handled by the TVS diode. A bidirectional one was chosen to enable the polarity protection later without the TVS interfering. 

Reverse Polarity protection via P-FET according to [this great video](https://www.youtube.com/watch?v=IrB-FPcv1Dc&t=184s).


## Part Selection
Everything is optimized for JLCPCB, because they are great for (cheap) hobby projects. 
[JLCPCB Search](https://jlcpcb.com/parts/) was used for simple part lookup, usually with "basic parts".
[LCSC Search](https://www.lcsc.com/) should show the same parts from the same database, but doesn't add much. 
[JLCParts](https://yaqwsx.github.io/jlcparts/#/) was used to find more complex parts. 
[easyeda2kicad](https://github.com/uPesy/easyeda2kicad.py) to import parts into KiCAD. You should use generic footprints & symbols for simple parts instead. 

Basic parts are given preference, because they are cheaper for assembly service. 

Footprints: 
- THT are dead, SMD is king
- QFN are pain, BGA impossible
- SOT-23-5, SOP-8, SMA and similar are good choices
- 0603 or bigger
-> Soldering by hand should be possible, even if reflow is easier

Generic resistors and capacitors use the [E6 series](https://en.wikipedia.org/wiki/E_series_of_preferred_numbers) for best compatibility.

Some footprints might use minor variations for parts that I already have, sorry for that. 
Others are based on availability and consistency, e.g. 22uF caps are the largest capacity I could get as a cheap basic part in 0603. 

### Microcontroller
Needs: 
- No WiFi
- Small size
- >=3 GPIOs
- Easy integration with PlatformIO
- Works with unregulated 3..5V

Choices:
- Full-size Arduinos are wasted energy, space and cost
- STM32 are industrial use, but not great to integrate into hobbyist projects with arduino libs
- ESP\* are wasteful without WiFi and are 3.3V only
- **ATtiny\*** fit the bill, ATtiny85 to get the most out of the form factor

### Lights
RGBs are easy and cheap to get, but RGBW makes white much easier and cleaner, maybe even more efficient. 
Analog stripes and discrete LEDs were considered, but increase part cost, need more IOs and offer fewer features. 

Use WS2812B for cheap RGB and SK6812 for pretty RGBW. 
APA102/SK9822 are a better version of the WS2812B with 4 instead of 3 wires, which makes great refresh rates and easy timings for complex programs. Imho, they are not necessary for simple lights with few chips. 
I added support for all types in the pinout regardless, leave Pin 3 empty for other chipsets. 

See [FastLED Chipset reference](https://github.com/FastLED/FastLED/wiki/Chipset-reference) for a few more suggestions or [Qwiic LED Stick](https://cdn.sparkfun.com/assets/b/c/9/c/1/Qwiic_LED_Stick_-_Schematic.pdf) for a great hookup reference. 

Signal lines have series resistors (300..500 Ohm) to match impedances and protect the IC from overcurrent, see [this great guide from Adafruit](https://learn.adafruit.com/adafruit-neopixel-uberguide/best-practices) and [this great image](https://forum.arduino.cc/t/arduino-ws2812b-data-pin-resistor/533031/3). They will prevents shorts from overloading the IO while keeping a low profile in relation to the huge impedance of the LEDs. 

Close to the connector are some huge capacitors as well to stabilize LED voltages, but they are smaller than recommended due to the low number of LEDs and my disregard for (aging) electrolyte capacitors. 

### AIO Power Management Chip (disqualified)
IP5306 is a great chip that can do everything at once, including:
1. Protection (with reasonable cutoffs)
2. Charging
3. Powerpath
4. Boost to 5V

But it has a few quirks due to it being designed for powerbanks:
1. Mostly chinese documentation, [small EN translations]()
2. Always boosts to 5V (wasteful, we can handle unfiltered here)
3. **Needs a key press to start and >=50mA to continue operation**
4. Indicators focused on LEDs with multiple variations, making IO readout difficult

A few of these problems can be mitigated by using the I2C variant and [this](https://github.com/bheesma-10/IP5306_I2C) library, but that is still not enough: 

1. Needs 3 additional IOs. 
2. No voltage detection, just basic status
3. More difficult to source from distributors

### TP4056 LiPo Board (disqualified)
Extend with power path, drop in FS312F-G, reduce charging current to <=600mA. Okay, but not great. 

### Charge Controller, Battery Protection, Load Sharing/Power Path
Dedicated parts can be optimised better for these use cases. 
[This video](https://www.youtube.com/watch?v=-SJbdPvgQnE&list=PL68C2022EF3AFB717) is a great summary on what to look for.

#### BMS
BMS ≠ Charger! These components are safety devices and just cut the power. 
They are usually integrated in packs (look for "protected"), but a dedicated one is safer and has healthier thresholds. 
They are usually tiny and behind tape, made with only a DW01-P + 2 MOSFETs. 

- [FS312F-G](https://www.lcsc.com/product-detail/Battery-Management-ICs_Fortune-Semicon-FS312F-G_C82736.html)(0.15€) popular drop-in replacement for DW01-P with better thresholds
  - usually acompanied by [FS8205A](https://www.lcsc.com/product-detail/MOSFETs_FUXINSEMI-FS8205A_C908265.html)(0.06€)
- [**XB8089D0**](https://www.lcsc.com/product-detail/Battery-Management-ICs_XySemi-XB8089D0_C2760005.html)(0.16€) similar, but with **integrated FETs** with compatible performance

#### Charger 
- [TP4056](https://www.lcsc.com/product-detail/Battery-Management-ICs_UMW-Youtai-Semiconductor-Co-Ltd-TP4056_C725790.html)(0.08€!) always available and default for chinese PCBs, but little [(EN)](https://dlnmh9ip6v2uc.cloudfront.net/datasheets/Prototyping/TP4056.pdf) documentation and bigger package (SOP-8). 1A, 1.5%
- [MCP73831-2\*](https://www.lcsc.com/product-detail/Battery-Management-ICs_Microchip-Tech-MCP73831T-2ACI-OT_C424093.html)(0.76€) widely available and often used in open projects. Great documentation and package (SOT-23-5). 0.5A, 0.75%
- [**BQ21040**](https://www.lcsc.com/product-detail/Battery-Management-ICs_Texas-Instruments-BQ21040DBVR_C202311.html)(0.68€) often recommended, with great documentation. 0.8A, 1%, Timer, NTC support. 
  - Other BQ24\* have difficult packages and are more expensive
- LTC\* are nice in theory, but surprisingly expensive from China (>1.50€)
- MCP73871 is an awesome solution with integrated power path, switching and more, but only come in QFN-20
- ...

#### Power Path 
Power Path Controllers: 
- BQ2403x : AIO, but QFN only
- ...
-> Not feasible for cheap hobby projects; would use MCP73871 if going QFN

Ideal Diodes/Load Switch/PowerMux:  
More complex ICs with internal transistors, but great performance. Usually more expensive, build for bigger loads.
- [LTC4412](https://www.lcsc.com/product-detail/Gate-Drive-ICs_Analog-Devices-LTC4412ES6-PBF_C107968.html)(2,73€!!): good docs, but controller only and stupid expensive
- [LM66100](https://www.lcsc.com/product-detail/Power-Distribution-Switches_Texas-Instruments-LM66100DCKR_C2869734.html)(0.54€): well known and great performance, okay price for ID (0.5uA, 0.01V, 1.5A)
- [MT9700](https://www.lcsc.com/product-detail/Power-Distribution-Switches_XI-AN-Aerosemi-Tech-MT9700_C89855.html)(0.05!) : Common part, crazy cheap, okay performance. Not sure if proper diode? (1A)
- [MAX40203](https://www.lcsc.com/product-detail/Power-Distribution-Switches_Maxim-Integrated-MAX40203AUK-T_C2653639.html)(0.65€) : great fit (1A)
- [LTC4415](https://www.lcsc.com/products/Power-Distribution-Switches_969.html?keyword=LTC4415)(6.44€) Dual package, exact match, nice! Huge package, 16 leads, expensive :/ (4A)
- [LM66200](https://www.lcsc.com/product-detail/Power-Distribution-Switches_Texas-Instruments-LM66200DRLR_C3235556.html)(0.43€) Dual package, cheap, perfect use-case. Tiny package (SOT-583) :/
-> Use two of these if you need high power, high efficiency. Not worth it for small runs, use default solution of P-FET + D

P-FET:  
I > 1.5A, R_DSon < 60mR and V_GS < 0.5 .. 1.5V. 
R is important to loose as little battery energy as possible. Extremely low V_GS could switch too soon due to diode, too high would increase R_DSon.
- [DMP1045U](https://www.lcsc.com/product-detail/MOSFETs_Diodes-Incorporated-DMP1045U-7_C177033.html)(0.37€) popular, great stats and package (4A 31mΩ@4.5V)
- [DMP2035U](https://www.lcsc.com/product-detail/MOSFETs_Diodes-Incorporated-Diodes-Incorporated-DMP2035U-7_C110499.html)(0.12€) good values, fair price, good documentation (3.6A 35mΩ@4.5V)
- [WST2339](https://www.lcsc.com/product-detail/MOSFETs_Winsok-Semicon-Winsok-Semicon-WST2339_C148354.html)(0.12€) great (7.1A 19mΩ@4.5V)
- [AP2335](https://www.lcsc.com/product-detail/MOSFETs_ALLPOWERShenZhen-Quan-Li-Semiconductor-ALLPOWERShenZhen-Quan-Li-Semiconductor-AP2335_C2828582.html)(0.08€) great (7A 19mΩ@4.5V)
- [SI2301S](https://www.lcsc.com/product-detail/MOSFETs_MDD-Microdiode-Electronics-SI2301S_C427389.html)(0.01€!!) with good stats for great price (2.3A 70mΩ@4.5V)
- [**AO3415**](https://www.lcsc.com/product-detail/MOSFETs_Guangdong-Hottech-Guangdong-Hottech-AO3415_C181094.html)(0.04€) good values and very available, somewhat known (4A 41mΩ@4.5V)
- ...
Most should be pin compatible, feel free to exchange however wanted. 

Diode:  
I > 1.5A, I_r < 50uA and V_f < 0.4V.
Problem: I_r and V_f usually counter each other. Former is important [to keep the FET closed](https://electronics.stackexchange.com/questions/293353/load-sharing-charging-circuit-for-lithium-polymer), latter for efficient wall power because of the waste heat. 
- MBR120 : well known, but max 1A, high leakage (>=1mA)
- [SS14](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_MDD-Microdiode-Electronics-SS14_C2480.html)(0.01€): Well known, basic part, bad values (550mV@1A, 0.5..10mA)
- [**SS34**](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_MDD-Microdiode-Electronics-SS34_C8678.html)(0.03€) : Default, basic part, okay values (500mV@3A, 0.2..6mA)
- [SS54](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_Slkor-SLKORMICRO-Elec-SS54_C513478.html) ? (550mV@5A, 0.5..20mA)
- SB220 : known, but THT
- 1N5817 : somewhat known, but THT. also with 8/9 postfix
- [MBR0520LT1G](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_onsemi-MBR0520LT1G_C23848.html)(0.10€) : (385mV@500mA, 0.075..5mA)
- [MBRS320](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_onsemi-MBRS3201T3G_C236150.html)(0.37€) : expensive, but good values for a regular diode
- [SL22-E3/52T](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_Vishay-Intertech-SL22-E3-52T_C142616.html)(0.37€) expensive for it's value (440mV@2A, 0.4..10mA)
- SD103AWS : great at blocking, not great voltage drop (600mV@200mA, <0.05uA)
- [RSX201VAM30](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_ROHM-Semicon-RSX201VAM30TR_C308755.html)(0.08€) seems like a good pick? (460mV@1.5A, 0.007 .. 0.03mA)
- [PMEG3010ER](https://www.lcsc.com/product-detail/Schottky-Barrier-Diodes-SBD_Nexperia-PMEG3010ER-115_C456106.html)(0.11€) (360mV@1A, 0.6 .. 1.5mA)
- ...



