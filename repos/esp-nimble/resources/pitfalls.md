# NimBLE 常见陷阱汇总

> 所有陷阱均对应仓库示例与头文件中的真实行为。每条给出错误写法与正确写法。

## 1. Host 未同步就发起 GAP 过程

所有 `ble_gap_adv_start` / `ble_gap_disc` / `ble_gap_connect` 必须在 `ble_hs_cfg.sync_cb` 被回调（Host 已与 Controller 同步）之后调用。
- 错误：在 `main()` 中 init 完直接 `ble_gap_adv_start`。
- 正确：在 `on_sync` 中调用，并用 `ble_hs_synced()` 辅助判断。

## 2. 连接句柄未保存

`BLE_GAP_EVENT_CONNECT` 中必须保存 `event->connect.conn_handle`；`BLE_GAP_EVENT_DISCONNECT` 中复位为 `BLE_HS_CONN_HANDLE_NONE`。后续 notify / terminate / update 全部依赖该句柄。

## 3. 广播数据 > 31 字节

`ble_gap_adv_set_fields` 返回 `BLE_HS_EMSGSIZE`。长设备名应放入 scan response（`ble_gap_adv_rsp_set_fields`），或使用扩展广播（`ble_gap_ext_adv_set_data`，最大 1650 字节）。

## 4. 连接前未取消扫描

`ble_gap_connect` 在扫描进行中返回 `BLE_HS_EBUSY`。必须先 `ble_gap_disc_cancel()`（见 `apps/blecent/src/main.c` 的 `blecent_connect_if_interesting`）。

## 5. notify 数据未转 mbuf

`ble_gatts_notify_custom` 第三参为 `struct os_mbuf *`，不能传裸指针。必须 `ble_hs_mbuf_from_flat()` 转换（见 `apps/blehr/src/main.c`）。

## 6. GATT 服务表未 count / 未以 {0} 结尾

注册顺序必须是 `ble_gatts_count_cfg(svcs)` → `ble_gatts_add_svcs(svcs)`，且服务表 / 特征表 / 描述符表三层都以 `{ 0 }` 结尾。跳过 `count_cfg` 会导致 ATT 句柄分配错误。

## 7. access_cb 用错 ctxt 字段

读 / 写特征用 `ctxt->chr->uuid`，读 / 写描述符用 `ctxt->dsc->uuid`，必须按 `ctxt->op`（`BLE_GATT_ACCESS_OP_READ_CHR` 等）分支判断，否则描述符访问时解引用空指针。

## 8. own_addr_type 与地址设置不匹配

设置了 NRPA / 静态随机地址（`ble_hs_id_set_rnd`）后，GAP API 的 `own_addr_type` 必须为 `BLE_OWN_ADDR_RANDOM`，不能写 `BLE_OWN_ADDR_PUBLIC`（见 `apps/scanner`、`apps/advertiser`）。推荐用 `ble_hs_id_infer_auto` 自动推断。

## 9. 配对 PASSKEY 事件未响应

`BLE_GAP_EVENT_PASSKEY_ACTION` 到达后，必须按 `params.action`（`BLE_SM_IOACT_INPUT`/`_DISP`/`_NUMCMP`）调用 `ble_sm_inject_io(conn_handle, &io)`，否则配对超时失败。Just Works（`BLE_SM_IOACT_NONE`）无需注入。

## 10. 扩展广播用了 legacy API

扩展广播实例的数据必须用 `ble_gap_ext_adv_set_data(instance, mbuf)`，不能混用 `ble_gap_adv_set_fields`。扩展广播数据走 mbuf（`os_msys_get_pkthdr`）。

## 11. 重复注册 GATT 服务

`ble_gatts_add_svcs` 应在初始化阶段调用一次。若在每次 `on_sync` 中重复 add，会导致句柄重复 / 报错。

## 12. 误用 BLE_HS_FOREVER

`BLE_HS_FOREVER`（INT32_MAX）作为 `duration_ms` 表示"持续到被显式停止"，广播不会自动结束。需要限时广播时传有限毫秒数，结束会收到 `BLE_GAP_EVENT_ADV_COMPLETE`。

## 13. 在 `ble_hs_cfg.sm_io_cap` 上设错能力

IO 能力决定配对模型。Just Works 用 `BLE_HS_IO_NO_INPUT_OUTPUT`；需要 MITM（Passkey / Numeric Comparison）必须设 `sm_mitm=1` 且能力匹配（如键盘/显示器）。

## 14. GATT 客户端 CCCD 写值错误

订阅通知写 `{0x01, 0x00}`，订阅指示写 `{0x02, 0x00}`，取消写 `{0x00, 0x00}`。写错值客户端收不到 notify（见 `apps/blecent` 的 CCCD 写入）。

## 15. Mesh 缺少 Configuration Server

Mesh root element 必须包含 `BT_MESH_MODEL_CFG_SRV`，否则无法被 provision。`bt_mesh_init` 必须在 Host `on_sync` 之后调用。
