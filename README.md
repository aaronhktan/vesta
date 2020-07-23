# <img src="/docs/icon/icon.png?raw=true" height="48"> Vesta - Homebridge Plugin Collection

*Vesta is the same as the Earth; under both of them is a perpetual fire; the earth and the hearth are symbols of the home.*
—Ovid, Fasti VI

This repository brings together all the plugins I've written for my indoor environment smart home automation project.

## Components
- `homebridge-bmp280`: Temperature and barometric air pressure.
- `homebridge-dht22`: Temperature and humidity.
- `homebridge-pms7003`: Particulate matter, dust, PM2.5/PM10.
- `homebridge-sgp30`: Equivalent CO2, volatile organic compounds.
- `homebride-soil-sensor`: Analog capacitive soil moisture sensor readings.
- `homebridge-veml6030`: White and ambient light brightness.
- `homebridge-display`: Displays data on 16x2 character display.

## Technical details
### Hardware
- Raspberry Pi! Most of these plugins should work on both the Raspberry Pi 3 and Raspberry Pi Zero. Have not tested on others.
- The soil sensor code runs on either (1) Onion Omega + Arduino Dock, or (2) an ESP8266 running MicroPython.
- I've added a Fritzing diagram under `docs/` to show how everything is wired up.

### Protocols
- **I2C, SMBus**: SGP30, VEML6030. These both rely on `i2c-dev` and `ioctl` from Linux.
- **SPI**: BMP280. This relies on `spidev` and `ioctl` from Linux.
- **UART**: PMS7003. This relies on `termios` from Linux. Also uses `epoll` for asynchronous I/O!
- **GPIO toggling**: DHT22. This uses the `BCM2835` library.
- **MQTT**: Most of the plugins use the `MQTT.js` library, with the exception of the soil sensor, which relies on `paho-mqtt` and `umqtt` for the Onion Omega and MicroPython parts, respectively.

### Data flow overview
- Homebridge runs on the Raspberry Pi, and has access to all the GPIO pins (provided that the `homebridge` user has been added to the appropriate groups).
- At a set interval (default 60 seconds, but configurable by plugin), Homebridge polls for data from all the devices connected via hardware.
    - The Node.js plugins invoke the C/Linux API(s) to read data from the hardware modules from JavaScript. This is done via `node-bindings` (to allow Node.js to talk to the native C++ binding), and `node-addon-api` (the C++ API that implements the bridge between JavaScript and C APIs).
    - Data that is read is then (1) propagated over MQTT, (2) saved to a local store of for Eve's HomeKit app to consume.
- Independently, the soil moisture sensors continuously monitor the soil moisture of the pot they're sitting in.
    - In the Omega + Arduino setup, the Arduino dock reads the voltage (built-in 10-bit ADC), and "prints" these over UART for the Omega to consume. The Omega uses a blocking read on the `tty` port where it is connected; once it receives data, it transmits the reading over MQTT to the Raspberry Pi.
    - On the ESP8266, the 10-bit ADC is limited to a max voltage of 1V, but the NodeMCU board I have has a built-in voltage divider to safely connect the soil moisture sensor. Once it reads a voltage, it transmits the reading over MQTT to the Raspberry Pi.
    - The plugin running on the Raspberry Pi monitors for messages over MQTT. When it receives a reading, averages are calculated and data is stored.
- Outside of Homebridge, when the Raspberry Pi boots, systemd launches the `homebridge-display` script. This script subscribes to (some) of the MQTT topics, and displays the data sent on those topics.

## Picture
<img src="/docs/fritzing/vesta.png?raw=true">

