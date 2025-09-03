# ESPHome E-ink 4 colors dashboard

本專案利用 7.5 吋四色電子紙搭配 ESPHome 與 Home Assistant，在指定的時段內固定會自動更新，以顯示天氣預報與行事曆。

<img src="readme_img/far_view.jpg" width="50%" />

四色資訊板顯示內容包括:
- 今天日期
- 當下天氣預報
- 未來四小時天氣預報
- 當月月曆
- 未來一週內最近四筆行事曆(會顯示時間日期與標題)

<img src="readme_img/close_view.jpg" width="50%" />

以下將說明硬體架構、ESPHome yaml code與Home assistant yaml code

## Hardware 硬體架構

<img src="readme_img/circuit.jpg" width="50%" />

- [7.5吋四色eink電子紙](item.taobao.com/item.htm?id=809151983431) - GDEM075F52 （24Pin）
- [24pin轉2.54pin](item.taobao.com/item.htm?id=809151983431) - C02 轉接板
- [ESP32-S3核心板 開發板](https://item.taobao.com/item.htm?id=724415068331&skuId=5030810340877) - ESP32-S3N16R8，銲接排針（向下），無資料線，CH343P
- [杜邦線](https://detail.tmall.com/item.htm?&id=14466195609&skuId=5922240227983) - 杜邦線21CM 母對母 2.54mm（1排40P） **可選短一點的10CM**
- [IKEA RÖDALM 相框 13x18公分](https://www.ikea.com.tw/zh/products/wall-decoration/frames/rodalm-art-50550033) - 外框有木黑白三色，**注意裱框紙開孔較小，需要挖大**

### 硬體安裝:

1. 取一排8條杜邦線，不需要撕成一條一條，撕成一整排比較好接
2. 用杜邦線根據下表接法連接ESP32S3開發板與C02轉接板
   
| ESP32S3 | C02轉接板 |
|:-------:|:-----:|
| GPIO14 | BUSY |
| GPIO13 | RES |
| GPIO12 | D/C |
| GPIO11 | CS |
| GPIO10 | SCK |
| GPIO9 | SDI |
| GND | GND |
| 3V3 | 3.3V |

3. C02轉接板接上電子紙的FPC排線
4. 供電與燒錄要從開發板上的`USB`才對

## 軟體安裝

1. 將`/fonts`資料夾內的檔案及`eink-4c-dashboard.yaml`放到HA/config/esphome的資料夾內
2. 將`eink_dashboard_sensor.yaml`放到HA/config/packages內 >> [詳細說明](#ha-template-sensor-說明)
3. 將`/images`內的`bg_4c.png`放到HA/config/esphome/images的資料夾內
4. 將`eink-4c-dashboard.yaml`及`eink_dashboard_sensor.yaml`的內容修改成自己HA裡的實體ID，**解說在下方**
5. HA檢查YAML code有無錯誤
    1. 開發工具>YAML>檢查設定內容，確認左下角通知沒有出現錯誤
    2. YAML 設定新載入中>模板實體
    3. 開發工具>狀態>檢查`sensor.eink_sensors`及`sensor.upcoming_calendar_events`有確實出現，以及內容是自己想要的
6. 在ESPhome將`eink_dashboard_sensor.yaml`燒錄至ESP32S3模組
7. 等待可在HA內看到模組上線後，手動按"Screen Refresh"確認螢幕可正常顯示內容

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

因為預設螢幕不會自動更新`update_interval: never`，是由HA內建自動化根據想要的間隔時間來按下更新面板按鈕

自動化流程如下:

#### 1. 每隔多久時間觸發一次自動化:  

```YAML
  trigger:
    - platform: time_pattern     
      # /1 表示每1個小時，要每2小時就寫 /2
      hours: "/1" 
      # 1 表示在該小時的15分時執行，如06:15、07:15、08:15
      minutes: 15
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
     - action: button.press
       data:
         entity_id: button.eink_4c_dashboard_screen_refresh
```

自動化設定完記得至開發工具>YAML>檢查設定

確定都對後在`YAML 設定新載入中`按下`自動化`重新載入自動化才會生效

## HA template sensor 說明 

**要先確認在已經將以下程式碼寫在`configuration.yaml`內，這樣`eink_dashboard_sensor.yaml`檔案放進去才會生效**

![](https://user-images.githubusercontent.com/56766371/184566430-d2dff49b-38cd-4ddd-a775-eaadf7099fc1.png)

此template sensor是將想要的資料格式化後再丟給資訊面板顯示，內容包含兩部分一個是天氣預報，一個是取得最近7日的行事曆標題

### 天氣預報:

由於預設是顯示取得每小時的預報，**請先確認目前用的天氣整合有支援小時預報 (內建的met.no有)**

以下YAML表示每小時的1分將會呼叫取得"每小時"的天氣預報服務，同時更新內容在sensor.eink_sensors裡面

要注意更新面板的時機要在更新天氣預報之後，不然都會看到前一個小時的預報

會使用天氣預報回傳結果的第1組當作這小時的預報，並顯示第2~5組做未來每小時的預報

`attributes`是將要使用的資訊從天氣預報拆分成出來，分別是:
- 這小時的氣溫:  `today_temperature`
- 這小時的濕度:  `today_humidity`
- 這小時的降雨機率:  `today_precipitation`
- 未來四小時的時間:  `forecast_weekday_1`, `forecast_weekday_2`, `forecast_weekday_3`, `forecast_weekday_4`
- 未來四小時的天氣圖示:  `forecast_condition_1`, `forecast_condition_2`, `forecast_condition_3`, `forecast_condition_4`
- 未來四小時的氣溫:  `forecast_temperature_1`, `forecast_temperature_2`, `forecast_temperature_3`, `forecast_temperature_4`

### 取得行事曆:

以下YAML表示每小時的2分將會呼叫取得最近7日的行事曆，同時更新內容在sensor.upcoming_calendar_events裡面

要注意更新面板的時機要在更新之後，不然都會看到前一個小時的內容

`attributes`是將要使用的資訊從天氣預報拆分成出來，分別是:
- 最近四筆行事曆的日期:  `events_date_1`, `events_date_2`, `events_date_3`, `events_date_4`
- 最近四筆行事曆的時間:  `forecast_condition_1`, `forecast_condition_2`, `forecast_condition_3`, `forecast_condition_4`
- 最近四筆行事曆的內容:  `forecast_temperature_1`, `forecast_temperature_2`, `forecast_temperature_3`, `forecast_temperature_4`

