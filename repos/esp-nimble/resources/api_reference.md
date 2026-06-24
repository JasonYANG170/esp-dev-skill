# NimBLE Host API 快速参考

> 所有签名均取自仓库 `nimble/host/include/host/` 头文件。未列出的 API 即视为本 Skill 不推荐使用。

## 通用宏与返回码（`ble_hs.h`）

```c
#define BLE_HS_FOREVER              INT32_MAX   /* 无限等待 */
#define BLE_HS_CONN_HANDLE_NONE     0xffff      /* 无效连接句柄 */

/* Host 返回码（节选） */
#define BLE_HS_EBUSY                11
#define BLE_HS_ENOADDR              13
#define BLE_HS_ECONTROLLER          16
#define BLE_HS_EMSGSIZE             18
#define BLE_HS_EINVAL               30
#define BLE_HS_ENOTCONFIGURED       34
```

## ble_hs 初始化与状态

```c
// 头文件: host/ble_hs.h
struct ble_hs_cfg {
    ble_gatt_register_fn *gatts_register_cb;
    // ... SM、存储、连接配置 ...
    ble_hs_reset_fn *reset_cb;
    ble_hs_sync_fn *sync_cb;
    ble_store_status_fn *store_status_cb;
};
extern struct ble_hs_cfg ble_hs_cfg;

int  ble_hs_is_enabled(void);
int  ble_hs_synced(void);
int  ble_hs_start(void);
void ble_hs_init(void);
int  ble_hs_shutdown(int reason);
uint8_t ble_hs_get_enabled_state(void);
```

## mbuf 工具（`ble_hs.h` / `ble_hs_mbuf.h`）

```c
struct os_mbuf *ble_hs_mbuf_from_flat(const void *buf, int len);
int  ble_hs_mbuf_to_flat(struct os_mbuf *om, void *flat, int max_len, int *out_len);
```

## 地址管理（`ble_hs_id.h`）

```c
int ble_hs_id_gen_rnd(int nrpa, ble_addr_t *out_addr);
int ble_hs_id_set_rnd(const uint8_t *rnd_addr);
int ble_hs_id_copy_addr(uint8_t id_addr_type, uint8_t *out_id_addr, int *out_is_nrpa);
int ble_hs_id_infer_auto(int privacy, uint8_t *out_addr_type);

// util.h
int ble_hs_util_ensure_addr(int prefer_random);
```

## GAP（`ble_gap.h`）

### 广播模式常量
```c
#define BLE_GAP_CONN_MODE_NON   0
#define BLE_GAP_CONN_MODE_DIR   1
#define BLE_GAP_CONN_MODE_UND   2

#define BLE_GAP_DISC_MODE_NON   0
#define BLE_GAP_DISC_MODE_LTD   1
#define BLE_GAP_DISC_MODE_GEN   2

#define BLE_GAP_ROLE_MASTER     0
#define BLE_GAP_ROLE_SLAVE      1
```

### 本机地址类型
```c
#define BLE_OWN_ADDR_PUBLIC              0
#define BLE_OWN_ADDR_RANDOM              1
#define BLE_OWN_ADDR_RPA_PUBLIC_DEFAULT  2
#define BLE_OWN_ADDR_RPA_RANDOM_DEFAULT  3
```

### GAP 事件类型（`event->type`）
```c
BLE_GAP_EVENT_CONNECT, _DISCONNECT, _CONN_UPDATE, _CONN_UPDATE_REQ,
_L2CAP_UPDATE_REQ, _TERM_FAILURE, _DISC, _DISC_COMPLETE, _ADV_COMPLETE,
_ENC_CHANGE, _PASSKEY_ACTION, _NOTIFY_RX, _NOTIFY_TX, _SUBSCRIBE, _MTU,
_IDENTITY_RESOLVED, _REPEAT_PAIRING, _PHY_UPDATE_COMPLETE, _EXT_DISC,
_PERIODIC_SYNC, _PERIODIC_REPORT, _PERIODIC_SYNC_LOST, _SCAN_REQ_RCVD, ...
```

### 事件订阅原因
```c
#define BLE_GAP_SUBSCRIBE_REASON_WRITE    1
#define BLE_GAP_SUBSCRIBE_REASON_TERM     2
#define BLE_GAP_SUBSCRIBE_REASON_RESTORE  3
#define BLE_GAP_REPEAT_PAIRING_RETRY      1
#define BLE_GAP_REPEAT_PAIRING_IGNORE     2
```

### 关键结构体
```c
struct ble_gap_sec_state {
    unsigned encrypted:1;
    unsigned authenticated:1;
    unsigned bonded:1;
    unsigned key_size:5;
    unsigned authorize:1;
};

struct ble_gap_conn_desc {
    struct ble_gap_sec_state sec_state;
    ble_addr_t our_id_addr, peer_id_addr, our_ota_addr, peer_ota_addr;
    uint16_t conn_handle, conn_itvl, conn_latency, supervision_timeout;
    uint8_t  role, master_clock_accuracy;
};

struct ble_gap_conn_params {
    uint16_t scan_itvl, scan_window;
    uint16_t itvl_min, itvl_max;        /* 1.25ms 单位 */
    uint16_t latency;
    uint16_t supervision_timeout;        /* 10ms 单位 */
    uint16_t min_ce_len, max_ce_len;
};

struct ble_gap_adv_params {
    uint8_t conn_mode, disc_mode;
    uint16_t itvl_min, itvl_max;
    uint8_t channel_map, filter_policy;
    uint8_t high_duty_cycle:1;
};

struct ble_gap_disc_params {
    uint8_t filter_policy, limited, passive, filter_duplicates;
    uint16_t itvl, window;               /* 0.625ms 单位 */
};

struct ble_gap_ext_adv_params {
    // ... own_addr_type, primary_phy, secondary_phy, tx_power, sid,
    //     scannable, connectable, legacy_pdu, scan_req_notif, itvl_min/max ...
};

struct ble_gap_periodic_adv_params {
    // ... itvl_min, itvl_max, include_tx_power ...
};
```

### GAP 函数
```c
// 连接查询
int ble_gap_conn_find(uint16_t handle, struct ble_gap_conn_desc *out_desc);
int ble_gap_conn_find_by_addr(const ble_addr_t *addr, struct ble_gap_conn_desc *out_desc);
int ble_gap_set_event_cb(uint16_t conn_handle, ble_gap_event_fn *cb, void *cb_arg);

// 传统广播
int ble_gap_adv_start(uint8_t own_addr_type, const ble_addr_t *direct_addr,
                      int32_t duration_ms, const struct ble_gap_adv_params *params,
                      ble_gap_event_fn *cb, void *cb_arg);
int ble_gap_adv_stop(void);
int ble_gap_adv_active(void);
int ble_gap_adv_set_data(const uint8_t *data, int data_len);
int ble_gap_adv_rsp_set_data(const uint8_t *data, int data_len);
int ble_gap_adv_set_fields(const struct ble_hs_adv_fields *fields);
int ble_gap_adv_rsp_set_fields(const struct ble_hs_adv_fields *fields);

// 扩展广播
int ble_gap_ext_adv_configure(uint8_t instance, struct ble_gap_ext_adv_params *params,
                              ble_gap_event_fn *gaf_cb, void *cb_arg);
int ble_gap_ext_adv_set_addr(uint8_t instance, const ble_addr_t *addr);
int ble_gap_ext_adv_start(uint8_t instance, int duration, int max_events);
int ble_gap_ext_adv_stop(uint8_t instance);
int ble_gap_ext_adv_set_data(uint8_t instance, struct os_mbuf *data);
int ble_gap_ext_adv_rsp_set_data(uint8_t instance, struct os_mbuf *data);
int ble_gap_ext_adv_remove(uint8_t instance);
int ble_gap_ext_adv_clear(void);
int ble_gap_ext_adv_active(uint8_t instance);

// 周期广播
int ble_gap_periodic_adv_configure(uint8_t instance, struct ble_gap_periodic_adv_params *params);
int ble_gap_periodic_adv_start(uint8_t instance);
int ble_gap_periodic_adv_stop(uint8_t instance);
int ble_gap_periodic_adv_set_data(uint8_t instance, struct os_mbuf *data);
int ble_gap_periodic_adv_sync_create(const ble_addr_t *addr, uint8_t adv_sid,
                                     ble_gap_event_fn *cb, void *cb_arg);
int ble_gap_periodic_adv_sync_terminate(uint16_t sync_handle);

// 扫描
int ble_gap_disc(uint8_t own_addr_type, int32_t duration_ms,
                 const struct ble_gap_disc_params *disc_params,
                 ble_gap_event_fn *cb, void *cb_arg);
int ble_gap_ext_disc(uint8_t own_addr_type, uint16_t duration, uint16_t period,
                     const struct ble_gap_ext_disc_params *ext_disc_params,
                     const struct ble_gap_disc_params *phy_opts,
                     ble_gap_event_fn *cb, void *cb_arg);
int ble_gap_disc_cancel(void);
int ble_gap_disc_active(void);

// 连接
int ble_gap_connect(uint8_t own_addr_type, const ble_addr_t *peer_addr,
                    int32_t duration_ms, const struct ble_gap_conn_params *params,
                    ble_gap_event_fn *cb, void *cb_arg);
int ble_gap_conn_cancel(void);
int ble_gap_conn_active(void);

// 连接管理
int ble_gap_terminate(uint16_t conn_handle, uint8_t hci_reason);
int ble_gap_update_params(uint16_t conn_handle, const struct ble_gap_upd_params *params);
int ble_gap_set_data_len(uint16_t conn_handle, uint16_t tx_octets, uint16_t tx_time);
int ble_gap_conn_rssi(uint16_t conn_handle, int8_t *out_rssi);

// 安全
int ble_gap_security_initiate(uint16_t conn_handle);
int ble_gap_pair_initiate(uint16_t conn_handle);
int ble_gap_encryption_initiate(uint16_t conn_handle, uint8_t key_size,
                                uint8_t enc, uint8_t mitm, uint8_t auth, uint8_t smp);
int ble_gap_unpair(const ble_addr_t *peer_addr);
int ble_gap_unpair_oldest_peer(void);
int ble_gap_unpair_oldest_except(const ble_addr_t *peer_addr);
int ble_gap_set_priv_mode(const ble_addr_t *peer_addr, uint8_t priv_mode);

// PHY（mask 用 BLE_GAP_LE_PHY_*_MASK；读取/事件返回用 BLE_GAP_LE_PHY_* 枚举）
int ble_gap_read_le_phy(uint16_t conn_handle, uint8_t *tx_phy, uint8_t *rx_phy);
int ble_gap_set_prefered_default_le_phy(uint8_t tx_phys_mask, uint8_t rx_phys_mask);
int ble_gap_set_prefered_le_phy(uint16_t conn_handle, uint8_t tx_phys_mask,
                                uint8_t rx_phys_mask, uint8_t phy_opts);

// 白名单
int ble_gap_wl_set(const ble_addr_t *addrs, uint8_t white_list_count);
```

### 连接参数更新（`ble_gap.h`）

> 注意：`ble_gap_update_params` 用的是 `ble_gap_upd_params`（更新专用），不是 `ble_gap_connect` 用的 `ble_gap_conn_params`（含 scan_itvl/scan_window）。两者 interval/latency/timeout 字段同名但属不同结构体。

```c
/* 更新专用结构体（itvl 单位 1.25ms，supervision_timeout 单位 10ms） */
struct ble_gap_upd_params {
    uint16_t itvl_min, itvl_max;   /* 1.25ms 单位 */
    uint16_t latency;
    uint16_t supervision_timeout;  /* 10ms 单位 */
    uint16_t min_ce_len, max_ce_len; /* 0.625ms 单位；0=让 controller 自选 */
};

/* 事件 union 字段（event->conn_update / event->conn_update_req / event->phy_updated）*/
// BLE_GAP_EVENT_CONN_UPDATE:        event->conn_update.{status, conn_handle}
// BLE_GAP_EVENT_CONN_UPDATE_REQ /
// BLE_GAP_EVENT_L2CAP_UPDATE_REQ:   event->conn_update_req.{peer_params, self_params, conn_handle}
//   回调返回 0=接受，非 0 HCI 错误码=拒绝（self_params 默认已拷自 peer_params）
```

### LE PHY 常量（`ble_gap.h`）

```c
/* 单一 PHY 枚举（read_le_phy / phy_updated 返回值）*/
#define BLE_GAP_LE_PHY_1M       1
#define BLE_GAP_LE_PHY_2M       2
#define BLE_GAP_LE_PHY_CODED    3

/* PHY 位掩码（set_prefered_*_le_phy 的 phys_mask 参数，可按位或）*/
#define BLE_GAP_LE_PHY_1M_MASK      0x01
#define BLE_GAP_LE_PHY_2M_MASK      0x02
#define BLE_GAP_LE_PHY_CODED_MASK   0x04
#define BLE_GAP_LE_PHY_ANY_MASK     0x0F   /* 无偏好 */

/* Coded PHY 编码选项（phy_opts 参数；mask 含 CODED 时必给）*/
#define BLE_GAP_LE_PHY_CODED_ANY    0   /* controller 自选 S=2 或 S=8 */
#define BLE_GAP_LE_PHY_CODED_S2     1   /* 500 kbps，中等距离 */
#define BLE_GAP_LE_PHY_CODED_S8     2   /* 125 kbps，最远距离 */

// BLE_GAP_EVENT_PHY_UPDATE_COMPLETE: event->phy_updated.{status, conn_handle, tx_phy, rx_phy}
```

## 广播数据字段（`ble_hs_adv.h`）

```c
struct ble_hs_adv_fields {
    uint8_t  flags;
    uint8_t *name;             size_t name_len;        uint8_t name_is_complete;
    int16_t  tx_pwr_lvl;       uint8_t tx_pwr_lvl_is_present;
    ble_uuid16_t  *uuids16;    uint8_t num_uuids16;    uint8_t uuids16_is_complete;
    ble_uuid32_t  *uuids32;    uint8_t num_uuids32;    uint8_t uuids32_is_complete;
    ble_uuid128_t *uuids128;   uint8_t num_uuids128;   uint8_t uuids128_is_complete;
    uint8_t *mfg_data;         size_t mfg_data_len;
    uint8_t *svc_data_uuid16;  size_t svc_data_uuid16_len;
    uint8_t *svc_data_uuid32;  size_t svc_data_uuid32_len;
    uint8_t *svc_data_uuid128; size_t svc_data_uuid128_len;
    uint16_t appearance;       uint8_t appearance_is_present;
    uint16_t adv_itvl;         uint8_t adv_itvl_is_present;
    uint8_t *uri;              size_t uri_len;
    uint8_t *public_tgt_addr;  uint8_t num_public_tgt_addrs;
    uint8_t *slave_itvl_range;
};

int ble_hs_adv_parse_fields(struct ble_hs_adv_fields *fields, const uint8_t *data, int data_len);
int ble_hs_adv_set_fields_mbuf(const struct ble_hs_adv_fields *fields, struct os_mbuf *om);

// 广播 flags
#define BLE_HS_ADV_F_DISC_GEN       0x02
#define BLE_HS_ADV_F_DISC_LTD       0x01
#define BLE_HS_ADV_F_BREDR_UNSUP    0x04
#define BLE_HS_ADV_TX_PWR_LVL_AUTO  (-128)

// 广播报告事件类型
#define BLE_HCI_ADV_RPT_EVTYPE_ADV_IND      0x00
#define BLE_HCI_ADV_RPT_EVTYPE_DIR_IND      0x01
#define BLE_HCI_ADV_RPT_EVTYPE_SCAN_IND     0x02
#define BLE_HCI_ADV_RPT_EVTYPE_NONCONN_IND  0x03
#define BLE_HCI_ADV_RPT_EVTYPE_SCAN_RSP     0x04

// PHY
#define BLE_HCI_LE_PHY_1M    1
#define BLE_HCI_LE_PHY_2M    2
#define BLE_HCI_LE_PHY_CODED 3
```

## GATT 通用（`ble_gatt.h`）

### 特征权限标志
```c
#define BLE_GATT_CHR_F_BROADCAST              0x00000001
#define BLE_GATT_CHR_F_READ                   0x00000002
#define BLE_GATT_CHR_F_WRITE_NO_RSP           0x00000004
#define BLE_GATT_CHR_F_WRITE                  0x00000008
#define BLE_GATT_CHR_F_NOTIFY                 0x00000010
#define BLE_GATT_CHR_F_INDICATE               0x00000020
#define BLE_GATT_CHR_F_AUTH_SIGN_WRITE        0x00000040
#define BLE_GATT_CHR_F_RELIABLE_WRITE         0x00000080
#define BLE_GATT_CHR_F_READ_ENC               0x00000200
#define BLE_GATT_CHR_F_READ_AUTHEN            0x00000400
#define BLE_GATT_CHR_F_WRITE_ENC              0x00001000
#define BLE_GATT_CHR_F_WRITE_AUTHEN           0x00002000
#define BLE_GATT_CHR_F_NOTIFY_INDICATE_ENC    0x00008000
#define BLE_GATT_CHR_F_NOTIFY_INDICATE_AUTHEN 0x00010000

#define BLE_GATT_SVC_TYPE_PRIMARY    1
#define BLE_GATT_SVC_TYPE_SECONDARY  2

// CCCD UUID
#define BLE_GATT_DSC_CLT_CFG_UUID16  0x2902
```

### 服务表结构
```c
struct ble_gatt_dsc_def {
    const ble_uuid_t *uuid;
    uint8_t att_flags;
    ble_gatt_access_fn *access_cb;
    uint8_t min_key_size;
};

struct ble_gatt_chr_def {
    const ble_uuid_t *uuid;
    ble_gatt_access_fn *access_cb;
    uint16_t *val_handle;       /* Host 回填特征值句柄 */
    uint32_t flags;
    uint16_t min_key_size;
    struct ble_gatt_dsc_def *descriptors;
};

struct ble_gatt_svc_def {
    uint8_t type;
    const ble_uuid_t *uuid;
    struct ble_gatt_chr_def *characteristics;
    struct ble_gatt_svc_def **includes;
};
```

### 访问上下文
```c
enum ble_gatt_access_op {
    BLE_GATT_ACCESS_OP_READ_CHR,
    BLE_GATT_ACCESS_OP_WRITE_CHR,
    BLE_GATT_ACCESS_OP_READ_DSC,
    BLE_GATT_ACCESS_OP_WRITE_DSC,
};

struct ble_gatt_access_ctxt {
    enum ble_gatt_access_op op;
    const struct ble_gatt_svc_def *svc;
    const struct ble_gatt_chr_def *chr;
    const struct ble_gatt_dsc_def *dsc;
    struct os_mbuf *om;     /* 读：填入返回值；写：从这取数据 */
};
```

### GATT 服务端函数
```c
int ble_gatts_count_cfg(const struct ble_gatt_svc_def *defs);
int ble_gatts_add_svcs(const struct ble_gatt_svc_def *svcs);
int ble_gatts_add_dynamic_svcs(const struct ble_gatt_svc_def *svcs);
int ble_gatts_delete_svc(const ble_uuid_t *uuid);
int ble_gatts_svc_set_visibility(uint16_t handle, int visible);
int ble_gatts_reset(void);
int ble_gatts_start(void);
int ble_gatts_find_svc(const ble_uuid_t *uuid, uint16_t *out_handle);
int ble_gatts_find_chr(const ble_uuid_t *svc_uuid, const ble_uuid_t *chr_uuid,
                       uint16_t *out_dsc_handle, uint16_t *out_val_handle);
int ble_gatts_find_dsc(const ble_uuid_t *svc_uuid, const ble_uuid_t *chr_uuid,
                       const ble_uuid_t *dsc_uuid, uint16_t *out_handle);

// 通知 / 指示
int ble_gatts_notify(uint16_t conn_handle, uint16_t chr_val_handle);
int ble_gatts_notify_custom(uint16_t conn_handle, uint16_t att_handle, struct os_mbuf *om);
int ble_gatts_notify_multiple_custom(uint16_t conn_handle, uint16_t num, struct os_mbuf *om);
int ble_gatts_indicate(uint16_t conn_handle, uint16_t chr_val_handle);
int ble_gatts_indicate_custom(uint16_t conn_handle, uint16_t chr_val_handle, struct os_mbuf *om);
```

### GATT 客户端函数
```c
// 发现
int ble_gattc_disc_all_svcs(uint16_t conn_handle, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_disc_svc_by_uuid(uint16_t conn_handle, const ble_uuid_t *uuid, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_find_inc_svcs(uint16_t conn_handle, uint16_t start_handle, uint16_t end_handle, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_disc_all_chrs(uint16_t conn_handle, uint16_t start_handle, uint16_t end_handle, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_disc_chrs_by_uuid(uint16_t conn_handle, uint16_t start_handle, uint16_t end_handle, const ble_uuid_t *uuid, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_disc_all_dscs(uint16_t conn_handle, uint16_t chr_val_handle, uint16_t end_handle, ble_gatt_mtu_fn *cb, void *cb_arg);

// 读
int ble_gattc_read(uint16_t conn_handle, uint16_t attr_handle, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_read_by_uuid(uint16_t conn_handle, uint16_t start, uint16_t end, const ble_uuid_t *uuid, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_read_long(uint16_t conn_handle, uint16_t handle, uint16_t offset, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_read_mult(uint16_t conn_handle, const uint16_t *handles, int num_handles, ble_gatt_mtu_fn *cb, void *cb_arg);

// 写
int ble_gattc_write(uint16_t conn_handle, uint16_t attr_handle, struct os_mbuf *txom, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_write_flat(uint16_t conn_handle, uint16_t attr_handle, const void *data, int data_len, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_write_long(uint16_t conn_handle, uint16_t attr_handle, uint16_t offset, struct os_mbuf *txom, ble_gatt_mtu_fn *cb, void *cb_arg);
int ble_gattc_write_no_rsp(uint16_t conn_handle, uint16_t attr_handle, struct os_mbuf *txom);
int ble_gattc_write_no_rsp_flat(uint16_t conn_handle, uint16_t attr_handle, const void *data, int data_len);
int ble_gattc_signed_write(uint16_t conn_handle, uint16_t attr_handle, struct os_mbuf *txom);
int ble_gattc_write_reliable(uint16_t conn_handle, struct ble_gatt_attr *attrs, int num_attrs, ble_gatt_mtu_fn *cb, void *cb_arg);

// MTU
int ble_gattc_exchange_mtu(uint16_t conn_handle, ble_gatt_mtu_fn *cb, void *cb_arg);
```

> GATT 客户端回调签名：`typedef int (*ble_gatt_mtu_fn)(uint16_t conn_handle, const struct ble_gatt_error *error, struct ble_gatt_attr *attr, void *arg);`

## SMP（`ble_sm.h`）

```c
struct ble_sm_io {
    uint8_t action;       // BLE_SM_IOACT_*
    union {
        uint32_t passkey;
        uint8_t  numcmp_accept;
        // OOB 数据 ...
    };
};

#define BLE_SM_IOACT_INPUT    2   /* 输入 passkey */
#define BLE_SM_IOACT_DISP     3   /* 显示 passkey */
#define BLE_SM_IOACT_NUMCMP   4   /* 数值比较 */
#define BLE_SM_IOACT_NONE     5   /* Just Works */

int  ble_sm_inject_io(uint16_t conn_handle, struct ble_sm_io *pkey);
int  ble_sm_sc_oob_generate_data(struct ble_sm_sc_oob_data *oob_data);
int  ble_sm_configure_static_passkey(uint32_t passkey, bool enable);
int  ble_sm_get_static_passkey_config(uint32_t *passkey, bool *enabled);
void ble_sm_proc_deinit(void);
```

## 存储与绑定（`ble_store.h`）

```c
int ble_store_util_delete_peer(const ble_addr_t *peer_addr);   // 删除某设备所有数据
int ble_store_util_status_rr(struct ble_store_status_event *event, void *arg); // 回收绑定（轮转）

// 配置在 ble_hs_cfg.store_status_cb
```

## UUID（`ble_uuid.h`）

```c
typedef struct { uint8_t type; uint16_t value; } ble_uuid16_t;
typedef struct { uint8_t type; uint32_t value; } ble_uuid32_t;
typedef struct { uint8_t type; uint8_t value[16]; } ble_uuid128_t;
typedef ble_uuid_t;   // 通用基类

#define BLE_UUID16_INIT(val16)        { .u.type=BLE_UUID_TYPE_16, .value=(val16) }
#define BLE_UUID16_DECLARE(val16)     ((ble_uuid_t *)(&((ble_uuid16_t[]){{BLE_UUID_TYPE_16,(val16)}})))
#define BLE_UUID128_DECLARE(...)      ((ble_uuid_t *)(...))

uint16_t ble_uuid_u16(const ble_uuid_t *uuid);
const char *ble_uuid_to_str(const ble_uuid_t *uuid, char *dst);   // dst 至少 BLE_UUID_STR_LEN
```

## FreeRTOS NPL 端口（`porting/npl/esp-idf`、`porting/npl/freertos`）

```c
// 头文件: nimble/nimble_port_freertos.h
void nimble_port_freertos_init(TaskFunction_t host_task_fn);   // 启动 host 任务（ESP-IDF）
void nimble_port_freertos_deinit(void);
esp_err_t esp_nimble_enable(void *host_task);
esp_err_t esp_nimble_disable(void);
```
> ESP32/ESP32-C3/ESP32-S3/ESP32-C2 使用 `porting/npl/freertos`；ESP32-H4 等新芯片使用 `porting/npl/esp-idf`（见 `porting/npl/esp-idf/README.md`）。
