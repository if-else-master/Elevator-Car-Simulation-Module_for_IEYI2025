# 電梯車廂模擬模組 (Elevator Car Simulation Module)

## 系統概述

這是一個結合影像辨識技術和實體硬體的智能電梯控制系統，專為IEYI2025競賽設計。系統包含Python端的控制程序和Arduino端的硬體模組，能夠透過攝影機檢測電梯內的人員密度，並自動啟動Express模式以提升運行效率。

## 主要功能

- **智能影像辨識**: 使用OpenCV背景減法技術檢測電梯內物體密度
- **自動Express模式**: 當檢測到電梯過載時自動啟動Express模式
- **多樓層控制**: 支援3層樓電梯運行邏輯
- **硬體整合**: Arduino控制TFT螢幕、LED燈條和按鈕
- **實時通訊**: Python與Arduino間的序列通訊
- **音效回饋**: 樓層到達音效提示

## 影像辨識演算法

### 背景減法技術 (Background Subtraction)

系統採用OpenCV的MOG2 (Mixture of Gaussians)背景減法器進行物體檢測：

```python
self.background_subtractor = cv2.createBackgroundSubtractorMOG2(
    history=300,         # 歷史幀數：300幀
    varThreshold=50,     # 變化檢測閾值：50（降低敏感度）
    detectShadows=True   # 啟用陰影檢測
)
```

### 多層次強度檢測

系統實施分層檢測機制來區分真正的物體和陰影干擾：

1. **高強度檢測** (預設220): 檢測實際物體
2. **中等強度檢測** (預設150): 檢測陰影區域
3. **形態學處理**: 使用開運算和閉運算去除噪音

### 學習率動態調整

```python
# 初始學習率：快速建立背景模型
self.initial_learning_rate = 0.01

# 穩定學習率：防止靜態物體被學習為背景
self.stable_learning_rate = 0.0005
```

### 檢測區域優化

系統動態設定檢測區域，排除邊緣干擾：

```python
# 左邊界往右調整10%，右邊界往左調整3%
left_offset = int(width * 0.1)
right_offset = int(width * 0.03)
self.detection_roi = (left_offset, 0, width - left_offset - right_offset, height)
```

## 電梯控制邏輯

### 請求處理系統

電梯支援兩種請求類型：
- **內部請求** (INTERNAL): 電梯內按鈕
- **外部請求** (UP/DOWN): 樓層外呼叫按鈕

### 運行模式

#### 1. 正常模式 (NORMAL)
- 處理所有內部和外部請求
- 使用最短路徑演算法選擇下一站
- 支援中途停靠邏輯

#### 2. 手動Express模式 (EMERGENCY)
- 僅處理內部請求
- 外部請求暫存至`pending_external_requests`
- 人工按鈕控制啟動/解除

#### 3. 自動Express模式 (FULL)
- 當物體檢測率超過閾值時自動啟動
- 使用與手動模式相同的邏輯
- 檢測率降低時自動解除

### 移動動畫與中途停靠

```python
# 移動參數
movement_time = 3.0  # 3秒移動時間
frame_interval = 30  # 30ms更新間隔
```

中途停靠邏輯：
- **上行**: 檢查是否有同向或內部請求的樓層
- **下行**: 檢查是否有同向或內部請求的樓層
- **Express模式特殊規則**: 1樓往上時可在2樓停靠

## 硬體配置

### Arduino端組件

#### TFT螢幕 (ST7789)
- **解析度**: 240×240 像素
- **連接腳位**:
  - CS: Pin 10
  - DC: Pin 9  
  - RST: Pin 8
  - MOSI: Pin 11
  - SCLK: Pin 13
  - BLK: Pin 12

#### WS2812B LED燈條
- **燈條A**: Pin 2 (8顆LED)
- **燈條B**: Pin 3 (8顆LED)
- **亮度**: 40/255 (約15%)
- **顏色**: 溫暖黃光 RGB(255, 180, 80)

#### 按鈕配置
- **緊急按鈕**: Pin 4
- **3樓按鈕**: Pin 5
- **2樓按鈕**: Pin 6  
- **1樓按鈕**: Pin 7
- **防彈跳時間**: 200ms

### 顯示邏輯

#### 樓層顯示格式
- **靜止**: 顯示當前樓層數字
- **上行**: `^目標樓層`
- **下行**: `v目標樓層`

#### 狀態顯示
- **NORMAL**: 白色邊框和文字
- **EMERGENCY**: 紅色邊框和文字 (手動模式)
- **EXPRESS**: 綠色邊框和文字 (自動模式)

## 通訊協定

### Python → Arduino

```
STATUS:狀態;FLOOR:當前樓層;DIR:方向;TARGET:目標樓層;
```

範例:
```
STATUS:NORMAL;FLOOR:2;DIR:UP;TARGET:3;
STATUS:FULL;FLOOR:1;DIR:IDLE;TARGET:1;
```

### Arduino → Python

#### 按鈕事件
```
BUTTON:1          # 1樓按鈕
BUTTON:2          # 2樓按鈕  
BUTTON:3          # 3樓按鈕
BUTTON:EMERGENCY_ON   # 啟動緊急模式
BUTTON:EMERGENCY_OFF  # 解除緊急模式
```

#### 音效播放
```
PLAY_SOUND:1f.mp3     # 播放1樓音效
PLAY_SOUND:em.mp3     # 播放緊急音效
```

## 安裝與設定

### Python環境需求

```bash
pip install opencv-python
pip install pillow
pip install pyserial
pip install numpy
```

### Arduino函式庫需求

```cpp
#include <Adafruit_GFX.h>
#include <Adafruit_ST7789.h>
#include <Adafruit_NeoPixel.h>
```

### 序列埠設定

預設序列埠: `/dev/cu.usbserial-1230`

如需修改，請更新：
- `test.py` 第60行和第262行
- 或讓系統自動檢測Arduino裝置

## 使用方法

### 1. 硬體連接
1. 將Arduino與電腦透過USB連接
2. 確認TFT螢幕、LED燈條和按鈕正確連接
3. 設定攝影機（預設使用攝影機索引0）

### 2. 啟動系統
```bash
cd Code
python test.py
```

### 3. 系統操作
- **重置背景**: 點擊"Reset Background"按鈕
- **調整敏感度**: 使用滑動條調整物體和陰影檢測閾值
- **Express模式閾值**: 調整自動啟動Express模式的物體密度百分比
- **手動控制**: 使用GUI按鈕或Arduino實體按鈕發送請求

### 4. 監控資訊
- 攝影機畫面顯示檢測結果（紅色=物體，黃色=陰影）
- 控制台輸出詳細的系統狀態和通訊記錄
- Arduino TFT螢幕顯示當前樓層和運行狀態

## 技術特色

### 1. 智能檢測優化
- 動態學習率防止背景模型污染
- 多層次閾值區分物體和陰影
- 形態學處理提升檢測準確性

### 2. 電梯邏輯優化
- 中途停靠提升效率
- Express模式特殊規則
- 請求暫存機制

### 3. 使用者體驗
- 實時視覺化檢測結果
- 音效回饋
- 直觀的TFT顯示介面

## 故障排除

### 常見問題

1. **Arduino連接失敗**
   - 檢查USB連接
   - 確認序列埠路徑
   - 查看控制台輸出的可用埠列表

2. **攝影機無畫面**
   - 確認攝影機權限
   - 嘗試更改`cv2.VideoCapture(0)`中的索引值

3. **檢測不準確**
   - 點擊"Reset Background"重新建立背景模型
   - 調整檢測閾值滑動條
   - 確保光線穩定

4. **TFT螢幕無顯示**
   - 檢查SPI連接
   - 確認電源供應
   - 驗證腳位定義

## 開發團隊

此專案為IEYI2025競賽開發，整合了電腦視覺、嵌入式系統和電梯控制邏輯等多項技術。<br>
感謝 masonzeng702550<br>
感謝 Jimmymao330<br>

聯絡方式：rayc57429@gmail.com

## 版本資訊

- **當前版本**: v1.0
- **最後更新**: 2025年
- **相容性**: Python 3.10+, Arduino IDE 1.8+ 
