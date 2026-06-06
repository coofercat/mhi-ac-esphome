# Software

Before you go very far, you'll need to flash the ESPHome software onto your ESP32. This will involve connecting it up with a cable to wherever you run ESPHome (in my case, that's my Home Assistant server). How you do this depends on your situation, I'll assume you know how to do it (if not, there are some good guides on the ESPHome website and around the web).

I used the code below to make my project work. It leverages this project: https://github.com/ginkage/MHI-AC-Ctrl-ESPHome. There's lots of great information there, including lots of debugging and troubleshooting information. The main thing to check is the "frame size". If you get a lot of `mhi_ac_ctrl_core.loop error: -2` in the log, then change the frame size. It's all marked up in the code below.

It's perfectly okay to run the ESP without connecting it to an actual aircon unit. It's well worth trying it while it's still on the desk, but it'll emit a lot of `mhi_ac_ctrl_core.loop error: -4` errors. Each of these flashes the on-board LED too, so it's actually quite easy to see if it's running just by looking at it.

```
esphome:
  name: mitsubishi-aircon-1
  friendly_name: mitsubishi-aircon-1
  min_version: 2024.6.0
  platformio_options:
    # Run CPU at 160Mhz to fix mhi_ac_ctrl_core.loop error: -2
    board_build.f_cpu: 160000000L

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: !secret ha_api_encryption
  services:
    - service: set_external_room_temperature
      variables:
        temperature_value: float # temperature to set in Celsius
      then:
        - climate.mhi.set_external_room_temperature:
            temperature: !lambda "return temperature_value;"

ota:
  - platform: esphome
    password: !secret esphome_ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: True

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Mitsubishi-Aircon-1"
    password: "changeMe"

captive_portal:

external_components:
  - source: github://ginkage/MHI-AC-Ctrl-ESPHome@master
    components: [MhiAcCtrl]

# Version 4.2

MhiAcCtrl:
  # Only 20 (legacy) or 33 (includes 3D auto and vertical vanes) possible.
  # If you encounter mhi_ac_ctrl_core.loop error: -2 errors, change the frame_size to 20
  frame_size: 33
  sck_pin: 14
  mosi_pin: 13
  miso_pin: 12
  initial_vertical_vanes_position: 5
  initial_horizontal_vanes_position: 8
  # Update the following to change the default room temp timeout
  room_temp_timeout: 60

button:
  - platform: restart
    name: Restart
    entity_category: diagnostic

climate:
  - platform: MhiAcCtrl
    name: "MHI Air Conditioner"
    temperature_offset: true
    visual_min_temperature: 17.0
    visual:
      temperature_step:
        target_temperature: 0.5
        current_temperature: 0.1

time:
  - platform: homeassistant
    id: homeassistant_time

binary_sensor:
  - platform: MhiAcCtrl
    defrost:
      name: "Defrost"
    vanes_3d_auto_enabled:
      name: "3D Auto"

sensor:
  - platform: uptime
    name: Uptime
  - platform: wifi_signal
    name: WiFi Signal
    update_interval: 60s
  - platform: MhiAcCtrl
    outdoor_temperature:
      name: "Outdoor temperature"
    return_air_temperature:
      name: "Return air temperature"
    outdoor_unit_fan_speed:
      name: "Outdoor unit fan speed"
    indoor_unit_fan_speed:
      name: "Indoor unit fan speed"
    compressor_frequency:
      name: "Compressor frequency"
    indoor_unit_total_run_time:
      name: "Indoor unit total run time"
    compressor_total_run_time:
      name: "Compressor total run time"
    current_power:
      name: "Current power"
    energy_used:
      name: "Energy used"
    indoor_unit_thi_r1:
      name: "Indoor (U-bend) HE temp 1"
    indoor_unit_thi_r2:
      name: "Indoor (capillary) HE temp 2"
    indoor_unit_thi_r3:
      name: "Indoor (suction header) HE temp 3"
    outdoor_unit_tho_r1:
      name: "Outdoor HE temp"
    outdoor_unit_expansion_valve:
      name: "Outdoor unit exp. valve"
    outdoor_unit_discharge_pipe:
      name: "Outdoor unit discharge pipe"
    outdoor_unit_discharge_pipe_super_heat:
      name: "Outdoor unit discharge pipe super heat"
    protection_state_number:
      name: "Compressor protection code"
    error_code:
      name: "Error code"
    vanes_pos:
      name: "Vanes"
    vanesLR_pos:
      name: "Vanes Left-Right"

text_sensor:
  - platform: version
    name: ESPHome Version
  - platform: wifi_info
    ip_address:
      name: IP
    ssid:
      name: SSID
    bssid:
      name: BSSID
  - platform: MhiAcCtrl
    protection_state:
      name: "Compressor protection status"

select:
  - platform: MhiAcCtrl
    vertical_vanes:
      name: Fan Control Up Down
    horizontal_vanes:
      name: Fan Control Left Right
    fan_speed:
      name: "Fan Speed"

switch:
  - platform: MhiAcCtrl
    vanes_3d_auto:
      name: "3D Auto"
```
