# CallbackButton_PCB

This is the PCB for [CallbackButton](https://github.com/ModischFabrications/CallbackButton).

Open like regular in KiCad. Look into the docs zip under [Actions/KiCad Exports](https://github.com/ModischFabrications/CallbackButton_PCB/actions/workflows/exports.yml) to see renderings. 

This schematic could be soldered manually onto a perfboard, but it's preferrably to order a PCB instead. 

## Decisions

### Test Points & Extensions
We would like to use this schematic as long as possible, so simple connectors for I2C make future additions easier. 

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

Introducing series resistances into every single connection could make it safer, but I can't be arsed at the moment. Active-Low or High doesn't really matter either, a cross between adjacent leads would result in one overloading regardless. Feel free to insert ~300 ohm everywhere if you want to be safe. 
