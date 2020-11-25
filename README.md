# MouseJack Exploitation on Catalina 10.15.5

## The Exploit
I had a lot of fun recently exploiting the now old MouseJack vulnerabilities. Essentially this exploit is the result of unifying USB recievers sending unencrypted data between mouse (or other peripheral) and computer. More can be read about it here. (https://www.mousejack.com)

## Hardware
To perform this exploit you need hardware that is specially crafted to send and recieve radio signals. There's a good USB radio dongle on Amazon made by Crazyradio PA that is ideal for this (https://www.amazon.com/gp/product/B00VYA3A2U/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1). I used this board, but you can technically do this with a Logitech Unifying dongle (ones that are based on C-U0007 Nordic Semiconductors) or other nRF24LU1+ chips like the SparkFun's breakout board. The advantage of the Crazyradio one for me was it has a factory programmer bootloader already installed in the top 2KB of flash, which is really conveninent. 

## Setup
First, obtain the firmware from here (https://github.com/BastilleResearch/mousejack) and build it. They have a tutorial that is pretty good so I would follow that. This takes care of flashing the stock Crazyradio firmware for you as long as you initialize the git submodule `nrf-research-firmware` and follow their steps. 

## Changes for Catalina (and OSX)
Basically when you try to build the firmware within `nrf-research-firmware` you will probably come up on failure for a few reasons. Your error will probably look something like this:

```
usage: grep [-abcDEFGHhIiJLlmnOoqRSsUVvwxZ] [-A num] [-B num] [-C[num]]
	[-e pattern] [-f file] [--binary-files=value] [--color=when]
	[--context[=num]] [--directories=action] [--label] [--line-buffered]
	[--null] [pattern] [file ...]
/bin/sh: line 0: test: -lt: unary operator expected
mkdir -p bin
sdcc --model-large --std-c99 -c src/main.c -o bin/main.rel
sdcc --model-large --std-c99 -c src/usb.c -o bin/usb.rel
sdcc --model-large --std-c99 -c src/usb_desc.c -o bin/usb_desc.rel
sdcc --model-large --std-c99 -c src/radio.c -o bin/radio.rel
sdcc --xram-loc 0x8000 --xram-size 2048 --model-large bin/main.rel bin/usb.rel bin/usb_desc.rel bin/radio.rel -o bin/dongle.ihx
objcopy -I ihex bin/dongle.ihx -O binary bin/dongle.bin
make: objcopy: No such file or directory
make: *** [dongle.bin] Error 1
```

First, make sure you're using `sudo`, but more importantly Mac has no `objcopy` or `objdump`, which is a problem. Luckily there are equivalents. First, install `crosstools-ng` from Brew, `brew install crosstool-ng`. This will install what is basically a Linux toolchain for software development - that is to say, it's a bunch of small programs. Some are binaries and others are static libraries. Within this **used** to be an `objcopy` equivalent called `gobjdump` (https://pigiuz.wordpress.com/2013/09/12/resolving-symbols-conflict-between-libraries-on-osx-with-objcopy/). Now, it's apparently deprecated and we'll need to use the (largely undocumented on brew) `sdobjcopy`. All this requires is installing the `crosstools-ng` package from brew or MacPorts and changing the `nrf-research-firmware` Makefile.

Let's change these lines:

```
dongle.bin: $(OBJS)
	$(SDCC) $(LDFLAGS) $(OBJS:%=bin/%) -o bin/dongle.ihx
	**objcopy -> sdobjcopy** -I ihex bin/dongle.ihx -O binary bin/dongle.bin
	**objcopy -> sdobjcopy** --pad-to 26622 --gap-fill 255 -I ihex bin/dongle.ihx -O binary bin/dongle.formatted.bin
	**objcopy -> sdobjcopy** -I binary bin/dongle.formatted.bin -O ihex bin/dongle.formatted.ihx
``` 

Which hopefully results in a nice clean make:

```
usage: grep [-abcDEFGHhIiJLlmnOoqRSsUVvwxZ] [-A num] [-B num] [-C[num]]
	[-e pattern] [-f file] [--binary-files=value] [--color=when]
	[--context[=num]] [--directories=action] [--label] [--line-buffered]
	[--null] [pattern] [file ...]
/bin/sh: line 0: test: -lt: unary operator expected
sdcc --model-large --std-c99 -c src/main.c -o bin/main.rel
sdcc --model-large --std-c99 -c src/usb.c -o bin/usb.rel
sdcc --model-large --std-c99 -c src/usb_desc.c -o bin/usb_desc.rel
sdcc --model-large --std-c99 -c src/radio.c -o bin/radio.rel
sdcc --xram-loc 0x8000 --xram-size 2048 --model-large bin/main.rel bin/usb.rel bin/usb_desc.rel bin/radio.rel -o bin/dongle.ihx
sdobjcopy -I ihex bin/dongle.ihx -O binary bin/dongle.bin
sdobjcopy --pad-to 26622 --gap-fill 255 -I ihex bin/dongle.ihx -O binary bin/dongle.formatted.bin
sdobjcopy -I binary bin/dongle.formatted.bin -O ihex bin/dongle.formatted.ihx
```

# Wrapping
This exploit is fun and underused - probably because it requires external hardware. The fact that it's completely remote, has no authentication and has a 100 meter radius is a security nightmare for blue teams, and I will definitely look to this more on pentests. My advice is to not let it stop you, hardware gets cheaper every day. Plus you can dual-purpose the Crazyradio board (if you went with that one) and re-flash the bootloader to do whatever other radio signal projects you can conjure up.
