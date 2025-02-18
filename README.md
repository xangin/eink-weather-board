# ESPHome E-ink Weather board

<a href="https://www.buymeacoffee.com/xangin" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="30" width="130"></a>

Thanks to [Madelena](https://github.com/Madelena/esphome-weatherman-dashboard/) for inspiring this project and providing the source code for weather data.

This project is a vertical E-ink weather forecast board placed at the entrance, automatically updating and displaying content at specified intervals.

There is also a horizontal version that can display temperature and humidity information for individual rooms on the E-ink: [E-ink Dashboard](https://github.com/xangin/esphome-eink-dashboard)

<img src="https://i.imgur.com/3Hk3BAW.jpg" width="66%" />

The vertical weather board displays:
- Today's date
- Current weather forecast
- Weather forecast for the next four hours

<img src="https://i.imgur.com/gmaCiZK.jpg" width="66%" />

Below are the hardware structure, ESPHome yaml code, and Home assistant yaml code.

## Hardware

- [Waveshare 7.5-inch black and white E-ink screen](https://detail.tmall.com/item.htm?id=606005913066) - bare screen without casing
- [Waveshare E-ink screen SPI driver board ESP32](https://detail.tmall.com/item.htm?id=605757128869) - WiFi + Bluetooth version (ESP32)
- [IKEA RÖDALM Frame 13x18 cm](https://www.ikea.com.tw/zh/products/wall-decoration/frames/rodalm-art-50550033) - available in black, white, and wooden color. Note: The mat opening is small and needs to be enlarged

## Installation

1. Place the files in the `/fonts` folder and `e-ink-weather-board.yaml`in the HA/config/esphome folder.
2. Place `eink_dashboard_sensor.yaml` in the HA/config/packages folder.
3. PPlace `background.png` from the `/images` folder in the HA/config/esphome/images folder.
4. Modify `e-ink-weather-board.yaml` and `eink_dashboard_sensor.yaml` to match your HA entity IDs. **Instructions are below**
5. Check for YAML code errors in HA.
   1. Developer Tools > YAML > Check Configuration, ensure no errors in the bottom left notification.
   2. YAML Configuration Reloading > Template Entities
   3. Developer Tools > States > Check `sensor.eink_sensors` to ensure it appears and contains the desired content.
6. Flash e-ink-weather-board.yaml to the ESP32 module in ESPHome.
7. Done!

## ESPHome yaml Explanation

### Manually update the panel in HA

```YAML
button:
  - platform: template
    name: '${devicename} Refresh'
    icon: 'mdi:update'
    on_press:
      then:
        - component.update: 'my_display'
    internal: false
```


### Pass HA time to ESPHome

```YAML
time:
  - platform: homeassistant
    id: ha_time
```


### Panel update timing

Note: Long-term power to this panel may cause increasing blurriness. It is recommended to power the ESP32 only when refreshing or enter Deep Sleep to avoid this issue.

The default screen does not automatically update `update_interval: never`. It is updated by HA automation based on the desired interval.

Automation process:

#### 1. Trigger automation at intervals:

```YAML
  trigger:
    - platform: time_pattern     
      # /2 means every 2 hours, use /1 for every hour
      hours: "/2" 
      # 1 means it runs at the 1st minute of the hour, like 06:01, 08:01, 10:01
      minutes: 1
```

#### 2. Condition: 

Set the update period in HA > Settings > Devices & Services > Helpers > Add Helper > "Daily Timer" > Set the desired update period, name it `eink_refresh_time`.

```YAML
  condition:
    #Update only during allowed periods
    - condition: state
      entity_id: binary_sensor.eink_refresh_time
      state: 'on'
```

#### 3. Action:  

```YAML
   action:
   #Press the update button, replace with your entity ID
     - service: button.press
       data:
         entity_id: button.eink_weather_board_screen_refresh
```

自動化設定完記得至開發工具>YAML>檢查設定

Remember to check the configuration in Developer Tools > YAML > Check Configuration.

Once everything is correct, reload the automation in YAML Configuration Reloading > Automation.

## HA template sensor Explanation

**Ensure the following code is written in`configuration.yaml` for  `eink_dashboard_sensor_new.yaml`to take effect**

![](https://user-images.githubusercontent.com/56766371/184566430-d2dff49b-38cd-4ddd-a775-eaadf7099fc1.png)

Ensure the weather integration supports hourly forecasts (built-in met.no does).

The following YAML calls the "hourly" weather forecast service at 1 minute past every hour and updates sensor.eink_sensors with the content.

Update the panel after updating the weather forecast to avoid seeing the previous hour's forecast.

```YAML

  - trigger:
      - platform: time_pattern     
        hours: "/1" 
        minutes: 1
    action:
      - service: weather.get_forecasts
        target:
          entity_id: weather.myhome #replace with your weather forecast entity id
        data:
          type: hourly
        response_variable: hourly  

```

Use the first set of the weather forecast as this hour's forecast and display the 2nd to 5th sets for the next four hours.

`attributes`separates the information from the weather forecast as:
- Temperature for this hour: `today_temperature`
- Humidity for this hour: `today_humidity`
- Precipitation probability for this hour: `today_precipitation`
- Time for the next four hours:  `forecast_weekday_1`, `forecast_weekday_2`, `forecast_weekday_3`, `forecast_weekday_4`
- Weather icons for the next four hours:  `forecast_condition_1`, `forecast_condition_2`, `forecast_condition_3`, `forecast_condition_4`
- Temperature for the next four hours:   `forecast_temperature_1`, `forecast_temperature_2`, `forecast_temperature_3`, `forecast_temperature_4`

## References
- https://github.com/Madelena/esphome-weatherman-dashboard/
