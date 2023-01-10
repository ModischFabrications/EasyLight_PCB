# EasyLight_PCB

***WIP!***

This is the PCB for [TODO](https://github.com/ModischFabrications/TODO).

Open like regular in KiCad. Look into the docs zip under [Actions/KiCad Exports](https://github.com/ModischFabrications/EasyLight_PCB/actions/workflows/exports.yml) to see renderings. 

This schematic could be soldered manually onto a perfboard, but it's preferrably to order a PCB instead. 

## Decisions

### Charge Controller, Battery Protection, Load Sharing/Power Path

#### AIO Chip (disqualified)
IP5306 is a great chip that can do everything at once, including:
1. Protection (with reasonable cutoffs)
2. Charging
3. Powerpath
4. Boost to 5V

But it has a few quirks due to it being designed for powerbanks:
1. Mostly chinese documentation, [small EN translations]()
2. Always boosts to 5V (we can handle unfiltered here)
3. **Needs a key press to start and >=50mA to continue operation**
4. Indicators focused on LEDs with multiple variations, making IO readout difficult

A few of these problems can be mitigated by using the I2C variant and [this](https://github.com/bheesma-10/IP5306_I2C) library, but that is still not enough: 

1. Needs 3 additional IOs. 
2. No voltage detection, just basic status
3. More difficult to source from distributors

#### Dedicated Parts
BMS:
- [FS312F-G](https://www.lcsc.com/product-detail/Battery-Management-ICs_Fortune-Semicon-FS312F-G_C82736.html)(0.15€) popular drop-in replacement for DW01-P with better thresholds
  - usually acompanied by [FS8205A](https://www.lcsc.com/product-detail/MOSFETs_FUXINSEMI-FS8205A_C908265.html)(0.06€)
- [**XB8089D0**](https://www.lcsc.com/product-detail/Battery-Management-ICs_XySemi-XB8089D0_C2760005.html)(0.16€) similar, but with **integrated FETs** with compatible performance

Charger: 
- [TP4056](https://www.lcsc.com/product-detail/Battery-Management-ICs_UMW-Youtai-Semiconductor-Co-Ltd-TP4056_C725790.html)(0.08€!) always available and default for chinese PCBs, but little [(EN)](https://dlnmh9ip6v2uc.cloudfront.net/datasheets/Prototyping/TP4056.pdf) documentation and bigger package (SOP-8). Allegedly up to 1A. 
- [MCP73831-2\*](https://www.lcsc.com/product-detail/Battery-Management-ICs_Microchip-Tech-MCP73831T-2ACI-OT_C424093.html)(0.76€) widely available and often used in open projects. Great documentation and package (SOT-23-5)
- [BQ21040](https://www.lcsc.com/product-detail/Battery-Management-ICs_Texas-Instruments-BQ21040DBVR_C202311.html)(0.68€) with great documentation
  - Other BQ24\* have difficult packages and are more expensive
- LTC\* are nice in theory, but surprisingly expensive from China (>1.50€)


Power Path P-FET (all pin compatible): 
- [DMP1045U](https://www.lcsc.com/product-detail/MOSFETs_Diodes-Incorporated-DMP1045U-7_C177033.html)(0.37€) popular, great stats and package (4A 31mΩ@4.5V)
- [DMP2035U](https://www.lcsc.com/product-detail/MOSFETs_Diodes-Incorporated-Diodes-Incorporated-DMP2035U-7_C110499.html)(0.12€) midway (35mΩ@4.5V, 3.6A)
- [**AO3415**](https://www.lcsc.com/product-detail/MOSFETs_Guangdong-Hottech-Guangdong-Hottech-AO3415_C181094.html)(0.04€) good values and very available, somewhat known (41mΩ@4.5V,4A)
- [**WST2339**](https://www.lcsc.com/product-detail/MOSFETs_Winsok-Semicon-Winsok-Semicon-WST2339_C148354.html)(0.12€) great (7.1A 19mΩ@4.5V)
- [AP2335](https://www.lcsc.com/product-detail/MOSFETs_ALLPOWERShenZhen-Quan-Li-Semiconductor-ALLPOWERShenZhen-Quan-Li-Semiconductor-AP2335_C2828582.html)(0.08€) great (19mΩ@4.5V,5A)
- [SI2301S](https://www.lcsc.com/product-detail/MOSFETs_MDD-Microdiode-Electronics-SI2301S_C427389.html)(0.01€!!) with good stats for greate price (2.3A 70mΩ@4.5V)
- A few others with worse stats or higher price

### Battery Level/Charge Detection
ATtiny85 can read VCC via internal ADC, which makes battery voltage readings easy due to the unregulated output. There is a slight drop from the power path, but it's probably smaller than the ADC variance. 

Most chargers also offer a CRG/STAT output, see https://electronics.stackexchange.com/a/624987/217104 on how to read that. 

### Button inputs
Using 1 input per Pin is easy to program, but wasteful for MCUs with few IOs. Chaining button to an analog input is much more efficient and can support at least 32 buttons per pin at very detectable 32mV increments. Smarter chaining could also permit multiple buttons at once with decades (1K, 2K, 4K, ...). See http://www.ignorantofthings.com/2018/07/the-perfect-multi-button-input-resistor.html

### Test Points & Extensions
I would like to use this schematic for as much as possible, so there are a few "generics" that might not be always needed. 

### LiPo instead of LiFePo4
LiFePo4 is a great chemistry because it is 3.0V to 3.6V by default, which makes voltage regulators for the ESP32 (also 3.0 to 3.6V) obsolete. 
Sadly, they aren't readily available in small pouches, the smallest ones have AA format, which is too big for our button. 

LiPo has a wide variety of form factors and fits our geometry better, but is 4.2V to 2.7V. 
Discharging to 3V only uses roughly 80% of power. Keeping the minimum voltage high is also healthier for the battery. 
A boost-converter could discharge lower, but it would burn more power when idle. 

This means we need a highly efficient voltage regulator. 
It needs low I_Q, low V_DO while offering V_OUT of ~3V (to allow for more dropout) at <= 500mA. 

### Electrical
Short-circuit-resistance of the power leads is archieved using a polyfuse. Resistance of the inputs is build-in, as they are high impedance, but the outputs are at risk. 
