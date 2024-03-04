# Cumulative Acid Dosing Over Time
I wanted to know when the acid drum for the peristaltic pump is nearing empty.
My pool acid is automatically fed by the Halo's peristaltic pump from a 5L drum next to my pump-house, but there are no warnings or notifications about the drum being empty or nearing empty.

The Halo tracks acid dosing via the `sensor.hchlor_dosing_pump_today_ml` sensor, which measures the amount of acid dosed in a calendar day. However, this sensor resets at midnight so it does not track cumulative acid dosage over time.

## Requirements:
- Track and show cumulative volume of acid dosed over any period of time.
- Show how much acid is still available in the drum.
- Allow me to define the total volume of the acid drum. Default to 5L but allow any volume.
- Create an alert (push/mobile/email) when the acid drum remaining volume drops below a certain level. e.g. 1L remaining.
- Allow me to reset the cumulative volume counter when I refill the acid feed drum.

### Bonus features:
- Show the average daily acid usage over the last 1, 3, 7 and 30 days.

## Method
### Sensors and Entities:
1. Create `input_number` entity to store cumulative acid dosage each day from the HCHLOR default sensor.
2. Create `input_number` entity to store total drum volume.
3. Create a Statistics sensor to calculate average daily acid usage by the HCHLOR default sensor.
4. Create a Template sensor to show how much acid is still available in the drum.
5. Create a Template sensor to show Average Daily Usages: 3, 7, 30 days.
6. Create a virtual button that will reset the cumulative volume counter when you refill the acid feed drum.

### Automations:
1. Update cumulative acid dosage each day.
2. Reset the cumulative volume counter when the virtual button is pressed.
3. Alert when the acid drum remaining volume drops below 1L.

### Dashboard:
1. Create a Lovelace Card that tracks everything with a button to reset the acid volume when refilled.

![image|351x500](upload://6VS702WhwobWejLWUeVjYT2CRkz.png)


## Guide
### configuration.yaml
Add the following lines to `configuration.yaml`

```
# Pool Acid Sensors and Variables
  - platform: template
    sensors:
      # Calculate the sum of acid dosed today plus cumulative acid dosed
      pool_acid_dosed_today_plus_cumulative:
        friendly_name: "Pool Acid Dosed Today + Cumulative"
        unit_of_measurement: 'mL'
        value_template: "{{ states('sensor.hchlor_dosing_pump_today_ml') | float + states('input_number.pool_acid_cumulative_dosed') | float }}"
      # Show how much acid is still available in the drum.
      # This sensor will start at the volume of the drum (as stored in input_number.pool_acid_drum_volume) and decrease by the amount dosed each day.
      pool_acid_remaining:
        friendly_name: "Pool Acid Drum Volume Remaining"
        unit_of_measurement: 'mL'
        value_template: "{{ states('input_number.pool_acid_drum_volume') | float - states('input_number.pool_acid_cumulative_dosed') | float - states('sensor.hchlor_dosing_pump_today_ml') | float }}"
      # Average Daily Usages: 3, 7, 30 days
      pool_acid_average_daily_usage_3_days:
        friendly_name: "Pool Acid Average Daily Usage over Last 3 Days"
        unit_of_measurement: 'mL'
        value_template: "{{ states('sensor.pool_acid_usage_statistics') | float / 3 }}"
      pool_acid_average_daily_usage_7_days:
        friendly_name: "Pool Acid Average Daily Usage over Last 7 Days"
        unit_of_measurement: 'mL'
        value_template: "{{ states('sensor.pool_acid_usage_statistics') | float / 7 }}"
      pool_acid_average_daily_usage_30_days:
        friendly_name: "Pool Acid Average Daily Usage over Last 30 Days"
        unit_of_measurement: 'mL'
        value_template: "{{ states('sensor.pool_acid_usage_statistics') | float / 30 }}"

  # Statistics sensor to calculate average daily acid usage by the HCHLOR default sensor
  - platform: statistics
    name: pool_acid_usage_statistics
    entity_id: sensor.hchlor_dosing_pump_today_ml
    state_characteristic: mean
    max_age:
      days: 1000

# Create input_number entities to store cumulative dosed and drum volume
# Update the max values below if you have a drum size larger than 25L
input_number:
  # Store the cumulative volume of acid dosed 
  pool_acid_cumulative_dosed:
    name: Pool Acid Cumulative Dosed
    initial: 0
    min: 0
    max: 25000
    step: 1
  # Store the total volume of the acid drum
  pool_acid_drum_volume:
    name: Pool Acid Drum Volume
    initial: 5000
    min: 0
    max: 25000
    step: 1

# Create a virtual button that will reset the cumulative volume counter when you refill the acid feed drum
input_boolean:
  reset_pool_acid_cumulative_dosed:
    name: Reset Pool Acid Cumulative Dosed
    initial: off
```

### Automations:

#### 1. Update cumulative acid dosage each day.
```
alias: "Pool: Daily Update Pool Acid Cumulative Dosed"
description: Update cumulative acid dosage each day
trigger:
  - platform: time
    at: "23:50:00"
    enabled: true
action:
  - service: input_number.set_value
    target:
      entity_id: input_number.pool_acid_cumulative_dosed
    data:
      value: >-
        {{ states('input_number.pool_acid_cumulative_dosed') | float +
        states('sensor.hchlor_dosing_pump_today_ml') | float }}
```
#### 2. Reset the cumulative volume counter when the virtual button is pressed.
```
alias: "Pool: Reset Pool Acid Cumulative Dosed"
description: Reset the cumulative volume counter when the virtual button is turned on
trigger:
  - platform: state
    entity_id: input_boolean.reset_pool_acid_cumulative_dosed
    to: "on"
action:
  - service: input_number.set_value
    target:
      entity_id: input_number.pool_acid_cumulative_dosed
    data:
      value: 0
  - service: input_boolean.turn_off
    target:
      entity_id: input_boolean.reset_pool_acid_cumulative_dosed
    data: {}
mode: single
```
#### 3. Alert when the acid drum remaining volume drops below 1L.
```
alias: "Pool: Alert on Low Acid Volume"
description: Alert when the acid drum remaining volume drops below 1L
trigger:
  - platform: numeric_state
    entity_id: sensor.pool_acid_remaining
    below: 1000
condition: []
action:
  - repeat:
      while:
        - condition: numeric_state
          entity_id: sensor.pool_acid_remaining
          below: 1000
      sequence:
        - service: notify.mobile_app_m2102j20sg
          data:
            message: "Alert: Acid drum volume is below 1L. Please refill."
        - delay:
            hours: 12
mode: single
```

### Lovelace Dashboard card:

Dependency on the via the [multiple-entity-row](https://github.com/benct/lovelace-multiple-entity-row) HACS front-end component available via this GitHub:
```
      - type: vertical-stack
        title: Pool Acid
        cards:
          - type: entities
            entities:
              - type: custom:multiple-entity-row
                entity: sensor.hchlor_dosing_pump_today_ml
                show_state: true
              - type: custom:multiple-entity-row
                entity: sensor.pool_acid_dosed_today_plus_cumulative
                show_state: true
              - type: custom:multiple-entity-row
                entity: input_number.pool_acid_cumulative_dosed
                show_state: true
              - type: custom:multiple-entity-row
                entity: sensor.pool_acid_remaining
                show_state: true
              - type: custom:multiple-entity-row
                entity: input_number.pool_acid_drum_volume
                show_state: true
              - entity: input_boolean.reset_pool_acid_cumulative_dosed
                type: button
          - type: history-graph
            entities:
              - entity: sensor.pool_acid_average_daily_usage_3_days
              - entity: sensor.pool_acid_average_daily_usage_7_days
              - entity: sensor.pool_acid_average_daily_usage_30_days
            hours_to_show: 720
            refresh_interval: 0

```
