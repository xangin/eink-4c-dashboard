# ESPHome E-ink 4-Color Dashboard

English | [正體中文](README_zh-tw.md)

A big thanks to [tsunglung](https://github.com/tsunglung/esphome_epaper) for developing the 4-color e-ink display ESPHome component used in this project.

This project utilizes a 7.5-inch 4-color e-ink display with ESPHome and Home Assistant to automatically update at fixed intervals within a specified time period, displaying weather forecasts and calendar events.

<img src="readme_img/far_view.jpg" width="50%" />

The 4-color information dashboard displays the following content:

  - Today's date
  - Current weather forecast
  - Weather forecast for the next four hours
  - Current month's calendar
  - The next four upcoming calendar events within the next week (showing time, date, and title)

<img src="readme_img/close_view.jpg" width="50%" />

The following sections will explain the hardware architecture, ESPHome YAML code, and Home Assistant YAML code.

## Hardware Architecture

<img src="readme_img/circuit.jpg" width="50%" />

  - [7.5-inch 4-color e-ink display](https://item.taobao.com/item.htm?id=809151983431) - GDEM075F52 (24Pin)
  - [FPC-24pins to 2.54mm-8pins adapter](https://item.taobao.com/item.htm?id=809151983431) - C02 adapter board
  - [ESP32-S3 Core Development Board](https://item.taobao.com/item.htm?id=724415068331&skuId=5030810340877) - ESP32-S3N16R8, with soldered headers (pointing down), no data cable, CH343P
  - [Dupont line](https://detail.tmall.com/item.htm?&id=14466195609&skuId=5922240227983) - 21CM female-to-female DuPont wires, 2.54mm (1 row of 40P) **Shorter 10CM wires are an option**
  - [IKEA RÖDALM Photo Frame 13x18cm](https://www.ikea.com.tw/zh/products/wall-decoration/frames/rodalm-art-50550033) - Available in wood, black, and white frames. **Note: The mat opening is too small and needs to be enlarged.**

### Hardware Installation:

1.  Take a strip of 8 dupont line. There's no need to separate them individually; keeping them in a single strip makes connections easier.
2.  Use the dupont line to connect the ESP32S3 development board and the C02 adapter board according to the table below.

| ESP32S3 | C02 Adapter Board |
|:-------:|:-----------------:|
| GPIO14  | BUSY              |
| GPIO13  | RES               |
| GPIO12  | D/C               |
| GPIO11  | CS                |
| GPIO10  | SCK               |
| GPIO9   | SDI               |
| GND     | GND               |
| 3V3     | 3.3V              |

3.  After connecting the wires, attach the C02 adapter board to the e-ink display's FPC cable.
4.  Power and flashing are both done through the `USB` port on the development board.

## Software Installation

1.  Place the files from the `/fonts` folder and `eink-4c-dashboard.yaml` into your `homeassistant/esphome` directory.
2.  Place `eink_dashboard_sensor.yaml` into your `homeassistant/packages` directory.
3.  Modify the contents of `eink_dashboard_sensor.yaml` to match the entity IDs in your Home Assistant setup. >> [Detailed Explanation](#ha-template-sensor-explanation)
4.  Place the `bg_4c.png` file from the `/images` folder into your `homeassistant/esphome/images` directory.
5.  In Home Assistant, check your YAML configuration for errors.
    1.  Go to Developer Tools > YAML > Check Configuration. Ensure no errors appear in the bottom-left notification area.
    2.  Under YAML configuration reloading > Template Entities.
    3.  Go to Developer Tools > States. Verify that `sensor.eink_sensors` and `sensor.upcoming_calendar_events` have been created correctly and their content is correct.
6.  In ESPHome, compile and flash `eink-4c-dashboard.yaml` to the ESP32S3 module.
7.  Wait for the module to come online in Home Assistant, then manually press the "Screen Refresh" button to confirm that the display shows the content correctly.

## Panel Update Schedule

Because this e-ink display model does not support partial refresh, frequent updates are unnecessary. The program is set by default not to update automatically: `update_interval: never`.

Instead, an automation is set up directly within ESPHome to update once every hour. The screen will only refresh when the Times of the Day Sensor from Home Assistant is `True`.

**First, create a "Times of the Day Sensor" helper in Home Assistant:**

Go to HA Settings > Devices & Services > Helpers > Create Helper > "Times of the Day Sensor". Set the time period you want the display to update and name it `eink_refresh_time`.

### ESPHome YAML Explanation:

Since the calendar updates at 5 minutes past the hour and the weather forecast updates at 10 minutes past the hour, the ESPHome automation is set to update at 15 minutes past every hour.

```yaml
time:
  - platform: homeassistant
    id: ha_time
    on_time: 
      seconds: 0
      minutes: 15
      hours: "/1"
      then: 
        - if:
            condition:
              - binary_sensor.is_on: eink_refresh_time
            then:     
              - component.update: 'my_display' 
```

## HA Template Sensor Explanation

**First, ensure that you have included the following code in your `configuration.yaml` file so that files in the `packages` directory, like `eink_dashboard_sensor.yaml`, will be loaded.**

<img src="https://github.com/user-attachments/assets/8f80e777-7958-4027-87f4-2ae585448135" />

This template sensor formats the desired data before sending it to the information panel. It consists of two parts: one for the weather forecast and one for retrieving the titles of the next four calendar events within the next 7 days.

### Weather Forecast:

The default setting is to display the hourly forecast. **Please ensure that your current weather integration supports hourly forecasts (the built-in met.no integration does).**

The YAML below calls the service to get the "hourly" weather forecast at 10 minutes past every hour and updates the content in `sensor.eink_sensors`.

```yaml
  - trigger:
      # Updates at 10 minutes past every hour
      - platform: time_pattern      
        hours: "/1" 
        minutes: 10 
    action:
      - action: weather.get_forecasts
        target:
          entity_id: weather.myhome #REPLACE with your weather entity id   
        data:
          type: hourly
        response_variable: hourly      
      - variables:
          hourly_forecasts: "{{ hourly['weather.YOUR_WEATHER_ID'].forecast }}"
```

Note that the panel update time must be after the weather forecast update time; otherwise, you will see the forecast from the previous hour.

The 1st set of the weather forecast results will be used as the current hour's forecast, and the 2nd to 5th sets will be displayed as the forecast for the upcoming hours.

The `attributes` are used to split the necessary information from the weather forecast, which are:

  - Current hour's temperature: `today_temperature`
  - Current hour's precipitation probability: `today_precipitation`
  - Time for the next four hours: `forecast_weekday_1`, `forecast_weekday_2`, `forecast_weekday_3`, `forecast_weekday_4`
  - Weather icons for the next four hours: `forecast_condition_1`, `forecast_condition_2`, `forecast_condition_3`, `forecast_condition_4`
  - Temperature for the next four hours: `forecast_temperature_1`, `forecast_temperature_2`, `forecast_temperature_3`, `forecast_temperature_4`

### Fetching Calendar Events:

The YAML below calls the service to get calendar events for the next 7 days at 5 minutes past every hour and updates the content in `sensor.upcoming_calendar_events`.

```yaml
  - trigger:
      - platform: time_pattern
        hours: "/1"  # Updates at 5 minutes past every hour
        minutes: 5
    action:
      - action: calendar.get_events
        target:
          entity_id: calendar.sfcasa
        data:
          end_date_time: "{{ (now() + timedelta(days=7)).isoformat() }}"  # Get events for the next 7 days
        response_variable: agenda
      - variables:
          my_events: >
            {{ agenda["calendar.YOUR_CANLENDAR_ID"].events }} 
```

Note that the panel update time must be after this update; otherwise, you will see the content from the previous hour.

The `attributes` are used to split the necessary information from the calendar. If there are no events, it will display a blank space. The attributes are:

  - Date of the next four events: `events_date_1`, `events_date_2`, `events_date_3`, `events_date_4`
  - Time of the next four events: `events_time_1`, `events_time_2`, `events_time_3`, `events_time_4`
  - Title of the next four events: `events_title_1`, `events_title_2`, `events_title_3`, `events_title_4`
