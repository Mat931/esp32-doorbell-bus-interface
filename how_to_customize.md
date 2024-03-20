After uploading the [basic configuration](https://github.com/Mat931/esp32-doorbell-bus-interface/blob/main/how_to_flash.md#esphome-example-configuration) to the ESP32 and connecting the board to your doorbell intercom you need to customize the configuration for your specific system.

## Basics

- Press the `Logs` button in the ESPHome dashboard to observe received messages.
- Alternatively after connecting the board to Home Assistant you can use the debug sensor named `Doorbell Intercom` for long-term observation of received messages.
- Received messages are structured like this:
  - `55.FF.XX...XX (10)`: Raw data in hexadecimal format and message length.
  - `[XXXX > XXXX]`: Source and destination addresses, `»` means the message is a retransmission. If you see six characters (`XXXXXX`) instead of four your system is using three-byte addresses. When transmitting messages you need to add `three_byte_address: true` in the ESPHome config. More info about addresses is available [here](https://github.com/Mat931/esp32-doorbell-bus-interface/wiki/Address-Types).
  - `Type: XX`: Message type, if it's 0x80 or above then the message is a reply. For example if the initial message has type `01` then the reply type will be `81`. More info about message types is available [here](https://github.com/Mat931/esp32-doorbell-bus-interface/wiki/Message-Types).
  - `Data: XX` Data attached to the message (optional). The meaning of the data depends on the message type.

## Receiving doorbell notifications

- Ring your doorbell and write down the exact time.
- Check the log and write down all messages that were transmitted at that time or 1-2 seconds later.
- To avoid false positives start a manual video or call to the outdoor station and compare all received messages to the ones written down in the last step. If some messages are identical they can't be used for doorbell notifications. Remove them from your notes.
- Find your indoor station's address. It was probably the source or destination of most received messages. Indoor station addresses can start for example with `10XX` or `0001XX`. The last two characters of the address represent the apartment number set by the dials on the back of your indoor station and are displayed in hexadecimal format. For example if the dials are set to `12` (decimal) then the last two characters of the address will be `0c` (hexadecimal).
- Messages that have your indoor station address as source, destination or in the data can be used for doorbell notifications. For increased reliability it is best to use multiple message types.

```yaml
      - if:
          condition:
            or:
              # Type 01, Outdoor Station (0x2001) > Indoor Station #12 (0x100c)
              - lambda: 'return (x.get_message_type() == 0x01) && (x.get_source_address() == 0x2001) && (x.get_destination_address() == 0x100c);'
              # Type 81, Indoor Station #12 (0x100c) > Outdoor Station (0x2001)
              - lambda: 'return (x.get_message_type() == 0x81) && (x.get_source_address() == 0x100c) && (x.get_destination_address() == 0x2001);'
              # Type 0a, Outdoor Station (0x2001) > Gateway (0x3001), Data contains indoor station address (0x10, 0x0c)
              - lambda: 'std::vector<uint8_t> d = {0x01, 0x10, 0x0c}; return (x.get_message_type() == 0x0A) && (x.get_source_address() == 0x2001) && (x.get_destination_address() == 0x3001) && (x.get_data() == d);'
          then:
          - binary_sensor.template.publish:
              id: doorbell_outdoor
              state: ON
          - binary_sensor.template.publish:
              id: doorbell_outdoor
              state: OFF
```

- For indoor doorbells fewer messages are transmitted.

```yaml
      - if:
          condition:
            or:
              # Type 11, Source: Indoor Station (0x100C)
              - lambda: 'return (x.get_message_type() == 0x11) && (x.get_source_address() == 0x100c);'
              # Type 91, Destination: Indoor Station (0x100C)
              - lambda: 'return (x.get_message_type() == 0x91) && (x.get_destination_address() == 0x100c);'
          then:
          - binary_sensor.template.publish:
              id: doorbell_indoor
              state: ON
          - binary_sensor.template.publish:
              id: doorbell_indoor
              state: OFF
```

## Opening your front door

- Press the door opener button on your indoor station and write down the exact time. Activating the display of your indoor station might already send some irrelevant messages.
- Check the log and write down all messages that were transmitted at that time or 1-2 seconds later.
- Look for a message with type `0d` or `0e`. Use that message's addresses, type and data in your config. If there are none, you need to find the right message by trial and error.

```yaml
on_...:
  - remote_transmitter.transmit_abbwelcome:
      source_address: 0x100c # your indoor station address
      destination_address: 0x4001 # door address
      message_type: 0x0d # unlock door
      data: [0xab, 0xcd, 0xef] # message data
```

- To provide feedback to Home Assistant when the door has been opened look for a message with type `8d` or `8e`.

```yaml
      - if:
          condition:
            and:
              - lambda: 'return (x.get_message_type() == 0x8d);' # Door opener response
              - lambda: 'return (x.get_source_address() == 0x4001);' # Door address
          then:
          - lock.template.publish:
              id: main_door
              state: UNLOCKING
          - delay: 3s # How long the door usually stays unlocked
          - lock.template.publish:
              id: main_door
              state: LOCKED
```
