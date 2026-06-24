# arduino-esp32 API 速查（取自 `docs/en/api/*.rst` 与库头文件）

> 本文档所有签名、枚举、宏均来自仓库真实文档与源码。未列出的 API = 不在本 skill 覆盖范围；如需扩展请回仓库 `docs/en/api/` 核对后再加。

## GPIO（`docs/en/api/gpio.rst`）

```cpp
void pinMode(uint8_t pin, uint8_t mode);      // INPUT / OUTPUT / INPUT_PULLUP / INPUT_PULLDOWN
void digitalWrite(uint8_t pin, uint8_t val);  // HIGH / LOW
int  digitalRead(uint8_t pin);

attachInterrupt(uint8_t pin, voidFuncPtr handler, int mode);          // ISR 无参
attachInterruptArg(uint8_t pin, voidFuncPtrArg handler, void *arg, int mode);
detachInterrupt(uint8_t pin);
// 中断模式: DISABLED RISING FALLING CHANGE ONLOW ONHIGH ONLOW_WE ONHIGH_WE
```

## Serial / UART（`docs/en/api/serial.rst`）

```cpp
void begin(unsigned long baud, uint32_t config=SERIAL_8N1,
           int8_t rxPin=-1, int8_t txPin=-1, bool invert=false,
           unsigned long timeout_ms=20000UL, uint8_t rxfifo_full_thrhd=120);
void end();
int  available(); int availableForWrite();
int  read();  size_t read(uint8_t *buf, size_t size); size_t readBytes(uint8_t *buf, size_t len);
int  peek();  void flush();  void flush(bool txOnly);
size_t write(uint8_t); size_t write(const uint8_t *buf, size_t size); size_t write(const char *s);

bool setPins(int8_t rxPin, int8_t txPin, int8_t ctsPin=-1, int8_t rtsPin=-1);
size_t setRxBufferSize(size_t n);   // 必须在 begin() 前
size_t setTxBufferSize(size_t n);   // 必须在 begin() 前
bool setRxTimeout(uint8_t symbols_timeout);
bool setRxFIFOFull(uint8_t fifoBytes);
void onReceive(OnReceiveCb fn, bool onlyOnTimeout=false);            // typedef std::function<void(void)>
void onReceiveError(OnReceiveErrorCb fn);                            // typedef std::function<void(hardwareSerial_error_t)>
void eventQueueReset();
bool setHwFlowCtrlMode(SerialHwFlowCtrl mode=UART_HW_FLOWCTRL_CTS_RTS, uint8_t threshold=64);
bool setMode(SerialMode mode);     // UART_MODE_UART / UART_MODE_RS485_HALF_DUPLEX / UART_MODE_IRDA ...
bool setIrdaDirection(esp32_uart_irda_direction_t dir);              // ESP32_UART_IRDA_TX / _RX
bool setClockSource(SerialClkSrc clkSrc);   // UART_CLK_SRC_APB / _PLL / _XTAL / _RTC / _REF_TICK
bool setRxInvert(bool); bool setTxInvert(bool); bool setCtsInvert(bool); bool setRtsInvert(bool);
void setDebugOutput(bool enable);
uint32_t baudRate(); void updateBaudRate(unsigned long baud);
// 配置常量: SERIAL_5/6/7/8 N/E O 1/2 -> e.g. SERIAL_8N1
// 流控: UART_HW_FLOWCTRL_DISABLE/_RTS/_CTS/_CTS_RTS
// 错误: UART_NO_ERROR / _BREAK_ERROR / _BUFFER_FULL_ERROR / _FIFO_OVF_ERROR / _FRAME_ERROR / _PARITY_ERROR
// HAL loopback: uart_internal_loopback(uartNum, rxPin); uart_internal_hw_flow_ctrl_loopback(uartNum, ctsPin);
```

## I2C / Wire（`docs/en/api/i2c.rst`）

```cpp
bool begin();                                              // 主机默认
bool begin(int sdaPin, int sclPin, uint32_t frequency);    // 主机自定义
bool begin(uint8_t addr, int sdaPin, int sclPin, uint32_t frequency); // 从机
bool setPins(int sdaPin, int sclPin);   // 必须在 begin() 前
bool setClock(uint32_t frequency); uint32_t getClock();
void setTimeOut(uint16_t ms); uint16_t getTimeOut();
void beginTransmission(uint16_t address);
uint8_t endTransmission(bool sendStop=true);
uint8_t requestFrom(uint16_t address, uint8_t size, bool sendStop=true);
size_t write(uint8_t); size_t write(const uint8_t*, size_t);
void onReceive(const std::function<void(int)>& cb);   // void(int numBytes)
void onRequest(const std::function<void()>& cb);      // void()
size_t slaveWrite(const uint8_t*, size_t);            // 仅 ESP32
bool end();
```

## SPI（`docs/en/api/spi.rst`，基础遵循 Arduino SPI）

```cpp
SPIClass(spi_host_device_t host);   // FSPI / HSPI
void begin(int8_t sck=-1, int8_t miso=-1, int8_t mosi=-1, int8_t ss=-1);
void beginTransaction(SPISettings settings);   // SPISettings(freq, bitOrder, mode)
void endTransaction();
uint8_t transfer(uint8_t data); void transfer(uint8_t *buf, size_t count);
void end();
// 位序: MSBFIRST / LSBBFIRST; 模式: SPI_MODE0..3
```

## LEDC / PWM（`docs/en/api/ledc.rst`，3.x）

```cpp
bool ledcSetClockSource(ledc_clk_cfg_t source); ledc_clk_cfg_t ledcGetClockSource();
bool ledcAttach(uint8_t pin, uint32_t freq, uint8_t resolution);                 // 分辨率 1-14（ESP32 1-20）
bool ledcAttachChannel(uint8_t pin, uint32_t freq, uint8_t resolution, int8_t channel);
bool ledcWrite(uint8_t pin, uint32_t duty);
bool ledcWriteChannel(uint8_t channel, uint32_t duty);
uint32_t ledcRead(uint8_t pin); uint32_t ledcReadFreq(uint8_t pin);
uint32_t ledcWriteTone(uint8_t pin, uint32_t freq);                              // freq=0 停止
uint32_t ledcWriteNote(uint8_t pin, note_t note, uint8_t octave);                // NOTE_C..NOTE_B
bool ledcDetach(uint8_t pin);
uint32_t ledcChangeFrequency(uint8_t pin, uint32_t freq, uint8_t resolution);
bool ledcOutputInvert(uint8_t pin, bool out_invert);
bool ledcFade(uint8_t pin, uint32_t start_duty, uint32_t target_duty, int max_fade_time_ms);
bool ledcFadeWithInterrupt(uint8_t pin, uint32_t s, uint32_t t, int ms, void (*fn)(void));
bool ledcFadeWithInterruptArg(uint8_t pin, uint32_t s, uint32_t t, int ms, void (*fn)(void*), void *arg);
// Arduino 兼容
void analogWrite(uint8_t pin, int value);                  // 0..255
void analogWriteResolution(uint8_t pin, uint8_t resolution);
void analogWriteFrequency(uint8_t pin, uint32_t freq);
```

## ADC（`docs/en/api/adc.rst`）

```cpp
uint16_t analogRead(uint8_t pin);
uint32_t analogReadMilliVolts(uint8_t pin);     // 校准毫伏
void analogReadResolution(uint8_t bits);        // 1..16（ESP32 实改硬件 9..12）
void analogSetAttenuation(adc_attenuation_t a); // ADC_ATTEN_DB_0 / _DB_2_5 / _DB_6 / _DB_11
void analogSetPinAttenuation(uint8_t pin, adc_attenuation_t a);
void analogSetWidth(uint8_t bits);              // 仅 ESP32，9..12

// 连续模式
bool analogContinuous(const uint8_t pins[], size_t pins_count, uint32_t conversions_per_pin,
                      uint32_t sampling_freq_hz, void (*userFunc)(void));
bool analogContinuousRead(adc_continuous_result_t **buffer, uint32_t timeout_ms);
bool analogContinuousStart(); bool analogContinuousStop(); bool analogContinuousDeinit();
void analogContinuousSetAtten(adc_attenuation_t a); void analogContinuousSetWidth(uint8_t bits);

typedef struct {
    uint8_t pin; uint8_t channel; int avg_read_raw; int avg_read_mvolts;
} adc_continuous_result_t;
```

## Timer（`docs/en/api/timer.rst`）

```cpp
hw_timer_t * timerBegin(uint32_t frequency);       // Hz，自动启动
void timerEnd(hw_timer_t *t);
void timerStart(hw_timer_t *t); void timerStop(hw_timer_t *t); void timerRestart(hw_timer_t *t);
void timerWrite(hw_timer_t *t, uint64_t val);
uint64_t timerRead(hw_timer_t *t); uint64_t timerReadMicros(hw_timer_t *t);
uint64_t timerReadMillis(hw_timer_t *t); double timerReadSeconds(hw_timer_t *t);
uint16_t timerGetFrequency(hw_timer_t *t);
void timerAttachInterrupt(hw_timer_t *t, void (*fn)(void));
void timerAttachInterruptArg(hw_timer_t *t, void (*fn)(void*), void *arg);
void timerDetachInterrupt(hw_timer_t *t);
void timerAlarm(hw_timer_t *t, uint64_t alarm_value, bool autoreload, uint64_t reload_count); // reload_count=0 无限
```

## Preferences / NVS（`docs/en/api/preferences.rst`）

```cpp
bool begin(const char *name, bool readOnly=false, const char* partition_label=NULL);
void end(); bool clear(); bool remove(const char *key); bool isKey(const char *key);
PreferenceType getType(const char *key); size_t freeEntries();
// put: putBool/putChar/putUChar/putShort/putUShort/putInt/putUInt/putLong/putULong/
//      putLong64/putULong64/putFloat/putDouble/putString/putBytes
// get:  对应 get...，含默认值参数（float/double 默认 NAN，bool 默认 false）
size_t getString(const char *key, char *value, size_t maxLen);
String getString(const char *key, String defaultValue=String());
size_t getStringLength(const char *key);
size_t getBytes(const char *key, void *buf, size_t maxLen); size_t getBytesLength(const char *key);
// PreferenceType 枚举: PT_I8 PT_U8 PT_I16 PT_U16 PT_I32 PT_U32 PT_I64 PT_U64 PT_STR PT_BLOB PT_INVALID
```

## Wi-Fi（`docs/en/api/wifi.rst`）

```cpp
// 通用
void mode(wifi_mode_t); wifi_mode_t getMode();
bool setHostname(const char *); const char *getHostname();
wifi_event_id_t onEvent(WiFiEventCb, arduino_event_id_t=ARDUINO_EVENT_MAX);
wifi_event_id_t onEvent(WiFiEventSysCb, ...); wifi_event_id_t onEvent(WiFiEventFuncCb, ...);
void removeEvent(...);
static void useStaticBuffers(bool); bool setDualAntennaConfig(uint8_t a1,uint8_t a2,wifi_rx_ant_t,wifi_tx_ant_t);

// STA
wl_status_t begin(const char* ssid, const char* passphrase=NULL, int32_t channel=0,
                  const uint8_t* bssid=NULL, bool tryConnect=true);
wl_status_t begin();  // 用已 config() 的参数启动
bool connect(...);    // 同 begin 参数
bool config(IPAddress local_ip, IPAddress gateway, IPAddress subnet, IPAddress dns1=0, IPAddress dns2=0);
bool reconnect(); bool disconnect(bool wifioff=false, bool eraseap=false);
bool isConnected(); bool setAutoReconnect(bool); bool getAutoReconnect();
bool setMinSecurity(wifi_auth_mode_t);     // 默认 WIFI_AUTH_WPA2_PSK
IPAddress localIP(); IPAddress subnetMask(); IPAddress gatewayIP(); IPAddress dnsIP(uint8_t=0);
String SSID(); wifi_auth_mode_t encryptionType(); int32_t RSSI(); uint8_t * BSSID(); int8_t channel();

// AP
bool softAP(const char* ssid, const char* passphrase=NULL, int channel=1, int ssid_hidden=0,
            int max_connection=4, bool ftm_responder=false);
bool softAPConfig(IPAddress local_ip, IPAddress gateway, IPAddress subnet);
bool softAPdisconnect(bool wifioff=false);
uint8_t softAPgetStationNum(); IPAddress softAPIP(); String softAPSSID();

// Scan
int16_t scanNetworks(bool async=false, bool show_hidden=false, bool passive=false,
                     uint32_t max_ms_per_chan=300, uint8_t channel=0);
int16_t scanComplete(); void scanDelete();
bool getNetworkInfo(uint8_t item, String &ssid, uint8_t &enc, int32_t &rssi, uint8_t* &bssid, int32_t &channel);

// WiFiMulti
bool addAP(const char *ssid, const char *passphrase=NULL);
uint8_t run(uint32_t connectTimeout=5000);
```

> 事件类型前缀 `ARDUINO_EVENT_*`（如 `ARDUINO_EVENT_WIFI_STA_GOT_IP` / `_STA_DISCONNECTED` / `_STA_CONNECTED`），
> 详见 `libraries/WiFi/src/WiFiGeneric.h`。

## Network（3.x 统一网络库 `libraries/Network/`）

```cpp
#include <Network.h>
NetworkClient client;          // TCP 客户端（WiFiClient 仍兼容别名）
NetworkServer server(port);    // TCP 服务端
NetworkUDP udp;                // UDP
// 服务端：NetworkServer::accept() 推荐替代 WiFiServer::available()
// WiFiClient::flush() 不再清接收缓冲；用 clear()（3.x）
```

## BLE（`libraries/BLE`，`docs/en/api/ble.rst` + `libraries/BLE/README.md`）

```cpp
#include <BLEDevice.h>   // 自动拉入 BLEUtils/BLEServer/BLEScan/BLEAdvertising
// ESP32=Bluedroid; C3/C5/C6/H2/S3=NimBLE; 用 getBLEStackString() 判断

// BLEDevice（静态单例）
static bool init(String deviceName = "");
static BLEServer *createServer(); static BLEClient *createClient();
static BLEScan *getScan(); static BLEAdvertising *getAdvertising();
static void startAdvertising(); static void stopAdvertising();
static esp_err_t setMTU(uint16_t mtu); static uint16_t getMTU();
static BLEAddress getAddress(); static String getDeviceName();
static BLEStack getBLEStack(); static String getBLEStackString();
static bool isHostedBLE();
static void setPower(esp_power_level_t, esp_ble_power_type_t = ESP_BLE_PWR_TYPE_DEFAULT);
static void deinit(bool release_memory = false);
static bool setOwnAddrType(uint8_t type); static bool setOwnAddr(uint8_t *addr);
static void setSecurityCallbacks(BLESecurityCallbacks *pCallbacks);

// BLEScan
BLEScan *getScan(); void setAdvertisedDeviceCallbacks(BLEAdvertisedDeviceCallbacks*);
void setActiveScan(bool); void setInterval(int); void setWindow(int);
BLEScanResults *start(int seconds, bool async=false); int16_t stop();
BLEScanResults *getResults(); void clearResults();

// BLEServer / BLEService / BLECharacteristic
BLEService *createService(const char* uuid);
BLECharacteristic *createCharacteristic(const char* uuid, uint32_t properties);
void setCallbacks(BLEServerCallbacks*); void advertiseOnDisconnect(bool);
void start(); void startAdvertising();
void setValue(const char*); void setValue(uint8_t*, size_t);
void notify(); bool indicate(); void addDescriptor(BLEDescriptor*);
// properties（按位或）: PROPERTY_READ PROPERTY_WRITE PROPERTY_WRITE_NR
//   PROPERTY_NOTIFY PROPERTY_INDICATE PROPERTY_BROADCAST
//   (NimBLE 专属) PROPERTY_READ_ENC/_WRITE_ENC/_READ_AUTHEN/_WRITE_AUTHEN/_READ_AUTHOR/_WRITE_AUTHOR

// BLEClient / BLERemoteService / BLERemoteCharacteristic
bool connect(BLEAdvertisedDevice* / BLEAddress); void disconnect();
bool setMTU(uint16_t); BLERemoteService *getService(BLEUUID);
BLERemoteCharacteristic *getCharacteristic(BLEUUID);
String readValue(); bool canRead(); bool canNotify();
bool registerForNotify(void (*cb)(BLERemoteCharacteristic*, uint8_t*, size_t, bool));
bool writeValue(const char*, size_t);

// 安全（BLESecurity）
void setCapability(esp_ble_io_cap_t);           // ESP_IO_CAP_NONE/_OUT/_IN/_IO/_KBDISP
void setAuthenticationMode(esp_ble_auth_req_t); // ESP_LE_AUTH_NO_BOND/_BOND/_REQ_MITM/_REQ_SC_MITM_BOND
void setPassKey(bool static_key, uint32_t passkey);

// 信标
BLEBeacon::setManufacturerId(uint16_t); setMajor(uint16_t); setMinor(uint16_t);
  setSignalPower(int8_t); setProximityUUID(BLEUUID); String getData();
BLEEddystoneURL(&device); BLEEddystoneTLM(&device);  // 见 Beacon_Scanner
```

## WebServer（`libraries/WebServer`，`libraries/WebServer/src/WebServer.h`）

```cpp
#include <WebServer.h>   // 自动拉入 Network.h
WebServer server(int port = 80);

void begin(); void begin(uint16_t port); void handleClient(); void close(); void stop();
// 路由：on(uri, fn) / on(uri, method, fn) / on(uri, method, fn, uploadFn)
RequestHandler &on(const Uri &uri, HTTPMethod method, THandlerFunction fn, THandlerFunction ufn);
bool removeRoute(const char *uri, HTTPMethod method);
void serveStatic(const char *uri, fs::FS &fs, const char *path, const char *cache_header = NULL);
void addHandler(RequestHandler *handler); bool removeHandler(RequestHandler *handler);
void onNotFound(THandlerFunction fn); void onFileUpload(THandlerFunction ufn);
// 中间件（3.x）
WebServer &addMiddleware(Middleware *); WebServer &addMiddleware(Middleware::Function fn);

// 请求
String uri(); HTTPMethod method(); NetworkClient &client();
int args(); bool hasArg(const String&); String arg(const String&); String arg(int); String argName(int);
String pathArg(unsigned int i);
void collectHeaders(const char *keys[], size_t count); void collectAllHeaders();
int headers(); String header(const String&); String headerName(int); bool hasHeader(const String&);
int clientContentLength() const;

// 响应
void send(int code, const char *content_type, const String &content);
void send_P(int code, PGM_P type, PGM_P content, size_t len = 0);
void sendHeader(const String &name, const String &value, bool first = false);
void setContentLength(size_t); void sendContent(const String&);
template<typename T> size_t streamFile(T &file, const String &contentType, int code = 200);
void chunkResponseBegin(const char *contentType); void chunkWrite(const char*, size_t); void chunkResponseEnd();

// 上传
HTTPUpload &upload();   // status: UPLOAD_FILE_START/_WRITE/_END/_ABORTED
HTTPRaw &raw();         // status: RAW_START/_WRITE/_END/_ABORTED

// 认证
bool authenticate(const char *user, const char *pass); bool authenticate(THandlerFunctionAuthCheck fn);
bool authenticateBasicSHA1(const char *user, const char *sha1Base64);
void requestAuthentication(HTTPAuthMethod mode = BASIC_AUTH, const char *realm = NULL, const String &failMsg = String(""));
// HTTPAuthMethod: BASIC_AUTH / DIGEST_AUTH / OTHER_AUTH

// 杂项
void enableCORS(bool = true); void enableCrossOrigin(bool = true);
void enableETag(bool, ETagFunction fn = nullptr); void enableDelay(bool);
static String urlDecode(const String&); static String responseCodeToString(int);
// 常量: HTTP_UPLOAD_BUFLEN=1436  HTTP_DOWNLOAD_UNIT_SIZE=1436
//       HTTP_MAX_DATA_WAIT / HTTP_MAX_POST_WAIT / HTTP_MAX_SEND_WAIT / HTTP_MAX_CLOSE_WAIT = 5000ms
```

## OTA — Update / ArduinoOTA / HTTPUpdate（`libraries/Update`、`ArduinoOTA`、`HTTPUpdate`）

```cpp
#include <Update.h>          // 底层流式 API
// 错误码: UPDATE_ERROR_OK(0) _WRITE(1) _ERASE(2) _READ(3) _SPACE(4) _SIZE(5) _STREAM(6)
//         _MD5(7) _MAGIC_BYTE(8) _ACTIVATE(9) _NO_PARTITION(10) _BAD_ARGUMENT(11) _ABORT(12) _DECRYPT(13) _SIGN(14)
// 目标: U_FLASH(0) U_FLASHFS(100) U_SPIFFS(101) U_FATFS(102) U_LITTLEFS(103) U_AUTH(200)
//   UPDATE_SIZE_UNKNOWN = 0xFFFFFFFF

bool begin(size_t size = UPDATE_SIZE_UNKNOWN, int command = U_FLASH, int ledPin = -1, uint8_t ledOn = LOW, const char *label = NULL);
size_t write(uint8_t *data, size_t len); size_t writeStream(Stream &data);
template<typename T> size_t write(T &data);  // 用 available()+read()，比 writeStream 快
bool end(bool evenIfRemaining = false); void abort();
UpdateClass &onProgress(void fn(size_t, size_t));
bool setMD5(const char *hex); String md5String(); void md5(uint8_t *out);
uint8_t getError(); const char *errorString(); void printError(Print&);
bool hasError(); bool isRunning(); bool isFinished(); size_t size(); size_t progress(); size_t remaining();
bool canRollBack(); bool rollBack();
// AES 解密（加密镜像）:
bool setupCrypt(const uint8_t *key = 0, size_t addr = 0, uint8_t cfg = 0xf, int mode = U_AES_DECRYPT_AUTO);
bool setCryptKey(const uint8_t *key); bool setCryptMode(int); void setCryptAddress(size_t); void setCryptConfig(uint8_t);
// U_AES_DECRYPT_NONE(0) _AUTO(1) _ON(2)

#include <ArduinoOTA.h>     // IDE / espota.py 推送（默认端口 3232，依赖 mDNS）
ArduinoOTA.setPort(uint16_t); .setHostname(const char*); .setPassword(const char*); .setPasswordHash(const char*);
  .setPartitionLabel(const char*); .setRebootOnSuccess(bool); .setMdnsEnabled(bool);
  .onStart(fn); .onEnd(fn); .onProgress(fn(unsigned,unsigned)); .onError(fn(ota_error_t));
  .setSignature(UpdaterVerifyClass *); .setUpdaterInstance(UpdateClass *);
void begin(); void end(); void handle(); int getCommand();  // U_FLASH / U_SPIFFS
// ota_error_t: OTA_AUTH_ERROR OTA_BEGIN_ERROR OTA_CONNECT_ERROR OTA_RECEIVE_ERROR OTA_END_ERROR

#include <HTTPUpdate.h>     // HTTP(S) 拉取
extern HTTPUpdate httpUpdate;
t_httpUpdate_return update(NetworkClient &c, const String &url, const String &curVer = "", HTTPUpdateRequestCB = NULL);
t_httpUpdate_return update(NetworkClient &c, const String &host, uint16_t port, const String &uri = "/", ...);
t_httpUpdate_return updateSpiffs(...); updateFatfs(...); updateLittlefs(...); updateFs(...);  // + HTTPClient& 重载
void rebootOnUpdate(bool); void setFollowRedirects(followRedirects_t); void setLedPin(int, uint8_t);
void setMD5sum(const String&); void setAuthorization(const String& user, const String& pass); setAuthorization(const String& token);
void onStart(cb); onEnd(cb); onError(cb(int)); onProgress(cb(int,int));
int getLastError(); String getLastErrorString();
// HTTPUpdateResult: HTTP_UPDATE_FAILED HTTP_UPDATE_NO_UPDATES HTTP_UPDATE_OK
// HTTP 错误(-100..-108): HTTP_UE_TOO_LESS_SPACE _SERVER_NOT_REPORT_SIZE _SERVER_FILE_NOT_FOUND
//   _SERVER_FORBIDDEN _SERVER_WRONG_HTTP_CODE _SERVER_FAULTY_MD5 _BIN_VERIFY_HEADER_FAILED _BIN_FOR_WRONG_FLASH _NO_PARTITION
```

## Zigbee（`libraries/Zigbee`，`docs/en/zigbee/*.rst`；仅 C6/C5/H2 原生）

```cpp
#include "Zigbee.h"   // 单例 Zigbee (ZigbeeCore) + ZigbeeEP 子类
// zigbee_role_t: ZIGBEE_COORDINATOR(0) ZIGBEE_ROUTER(1) ZIGBEE_END_DEVICE(2)
// ZIGBEE_DEFAULT_ED_CONFIG() / ZIGBEE_DEFAULT_UART_RCP_RADIO_CONFIG()

bool Zigbee.begin(zigbee_role_t role = ZIGBEE_END_DEVICE, bool erase_nvs = false);
bool Zigbee.begin(esp_zb_cfg_t *role_cfg, bool erase_nvs = false);
void start(); void stop(); bool started(); bool connected(); zigbee_role_t getRole();
bool addEndpoint(ZigbeeEP *ep);
void setPrimaryChannelMask(uint32_t); void setScanDuration(uint8_t); uint8_t getScanDuration();
void setRxOnWhenIdle(bool); bool getRxOnWhenIdle(); void setTimeout(uint32_t ms);
void scanNetworks(uint32_t channel_mask = ESP_ZB_TRANSCEIVER_ALL_CHANNELS_MASK, uint8_t scan_duration = 5);
  int16_t scanComplete();   // -2 fail / -1 running / 0 none / >0 count
  zigbee_scan_result_t *getScanResult(); void scanDelete();
void openNetwork(uint8_t seconds); void closeNetwork(); void setRebootOpenNetwork(uint8_t seconds);
void setRadioConfig(esp_zb_radio_config_t); esp_zb_radio_config_t getRadioConfig();
void setHostConfig(esp_zb_host_config_t); esp_zb_host_config_t getHostConfig();
void setDebugMode(bool); void factoryReset(bool restart = true);
void onGlobalDefaultResponse(void (*cb)(zb_cmd_type_t, esp_zb_zcl_status_t, uint8_t, uint16_t));
static const char *formatIEEEAddress(const esp_zb_ieee_addr_t); static const char *formatShortAddress(uint16_t);

// ZigbeeEP（基类，所有具体端点继承）
bool setManufacturerAndModel(const char *name, const char *model);  // 各 ≤32 字符
void setVersion(uint8_t); void setHardwareVersion(uint8_t); uint8_t getEndpoint();   // 必须在 begin() 前
bool setPowerSource(uint8_t source, uint8_t percentage = 0xff, uint8_t voltage = 0xff);  // ZB_POWER_SOURCE_MAINS/_BATTERY
bool setBatteryPercentage(uint8_t); bool setBatteryVoltage(uint8_t); bool reportBatteryPercentage();
bool addTimeCluster(tm time = {}, int32_t gmt_offset = 0);
  bool setTime(tm); bool setTimezone(int32_t); tm getTime(...); int32_t getTimezone(...);
bool addOTAClient(uint32_t file_ver, uint32_t dl_ver, uint16_t hw_ver, uint16_t mfr = 0x1001, uint16_t img = 0x1011, uint8_t max_data = 223);
  void requestOTAUpdate();
bool bound(); std::vector<esp_zb_binding_info_t> getBoundDevices(); std::list<zb_device_params_t*> getBoundDevices();
  void printBoundDevices(Print & = Serial); void allowMultipleBinding(bool); void setManualBinding(bool); void clearBoundDevices();
char *readManufacturer(uint8_t ep, uint16_t short_addr, esp_zb_ieee_addr_t); char *readModel(...);
void onIdentify(void (*cb)(uint16_t)); void onDefaultResponse(void (*cb)(zb_cmd_type_t, esp_zb_zcl_status_t));
// 具体端点类: ZigbeeLight ZigbeeSwitch ZigbeeTempSensor ZigbeeGateway ZigbeeDimmableLight
//   ZigbeeColorDimmerLight ZigbeeWindowCovering ...
```

## Matter（`libraries/Matter`，`docs/en/matter/*.rst`；需 Huge APP 分区 + 擦全 flash）

```cpp
#include <Matter.h>   // 单例 Matter + MatterEndPoint 子类

bool Matter.begin();   // 必须在所有 endpoint.begin() 之后
bool isDeviceCommissioned(); bool isDeviceConnected();
bool isWiFiConnected(); bool isThreadConnected();
bool isWiFiStationEnabled(); bool isWiFiAccessPointEnabled(); bool isThreadEnabled(); bool isBLECommissioningEnabled();
void decommission();   // 出厂复位（擦 Matter 凭据）
String getManualPairingCode(); String getOnboardingQRCodeUrl();
void onEvent(void (*cb)(matterEvent_t, const chip::DeviceLayer::ChipDeviceEvent *));
// 事件: MATTER_COMMISSIONING_COMPLETE MATTER_WIFI_CONNECTIVITY_CHANGE ...

// MatterEndPoint（基类）+ 具体子类
// 灯: MatterOnOffLight MatterDimmableLight MatterColorTemperatureLight MatterColorLight(HSV) MatterEnhancedColorLight
// 传感器: MatterTemperatureSensor MatterHumiditySensor MatterPressureSensor MatterContactSensor
//   MatterWaterLeakDetector MatterOccupancySensor MatterLightSensor MatterRainSensor ...
// 控制: MatterFan MatterThermostat MatterOnOffPlugin MatterDimmablePlugin
//   MatterGenericSwitch(智能按钮) MatterWindowCovering(窗帘)
ep.begin(); ep.begin(bool initialState);   // 部分端点支持初始状态
ep.onChange(bool (*cb)(bool state));       // 灯/插座回调（返回 true 表示成功）
ep.getOnOff(); ep.setOnOff(bool); ep.toggle(); ep.updateAccessory();   // 同步本地状态到网络
```

## ESP-NOW（`docs/en/api/espnow.rst`，`libraries/ESP_NOW`）

```cpp
// ESP_NOW_Class
bool begin(const uint8_t *pmk=NULL); bool end();
int getTotalPeerCount(); int getEncryptedPeerCount();
int getVersion(); int getMaxDataLen();
const uint8_t *BROADCAST_ADDR;
void onNewPeer(void (*cb)(const esp_now_recv_info_t*, const uint8_t*, int, void*), void *arg);

// ESP_NOW_Peer（抽象，需继承）
ESP_NOW_Peer(const uint8_t *mac_addr, uint8_t channel, wifi_interface_t iface, const uint8_t *lmk);
bool add(); bool remove(); size_t send(const uint8_t *data, int len);
const uint8_t *addr() const; void addr(const uint8_t*);
uint8_t getChannel() const; void setChannel(uint8_t);
wifi_interface_t getInterface() const; void setInterface(wifi_interface_t);
bool isEncrypted() const; void setKey(const uint8_t *lmk);
virtual void onReceive(const uint8_t *data, int len, bool broadcast);
virtual void onSent(bool success);
```

## 深睡眠（ESP-IDF `<esp_sleep.h>`，Arduino 可直接用）

```cpp
esp_sleep_enable_timer_wakeup(uint64_t time_in_us);
esp_sleep_enable_ext0_wakeup(gpio_num_t gpio, int level);        // 仅 ESP32
esp_sleep_enable_ext1_wakeup(uint64_t mask, esp_sleep_ext1_wakeup_mode_t mode);
esp_sleep_enable_touchpad_wakeup();
void esp_deep_sleep_start();                                      // 不返回
esp_sleep_wakeup_cause_t esp_sleep_get_wakeup_cause();
touch_pad_t esp_sleep_get_touchpad_wakeup_status();
// RTC 内存: RTC_DATA_ATTR <type> var;
```

## FreeRTOS（原生，`<freertos/*.h>`）

```cpp
BaseType_t xTaskCreate(TaskFunction_t, const char*, uint32_t stack, void*, UBaseType_t prio, TaskHandle_t*);
BaseType_t xTaskCreatePinnedToCore(..., BaseType_t core);
void vTaskDelete(TaskHandle_t); void vTaskDelay(const TickType_t);
QueueHandle_t xQueueCreate(UBaseType_t count, UBaseType_t size);
SemaphoreHandle_t xSemaphoreCreateMutex(); / xSemaphoreCreateBinary(); / xSemaphoreCreateCounting(max,init);
```
