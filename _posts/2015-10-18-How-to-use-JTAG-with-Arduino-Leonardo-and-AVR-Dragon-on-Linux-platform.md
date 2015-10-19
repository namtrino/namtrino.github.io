---
layout: post
title: How to use JTAG with Arduino Leonardo and AVR Dragon on Linux platform
excerpt: "A step-by-step example to setup JTAG on Arduino-Leonardo with AVR Dragon on Linux Platform"
tags: [AVR, Arduino-Leonardo, Linux, JTAG, AVR-Dragon]
modified: 2015-10-18
comments: true
---

##Hi
The step of an example below is tested with Ubuntu-14.04, x86_64 host machine.

####1. Read “Memory Programming” section on the link below
[atmega32u4 datasheet](http://www.atmel.com/Images/Atmel-7766-8-bit-AVR-ATmega16U4-32U4_Datasheet.pdf)

####2. Check JTAGEN by reading FUSE

  {% highlight console %}
  sudo avrdude -v -P usb -c dragon_isp -p m32u4 -U lfuse:r:low_fuse_default.hex:h -U hfuse:r:high_fuse_default.hex:h
  {% endhighlight %}

####3. Evaluate high FUSE
  {% highlight console  %}
  $ cat high_fuse_default.hex
  0xd8
  $ echo "obase=2; ibase=16; D8" | bc
  11011000
  $ echo "obase=16; ibase=2; 00011000" | bc
  18
  $ echo 0x18 > jtag_high_fuse.hex
  $ cat jtag_high_fuse.hex
  0x18
  $
  {% endhighlight %}

####4. Program new FUSE to target
  > You cannot use the hex value from file to set the FUSE
  > as shown below
  $sudo avrdude -v -P usb -c dragon_isp  -p m32u4 -U  hfuse:w:**__**jtag_high_fuse.hex:h**__**
  so, you need to set the hex value directly
  {% highlight console  %}
  $sudo avrdude -v -P usb -c dragon_isp  -p m32u4 -U  hfuse:w:0x18:m
  .
  .
  .
  avrdude: AVR device initialized and ready to accept instructions

  Reading | ################################################## | 100% 0.15s

  avrdude: Device signature = 0x1e9587
  avrdude: safemode: lfuse reads as FF
  avrdude: safemode: hfuse reads as D8
  avrdude: safemode: efuse reads as CB
  avrdude: reading input file "0x18"
  avrdude: writing hfuse (1 bytes):

  Writing | ################################################## | 100% 0.15s

  avrdude: 1 bytes of hfuse written
  avrdude: verifying hfuse memory against 0x18:
  avrdude: load data hfuse data from input file 0x18:
  avrdude: input file 0x18 contains 1 bytes
  avrdude: reading on-chip hfuse data:

  Reading | ################################################## | 100% 0.05s

  avrdude: verifying ...
  avrdude: 1 bytes of hfuse verified

  avrdude: safemode: lfuse reads as FF
  avrdude: safemode: hfuse reads as 18
  avrdude: safemode: efuse reads as CB
  avrdude: safemode: Fuses OK (H:CB, E:18, L:FF)

  avrdude done.  Thank you.

  $
  {% endhighlight %}

####5. Code & Build
  {% highlight console  %}
  $ cat main.c
  #include <avr/io.h>
  #include <util/delay.h>

   void main(void){
   DDRD |= (1<<PD4);

   for(;;){
    PORTD |= (1<<PD4);
    _delay_ms(500.00);

     PORTD &= ~(1<<PD4);
    _delay_ms(500.00);
   }
  }
  $ avr-gcc -g -Os -DF_CPU=16000000UL -mmcu=atmega32u4 main.c
  $ file a.out
  a.out: ELF 32-bit LSB  executable, Atmel AVR 8-bit, version 1 (SYSV), statically linked, not stripped
  $ avr-objcopy -O ihex a.out output.hex
  $
  {% endhighlight %}

####6. Flash output.hex to target
  {% highlight console  %}
  sudo avrdude -v -P usb -c dragon_isp -p m32u4 -U flash:w:output.hex
  {% endhighlight %}

####7. Wiring JTAG
  [Link1](http://www.atmel.com/webdoc/avrdragon/avrdragon.using_ocd_physical_jtag.html)
  [Link2](http://www.atmel.com/webdoc/atmelice/atmelice.using_ocd_physical_jtag.html)

####8. Test JTAG connection
  {% highlight console  %}
  $ sudo avarice --dragon --jtag usb
  AVaRICE version 2.11, Jan 17 2014 02:51:59

  Defaulting JTAG bitrate to 250 kHz.

  JTAG config starting.
  Found a device: AVRDRAGON
  Serial number:  00:a2:00:06:90:52
  Reported JTAG device ID: 0x9587
  Configured for device ID: 0x9587 atmega32u4
  JTAG config complete.
  {% endhighlight %}

####9. Test JTAG functionality with “avr-gdb”
  {% highlight console  %}
  $ sudo avarice --dragon --jtag usb :4242
  AVaRICE version 2.11, Jan 17 2014 02:51:59
  Defaulting JTAG bitrate to 250 kHz.
  JTAG config starting.
  Found a device: AVRDRAGON
  Serial number:  00:a2:00:06:90:52
  Reported JTAG device ID: 0x9587
  Configured for device ID: 0x9587 atmega32u4
  JTAG config complete.
  Preparing the target device for On Chip Debugging.
  Disabling lock bits:  LockBits -> 0xff
  Enabling on-chip debugging:
  Extended Fuse byte -> 0xcb
  High Fuse byte -> 0x18
  Low Fuse byte -> 0xff
  Waiting for connection on port 4242.
  {% endhighlight %}


####10. Open new terminal and run “avr-gdb”
  {% highlight console  %}
  $ avr-gdb
  .
  .
  .
  (gdb) file a.out Reading symbols from a.out...done.
  (gdb) target remote localhost:4242Remote debugging using localhost:42420x00000000 in __vectors ()
  (gdb) l
  1 #include <avr/io.h>
  2 #include <util/delay.h>
  3
  4 void main(void){
  5  DDRD |= (1<<PD4);
  6
  7  for(;;){
  8   PORTD |= (1<<PD4);
  9   _delay_ms(500.00);
  10 (gdb) continue
  Continuing.
  {% endhighlight %}

