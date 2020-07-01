---
layout: post
title: "A Nixie Tube Clock Design"
author: "Jose Logreira"
categories: nuvitron
tags: [sample]
image: circuit_front.jpeg
---


Digital clocks are fun electronic projects. Add some 160V [nixie tubes](https://en.wikipedia.org/wiki/Nixie_tube) and you get an extra layer of vintage art.

I worked for __[Nuvitron](https://nuvitron.com)__ in the past, and I took part in the design and programming of their base circuit for the [Nixie Tube Clock lineup](https://nuvitron.com/the-vintage-electronics-shop). They were kind enough to allow me to document most of the things. This post is just a summary. I created a [GitHub Page](https://joselogreira.github.io/nixie_clock/) for the project, and also all electronic design files are also in a [GitHub repo](https://github.com/joselogreira/nixie_clock).

<iframe width="560" height="315" src="https://www.youtube.com/embed/HfNS7TEvCeo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The clock uses the type [IN-12](http://www.tube-tester.com/sites/nixie/data/in-12a.htm) tubes. There're plenty of them on ebay. This clock has had three major iterations, with the one described being the last one. To this date, the production-version hardware has not changed, although I've done some firmware code refactoring. Learning better C programming techniques over time made me realize that some of the code structure can still be improved, so it's still open to changes.

Some of the interesting things about this nixie clock implementation are described below

## Small PCB footprint

Main PCB is only 12.5 x 4.5cms. The only missing components are the three user buttons and power connector, all located in a small secondary board in the back of the clock. PCB was designed using [KiCAD](https://kicad-pcb.org/). Low voltage traces had a minimum separation of 8mils and high voltage traces, almost 24mils. Minimum footprints pitch is 0.5mm (the MCU pads). Most of the passive SMD components are 0603, all other components are bigger, so this allows the whole circuit to be hand-soldered with some practice and a regular chisel tip soldering iron.

![pcb front](/assets/img/pcb_front.png)
![pcb back](/assets/img/pcb_back.png)

It's a double layer design, with most of the SMD components at the top, and only the RGB LEDs at the bottom to illuminate the tubes from their back to give nice background colours.

![leds](/assets/img/nc_leds.jpg)

## No external RTC

It is common to find independent [Real Time Clock](https://en.wikipedia.org/wiki/Real-time_clock) (RTC) chips to keep track of time. These chips have back-up battery so when external power is removed, the microcontroller shuts down but the RTC is still ON, with the advantage of consuming very little current (200mAh coin cell battery may last for a few years).

This design uses the [ATmega324PB](http://ww1.microchip.com/downloads/en/DeviceDoc/40001908A.pdf) microcontroller. It's optimized for low power consumption. It runs at 2MHz clock (relatively slow frequency), and can operate down to 1.8V supply voltage. In addition, it includes an internal RTC as peripheral, which is basically a timer operating asynchronously with an external 32.768Hz quartz crystal. These characteristics allow to discard an external RTC and use the different [sleep modes](https://joselogreira.github.io/nixie_clock/docs/sleep/) to keep track of time when there's no external power (using a back-up battery, of course).

![leds](/assets/img/nc_mcu.jpg)

## Factory and user testing

A few batches of this circuit were produced. It was big enough to be willing to automate most of the testing, but small enought not to invest sophisticated hardware and software setups for testing. So, the in-between choice was to make use of some of the additional GPIO and memory of the MCU to desing a self-test mechanism: a way to measure system voltages, to estimate proper crystal frequencies and to easily test RGB LEDs and sounds.

The circuit factory does not assemble the nixie tubes. That is a subsequent step during the clock assembly, so testing the tubes should also be part of the firmware, either for the clock assembler or the user.

The factory self-test relies on the use of a USB-to-serial converter, to display test results on a PC virtual serial port terminal. It looks for three things:
* __System voltages:__ Boost output voltage, input voltage, MCU voltage, boost controller voltage.
* __Crystals timing:__ It uses the 16MHz ceramic resonator as a reference to measure the 32,768KHz quartz crystal precission. 
* __User feedback of LEDs and buzzer:__ It switches between all LED colours and asks the operator whether they are correctly displayed. Then it plays some musical notes and asks whether they are properly heard.

The system isn't fools' proof, since it depends on operator feedback, but no doubt it's better than doing no testing at all, or spending hundreds of dollars on specialized test equipment (remember, these were small production batches).

![NC testing](/assets/img/nc_testing.png)

Above is a screenshot of the voltages testing sequence, where all voltages are tested  three times, with the boost converter enabled and disabled. All test results are stored in internal EEPROM memory, so they can be later downloaded for further failure analysis or just tracking whether the circuits were properly tested or not.

When the user or the clock assembler wants to test the nixie tubes and LEDs, a test sequence can be entered at start-up by pressing a button. It cycles through all the tubes' digits and LED colors, so a visual inspection is straightforward. The image below shows the circuit performing that sequence.

![NC user test](/assets/img/nc_user_test.jpeg)

## Low power consumption

Nixie tubes require high voltage, but low power. Each tube's digit consumes less than 2mA @ 160V. Besides, the [multiplexing](https://joselogreira.github.io/nixie_clock/docs/multiplexing/) scheme allows for only one tube to be active at a time. Apart from the tubes, the full-on RGB LEDs are the other big consumers of current. Lab measurements show a maximum current consumption of 70mA @ 12V input voltage. __That is, 0.84W!__. That makes me think that it could run out of a small lead-acid battery or something similar!, but that in itself is just another project.