# [EMHASS](https://emhass.readthedocs.io/) conf for Deye inverters using Solar Assistant

## Prerequisites
1. Set up EMHASS according to Docs.
2. Read EMHASS docs to familiarize yourself with the concepts
3. [Nordpool](https://github.com/custom-components/nordpool) integration installed
4. [Solcast](https://github.com/oziee/ha-solcast-solar) integration installed

## Nordpool setup
1. Create Nordpool sensor without additional costs. Rename it to sensor.nordpool_sell and use EUR/kWh
2. Create Nordpool sensor with additional costs. Rename it to sensor.nordpool_buy_eur_kwh
   Use ```{% set s = { \"day\": 0.0617, \"night\": 0.0354, \"additions\": 0.0123} %} {% if now().weekday() in (5,6) %} {{(s.night+s.additions)|float}} {% else %} {% if now().hour >= 7 and now().hour < 22%} {{(s.day+s.additions)|float}} {% else %} {{(s.night+s.additions)|float}} {% endif %} {% endif %}``` as additional costs for EE Võrk 4 or adjust according to your electricity package

## Create Solcast 24h sensor
Add to HA configuration.yaml
```
sensor:
  - platform: template
    sensors:
      solcast_24hrs_forecast:
            value_template: >-
              {%- set power = ((state_attr('sensor.solcast_pv_forecast_forecast_today', 'detailedHourly') | map(attribute='pv_estimate') | list + state_attr('sensor.solcast_pv_forecast_forecast_tomorrow', 'detailedHourly') | map(attribute='pv_estimate') | list))[now().hour:][:24] %}
              {%- set values_all = namespace(all=[]) %}
              {% for i in range(power | length) %}
              {%- set v = (power[i] | float |multiply(1000) ) | int(0) %}
                {%- set values_all.all = values_all.all + [ v ] %}
              {%- endfor %} {{ (values_all.all)[:48] }}
```

## Create shell commands
Add to HA configuration.yaml
```
shell_command:
  publish_data: "curl -i -H \"Content-Type:application/json\" -X POST -d '{}' http://localhost:5001/action/publish-data"
  mpc: "curl -i -H \"Content-Type: application/json\" -X POST -d '{\"load_cost_forecast\":{{((state_attr('sensor.nordpool_buy_eur_kwh', 'raw_today') | map(attribute='value') | list  + state_attr('sensor.nordpool_buy_eur_kwh', 'raw_tomorrow') | map(attribute='value') | list))[now().hour:][:24] }},\"prod_price_forecast\":{{((state_attr('sensor.nordpool_sell', 'raw_today') | map(attribute='value') | list  + state_attr('sensor.nordpool_sell', 'raw_tomorrow') | map(attribute='value') | list))[now().hour:][:24]}}, \"prediction_horizon\": {{((state_attr('sensor.nordpool_buy_eur_kwh', 'raw_today') | map(attribute='value') | list  + state_attr('sensor.nordpool_buy_eur_kwh', 'raw_tomorrow') | map(attribute='value') | list))[now().hour:][:24] | count}}, \"soc_init\":{{(states('sensor.battery_state_of_charge')|float(0))/100}}, \"soc_final\": 0.2, \"def_total_hours\":[5,5], \"pv_power_forecast\":{{ states('sensor.solcast_24hrs_forecast') }}}' http://localhost:5001/action/naive-mpc-optim"
```
 - This creates 2 deferrable loads, both should run 5h. This part is not fully optimized yet as it would be best to track how many hours it has already run in the last 24h
 - MPC is used because it's the only optimization that can take initial battery SOC in consideration

## Create helper for turning EMHASS automations on or off
Name it input_boolean.emhass_automation

## Create automations
Take these as inspiration and adjust to your own sensor and switch names
```
- id: '1697992786350'
  alias: 'EMHASS: optimization'
  description: ''
  trigger:
  - platform: time_pattern
    hours: /1
  condition: []
  action:
  - service: shell_command.mpc
    data: {}
  - delay:
      hours: 0
      minutes: 0
      seconds: 45
      milliseconds: 0
  - service: shell_command.publish_data
    data: {}
  mode: single
- id: '1697994471293'
  alias: 'EMHASS: start charging'
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition:
  - condition: numeric_state
    entity_id: sensor.p_batt_forecast
    below: -300
  - condition: numeric_state
    entity_id: sensor.pv_energy
    below: 100
  - condition: state
    entity_id: input_boolean.emhass_automation
    state: 'on'
  action:
  - service: number.set_value
    data:
      value: '100'
    target:
      entity_id: number.capacity_point_5
  - service: switch.turn_on
    data: {}
    target:
      entity_id: switch.grid_charge_point_5
  - service: number.set_value
    data:
      value:
        '[object Object]':
    target:
      entity_id: number.max_charge_current
  mode: single
- id: '1697994777457'
  alias: 'EMHASS: stop charging'
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition:
  - condition: numeric_state
    entity_id: sensor.p_batt_forecast
    above: -300
  - condition: state
    entity_id: input_boolean.emhass_automation
    state: 'on'
  action:
  - service: switch.turn_off
    data: {}
    target:
      entity_id: switch.grid_charge_point_5
  mode: single
- id: '1697994880327'
  alias: 'EMHASS: start discharging'
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition:
  - condition: numeric_state
    entity_id: sensor.p_batt_forecast
    above: 100
  - condition: state
    entity_id: input_boolean.emhass_automation
    state: 'on'
  action:
  - service: number.set_value
    data:
      value: '20'
    target:
      entity_id: number.capacity_point_5
  mode: single
- id: '1697995031734'
  alias: 'EMHASS: stop discharging'
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition:
  - condition: numeric_state
    entity_id: sensor.p_batt_forecast
    below: 100
  - condition: numeric_state
    entity_id: sensor.battery_state_of_charge
    below: sensor.soc_batt_forecast
  - condition: state
    entity_id: input_boolean.emhass_automation
    state: 'on'
  action:
  - service: number.set_value
    data:
      value: '100'
    target:
      entity_id: number.capacity_point_5
  mode: single
- id: '1697999382917'
  alias: 'EMHASS: heatpump on'
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition:
  - condition: numeric_state
    entity_id: sensor.p_deferrable0
    above: 100
  action:
  - service: number.set_value
    data:
      value: '0'
    target:
      entity_id: number.heat_offset_s1_47011_2
  - service: number.set_value
    data:
      value: '-1'
    target:
      entity_id: number.heat_offset_s1_47011
  mode: single
- id: '1697999489627'
  alias: 'EMHASS: heatpump off'
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition:
  - condition: numeric_state
    entity_id: sensor.p_deferrable0
    below: 100
  action:
  - service: number.set_value
    data:
      value: '-3'
    target:
      entity_id: number.heat_offset_s1_47011_2
  - service: number.set_value
    data:
      value: '-4'
    target:
      entity_id: number.heat_offset_s1_47011
  mode: single
- id: '1698034680127'
  alias: 'EMHASS: publish results '
  description: ''
  trigger:
  - platform: time_pattern
    minutes: /5
  condition: []
  action:
  - service: shell_command.publish_data
    data: {}
  mode: single
```
