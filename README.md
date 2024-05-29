# ESPHome E-ink Weather board

<a href="https://www.buymeacoffee.com/xangin" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="30" width="130"></a>

感謝[Madelena](https://github.com/Madelena/esphome-weatherman-dashboard/)提供本專案硬體架構與程式碼的發想

Thanks [Madelena](https://github.com/Madelena/esphome-weatherman-dashboard/) for inspired this project ideas and source code of weather data

本專案為直式的E-ink天氣預報資訊板放在玄關處，在指定的時段內每格固定，會自動更新顯示內容

另有橫式除了天氣資訊外，還能將個房間的溫濕度等資訊顯示在E-ink上: [E-ink Dashboard](https://github.com/xangin/esphome-eink-dashboard)

<img src="https://i.imgur.com/3Hk3BAW.jpg" width="66%" />

直式天氣資訊板顯示內容包括:
- 今天日期
- 當下天氣預報
- 未來四小時天氣預報

<img src="https://i.imgur.com/gmaCiZK.jpg" width="66%" />

以下將說明硬體架構、ESPHome yaml code與Home assistant yaml code

## Hardware 硬體架構

- [微雪 7.5吋黑白墨水屏裸屏](https://detail.tmall.com/item.htm?id=606005913066) - 不帶外殼
- [微雪 墨水屏裸屏 SPI驅動板 ESP32](https://detail.tmall.com/item.htm?id=605757128869) - wifi+藍牙版本(ESP32)
- [IKEA RIBBA 相框 13x18公分](https://www.ikea.com.tw/zh/products/wall-decoration/frames/ribba-art-40378415) - 外框有黑白兩色，**注意裱框紙開孔較小，需要挖大**

## Installation 安裝方式

1. 將`/fonts`資料夾內的檔案及`e-ink-weather-board.yaml`放到HA/config/esphome的資料夾內
2. 將`eink_dashboard_sensor.yaml`放到HA/config/packages內
3. 將`/images`內的`background.png`放到HA/config/esphome/images的資料夾內
4. 將`e-ink-weather-board.yaml`及`eink_dashboard_sensor.yaml`的內容修改成自己HA裡的實體ID，**解說在下方**
5. HA檢查YAML code有無錯誤
    1. 開發工具>YAML>檢查設定內容，確認左下角通知沒有出現錯誤
    2. YAML 設定新載入中>模板實體
    3. 開發工具>狀態>檢查`sensor.eink_sensors`有確實出現，以及內容是自己想要的
6. 在ESPhome將`e-ink-weather-board.yaml`燒錄至ESP32模組
7. 完成!

## ESPHome yaml 說明

### 在HA內手動更新面板

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


### 將HA的時間帶進ESPHome

```YAML
time:
  - platform: homeassistant
    id: ha_time
```


### 面板更新時機

請注意，此面板長時間通電會導致越刷越模糊，黑底會變成不是純黑，建議只有刷新時才將ESP32通電，或是刷新完進入Deep sleep以避免此情形發生

因為預設螢幕不會自動更新`update_interval: never`，是由HA內建自動化根據想要的間隔時間來按下更新面板按鈕

自動化流程如下:

#### 1. 每隔多久時間觸發一次自動化:  

```YAML
  trigger:
    - platform: time_pattern     
      # /2 表示每2個小時，要每小時就寫 /1
      hours: "/2" 
      # 1 表示在該小時的1分時執行，如06:01、08:01、10:01
      minutes: 1
```

#### 2. 條件: 

在HA設定>裝置與服務>助手>新增助手>"每日定時感測器">設定想要更新的時段，名稱填`eink_refresh_time`

```YAML
  condition:
    #在可更新的時段內才更新
    - condition: state
      entity_id: binary_sensor.eink_refresh_time
      state: 'on'
```

#### 3. 動作:  

```YAML
   action:
   #按下更新螢幕的按鈕，記得更換為自己的實體ID
     - service: button.press
       data:
         entity_id: button.eink_weather_board_screen_refresh
```

自動化設定完記得至開發工具>YAML>檢查設定

確定都對後在`YAML 設定新載入中`按下`自動化`重新載入自動化才會生效

## HA template sensor 說明 

**要先確認在已經將以下程式碼寫在`configuration.yaml`內，這樣`eink_dashboard_sensor_new.yaml`檔案放進去才會生效**

![](https://user-images.githubusercontent.com/56766371/184566430-d2dff49b-38cd-4ddd-a775-eaadf7099fc1.png)

在HA 2023.12之後，天氣預報改由呼叫service來取得未來的資訊，所以利用此template sensor將想要的資料格式化後再丟給天氣板顯示

由於預設是顯示取得每小時的預報，**請先確認目前用的天氣整合有支援小時預報 (內建的met.no有)**

以下YAML表示每小時的1分將會呼叫取得"每小時"的天氣預報服務，同時更新內容在sensor.eink_sensors裡面

要注意更新面板的時機要在更新天氣預報之後，不然都會看到前一個小時的預報

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

會使用天氣預報回傳結果的第1組當作這小時的預報，並顯示第2~5組做未來每小時的預報

`attributes`是將要使用的資訊從天氣預報拆分成出來，分別是:
- 這小時的氣溫:  `today_temperature`
- 這小時的濕度:  `today_humidity`
- 這小時的降雨機率:  `today_precipitation`
- 未來四小時的時間:  `forecast_weekday_1`, `forecast_weekday_2`, `forecast_weekday_3`, `forecast_weekday_4`
- 未來四小時的天氣圖示:  `forecast_condition_1`, `forecast_condition_2`, `forecast_condition_3`, `forecast_condition_4`
- 未來四小時的氣溫:  `forecast_temperature_1`, `forecast_temperature_2`, `forecast_temperature_3`, `forecast_temperature_4`


## References
- https://github.com/Madelena/esphome-weatherman-dashboard/
