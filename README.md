
# DeRex

C++ code for (attempting to) repeating Dexcom G5/G6 transmitter messages to/from a smart device

NOTES FROM THE SOFTWARE BUILD JAN - APR 2021

This was a senior project undertaken by Arizona State University seniors from August 2020 - Apr 2021.

The (proposed) function of this software was to use the Arduino Nano 33 IoT as a repeater for Dexcom devices,
thereby increasing the usable range of the transmitter in relation to its endpoint (a smartphone or the Dexcom
receiver).  The impetus was the range being too small for a sampling of 350 users, and by creating a "hub"-like device
that could be placed in various parts of the home, the range would hopefully increase without the Type I or II
being tethered to his/her phone.  Decryption of the data or methods used by Dexcom was not the goal,
only the relaying of messages to and from the transmitters.  The software here is designed to support up
to 4 transmitters at a time, which is about 70% of the memory (it seems pushing the memory beyond this
point causes serial port problems with the Arduino Nano 33 IoT).  It is not reliable long-term at 4 nodes due to
hangs or crashes so 3 or fewer is recommended

The project did not achieve its ultimate goal and the code is being placed on GitHub as-is.
No support or additional files aside from what is here in this archive is implied or offered; Arizona
State University (ASU) and its employees or students assume no liability for those who decide to utilise
this software.  Copying it and using it is the sole responsibility of you, the end user.  It is our sincere
hope that this software and the hardware to go with it are transformed into a safe, working product for
those who use the Dexcom CGM system.  When in doubt, always fall back to a working CGM system or fingersticks
and seek medical help if and when necessary.  We wish you the best of luck!

There is an open-source licence attached to this project.  Please read it if you desire deep, restful sleep.

The following are some items learned from the design and construction of the software:

1.  connect() and disconnect() are not 100% reliable.  Connect times are about 1-1.2 seconds; after the first
    time, the connect time increases to 1.4-2.0 seconds for some reason, which is half of the BLE connect time, so
    there is not a lot of time left to refresh a profile, for example.  disconnect() will fail but it's rare
    when it does, a peripheral disconnect is never registered and the routine to disconnect will run forever.

2.  Weak signals cause hangs.  Somewhere between -85 and -90 dBm is the limit for this hardware or the software
    controlling it.  The power level needs to be checked before connecting otherwise more hangs occur; at -90 dBm,
    the software will run for up to 6 hours (max recorded time before stall or hang occurs).  Something is 
    probably not being returned in a low-level function and there is no timeout.

3.  Converting String() from the global tables was definitely worth it.  There was unusual memory corruption that
    would apparently interfere with the Arduino Serial port (or just crash the system).

4.  There is no callback event for descriptors but the BLE spec. mentions reading and writing descriptors for
    medical devices such as CGMs.

5.  The online documentation leaves a lot to be desired.  For some functions, it is wrong or too vague/
    too general.  Hopefully this will improve with later generations of ArduinoBLE library releases.

6.  The auto-generated services, characteristics and descriptors is strange.  The line that instantiates the
    services in the GATT.cpp file had to be commented out.  It seems this was done for users who don't want to
    mess with the overhead of BLE specifications but there should be a (documented) option to turn it off.

7.  It is unknown if the ArduinoBLE library saves the profile of each peripheral it scans.  It was assumed
    it does not and so global char-based array tables were used to store service, characteristic and descriptor
    data for each peripheral (so that they could be advertised later).

8.  There was some assistance provided by some Arduino forum programmers - for that we are thankful.  Unfortunately,
    the support was not always there and our project was compressed into a roughly 3-month schedule.  We are also not
    the greatest programmers but we did our best.  Additional support and more time may have led to success.

9.  The stumbling block at the end was the refresh cycle.  The software would detect a stale entry, due to
    a callback that usually implied a read later on, and when it tried to connect it would always time out.  The
    Arduino connect() routine was taking longer than expected, usually 0.5 seconds longer than the first
    successful time, which was weird because multiple devices could be read successfully but when a re-read of 
    an already saved profile was attempted it would fail.  After the semester was over, the solution was found
    to be separating multiple conditional function calls on one line into several.  Order of operations for the
    Arduino IDE is not fully understood (see item 12).

10. Sometimes digging into the main library functions within ArduinoBLE.h were helpful.  The prototypes help explain
    some of the functionality and definitely show the acceptable datatypes to be passed in/retrieved.  They also
    show hidden functions or parameters that are not called out in the online reference pages.

11. Serial.print() functions were largely commented out.  This sped up access to transmitters significantly.  Speeding
    up the serial port baud rate did not have much of an effect but it probably helped.  It turns out that 
    115200 bps is good under almost all conditions; turning off data read printouts helps a ton.  It was though
    that serial port messages were slowing down access cycles (beyond 4 seconds) but even with all of them off
    the access takes 5.5 seconds, which is longer than allowed and a forced disconnect still occurs (only after
    callbacks are enabled).

12. Order of operations is somewhat strange with the Arduino IDE.  It was "assumed" (yes, we know) function calls on a
    single line would be evaluated left to right, and parenthetical references would increase priority, but that
    did not seem to be the case.  Some statements that were split into multiple if-thens worked much better (few
    or no hangs, strange function duplication eliminated - a function that printed out a debug message would strangely
    repeat a duplicate debug message, no memory corruption, etc.).  Maybe the IDE (or even the ARM proc) performs some
    optimisation that is not visible?

13. The code still hangs.  It takes a while and it's more common with > 70% memory usage.  It's also disappointing that
    with greater than 70% memory usage the Arduino IDE shows a "instability may occur" message after compiling.  That
    indicates memory handling by the IDE is terrible, and you have to sacrifice 25-30% of your memory to compensate.
    With MAX_NODES = 4, MAX_BYTES = 24 (should be 512), MAX_CMDS = 64 the memory is around 71% and it will hang after
    several hours of running - usually after connecting but before executing the next statement, which is an Arduino
    connect() library issue or some other hardware problem.

14. Scanning becomes harder when advertising is on at the "same time" (whether or not it is really on at the same time
    is debatable as that would mean the antenna is being used for transmit and receive simultaneously, which would be
    difficult - it's more likely it's multiplexing quickly).  Anyway, it could be the time it spends in the peripheral
    section of the code vs. the central section of the code stealing detection time away from the scan detector.  If
    MAX_NODES = 1, then once a device is discovered successfully, the scanner turns off and its all advert then.
