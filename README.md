# Control air purifier 4 compact with mushroom fan card
![preview](https://github.com/marcoleder/cpa4-mushroom/assets/32636827/5e71cdd7-22e3-4365-a483-dfc6f0c78f3b)\
Welcome to this step-by-step guide to controlling the xiaomi smart air purifier 4 compact using the mushroom fan card in Home Assistant. This guide is based on using Home Assistant's <a href="https://www.home-assistant.io/integrations/fan.template" target="_blank">fan template</a> to create a custom fan entity, which allows for better integration with the mushroom fan card (ability to control speed and modes).

## Requirements

To use this setup, the following components need to be installed through the <a href="https://hacs.xyz/" target="_blank">Home Assistant Community Store (HACS)</a>:

1. <a href="https://github.com/piitaya/lovelace-mushroom" target="_blank">`lovelace-mushroom` by piitaya</a>
2. <a href="https://github.com/al-one/hass-xiaomi-miot" target="_blank">`xiaomi-miot`</a>

If you're unfamiliar with HACS or how to integrate Xiaomi devices via miot into Home Assistant, please consult the guides online. Apart from these, you need the ability to modify your `configuration.yaml` file, which can be done either through the File editor add-on in Home Assistant or through Docker if that's your setup. Once you have integrated the purifier via the miot integration, you can start with the configuration below.

## Configuration

Remember to replace fan.YOUR_XIAOMI_FAN_ENTITY and fan.YOUR_DESIRED_FAN_NAME with your actual fan entity ID and the desired name for your fan, respectively.

### Step 1: Adding to Configuration.yaml
Copy the provided YAML code into your Home Assistant's `configuration.yaml` file.

The `sensor` section creates a combined sensor that shows the fan's power state, speed, and mode.
It has been migrated from sensor: - platform: template to the new syntax to accomodate version 2026.x.x breaking changes.
Its implementation is now directly in the fan section at the end of this section (Step 1).

---
The `input_select` provides a way to switch between different modes ("auto" and "favorite").
```yaml
# Fan oscillation modes (without sleep mode)
input_select:
  air_purifier_mode:
    name: Air Purifier Mode
    options:
      - "auto"
      - "favorite"
    initial: "auto"
```
---
The `automation` is WIP. ATM, it does not correctly set the state upon reboot but updates fine when the mode is changed in the xiaomi app.
```yaml
# Automation to correctly update state of fan oscillation modes
alias: FanInit
description: Set Input Select based on Air Purifier Mode
trigger:
  - platform: homeassistant
    event: start
  - platform: template
    value_template: "{{ state_attr('fan.YOUR_XIAOMI_FAN_ENTITY', 'air_purifier.mode') }}"
condition: []
action:
  - service: input_select.select_option
    target:
      entity_id: input_select.air_purifier_mode
    data_template:
      option: >
        {% set modes = { 0: 'Auto', 2: 'Favorite' } %} {{
        modes[state_attr('fan.YOUR_XIAOMI_FAN_ENTITY',
        'air_purifier.mode')] }}
mode: single
```
---
The `fan` section is where the magic happens. Here, a new fan entity is created based on the fan template from Home Assistant. This entity can be directly controlled by the Mushroom Fan Card.
It has been migrated to comply with version 2026.x.x and the therein introduced breaking changes from fan: - platform: template to template: - fan: and the sensor has been directly included as well.
```yaml
# Actual entity for smart air purifier
template:
  - fan:
    - unique_id: bedroom_fan
      turn_on:
      - target:
          entity_id:
          - fan.zhimi_cpa4_e409_air_purifier
        action: fan.turn_on
      turn_off:
      - target:
          entity_id:
          - fan.zhimi_cpa4_e409_air_purifier
        action: fan.turn_off
      set_percentage:
      - target:
          entity_id:
          - number.zhimi_cpa4_e409_favorite_level
        data:
          value: '{{ (percentage * 14) / 100 | round(0, ''floor'') }}'
        action: number.set_value
      set_oscillating:
      - target:
          entity_id:
          - input_select.air_purifier_mode
        action: input_select.select_next
      - target:
          entity_id:
          - fan.zhimi_cpa4_e409_air_purifier
        data_template:
          preset_mode: '{{ states(''input_select.air_purifier_mode'') }}'
        action: fan.set_preset_mode
      speed_count: 15
      preset_modes:
      - auto
      - favorite
      default_entity_id: fan.bedroom_fan
      name: Bedroom fan
      oscillating: '{{ states(''input_select.air_purifier_mode'') }}'
      percentage: "{% if is_state('input_select.air_purifier_mode', 'auto') %}\n  {{((state_attr('fan.zhimi_cpa4_e409_air_purifier', 'custom_service.moto_speed_rpm') - 400)\n  / (2050 - 400)) * 100 | round(0, 'floor') }}\n{% else %}\n  {{ (states('number.zhimi_cpa4_e409_favorite_level') | int) * (100/14) }}\n{% endif %}"
      preset_mode: '{{ states(''input_select.air_purifier_mode'') }}'
      state: '{{ is_state(''fan.zhimi_cpa4_e409_air_purifier'', ''on'') }}'
  # Shows powerState, speed, mode
  - sensor:
    - default_entity_id: sensor.bedroom_fan_combined
      name: Bedroom Fan Combined
      state: '{{ states(''fan.zhimi_cpa4_e409_air_purifier'') }}, {{ states.fan.bedroom_fan.attributes.percentage }}%, {{ states.input_select.air_purifier_mode.state }}'
```
### Step 2: Creating Frontend Card
Create a new card in your Lovelace UI using the provided YAML code. This card enables control over the Xiaomi Smart Air Purifier through the mushroom fan card.
```yaml
type: custom:mushroom-fan-card
entity: fan.YOUR_DESIRED_FAN_NAME
icon_animation: true
show_oscillate_control: true
show_percentage_control: true
collapsible_controls: true
```
## Card Functionality
Here's a brief explanation of the functionalities of the mushroom fan card:

- The oscillation button now toggles between the "auto" and "favorite" modes instead of controlling oscillation.
- The percentage control allows you to adjust the fan's speed, given that manual mode / favorite mode is selected.

Please remember that this is still a work in progress. Feedback and suggestions are always welcome!

## Disclaimer
The sensor creation is an optional step but is a nice addition. Please note that the "Sleep" mode is excluded from the `input_select` options, as it is controlled via automation on the original fan entity and not the custom one we create. The author of this guide is not responsible for any loss of functionality or data due to following these instructions.

License: MIT

