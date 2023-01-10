# EasyLight_PCB

***WIP!***

This is the PCB for [TODO](https://github.com/ModischFabrications/TODO).

Open like regular in KiCad. Look into the docs zip under [Actions/KiCad Exports](https://github.com/ModischFabrications/EasyLight_PCB/actions/workflows/exports.yml) to see renderings. 

This schematic could be soldered manually onto a perfboard, but it's preferrably to order a PCB instead. 

## Decisions

### Button inputs
Using 1 input per Pin is easy to program, but wasteful for small processors. Chaining button to an analog input is much more efficient and can support at least 32 buttons per pin at very detectable 32mV increments. Smarter chaining could also permit multiple buttons at once with decades (1K, 2K, 4K, ...). See https://rayshobby.net/wordpress/multiple-button-inputs-using-arduino-analog-pin/

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
