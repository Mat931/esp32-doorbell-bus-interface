The ESP32 on this board must be programmed with a firmware like [ESPHome](https://esphome.io/).

## ESPHome Example Configuration
```yaml
esp32:
  board: esp32dev
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_FREERTOS_UNICORE: y # Only required if you have a single core ESP32

esphome:
  name: abb-welcome-demo
  friendly_name: "ABB Welcome Demo"
  on_boot:
    - lock.template.publish:
        id: front_door
        state: LOCKED
    - binary_sensor.template.publish:
        id: doorbell_indoor
        state: OFF
    - binary_sensor.template.publish:
        id: doorbell_outdoor
        state: OFF

wifi:
  networks:
    - ssid: !secret wifi_ssid
      password: !secret wifi_password

logger:

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

remote_transmitter:
  pin: GPIO26
  carrier_duty_percent: 100%
  non_blocking: true

remote_receiver:
  pin:
    number: GPIO25
    mode:
      input: True
  dump: [abbwelcome]
  filter: 8us
  tolerance:
    type: time
    value: 26us
  idle: 1500us
  buffer_size: 15kB
  rmt_symbols: 320
  clock_resolution: 500000
  on_abbwelcome:
    then:
      - lambda: "char buf[192]; x.format_to(buf, 192); id(doorbell_intercom).publish_state(buf);"
      - if:
          condition:
            and:
              - lambda: "return (x.get_message_type() == 0x8d);" # unlock door response
              - lambda: "return (x.get_source_address() == 0x4001);" # door address
          then:
            - lock.template.publish:
                id: front_door
                state: UNLOCKING
            - delay: 3s # time how long the door usually stays unlocked
            - lock.template.publish:
                id: front_door
                state: LOCKED
      - if:
          condition:
            and:
              - lambda: "return (x.get_message_type() == 0x11);" # doorbell indoor
              - lambda: "return (x.get_source_address() == 0x1001);" # your indoor station address
          then:
            - binary_sensor.template.publish:
                id: doorbell_indoor
                state: ON
            - binary_sensor.template.publish:
                id: doorbell_indoor
                state: OFF
      - if:
          condition:
            and:
              - lambda: "return (x.get_message_type() == 0x01);" # doorbell outdoor
              - lambda: "return (x.get_source_address() == 0x2001);" # outdoor station address
              - lambda: "return (x.get_destination_address() == 0x1001);" # your indoor station address
          then:
            - binary_sensor.template.publish:
                id: doorbell_outdoor
                state: ON
            - binary_sensor.template.publish:
                id: doorbell_outdoor
                state: OFF

binary_sensor:
  - platform: template
    name: Doorbell indoor
    id: doorbell_indoor
    icon: mdi:bell
    filters:
      - delayed_off: 1s
  - platform: template
    name: Doorbell outdoor
    id: doorbell_outdoor
    icon: mdi:bell
    filters:
      - delayed_off: 1s

sensor:
  - platform: internal_temperature
    name: "Internal Temperature"

text_sensor:
  - platform: template
    name: "Doorbell Intercom" # Receive debug messages
    id: doorbell_intercom
    icon: mdi:bell-circle
    update_interval: never

lock:
  - platform: template
    name: "Front Door"
    id: front_door
    on_unlock:
      - then:
          - lock.template.publish:
              id: front_door
              state: LOCKED
    lock_action:
      - then:
          - lock.template.publish:
              id: front_door
              state: LOCKED
    unlock_action:
      - then:
          - remote_transmitter.transmit_abbwelcome:
              source_address: 0x1001 # your indoor station address
              destination_address: 0x4001 # door address
              three_byte_address: false # address length of your system
              message_type: 0x0d # unlock door
              data: [0xab, 0xcd, 0xef] # door opener secret code, see receiver dump

button:
  - platform: restart
    name: "Reboot"
  - platform: safe_mode
    name: "Reboot (Safe Mode)"
  - platform: template
    name: "Stop Doorbell"
    on_press:
      - remote_transmitter.transmit_abbwelcome:
          source_address: 0x1001 # your indoor station address
          destination_address: 0x2001 # outdoor station address
          three_byte_address: false # address length of your system
          message_type: 0x02 # end call (stops your doorbell ringtone)
          data: [0x00]

status_led:
  pin:
    number: GPIO2
    ignore_strapping_warning: true
```

## Flashing

For flashing you need a USB-to-serial adapter that supports 3.3V and has the required pins. The board includes an auto-reset circuit so you don't need to manually ground GPIO0 or press any buttons during flashing. The rest of the flashing process is identical to other ESP32 and ESPHome devices.

| USB-to-serial adapter | Programming header of the board |
| --------------------- | ------------------------------- |
| VCC                   | 3V3                             |
| GND                   | GND                             |
| TX                    | RX                              |
| RX                    | TX                              |
| DTR                   | DTR                             |
| RTS                   | RTS                             |

If your USB-to-serial adapter can't supply enough power to flash the ESP32 you can disconnect `3V3` and connect an external power supply (12-30V DC) to the `BUS+` and `BUS-` pins.

 ![PCB image](https://github.com/Mat931/esp32-doorbell-bus-interface/blob/main/images/5.png)

## Installation

Before connecting the device to the ABB-Welcome bus it is recommended to check the bus voltage. It should be between 20V and 30V DC. Above 36V the device can take damage. Connect the positive wire to `BUS+` and the negative wire to `BUS-`. If the TX LED lights up continuously after connecting you should disconnect the wires and check your yaml configuration. The device has protection against reverse polarity and a resettable fuse.

Note: You are responsible for connecting the device to your system and for the firmware you upload. The author of this project takes no responsibility for any damages.
