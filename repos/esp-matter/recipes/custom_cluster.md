# 自定义 Cluster（厂商扩展）

> **适用摘要**: 在 Matter 设备上加一个厂商自定义 cluster —— 编写 cluster XML 模板、用 `zap_regen_all.py` 生成 app-common 代码、实现属性/命令回调、用 esp-matter 低层 API 把它挂到 endpoint。

## 触发意图

- "加一个自定义 cluster"
- "vendor custom cluster"
- "自定义命令和属性"
- "AttributeAccessInterface / CommandHandlerInterface"
- "ZAP 自定义 cluster"

## 前置条件

| 条件 | 要求 |
|---|---|
| 文档参考 | `docs/en/developing.rst` 的 Custom Cluster 章节 |
| 工具 | connectedhomeip 的 zap 运行环境（`scripts/activate.sh`） |

## 分步说明

### 1. 设计 cluster 并写 XML 模板

cluster ID 的高 16 位是 VendorID（Espressif 为 `0x131B`），低 16 位自分配。示例（来自 `developing.rst`）：

```xml
<?xml version="1.0"?>
<configurator>
  <domain name="CHIP"/>
  <cluster>
    <domain>General</domain>
    <name>Sample ESP</name>
    <code>0x131BFC20</code>
    <define>SAMPLE_ESP_CLUSTER</define>
    <description>The Sample ESP cluster showcases a manufacturer custom cluster</description>

    <attribute side="server" code="0x0000" define="SAMPLE_BOOLEAN" type="boolean"
               writable="true" default="false" optional="false">SampleBoolean</attribute>
    <attribute side="server" code="0x0001" define="SAMPLE_CHAR_STR" type="char_string"
               writable="false" optional="false">SampleCharStr</attribute>

    <command source="client" code="0x00" name="CommandwithoutArgs" optional="false">
      <description>Simple command without any parameters and without a response.</description>
    </command>
    <command source="client" code="0x01" name="CommandWithArgs" response="CommandWithArgsResponse" optional="false">
      <arg name="Arg1" type="int8u"/>
      <arg name="Arg2" type="int8u"/>
    </command>
    <command source="server" code="0x02" name="CommandWithArgsResponse" optional="false" disableDefaultResponse="true">
      <arg name="ResponseArg" type="int8u"/>
    </command>

    <event side="server" code="0x0000" name="TestEvent" priority="info" isFabricSensitive="true" optional="false">
      <field id="1" name="EventData" type="int32u"/>
    </event>
  </cluster>
</configurator>
```

### 2. 注册到 ZAP 配置并重新生成代码

把模板根目录加到 `zcl.json` 和 `zcl-with-test-extensions.json` 的 `xmlRoot`/`xmlFile` 数组，然后：

```bash
cd connectedhomeip/connectedhomeip
source ./scripts/activate.sh
./scripts/tools/zap_regen_all.py
```

codegen 会为 Android/Darwin/Python 控制器、chip-tool 以及 app-common 生成代码。

### 3. （可选）属性托管方式：Attribute Accessors vs AAI

- **Attribute Accessors**：默认方式，复杂类型（struct/array）不能用它。
- **AttributeAccessInterface (AAI)**：继承 `chip::app::AttributeAccessInterface`。若用 AAI，需把属性加到 `zcl.json`/`zcl-with-test-extensions.json` 的 `attributeAccessInterfaceAttributes` 数组，重新 `zap_regen_all.py`，对应的 Accessor API 会被移除。

> 复杂类型（struct/array）属性 **必须** 用 AAI。

### 4. （可选）命令托管方式：Ember 回调 vs CHI

- **Ember command callbacks**：默认。
- **CommandHandlerInterface (CHI)**：继承后把 cluster 加到 `config-data.yaml` 的 `CommandHandlerInterfaceOnlyClusters` 数组再重新生成。

### 5. 在 esp-matter 数据模型里挂上自定义 cluster（不走 ZAP）

如果示例用的是 esp-matter API（非 zap 文件），直接用低层 API：

```cpp
#include <esp_matter.h>
using namespace esp_matter;
using namespace esp_matter::cluster;

uint32_t custom_cluster_id = 0x131bfc20;
cluster_t *cluster = cluster::create(endpoint, custom_cluster_id, CLUSTER_FLAG_SERVER);

// 自定义属性
uint32_t custom_attribute_id = 0x0;
uint16_t default_value = 100;
attribute_t *attr = attribute::create(cluster, custom_attribute_id, ATTRIBUTE_FLAG_NONE,
                                      esp_matter_uint16(default_value));

// 自定义命令
static esp_err_t command_callback(const ConcreteCommandPath &command_path, TLVReader &tlv_data, void *opaque_ptr) {
    ESP_LOGI(TAG, "Custom command callback");
    return ESP_OK;
}
uint32_t custom_command_id = 0x0;
command_t *cmd = command::create(cluster, custom_command_id, COMMAND_FLAG_ACCEPTED, command_callback);
```

### 6. 触发事件（用 connectedhomeip 的 EventLogging）

```cpp
#include <app/EventLogging.h>
// 当事件发生时
chip::app::LogEvent(/* event data */, /* endpoint */, /* event_id */);
// 事件会报告给订阅了该事件的客户端
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| cluster ID 冲突 | 高 16 位没用自己的 VendorID | 高 16 位用你的 VID（Espressif=`0x131B`） |
| 复杂属性读写异常 | 用了 Attribute Accessors | 复杂类型必须用 AAI |
| chip-tool 找不到命令 | 没重新跑 `zap_regen_all.py` | 改完 XML/配置后重新生成并重新编译 chip-tool |
| 事件不报告 | 没订阅 / 没调 `LogEvent` | 客户端 `chip-tool cluster read-event` 且设备端 `chip::app::LogEvent()` |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Custom Cluster（Cluster XML Template / Cluster Implementation / Custom Cluster Attributes/Commands/Events/Functions）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/app_guide.rst`（带 delegate 的 cluster 列表）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/esp_matter_data_model.h`（`cluster::create` / `attribute::create` / `command::create`）
