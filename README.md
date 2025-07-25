# OpenBMC 作業二：為 ROMULUS 平台新增自定義 Sensor
本文檔記錄了成功為 ROMULUS 平台新增自定義虛擬感測器 MY_HOMEWORK_TEMP 的完整過程，並透過 WebUI 和 ipmitool sdr elist 驗證。

## 環境設定

* **主機作業系統：** Windows 11
* **虛擬化軟體：** VirtualBox
* **虛擬機作業系統：** Ubuntu 22.04
* **目標平台：** ROMULUS (AST2500)
* **OpenBMC 版本：** 2.14.0
* **編譯方式：** Yocto/BitBake

## 實作目標
1. 新增自定義感測器 MY_HOMEWORK_TEMP 到 OpenBMC
2. 在 WebUI 中顯示感測器數值 ✅
3. 透過 `ipmitool sdr elist` 指令確認感測器 ✅

## 第一部分：WebUI 整合

## 1. 編譯 OpenBMC 映像檔

```bash
git clone https://github.com/openbmc/openbmc.git
cd openbmc

# 設定編譯環境
. setup romulus

# 修改 local.conf 加入 ipmitool
echo 'IMAGE_INSTALL:append = " ipmitool"' >> build-romulus/conf/local.conf
echo 'IMAGE_INSTALL:append = " phosphor-virtual-sensor"' >> build-romulus/conf/local.conf

# 編譯
bitbake obmc-phosphor-image
```

## 2. 建立虛擬感測器設定

```bash
在 BMC 執行時建立設定：
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

systemctl restart phosphor-virtual-sensor
```

## 3. WebUI 驗證

```bash
瀏覽器開啟：https://localhost:2443
登入：root / 0penBmc
導航：Hardware Status → Sensors
確認顯示：MY_HOMEWORK_TEMP = 25°C ✅
```

## 第二部分：IPMI 整合

## 1. 修改 IPMI Sensor YAML 配置

```bash
在編譯前修改 meta-ibm/meta-romulus/recipes-phosphor/configuration/romulus-yaml-config/romulus-ipmi-sensors.yaml，新增第 146 項：
yaml146:
    bExp: 0
    entityID: 7
    entityInstance: 1
    interfaces:
        xyz.openbmc_project.Sensor.Value:
            Value:
                Offsets:
                    255:
                        type: double
    multiplierM: 1
    mutability: Mutability::Write|Mutability::Read
    offsetB: 0
    path: /xyz/openbmc_project/sensors/temperature/MY_HOMEWORK_TEMP
    rExp: 0
    readingType: readingData
    scale: 0
    sensorNamePattern: nameLeaf
    sensorReadingType: 1
    sensorType: 1
    serviceInterface: org.freedesktop.DBus.Properties
    unit: xyz.openbmc_project.Sensor.Value.Unit.DegreesC
關鍵參數說明：

sensorType: 1 - 溫度感測器
entityID: 7 - Entity ID
serviceInterface: org.freedesktop.DBus.Properties - 正確的 D-Bus 介面
offsetB: 0 和 scale: 0 - 確保數值正確轉換
```

## 2. 重新編譯並啟動

```bash
bashbitbake obmc-phosphor-image

# 啟動 QEMU
qemu-system-arm -m 512 -M romulus-bmc -nographic \
  -drive file=obmc-phosphor-image-romulus.static.mtd,format=raw,if=mtd \
  -net nic -net user,hostfwd=:127.0.0.1:2222-:22
```

## 3. IPMI 驗證

```bash
bash# 在 BMC 內部執行
ipmitool sdr elist | grep MY_HOMEWORK_TEMP
# 輸出：MY_HOMEWORK_TEMP | 92h | ok | 7.1 | 25.000 degrees C ✅
```

# 詳細資訊

```bash
ipmitool sensor get "MY_HOMEWORK_TEMP"

# 顯示完整感測器資訊，包含閾值設定
技術要點與問題解決
WebUI 整合關鍵

Associations 必要性：WebUI 透過 Redfish API 查詢感測器，需要正確的 inventory 關聯
路徑問題：romulus 使用 /system/chassis 而非 /system/board

IPMI 整合關鍵

serviceInterface 設定：必須使用 org.freedesktop.DBus.Properties 而非 xyz.openbmc_project.VirtualSensor
數值轉換參數：

offsetB 影響基準值偏移
scale 影響數值縮放
錯誤設定會導致顯示 0 度或 "Disabled"
```


常見問題

* **ipmitool 未安裝：** 需在 local.conf 中加入 IMAGE_INSTALL
* **感測器顯示 Disabled：** Virtual Sensor 服務未正確載入
* **溫度顯示 0 度：** YAML 中的數值轉換參數設定錯誤

永久化方案
若需將設定永久寫入映像檔：
## 1. 建立 bbappend

```bash
mkdir -p meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-virtual-sensor
cat > meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-virtual-sensor/phosphor-virtual-sensor_%.bbappend << 'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI += "file://virtual_sensor_config.json"

do_install:append() {
    install -d ${D}${datadir}/phosphor-virtual-sensor
    install -m 0644 ${WORKDIR}/virtual_sensor_config.json ${D}${datadir}/phosphor-virtual-sensor/
}
EOF
```

## 2. 放入設定檔並重新編譯
作業成果總結

* ✅  成功新增 MY_HOMEWORK_TEMP 虛擬感測器
* ✅  WebUI 正常顯示溫度值 25°C
* ✅  ipmitool sdr elist 正確顯示感測器狀態和數值
* ✅  了解 OpenBMC 的 D-Bus 與 IPMI 層整合機制
* ✅  掌握 phosphor-virtual-sensor 和 phosphor-ipmi-host 的協作方式

參考資源

* OpenBMC Sensor Architecture
* phosphor-virtual-sensor
* phosphor-host-ipmid


這個更新版本完整記錄了 WebUI 和 ipmitool 的實作過程，特別強調了 IPMI 整合的關鍵設定和問題解決方法！
