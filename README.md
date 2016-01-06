node-rpio
=========

This is a high performance nodejs addon which provides access to the Raspberry
Pi GPIO interface, supporting regular GPIO as well as i²c, PWM, and SPI.

[![Build Status](https://travis-ci.org/jperkin/node-rpio.svg?branch=master)](https://travis-ci.org/jperkin/node-rpio)
[![NPM version](https://badge.fury.io/js/rpio.svg)](http://badge.fury.io/js/rpio)

## Quickstart

All these examples use the physical numbering (P01-P40) and assume that the
example is started with:

```js
var rpio = require('rpio');
```

### Read a pin

Setup pin P11 / GPIO17 for read-only input and print its current value:

```js
rpio.open(11, rpio.INPUT);
console.log('Pin 11 is currently set ' + (rpio.read(11) ? 'high' : 'low'));
```

### Blink an LED

Blink an LED attached to P12 / GPIO18 a few times:

```js
/*
 * Set the initial state to low.  The state is set prior to the pin becoming
 * active, so is safe for devices which require a stable setup.
 */
rpio.open(12, rpio.OUTPUT, rpio.LOW);

/*
 * The sleep functions block, but rarely in these simple programs does one care
 * about that.  Use a setInterval()/setTimeout() loop instead if it matters.
 */
for (var i = 0; i < 5; i++) {
        /* On for 1 second */
        rpio.write(12, rpio.HIGH);
        rpio.sleep(1);

        /* Off for half a second (500ms) */
        rpio.write(12, rpio.LOW);
        rpio.msleep(500);
}
```

### Poll a button switch for events

Configure the internal pulldown resistor on P15 / GPIO22 and watch the pin for
state changes from an attached button switch:

```js
rpio.open(15, rpio.INPUT, rpio.PULL_DOWN);

function pollcb(pin)
{
        /*
         * Interrupts aren't supported by the underlying hardware, so events
         * may be missed during the 1ms poll window.  The best we can do is to
         * print the current state after a event is detected.
         */
        var state = rpio.read(pin) ? 'pressed' : 'released';
        console.log('Button event on P%d (button currently %s)', pin, state);
}

rpio.poll(15, pollcb);
```

A collection of example programs are also available in the
[examples](https://github.com/jperkin/node-rpio/tree/master/examples)
directory.

## Features

There are lots of GPIO modules available for node.js.  Why use this one?

### Performance

It's very fast.  Part of the module is a native addon which links against Mike
McCauley's [bcm2835](http://www.open.com.au/mikem/bcm2835/) library, providing
direct access to the hardware via `/dev/mem` and `/dev/gpiomem`.

Most alternative GPIO modules use the slower `/sys` file system interface.

How much faster?  Here is a [simple
test](https://gist.github.com/jperkin/e1f0ce996c83ccf2bca9) which calculates
how long it takes to switch a pin on and off 1 million times:

* rpi-gpio (using `/sys`): `701.023` seconds
* rpio (using `/dev/*mem`): `2.907` seconds

So rpio can be anywhere up to **250x faster** than the alternatives.

### Hardware support

While `/sys` provides a simple interface to GPIO, not all hardware features are
supported, and it's not always possible to handle certain types of hardware,
especially when employing an asynchronous model.  Using the `/dev/*mem`
interface means rpio can support a lot more functionality:

* rpio supports sub-millisecond access, with features to support multiple
  reads/writes directly with hardware rather than being delayed by the event
  loop.

* Output pins can be configured with a default state prior to being enabled,
  required by some devices and not possible to configure via `/sys`.

* Internal pullup/pulldown registers can be configured.

* Hardware i²c, PWM, and SPI functions are supported.

### Simple programming

rpio tries to make it simple to program devices, rather than having to jump
through hoops to support an asynchronous workflow.  Some parts of rpio block,
but that is intentional in order to provide a simpler interface, as well as
being able to support time-sensitive devices.

The aim is to provide an interface familiar to Unix programmers, with the
performance to match.

## Install

Easily install the latest via npm:

```bash
$ npm install rpio
```

## API

Start by requiring the addon.

```js
var rpio = require('rpio');
```

### GPIO

General purpose I/O tries to follow a standard open/read/write/close model.

Some useful constants are provided for use by all supporting functions:

* `rpio.HIGH` - pin high/1/on
* `rpio.LOW` - pin low/0/off

These can be useful to avoid magic numbers in your code.

#### `rpio.init([options])`

Initialise the bcm2835 library.  This will be called automatically by `.open()`
using the default option values if not called explicitly.  The default values
are:

```js
var options = {
        gpiomem: true,          /* Use /dev/gpiomem */
        mapping: 'physical',    /* Use the P1-P40 numbering scheme */
}
```

##### `gpiomem`

There are two device nodes for GPIO access.  The default is `/dev/gpiomem`
which, when configured with `gpio` group access, allows users in that group to
read/write directly to that device.  This removes the need to run as root, but
is limited to GPIO functions.

For non-GPIO functions (i²c, PWM, SPI) the `/dev/mem` device is required for
full access to the Broadcom peripheral address range and the program needs to
be executed as the root user (e.g. via sudo).  If you do not explicitly call
`.init()` when using those functions, the library will do it for you with
`gpiomem: false`.

You may also need to use `gpiomem: false` if you are running on an older Linux
kernel which does not support the `gpiomem` module.

rpio will throw an exception if you try to use one of the non-GPIO functions
after already opening with `/dev/gpiomem`, as well as checking to see if you
have the necessary permissions.

Valid options:

* `true`: use `/dev/gpiomem` for non-root but GPIO-only access
* `false`: use `/dev/mem` for full access but requires root

#### `mapping`

There are two naming schemes when referring to GPIO pins:

* By their physical header location: Pins 1 to 26 (A/B) or Pins 1 to 40 (A+/B+)
* Using the Broadcom hardware map: GPIO 0-25 (B rev1), GPIO 2-27 (A/B rev2, A+/B+)

Confusingly however, the Broadcom GPIO map changes between revisions, so for
example P3 maps to GPIO0 on Model B Revision 1 models, but maps to GPIO2 on all
later models.

This means the only sane default mapping is the physical layout, so that the
same code will work on all models regardless of the underlying GPIO mapping.

If you prefer to use the Broadcom GPIO scheme for whatever reason, you can set
`mapping` to `gpio` to switch to the GPIOxx naming.

Valid options:

* `gpio`: use the Broadcom GPIOxx naming
* `physical`: use the physical P01-P40 header layout

Examples:

```js
rpio.init({gpiomem: false});    /* Use /dev/mem for i²c/PWM/SPI */
rpio.init({mapping: 'gpio'});   /* Use the GPIOxx numbering */
```

#### `rpio.open(pin, mode[, option])`

Open a pin for input or output.  Valid modes are:

* `rpio.INPUT` - pin is input (read-only).
* `rpio.OUTPUT` - pin is output (read-write).
* `rpio.PWM` - configure pin for hardware PWM (see PWM section below).

For input pins, `option` can be used to configure the internal pullup or
pulldown resistors using options as described in the `.pud()` documentation
below.

For output pins, `option` defines the initial state of the pin, rather than
having to issue a separate `.write()` call.  This can be critical for devices
which must have a stable value, rather than relying on the initial floating
value when a pin is enabled for output but hasn't yet been configured with a
value.

Examples:

```js
/* Configure P11 as input with the internal pulldown resistor enabled */
rpio.open(11, rpio.INPUT, rpio.PULL_DOWN);

/* Configure P12 as output with the initiate state set high */
rpio.open(12, rpio.OUTPUT, rpio.HIGH);

/* Configure P13 as output, but leave it in its initial undefined state */
rpio.open(13, rpio.OUTPUT);
```

#### `rpio.mode(pin, mode)`

Switch a pin that has already been opened in one mode to a different mode.
This is provided primarily for performance reasons, as it avoids some of the
setup work done by `.open()`.

Example:

```js
rpio.mode(12, rpio.INPUT);      /* Switch P12 back to input mode */
```

#### `rpio.read(pin)`

Read the current value of `pin`, returning either `1` (high) or `0` (low).

Example:

```js
console.log('Pin 12 = %d', rpio.read(12));
```

#### `rpio.readn(pin, buffer[, length])`

Read `length` bits from `pin` into `buffer` as fast as possible.  If `length`
isn't specified it defaults to `buffer.length`.

This is useful for devices which send out information faster than the
JavaScript function call overhead can handle, e.g. if you need microsecond
accuracy.  See
[dht11.js](https://github.com/jperkin/node-rpio/blob/master/examples/dht11.js)
for an example which uses this to pull data from a DHT11 temperature/humidity
sensor.

Example:

```js
var buf = new Buffer(10000);

/* Read the value of Pin 12 10,000 times in a row, storing the values in buf */
rpio.readn(12, buf);
```

#### `rpio.write(pin, value)`

Set the specified pin either high or low, using either the
`rpio.HIGH`/`rpio.LOW` constants, or simply `1` or `0`.

Example:

```js
rpio.write(13, rpio.HIGH);
```

#### `rpio.pud(pin, state)`

Configure the pin's internal pullup or pulldown resistors, using the following
`state` constants:

* `rpio.PULL_OFF` - disable configured resistors.
* `rpio.PULL_DOWN` - enable the pulldown resistor.
* `rpio.PULL_UP` - enable the pullup resistor.

Example:

```js
rpio.pud(11, rpio.PULL_UP);
```

#### `rpio.poll(pin, cb[, direction])`

Watch `pin` for changes and execute the callback `cb()` on events.  `cb()`
takes a single argument, the pin which triggered the callback.

The optional `direction` argument can be used to watch for specific events:

* `rpio.POLL_LOW` - poll for falling edge transitions to low.
* `rpio.POLL_HIGH` - poll for rising edge transitions to high.
* `rpio.POLL_BOTH` - poll for both transitions (the default).

Due to hardware/kernel limitations we can only poll for changes, and the event
detection only says that an event occurred, not which one.  The poll interval
is a 1ms `setInterval()` and transitions could come in between detecting the
event and reading the value.  Therefore this interface is only useful for
events which transition slower than approximately 1kHz.

To stop watching for `pin` changes, call `.poll()` again, setting the callback
to `null` (or anything else which isn't a function).

Example:

```js
function cb(pin)
{
        console.log('Pin %d changed, is now %d', pin, rpio.read(pin));
}
rpio.poll(11, cb);
```

#### `rpio.close(pin)`

Reset `pin` to `rpio.INPUT` and clear any pullup/pulldown resistors and poll
events.

Examples:

```js
rpio.close(11);
rpio.close(12);
rpio.close(13);
```

#### GPIO demo

The code below continuously flashes an LED connected to pin 11 at 100Hz.

```js
var rpio = require('rpio');

/* Configure P11 as an output pin, setting its initial state to low */
rpio.open(11, rpio.OUTPUT, rpio.LOW);

/* Set the pin high every 10ms, and low 5ms after each transition to high */
setInterval(function() {
        rpio.write(11, rpio.HIGH);
        setTimeout(function() {
                rpio.write(11, rpio.LOW);
        }, 5);
}, 10);
```

### i²c

i²c is primarily of use for driving LCD displays, and makes use of pins 3 and 5
(GPIO0/GPIO1 on Rev 1, GPIO2/GPIO3 on Rev 2 and newer).  The bcm2835 library
automatically detects which Raspberry Pi revision you are running, so you do
not need to worry about which i²c bus to configure.

To get started call `.i2cBegin()` which assigns pins 3 and 5 to i²c use.  Until
`.i2cEnd()` is called they won't be available for GPIO use.

`.i2cBegin()` will call `.init()` if it hasn't already been called, with
`gpiomem: false` set.  Hardware i²c support requires `/dev/mem` access and
therefore root.

```js
rpio.i2cBegin();
```

Configure the slave address.  This is between `0 - 0x7f`, and it can be helpful
to run the `i2cdetect` program to figure out where your devices are if you are
unsure.

```js
rpio.i2cSetSlaveAddress(0x20);
```

Set the baud rate.  You can do this two different ways, depending on your
preference.  Either use `.i2cSetBaudRate()` to directly set the speed in hertz,
or `.i2cSetClockDivider()` to set it based on a divisor of the base 250MHz
rate.

```js
rpio.i2cSetBaudRate(100000);    /* 100kHz */
rpio.i2cSetClockDivider(2500);  /* 250MHz / 2500 = 100kHz */
```

Read from and write to the i²c slave.  Both functions take a buffer and
optional length argument, defaulting to the length of the buffer if not
specified.

```js
var txbuf = new Buffer([0x0b, 0x0e, 0x0e, 0x0f]);
var rxbuf = new Buffer(32);

rpio.i2cWrite(txbuf);           /* Sends 4 bytes */
rpio.i2cRead(rxbuf, 16);        /* Reads 16 bytes */
```

Finally, turn off the i²c interface and return the pins to GPIO.

```js
rpio.i2cEnd();
```

#### i²c demo

The code below writes two strings to a 16x2 LCD.

```js
var rpio = require('rpio');

/*
 * Magic numbers to initialise the i2c display device and write output,
 * cribbed from various python drivers.
 */
var init = new Buffer([0x03, 0x03, 0x03, 0x02, 0x28, 0x0c, 0x01, 0x06]);
var LCD_LINE1 = 0x80, LCD_LINE2 = 0xc0;
var LCD_ENABLE = 0x04, LCD_BACKLIGHT = 0x08;

/*
 * Data is written 4 bits at a time with the lower 4 bits containing the mode.
 */
function lcdwrite4(data)
{
        rpio.i2cWrite(Buffer([(data | LCD_BACKLIGHT)]));
        rpio.i2cWrite(Buffer([(data | LCD_ENABLE | LCD_BACKLIGHT)]));
        rpio.i2cWrite(Buffer([((data & ~LCD_ENABLE) | LCD_BACKLIGHT)]));
}
function lcdwrite(data, mode)
{
        lcdwrite4(mode | (data & 0xF0));
        lcdwrite4(mode | ((data << 4) & 0xF0));
}

/*
 * Write a string to the specified LCD line.
 */
function lineout(str, addr)
{
        lcdwrite(addr, 0);

        str.split('').forEach(function (c) {
                lcdwrite(c.charCodeAt(0), 1);
        });
}

/*
 * We can now start the program, talking to the i2c LCD at address 0x27.
 */
rpio.i2cBegin();
rpio.i2cSetSlaveAddress(0x27);
rpio.i2cSetBaudRate(10000);

for (var i = 0; i < init.length; i++)
        lcdwrite(init[i], 0);

lineout('node.js i2c LCD!', LCD_LINE1);
lineout('npm install rpio', LCD_LINE2);

rpio.i2cEnd();
```

### PWM

Pulse Width Modulation (PWM) allows you to create analog output from the
digital pins.  This can be used, for example, to make an LED appear to pulse
rather than be fully off or on.

The Broadcom chipset supports hardware PWM, i.e. you configure it with the
appropriate values and it will generate the required pulse.  This is much more
efficient and accurate than emulating it in software (by setting pins high and
low at particular times), but you are limited to only certain pins supporting
hardware PWM:

* 26-pin models: pin 12
* 40-pin models: pins 12, 19, 33, 35

Hardware PWM also requires `gpiomem: false` and root privileges.  `.open()`
will call `.init()` with the appropriate values if you do not explicitly call
it yourself.

To enable a PIN for PWM, use the `rpio.PWM` argument to `open()`:

```js
rpio.open(12, rpio.PWM); /* Use pin 12 */
```

Set the PWM refresh rate with `pwmSetClockDivider()`.  This is a power-of-two
divisor of the base 19.2MHz rate, with a maximum value of 4096 (4.6875kHz).

```js
rpio.pwmSetClockDivider(64);    /* Set PWM refresh rate to 300kHz */
```

Set the PWM range for a pin with `pwmSetRange()`.  This determines the maximum
pulse width.

```js
rpio.pwmSetRange(12, 1024);
```

Finally, set the PWM width for a pin with `pwmSetData()`.

```js
rpio.pwmSetData(12, 512);
```

#### PWM demo

The code below pulses an LED 5 times before exiting.

```js
var rpio = require('rpio');

var pin = 12;           /* P12/GPIO18 */
var range = 1024;       /* LEDs can quickly hit max brightness, so only use */
var max = 128;          /*   the bottom 8th of a larger scale */
var clockdiv = 8;       /* Clock divider (PWM refresh rate), 8 == 2.4MHz */
var interval = 5;       /* setInterval timer, speed of pulses */
var times = 5;          /* How many times to pulse before exiting */

/*
 * Enable PWM on the chosen pin and set the clock and range.
 */
rpio.open(pin, rpio.PWM);
rpio.pwmSetClockDivider(clockdiv);
rpio.pwmSetRange(pin, range);

/*
 * Repeatedly pulse from low to high and back again until times runs out.
 */
var direction = 1;
var data = 0;
var pulse = setInterval(function() {
        rpio.pwmSetData(pin, data);
        if (data === 0) {
                direction = 1;
                if (times-- === 0) {
                        clearInterval(pulse);
                        rpio.open(pin, rpio.INPUT);
                        return;
                }
        } else if (data === max) {
                direction = -1;
        }
        data += direction;
}, interval, data, direction, times);
```

### SPI

SPI switches pins 19, 21, 23, 24 and 25 (GPIO7-GPIO11) to a special mode where
you can bulk transfer data at high speeds to and from SPI devices, with the
controller handling the chip enable, clock and data in/out functions.

```js
/*
 *  Pin | Function
 * -----|----------
 *   19 |   MOSI
 *   21 |   MISO
 *   23 |   SCLK
 *   24 |   CE0
 *   25 |   CE1
 */
```

Once SPI is enabled, the SPI pins are unavailable for GPIO use until `spiEnd()`
is called.

Use `.spiBegin()` to initiate SPI mode.  SPI requires `gpiomem: false` and root
privileges.  `.spiBegin()` will call `.init()` with the appropriate values if
you do not explicitly call it yourself.

```js
rpio.spiBegin();           /* Switch GPIO7-GPIO11 to SPI mode */
```

Choose which of the chip select / chip enable pins to control:

```js
/*
 *  Value | Pin
 *  ------|---------------------
 *    0   | SPI_CE0 (24 / GPIO8)
 *    1   | SPI_CE1 (25 / GPIO7)
 *    2   | Both
 */
rpio.spiChipSelect(0);
```

Commonly chip enable (CE) pins are active low, and this is the default.  If
your device's CE pin is active high, use `spiSetCSPolarity()` to change the
polarity.

```js
rpio.spiSetCSPolarity(0, rpio.HIGH);    /* Set CE0 high to activate */
```

Set the SPI clock speed with `spiSetClockDivider()`.  This is a power-of-two
divisor of the base 250MHz rate, defaulting to `0 == 65536 == 3.81kHz`.

```js
rpio.spiSetClockDivider(128);   /* Set SPI speed to 1.95MHz */
```

Set the SPI Data Mode:

```js
/*
 *  Mode | CPOL | CPHA
 *  -----|------|-----
 *    0  |  0   |  0
 *    1  |  0   |  1
 *    2  |  1   |  0
 *    3  |  1   |  1
 */
rpio.spiSetDataMode(0);         /* 0 is the default */
```

Once everything is set up we can transfer data.  Data is sent and received in
8-bit chunks via buffers which should be the same size.

```js
var txbuf = new Buffer([0x3, 0x0, 0xff, 0xff]);
var rxbuf = new Buffer(txbuf.length);

rpio.spiTransfer(txbuf, rxbuf, txbuf.length);
```

If you only need to send data and do not care about the data coming back, you
can use the slightly faster `spiWrite()` call:

```js
rpio.spiWrite(txbuf, txbuf.length);
```

When you're finished call `.spiEnd()` to release the pins back to general
purpose use.

```js
rpio.spiEnd();
```

#### SPI demo

The code below reads the 128x8 contents of an AT93C46 serial EEPROM.

```js
var rpio = require('rpio');

rpio.spiBegin();
rpio.spiChipSelect(0);                  /* Use CE0 */
rpio.spiSetCSPolarity(0, rpio.HIGH);    /* AT93C46 chip select is active-high */
rpio.spiSetClockDivider(128);           /* AT93C46 max is 2MHz, 128 == 1.95MHz */
rpio.spiSetDataMode(0);

/*
 * There are various magic numbers below.  A quick overview:
 *
 *   tx[0] is always 0x3, the EEPROM READ instruction.
 *   tx[1] is set to var i which is the EEPROM address to read from.
 *   tx[2] and tx[3] can be anything, at this point we are only interested in
 *     reading the data back from the EEPROM into our rx buffer.
 *
 * Once we have the data returned in rx, we have to do some bit shifting to
 * get the result in the format we want, as the data is not 8-bit aligned.
 */
var tx = new Buffer([0x3, 0x0, 0x0, 0x0]);
var rx = new Buffer(4);
var out;
var i, j = 0;

for (i = 0; i < 128; i++, ++j) {
        tx[1] = i;
        rpio.spiTransfer(tx, rx, 4);
        out = ((rx[2] << 1) | (rx[3] >> 7));
        process.stdout.write(out.toString(16) + ((j % 16 == 0) ? '\n' : ' '));
}
rpio.spiEnd();
```

### Misc

To make code simpler a few sleep functions are supported, using the hardware
directly so should be reasonably accurate.

```js
rpio.sleep(n);          /* Sleep for n seconds */
rpio.msleep(n);         /* Sleep for n milliseconds */
rpio.usleep(n);         /* Sleep for n microseconds */
```

## Authors And Licenses

Mike McCauley wrote `src/bcm2835.{c,h}` which are under the GPL.

I wrote the rest, which is under the ISC license unless otherwise specified.
