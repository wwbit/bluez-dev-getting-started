# BlueZ D-Bus 框架

## 一、整体架构

```
应用层 (adapter.c, device.c, gatt-client.c, profiles/* ...)
    │
    ▼
GDBus 封装层 (gdbus/)          ← BlueZ 自研的轻量 libdbus 封装
    ├── mainloop.c    桥接 D-Bus → glib 事件循环
    ├── object.c      对象/接口管理（最核心）
    ├── client.c      D-Bus 代理客户端，调用外部服务
    ├── watch.c       信号 / 服务状态监听
    └── polkit.c      PolicyKit 权限检查
    │
    ▼
libdbus (系统 D-Bus 库)
    │
    ▼
D-Bus System Bus
```

---

## 二、注册顺序：先注册"总的"，再注册"子对象"

### 2.1 启动时先注册顶层 ObjectManager

`main.c:1402`：

```c
g_dbus_attach_object_manager(conn);
```

在 `/` 路径上注册 `org.freedesktop.DBus.ObjectManager`，它管理整个对象树。客户端通过它一次性发现所有对象。

### 2.2 然后各模块依次注册子对象

```c
// 适配器发现后
g_dbus_register_interface(dbus_conn,
    adapter->path,          // "/org/bluez/hci0"
    ADAPTER_INTERFACE,      // "org.bluez.Adapter1"
    adapter_methods,        // 方法表: StartDiscovery ...
    NULL,                   // 信号表: 无自定义
    adapter_properties,     // 属性表: Address, Name, Powered ...
    adapter,                // user_data → struct btd_adapter *
    adapter_free);          // 注销回调
```

### 2.3 最终对象树

```
/                                       ← ObjectManager（"总的"）
│   org.freedesktop.DBus.ObjectManager
│
├── /org/bluez                         ← AgentManager1 + ProfileManager1
│
└── /org/bluez/hci0                    ← Adapter1 + GattManager1 + ...
    │
    ├── /org/bluez/hci0/dev_00_11_...  ← Device1 + Battery1 + ...
    │   ├── serviceXXXX                ← GattService1（远端 GATT）
    │   │   └── charYYYY               ← GattCharacteristic1
    │   │       └── descZZZZ           ← GattDescriptor1
    │   └── playerX                    ← MediaPlayer1
    │
    ├── serviceXXXX                    ← GattService1（本地 GATT）
    │   └── charYYYY                   ← GattCharacteristic1
    │       └── descZZZZ               ← GattDescriptor1
    │
    └── advertisingXXXX                ← LEAdvertisement1
```

---

## 三、mainloop 桥接 (gdbus/mainloop.c)

把 D-Bus 的 I/O 机制接入 GLib 事件循环：

### 3.1 Watch：socket fd → GLib

```c
// 核心代码 mainloop.c:112-143
static dbus_bool_t add_watch(DBusWatch *watch, void *data)
{
    int fd = dbus_watch_get_unix_fd(watch);
    GIOChannel *chan = g_io_channel_unix_new(fd);

    info->id = g_io_add_watch(chan, cond, watch_func, info);
    // 当 socket 可读/可写时回调 watch_func
}
```

`watch_func` 调用 `dbus_watch_handle()` 处理 I/O，然后检查 dispatch 状态，将消息分发放入 GLib idle 队列。

### 3.2 Timeout：D-Bus timer → GLib

```c
// mainloop.c:192-208
static dbus_bool_t add_timeout(DBusTimeout *timeout, void *data)
{
    int interval = dbus_timeout_get_interval(timeout);
    handler->id = g_timeout_add(interval, timeout_handler_dispatch, handler);
}
```

### 3.3 启动时的完整调用链

```c
// mainloop.c:273-293
DBusConnection *g_dbus_setup_bus(DBusBusType type, const char *name, DBusError *error)
{
    conn = dbus_bus_get(DBUS_BUS_SYSTEM, error);   // 获取系统总线连接
    g_dbus_request_name(conn, name, error);        // 请求 bus name "org.bluez"
    setup_dbus_with_main_loop(conn);               // 设置 watch/timeout/dispatch
    return conn;
}
```

---

## 四、核心：Object/Interface 管理 (gdbus/object.c)

### 4.1 数据结构（内存中的对象树表示）

```c
struct generic_data {       // 每个 object path 一个
    unsigned int refcount;
    DBusConnection *conn;
    char *path;             // "/org/bluez/hci0"
    GSList *interfaces;     // 挂载的 interface 链表
    GSList *objects;        // 子对象链表（树结构）
    GSList *added;          // 待发送 InterfacesAdded 的接口
    GSList *removed;        // 待发送 InterfacesRemoved 的接口名
    guint process_id;       // idle callback id，用于延迟批量处理
    gboolean pending_prop;  // 是否有待发送的 PropertiesChanged
    char *introspect;       // 缓存的 Introspect XML
    struct generic_data *parent;  // 父对象
};

struct interface_data {     // 每个 interface 一个
    char *name;             // "org.bluez.Adapter1"
    const GDBusMethodTable *methods;
    const GDBusSignalTable *signals;
    const GDBusPropertyTable *properties;
    GSList *pending_prop;   // 待合并发送的 property 变更
    void *user_data;        // 模块透传的业务对象指针
    GDBusDestroyFunction destroy;
};
```

### 4.2 关键关系：1 path = N interface, 1 interface = 3 个表

```
/org/bluez/hci0                                    ← 1 个 path
├── interface "org.freedesktop.DBus.Introspectable" → 方法表: Introspect
│                                                     （框架自动加）
├── interface "org.freedesktop.DBus.Properties"      → 方法表: Get/Set/GetAll
│                                    + 信号表: PropertiesChanged
│                                                     （有属性表时框架自动加）
└── interface "org.bluez.Adapter1"                   → 方法表: StartDiscovery...
│                                    + 属性表: Address, Name, Powered...
│                                                     （模块手动注册）
```

**3 个 interface → 3 套方法表、1 套属性表、1 套信号表。** 代码里就是三次 `add_interface()` 调用：

```c
// object.c:1366 — 首次创建 path 时自动加
add_interface(data, "org.freedesktop.DBus.Introspectable",
              introspect_methods, NULL, NULL, data, NULL);

// object.c:1472 — register_interface 时如果 properties 非 NULL，自动加
add_interface(data, "org.freedesktop.DBus.Properties",
              properties_methods, properties_signals, NULL, data, NULL);

// object.c:1466 — 调用者手动传进来的
add_interface(data, "org.bluez.Adapter1",
              adapter_methods, NULL, adapter_properties, adapter, adapter_free);
```

---

## 五、g_dbus_register_interface() 参数详解

```c
gboolean g_dbus_register_interface(
    DBusConnection *connection,          // ① D-Bus 连接
    const char *path,                    // ② 对象路径
    const char *name,                    // ③ 接口名
    const GDBusMethodTable *methods,     // ④ 方法表
    const GDBusSignalTable *signals,     // ⑤ 信号表
    const GDBusPropertyTable *properties,// ⑥ 属性表
    void *user_data,                     // ⑦ 业务数据指针
    GDBusDestroyFunction destroy         // ⑧ 析构回调
);
```

### 实际调用对应关系（adapter.c:9387）

```c
g_dbus_register_interface(dbus_conn,
    adapter->path,              // ② "/org/bluez/hci0"
    ADAPTER_INTERFACE,          // ③ "org.bluez.Adapter1"
    adapter_methods,            // ④ StartDiscovery, StopDiscovery ...
    NULL,                       // ⑤ 无自定义信号
    adapter_properties,         // ⑥ Address, Name, Powered ...
    adapter,                    // ⑦ struct btd_adapter * 透传给所有回调
    adapter_free);              // ⑧ 注销时释放 adapter
```

### 各参数说明

**① connection**：启动时创建的系统总线连接，全局共享。`dbus-common.c` 保存：

```c
static DBusConnection *connection = NULL;
void set_dbus_connection(DBusConnection *conn) { connection = conn; }
DBusConnection *btd_get_dbus_connection(void) { return connection; }
```

**② path**：对象路径，D-Bus 格式，斜杠分隔。类比 **门牌号**。例如 `/org/bluez/hci0` 指"第 0 号蓝牙适配器"。

**③ name**：接口名，点号分隔。类比 **身份/合同**。例如 `org.bluez.Adapter1` 表示"这是一个蓝牙适配器"。同一个 path 可以挂多个 interface。比如 `/org/bluez/hci0` 同时挂 `org.bluez.Adapter1`、`org.freedesktop.DBus.Introspectable`、`org.freedesktop.DBus.Properties`。

| | path | interface |
|---|---|---|
| 含义 | **哪个** 蓝牙适配器 | **是什么** 类型的对象 |
| 格式 | `/org/bluez/hci0` | `org.bluez.Adapter1` |
| 多对多 | 1 个 path 可挂多个 interface | 1 个 interface 可出现在多个 path |

**④ methods**：`GDBusMethodTable` 数组，`{ }` 结尾。每个元素：

```c
struct GDBusMethodTable {
    const char *name;              // 方法名
    GDBusMethodFunction function;  // 处理函数
    GDBusMethodFlags flags;        // ASYNC / NOREPLY / EXPERIMENTAL ...
    unsigned int privilege;        // PolicyKit 权限级别
    const GDBusArgInfo *in_args;   // 入参 (name + D-Bus 签名)
    const GDBusArgInfo *out_args;  // 出参 (name + D-Bus 签名)
};
```

定义示例（用宏简化）：

```c
static const GDBusMethodTable adapter_methods[] = {
    // 异步方法：返回 NULL，稍后手动发 reply
    { GDBUS_ASYNC_METHOD("StartDiscovery", NULL, NULL, start_discovery) },

    // 同步方法：入参 a{sv}，无出参
    { GDBUS_METHOD("SetDiscoveryFilter",
                GDBUS_ARGS({ "properties", "a{sv}" }), NULL,
                set_discovery_filter) },

    // experimental 方法：仅 --experimental 下可用
    { GDBUS_EXPERIMENTAL_ASYNC_METHOD("ConnectDevice",
                GDBUS_ARGS({ "properties", "a{sv}" }), NULL,
                connect_device) },
    { }   // 哨兵
};
```

注册宏含义：

| 宏 | 行为 |
|----|------|
| `GDBUS_METHOD` | 同步方法，框架自动发 reply |
| `GDBUS_ASYNC_METHOD` | 异步方法，返回 NULL，后续手动回复 |
| `GDBUS_NOREPLY_METHOD` | 不等待回复 |
| `GDBUS_EXPERIMENTAL_METHOD` | 仅 `--experimental` 时可见 |
| `GDBUS_TESTING_METHOD` | 仅 `--testing` 时可见 |

**⑤ signals**：`GDBusSignalTable` 数组，`{ }` 结尾。定义自定义信号，没有就传 NULL。框架要求发出信号时必须有对应表项（校验签名用）。

**⑥ properties**：`GDBusPropertyTable` 数组，`{ }` 结尾。

```c
struct GDBusPropertyTable {
    const char *name;              // 属性名
    const char *type;              // D-Bus 类型签名
    GDBusPropertyGetter get;       // Getter（NULL = 不可读）
    GDBusPropertySetter set;       // Setter（NULL = 只读）
    GDBusPropertyExists exists;    // 条件存在性判断
    GDBusPropertyFlags flags;
};
```

定义示例：

```c
static const GDBusPropertyTable adapter_properties[] = {
    { "Address",     "s",  property_get_address },                          // 只读
    { "Powered",     "b",  property_get_powered, property_set_powered },    // 可读写
    { "Modalias",    "s",  property_get_modalias, NULL,                     // 条件存在
                      property_exists_modalias },
    { }
};
```

**关键：属性表只存函数指针，不存值。** 真正的值在 `struct btd_adapter` 里：

```c
struct btd_adapter {
    bdaddr_t bdaddr;      // ← "Address" 的数据源
    char *name;           // ← "Name" 的数据源
    gboolean powered;     // ← "Powered" 的数据源
    ...
};
```

Getter 就是个桥接函数：从业务对象取值 → 写进 D-Bus 消息：

```c
static gboolean property_get_address(const GDBusPropertyTable *property,
                                      DBusMessageIter *iter, void *data)
{
    struct btd_adapter *adapter = data;    // data = user_data
    char addr[18];
    ba2str(&adapter->bdaddr, addr);        // 从 adapter 读真实值
    dbus_message_iter_append_basic(iter, DBUS_TYPE_STRING, &addr);
    return TRUE;
}
```

外部只能通过 Property.Get / Property.Set → 属性表函数指针 来读写。无法绕过。

**⑦ user_data**：透传给所有回调的上下文指针。在方法处理函数中取回：

```c
static DBusMessage *start_discovery(DBusConnection *conn,
                                     DBusMessage *msg, void *user_data)
{
    struct btd_adapter *adapter = user_data;   // 直接拿到业务对象
    ...
}
```

**⑧ destroy**：接口注销时调用，用于释放 user_data。

### 注册时框架自动做的事

1. 调用 `object_path_ref()`：如果该 path 未注册，创建 `generic_data`，注册到 D-Bus connection，自动添加 **Introspectable** 接口
2. 调用 `add_interface()`：将接口加入 `data->interfaces`，如果有 root 则加入 `data->added`，通过 `g_idle_add(process_changes)` 延迟发 **InterfacesAdded** 信号
3. 如果 properties 非 NULL，自动添加 **org.freedesktop.DBus.Properties** 接口（Get / Set / GetAll / PropertiesChanged）

---

## 六、Method 调用分发流程

D-Bus 消息到达时，`generic_message()` 作为 `DBusObjectPathVTable.message_function` 被回调：

```c
// object.c:1126
static DBusHandlerResult generic_message(DBusConnection *connection,
                DBusMessage *message, void *user_data)
{
    struct generic_data *data = user_data;
    struct interface_data *iface;
    const GDBusMethodTable *method;
    const char *interface;

    // D-Bus 消息包含四个关键字段:
    //   destination: org.bluez  (服务名)
    //   path:        /org/bluez/hci0  (对象路径 --- 必填)
    //   interface:   org.bluez.Adapter1  (接口名 --- 协议上可选，框架强制要求)
    //   member:      StartDiscovery  (方法名 --- 必填)

    interface = dbus_message_get_interface(message);

    // 1. 按 interface 名找到 interface_data
    iface = find_interface(data->interfaces, interface);
    if (iface == NULL)
        return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

    // 2. 遍历 methods[] 匹配 method name + 参数签名
    for (method = iface->methods; method && method->name && method->function;
              method++) {
        if (!dbus_message_is_method_call(message, iface->name, method->name))
            continue;
        if (!g_dbus_args_have_signature(method->in_args, message))
            continue;

        // 3. 检查 experimental/testing 标记
        if (check_experimental(...))
            return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;

        // 4. PolicyKit 权限检查
        if (check_privilege(connection, message, method, iface->user_data))
            return DBUS_HANDLER_RESULT_HANDLED;

        // 5. 调用实际的方法函数
        return process_message(connection, message, method, iface->user_data);
    }
    return DBUS_HANDLER_RESULT_NOT_YET_HANDLED;
}
```

调用时必须指定 path 和 interface。interface 在 D-Bus 协议上可选，但 bluetoothd 框架强制要求匹配，否则直接返回 `NOT_YET_HANDLED`。

---

## 七、信号：服务端主动通知

### 7.1 三种通信方式对比

| 方式 | 发起方 | 例子 |
|------|--------|------|
| **Method Call** | 客户端 → 服务端 | `StartDiscovery` 让 daemon 开始扫描 |
| **Method Return** | 服务端 → 客户端 | 返回调用结果 |
| **Signal** | 服务端 → 所有客户端 | `PropertiesChanged` daemon 通知"Powered 变了" |

### 7.2 最常用：PropertiesChanged

当适配器状态变化时，模块调用：

```c
// adapter.c:456 — Class of Device 变化
g_dbus_emit_property_changed(dbus_conn, adapter->path,
                              ADAPTER_INTERFACE, "Class");
```

框架自动组装并发送：

```
Signal: org.freedesktop.DBus.Properties.PropertiesChanged
参数:
  "org.bluez.Adapter1",       ← 哪个 interface
  {"Class": <新值>},           ← 哪些属性变了
  []                           ← 哪些属性失效了
```

`g_dbus_emit_property_changed` 会先校验 signal 表中是否注册了该信号，签名是否匹配。

### 7.3 自定义信号（如 ObjectManager）

```c
static const GDBusSignalTable manager_signals[] = {
    { GDBUS_SIGNAL("InterfacesAdded",
        GDBUS_ARGS({ "object", "o" }, { "interfaces", "a{sa{sv}}" })) },
    { GDBUS_SIGNAL("InterfacesRemoved",
        GDBUS_ARGS({ "object", "o" }, { "interfaces", "as" })) },
    { }
};
```

客户端不用轮询 Introspect，直接收信号就知道新对象添加/移除了。

---

## 八、Property 变更的批量合并

`g_dbus_emit_property_changed()` 采用延迟合并策略，防止同一个 mainloop 迭代内多次属性变更发多条信号：

```
1. 变更的 property 加入 iface->pending_prop 链表
2. data->pending_prop = TRUE
3. g_idle_add(process_changes, data) → 延迟到当前事件循环结束时处理
4. process_changes() 将同一 interface 的所有属性变更合并为一条 PropertiesChanged 信号
```

支持 `G_DBUS_PROPERTY_CHANGED_FLAG_FLUSH` 标志立即发送（跳过合并）。

`g_dbus_send_message` 在发信号前也会先 flush 所有 pending 的 ObjectManager 信号（InterfacesAdded/Removed），保证消息顺序。

---

## 九、信号发送前的校验

```c
// object.c:1750
gboolean g_dbus_emit_signal_valist(DBusConnection *connection,
                const char *path, const char *interface,
                const char *name, int type, va_list args)
{
    // 1. 校验 path 已注册
    // 2. 校验 interface 存在于该 path
    // 3. 校验 signal name 在信号表中注册过
    // 4. 校验参数签名与注册时的 args 一致
    if (!check_signal(connection, path, interface, name, &args_info))
        return FALSE;

    signal = dbus_message_new_signal(path, interface, name);
    dbus_message_append_args_valist(signal, type, args);
    return g_dbus_send_message(connection, signal);
}
```

---

## 十、Client/Proxy (gdbus/client.c)

bluetoothd 作为客户端调用外部 D-Bus 服务时使用：

```c
GDBusClient *client = g_dbus_client_new(connection, "org.example", "/");
GDBusProxy *proxy = g_dbus_proxy_new(client, "/org/example/obj", "org.example.Iface1");

// 读属性（从缓存读）
g_dbus_proxy_get_property(proxy, "SomeProperty", &iter);

// 调方法
g_dbus_proxy_method_call(proxy, "SomeMethod", setup_func, return_func, data, destroy);

// 监听属性变化
g_dbus_proxy_set_property_watch(proxy, property_changed_cb, data);
```

---

## 十一、Watch 监听 (gdbus/watch.c)

```c
// 监听服务上线/下线（跟踪 NameOwner 变化）
g_dbus_add_service_watch(conn, "org.example", on_connect, on_disconnect, data, free);

// 监听特定信号（使用 D-Bus filter 匹配规则）
g_dbus_add_signal_watch(conn, sender, path, interface, member, callback, data, free);

// 监听属性变更（帮你在 PropertiesChanged 上建 filter）
g_dbus_add_properties_watch(conn, sender, path, interface, callback, data, free);
```

---

## 十二、Security / PolicyKit (gdbus/polkit.c)

方法注册时可指定 `privilege` 字段：

```c
// check_privilege() 根据 method->privilege 匹配 security_table
// 调用 PolicyKit 进行授权检查，通过后才执行 method->function
```

---

## 十三、启动流程 (main.c)

```c
main()
├── init_defaults()            // 默认配置
├── mainloop_init()            // 初始化 GLib 主循环
├── load_config() + parse_config()  // 加载 /etc/bluetooth/main.conf
├── connect_dbus()
│    ├── g_dbus_setup_bus(SYSTEM, "org.bluez")
│    │    ├── dbus_bus_get(DBUS_BUS_SYSTEM)
│    │    ├── g_dbus_request_name("org.bluez")
│    │    └── setup_dbus_with_main_loop()
│    ├── g_dbus_attach_object_manager()      // 注册 ObjectManager at "/"
│    └── g_dbus_set_debug()
├── g_dbus_set_flags(EXPERIMENTAL|TESTING)
├── adapter_init()              // 发现适配器 → 注册 Adapter1
├── btd_device_init()           // 设备管理
├── btd_agent_init()            // 注册 AgentManager1 at "/org/bluez"
├── btd_profile_init()          // 注册 ProfileManager1 at "/org/bluez"
├── plugin_init()               // 加载插件（可能注册更多接口）
└── mainloop_run_with_signal()  // 进入事件循环
```

---

## 十四、完整 D-Bus 清单

### 固定路径

| Path | Interface | 说明 |
|------|-----------|------|
| `/` | `org.freedesktop.DBus.ObjectManager` | 对象树管理器 |
| `/org/bluez` | `org.bluez.AgentManager1` | 配对代理管理 |
| `/org/bluez` | `org.bluez.ProfileManager1` | 传统 Profile 管理 |

### 适配器级（/org/bluez/hci\<N\>）

| Interface | 说明 |
|-----------|------|
| `org.bluez.Adapter1` | 核心适配器接口 |
| `org.bluez.GattManager1` | 本地 GATT 服务注册 |
| `org.bluez.LEAdvertisingManager1` | LE 广播管理 |
| `org.bluez.AdvertisementMonitorManager1` | 广播监控 (exp.) |
| `org.bluez.BatteryProviderManager1` | 电池信息管理 |
| `org.bluez.Media1` | 媒体传输管理 (audio) |
| `org.bluez.NetworkServer1` | PAN 服务 (network) |
| `org.bluez.AdminPolicySet1` | 管理策略 (plugin) |
| `org.bluez.AdminPolicyStatus1` | 策略状态 (plugin) |

### 设备级（/org/bluez/hci\<N\>/dev_XX...）

| Interface | 说明 |
|-----------|------|
| `org.bluez.Device1` | 核心设备接口 |
| `org.bluez.DeviceSet1` | 设备集合 |
| `org.bluez.Battery1` | 设备电池 |
| `org.bluez.BatteryProvider1` | 电池提供者 |
| `org.bluez.Bearer.BREDR1` | BR/EDR 承载 |
| `org.bluez.Bearer.LE1` | LE 承载 |
| `org.bluez.MediaTransport1` | 媒体传输 |
| `org.bluez.MediaEndpoint1` | 媒体端点 (BAP) |
| `org.bluez.MediaControl1` | 媒体控制 (AVRCP) |
| `org.bluez.MediaAssistant1` | 广播助手 (BASS) |
| `org.bluez.MediaPlayer1` | 媒体播放器 |
| `org.bluez.MediaItem1` | 媒体项 |
| `org.bluez.MediaFolder1` | 媒体目录 |
| `org.bluez.Telephony1` | 电话接口 (HFP) |
| `org.bluez.Call1` | 通话接口 |
| `org.bluez.Network1` | 网络连接 (PAN) |
| `org.bluez.Input1` | 输入设备 (HID) |

### GATT 服务（本地 / 远端）

| Interface | 说明 |
|-----------|------|
| `org.bluez.GattService1` | GATT 服务 |
| `org.bluez.GattCharacteristic1` | GATT 特征 |
| `org.bluez.GattDescriptor1` | GATT 描述符 |

### LE 广播

| Interface | 说明 |
|-----------|------|
| `org.bluez.LEAdvertisement1` | LE 广播实例 |
| `org.bluez.GattProfile1` | GATT Profile 广播 |

### 广告监控 (experimental)

| Interface | 说明 |
|-----------|------|
| `org.bluez.AdvertisementMonitor1` | 广告监控实例 |

### 标准 D-Bus 接口（框架自动生成）

| Interface | 说明 |
|-----------|------|
| `org.freedesktop.DBus.Introspectable` | 内省，每个 object path 自动注册 |
| `org.freedesktop.DBus.Properties` | 属性读写，有属性表时自动注册 |

### Profile/Agent（外部注册到 bluetoothd）

| Interface | 说明 |
|-----------|------|
| `org.bluez.Profile1` | 传统 Profile 回调 |
| `org.bluez.Agent1` | 配对代理 |
