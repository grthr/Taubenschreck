# Taubenschreck

Quick and dirty experiment to defer pigeons from my balcony.

Components:

* Raspberry Pi
* PIR sensor
* Bluetooth speaker
* Power supply

## Main unit

Raspberry Pi Zero W with Raspbian Lite powered by a standard USB power supply.

## Sensor

PIR sensor HC-SR501. VCC is 5V. Connect VCC, OUT and GRD to GPIO of the pi.

```python
import RPi.GPIO as gpio
import time

gpio_pin = 13 # pin number on the board where OUT is connected
gpio.setmode(gpio.BOARD)
gpio.setup(gpio_pin, GPIO.IN)

while True:
    print(gpio.input(gpio_pin)) # prints 1 in case of motion detected, 0 otherwise
    time.sleep(0.5)
```

## Connect the speaker

Any bluetooth speaker compatible with A2DP.

Install:

```bash
sudo apt install bluealsa
sudo service bluealsa start
```

Connect:

```bash
sudo bluetoothctl scan on
BT_ADDR=12:34:56:78:90:AB
# find the address of the device you wish to connect to, e.g. 12:34:56:78:90:AB
sudo bluetoothctl pair $BT_ADDR
sudo bluetoothctl trust $BT_ADDR
```

## Play sample audio file

`aplay` just needs two parameters: PCM audio device and audio file to be played. Some sameple audio files for speaker test available in `/usr/share/sounds/alsa/`.

The parameter of the audio device in the case of `bluealsa` needs further configuration using sub-parameters `SRV`, `DEV` and `PROFILE`.

Test audio connection:

```bash
aplay -D bluealsa:SRV=org.bluealsa,DEV=$BT_ADDR,PROFILE=a2dp /usr/share/sounds/alsa/Front_Center.wav
```

If you need to adjust the volume, check `amixer`:

```bash
amixer # prints current mixer settings
amixer set PCM 50% # sets new volume to 50 %
```

## Get alarm audio file

I downloaded the sound of a peregrine from this website: https://www.greifvogel.info/ruf.php

Then converted ogg to wav using a free online audio converter and downloaded to the pi as `~/wanderfalke.wav`.

## Write the script

```python
import RPi.GPIO as gpio
import time
import os

bt_addr = '12:34:56:78:90:AB' # BT speaker device address

# Setup GPIO
gpio_pin = 13 # pin number on the board where OUT is connected
gpio.setmode(gpio.BOARD)
gpio.setup(gpio_pin, GPIO.IN)

# Setup command
bluealsa_settings = {
    'SRV': 'org.bluealsa',
    'DEV': bt_addr,
    'PROFILE': 'a2dp'
}
audio_device = ','.join(f'{key}={value}' for (key, value) in bluealsa_settings.items())
audio_file_path = '~/wanderfalke.wav'
command = f'aplay -D {audio_device} {audio_file_path}'

time.sleep(2)

while True:
    if gpio.input(gpio_pin):
        os.system(command)
        time.sleep(10) # extend interval to prevent another alarm
    time.sleep(0.5)
```

## Space for improvements

* Log data from motion detection to influx DB to keep track of the setup. Needs some more sophisticated loop/sleep mechanism to not log every 0.5 seconds.
* Bluetooth speaker needs power to charge/survive. In my case, it has a 5V mini USB plug. Might get enough power from the pi?
* Some bluetooth speakers shut down if no audio is played. Might be better to not use bluetooth here as wireless is not needed.
* Ideally power the pi using a combination of powerbank and solar panel.
