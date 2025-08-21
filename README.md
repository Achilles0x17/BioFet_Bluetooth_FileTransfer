# 藍牙檔案傳輸系統
## 技術文件

### 執行摘要

本文件記錄了一個跨平台藍牙檔案傳輸系統，能夠在 Linux 與 Android 裝置之間實現無縫的檔案交換。系統由基於 PyQt6 的 Linux 應用程式以及基於 Kivy 的 Android 應用程式組成，使用 RFCOMM 藍牙協議以實現可靠的資料傳輸。

---

## 系統架構

### 概覽

系統採用點對點架構，由 Linux 應用程式發起連線並將檔案傳送至 Android 裝置。通訊透過藍牙序列埠配置檔（SPP）使用 RFCOMM 協議進行。

### 主要組件

- **Linux 應用程式**：基於 PyQt6 的 GUI 應用程式，具備藍牙裝置偵測及檔案管理功能
- **Android 應用程式**：基於 Kivy 的應用程式，提供檔案儲存與匯出功能
- **協議**：自訂二進位檔案傳輸及目錄同步協議
- **儲存**：Android 應用程式專用內部儲存，採結構化目錄管理

---

## Linux 應用程式架構

### 核心類別

#### BluetoothScanner 類別
```python
class BluetoothScanner(QObject):
```

**職責：**
- 實作 PyQt6 的藍牙裝置偵測
- 使用 QBluetoothDeviceDiscoveryAgent 掃描附近裝置
- 提供即時裝置偵測並透過訊號進行溝通
- 自動過濾並顯示可偵測的藍牙裝置

**主要功能：**
- 非同步裝置掃描，並具備逾時處理
- 根據 MAC 位址過濾重複裝置
- 使用 Qt 訊號確保執行緒安全的 GUI 更新
- 自動停止掃描以避免資源耗盡

#### BluetoothClientGUI 類別
```python
class BluetoothClientGUI(QMainWindow):
```

**職責：**
- 主應用程式視窗，提供完整的檔案傳輸介面
- 實作三狀態連線管理（已斷線/連線中/已連線）
- 提供即時日誌顯示，訊息類型以顏色區分
- 支援目錄同步操作

**GUI 元件：**
- 裝置選擇清單，具備 MAC 位址驗證
- 連線狀態指示器與視覺回饋
- 檔案傳輸進度監控
- 遠端指令輸入介面

#### BluetoothFileClient 類別
```python
class BluetoothFileClient:
```

**職責：**
- 負責所有藍牙通訊協議
- 實作可靠的連線建立與埠掃描
- 管理目錄同步並檢查檔案完整性
- 支援目錄掃描及批次檔案操作

**傳輸協議功能：**
- 目錄同步流程
- 目錄掃描與檔案列舉
- 批次檔案驗證與過濾
- 提供使用者確認對話框，列出詳細檔案清單
- 依序傳輸檔案並處理錯誤

### 連線管理

#### 埠偵測流程
```python
def find_and_connect(self, cancel_event=None):
    # Try common RFCOMM ports first (1, 2, 3, then scan others)
    port_order = [1, 2, 3] + list(range(4, 31))
```

**功能：**
- 優先嘗試常用的 RFCOMM 埠
- 支援使用者中斷操作
- 採用漸進式逾時策略
- 提供詳細連線嘗試日誌

---

## 檔案傳輸協議

### 目錄同步協議

```
1. Send "DIRSYNC" header (8 bytes)
2. Wait for "READY" response
3. Send directory name (4-byte length + name)
4. Send file count (4 bytes)
5. Wait for "CLEARED" response
6. For each file:
   - Send "NEWFILE" header
   - Wait for "READY"
   - Send metadata (filename, size, type)
   - Wait for "READY" or "REJECT"
   - Stream file data
   - Wait for "SAVED" confirmation
```

### 錯誤處理與恢復

- 連線逾時管理與漸進重試
- 檔案損壞檢測（透過大小驗證）
- 傳輸過程可由使用者取消
- 提供完整錯誤日誌與使用者回饋

---

## Android 應用程式架構

### 核心類別

#### MyGridLayout 類別
```python
class MyGridLayout(Widget):
```

**職責：**
- 主 UI 控制器，管理藍牙通訊
- 實作檔案儲存與管理系統
- 提供使用 Android Storage Access Framework 的匯出功能
- 支援多工傳輸操作

### 藍牙伺服器實作

```python
def start_bluetooth_server(self):
    # Standard UUID for SPP (Serial Port Profile)
    MY_UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
```

**功能：**
- 透過 PyJNIus 使用 Android 藍牙 API
- 實作 RFCOMM socket 監聽
- 支援多次連線嘗試
- 提供自動服務偵測

### 檔案儲存系統

#### 內部儲存管理
```python
def initialize_storage(self):
    context = mActivity.getApplicationContext()
    files_dir = context.getFilesDir()
    self.app_storage_path = files_dir.getAbsolutePath()
```

**功能：**
- 使用 Android 應用程式專用內部儲存
- 建立結構化目錄以管理接收的檔案
- 維護記憶體中的檔案目錄以加速存取
- 自動整理檔案並維持目錄結構

#### 儲存功能

- **目錄組織**：依來源目錄名稱分類檔案
- **Metadata 追蹤**：紀錄檔案大小、類型及路徑
- **匯出功能**：整合 Android Storage Access Framework
- **儲存優化**：自動索引檔案並建立目錄結構

### 傳輸處理

#### 協議實作

Android 應用程式負責配合 Linux 端的傳輸協議：

**接受連線**
- 在 RFCOMM socket 上監聽
- 完成連線確認握手
- 建立雙向通訊

**接收檔案**
- 驗證接收目錄的 metadata
- 支援多檔案串流接收
- 提供批次操作即時進度回饋
- 確保所有檔案完整性

**目錄同步**
- 清空現有目錄內容
- 依序接收多個檔案
- 維持傳輸狀態一致
- 提供批次操作回饋

### 使用者介面功能

#### 指令系統
```python
def send_message(self):
    # Built-in commands: list, clear, show, export, shutdown
```

**功能：**
- 提供互動式檔案管理指令介面
- 快速操作按鈕執行常用操作
- 即時日誌顯示，訊息以顏色區分
- 連線與傳輸狀態指示

#### 匯出功能
```python
def create_file_with_extension(filetype, default_name="received_file"):
    intent = Intent(Intent.ACTION_CREATE_DOCUMENT)
```

**功能：**
- 整合 Android Storage Access Framework
- 支援多種檔案類型與 MIME type
- 使用者可自訂檔名
- 直接匯出至使用者可存取的儲存位置

---

## 執行緒與併發處理

**Linux 端執行緒管理：**
- **GUI 執行緒**：處理 PyQt6 GUI 更新與使用者互動
- **藍牙掃描執行緒**：非同步裝置掃描，透過 Qt 訊號更新 GUI
- **連線執行緒**：執行埠掃描與連線建立，支援取消操作
- **指令接收執行緒**：持續監聽 Android 裝置指令，管理傳輸狀態
- **檔案傳輸執行緒**：獨立處理檔案傳送，避免 UI 凍結

---

## 建置系統與部署

### 虛擬環境設定

```bash
python3 -m venv bluetooth_transfer_env
source bluetooth_transfer_env/bin/activate
pip install --upgrade pip
pip install PyQt6
pip install kivy
pip install buildozer
pip install pyinstaller
deactivate
```

### Android APK 建置 (Buildozer)

```ini
[app]
title = Bluetooth File Server
package.name = bluetoothfileserver
package.domain = org.example
requirements = python3,kivy,pyjnius
android.permissions = BLUETOOTH, BLUETOOTH_ADMIN, BLUETOOTH_CONNECT, BLUETOOTH_ADVERTISE, ACCESS_FINE_LOCATION, READ_EXTERNAL_STORAGE,WRITE_EXTERNAL_STORAGE
```

```bash
buildozer -v android debug
```

### Linux 可執行檔建置 (PyInstaller)

```bash
pyinstaller --hidden-import=PyQt6 --hidden-import=PyQt6.QtCore --hidden-import=PyQt6.QtWidgets --hidden-import=PyQt6.QtGui --hidden-import=PyQt6.QtBluetooth path/to/your/client.py -F
```

---

## 結論

此藍牙檔案傳輸系統展示了跨平台 Python 應用程式開發的可行性。Linux 端使用 PyQt6，Android 端使用 Kivy，提供原生體驗的 UI。自訂二進位協議確保資料可靠傳輸，而 Buildozer 與 PyInstaller 提供簡易的部署與發行流程。

### 解決的實務開發挑戰

- UI 在檔案傳輸過程中保持響應
- 藍牙資料可靠傳輸
- 跨平台簡單部署

系統架構清晰，可方便後續擴充功能。