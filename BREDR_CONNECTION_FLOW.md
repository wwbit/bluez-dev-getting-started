# BlueZ BR/EDR 设备连接流程详细指南

## 目录
1. [入口：dev_connect 的承载选择逻辑](#1-入口dev_connect)
2. [device_connect_profiles 的并发保护与分叉](#2-device_connect_profiles)
3. [SDP 服务发现全流程](#3-sdp-服务发现)
4. [SDP 结果如何变成 btd_service](#4-sdp-结果如何变成-btd_service)
5. [SDP 完成后谁来触发 profile 连接](#5-sdp-完成后谁来触发-profile-连接)
6. [create_pending_list：连接队列的构建与排序](#6-create_pending_list)
7. [connect_next：一次只发一个连接](#7-connect_next)
8. [profile->connect 内部：发起即返回，回调驱动完成](#8-profile-connect-的异步回调链)
9. [device_profile_connected：回调驱动的队列推进](#9-device_profile_connected)
10. [add_depends：依赖管理的完整机制](#10-add_depends)
11. [完整时序图](#11-完整时序图)
12. [附录：关键数据结构与文件索引](#12-附录)

---

## 1. 入口：dev_connect

文件：`src/device.c:2807`，函数签名：

```c
static DBusMessage *dev_connect(DBusConnection *conn, DBusMessage *msg,
                                void *user_data)
```

`user_data` 即 `struct btd_device *dev`。返回 `NULL` = 异步（稍后通过 `g_dbus_send_reply` 回复），非 `NULL` = 同步返回。

### 1.1 整体结构

```c
// 第 1 步：bonding 互斥
if (dev->bonding)
    return btd_error_in_progress(msg);

// 第 2 步：承载选择（三层 if-else）
if (dev->bredr_state.connected) {
    if (dev->bredr_state.svc_resolved &&
        find_service_with_state(dev->services, BTD_SERVICE_STATE_CONNECTED))
        bdaddr_type = dev->bdaddr_type;
    else
        bdaddr_type = BDADDR_BREDR;
} else if (dev->le_state.connected && dev->bredr) {
    bdaddr_type = BDADDR_BREDR;
} else {
    bdaddr_type = select_conn_bearer(dev);
}

// 第 3 步：auth_failures 重置
dev->auth_failures = 0;

// 第 4 步：LE 分支
if (bdaddr_type != BDADDR_BREDR) {
    if (dev->connect)
        return btd_error_in_progress(msg);
    if (dev->le_state.connected)
        return dbus_message_new_method_return(msg);  // 幂等
    btd_device_set_temporary(dev, false);
    ...
    err = device_connect_le(dev);
    ...
    dev->connect = dbus_message_ref(msg);  // ★ 保存引用，进入异步
    return NULL;
}

// 第 5 步：BR/EDR 分支
return device_connect_profiles(dev, bdaddr_type, msg, NULL);
//                                   uuid=NULL → 连接所有 auto_connect profile
```

### 1.2 承载选择详解（三层 if-else）

**情况 A：BR/EDR 已连**

```c
if (dev->bredr_state.connected) {
```

此时 BR/EDR ACL 链路已存在。进一步判断两个条件：

```c
    if (dev->bredr_state.svc_resolved &&
        find_service_with_state(dev->services, BTD_SERVICE_STATE_CONNECTED))
        bdaddr_type = dev->bdaddr_type;
    else
        bdaddr_type = BDADDR_BREDR;
```

`find_service_with_state(dev->services, CONNECTED)` 在 `device.c:360`：

```c
static GSList *find_service_with_state(GSList *list, btd_service_state_t state)
{
    for (l = list; l != NULL; l = g_slist_next(l)) {
        struct btd_service *service = l->data;
        if (btd_service_get_state(service) == state)
            return l;
    }
    return NULL;   // 没找到
}
```

遍历 `dev->services` 链表，找第一个状态匹配的 `btd_service`。返回 `NULL` = 没有。

**两个条件都为 TRUE**：BR/EDR 侧 SDP 已做，且至少一个 profile 连接成功了。BR/EDR 的活已经干完，如果是双模设备，这次 Connect() 应该去 LE 侧连 GATT services。

**任一为 FALSE**：要么 SDP 没做，要么 profile 全失败了，需要重新走 BR/EDR。

| svc_resolved | 有 CONNECTED | 结论 |
|:---:|:---:|------|
| Y | Y | BR/EDR 活干完了 → dev->bdaddr_type（可能是 LE）|
| N | - | SDP 没做 → BDADDR_BREDR |
| Y | N | profile 全失败了 → BDADDR_BREDR (重试) |

**情况 B：LE 已连，但设备支持 BR/EDR**

```c
} else if (dev->le_state.connected && dev->bredr) {
    bdaddr_type = BDADDR_BREDR;
```

隐含 `bredr_state.connected == false`（否则进情况 A）。LE 已连但传统 profile 还没连 → 走 BR/EDR 补连。

**情况 C：两种承载都没连**

```c
} else {
    bdaddr_type = select_conn_bearer(dev);
```

根据设备类型、地址类型、最近出现时间来决定。

### 1.3 LE 分支的关键操作

```c
btd_device_set_temporary(dev, false);
// 临时设备标记清除：用户主动 Connect() 的设备不应被自动过期清理

if (dev->disable_auto_connect) {
    dev->disable_auto_connect = FALSE;
    device_set_auto_connect(dev, TRUE);
}
// 之前因为临时设备而禁用的 auto_connect 重新启用

dev->connect = dbus_message_ref(msg);
// ★ 保存原始 D-Bus 消息的引用，后续异步完成时通过它回复
// dev->connect 一旦非 NULL 即表示"有异步操作进行中"，阻止并发连接
return NULL;
// NULL = 不立即回复 D-Bus，等待异步操作完成
```

---

## 2. device_connect_profiles

文件：`src/device.c:2743`

```c
DBusMessage *device_connect_profiles(struct btd_device *dev,
        uint8_t bdaddr_type, DBusMessage *msg, const char *uuid)
```

### 2.1 参数

| 参数 | 含义 |
|------|------|
| `bdaddr_type` | BDADDR_BREDR 或 LE 类型 |
| `msg` | 原始 D-Bus 调用消息 |
| `uuid` | `NULL` = 连所有 auto_connect profile；非 NULL = 只连指定 UUID（ConnectProfile）|

### 2.2 三个并发锁

```c
if (dev->pending || dev->connect || dev->browse)
    return btd_error_in_progress_str(msg, ERR_BREDR_CONN_BUSY);
```

| 锁 | 类型 | 非空时的含义 |
|----|------|-------------|
| `dev->pending` | `GSList *` | profile 连接队列正在逐个执行 |
| `dev->connect` | `DBusMessage *` | 有一个异步连接操作正在等结果 |
| `dev->browse` | `struct browse_req *` | SDP 或 GATT 发现正在进行 |

三个锁保证同一时刻设备上只有一个连接状态机在走。

### 2.3 adapter 电源检查

```c
if (!btd_adapter_get_powered(dev->adapter))
    return btd_error_not_ready_str(msg, ERR_BREDR_CONN_ADAPTER_NOT_POWERED);
```

### 2.4 核心分叉：svc_resolved

```c
btd_device_set_temporary(dev, false);

if (!state->svc_resolved)
    goto resolve_services;
```

**`svc_resolved` 是整个函数的分水岭**：

- **FALSE** → `device_browse_sdp()` 或 `device_browse_gatt()` → 异步 SDP/GATT 发现
- **TRUE** → `create_pending_list()` → `connect_next()` → 直接连接 profile

### 2.5 svc_resolved == TRUE 路径

```c
dev->pending = create_pending_list(dev, uuid);

if (!dev->pending) {
    // 没有可连的 profile
    if (dev->svc_refreshed) {
        if (dbus_message_is_method_call(msg, DEVICE_INTERFACE, "Connect") &&
            find_service_with_state(dev->services, CONNECTED)) {
            // Connect() 且已有 profile 连上 → 幂等成功
            return dbus_message_new_method_return(msg);
        } else {
            // ConnectProfile() 或没 profile 连上 → 返回错误
            return btd_error_profile_unavailable(msg);
        }
    }
    goto resolve_services;  // svc_refreshed==FALSE → SDP 结果过期 → 重做
}
```

`svc_refreshed` 与 `svc_resolved` 的区别：
- `svc_resolved`：SDP 完成后设为 TRUE，断开连接时不清除（保留用于快速重连）
- `svc_refreshed`：SDP 完成后设为 TRUE，但 **ACL 断开时会被清为 FALSE**

因此可能出现 `svc_resolved==TRUE` 但 `svc_refreshed==FALSE` 的情况（设备断连后重连时），此时 pending 为空会触发 goto resolve_services 重新做 SDP。

```c
err = connect_next(dev);
if (err == -EALREADY)
    return dbus_message_new_method_return(msg);   // 幂等

dev->connect = dbus_message_ref(msg);  // 保存引用，进入异步
return NULL;
```

### 2.6 resolve_services 分支

```c
resolve_services:
    if (bdaddr_type == BDADDR_BREDR)
        err = device_browse_sdp(dev, msg);
    else
        err = device_browse_gatt(dev, msg);
    if (err < 0)
        return btd_error_failed(msg, ...);
    return NULL;  // 异步
```

---

## 3. SDP 服务发现

### 3.1 搜索列表

```c
// device.c:315
static const uint16_t uuid_list[] = {
    L2CAP_UUID,            // 0x0100
    PNP_INFO_SVCLASS_ID,   // 0x1200
    PUBLIC_BROWSE_GROUP,   // 0x1002
    0
};
```

### 3.2 入口

```c
// device.c:6744
static int device_browse_sdp(struct btd_device *device, DBusMessage *msg)
{
    req = browse_request_new(device, BROWSE_SDP, msg);
    // → device->browse = req  ← 加锁

    sdp_uuid16_create(&uuid, uuid_list[req->search_uuid++]);
    // search_uuid: 0→1, uuid = L2CAP_UUID (0x0100)

    err = bt_search(src, dst, &uuid, browse_cb, req, NULL, req->sdp_flags);
}
```

### 3.3 bt_search vs bt_search_service

`sdp-client.c`:

```c
bt_search()         → filter_svc_class = FALSE  // svclass 不需要精确匹配
bt_search_service() → filter_svc_class = TRUE   // svclass 必须精确匹配
```

第一步用 `bt_search(L2CAP_UUID)` —— 宽松匹配，匹配更多的 record。后续步骤用 `bt_search_service()` —— 精确匹配。

### 3.4 SDP 客户端完整交互过程

```
bt_search()
  └─ create_search_context()
       ├─ get_cached_sdp_session(src, dst)
       │   SDP 的 L2CAP session 缓存 2 秒。命中则复用，未命中创建新的。
       │
       ├─ 缓存未命中 → sdp_connect(src, dst, SDP_NON_BLOCKING)
       │     socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP)
       │     bind(src)
       │     connect(dst, PSM=0x0001)          ← SDP 固定 PSM
       │     // ★ 内核在此过程中自动建立 ACL（如果尚未建立）
       │     // 这是 BR/EDR 连接中 ACL 最常见的触发点
       │
       └─ g_io_add_watch(G_IO_OUT, connect_watch)
            // 等 L2CAP 非阻塞 connect 完成
```

L2CAP 连接完成后 `connect_watch` 触发：

```c
connect_watch:
  ├─ getsockopt(SO_ERROR) → 检查连接是否成功
  ├─ sdp_set_notify(search_completed_cb)      // 注册响应解析回调
  ├─ sdp_service_search_attr_async(search, attrids)
  │     构建并发送 PDU:
  │       pdu_id        = SDP_SVC_SEARCH_ATTR_REQ
  │       ServiceSearchPattern = [L2CAP_UUID]
  │       AttributeIDList      = [0x0000-0xFFFF]  // 请求所有属性
  │       MaxAttributeByteCount = 0xFFFF
  └─ g_io_add_watch(G_IO_IN, search_process_cb)
       // 等远端回复
```

远端回复到达：

```c
search_process_cb:
  └─ sdp_process(session) → recv() → 解析 SDP_SVC_SEARCH_ATTR_RSP
       │  ContinuationState == 0 → 完整响应
       │  ContinuationState != 0 → 数据太大需要继续请求（客户端自动处理）
       └─ search_completed_cb
            ├─ sdp_extract_pdu() → sdp_record_t 链表
            ├─ filter_svc_class 过滤（仅 bt_search_service）
            ├─ cache_sdp_session() → 缓存 2 秒（下一步搜索复用）
            └─ ctxt->cb(recs, err, user_data)  → browse_cb (device.c)
```

### 3.5 browse_cb：多步搜索的编排

```c
// device.c:6130
static void browse_cb(sdp_list_t *recs, int err, gpointer user_data)
{
    struct browse_req *req = user_data;

    // 提前终止条件
    if (err < 0 || (req->search_uuid == 2 && req->records)) {
        if (err == -ECONNRESET && req->reconnect_attempt < 1) {
            req->search_uuid--;
            req->reconnect_attempt++;   // 重试一次
        } else
            goto done;
    }

    update_bredr_services(req, recs);

    if (uuid_list[req->search_uuid]) {
        sdp_uuid16_create(&uuid, uuid_list[req->search_uuid++]);
        bt_search_service(src, dst, &uuid, browse_cb, ...);
        return;
    }

done:
    search_cb(recs, err, user_data);
}
```

### 3.6 search_uuid 追踪

```
初始 search_uuid = 0

步骤 1: device_browse_sdp 内部
  uuid = uuid_list[0] = L2CAP_UUID
  search_uuid → 1

步骤 2: browse_cb 第 1 次
  search_uuid = 1, req->records = NULL
  条件 (1==2 && NULL) = FALSE → 不走 done
  update_bredr_services → 处理 L2CAP 的 records，提取 UUID
  uuid_list[1] = PNP_INFO_SVCLASS_ID → 继续搜索
  search_uuid → 2

步骤 3: browse_cb 第 2 次
  search_uuid = 2, req->records = {L2CAP 的记录} ≠ NULL
  条件 (2==2 && records≠NULL) = TRUE → goto done
  ★ 注意: update_bredr_services 未被调用！PNP 的结果由 search_cb 处理
  → search_cb(recs, err, user_data)   // recs 是 PNP 搜索的结果
```

第三步 `PUBLIC_BROWSE_GROUP` 被跳过——如果 L2CAP 已经返回了足够的 record，不再需要搜所有公开服务。

### 3.7 search_cb：最终汇总

```c
// device.c:6068
static void search_cb(sdp_list_t *recs, int err, gpointer user_data)
{
    // 1. 处理 PNP 的搜索结果
    update_bredr_services(req, recs);

    // 2. 保存到设备
    device->tmp_records = req->records;

    // 3. 提取 GATT primary service handles
    primaries = device_services_from_record(device, req->profiles_added);
    device_register_primaries(device, primaries, BT_ATT_PSM);

    // 4. ★ UUID → Profile 匹配
    device_probe_profiles(device, req->profiles_added);

    // 5. D-Bus 通知 UUIDs 属性变化
    g_dbus_emit_property_changed(..., "UUIDs");

    // 6. 标记完成
    device_svc_resolved(device, BROWSE_SDP, BDADDR_BREDR, err);
}
```

### 3.8 device_svc_resolved

```c
state->svc_resolved = true;                         // ★ 下次不做 SDP
device_set_svc_refreshed(dev, true);                 // ServicesResolved = true
store_device_info(dev);                              // 持久化到磁盘
browse_request_complete(req, type, bdaddr_type, err); // ★ 触发 profile 连接
```

---

## 4. SDP 结果如何变成 btd_service

### 4.1 update_record：UUID 提取

```c
// device.c:5853
static int update_record(struct browse_req *req, const char *uuid, sdp_record_t *rec)
{
    // record 去重
    if (sdp_list_find(req->records, rec, rec_cmp))
        return -EALREADY;

    req->records = sdp_list_append(req->records, sdp_copy_record(rec));

    // UUID 去重 → 新 UUID 加入 profiles_added
    if (g_slist_find_custom(req->device->uuids, uuid, ...) == NULL &&
        g_slist_find_custom(req->profiles_added, uuid, ...) == NULL) {
        req->profiles_added = g_slist_append(req->profiles_added, g_strdup(uuid));
    }
    return 0;
}
```

SDP response 中每条 `sdp_record_t` 的 `ServiceClassIDList`（属性 0x0001）包含 UUID。一个 record 可能包含多个 UUID（如 HFP AG: `0x111f` + `0x1203`），每个都被独立提取。

### 4.2 device_probe_profiles → 全局匹配

```c
// device.c:5752
void device_probe_profiles(struct btd_device *device, GSList *uuids)
{
    btd_profile_foreach(dev_probe, &d);
    // → 遍历所有已注册的 btd_profile
    // → 每个调用 probe_service() 检查
}
```

### 4.3 probe_service

```c
// device.c:5658
static struct btd_service *probe_service(device, profile, uuids)
{
    if (profile->device_probe == NULL)     return NULL;    // 无 probe 回调
    if (!device_match_profile(...))        return NULL;    // UUID 不匹配
    if (find_service_with_profile(...))    return NULL;    // 已存在

    service = service_create(device, profile);
    if (service_probe(service)) {           // 调用 profile->device_probe
        btd_service_unref(service);
        return NULL;
    }
    return service;   // 加入 dev->services
}
```

**匹配条件**: `profile->remote_uuid` 在 SDP 返回的 UUID 列表中。

### 4.4 状态变化

```c
// service.c:216
int service_probe(struct btd_service *service)
{
    err = profile->device_probe(service);   // 如 a2dp_source_probe()
    if (err == 0)
        change_state(service, BTD_SERVICE_STATE_DISCONNECTED);
    return err;
}
```

**此时所有匹配到的 service 状态: UNAVAILABLE → DISCONNECTED。尚未发起任何连接。**

---

## 5. SDP 完成后谁来触发 profile 连接

这是从 SDP 到 profile 连接最关键的衔接点。

### 5.1 browse_request_complete

```c
// device.c:3052
static void browse_request_complete(struct browse_req *req, uint8_t type,
                                    uint8_t bdaddr_type, int err)
{
    if (dbus_message_is_method_call(req->msg, DEVICE_INTERFACE, "Pair")) {
        reply = g_dbus_create_reply(req->msg, DBUS_TYPE_INVALID);
        goto done;   // Pair → 只回复配对成功，不连接 profile
    }

    // Connect 请求 → 先解锁，再重入
    msg = dbus_message_ref(req->msg);
    browse_request_free(req);    // ★ device->browse = NULL
    req = NULL;

    if (dbus_message_is_method_call(msg, DEVICE_INTERFACE, "Connect"))
        reply = dev_connect(dbus_conn, msg, dev);         // ★ 重入入口
    else if (dbus_message_is_method_call(msg, DEVICE_INTERFACE, "ConnectProfile"))
        reply = connect_profile(dbus_conn, msg, dev);
}
```

### 5.2 为什么必须先 browse_request_free

`device_connect_profiles` 第一行：

```c
if (dev->pending || dev->connect || dev->browse)
    return btd_error_in_progress_str(...);
```

如果不释放 `device->browse`，重入 `dev_connect()` → `device_connect_profiles()` 时会被自己的 browse 标记挡住，返回 InProgress。 **必须先 free 再重入。**

### 5.3 重入后的路径

```
第一次 device_connect_profiles():
  svc_resolved = FALSE → device_browse_sdp()

SDP 完成 → browse_request_complete → browse_request_free → dev_connect 重入

第二次 device_connect_profiles():
  svc_resolved = TRUE → create_pending_list → connect_next
```

**触发 profile 连接的就是 `browse_request_complete` 中的 `dev_connect()` 重入。** 重入时 `svc_resolved` 已经是 TRUE，直接走连接路径。

### 5.4 Pair 与 Connect 的区别

| 入口 | SDP 后的行为 |
|------|-------------|
| `Connect()` | browse_request_complete → dev_connect 重入 → connect_next |
| `ConnectProfile(uuid)` | 同上 → connect_profile 重入 → connect_next |
| `Pair()` | 直接回复 Pair 成功，**不连接任何 profile** |
| `device_browse_sdp(dev, NULL)` (被动刷新) | req->msg == NULL → 跳过 |

---

## 6. create_pending_list

```c
// device.c:2609
static GSList *create_pending_list(struct btd_device *dev, const char *uuid)
```

### 6.1 uuid != NULL（ConnectProfile）

```c
service = find_connectable_service(dev, uuid);
if (service && btd_service_is_allowed(service))
    return g_slist_prepend(dev->pending, service);  // 单个 service
```

### 6.2 uuid == NULL（Connect 全部）

```c
for (l = dev->services; l; l = l->next) {
    struct btd_service *service = l->data;
    struct btd_profile *p = btd_service_get_profile(service);

    if (!p->auto_connect)                continue;   // 不自动连接
    if (!btd_service_is_allowed(service)) continue;   // adapter 禁止此 UUID
    if (g_slist_find(dev->pending, service)) continue; // 已在队列
    if (state != DISCONNECTED)           continue;   // 已 CONNECTED 或 CONNECTING

    dev->pending = g_slist_append(dev->pending, service);
}
```

### 6.3 排序

```c
dev->pending = btd_profile_sort_list(dev->pending, get_service_profile, NULL);
```

排序规则（`profile.c:2679`）：

1. 按 `priority` 降序：`HIGH(2)` → `MEDIUM(1)` → `LOW(0)`
2. 处理 `after_services` 依赖：声明了依赖的 profile **排在它所依赖的 profile 后面**

A2DP Source 声明了 `after_services = {A2DP_SINK_UUID}`：

```
排序后: [HFP_AG, A2DP_Sink, A2DP_Source]
          ↑2        ↑1              ↑0
     HIGH优先   Source依赖它     依赖项在后面
```

**这个排序是保证连接顺序的主要机制。** 依赖项排在前面意味着 `connect_next` 会先连它。

---

## 7. connect_next

```c
// device.c:2333
static int connect_next(struct btd_device *dev)
{
    while (dev->pending) {
        service = dev->pending->data;    // 取队头

        err = btd_service_connect(service);
        // 内部依次执行:
        //   add_depends(service)
        //   profile->connect(service)
        //   change_state(CONNECTING)

        if (!err)
            return 0;  // ★ 成功发起就退出
        // 失败 → 移除队头，试下一个
        dev->pending = g_slist_delete_link(dev->pending, dev->pending);
    }
    return err;  // 全失败
}
```

**一次只发起一个连接。** 成功后立即返回 0，后续 profile 由 `device_profile_connected` 驱动。

---

## 8. profile->connect 的异步回调链

### 8.1 以 A2DP Source 为例

```c
// a2dp.c: a2dp_source_connect()
  └─ source_connect(service)           // source.c:278
       └─ source_setup_stream(service, NULL)
            ├─ source->session = a2dp_avdtp_get(device)
            └─ a2dp_discover(session, discovery_complete, source)
                 // 发送 AVDTP Discover Command (L2CAP PSM 25)
                 // 注册 discovery_complete 为回调
                 return 0;  // ★ 立即返回，不等完成
```

**`profile->connect()` 只发起连接，不等待完成。** 控制权立即交还事件循环。

### 8.2 AVDTP 的异步回调链

```
远端回复 AVDTP Discover Response
  └─ discovery_complete()
       └─ a2dp_select_capabilities(..., select_complete, source)

Codec 选择完成
  └─ select_complete()
       └─ a2dp_config(..., stream_setup_complete, source)

Stream 配置完成
  └─ stream_setup_complete()
       └─ 等待 AVDTP OPEN 事件

AVDTP Stream 变为 OPEN
  └─ stream_state_changed(..., AVDTP_STATE_OPEN, ...)
       └─ btd_service_connecting_complete(source->service, 0)
            └─ change_state(CONNECTED)  ← ★ 触发后续
```

### 8.3 失败

任意阶段失败调用 `btd_service_connecting_complete(service, err)` → `change_state(DISCONNECTED, err)`。

---

## 9. device_profile_connected

### 9.1 触发路径

```
btd_service_connecting_complete(service, 0)
  └─ change_state(service, CONNECTED)
       └─ 遍历全局 state_callbacks
            └─ service_state_changed(old=CONNECTING, new=CONNECTED)
                 └─ device_profile_connected(dev, profile, 0)
```

`service_state_changed` 在 `btd_device_init()` 中注册（`device.c:8218`）：

```c
static void service_state_changed(service, old_state, new_state, user_data)
{
    if (new_state == CONNECTING || new_state == DISCONNECTING)
        return;   // 忽略中间态

    if (old_state == CONNECTING)
        device_profile_connected(device, profile, err);
    else if (old_state == DISCONNECTING)
        device_profile_disconnected(device, profile, err);
}
```

### 9.2 函数全貌

```c
static void device_profile_connected(struct btd_device *dev,
                struct btd_profile *profile, int err)
{
    if (!err)
        btd_device_set_temporary(dev, false);

    // pending 已空 → 结束
    if (dev->pending == NULL)
        goto done;

    // ACL 级别故障 → 放弃整个队列
    if (!btd_device_is_connected(dev)) {
        switch (-err) {
        case EHOSTDOWN:     // page timeout — 远端没响应
        case EHOSTUNREACH:  // adapter 断电
        case ECONNABORTED:  // adapter 被关闭
            goto done;      // 链路级别故障，继续逐个重试无意义
        }
    }

    // 从 pending 移除已完成的 profile
    pending = dev->pending->data;
    l = find_service_with_profile(dev->pending, profile);
    if (l != NULL)
        dev->pending = g_slist_delete_link(dev->pending, l);

    // 只有队头完成才继续
    if (profile != btd_service_get_profile(pending))
        return;

    // 连接下一个
    if (connect_next(dev) == 0)
        return;

done:
    g_slist_free(dev->pending);
    dev->pending = NULL;

    if (!dev->connect)
        return;   // 无待回复的 D-Bus 消息

    // Connect(): 至少有一个连上就成功
    if (dbus_message_is_method_call(dev->connect, DEVICE_INTERFACE, "Connect")) {
        if (!err)
            dev->general_connect = TRUE;
        else if (find_service_with_state(dev->services, CONNECTED))
            err = 0;  // 其他 profile 连上了 → 重置错误
    }

    // 回复 D-Bus
    if (err) {
        if (err == -EHOSTDOWN && dev->le && !dev->le_state.connected)
            err = device_connect_le(dev);  // fallback 到 LE
        if (err)
            g_dbus_send_message(dbus_conn, btd_error_bredr_errno(dev->connect, err));
    } else {
        g_dbus_send_reply(dbus_conn, dev->connect, DBUS_TYPE_INVALID);
    }
    dbus_message_unref(dev->connect);
    dev->connect = NULL;
}
```

### 9.3 为什么需要"只有队头完成才继续"

`after_services` 依赖存在时，非队头的 profile 可能先完成。如果不管队头、谁完成都推进队列，会出现并发连接混乱。队头判断保证了每次只有一个 profile 在队列头部被处理。

---

## 10. add_depends

### 10.1 函数本身

```c
// service.c:245
static void add_depends(struct btd_service *service)
{
    struct btd_profile_uuid_cb *after = &service->profile->after_services;

    queue_destroy(service->depends, NULL);    // 清空旧依赖
    service->depends = queue_new();

    for (i = 0; i < after->count; ++i) {
        dep = btd_device_get_service(service->device, after->uuids[i]);
        //   → 在 dev->services 链表中按 UUID 查找

        if (!dep)                     continue;   // 不存在
        if (dep->state != CONNECTING) continue;   // ★ 重要的条件

        // 建立双向关联
        queue_push_tail(service->depends, dep);   // service: "我等 dep"
        queue_push_tail(dep->dependents, service); // dep: "我完了通知 service"
    }
}
```

**被调用的两个位置**：

```c
btd_service_connect(service)   → add_depends(service) → profile->connect(service)
service_accept(service)        → add_depends(service) → profile->accept(service)
```

### 10.2 "等"不是阻塞

`add_depends` 之后 **`profile->connect()` 立即被调用**，不等待。

```c
add_depends(service);           // 建立依赖关系
err = profile->connect(service); // 紧接着就发起连接！不阻塞
change_state(service, CONNECTING);
```

"等"指的是 `after->func` 回调的触发时机被延迟——只有当 depends 队列中的所有被依赖方都完成（状态变为非 CONNECTING）后，`after->func` 才被调用。

### 10.3 依赖建立的唯一前提

```c
if (dep->state != BTD_SERVICE_STATE_CONNECTING)
    continue;
```

**只有被依赖的 service 正处于 CONNECTING 状态时，才会建立依赖。** 如果 dep 已经是 CONNECTED（完成了）或 DISCONNECTED（没在连），`add_depends` 什么都不做。

这意味着 **被依赖方一定已经被某个触发者启动连接、正处于连接过程中**。

触发者可能是：
- `connect_next`（从 pending 队列取出、发起）
- `service_accept`（被动接受远端连接）

### 10.4 在正常 pending 队列场景中无效

正常流程中，pending 队列的排序保证了被依赖方排在前面，`connect_next` 的串行执行保证它先连完：

```
pending = [A2DP_Sink, A2DP_Source]   ← Sink 在前（排序保证）

connect_next → Sink → CONNECTING
Sink 完成 → CONNECTED
connect_next → Source
  → add_depends(Source)
       dep = Sink
       dep->state = CONNECTED（不是 CONNECTING）
       → continue → ★ 不建立依赖
```

**排序 + 串行执行已经保证了顺序。add_depends 在这里不创建任何依赖关系。**

### 10.5 真正起作用的场景：service_accept

远端主动连接时，被依赖方可能正在 CONNECTING：

```
connect_next → Sink → CONNECTING

同时远端发起 AVDTP 连接（对应此设备的 Source 角色）
  → service_accept(Source)
       → add_depends(Source)
            dep = Sink
            dep->state = CONNECTING → ★ 建立依赖: Source->depends = {Sink}
```

### 10.6 谁来触发依赖 ready

```
被依赖方（Sink）连接完成
  └─ btd_service_connecting_complete(Sink, 0)
       └─ change_state(Sink, CONNECTED)
            └─ service_ready(Sink)               ← ★ 触发者
                 ├─ queue_foreach(Sink->dependents, depends_ready, Sink)
                 │    └─ depends_ready(Source, Sink)
                 │         ├─ queue_remove(Source->depends, Sink)
                 │         ├─ Source->depends 变空 → 所有依赖满足
                 │         └─ after->func(Source)   // 通知依赖方
                 │
                 └─ depends_ready(Sink, NULL)  // Sink 自己不需要等
```

触发者是 `change_state` 中的 `service_ready`。当被依赖方进入非 CONNECTING 状态（通常是 CONNECTED）时，`service_ready` 遍历它的 `dependents` 列表通知所有等待者。

### 10.7 after->func

A2DP Source 的 `after->func` 是 NULL：

```c
// a2dp.c:3774
.after_services = BTD_PROFILE_UUID_CB(NULL, A2DP_SINK_UUID),
```

所以即使 depends 被满足，也不执行任何回调。 **A2DP 的依赖顺序完全由 `btd_profile_sort_list` + `connect_next` 串行保证。**

唯一使用 `after->func` 非 NULL 的是 BAP（LE Audio）：

```c
// bap.c:4034
.after_services = BTD_PROFILE_UUID_CB(bap_services_ready,
                    VCS_UUID_STR, TMAS_UUID_STR, GMAS_UUID_STR),

static void bap_services_ready(struct btd_service *service)
{
    data->services_ready = true;
    if (data->bap_ready)
        bap_ucast_start(data);  // 所有前置 service 连完 → 启动音频流
}
```

---

## 11. 完整时序图

```
Connect()

dev_connect()
  ├─ bonding → InProgress
  ├─ 承载选择 → BDADDR_BREDR
  └─ device_connect_profiles(dev, BDADDR_BREDR, msg, NULL) [第 1 次]

device_connect_profiles [第 1 次]
  ├─ pending || connect || browse → InProgress
  ├─ adapter 未开机 → NotReady
  ├─ svc_resolved = FALSE → resolve_services
  │
  └─ device_browse_sdp(dev, msg)
       ├─ browse_request_new → device->browse = req
       │
       ├─ bt_search(L2CAP_UUID)
       │    ├─ sdp_connect → L2CAP PSM 0x0001 → 内核建立 ACL
       │    ├─ send(SDP_SVC_SEARCH_ATTR_REQ)
       │    ├─ recv(SDP_SVC_SEARCH_ATTR_RSP) → sdp_extract_pdu → recs
       │    └─ browse_cb [第 1 次]
       │         ├─ update_bredr_services → profiles_added 填充 UUID
       │         └─ bt_search_service(PNP_INFO)
       │              └─ ... → browse_cb [第 2 次]
       │                   ├─ search_uuid==2, records≠NULL → goto done
       │                   └─ search_cb(recs)
       │                        ├─ update_bredr_services
       │                        ├─ device_probe_profiles(profiles_added)
       │                        │    HFP_AG_UUID   → service_create → DISCONNECTED
       │                        │    A2DP_SINK_UUID → service_create → DISCONNECTED
       │                        │    A2DP_SOURCE   → service_create → DISCONNECTED
       │                        │
       │                        └─ device_svc_resolved()
       │                             ├─ svc_resolved = TRUE
       │                             └─ browse_request_complete()
       │                                  ├─ browse_request_free → device->browse = NULL
       │                                  └─ dev_connect() 重入

device_connect_profiles [第 2 次]
  ├─ svc_resolved = TRUE
  ├─ create_pending_list(dev, NULL)
  │    ├─ 过滤: auto_connect && DISCONNECTED && allowed
  │    └─ btd_profile_sort_list → [HFP_AG, A2DP_Sink, A2DP_Source]
  │
  ├─ connect_next(dev)
  │    └─ btd_service_connect(HFP_AG)
  │         ├─ add_depends(HFP_AG)   // 无依赖 → 空
  │         ├─ hfp_ag_connect() → RFCOMM connect → return 0
  │         └─ change_state(CONNECTING)
  │
  └─ dev->connect = msg → return NULL (异步)

[HFP AG 连接完成]
  btd_service_connecting_complete(HFP_AG, 0)
    └─ change_state(CONNECTED)
         └─ service_state_changed → device_profile_connected
              ├─ 从 pending 移除 HFP_AG
              ├─ HFP_AG == pending->data → YES → connect_next
              │    └─ btd_service_connect(A2DP_Sink)
              │         ├─ add_depends(Sink)    // 无依赖 → 空
              │         └─ Sink → CONNECTING → AVDTP 连接
              └─ service_ready(HFP_AG)  // 无 dependents

[A2DP Sink 连接完成]
  device_profile_connected(A2DP_Sink)
    └─ connect_next
         └─ btd_service_connect(A2DP_Source)
              ├─ add_depends(Source)
              │    Sink state = CONNECTED → 跳过！不建依赖
              └─ Source → CONNECTING → AVDTP 连接

[A2DP Source 连接完成]
  device_profile_connected(A2DP_Source)
    ├─ pending == NULL → goto done
    └─ g_dbus_send_reply(dev->connect, SUCCESS)
         dev->connect = NULL (解锁)

[D-Bus 客户端收到 Connect() 回复]
```

---

## 12. 附录

### 12.1 关键数据结构

**`struct btd_device`** (device.c:201)

| 字段 | 类型 | 用途 |
|------|------|------|
| `pending` | `GSList *` | 待连接队列，按 priority + after_services 排序 |
| `connect` | `DBusMessage *` | 非 NULL = 异步操作进行中，阻止并发 |
| `browse` | `struct browse_req *` | 非 NULL = SDP/GATT 发现进行中，阻止并发 |
| `services` | `GSList *` | SDP 匹配后创建的所有 `btd_service` |
| `bonding` | `struct bonding_req *` | 配对进行中时阻止 Connect() |

**`struct bearer_state`** (device.c:156)

| 字段 | 类型 | 用途 |
|------|------|------|
| `svc_resolved` | `bool` | SDP 完成后设 TRUE，ACL 断开不清除 |
| `connected` | `bool` | ACL 链路存在 |
| `svc_refreshed` | `bool` | SDP 完成后设 TRUE，**ACL 断开时被清除** |

**`struct btd_service`** (service.c:39)

| 字段 | 类型 | 用途 |
|------|------|------|
| `state` | `enum` | `UNAVAILABLE→DISCONNECTED→CONNECTING→CONNECTED→DISCONNECTING` |
| `depends` | `queue *` | 依赖列表 — 等这些 service 连完 |
| `dependents` | `queue *` | 被依赖列表 — 我连完后通知这些 service |

**`struct browse_req`** (device.c:128)

| 字段 | 类型 | 用途 |
|------|------|------|
| `msg` | `DBusMessage *` | 原始 D-Bus 消息（用于异步回复）|
| `profiles_added` | `GSList *` | 本次 SDP 新发现的 UUID |
| `records` | `sdp_list_t *` | 本次 SDP 获取的 record |
| `search_uuid` | `int` | 当前搜索步骤：0→L2CAP, 1→PNP, 2→BROWSE |

### 12.2 文件索引

| 文件 | 职责 |
|------|------|
| `src/device.c` | 设备连接编排、状态机、SDP 发现流程 |
| `src/service.c` | `btd_service` 生命周期、状态机、依赖管理 |
| `src/profile.c` | profile 注册注销、排序算法、external profile D-Bus |
| `src/profile.h` | `btd_profile` 结构体定义 |
| `src/sdp-client.c` | SDP 客户端：L2CAP 连接、PDU 编解码 |
| `src/adapter.c` | adapter 管理、mgmt 事件处理 |
| `profiles/audio/a2dp.c` | A2DP profile 实例 |
| `profiles/audio/source.c` | A2DP Source 角色异步回调链 |
