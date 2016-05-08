Contributing
============
If you own a device without information here, and you'd like to assist the
project by providing information on your device:

# lsusb
Provide a verbose `lsusb` for your device; if it only presents itself as one usb
device, name it `lsusb.pid` separately for each device, where `pid` is the device's
usb product id:

## Logitech G105 Keyboard
```shell
$ lsusb -vv -d 046d:c248 > g105/info/lsusb.c248
```
## Logitech G19 Keyboard
```shell
$ lsusb -vv -d 046d:c228 > g19/info/lsusb.c228
$ lsusb -vv -d 046d:c229 > g19/info/lsusb.c229
```
## Logitech G602 Mouse
```shell
$ lsusb -vv -d 046d:c537 > g602/info/lsusb.c537
```

# HID Report Descriptor
Typically each G-Series device has two or more usb interfaces, each with their
own HID Report Descriptor, one can obtain them from the kernel debugfs; to make
use of it, one must first mount it with the following command:
```shell
mount -t debugfs none /sys/kernel/debug
```
After which, your device's report descriptor can be found at:
`/sys/kernel/debug/hid/XXXX:VID:PID.YYYY/rdesc`

VID and PID refer to the usb vendor and product ids; for Logitech devices, VID
is always 046D. XXXX represents the 'bus' the device is on, typically this is
0003 for the usb bus, and YYYY is variable, and can be changed with each
unplug/replug of your device, or other operations against it. Name it similarly
to the lsusb file `rdesc.pid.n`, where n is the usb interface of the device.

## Logitech G105 Keyboard
```shell
$ cat /sys/kernel/debug/hid/0003:046D:C248.005E/rdesc > g105/info/rdesc.c248.0
$ cat /sys/kernel/debug/hid/0003:046D:C248.005F/rdesc > g105/info/rdesc.c248.1
```
## Logitech G19 Keyboard
```shell
$ cat /sys/kernel/debug/hid/0003:046D:C228.0016/rdesc > g19/info/rdesc.c228.0
$ cat /sys/kernel/debug/hid/0003:046D:C228.0017/rdesc > g19/info/rdesc.c228.1
$ cat /sys/kernel/debug/hid/0003:046D:C229.0018/rdesc > g19/info/rdesc.c229.0
```
## Logitech G602 Mouse
```shell
$ cat /sys/kernel/debug/hid/0003:046D:C537.0053/rdesc > g602/info/rdesc.c537.0
$ cat /sys/kernel/debug/hid/0003:046D:C537.0054/rdesc > g602/info/rdesc.c537.1
```

# Usbmon dumps
By using the usbmon kernel module one can monitor the actions performed on and
by the usb device in a simple to parse output. Note that usbmon cannot target
individual usb devices, but captures on a whole bus; as such, it is best to move
your devices in such a way that they are all on a separate usb bus.
## Good
The lsusb below is good, as each device is on its own usb bus, and as such we
avoid extraineous noise in dumping the logs.
```shell
$ lsusb
Bus 009 Device 020: ID 046d:c537 Logitech, Inc.
Bus 009 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 007 Device 012: ID 046d:c248 Logitech, Inc. G105 Gaming Keyboard
Bus 007 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```

## Bad
The following lsusb is not so good, as both the Lite-On Technology Corp. device
and the Logitech G602 share the same bus, which could produce unwanted data in
the usbmon dumps.
```shell
$ lsusb
Bus 009 Device 022: ID 04ca:004b Lite-On Technology Corp.
Bus 009 Device 020: ID 046d:c537 Logitech, Inc.
Bus 009 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 007 Device 012: ID 046d:c248 Logitech, Inc. G105 Gaming Keyboard
Bus 007 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```

In both cases you can ignore the Linux Foundation root hubs; they will always be
there and will not interfere with the dumps.

# Needed usbmon dumps
For each device, you will need a few different dumps from usbmon. Keyboards are
a bit simpler, as moving the keyboard won't taint the data as would moving a
mouse, so you just have to be really careful to not press any extra keys. An
extra keyboard and mouse are essential to this process to prevent data tainting.
## Gkey dumps
```shell
$ cat /sys/kernel/debug/usb/usbmon/7u > g105/info/raw/usbmon-g
# the 7 is because bus 7 for the G105 keyboard.
# press each gkey in order, g1-g6 for the g105, this will vary depending on the
# device in question.
C^
$ wc -l g105/info/raw/usbmon-g
24
# total line count, we will need to split this into separate files for each gkey
# a handy way to do this is to use the split command, part of gnu coreutils.
# one can use the bash shell to figure out the lines per key simply like so:
$ echo $((24/6))
4
# where 6 is the number of g-keys
$ split -d -l 4 g105/info/raw/usbmon-g g105/info/raw/usbmon-g
# splits the files into 4-line segments, prefixed with the name usbmon-g
$ for i in {5..0}
> do mv g105/info/raw/usbmon-g0$i g105/info/raw/usbmon-g$(($i+i)).log
> done
# renames the split files; split starts counting at 0, so we move them to
# match the actual filename to the gkey number (there is no g0, for example)
```

## Mkey dumps
More or less the same procedure as dumping the gkeys, except that there are only
three mkeys plus mr; adjust the above commands accordingly

# Logitech Gaming Software sniffing
Everything you just did, we do again, except this time we do it when doing usb
passthrough into a windows virtual machine with the Logitech Gaming Software
active. One thing must be done differently, however; after installing LGS, it
sets itself up to be started when windows boots; we do not want this behavior,
so disable it in the config options.
```shell
# start the virtual machine with only one device attached at a time
$ cat /sys/kernel/debug/usb/usbmon/7u | tee -a g105/info/lgs/usbmon-boot.log
# start the logitech gaming software now, and wait for the output to stop.
# this is to capture any special initialization commands sent to the device when
# lgs is started
```

After this you can repeat the same mkey and gkey captures as before, with one
small difference: when you press the mr key, it either turns the led on or off
depending on its prior state; in one sense it behaves very much like a capslock
key. These captures will more than likely be much longer than the raw linux
captures, as lgs will send more information to the keyboard to manipulate the
leds and so fourth.
