# LIS3DH 3-Axis Accelerometer

The [LIS3DH](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/datasheet/CD00274221.pdf) is a 3-Axis MEMS accelerometer. The LIS3DH application note can be found [here](http://www.st.com/web/en/resource/technical/document/application_note/CD00290365.pdf). This sensor has extensive functionality and this class has not yet implemented all of it.

The LPS25H can interface over I&sup2;C or SPI. This class addresses only I&sup2;C for the time being.

**To add this library to your project, add** `#require "LIS3DH.class.nut:1.2.0"` **to the top of your device code**

## Class Usage

### Constructor: LIS3DH(*i2cBus[, i2cAddress]*)

The class’ constructor takes one required parameter (a configured imp I&sup2;C bus) and an optional parameter (the I&sup2;C address of the accelerometer). The I&sup2;C address must be the address of your sensor or an I&sup2;C error will be thrown.

| Parameter     | Type         | Default | Description |
| ------------- | ------------ | ------- | ----------- |
| *i2cBus*      | hardware.i2c | N/A     | A pre-configured I&sup2;C bus |
| *i2cAddress*  | byte         | 0x30    | The I&sup2;C address of the accelerometer |

&nbsp;<br>

```squirrel
#require "LIS3DH.class.nut:1.2.0"

i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);

// Use a non-default I2C address (SA0 pulled high)
accel <- LIS3DH(i2c, 0x32);
```

## Class Methods

### init()

The *init()* method resets all control and interrupt registers to datasheet default values.

```squirrel
accel <- LIS3DH(i2c, 0x32);
accel.init();
```

### setDataRate(*rateHz*)

The *setDataRate()* method sets the Output Data Rate (ODR) of the accelerometer in Hz. Supported datarates are 0 (Shutdown), 1, 10, 25, 50, 100, 200, 400, 1250 (Normal Mode only), 1600 (Low-Power Mode only), and 5000 (Low-Power Mode only) Hz. The requested data rate will be rounded up to the closest supported rate and the actual data rate will be returned.

The default data rate is 0 (Shutdown). To take a reading with *getAccel()* you must set a data rate greater than 0.

```squirrel
local rate = accel.setDataRate(90);
server.log(format("Accelerometer running at %dHz", rate));
// Displays 'Accelerometer running at 100Hz'
```

### setLowPower(*state*)

The *setLowPower()* method configures the device to run in low-power or normal mode. The method takes one boolean parameter, *state*. When *state* is `true`, low-power mode is enabled. When *state* is `false`, normal mode is enabled. Normal mode guarantees high resolution; low-power mode reduces the current consumption. Higher data rates only support specific modes. See *setDataRate()* for details.

Normal mode is enabled by default.

```Squirrel
// Enable low-power mode
accel.setLowPower(true);
```

### enable([*state*])

The *enable()* method enables or disables all three axes on the accelerometer. The method takes an optional boolean parameter, *state*.  By default *state* is set to `true` and the accelerometer is enabled. When *state* is `false`, the accelerometer will be disabled.

```squirrel
function goToSleep() {
    imp.onidle(function() {
        // Set data rate to 0 and disable the accelerometer to save power
        accel.setDataRate(0);
        accel.enable(false);

        // Sleep for 1 hour
        server.sleepfor(3600);
    });
}
```

### getAccel(*[callback]*)

The *getAccel()* method reads the latest measurement from the accelerometer. The method takes an optional callback for asynchronous operation &mdash; it will block otherwise. The callback should take one parameter: a results table *(see below)*. If the callback is null or omitted, the method will return the results table.

```
{ "x": <xData>,
  "y": <yData>,
  "z": <zData> }
```

#### Synchronous example

```squirrel
accel.setDataRate(100);
local val = accel.getAccel();
server.log(format("Acceleration (G): (%0.2f, %0.2f, %0.2f)", val.x, val.y, val.z));
```

#### Asynchronous Example

```squirrel
accel.setDataRate(100);
accel.getAccel(function(val) {
    server.log(format("Acceleration (G): (%0.2f, %0.2f, %0.2f)", val.x, val.y, val.z));
});
```

### setRange(*range*)

The *setRange()* method sets the measurement range of the sensor in Gs. Supported ranges are (&plusmn;) 2, 4, 8 and 16G. The data rate will be rounded up to the closest supported range and the actual range will be returned.

The default measurement range is &plusmn;2G.

```squirrel
// Set sensor range to +/- 8G
local range = accel.setRange(8);
server.log(format("Range set to +/- %dG", range));
```

### getRange()

The *getRange()* method returns the currently-set measurement range of the sensor in Gs.

```squirrel
server.log(format("Current Sensor Range is +/- %dG", accel.getRange()));
```

### configureInertialInterrupt(*state[, threshold][, duration][, options]*)

This method configures the inertial interrupt generator:

| Parameter | Type | Default Value | Description |
| --------- | ---- | ------------- | ----------- |
| *state*     | Boolean | N/A | `true` to enable, `false` to disable |
| *threshold* | Float   | 2.0 | Inertial interrupts threshold in Gs |
| *duration*  | Integer | 5 | Number of samples exceeding threshold<br>required to generate interrupt |
| *options* | bitfield | *X_HIGH* \| *Y_HIGH* \| *Z_HIGH* | See table below |

```squirrel
// Configure the Inertial interrupt generator to generate an interrupt
// when acceleration on all three exceeds 1G.
accel.configureInertialInterrupt(true, 1.0, 10, LIS3DH.X_LOW | LIS3DH.Y_LOW | LIS3DH.Z_LOW | LIS3DH.AOI)
```

The default configuration for the Intertial Interrupt generator is to generate an interrupt when the acceleration on *any* axis exceeds 2G. This behavior can be changed by OR'ing together any of the following flags:

| Flag   | Description |
| ------ | ----------- |
| *X_LOW*  | Generates an interrupt when the x-axis acceleration goes below the threshold |
| *X_HIGH* | Generates an interrupt when the x-axis acceleration goes above the threshold |
| *Y_LOW*  | Generates an interrupt when the y-axis acceleration goes below the threshold |
| *Y_HIGH* | Generates an interrupt when the y-axis acceleration goes above the threshold |
| *Z_LOW*  | Generates an interrupt when the z-axis acceleration goes below the threshold |
| *Z_HIGH* | Generates an interrupt when the z-axis acceleration goes above the threshold |
| *AOI*   | Sets the AOI flag *(see ‘Inertial Interrupt Modes’ below)* |
| *SIX_D*  | Sets the 6D flag *(see ‘Inertial Interrupt Modes’ below)* |

#### Inertial Interrupt Modes

The following is taken from the from [LIS3DH Datasheet](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/datasheet/CD00274221.pdf) (section 8.21):

| AOI | 6D  | Interrupt Mode                     |
| --- | --- | ---------------------------------- |
|  0  |  0  | OR combination of interrupt events |
|  0  |  1  | 6-direction movement recognition   |
|  1  |  0  | AND combination of events          |
|  1  |  1  | 6-direction position recognition   |

**Movement Recognition (01)** An interrupt is generate when orientation move from unknown zone to known zone. The interrupt signal stay for a duration ODR.

**Direction Recognition (11)** An interrupt is generate when orientation is inside a known zone. The interrupt signal stay until orientation is inside the zone.

### configureFreeFallInterrupt(*state[, threshold][, duration]*)

The *configureFreeFallInterrupt()* method configures the intertial interrupt generator to generate interrupts when the device is in free fall (acceleration on all axis appraoches 0). The default *threshold* is 0.5G.The default *duration* is five samples.

```squirrel
accel.configureFreeFallInterrupt(true);
```

**Note** This method will overwrite any settings configured with the *configureInertialInterrupt()*.

### configureClickInterrupt(*state[, clickType][, threshold][, timeLimit][, latency][, window]*)

Configures the click interrupt generator:

| Parameter | Type | Default Value | Description |
| --- | --- | --- | --- |
| *state*     | Boolean | N/A | `true` to enable, `false` to disable |
| *clickType* | Constant | *LIS3DH.SINGLE_CLICK* | *LIS3DH.SINGLE_CLICK* or *LIS3DH.DOUBLE_CLICK* |
| *threshold* | Float | 1.1 | Threshold that must be exceeded to be considered a click |
| *timeLimit* | Float | 5 | Max time in *ms* the acceleration can spend above the threshold to be considered a click |
| *latency*   | Float | 10 | Min time in *ms* between the end of one click event and the start of another to be considered a *LIS3DH.DOUBLE_CLICK* |
| *window*    | Float | 50 | Max time in *ms* between the start of one click event and end of another to be considered a *LIS3DH.DOUBLE_CLICK* |

#### Single Click Example

```squirrel
// Configure a single click interrupt
accel.configureClickInterrupt(true, LIS3DH.SINGLE_CLICK);
```

#### Double Click Example

```squirrel
// configure a double click interrupt
accel.configureClickInterrupt(true, LIS3DH.DOUBLE_CLICK);
```

### configureDataReadyInterrupt(*state*)

Enables (*state* is `true`) or disables (*state* is `false`) data-ready interrupts on the INT1 line. The data-ready signal rises to 1 when a new set of acceleration data has been generated and it is available for reading. The interrupt is reset when the higher part of the data of all the enabled channels has been read.

```squirrel
accel.setDataRate(1); // 1 Hz
accel.configureDataReadyInterrupt(true);
```

### configureInterruptLatching(*state*)

Enables (*state* is `true`) or disables (*state* is `false`) interrupt latching. If interrupt latching is enabled, the interrupt signal will remain asserted until the interrupt source register is read by calling *getInterruptTable()*. If latching is disabled, the interrupt signal will remain asserted as long as the interrupt-generating condition persists.

Interrupt latching is disabled by default.

*See sample code in getInterruptTable()*

### getInterruptTable()

The *getInterruptTable()* method reads the LIS3DH’s *INT1_SRC* and *CLICK_SRC* registers, and returns the result as a table with the following fields:

```squirrel
{ "int1": bool,           // true if INT1 created the interrupt
  "xLow": bool,           // true if a xLow condition is present
  "yLow": bool,           // true if a yLow condition is present
  "zLow": bool,           // true if a zLow condition is present
  "xHigh": bool,          // true if a xHigh condition is present
  "yHigh": bool,          // true if a yHigh condition is present
  "zHigh": bool,          // true if a zHigh condition is present
  "click": bool,          // true if any click created the interrupt
  "singleClick": bool,    // true if a single click created the interrupt
  "doubleClick": bool }   // true if a double click created the interrupt
```

In the following example we setup an interrupt for double-click detection:

```squirrel
function interruptHandler() {
    if (int.read() == 0) return;

    // Get + clear the interrupt + clear
    local data = accel.getInterruptTable();

    // Check what kind of interrupt it was
    if (data.doubleClick) {
        server.log("Double Click");
    }
}

i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);
accel <- LIS3DH(i2c, 0x32);

int <- hardware.pinB;
int.configure(DIGITAL_IN, interruptHandler);

// Configure accelerometer
accel.setDataRate(100);

// Set up a double-click interrupt
accel.configureClickInterrupt(true, LIS3DH.DOUBLE_CLICK);
```

In the following example we setup an interrupt for free-fall detection:

```squirrel
function sensorSetup() {
    // Configure accelerometer
    accel.setDataRate(100);
    accel.configureInterruptLatching(true);

    // Setup a free fall interrupt
    accel.configureFreeFallInterrupt(true);
}

// Put imp to Sleep
function sleep(timer) {
    server.log("going to sleep for " + timer + " sec");
    if (server.isconnected()) {
        imp.onidle(function() { server.sleepfor(timer); });
    } else {
        imp.deepsleepfor(timer);
    }
}

// Take reading
function takeReading() {
    accel.getAccel(function(result) {
        if ("err" in result) {
            // check for error
            server.log(result.err);
        } else {
            // add timestamp to result table
            result.ts <- time();
            // log reading
            foreach(k, v in result) {
                server.log(k + ": " + v);
            }
        }
    });
}

function interruptHandler() {
    if (int.read() == 0) return;

    // Get + clear the interrupt + clear
    local data = accel.getInterruptTable();

    // Check what kind of interrupt it was
    if (data.int1) {
        server.log("Free Fall");
    }
    sleep(30);
}

i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);
accel <- LIS3DH(i2c, 0x32);

int <- hardware.pinB;
wake <- hardware.pin1;

int.configure(DIGITAL_IN);
wake.configure(DIGITAL_IN_WAKEUP);

// Handle WakeUp
switch(hardware.wakereason()) {
    case WAKEREASON_TIMER:
        server.log("WOKE UP B/C TIMER EXPIRED");
        takeReading();
        imp.wakeup(2, function() { sleep(30); })
        break;
    case WAKEREASON_PIN:
        server.log("WOKE UP B/C PIN HIGH");
        interruptHandler();
        break;
    default:
        server.log("WOKE UP B/C RESTARTED DEVICE, LOADED NEW CODE, ETC");
        sensorSetup();
        takeReading();
        imp.wakeup(2, function() { sleep(30); })
}
```

### configureHighPassFilter(*filters[, cutoff][, mode]*)

This method configures the high-pass filter.

| Parameter | Type | Default Value | Description |
| --- | --- | --- | --- |
| *filters*     | Constant | N/A | Select the filter(s) to enable/disable by OR-ing together any of the constants found in the filter table below |
| *cutoff* | Constant |*LIS3DH.HPF_CUTOFF1* | See high-pass filter cut-off frequency table below |
| *mode* | Constant | *LIS3DH.HPF_DEFAULT_MODE* | See modes in table below |

#### Filter table

| Filter | Description |
| ------ | --------- |
| *HPF_AOI_INT1* |  High-pass filter enabled for AOI function on interrupt 1 |
| *HPF_AOI_INT2* |  High-pass filter enabled for AOI function on interrupt 2 |
| *HPF_CLICK* | High-pass filter enabled for CLICK function |
| *HPF_FDS* | Filtered data selection. Enables data from internal filter sent to output register and FIFO |
| *HPF_DISABLED* | Disables all filters |

#### High-pass filter cut-off frequency table

| Cutoff | f [Hz] @1Hz | f [Hz] @10Hz | f [Hz] @25Hz | f [Hz] @50Hz | f [Hz] @100Hz | f [Hz] @200Hz | f [Hz] @400Hz | f [Hz] @1.6kHz | f [Hz] @5kHz |
| ---------- | ----------- | ------------ | ------------ | ------------ | ------------- | ------------- | ------------- | -------------- | ------------ |
| *HPF_CUTOFF1* | 0.02 | 0.2 | 0.5 | 1 | 2 | 4 | 8 | 32 | 100 |
| *HPF_CUTOFF2* | 0.008 | 0.08 | 0.2 | 0.5 | 1 | 2 | 4 | 16 | 50 |
| *HPF_CUTOFF3* | 0.004 | 0.04 | 0.1 | 0.2 | 0.5 | 1 | 2 | 8 | 25 |
| *HPF_CUTOFF4* | 0.002 | 0.02 | 0.05 | 0.1 | 0.2 | 0.5 | 1 | 4 | 12 |

#### Mode table

| Filter | Description |
| ------ | --------- |
| *HPF_DEFAULT_MODE* |  Normal mode (reset reading *HP_RESET_FILTER*) |
| *HPF_REFERENCE_SIGNAL* | Reference signal for filtering |
| *HPF_NORMAL_MODE* | Normal mode |
| *HPF_AUTORESET_ON_INTERRUPT* |  Autoreset on interrupt event |

#### Example

```squirrel
// Enable high-pass filter on click and intertial interrupt 1 with auto reset on interrupt event
accel.configureHighPassFilter(LIS3DH.HPF_AOI_INT1 | LIS3DH.HPF_CLICK, null, LIS3DH.HPF_AUTORESET_ON_INTERRUPT);

// Disable high pass filter
accel.configureHighPassFilter(LIS3DH.HPF_DISABLED);
```

### getDeviceId()

Returns the one-byte device ID of the sensor (from the *WHO_AM_I* register). The *getDeviceId()* method is a simple way to test if your LIS3DH sensor is correctly connected.

```squirrel
server.log(format("Device ID: 0x%02X", accel.getDeviceId()));
```

## License

The LIS3DH class is licensed under [MIT License](https://github.com/electricimp/lis3dh/blob/master/LICENSE).
