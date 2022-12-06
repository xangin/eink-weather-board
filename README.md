# ESPHome E-ink Weather board
感謝[Madelena](https://github.com/Madelena/esphome-weatherman-dashboard/)提供本專案硬體架構與程式碼的發想

本專案為直式的E-ink天氣預報資訊板放在玄關處，在指定的時段內每格固定，會自動更新顯示內容
另有橫式除了天氣資訊外，還能將個房間的溫濕度等資訊顯示在E-ink板上 [E-ink Dashboard](https://github.com/xangin/esphome-eink-dashboard)

<img src="https://user-images.githubusercontent.com/56766371/184495447-3c9c7ea9-0f64-442d-b695-d47738ba2d2d.jpg" width="66%" alt="Context"/>

直式天氣資訊板顯示內容包括:
- 今天日期
- 當下天氣預報
- 未來四小時天氣預報

<img src="https://user-images.githubusercontent.com/56766371/184495454-a9595013-f24a-4c55-b7f8-b29cdee5af32.jpg" width="66%" alt="Context"/>

以下將說明硬體架構、ESPHome yaml code與Home assistant yaml code

## Hardware 硬體架構

- [微雪 7.5吋黑白墨水屏裸屏](https://detail.tmall.com/item.htm?id=606005913066) - 不帶外殼
- [微雪 墨水屏裸屏 SPI驅動板 ESP32](https://detail.tmall.com/item.htm?id=605757128869) - wifi+藍牙版本(ESP32)
- [IKEA RIBBA 相框 13x18公分](https://www.ikea.com.tw/zh/products/wall-decoration/frames/ribba-art-40378415) - 外框有黑白兩色，**注意裱框紙開孔較小，需要挖大**

## Installation 安裝方式

1. 將`/fonts`資料夾及`e-ink-weather-board.yaml`放到HA/config/esphome的資料夾內
2. 將`/package`內的`eink_dashboard_sensor.yaml`放到HA/config/package內
3. 將`e-ink-weather-board.yaml`及`eink_dashboard_sensor.yaml`的內容修改成自己HA裡的實體ID，**解說在下方**
4. HA檢查YAML code有無錯誤
    1. 開發工具>YAML>檢查設定內容，確認左下角通知沒有出現錯誤
    2. YAML 設定新載入中>模板實體
    3. 開發工具>狀態>檢查`sensor.eink_sensors`有確實出現，以及內容是自己想要的
5. 在ESPhome將`e-ink-weather-board.yaml`燒錄至ESP32模組
6. 完成!

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


### 根據來自HA的binary sensor來決定現在是否要更新面板，晚上睡覺無人時就不用更新以延長面板壽命

```YAML
binary_sensor:
  - platform: homeassistant
    id: 'eink_refresh'
    entity_id: 'binary_sensor.eink_refresh_time'
```


### 面板更新時機

因為預設螢幕不會自動更新`update_interval: never`，是由HA內建自動化根據想要的間隔時間來按下更新面板按鈕
自動化流程如下:

#### 1. 等這個sensor有接收到資料後就執行腳本:  

```YAML
  - platform: homeassistant
    entity_id: sensor.eink_sensors
    attribute: forecast_temperature_4
    id: forecast_temperature_4
    on_value: 
      then:
        - script.execute: all_data_received 
```


## HA template sensor 說明

**要先確認在已經將以下程式碼寫在`configuration.yaml`內，這樣`eink_dashboard_sensor.yaml`檔案放進去才會生效**

![](https://user-images.githubusercontent.com/56766371/184566430-d2dff49b-38cd-4ddd-a775-eaadf7099fc1.png)


由於天氣預報是`weather`類型，非`sensor`類型，沒辦法直接丟給ESPHome處理，所以利用此template sensor

將想要的資料格式化後再丟給ESPHome顯示，`weather.myhome`是我用的天氣預報實體名稱，請記得更換成自己的ID

`attributes`是將要使用的資訊從天氣預報拆分成出來，分別是:
- 這小時的氣溫:  `today_temperature`
- 這小時的濕度:  `today_humidity`
- 這小時的降雨機率:  `today_precipitation`
- 未來四小時的時間:  `forecast_weekday_1`, `forecast_weekday_2`, `forecast_weekday_3`, `forecast_weekday_4`
- 未來四小時的天氣圖示:  `forecast_condition_1`, `forecast_condition_2`, `forecast_condition_3`, `forecast_condition_4`
- 未來四小時的氣溫:  `forecast_temperature_1`, `forecast_temperature_2`, `forecast_temperature_3`, `forecast_temperature_4`


