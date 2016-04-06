Evented communication with RFXtrx433.
=====================================

How to Use
==========

rfxcom depends on the serialport module.

To install
----------

<pre>
  npm install rfxcom
</pre>

The only dependency is serialport 2.0.0+.

To Use
------

<pre>
var rfxcom = require('rfxcom'),
    pg = require('pg').native,
    conString = "pg://user:password@localhost/user",
    client = new pg.Client(conString);

var rfxtrx = new rfxcom.RfxCom("/dev/ttyUSB0", {debug: true});

/*
 * This reports security updates from X10 security devices.
 */
rfxtrx.on("security1", function (evt) {
  if (evt.deviceStatus === rfxcom.security.MOTION) {
    console.log("Device %s %s detected motion.", evt.subtype, evt.id);
  } else if (evt.deviceStatus === rfxcom.security.NOMOTION) {
    console.log("Device %s %s reported motion stopped.", evt.subtype, evt.id);
  }
});

rfxtrx.on("elec2", function (evt) {
  // Requires a PostgreSQL table
  // CREATE TABLE energy (recorded_time timestamp DEFAULT NOW(),
  //                      device_id VARCHAR, current_watts FLOAT)
  client.query("INSERT INTO energy(device_id, current_watts) values($1, $2)",
                [evt.id, evt.currentWatts]);
});

rfxtrx.initialise(function () {
    console.log("Device initialised");
});
</pre>

LightwaveRf
-----------
There's a specialised Lighting5 prototype, which uses an RfxCom object.

<pre>
var rfxcom = require('rfxcom');

var rfxtrx = new rfxcom.RfxCom("/dev/ttyUSB0", {debug: true}),
    lightwaverf = new rfxcom.Lighting5(rfxtrx, rfxcom.lighting5.LIGHTWAVERF);

rfxtrx.initialise(function () {
  console.log("Device initialised");
  lightwaverf.switchOn("0xF09AC8/1", {mood: 0x03});
  lightwaverf.switchOn("0xF09AC8/2", {level: 0x10});
});
</pre>

I've tested it with both LightwaveRf lights, and the relay switch.

LightwaveRf lights get their identity from the remote used to pair, if you don't
have a remote, or if you want to specify the address manually, you can pair the
device by putting the device into pairing mode and turning on a device id, lightwaverf.switchOn("0xFFFFFF/1").

The device ids don't have to be unique, but it's advisable.

Lighting2
---------
There's a specialised Lighting2 prototype, which uses an RfxCom object.

<pre>
    var rfxtrx = new rfxcom.RfxCom("/dev/ttyUSB0", {debug: true}),
        lighting2 = new rfxcom.Lighting2(rfxtrx, rfxcom.lighting2.HOMEEASY_EU);

    lighting2.switchOn("0xF09AC8AA/1");
    lighting2.switchOff("0xF09AC8AA/1");
</pre>

The lighting2 message controls one of three subtypes, you need to specify the
subtype to the constructor, the options are in rfxcom.lighting2.


RfxCom system events
====================

System events are used to track conection and disconnection of the RFXtrx433, and to provide
low-level access to received data.

"connecting"
------------
Emitted when the RFXcom has successfully opened the serial port.

"connectfailed"
------------
Emitted if the RFXcom was unable to open the serial port.

"ready"
-------
Emitted when the RFXtrx433 is ready to accept data (after a delay to prevent it from entering the bootloader).

"disconnect"
------------
Emitted if the RFXtrx433 has been disconnected from the USB port

"response"
----------
Emitted when a response message is received from the RFXtrx 433, sends the
status (from the RFXtrx433) and the sequence number of the message the response
is for.

"status"
--------
Emitted when a "status" message is received from the RFXtrx 433.

"end"
--------
Emitted when the serial port "ends".

"drain"
--------
Emitted when the serial port emits a "drain" event.

"receive"
---------
Emitted when any message is received from the RFXtrx 433, and contains the raw bytes that were received.

RfxCom received data events
===========================

The events are mostly named from the message identifiers used in the RFXtrx documentation. Not all protocols
can be received (some are transmit-only), and a protocol must be enabled to be received. This can be done using RFXmngr.exe,
or the `enable()` function of the rfxcom object.

"elec2"
-------
Emitted when data is received from OWL electricity monitoring devices
CM119/CM160.

"security1"
-----------
Emitted when an X10 security device reports a status change.

"lighting5"
-----------
Emitted when a message is received from LightwaveRF type devices.

"th1-9"
-------
Emitted when a message is received from Oregon Scientific
Temperature/Humidity sensors.

"temp1-9"
---------
Emitted when a message is received from an Oregon Scientific temperature
sensor.

"rain"
------
Emitted when a message is received from an Oregon Scientific rain sensor.

"lighting2"
-----------
Emitted when a message is received from AC/HomeEasy type devices.

Connecting and disconnecting
===
The function `rfxtrx.initialise()` will attempt to connect to the RFXtrx433 hardware. If this succeeds, a 'connecting' event
is emitted, followed about 5.5 seconds later by a 'ready' event. If the device is not present (wrong device path, or device
not plugged in) a 'connectfailed' event is emitted. If the the hardware is subsequently unplugged, a 'disconnect' event
is emitted (this can happen before either the 'connecting' or 'ready' events are emitted).

If either the connection fails or the RFXtrx433 is unplugged, a subsequent call to `initialise()` will attempt to reconnect.
The 'disconnect'/'connectfailed' handler may make repeated attempts to reconnect,
but <em>must</em> allow an interval of at least `rfxcom.initialiseWaitTime` milliseconds between each attempt. While
disconnected, any data sent by a call to a command object is silently discarded, however the various received data event
handlers are preserved.

<em>Note:</em>

Some variants of Linux will create a differently-named device file if the RFtxr433 is disconnected and then reconnected,
even if it is reconnected to the same USB port. For example, `/dev/ttyUSB0` may become `/dev/ttyUSB1`. To avoid any
problems this may cause, use the equivalent alias device file in `/dev/serial/by-id/` when creating the RfxCom object.
This should look something like `/dev/serial/by-id/usb_RFXCOM_RFXtrx433_12345678-if00-port0`.
