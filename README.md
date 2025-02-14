# inetbox2mqtt -> Control your TRUMA heater over a MQTT broker
If you use this code give me a star, please. Thanks in advance.
## microPython version for ESP32 and RP pico w
- **communicate over MQTT protocol to simulate a TRUMA INETBOX**
- **include optional Truma DuoControl over GPIO-connections**
- **include optional MPU6050 Sensor for spiritlevel-feature**
- **tested with both ports (ESP32 / RPi pico w 2040)**

## Acknowledgement
The software is a derivative of the github project [INETBOX](https://github.com/danielfett/inetbox.py) by Daniel Fett. 
Thanks to him, as well as the preliminary work of [WoMoLIN](https://github.com/muccc/WomoLIN), these cool projects have become possible.

This project here was developed and tested for an ESP32 (first generation) with 4 MB memory. The software also works on other ESP32 models and probably, with small adjustments (UART address, pins), also on other hardware. The tests on a Raspberry Pi Pico W were successful, too. I will not always explicitly mention the RPi pico w in the following. The respective points apply to this chip as well. The minor deviations can be found at the end in the section **Running on RPI Pico W** for details. 

## Disclaimer
I have tested my solution for the ESP32 in about 10 different environments so far, including my own TRUMA/CPplus version. Most of the tests ran straight out of the box. 

The LIN module for the ESP32/RPi pico works logically a bit different than Daniel's software. On the other hand, the module in the current version for the ESP32/RPI pico w has proven to be very stable and CPplus-compatible. 

**Nevertheless, it should be mentioned here that I do not assume any liability or guarantee for its use.**

## Electrics
For the wiring of the LIN bus via the TJA1020 to the UART, please refer to the project [INETBOX](https://github.com/danielfett/inetbox.py) mentioned above. 

On the **ESP32** we recommend the use of UART2 (**Tx - GPIO17, Rx - GPIO16**):

![1](https://user-images.githubusercontent.com/65889763/200187420-7c787a62-4b06-4b8d-a50c-1ccb71626118.png)

On the **RPI pico w** we recommend the use of UART1 (**Tx - GPIO04, Rx - GPIO05**):

![grafik](https://user-images.githubusercontent.com/10268240/201338579-29c815ca-e5ef-4f25-b015-1749a59b3e99.png)

These are to be connected to the TJA1020. No level shift is needed (thanks to the internal construction of the TJA1020). It also works on 3.3V levels, even if the TJA1020 is operated at 12V. 

## MQTT topics - almost the same, but not exactly the same
The MQTT commands ***(set-topics)*** are identical to Daniel's command usage [INETBOX](https://github.com/danielfett/inetbox.py). 

We will try to keep the topics and payloads as equal as possible. However, the published topics look a little different. The ESP32 only sends selected topics and omits all the timers, checksum, command_counter, etc. (all self-explanatory). If there is a need for adaptation, I am at your disposal. The timing for the sending of the topic has also been modified, i.e. the ESP32 only sends a topic if something has changed for the individual topic. This is different with Daniel's program, which always writes the whole status register. With my program, there is an alive topic, which shows the status of the connection to the MQTT broker and to the CPplus (see also ESP32 GPIOs and LEDs).

## Status LEDs - Debugging will be easier
Since the ESP32 has so many GPIOs, I programmed two LEDs. The LEDs are to be connected in negative logic:

            GPIO-pin ----- 300-600 Ohm resistor ----- LED ----- +3.3V

GPIO12 indicates when the MQTT connection is up. 

GPIO14 indicates when the connection to the CPplus is established.

The search for ***connection errors*** (e.g. missing LIN signal, swapping rx/tx, defective TJA 1020) can be very time-consuming and annoying. Therefore GPIO14 has been supplemented to the extent that the LED already flickers slightly when signals are registered on the port rx line (equivalent is tx-output on the TJA 1020). This also happens before an ***CPplus INIT***, in other words a registration of the inetbox2mqtt at the CPplus has taken place. In this way, it can be detected very quickly whether there are connection problems on the rx line. If the INIT process has taken place, the LED lights up brightly. 

*Attention: The CPplus does not transmit continuously, therefore transmission pauses of 15-25s are normal.* 

## Integration of Truma DuoControl
Another functionality has been added. This is an additional function, at the moment not implemented in [INETBOX](https://github.com/danielfett/inetbox.py). 

The status changes of two GPIO inputs (GPIO18 and GPIO19) and the GPIO outputs (GPIO22 and GPIO23) are now also published to the broker. The reaction time for status-changes is approx. 10s. 

The associated topics are

- service/truma/control_center/duo_ctrl_gas_green
- service/truma/control_center/duo_ctrl_gas_red
- service/truma/control_center/duo_ctrl_i
- service/truma/control_center/duo_ctrl_ii

with the payloads ON/OFF. The outputs can be controlled with the SET commands

- service/truma/set/duo_ctrl_i
- service/truma/set/duo_ctrl_ii

Inputs and outputs are inverted, i.e. the inputs react with ON when connected to GND, the outputs switch to GND level when ON.

## Integration of MPU6050 for spiritlevel-Feature
A second optional feature has been added. For leveling of an RV-car a
MPU6050 IMU (inertial measurement unit) can be connected to the I2C bus. 

*Attention: The original version of this add-on used the GPIO00/01 for the I2C communication. Unfortunately, there is a conflict with the system UART (Tx=GPIO01). As a consequence, no debugging output was possible after initialising the I2C interface. It is important for all those who want to update to the current version that the pins for SDA and SCL have changed!*

Different pins are required here:

For **ESP32** please use I2C bus with SDA (GPIO26) and SCL (GPIO25).
For **RPI pico w** please use I2C bus 1 with SDA (GPIO02) and SCL (GPIO03).

Every 500ms the acceleration and gyroscopic values are read and combined and filtered by a [Kalman-Filter](https://www.navlab.net/Publications/Introduction_to_Inertial_Navigation_and_Kalman_Filtering.pdf).
For the moment the result (pitch and roll angle) is published via MQTT
every 10s.

The associated topics are

- service/spiritlevel/spirit_level_pitch
- service/spiritlevel/spirit_level_roll

## Alive topic
Short digression: The CPplus only sends 0x18 (with parity it is 0xD8) requests if an INETBOX is registered. This can be recognised by the third entry in the index menu on the CPplus, among other things. The port answers these requests. Only when it receives 0x18 messages, the connection to the CPplus is established and the registration has taken place. This makes it easy to find out if there is an electrical problem. If the LED (GPIO14, see Status LEDs) is lit, communication with the CPplus is established. The port also outputs this as an "alive" topic via the MQTT connection (approx. every 60 sec): connection OK => payload: ON; connection not OK => payload: OFF.

## Good news for Home Assistant users
To make it even easier to set up the INETBOX simulator together with the [Home Assistant](https://www.home-assistant.io/) smarthome system, the [auto-discovery function of home assistant](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery) is implemented.

After the port has connected to the MQTT broker, it sends the installation codes. If the Home Assistant server is also connected to the MQTT broker, the entities are all generated automatically. They all begin with 'truma_'. Since they are not persistent, this automatic generation also takes place when the Home Assistant server is restarted.

The Home Assistant's own MQTT broker, which is available as an add-on, can also be used. If you use other smart home systems, you can simply ignore the messages. In the [docs](https://github.com/mc0110/inetbox2mqtt/tree/main/doc), there is an example of a frontend solution in Home Assistant.

## MicroPython
After the first tests, I was amazed af how good and powerful the [microPython.org](https://docs.micropython.org/en/latest/) platform is. However, the software did not run with a kernel from July (among other things, the bytearray.hex was not implemented there yet).

The micropython MQTT packages are currently still experimental and cannot yet establish MQTT TLS connections. Thanks a lot to Thorsten [tve/mqboard](https://github.com/tve/mqboard) for his work.

## Installation instructions
### Alternative 1: With esptool
The .bin file contains both the python and the .py files. This allows the whole project to be flashed onto the ESP32 in one go. For this, you can use the esptool. In my case, it finds the serial port of the ESP32 automatically, but the port can also be specified. The ESP32 must be in programming mode (GPIO0 to GND at startup). The command to flash the complete .bin file to the ESP32 is:

    esptool.py write_flash 0 flash_dump_esp32_lin_v083_4M.bin

This is not a partition but the full image for the ESP32 and only works with the 4MB chips. The address 0 is not a typo.

After flashing, please reboot the ESP32 and connect it to a serial terminal (e.g. miniterm, putty, serialport) (baud rate: 115200) fur further steps like checking if everything is working ok.

### Alternative 2: With a microPython IDE
Handling the *.py files and adapting and testing them is much easier if you use a microPython IDE. I can recommend the [Thonny IDE](https://thonny.org/), which is available on various platforms (Windows, macOS, Linux) and can also handle different hardware (e.g. ESP8266, ESP32, Raspberry Pi 2).

To do this, you first have to install an up to date microPython version, to be found at [micropython/download](https://micropython.org/download/). My tests were done with upython-version 19.1-608.

Then all .py files (including the lib sub-directory) must be loaded onto the ESP32 via the IDE.

### Execution
If you put all files into the root directory of the ESP32 - either as complete .bin file with the esptool, or as .py files with a microPython IDE - the ESP32 will start the program after a reboot. You can abort a program in the IDE with CTRL-C. Since the files are set up in such a way that the program starts directly after booting, the program must first be interrupted. This is done with CTRL-C.

After that, you have full control with Thonny or another microPyhton IDE and can change, save, and execute the .py files.

### Credentials
On first run of the program, the ESP32 will ask for the credentials for the MQTT broker (IP, Wifi SSID and password, username and password). 
These are written in an encrypted file *credentials.dat* on the ESP32.

The entries are then displayed again for confirmation, and the query is repeated until you have confirmed with ***yes***.

The process of providing the credentials for an initial setup does not have to be repeated, as long as the file *credentials.dat* remains on the ESP32.
 
You can't directly write (and edit) this file. If you want to generate the file *credentials.dat*, please refer to my library [crypto_keys](https://github.com/mc0110/crypto_keys). There you will find an example of how to generate the file using Python. 

For placing the files and creating the credentials on the ESP32, it does not need to be connected to the CPplus.

If everything is correctly set up and the ESP32 is rebooted, it should connect to the MQTT broker with a `connected` confirmation message.

Then you can establish the connection between the ESP32 and the LIN bus. This connection is not critical and can be disconnected at any time and then re-established. It should not be necessary to re-initialise the CPplus.

### Running on a RPi pico W
Micropython can be installed very easily on the RPI pico W. Please use a current release (younger than 19.1 Oct.22) of Python here - analogous to the note for the ESP32. The installation is explained very well on the [Foundation pages](https://www.raspberrypi.com/documentation/microcontrollers/raspberry-pi-pico.html).
 
Fortunately, the entire **inetbox2mqtt** software also runs on this port. Since the GPIO pins for the support leds are present on the RPi-board, just like the GPIO pins for the connection to the Truma DuoControl, no changes are necessary here. The hardware is recognized by the software, therefore 
nothing is to do. If you want to use the **spiritlevel-addon**, then please note the corresponding pins for SDA (GPIO2) for SCL (GPIO3).
