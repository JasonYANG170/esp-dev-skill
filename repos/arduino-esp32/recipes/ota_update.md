# OTA 固件升级（ArduinoOTA / Update / HTTPUpdate / WebUpdater / Signed OTA）

> **适用摘要**: 使用 `ArduinoOTA`（IDE/网络推送）、`Update`（底层流式写入）、`httpUpdate`（HTTP(S) 拉取）、`OTAWebUpdater`（浏览器上传）实现固件升级，含分区表要求与签名验证流程。

## 触发意图

- "OTA 升级 / 固件空中升级"
- "Arduino IDE 推送 / 网络上传固件"
- "HTTP(S) 下载固件升级"
- "浏览器上传 .bin 升级"
- "签名 OTA / 防刷入未签名固件"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ArduinoOTA/examples/BasicOTA/`、`SignedOTA/`；`libraries/Update/examples/HTTPS_OTA_Update/`、`OTAWebUpdater/`、`Signed_OTA_Update/`；`libraries/HTTPUpdateServer/examples/WebUpdater/` |
| 分区表 | OTA 需 ≥2 个 `ota_` 分区 + 一个 `otadata` 分区；选 Board 自带的 "Default 4MB with spiffs..." 等含 OTA 的方案，或自定义 `partitions.csv`（见 SKILL.md 原则 10） |
| 网络 | ArduinoOTA / HTTPUpdate 需先连上 Wi-Fi（STA 模式） |
| mDNS | ArduinoOTA 依赖 mDNS 广播，主机需装 Bonjour(AirPrint)/Avahi |

## 分步说明

### 1. ArduinoOTA（IDE/espota.py 推送，取自 BasicOTA 示例）

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <ESPmDNS.h>
#include <NetworkUdp.h>
#include <ArduinoOTA.h>

const char *ssid = "..........";
const char *password = "..........";
uint32_t last_ota_time = 0;

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED) {
    Serial.println("Connection Failed! Rebooting...");
    delay(5000); ESP.restart();
  }

  ArduinoOTA
    .onStart([]() {
      String type = (ArduinoOTA.getCommand() == U_FLASH) ? "sketch" : "filesystem";
      Serial.println("Start updating " + type);
    })
    .onEnd([]() { Serial.println("\nEnd"); })
    .onProgress([](unsigned int progress, unsigned int total) {
      if (millis() - last_ota_time > 500) {
        Serial.printf("Progress: %u%%\n", progress / (total / 100));
        last_ota_time = millis();
      }
    })
    .onError([](ota_error_t error) {
      Serial.printf("Error[%u]: ", error);
      if (error == OTA_AUTH_ERROR)    Serial.println("Auth Failed");
      else if (error == OTA_BEGIN_ERROR)  Serial.println("Begin Failed");
      else if (error == OTA_CONNECT_ERROR)Serial.println("Connect Failed");
      else if (error == OTA_RECEIVE_ERROR)Serial.println("Receive Failed");
      else if (error == OTA_END_ERROR)    Serial.println("End Failed");
    });

  ArduinoOTA.begin();      // 默认端口 3232；hostname 默认 esp32-[MAC]
  Serial.print("IP: "); Serial.println(WiFi.localIP());
}
void loop() {
  ArduinoOTA.handle();     // 必须在 loop() 中持续调用
}
```

可选：`setPort(3232)` / `setHostname("myesp32")` / `setPassword("admin")`（PBKDF2-HMAC-SHA256 10000 轮）/ `setPasswordHash("<SHA256>")` / `setRebootOnSuccess(true)` / `setMdnsEnabled(true)` / `setPartitionLabel("spiffs")`（升级文件系统时）。

### 2. Update（底层流式 API，适用于任意来源）

```cpp
#include <Update.h>

// 常用错误码：UPDATE_ERROR_OK/_WRITE/_ERASE/_READ/_SPACE/_SIZE/_STREAM/
//             _MD5/_MAGIC_BYTE/_ACTIVATE/_NO_PARTITION/_BAD_ARGUMENT/_ABORT/_DECRYPT/_SIGN
// 目标：U_FLASH(0) U_SPIFFS(101) U_FATFS(102) U_LITTLEFS(103) U_AUTH(200)

Update.onProgress([](size_t done, size_t total) {
  Serial.printf("%u%%\n", 100 * done / total);
});

if (!Update.begin(firmwareSize)) {          // 或 UPDATE_SIZE_UNKNOWN（0xFFFFFFFF）
  Update.printError(Serial); return;
}
// 写入：Update.write(buf, len) 或 Update.writeStream(stream) 或模板 Update.write(client)
if (!Update.end(true)) {                    // true=即使未写完也收尾
  Update.printError(Serial); return;
}
Serial.println(Update.md5String());         // 完成后 MD5
if (Update.canRollBack()) Update.rollBack();  // 回滚到上一 OTA 分区
```

`begin(size, command=U_FLASH, ledPin=-1, ledOn=LOW, label=NULL)`；`setMD5(hexStr)` 校验；`isRunning()`/`isFinished()`/`progress()`/`remaining()`/`hasError()`/`abort()`。

### 3. HTTPUpdate（HTTP(S) 拉取，取自 HTTPS_OTA_Update 思路）

```cpp
#include <HTTPClient.h>
#include <NetworkClient.h>
#include <HTTPUpdate.h>

NetworkClient client;                          // HTTPS 用 NetworkClientSecure + setCACert()
httpUpdate.onProgress([](int cur, int total) {
  Serial.printf("OTA %d/%d\n", cur, total);
});

t_httpUpdate_return ret = httpUpdate.update(client, "http://example.com/firmware.bin");
// 或：httpUpdate.update(client, host, port, uri, currentVersion);
// 文件系统：httpUpdate.updateSpiffs(...) / updateFatfs(...) / updateLittlefs(...)
switch (ret) {
  case HTTP_UPDATE_FAILED:   Serial.printf("HTTP_UPDATE_FAILED (%d): %s\n", httpUpdate.getLastError(), httpUpdate.getLastErrorString().c_str()); break;
  case HTTP_UPDATE_NO_UPDATES: Serial.println("No update"); break;
  case HTTP_UPDATE_OK:       Serial.println("UPDATE OK"); break;   // 默认自动重启
}
```

辅助：`httpUpdate.rebootOnUpdate(false)` / `setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS)` / `setLedPin(pin, HIGH)` / `setMD5sum(hex)` / `setAuthorization(user, pass)` / `setAuthorization(bearerToken)`。HTTP 错误码（-100..-108）：`HTTP_UE_TOO_LESS_SPACE`/`_SERVER_NOT_REPORT_SIZE`/`_SERVER_FILE_NOT_FOUND`/`_SERVER_WRONG_HTTP_CODE`/`_BIN_VERIFY_HEADER_FAILED`/`_BIN_FOR_WRONG_FLASH`/`_NO_PARTITION`。

### 4. OTAWebUpdater（浏览器上传 .bin，取自 OTAWebUpdater 示例）

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <Update.h>

WebServer server(80);
const char *authUser = "admin", *authPass = "esp";

void handleUpdateEnd() {
  if (!server.authenticate(authUser, authPass)) return server.requestAuthentication();
  server.sendHeader("Connection", "close");
  if (Update.hasError()) {
    server.send(502, "text/plain", Update.errorString());
  } else {
    server.sendHeader("Refresh", "10");
    server.sendHeader("Location", "/");
    server.send(307);
    delay(500); ESP.restart();
  }
}

void handleUpload() {
  HTTPUpload &upload = server.upload();        // upload.buf / currentSize / totalSize / filename
  if (upload.status == UPLOAD_FILE_START) {
    size_t sz = server.hasArg("size") ? server.arg("size").toInt() : UPDATE_SIZE_UNKNOWN;
    if (!Update.begin(sz)) Update.printError(Serial);
  } else if (upload.status == UPLOAD_FILE_WRITE) {
    if (Update.write(upload.buf, upload.currentSize) != upload.currentSize)
      Update.printError(Serial);
  } else if (upload.status == UPLOAD_FILE_END) {
    if (Update.end(true)) Serial.println("Update Success");
    else Update.printError(Serial);
  }
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_AP); WiFi.softAP("esp32-ota");
  server.on("/update", HTTP_POST, handleUpdateEnd, handleUpload);
  server.begin();
}
void loop() { server.handleClient(); }
```

### 5. 签名 OTA（防刷入未签名固件，取自 SignedOTA 示例）

```cpp
#include <ArduinoOTA.h>
#include "public_key.h"     // 由 tools/bin_signing.py 生成

// 哈希与签名算法必须与签名工具一致
static UpdaterRSAVerifier sign(PUBLIC_KEY, PUBLIC_KEY_LEN, HASH_SHA256);   // 或 UpdaterECDSAVerifier

void setup() {
  WiFi.mode(WIFI_STA); WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED) { delay(5000); ESP.restart(); }

  ArduinoOTA.setSignature(&sign);             // 必须在 begin() 之前
  // ArduinoOTA.setPassword("xxx");           // 可叠加口令保护
  ArduinoOTA.onStart([](){}).onEnd([](){}).onError([](ota_error_t e){});
  ArduinoOTA.begin();
}
void loop() { ArduinoOTA.handle(); }
```

构建步骤（见示例 README）：
1. 生成密钥：`python tools/bin_signing.py --generate-key rsa-2048 --out private_key.pem`
2. 提取公钥：`python tools/bin_signing.py --extract-pubkey private_key.pem --out public_key.pem` → 转为 `public_key.h`
3. 在 sketch 目录用 `build_opt.h` 启用 `UPDATE_SIGN`
4. 编译并签名：`arduino-cli compile ... --export-binaries` → `python tools/bin_signing.py --bin build/x.bin --key private_key.pem --out firmware_signed.bin`（`--hash sha256`）
5. 推送：`python tools/espota.py -i <ip> -f firmware_signed.bin`

哈希类型 `HASH_SHA256`/`_SHA384`/`_SHA512`；签名算法 `UpdaterRSAVerifier`（rsa-2048/3072/4096）或 `UpdaterECDSAVerifier`（ecdsa-p256/p384）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `OTA_BEGIN_ERROR` / 空间不足 | 分区表无 OTA 分区 / sketch 太大 | 选含 `ota_0`/`ota_1`/`otadata` 的分区方案；或 `Huge APP` 但仅单 OTA |
| IDE 找不到设备 | mDNS 未广播 / 跨子网 | 主机装 Bonjour/Avahi；同子网；`ArduinoOTA.setMdnsEnabled(true)` |
| `handle()` 卡住 loop | 升级期间 loop 被阻塞 | 正常，升级时看门狗由库处理；确保 loop 持续调用 |
| `UPDATE_ERROR_MAGIC_BYTE` | 写入数据不是合法固件头 | 校验 URL/文件；HTTPS 证书；`setMD5` 是否匹配 |
| 签名验证失败 `OTA_END_ERROR` | 固件未签名 / 公钥不匹配 / 哈希不一致 | 用同一私钥签名；`--hash` 与 `HASH_*` 一致；`public_key.h` 来自对应私钥 |
| HTTPS OTA 失败 | 证书校验 | `NetworkClientSecure::setCACert(rootCA)`；或 `setInsecure()`（仅测试） |
| 升级后无效果 | otadata 未更新 / 未重启 | `Update.end(true)` 成功后重启；`setRebootOnSuccess(true)` |
| 回滚后跑老固件 | `canRollBack()` 触发 | `rollBack()` 后重启会切回上一分区 |

## 参考

- `libraries/ArduinoOTA/examples/BasicOTA/BasicOTA.ino`、`SignedOTA/SignedOTA.ino`
- `libraries/Update/examples/HTTPS_OTA_Update/`、`OTAWebUpdater/`、`Signed_OTA_Update/`、`SD_Update/`、`AWS_S3_OTA_Update/`
- `libraries/HTTPUpdate/src/HTTPUpdate.h`、`libraries/HTTPUpdateServer/examples/WebUpdater/`
- `libraries/Update/src/Update.h`（错误码、目标常量、AES 解密 API）
- 仓库文档 `docs/en/ota_web_update.rst`
