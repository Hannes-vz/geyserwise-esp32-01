
# Geyserwise simple button press with esp-01

Use [esp-01 from aliexpress](https://www.aliexpress.com/item/1005004626018608.html) or [esp-01 from Micro robotics](https://www.robotics.org.za/ESP-01?search=esp) to trigger red pin on geyserwise with the option of adding temperature reading.

## Esp firmware

I used this guide from [sigmdel](https://www.sigmdel.ca/michel/ha/esp8266/ESP01_AT_Firmware_en.html#v175) to install the firmware on esp-01


## Esp Home setup

Installed [esp home](https://www.esphome.io/components/esp8266) on esp-01

**Here is the my config file for esp home:**
```yaml
esphome:
  name: esp
  friendly_name: esp
  on_boot:
    then:
      switch.turn_off: geyserswitch #Ensures that the default state is off.

#Take note of the board type.
esp8266:
  board: esp07
  restore_from_flash: true

logger:
  level: DEBUG

#Api used for homeassistant
ota:
  password: "redacted"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  reboot_timeout: 0s #This prevents a bug I had where esp whould at random intervals restart.

mqtt:
  broker: 192.168.1.102
  username: redacted
  password: redacted
  reboot_timeout: 0s #Same for the wifi. This fixes it.
  log_topic: homeassistant/switch/esp/logs


binary_sensor:
    #Detect the state for the geyser element itself
  - platform: gpio
    pin: 3
    id: geyserstate
    
switch:
    #Relay for the the red button - Gpio 0 pin as output
  - platform: gpio
    pin: 0
    id: relay
    inverted: true #Gpio 0 pulls pin to ground to trigger relay

    #Loadshedding led that reads the grid frequency to determine state
  - platform: gpio
    pin: 1
    id: shedding_led
    name: shedding_led
    icon: "mdi:led-outline"
    inverted: true

    #Creates a virtual/template switch thats locked to the geyserstate.
    #When turned on a script will run that keeps pressing triggering the relay
    #until the geyserstate is at the desired state.

  - platform: template
    id: geyserswitch
    name: "geyserswitch"
    icon: "mdi:thermometer-water"
    lambda: |-  # used to set the state of the switch template.
      if (id(geyserstate).state) {
        return true;
      } else {
        return false;
      }
    turn_on_action:
      then:
        - script.execute: tempon
        - delay: 2s
        - script.stop: tempon
    turn_off_action:
      then:
        while:
          condition:
            - binary_sensor.is_on: geyserstate
          then:
            - button.press: relay_button
            - delay: 500ms

#This script is necessary because the geyser has a state where the red led is 
#flashing and if the button is pressed you cannot guaranteed the state.
script:
  - id: tempon
    then:
      while:
        condition:
          - binary_sensor.is_off: geyserstate
        then:
          - button.press: relay_button
          - delay: 500ms
  
#Turns relay on and off on button press
button:
  - platform: template
    id: relay_button
    name: "geyserbutton"
    on_press:
      then:
        - switch.turn_on: relay
        - delay: 50ms
        - switch.turn_off: relay

#Using the analog voltage, the temperature can be calculated.
sensor:
  - platform: adc
    pin: A0
    name: "Geyser temperature"
    id: esp_Geyser_temperature
    accuracy_decimals: 9
    update_interval: 1s
    unit_of_measurement: "°C"
    filters: 
      - round: 9
      - median:
          window_size: 11
          send_every: 11
          send_first_at: 1
        # Defines the temperature in voltage range.
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 0.58 -> 40
          - 0.59 -> 41
          - 0.60 -> 42
          - 0.61 -> 43
          - 0.63 -> 44
          - 0.64 -> 45
          - 0.65 -> 46
          - 0.67 -> 47
          - 0.69 -> 48
          - 0.71 -> 49
          - 0.72 -> 50
          - 0.74 -> 51
          - 0.75 -> 52
          - 0.76 -> 53
          - 0.77 -> 54
          - 0.79 -> 55
          - 0.81 -> 56
          - 0.82 -> 57
          - 0.83 -> 58
          - 0.85 -> 59
          - 0.86 -> 60
          - 0.88 -> 61
          - 0.89 -> 62
          - 0.90 -> 63
          - 0.91 -> 64
          - 0.92 -> 65
          - 0.94 -> 66
          - 0.95 -> 67
          - 0.99 -> 68
          - 1.00 -> 69
    
    #I put Led on geyserwise box to indicate when it is loadshedding. https://powerforum.co.za/topic/8451-sunsynk-inverter-monitoring/page/29/#comment-197235
  - platform: mqtt_subscribe
    id: grid_frequency
    topic: homeassistant/sensor/SunSynk/grid_frequency/state
    on_value_range:
      - below: 10
        then: 
          - switch.turn_on: shedding_led
      - above: 10
        then:
          - switch.turn_off: shedding_led

```
Take note you don't need to usd mqtt. I just used it for the grid_frequency because it clashes with the homeassistant api.
The sunsynk monitoring project I used you can find on [powerforum](https://powerforum.co.za/topic/8451-sunsynk-inverter-monitoring/page/29/#comment-197235)

## Circuit Diagrams

![Schematic](https://github.com/Hannes-vz/geyserwise-esp32-01/Pictures/Schematic.jpeg)


