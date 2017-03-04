# Driver for the ADS1015/ADS1115 Analog-Digital Converter

This driver consists mostly of the work of Radomir Dopieralski (@deshipu).
I added a few functions and changed the existing ones so it matches better
my needs for a project. Espcially were the functions time-optimized and made
IRQ-proof (no allocation of RAM in the IRQ-prone methods). It was tested with ESP8266 Micropython

## Features

Control the operation of the ADS1x15 ADC and read back the data

## Connection

The ADS1X15 use a I2C interface. So SCL and SDA have to be connected as minimum.
If in continuous mode the CPU shall be triggered, the ALERT/RDY pin has to be
connected too, and obviously VDD, GND and the analog input(2). You might also set the address pin to low (address = 72) or high (address = 73).


## Class

The driver contains the ADS1115 class and the derived ADS1015 class. Since the
two deviecs only differ by the conversion size, the same methods can be applied,
with different interpretation of the parameters.
```
adc = ADS1115(i2c, address, gain)
```
or
```
adc = ADS1015(i2c, address, gain)
```
The default value for the address is 73, for gain is 0. Gain is an index into a
table. It defines the full range of the ADC.  Acceptable values are:
```
0 : 6.144V # 2/3x
1 : 4.096V # 1x
2 : 2.048V # 2x
3 : 1.024V # 4x
4 : 0.512V # 8x
5 : 0.256V # 16x
```

## Methods

### adc.read()
```
value = adc.read(channel, rate)
```
Start a conversion on channel at speed rate and return the value.
Channel is the single input channel (0 .. 3), rate is the conversion rate. Suitable values are  (ADS1015 / ADS1115):
```
0 :  128/8 samples per second
1 :  250/16 samples per second
2 :  490/32 samples per second
3 :  920/64 samples per second
4 :  1600/128 samples per second (default)
5 :  2400/250 samples per second
6 :  3300/475 samples per second
7 :  - /860 samples per Second
```
The first value applies to the ADS1015, the second to the ADS1115. The time
required for a single conversion is 1/samples\_per\_second plus the time needed for communication with the ADC, which is about 1 ms on an esp8266 at 80 MHz. Slower conversion yiled in a less noisy result. The data sheet figures of the ads1x15
are given for the slowest sample rate.

### adc.diff()

```
value = adc.diff(channel1, channel2, rate)
```
Start a conversion for the difference between channel 1 and channel 2 at sampling
speed rate. Suitable values for channel 1 and channel 2 are:
```
0, 1  Difference between channel 0 and 1
0, 3  Difference between channel 0 and 3
1, 3  Difference between channel 1 and 3
2, 3  Difference between channel 2 and 3
```

###  adc.set_conv and adc.read_rev()

Pair of methods for a time optimized sequential reading triggered by a time. For using, you would first set the conersion parameters with set_conv() and then get
the values in a timer callback function with read_rev().
```
adc.set_con(channel, rate)
value = adc.read_rev()
```
The definition of channel and rate are the same as with adc.read(). The methods
read_rev() reads first the last conversion value back, and the starts a new
conversion. Care has to be taken, that the time needed for conversion and
communication is shorter than the timer period plus the time needed to process the data.
A sample code is shown below. The timing jitter observed on an esp8266 was
about 1 ms, but the time period is defined by the micro's timer, which has
it's own issues.

###  adc.alert_start() and adc.alert_read()

Pair of methods to start a contious sampling on a single channel and trigger
an alert once a certain threshold is reached,
```
adc.alert_start(channel, rate, threshold)
value = adc.alert_read()
```
The values of channel and rate are the same as for adc.read(). Threshold tells
trigger value anshould be within the range of the adc, 0..32767 for ADC1115 and
0..2047 for ADS1015. Rate should be chosen according to the input signal
change rate and the precision needed.


###  adc.conversion_start() and adc.alert_read()

Pair of methods to start a contious sampling on a single channel and trigger
an alert at every sample. This functions pair to be used in an irq-based set-up.
```
adc.conversion_start(channel, rate)
value = adc.alert_read()
```
The values of channel and rate are the same as for adc.read(). The timing jitter
observed is a few µs. However the ADC's timer is not very precise. In
applications where this is of importance some control and calibration of
the returned timing pattern has to be done.

###  adc.\_write_register()

Write to a register of the ADC.
```
adc._write_register(register, value)
```
Register is the number of the register according to the data sheet, value a 16 bit
quantity coded accordingly. Register numbers.
```
0: Conversion Register; holding the converted value
1: Configuration Register
2: Low Threshold
3: High Threshold
```
###  adc.\_read_register()

Read a register of the ADC.
```
value = adc._read_register(register, value)
```
Register is the number of the register according to the data sheet, value a 16 bit
quantity coded accordingly. Reading the conversion register returns the value of
the most recent sampling. Bit 15 of the config register is set when a conversion
is finished.

# Sample Code

## Continuous sampling triggered by the timer
```
from machine import I2C, Pin, Timer
import ads1x15
from time import sleep_ms, ticks_ms, ticks_us
from array import array

addr = 72
gain = 1

_BUFFERSIZE = const(512)
#
# Interrupt service routine zum messen
# diese wird vom Timer-interrupt aktiviert
#
def sample(x):
    global index_put, ads, irq_busy, data, timestamp
    if irq_busy:
        return
    irq_busy = True
    if index_put < _BUFFERSIZE:
        timestamp[index_put] = ticks_us()
        data[index_put] = ads.read_rev()
        index_put += 1
    irq_busy = False

data = array("h", [0] * _BUFFERSIZE)
timestamp = array("L", [0] * _BUFFERSIZE)
irq_busy = False

index_put = 0
ADC_RATE = 5

i2c = I2C(scl=Pin(5), sda=Pin(4), freq=400000)
ads = ads1x15.ADS1115(i2c, addr, gain)
# set the conversion rate tp 860 SPS = 1.16 ms; that leaves about
# 3 ms time for processing the data with a 5 ms timer
ads.set_conv(0, 7) # start the first conversion
ads.read_rev()
sleep_ms(ADC_RATE)
tim = Timer(-1)
tim.init(period=ADC_RATE, mode=Timer.PERIODIC, callback=sample)

while index_put < _BUFFERSIZE:
    pass

tim.deinit()

# at that point data contains the sampled values, and
# timestamp the timer ticks which correlate to the conversion time
#
```
The timing jitter seen here was +/- 500 us, with 90% ~ 5 ms and 5% each at about 450 and 550 µs. The timing inteference occured every second. At 160MHz clock, the
Jitter was about +/- 50 µs

## Continuous sampling trigged by the ADC

```
from machine import I2C, Pin, Timer
import ads1x15
from array import array

addr = 72
gain = 1

_BUFFERSIZE = const(512)
#
# Interrupt service routine zum messen
# diese wird vom Timer-interrupt aktiviert
#
def sample_auto(x):
    global index_put, ads, data
    if index_put < _BUFFERSIZE:
        data[index_put] = ads.alert_read()
        index_put += 1

data = array("h", [0] * _BUFFERSIZE)
index_put = 0

i2c = I2C(scl=Pin(5), sda=Pin(4), freq=400000)
irq_pin = Pin(13, Pin.IN, Pin.PULL_UP)
ads = ads1x15.ADS1115(i2c, addr, gain)
ads.conversion_start(0, 5)

irq_pin.irq(trigger=Pin.IRQ_FALLING, handler=sample_auto)

while index_put < _BUFFERSIZE:
    pass

irq_pin.irq(handler=None)
#
# at that point data contains 512 samples acquired at the given rate
#
```
The sampling rate achieved in my test was 251,9 SPS or 3.97 ms/sample, as told
by the ESP8266 clock, which may not be precise either.
