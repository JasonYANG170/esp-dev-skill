# AGENTS.md — esp-dev-skill 跨仓库通用约定

> 本文件只覆盖跨仓库的通用约定。各仓库的专属约定（包含模式、项目结构、构建细节）在 `repos/<repo>/AGENTS.md`。

## 作用域

esp-dev-skill 聚合 41 个 ESP 框架/SDK 子技能。主 `SKILL.md` 是路由器；真正的细节在每个 `repos/<repo>/` 里。

## 通用代码约定

- **语言**：C / C++（部分 Python 工具链脚本）。大多数 ESP 仓库是 C；esp-matter、connectedhomeip、esp-claw 等含 C++。
- **入口函数**：ESP-IDF = `void app_main(void)`；Arduino = `setup()` + `loop()`；ESP8266 RTOS SDK = `void app_main(void)`（IDF 风格）。
- **日志**：ESP-IDF/RTOS SDK 用 `ESP_LOGI/E/W/D`（`esp_log.h`，需 `esp_log_level_set`）；Arduino 用 `Serial.print`。
- **错误处理**：ESP-IDF 惯例 `esp_err_t` + `ESP_ERROR_CHECK`；命令式检查 `if (err != ESP_OK)`。
- **配置**：`sdkconfig` + Kconfig（`idf.py menuconfig`）；Arduino 用 `tools/sdk/.../` 预设 + `#define`。

## 选型决策树

1. 要 Arduino 体验？→ `arduino-esp32`
2. ESP8266？→ `ESP8266_RTOS_SDK`
3. 否则默认 → `esp-idf`
4. 在 IDF 之上要某垂直能力？→ 对应领域框架/协议 SDK（见主 SKILL.md 路由表）

## 跨仓库注意

- **版本对齐**：组件的 API 与你用的 ESP-IDF 版本要对齐（子技能里已尽量用 v5.x API）。
- **组件来源**：优先 ESP Component Registry（`idf.py add-dependency espressif/<comp>`），其次源码集成。
- **接地纪律**：所有 recipe/resource 均基于真实仓库；改动 API 签名前先查 `repos/<repo>/resources/api_reference.md` 或真实头文件。

## Do Not Modify

- `repos/<repo>/resources/` —— 各仓库 API 文档来源
- `SKILL.md` frontmatter —— 默认保留；仅为 skill 合规、路由准确性或受支持元数据更新
