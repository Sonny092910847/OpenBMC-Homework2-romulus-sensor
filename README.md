# OpenBMC 作業二：為 ROMULUS 平台新增自定義 Sensor
本文檔記錄了成功為 ROMULUS 平台新增自定義虛擬感測器 MY_HOMEWORK_TEMP 的完整過程。
## 環境設定

* **主機作業系統：** Windows 11
* **虛擬化軟體：**  VirtualBox
* **虛擬機作業系統：** Ubuntu 22.04
* **目標平台：** ROMULUS (AST2500)
* **OpenBMC 版本：** 2025/07/23 自行編譯
* **編譯方式：** Yocto/BitBake

## 實作方法
## 本次實作採用運行時動態設定方式，透過 phosphor-virtual-sensor 服務建立虛擬感測器。
## 1. 編譯 OpenBMC 映像檔

bash# 下載 OpenBMC 原始碼
git clone https://github.com/openbmc/openbmc.git
cd openbmc

# 設定編譯環境
source setup romulus build-romulus

# 使用預設設定編譯
bitbake obmc-phosphor-image
說明：

使用 romulus 平台的預設設定
romulus 預設已包含 phosphor-virtual-sensor 和 bmcweb
編譯時間約 2-4 小時（初次編譯）

編譯結果：

映像檔：tmp/deploy/images/romulus/obmc-phosphor-image-romulus.static.mtd
大小：32MB


## 2. 啟動 QEMU 測試環境

bashcd tmp/deploy/images/romulus/
qemu-system-arm -m 512 -M romulus-bmc -nographic \
  -drive file=obmc-phosphor-image-romulus.static.mtd,format=raw,if=mtd \
  -net nic -net user,hostfwd=:127.0.0.1:2222-:22,hostfwd=:127.0.0.1:2443-:443
等待系統啟動完成（看到 login 提示）。



## 3. 建立虛擬感測器

另開終端機，SSH 登入 BMC：
bashssh root@localhost -p 2222  # 密碼: 0penBmc
檢查現有設定：
bashcat /usr/share/phosphor-virtual-sensor/virtual_sensor_config.json
# 發現已有預設的 Virtual_Inlet_Temp sensor
建立包含 MY_HOMEWORK_TEMP 的新設定：
bashcat > /usr/share/phosphor-virtual-sensor/virtual_sensor_config.json << 'EOF'
[
  {
    "Desc": {
      "Name": "MY_HOMEWORK_TEMP",
      "SensorType": "temperature",
      "MaxValue": 127.0,
      "MinValue": -128.0
    },
    "Threshold": {
      "CriticalHigh": 85,
      "CriticalLow": 5,
      "WarningHigh": 70,
      "WarningLow": 10
    },
    "Associations": [
      [
        "chassis",
        "all_sensors",
        "/xyz/openbmc_project/inventory/system/chassis"
      ]
    ],
    "Params": {
      "ConstParam": [
        {
          "ParamName": "TEMP_BASE",
          "Value": 25.0
        }
      ]
    },
    "Expression": "TEMP_BASE"
  }
]
EOF

# 重啟服務套用設定
systemctl restart phosphor-virtual-sensor


## 4. 驗證感測器

檢查服務狀態
bashsystemctl status phosphor-virtual-sensor
# 應顯示：Added a new virtual sensor: MY_HOMEWORK_TEMP
D-Bus 驗證
bash# 確認感測器註冊
busctl tree xyz.openbmc_project.VirtualSensor
# 輸出：
# └─/xyz/openbmc_project/sensors/temperature/MY_HOMEWORK_TEMP

# 讀取感測器數值
busctl get-property xyz.openbmc_project.VirtualSensor \
  /xyz/openbmc_project/sensors/temperature/MY_HOMEWORK_TEMP \
  xyz.openbmc_project.Sensor.Value Value
# 輸出：d 25


## 5. WebUI 驗證

瀏覽器開啟：https://localhost:2443
接受安全性警告
登入：root / 0penBmc
導航：Hardware Status → Sensors
確認顯示：MY_HOMEWORK_TEMP = 25°C ✅


## 技術發現與解決過程

問題：WebUI 無法顯示感測器
初期遇到感測器在 D-Bus 註冊成功，但 WebUI 無法顯示的問題。
嘗試過的方法：

移除 Associations 欄位 - 失敗
修改不同的 inventory 路徑 - 失敗
檢查 ObjectMapper 關聯 - 找到問題


## 解決方案：

正確設定 Associations 欄位，指向有效的 chassis 路徑：
json"Associations": [
  ["chassis", "all_sensors", "/xyz/openbmc_project/inventory/system/chassis"]
]


## 關鍵發現

Associations 是必要的：沒有正確的關聯，WebUI 無法透過 Redfish API 找到感測器
romulus 沒有 board inventory：必須使用 /system/chassis 而非 /system/board/romulus_board
WebUI 已是 Vue 版本：現有的 WebUI 功能完整，問題在於 ObjectMapper 關聯


## 限制與注意事項

重要：此方法的修改是暫時性的！

修改僅存在於記憶體中
BMC 重啟後設定會消失
適合開發測試使用

若需永久保存，請參考下方的編譯時整合方法。
永久整合方法（選用）
若要將 sensor 設定永久寫入映像檔：
bash# 1. 建立 meta layer 結構
mkdir -p meta-custom/recipes-phosphor/sensors/phosphor-virtual-sensor/

# 2. 放入設定檔
cp virtual_sensor_config.json meta-custom/recipes-phosphor/sensors/phosphor-virtual-sensor/

# 3. 建立 bbappend
cat > meta-custom/recipes-phosphor/sensors/phosphor-virtual-sensor_%.bbappend << 'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI:append = " file://virtual_sensor_config.json"

do_install:append() {
    install -d ${D}${datadir}/phosphor-virtual-sensor/
    install -m 0644 ${WORKDIR}/virtual_sensor_config.json \
        ${D}${datadir}/phosphor-virtual-sensor/
}
EOF

# 4. 加入 layer 並重新編譯
# 在 build-romulus/conf/bblayers.conf 加入 meta-custom
# 執行 bitbake obmc-phosphor-image



## 與原始作業方法的比較

方面原始方法（修改 MTD）本次方法（運行時設定）實作複雜度需解包/重打包 MTD僅需編輯 JSON持久性永久寫入映像檔重啟後消失開發效率每次修改需重新打包即時生效適用場景生產部署開發測試
作業成果
✅ 成功新增 MY_HOMEWORK_TEMP 虛擬感測器
✅ 感測器在 D-Bus 正確註冊並可查詢
✅ WebUI 正常顯示溫度值 25°C
✅ 設定正確的警告與臨界閾值
✅ 了解 phosphor-virtual-sensor 的運作機制
✅ 掌握 OpenBMC sensor 架構與 WebUI 整合


## 參考資源

OpenBMC phosphor-virtual-sensor
OpenBMC Sensor Architecture
OpenBMC WebUI Vue
