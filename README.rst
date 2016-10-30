SSD1306 / SH1106 OLED Driver
============================

Interfacing OLED matrix displays with the SSD1306 (or SH1106) driver in Python 2 or 3 using
I2C on the Raspberry Pi. The particular kit I bought can be acquired for 
a few pounds from `eBay <http://www.ebay.co.uk/itm/191279261331>`_. Further 
technical details for the SSD1306 OLED display can be found in the
`datasheet <https://raw.githubusercontent.com/rm-hull/ssd1306/master/doc/tech-spec/SSD1306.pdf>`_ [PDF]. 
See also the `datasheet <https://raw.githubusercontent.com/rm-hull/ssd1306/sh1106-compat/doc/tech-spec/SH1106.pdf>`_ [PDF] for the SH1106 chipset.

The SSD1306 display is 128x64 pixels, and the board is `tiny`, and will fit neatly
inside the RPi case (the SH1106 is slightly different, in that it supports 132 x 64
pixels). My intention is to solder wires directly to the underside
of the RPi GPIO pins so that the pins are still available for other purposes.

.. image:: https://raw.githubusercontent.com/rm-hull/ssd1306/master/doc/mounted_display.jpg
   :alt: mounted

GPIO pin-outs
-------------

The SSD1306 device is an I2C device, so connecting to the RPi is very straightforward:

P1 Header
^^^^^^^^^

For prototyping, the P1 header pins should be connected as follows:

========== ====== ============ ======== ============== ========
Board Pin  Name   Remarks      RPi Pin  RPi Function   Colour
---------- ------ ------------ -------- -------------- --------
1          GND    Ground       P01-6    GND            Black
2          VCC    +3.3V Power  P01-1    3V3            White
3          SCL    Clock        P01-5    GPIO 3 (SCL)   Purple
4          SDA    Data         P01-3    GPIO 2 (SDA)   Grey
========== ====== ============ ======== ============== ========


.. image:: https://raw.githubusercontent.com/rm-hull/ssd1306/master/doc/GPIOs.png
   :alt: GPIOS

P5 Header
^^^^^^^^^

On rev.2 RPi's, right next to the male pins of the P1 header, there is a bare 
P5 header which features I2C channel 0, although this doesn't appear to be
initially enabled and may be configured for use with the Camera module. 

========== ====== ============ ======== ============== ========
Board Pin  Name   Remarks      RPi Pin  RPi Function   Colour
---------- ------ ------------ -------- -------------- --------
1          GND    Ground       P5-07    GND            Black
2          VCC    +3.3V Power  P5-02    3V3            White
3          SCL    Clock        P5-04    GPIO 29 (SCL)  Purple
4          SDA    Data         P5-03    GPIO 28 (SDA)  Grey
========== ====== ============ ======== ============== ========

.. image:: https://raw.githubusercontent.com/rm-hull/ssd1306/master/doc/RPi_P5_header.png
   :alt: P5 Header

Pre-requisites
--------------

This was tested with Raspian on a rev.2 model B, with a vanilla kernel version 4.1.16+. 
Ensure that the I2C kernel driver is enabled::

  $ dmesg | grep i2c
  [    4.925554] bcm2708_i2c 20804000.i2c: BSC1 Controller at 0x20804000 (irq 79) (baudrate 100000)
  [    4.929325] i2c /dev entries driver

or::

  $ lsmod | grep i2c
  i2c_dev                 5769  0 
  i2c_bcm2708             4943  0 
  regmap_i2c              1661  3 snd_soc_pcm512x,snd_soc_wm8804,snd_soc_core

If you have no kernel modules listed and nothing is showing using ``dmesg`` then this implies
the kernel I2C driver is not loaded. Enable the I2C as follows:

#. Run ``sudo raspi-config``
#. Use the down arrow to select ``9 Advanced Options``
#. Arrow down to ``A7 I2C``
#. Select **yes** when it asks you to enable I2C
#. Also select **yes** when it asks about automatically loading the kernel module
#. Use the right arrow to select the **<Finish>** button
#. Select **yes** when it asks to reboot

After rebooting re-check that the ``dmesg | grep i2c`` command shows whether 
I2C driver is loaded before proceeding.

Optionally, to improve permformance, increase the I2C baudrate from the default 
of 100KHz to 400KHz by altering ``/boot/config.txt`` to include::

  dtparam=i2c_arm=on,i2c_baudrate=400000

Then reboot.

Then add your user to the i2c group::

  $ sudo adduser pi i2c

Install some packages (python2)::

  $ sudo apt-get install i2c-tools python-smbus python-pip libfreetype6-dev 
  $ sudo pip install pillow

or (python3)::

  $ sudo apt-get install i2c-tools python3-smbus python3-pip libfreetype6-dev 
  $ sudo pip3 install pillow

Next check that the device is communicating properly (if using a rev.1 board, 
use 0 for the bus not 1)::

  $ i2cdetect -y 1
         0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
    00:          -- -- -- -- -- -- -- -- -- -- -- -- --
    10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    30: -- -- -- -- -- -- -- -- -- -- -- UU 3c -- -- --
    40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    70: -- -- -- -- -- -- -- --

According to the manual, "UU" means that probing was skipped, 
because the address was in use by a driver. It suggest that
there is a chip at that address. Indeed the documentation for
the device indicates it uses two addresses.

Installing the Python Package
-----------------------------

For python2, from the bash prompt, enter::

  $ sudo python setup.py install

This will install the Python files in ``/usr/local/lib/python2.7``
making them ready for use in other programs.

Alternatively for python3, type::

 $ sudo python3 setup.py install


Software Display Driver
-----------------------

The screen can be driven with python using the ``oled/device.py`` script.
There are two device classes and usage is very simple if you have ever
used `Pillow <https://pillow.readthedocs.io/en/latest/>`_ or PIL.

First, import and initialise the device:

.. code:: python

  from oled.device import ssd1306, sh1106
  from oled.render import canvas
  from PIL import ImageFont, ImageDraw

  # substitute sh1106(...) below if using that device
  device = ssd1306(port=1, address=0x3C)  # rev.1 users set port=0

The display device should now be configured for use. The specific ``ssd1306`` or 
``sh1106`` classes both expose a ``display()`` method which takes a 1-bit depth image. 
However, for most cases, for drawing text and graphics primitives, the canvas class
should be used as follows:

.. code:: python

  with canvas(device) as draw:
      font = ImageFont.load_default()
      draw.rectangle((0, 0, device.width, device.height), outline=0, fill=0)
      draw.text((30, 40), "Hello World", font=font, fill=255)

The ``canvas`` class automatically creates an
`ImageDraw <https://pillow.readthedocs.io/en/latest/reference/ImageDraw.html>`_
object of the correct dimensions and bit depth suitable for the device, so you
may then call the usual Pillow methods to draw onto the canvas.

As soon as the with scope is ended, the resultant image is automatically
flushed to the device's display memory and the ImageDraw object is
garbage collected.

Run the demos in the example directory::

  $ python examples/demo.py
  $ python examples/sys_info.py
  $ python examples/pi_logo.py
  $ python examples/maze.py

Notes
-----

#. Substitute ``python3`` for ``python`` in the above examples if you are using python3.
#. ``python-dev`` (apt-get) and ``psutil`` (pip/pip3) are required to run the ``sys_info.py`` example.
  See `install instructions <https://github.com/rm-hull/ssd1306/blob/master/examples/sys_info.py#L3-L7>`_
  for the exact commands to use.


References
----------

- https://learn.adafruit.com/monochrome-oled-breakouts
- https://github.com/adafruit/Adafruit_Python_SSD1306
- http://www.dafont.com/bitmap.php
- http://raspberrypi.znix.com/hipidocs/topic_i2cbus_2.htm
- http://martin-jones.com/2013/08/20/how-to-get-the-second-raspberry-pi-i2c-bus-to-work/

License
-------

The MIT License (MIT)

Copyright (c) 2016 Richard Hull

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
