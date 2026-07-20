# BlueZ D-Bus 开发入门指南

> 基于 BlueZ 源码 + busctl 实战，从零理解 BlueZ 的 D-Bus 架构。

---

## 目录

1. [D-Bus 基础概念](#1-d-bus-基础概念)
2. [CLI 工具三板斧](#2-cli-工具三板斧)
3. [D-Bus 数据格式（类型签名）](#3-d-bus-数据格式类型签名)
4. [调用方法、读写属性、订阅信号](#4-调用方法读写属性订阅信号)
5. [BlueZ 的 D-Bus 连接](#5-bluez-的-d-bus-连接)
6. [BlueZ 的对象和接口全览](#6-bluez-的对象和接口全览)
7. [对象详解：路径 + 接口 + 子对象](#7-对象详解路径--接口--子对象)
8. [接口 = 方法 + 属性 + 信号](#8-接口--方法--属性--信号)
9. [信号的构成](#9-信号的构成)
10. [OPP 服务启动全链路](#10-opp-服务启动全链路)
11. [PipeWire 怎么连接和监听 BlueZ](#11-pipewire-怎么连接和监听-bluez)
12. [架构总结](#12-架构总结)

---

## 1. D-Bus 基础概念

D-Bus 是 Linux 进程间通信（IPC）的消息总线，Core 概念和面向对象编程对应：

| D-Bus 概念 | 编程类比 | BlueZ 实例 |
|---|---|---|
| **Bus** | 网络/总线 | System Bus (`busctl --system`) |
| **Bus Name** | 进程名字 | `org.bluez` |
| **Object Path** | 对象指针 | `/org/bluez/hci0` |
| **Interface** | 接口/协议 | `org.bluez.Adapter1` |
| **Method** | 函数调用 | `StartDiscovery()` |
| **Property** | 成员变量 | `Powered`, `Address` |
| **Signal** | 事件通知 | `PropertiesChanged`, `InterfacesAdded` |

两种总线：
- **System Bus**：系统服务（BlueZ、NetworkManager、PipeWire 的 System Bus 连接）
- **Session Bus**：用户会话服务（桌面通知、obexd）

---

## 2. CLI 工具三板斧

三个工具互补，各管一个维度：

| 命令 | 看什么 |
|---|---|
| `busctl tree <服务>` | 对象的**层级关系**（父子） |
| `busctl introspect <服务> <路径>` | 该对象的**接口列表**（方法+属性+信号） |
| `busctl monitor <服务>` | **运行时**信号和方法调用（实时） |

### 常用命令速查

```bash
# 发现
busctl --system list                           # System Bus 上所有服务
busctl --user list                             # Session Bus 上所有服务
busctl tree org.bluez                          # BlueZ 的对象树

# 查看接口
busctl introspect org.bluez /org/bluez/hci0    # Adapter 的所有接口
busctl introspect org.bluez /                  # ObjectManager（InterfacesAdded/Removed 在这）

# 读属性
busctl get-property org.bluez /org/bluez/hci0 org.bluez.Adapter1 Powered
busctl get-property org.bluez /org/bluez/hci0/dev_XX... org.bluez.Device1 Name

# 调方法
busctl call org.bluez /org/bluez/hci0 org.bluez.Adapter1 StartDiscovery
busctl call org.bluez /org/bluez/hci0 org.bluez.Adapter1 RemoveDevice o /org/bluez/hci0/dev_XX...

# 监听
busctl monitor org.bluez                        # 所有信号
busctl monitor org.bluez | grep PropertiesChanged  # 只看属性变化
```

### 注意区分

```bash
busctl list              # 默认 Session Bus
busctl --system list     # 明确指定 System Bus
busctl --system          # 等价于 busctl --system list
```

---

## 3. D-Bus 数据格式（类型签名）

D-Bus 用单字符表示类型，组合起来叫签名（signature）。

### 基础类型

| 签名 | 类型 | 例 |
|---|---|---|
| `b` | boolean | `true` |
| `y` | byte | `12` |
| `n` | int16 | `-42` |
| `q` | uint16 | `65535` |
| `i` | int32 | `-100` |
| `u` | uint32 | `7995916` |
| `x` | int64 | 大整数 |
| `t` | uint64 | 大整数 |
| `d` | double | `3.14` |
| `s` | string | `"hello"` |
| `o` | object path | `/org/bluez/hci0` |
| `g` | signature | `a{sv}` |
| `h` | unix fd | 传 socket 用 |

### 复合类型

| 签名 | 含义 | 例 |
|---|---|---|
| `as` | 字符串数组 | `["A2DP", "HFP"]` |
| `ay` | 字节数组 | `[0x00, 0x1A, ...]` |
| `a{sv}` | 字典：string→variant | `{"Powered": true}` |
| `a{oa{sa{sv}}}` | ObjectManager 返回类型 | path→interface→props |
| `(su)` | 结构体 | `("hello", 42)` |
| `v` | variant（任意类型） | 可以是任何类型 |

### busctl 调方法时的格式

```bash
# 无参数方法
busctl call org.bluez /org/bluez/hci0 org.bluez.Adapter1 StartDiscovery

# 有参数：签名 + 值1 + 值2 + ...
busctl call org.bluez /org/bluez/hci0 org.bluez.Adapter1 RemoveDevice \
    o /org/bluez/hci0/dev_XX...

busctl call org.bluez /org/bluez/hci0 org.freedesktop.DBus.Properties Get \
    ss org.bluez.Adapter1 Powered

# 字典参数用 JSON 格式
busctl call org.bluez /org/bluez/hci0 org.bluez.Adapter1 SetDiscoveryFilter \
    'a{sv}' '{"Transport":"le","RSSI":-80}'
```

---

## 4. 调用方法、读写属性、订阅信号

### 调方法

```bash
busctl call <服务名> <对象路径> <接口名> <方法名> <签名> <参数...>
```

### 读属性

```bash
# 直接读（推荐）
busctl get-property <服务> <路径> <接口> <属性名>

# 底层方式
busctl call <服务> <路径> org.freedesktop.DBus.Properties Get ss <接口> <属性名>

# 读全部属性
busctl call <服务> <路径> org.freedesktop.DBus.Properties GetAll s <接口>
```

### 写属性

属性必须标记 `writable` 才能写（看 introspect 的 FLAGS 列）：

```bash
# 直接写（推荐）
busctl set-property org.bluez /org/bluez/hci0 org.bluez.Adapter1 Powered b false

# 底层方式
busctl call org.bluez /org/bluez/hci0 org.freedesktop.DBus.Properties \
    Set ssv org.bluez.Adapter1 Powered b false
```

### 订阅信号

```bash
# 实时监听所有信号
busctl monitor org.bluez

# 用 match rule 过滤
busctl monitor --match="type='signal',interface='org.freedesktop.DBus.ObjectManager',member='InterfacesAdded'" org.bluez
busctl monitor --match="type='signal',path='/org/bluez/hci0'" org.bluez
```

**关键**：信号是广播，发送方不关心谁在听，订阅方通过 match rule 告诉 dbus-daemon 要哪些。

### 方法 vs 信号

| | 方法 | 信号 |
|---|---|---|
| 方向 | Client → Server | Server → 所有监听者 |
| 有没有返回值 | 有 | 没有 |
| 谁主动 | 调用方 | 发送方 |

---

## 5. BlueZ 的 D-Bus 连接

BlueZ 生态有 **3 个进程、5 条 D-Bus 连接**：

| # | 谁 | 什么总线 | Bus Name | 作用 |
|---|---|---|---|---|
| ① | bluetoothd | System Bus | `org.bluez` | 蓝牙核心 API |
| ② | obexd | Session Bus | `org.bluez.obex` | 文件传输管理 |
| ③ | obexd bt plugin | System Bus | *(无/private)* | 跨总线跟 BlueZ 通信 |
| ④ | bluetooth-meshd | System Bus | `org.bluez.mesh` | Mesh 网络 |
| ⑤ | bluetoothd | *(非 D-Bus)* | N/A | 内核 mgmt socket（直连内核蓝牙子系统） |

**关键设计点**：
- `g_dbus_setup_bus()` — 申请 bus name，对外提供服务
- `g_dbus_setup_private()` — 私有连接，不申请 name，只做客户端/监听
- obexd 需要两条连接是因为主连接在 Session Bus，但 BlueZ 在 System Bus，必须额外建一条私有连接跨总线通信

### 获取连接的 API

```c
// bluetoothd 里
DBusConnection *conn = btd_get_dbus_connection();  // 主连接单例

// obexd 里
DBusConnection *conn = obex_setup_dbus_connection(OBEXD_SERVICE, &err);
DBusConnection *priv = g_dbus_setup_private(DBUS_BUS_SYSTEM, NULL, NULL);
```

---

## 6. BlueZ 的对象和接口全览

BlueZ 定义了约 **30 个 D-Bus 接口**，分布在 **8 类对象** 上：

```
/                                         ObjectManager（InterfacesAdded/Removed 信号）
│
└─ /org/bluez                             ① 根对象（Manager 集合地）
   │  接口: AgentManager1, ProfileManager1, HealthManager1
   │
   ├─ /org/bluez/hci0                     ② Adapter（一个蓝牙芯片一个）
   │  │  接口: Adapter1, Media1, GattManager1,
   │  │        LEAdvertisingManager1, NetworkServer1, BatteryProviderManager1...
   │  │
   │  ├─ /org/bluez/hci0/dev_XX_XX...    ③ Device（每个远程设备）
   │  │  │  接口: Device1 (+ MediaControl1/Network1/Input1/Battery1)
   │  │  │
   │  │  ├─ .../fdX                       ④ MediaTransport（音频数据通道）
   │  │  │  接口: MediaTransport1           ← PipeWire Acquire() 拿 fd
   │  │  │
   │  │  ├─ .../playerX                   ⑤ MediaPlayer
   │  │  │  接口: MediaPlayer1, MediaFolder1, MediaItem1
   │  │  │
   │  │  ├─ .../serviceXXXX/              ⑥ GATT Service
   │  │  │  └─ .../charYYYY/              ⑦ GATT Characteristic
   │  │  │     └─ .../descZZZZ/           ⑧ GATT Descriptor
```

### 完整接口列表

**核心管理类**（在 `/org/bluez`）：
`AgentManager1`, `ProfileManager1`, `HealthManager1`

**Adapter 相关**（在 `/org/bluez/hciX`）：
`Adapter1`, `Media1`, `GattManager1`, `LEAdvertisingManager1`,
`AdvertisementMonitorManager1`, `BatteryProviderManager1`, `NetworkServer1`

**Device 相关**（在 `.../dev_XX...`）：
`Device1`, `MediaControl1`, `Network1`, `Input1`, `Battery1`

**音频类**（在 device 子路径下）：
`MediaTransport1`, `MediaPlayer1`, `MediaFolder1`, `MediaItem1`

**BLE GATT 类**（在 device 子路径下）：
`GattService1`, `GattCharacteristic1`, `GattDescriptor1`, `GattProfile1`

**BLE 广播**（在 adapter 下）：
`LEAdvertisement1`, `AdvertisementMonitor1`

**其他**：
`Profile1`, `MediaEndpoint1`, `Agent1`, `BatteryProvider1`,
`Bearer.BREDR1`, `Bearer.LE1`, `DeviceSet1`, `Call1`, `Telephony1`

### Manager 模式

所有带 `Manager` 的接口都是同一个模式——外部程序注册对象进来，BlueZ 后续回调：

| Manager | 谁注册 | BlueZ 回调什么 |
|---|---|---|
| `Media1` | PipeWire | `SelectConfiguration()` / `SetConfiguration()` |
| `GattManager1` | 外部 BLE 应用 | 远程设备读写 GATT 时回调 |
| `LEAdvertisingManager1` | 外部应用 | 广播状态变化 |
| `ProfileManager1` | obexd | `NewConnection(fd)` |
| `AgentManager1` | 桌面环境 | 配对时弹 PIN 对话框 |

---

## 7. 对象详解：路径 + 接口 + 子对象

一个 D-Bus 对象由三部分构成：

```
/org/bluez/hci0/dev_XX.../fd0
┌─────────────────────────────┐
│ ① 对象路径：                │
│    /org/bluez/hci0/.../fd0  │
│                             │
│ ② 接口列表（方法+属性+信号）：│
│    org.bluez.MediaTransport1 │
│    org.freedesktop.DBus.Properties
│                             │
│ ③ 子对象（可能有或没有）：    │
│    (Transport 是叶子节点)    │
└─────────────────────────────┘
```

- `busctl tree` 看 ①②（层级关系）
- `busctl introspect` 看 ③（接口细节）

父对象通过属性引用子对象，子对象通过属性引用父对象：
```bash
# Device 对象的 .Adapter 属性指向父 Adapter
busctl introspect org.bluez /org/bluez/hci0/dev_XX... | grep Adapter
# .Adapter  property  o  "/org/bluez/hci0"

# MediaControl 的 .Player 属性指向子 MediaPlayer
# .Player   property  o  "/org/bluez/hci0/dev_XX.../player0"
```

对象树是**动态**的——设备连接时通过 `InterfacesAdded` 信号生长，断开时通过 `InterfacesRemoved` 剪枝。

---

## 8. 接口 = 方法 + 属性 + 信号

introspect 输出的 TYPE 列只有四种值：

```
org.bluez.Media1                    interface   -      -           ← 接口（容器）
.RegisterEndpoint                   method      oa{sv} -           ← 方法
.SupportedUUIDs                     property    as     [...]        ← 属性
.PropertiesChanged                  signal      sa{sv}as -          ← 信号
```

对应到源码就是 gdbus 的三张表：

```c
// 接口 = 三张注册表
g_dbus_register_interface(conn, path, interface_name,
    methods,      // GDBusMethodTable[]     ← 方法表
    signals,      // GDBusSignalTable[]     ← 信号表
    properties,   // GDBusPropertyTable[]   ← 属性表
    user_data, destroy);
```

### 方法表

```c
static const GDBusMethodTable media_methods[] = {
    { GDBUS_METHOD("RegisterEndpoint",
        GDBUS_ARGS({ "endpoint",   "o" },        // 输入参数
                    { "properties", "a{sv}" }),   // 输入参数
        NULL,                                     // 无返回值
        register_endpoint) },                     // C 回调函数
    { }  // 哨兵
};
```

### 属性表

```c
static const GDBusPropertyTable adapter_properties[] = {
    { "Powered",  "b", property_get_powered, property_set_powered },  // 可读写
    { "Address",  "s", property_get_address },                         // 只读
    { }
};
```

### 信号表

```c
static const GDBusSignalTable manager_signals[] = {
    { GDBUS_SIGNAL("InterfacesAdded",
        GDBUS_ARGS({ "object",     "o" },
                    { "interfaces", "a{sa{sv}}" })) },
    { GDBUS_SIGNAL("InterfacesRemoved",
        GDBUS_ARGS({ "object",     "o" },
                    { "interfaces", "as" })) },
    { }
};
```

---

## 9. 信号的构成

信号 = **信号名** + **参数签名** + **运行时数据**。没有返回值（是通知不是请求）。

BlueZ 总共有 **4 类信号**：

| 信号 | 所属接口 | 哪个路径 | 签名 | 触发时机 |
|---|---|---|---|---|
| `InterfacesAdded` | ObjectManager | `/` | `oa{sa{sv}}` | 新对象出现 |
| `InterfacesRemoved` | ObjectManager | `/` | `oas` | 对象消失 |
| `PropertiesChanged` | Properties | **所有对象** | `sa{sv}as` | 任何属性变化 |
| `Disconnected` | Device1/Bearer1 | device/bearer | `ss` | 意外断开 |

### PropertiesChanged 参数详解

```
信号: PropertiesChanged
参数:
  ├─ 参数1: s  "org.bluez.Adapter1"          ← 哪个接口的属性变了
  ├─ 参数2: a{sv} {"Powered": b false}        ← 变了哪些属性 + 新值
  └─ 参数3: as   []                            ← 失效属性（通常为空）
```

### 设计原则

BlueZ 故意不用很多自定义信号——绝大部分状态变化走 `PropertiesChanged`。外部程序只需要订阅 ObjectManager 的三条信号（`InterfacesAdded`、`InterfacesRemoved`、`PropertiesChanged`）就能感知一切变化，无需知道 BlueZ 的每个自定义信号名。

---

## 10. OPP 服务启动全链路

OPP（Object Push Profile，蓝牙文件传输）是 obexd 的一个**内置插件**，不是独立进程。

### 启动流程

```
编译期                      obexd 启动                      蓝牙连接
───────                     ──────────                      ──────────

Makefile.obexd               main.c:main()
  obexd_builtin_modules        ├─ manager_init()
  += opp                       │   注册 obexd 到 D-Bus
  │                            │
  ▼                            ├─ plugin_init()
genbuiltin 脚本                │   遍历 __obex_builtin[]
  生成 builtin.h:              │     └─ opp_init()
  __obex_builtin_opp           │        └─ obex_service_driver_register()
  │                            │           把 OPP driver 加入全局链表
  ▼                            │
OBEX_PLUGIN_DEFINE(            ├─ obex_server_init()
  opp, opp_init, opp_exit)     │   遍历所有 service driver
                               │     └─ init_server(OPP)
                               │        └─ transport->start()
                               │           └─ bluetooth_start()
                               │               OPP → UUID 映射
                               │               │
                               │           bluetooth_init()
                               │            (logind 激活时)
                               │               └─ g_dbus_add_service_watch("org.bluez")
                               │                  BlueZ上线时：
                               │                  └─ 注册 Profile1 接口
                               │                     └─ 调用 BlueZ RegisterProfile()
                               │                        将 OPP UUID 注册到内核 SDP
                               │
                               └─ g_main_loop_run()     远程手机发起 OPP 连接：
                                                            └─ NewConnection(fd)
                                                               → opp_connect()
                                                               → opp_chkput()
                                                               → opp_put()
```

### 关键设计点

| 层级 | 组件 | 作用 |
|---|---|---|
| **编译期** | `genbuiltin` + `OBEX_PLUGIN_DEFINE` | 生成 `__obex_builtin[]` 数组，把 OPP 编入 obexd |
| **插件加载** | `plugin_init()` → `opp_init()` | 把 `obex_service_driver` 注册到全局链表 |
| **服务启动** | `obex_server_init()` → `bluetooth_start()` | 为 OPP 创建 profile，映射 `OBEX_OPP(0x0002)` → UUID `00001105-...` |
| **BlueZ 协调** | `name_acquired()` → `RegisterProfile()` | 让内核 SDP 广播 OPP 服务 |
| **入站连接** | `NewConnection(fd)` → `opp_connect()` 等 | 处理文件推送 |

**核心**：obexd 是 OBEX 通用框架，OPP/FTP/PBAP/MAP 都是它的插件。每个插件只实现 `obex_service_driver` 的 7 个回调，框架处理 D-Bus 注册、SDP 广播、会话管理、传输进度通知等通用逻辑。

---

## 11. PipeWire 怎么连接和监听 BlueZ

### PipeWire 是双向的

```
PipeWire 监听 BlueZ              BlueZ 调用 PipeWire
──────────────────              ────────────────────
订阅 ObjectManager              调用 MediaEndpoint1
InterfacesAdded/Removed          SelectConfiguration()
                                SetConfiguration()
```

### PipeWire 关注的 BlueZ 对象

```
对象 ①: / (ObjectManager)
  用途: 发现所有对象 + 监听增删
  信号: InterfacesAdded, InterfacesRemoved

对象 ②: /org/bluez/hci0 (Adapter)
  org.bluez.Adapter1 → 读 Powered/Address
  org.bluez.Media1   → 调 RegisterEndpoint() 注册自己

对象 ③: /org/bluez/hci0/dev_XX... (Device)
  org.bluez.Device1       → 读 Name, Paired, Connected, UUIDs
  org.bluez.MediaControl1 → Play/Pause/Volume

对象 ④: .../fdX (MediaTransport)
  org.bluez.MediaTransport1 → 读 Codec/Configuration
                             Acquire() 拿 fd 传输音频
```

### 完整时序

```
PipeWire                         BlueZ
───────                          ─────
  │ ① GetManagedObjects() ──────→│  获取初始状态
  │ ② 订阅 InterfacesAdded/Removed
  │ ③ RegisterApplication() ────→│  告诉 BlueZ 自己有哪些 endpoint
  │                               │  ④ 创建 GDBusClient 发现 PipeWire
  │                               │     的 MediaEndpoint1 对象
  │                               │
  ════════ 蓝牙耳机连上了 ════════
  │                               │
  │ ⑤ InterfacesAdded 信号 ←─────│  耳机 Device 对象出现
  │ ⑥ InterfacesAdded 信号 ←─────│  MediaTransport 对象出现
  │                               │
  │ ⑦ SelectConfiguration() ←────│  协商 codec（SBC/AAC/LDAC?）
  │    PipeWire 选出最佳配置 ────→│
  │                               │
  │ ⑧ SetConfiguration() ←───────│  配置确定后
  │                               │
  │ ⑨ Acquire() 拿到 fd ────────→│  开始收/发音频 PCM
```

### PipeWire 暴露给 BlueZ 的接口

PipeWire 是 D-Bus **双角色**——既是 Client 也是 Server：

```
PipeWire 作为 Client:               PipeWire 作为 Server:
  调用 BlueZ 的:                      BlueZ 调用 PipeWire 的:
    GetManagedObjects()                 MediaEndpoint1.SelectConfiguration()
    RegisterEndpoint()                  MediaEndpoint1.SetConfiguration()
    RegisterApplication()               MediaEndpoint1.ClearConfiguration()
    Acquire()                           MediaEndpoint1.Release()
```

这和 obexd 注册 `Profile1` 给 BlueZ 调用的模式完全一样——BlueZ 的架构哲学是**核心做蓝牙协议，具体策略外包给外部程序**。音频交给 PipeWire，文件交给 obexd，电池交给 upower，配对 UI 交给桌面环境。

---

## 12. 架构总结

### BlueZ 源代码对应关系

| runtime 看到的 | 源码里的位置 |
|---|---|
| `org.bluez` bus name | `src/main.c:58` — `#define BLUEZ_NAME "org.bluez"` |
| `/org/bluez/hci0` 对象 | `src/adapter.c:9387` — `g_dbus_register_interface(...)` |
| `Adapter1` 接口 | `src/adapter.h:21` — `#define ADAPTER_INTERFACE "org.bluez.Adapter1"` |
| `StartDiscovery` 方法 | `src/adapter.c:3945` — `adapter_methods[]` |
| `Powered` 属性 | `src/adapter.c:3969` — `adapter_properties[]` |
| ObjectManager 信号 | `gdbus/object.c:1264` — `manager_signals[]` |
| gdbus 封装层 | `gdbus/gdbus.h` — `GDBusMethodTable` 等结构体 |
| `btd_get_dbus_connection()` | `src/dbus-common.c:48` — 全局单例获取 |

### 学习路径建议

```
第一步: busctl tree + introspect → 理解静态结构
第二步: busctl monitor → 观察运行时行为
第三步: 对照源码 → 理解每个接口的实现
第四步: 写代码调用 → 真正掌握
```

### D-Bus 设计原则

1. **面向对象模型**：路径 = 对象，接口 = 协议，方法 = 调用，属性 = 状态，信号 = 事件
2. **ObjectManager 模式**：`GetManagedObjects()` 一次性获取 + `InterfacesAdded/Removed` 增量更新
3. **Manager + 回调模式**：外部程序 Register → BlueZ 存引用 → 事件发生时回调
4. **属性变化都走 PropertiesChanged**：减少自定义信号数量，统一订阅方式
5. **fd-passing**：音频/文件数据不经过 D-Bus，只传 fd（如 `MediaTransport1.Acquire()`）
