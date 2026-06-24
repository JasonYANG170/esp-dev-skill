# WebServer HTTP 服务（路由 / 认证 / 文件上传 / 中间件 / OTA 上传）

> **适用摘要**: 使用 `WebServer` 库（基于 3.x `Network` 库）实现 HTTP 路由、Basic/Digest 认证、文件上传、`serveStatic`、流式响应，并与 `Update` 联动做网页 OTA。单客户端同步模型。

## 触发意图

- "HTTP 服务 / Web 控制"
- "WebServer 路由 / REST API"
- "HTTP Basic Auth / Digest 认证"
- "网页上传文件 / OTA Web 上传"
- "serveStatic 静态文件 / LittleFS"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/WebServer/examples/HelloServer/`、`AdvancedWebServer/`、`HttpBasicAuth/`、`FSBrowser/`、`WebUpdate/`、`UploadHugeFile/` |
| 网络 | 先 `WiFi.begin` 连上 AP（或 `softAP`）；`WebServer` 内部用 `NetworkServer` 监听 |
| 单客户端 | 该库只处理一个并发客户端；`handleClient()` 需在 `loop()` 中持续调用 |
| include | `<WebServer.h>`（自动拉入 `Network.h`） |

## 分步说明

### 基础路由（取自 HelloServer 示例）

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <NetworkClient.h>
#include <WebServer.h>
#include <ESPmDNS.h>

WebServer server(80);

void handleRoot() {
  server.send(200, "text/plain", "hello from esp32!");
}

void handleNotFound() {
  String msg = "File Not Found\n";
  msg += "URI: " + server.uri();
  msg += "\nMethod: " + String((server.method() == HTTP_GET) ? "GET" : "POST");
  msg += "\nArguments: " + String(server.args()) + "\n";
  for (int i = 0; i < server.args(); i++)
    msg += " " + server.argName(i) + ": " + server.arg(i) + "\n";
  server.send(404, "text/plain", msg);
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  if (MDNS.begin("esp32")) Serial.println("MDNS responder started");

  server.on("/", handleRoot);
  server.on("/inline", []() {              // 内联 lambda
    server.send(200, "text/plain", "this works as well");
  });
  server.onNotFound(handleNotFound);
  server.begin();
}
void loop() {
  server.handleClient();
  delay(2);                                // 让 CPU 切换其它任务
}
```

### 路由 / 方法绑定

```cpp
// 重载：on(uri, fn) / on(uri, method, fn) / on(uri, method, fn, uploadFn)
server.on("/api", HTTP_GET, handleGet);    // 限定方法
server.on("/upload", HTTP_POST, onUploadEnd, onUploadBody);  // 上传时 uploadFn 处理数据
bool removed = server.removeRoute("/api", HTTP_GET);
server.serveStatic("/", LittleFS, "/www/", "max-age=86400");   // 静态文件 + 缓存头
server.addHandler(...);                    // 自定义 RequestHandler
server.onNotFound([](){ server.send(404); });
server.onFileUpload([](){ /* 默认上传钩子 */ });
```

方法枚举：`HTTP_GET`/`HTTP_POST`/`HTTP_PUT`/`HTTP_DELETE`/`HTTP_PATCH`/`HTTP_HEAD`/`HTTP_OPTIONS`/`HTTP_ANY`。

### 参数 / 请求头 / 响应头

```cpp
// 查询参数与表单字段
server.args();                 // 参数总数
server.hasArg("name");
server.arg("name");            // 按名取值
server.arg(0);                 // 按序取值
server.argName(0);
server.pathArg(0);             // 路径参数（如 /api/<id>）

// 请求头（需先 collectHeaders）
const char *keys[] = {"Origin", "Host"};
server.collectHeaders(keys, 2);
server.headers(); server.header("Origin"); server.headerName(0); server.hasHeader("Host");
server.collectAllHeaders();    // 收集所有请求头

// 响应
server.send(200, "text/plain", "ok");          // 简单
server.send(200, "text/html", F("<h1>Hi</h1>"));
server.send_P(200, PSTR("text/plain"), PSTR("from progmem"));   // flash 字符串
server.sendHeader("Content-Encoding", "gzip");
server.sendHeader("Connection", "close", /*first=*/true);
server.setContentLength(1234);
server.sendContent("chunk1");
server.sendContent_P(PSTR("chunk2"));

// 流式（如文件）
File f = LittleFS.open("/big.dat", "r");
server.streamFile(f, "application/octet-stream");   // 返回写入字节数
server.client().write(...);    // 直接操作底层 NetworkClient

// 分块编码
server.chunkResponseBegin("text/event-stream");
server.chunkWrite(data, len);
server.chunkResponseEnd();
```

### Basic / Digest 认证（取自 HttpBasicAuth 示例）

```cpp
const char *user = "admin", *pass = "esp32";

server.on("/", []() {
  if (!server.authenticate(user, pass)) {
    return server.requestAuthentication();          // 默认 BASIC_AUTH
  }
  server.send(200, "text/plain", "Login OK");
});

// Digest + realm：
server.requestAuthentication(DIGEST_AUTH, "esp32-realm", "Auth failed");

// 自定义校验函数
server.authenticate([](HTTPAuthMethod mode, String user, String params[]) -> String * {
  // BASIC_AUTH: params[0]=输入密码, params[1]=realm；返回期望密码或 nullptr
  return nullptr;
});

// SHA1 密码校验（不传明文）
server.authenticateBasicSHA1(user, sha1Base64);
```

`HTTPAuthMethod`: `BASIC_AUTH` / `DIGEST_AUTH` / `OTHER_AUTH`。

### 文件上传（与 Update 联动做网页 OTA，取自 OTAWebUpdater 示例）

```cpp
#include <Update.h>

void handleUpdateEnd() {
  if (Update.hasError()) server.send(502, "text/plain", Update.errorString());
  else {
    server.sendHeader("Connection", "close");
    server.send(200, "text/plain", "Rebooting...");
    delay(500); ESP.restart();
  }
}

void handleUploadBody() {
  HTTPUpload &upload = server.upload();        // upload.status / filename / buf / currentSize / totalSize
  switch (upload.status) {
    case UPLOAD_FILE_START:
      if (!Update.begin(UPDATE_SIZE_UNKNOWN)) Update.printError(Serial);
      break;
    case UPLOAD_FILE_WRITE:
      if (Update.write(upload.buf, upload.currentSize) != upload.currentSize)
        Update.printError(Serial);
      break;
    case UPLOAD_FILE_END:
      if (!Update.end(true)) Update.printError(Serial);
      break;
    case UPLOAD_FILE_ABORTED:
      Update.abort(); break;
  }
}

server.on("/update", HTTP_POST, handleUpdateEnd, handleUploadBody);
```

`HTTPUpload` 字段：`status`(`UPLOAD_FILE_START`/`_WRITE`/`_END`/`_ABORTED`)、`filename`、`name`、`type`、`totalSize`、`currentSize`、`buf[HTTP_UPLOAD_BUFLEN=1436]`。也有 `HTTPRaw`/`raw()` 用于非 multipart 原始 body（`RAW_START`/`RAW_WRITE`/`RAW_END`/`RAW_ABORTED`）。

### 中间件（3.x 新增）

```cpp
#include "middleware/Middleware.h"
server.addMiddleware([](Middleware::Context ctx) {
  // 例如校验 Origin / 鉴权 / 限流 / 日志
  ctx.next();                       // 放行
  // 或 ctx.response().set(...) 短路返回
});
server.removeMiddleware(mwPtr);
```

### 杂项

```cpp
server.enableCORS(true);            // 跨域
server.enableCrossOrigin(true);
server.enableETag(true, [](FS &fs, const String &fName) -> String { return ""; });
server.enableDelay(false);          // 响应后是否 delay 让出 CPU
server.uri();                       // 当前请求 URI
server.method();                    // HTTPMethod
server.client();                    // 底层 NetworkClient
server.responseCode();              // 响应码
WebServer::urlDecode("%41%42");     // 静态工具
WebServer::responseCodeToString(404);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 页面打不开 / 超时 | `handleClient()` 未在 loop 调用 / 被阻塞 | loop 中持续调用；阻塞任务移到独立 FreeRTOS 任务 |
| 同时只能一个客户端 | 库为单客户端同步模型 | 高并发用 `NetworkServer` 自行实现或加反向代理 |
| `onUpload` 不触发 | 注册方式错 | 用 `on(uri, HTTP_POST, endFn, uploadFn)` 四参重载 |
| OTA 上传到一半失败 | `Update.begin` 未传 size / 客户端断开 | `UPDATE_SIZE_UNKNOWN` 允许未知大小；检查 Wi-Fi 稳定性 |
| 认证一直失败 | realm / Digest nonce 不一致 | 同一 realm；Digest 需稳定 nonce；用 `authenticateBasicSHA1` 避免明文 |
| 静态文件 404 | LittleFS/SPIFFS 未 `begin()` / 路径错 | 先 `LittleFS.begin(true)`；`serveStatic("/","LittleFS","/www/")` 路径加前导 `/` |
| 大响应 OOM | 用 `String` 拼大 HTML | 用 `sendContent` 分块 / `streamFile` / `send_P` 走 flash |
| CORS 失败 | 未启用 | `server.enableCORS(true)` 或手动 `sendHeader("Access-Control-Allow-Origin","*")` |
| 响应头重复 / 顺序错 | `sendHeader` 时机 | `first=true` 强制置顶；`send` 前设完所有头 |

## 参考

- `libraries/WebServer/examples/HelloServer/`、`AdvancedWebServer/`（SVG 动态图）
- `libraries/WebServer/examples/HttpBasicAuth/`（Basic/Digest 认证）
- `libraries/WebServer/examples/FSBrowser/`（LittleFS 文件浏览 + 上传）
- `libraries/WebServer/examples/WebUpdate/`、`UploadHugeFile/`（网页 OTA / 大文件上传）
- `libraries/WebServer/src/WebServer.h`、`src/middleware/Middleware.h`
- 仓库文档 `docs/en/libraries.rst`
