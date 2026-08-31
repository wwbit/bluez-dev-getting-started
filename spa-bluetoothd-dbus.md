# bluetoothd ↔ PipeWire SPA (bluez5) D-Bus 交互

> 范围:BlueZ bluetoothd 与 PipeWire `spa/plugins/bluez5` 插件经 **system bus** 的 D-Bus 交互。
> 覆盖 A2DP Source/Sink、HFP/HSP AG/HF。
> 代码引用:`bluez/` = 本仓库;`pipewire/` = `/home/jinwli/jinwli/upstream/pipewire`(行号以该 checkout 为准)。

---

## 0. 一句话总览

bluetoothd 在 system bus 上以 well-known 名 `org.bluez` 提供媒体服务;
PipeWire 的 bluez5 插件是同一条总线上的**无名字客户端**(只有唯一名 `:1.xx`)。
SPA 调 bluetoothd 的接口做**注册与控制**(RegisterApplication / RegisterProfile / Acquire / Release),
bluetoothd 回调 SPA 导出的对象做**协商与交付**(SelectConfiguration / SetConfiguration / ClearConfiguration / Profile1.NewConnection),
音频数据则通过 **fd passing** 交给 SPA,不经 D-Bus 消息。

```
                system bus
 ┌──────────────┐                    ┌─────────────────┐
 │ bluetoothd   │                    │ pipewire (bluez5)│
 │ :1.9         │◄──定向 method call──│ :1.47 (唯一名)    │
 │ org.bluez    │──定向 method call──►│ (无 well-known 名)│
 │ 导出: Media1 │                    │ 导出: MediaEndpoint1
 │  ProfileManager1                  │       Profile1
 │  MediaTransport1                  │       ObjectManager
 └──────┬───────┘                    └────────┬────────┘
        │ 内核 (HCI/L2CAP/SCO)                │
   ◄════╧════════════ 蓝牙链路 ════════════════╧════►  对端设备(耳机/手机)
```

---

## 1. 概念基础(先看这里)

### 1.1 两条独立的 D-Bus 连接,靠「唯一名 + 对象路径」寻址

- **bluetoothd**:系统总线上的 well-known 名 `org.bluez`,路径 `/org/bluez` 起。
- **bluez5 SPA 插件**:连接由宿主进程(PipeWire/会话管理器)经 `support.dbus` 提供(`bluez5-dbus.c:25`),**不请求 well-known 名**,只有总线分配的**唯一名** `:1.xx`。它把 `MediaEndpoint1` / `Profile1` 对象注册在**自己这条连接**上。
- bluetoothd 在收到注册消息时用 `dbus_message_get_sender(msg)` 记下这个唯一名,之后所有回调都以它为 destination 定向发送 —— 总线守护进程按"谁注册了该对象路径"路由到 PipeWire 的连接。
- 唯一名以 `:` 开头,gdbus 据此直接 watch(`gdbus/watch.c:262-265`),无需名字解析。

### 1.2 对象发现:ObjectManager 模式

D-Bus 没有"枚举连接上对象"的原生机制,对象又是动态的(设备、transport、端点随时增删),于是:

| 成员 | 类型 | 作用 |
|---|---|---|
| `GetManagedObjects()` | 方法 | 全量快照 `a{oa{sa{sv}}}`(路径→接口→属性→值) |
| `InterfacesAdded(o, a{sa{sv}})` | 信号 | 新增对象/接口的增量通知 |
| `InterfacesRemoved(o, as)` | 信号 | 对象移除通知 |

标准用法:**先快照、后跟信号**,不用轮询。
bluetoothd 侧由 `GDBusClient` 自动完成(`gdbus/client.c:1456 g_dbus_client_new_full`:挂 NameOwnerChanged watch、订阅两个信号、`AddMatch` 加 `path_namespace` 过滤规则,首轮 `GetManagedObjects` 见 `client.c:1401`);bluetoothd 自己也把 ObjectManager 挂在 `/` 上(`src/main.c:1402`),供 SPA 反向订阅。

### 1.3 fd passing:音频不走 D-Bus

音频是连续高带宽流,D-Bus 消息不适合。交接方式是 **`DBUS_TYPE_UNIX_FD` 的 fd passing**:方法回复/调用参数里携带文件描述符,内核直接复制到对端进程。A2DP 的 fd 是 **AVDTP transport socket**,HFP 的 fd 是 **RFCOMM socket**。

### 1.4 接口清单(谁导出、谁调用)

| 接口 | 导出者 | 调用方向 | 用途 |
|---|---|---|---|
| `org.bluez.Media1` | bluetoothd (`media.c:3519`) | SPA → bluetoothd | RegisterApplication / UnregisterApplication(端点批量注册) |
| `org.bluez.ProfileManager1` | bluetoothd (`src/profile.c:2562`) | SPA → bluetoothd | RegisterProfile / UnregisterProfile(HFP/HSP 业务外包) |
| `org.bluez.MediaTransport1` | bluetoothd (`transport.c:1210`) | SPA → bluetoothd | Acquire / TryAcquire / Release;属性 State、Volume、Codec… |
| `org.bluez.MediaEndpoint1` | SPA (`bluez5-dbus.c:5597`) | bluetoothd → SPA | SelectConfiguration / SetConfiguration / ClearConfiguration / Release |
| `org.bluez.Profile1` | SPA (`backend-native.c:3649`) | bluetoothd → SPA | NewConnection / RequestDisconnection / Release |

---

## 2. 注册与发现(启动期,一次性)

### MSC-1:SPA 上线注册

```
bluez5 SPA (:1.47)                         bluetoothd                           说明
     |                                          |
     | ① Media1.RegisterApplication(/MediaEndpoint, {})
     |----------------------------------------->|  ② register_app (media.c:3449)
     |                                          |      → create_app → GDBusClient:
     |                                          |      · NameOwnerChanged watch(:1.47)
     |                                          |      · InterfacesAdded/Removed watch
     |                                          |      · AddMatch (path_namespace)
     |                                          |  ③ GetManagedObjects → 枚举全部端点
     |                                          |      proxy_added_cb ×N (media.c:3319)
     |                                          |      → media_endpoint{sender,path}
     |                                          |      → 每个端点挂 disconnect watch
     | ④ reply(此时端点已全部就绪)              |      (client_ready_cb, media.c:3274)
     |<-----------------------------------------|
     |                                          |
     | ⑤ ProfileManager1.RegisterProfile(/Profile/HFPAG, UUID=HFP_AG, {Version,Features})
     |----------------------------------------->|  ⑥ ext_profile{owner,path} (src/profile.c:648)
     |                                          |      + disconnect watch (src/profile.c:2532)
     |                                          |      + ext_start_servers: RFCOMM 监听 + SDP 记录
     | ⑦ reply                                 |
     |<-----------------------------------------|
     |  (HSP_AG / HFP_HF / HSP_HS 同理重复⑤)
```

**数据路径(bluetoothd 收到 ① 之后)**:`register_app` → `create_app`(`media.c:3394`)→ `g_dbus_client_new_full(conn, sender, path, path)`(`media.c:3405`)完成全部订阅 → 首轮 `GetManagedObjects` 回复解析出每个 `MediaEndpoint1` proxy → `proxy_added_cb`(`media.c:3319`)→ `app_register_endpoint`(`media.c:2986`,读 UUID/Codec/Capabilities/QoS 属性)→ `media_endpoint_create`(`media.c:1563`,存 `sender`+`path`,挂 disconnect watch)。**RegisterApplication 的 reply 被扣住,直到发现完成才回**(`client_ready_cb`,`media.c:3274-3311`),所以 SPA 的调用返回 = 注册与发现都已生效。

PipeWire 用 raw libdbus 注册对象时**不发** InterfacesAdded(先注册对象、后调 RegisterApplication,靠快照);它的 telephony 后端除外(`telephony.c:1177`)。而 SPA 订阅 bluetoothd 的方向,InterfacesAdded 是主要手段 —— 设备、transport 都是运行时动态出现的。

---

## 3. A2DP(Source 与 Sink 对称)

角色说明:**A2DP Source** 端点 = 本机作音源(PC 放歌给耳机);**A2DP Sink** 端点 = 本机作接收者(手机放歌给 PC)。SPA 两类都注册(`/MediaEndpoint/A2DPSource/<codec>`、`/MediaEndpoint/A2DPSink/<codec>`,`bluez5-dbus.c:488`)。D-Bus 消息序列对两种角色完全一致,差异只在端点 UUID、触发方(Source 场景由 WirePlumber 调 `Device1.ConnectProfile` 本地发起;Sink 场景通常远端发起)和 fd 的读写方向。

### MSC-2:A2DP 流建立 → 播放 → 拆除

```
bluez5 SPA                    bluetoothd                              对端设备 (AVDTP)
     |                            |                                        |
     | (AVDTP 连接建立: 本地或远端发起, SDP 完成)                            |
     |                            |◄──── AVDTP Discover/GetCapabilities ───|
     |                            |   SEP 选择 (a2dp.c:3005/3174)
     | ① SelectConfiguration(远端 capabilities)
     |<---------------------------|
     | ② reply: 选定的 configuration (ay)
     |----------------------------|──── AVDTP SetConfiguration ───────────►|
     |                            |◄──── AVDTP Open (流建立) ──────────────|
     |                            |  ③ 创建 MediaTransport1 (/sepN/fdM)
     |                            |     (media_transport_create, transport.c:2635)
     | ④ SetConfiguration(transport路径, 属性{Device,UUID,Codec,Configuration,State,Volume})
     |<---------------------------|
     |   (SPA: 建 spa_bt_transport, 连接 profile, 建 media-sink/source 节点)
     |                            |
     |   (SPA 订阅 transport 的 PropertiesChanged: State/Volume 变化)
     |                            |
     | ⑤ Acquire()                |
     |--------------------------->|   ⑥ 单 owner 检查 → 流 resume/start
     | ⑦ reply(fd, mtu_r, mtu_w)  |      fd = AVDTP transport socket
     |<---------------------------|      (transport.c:526-530, DBUS_TYPE_UNIX_FD)
     |                            |
     |═════════ 音频数据直接 read/write fd,不经 D-Bus ═════════════════════|
     |                            |
     | ⑧ Release()                |   流 suspend, State → idle
     |--------------------------->|
     |                            |
     | ⑨ ClearConfiguration(transport路径)   (流关闭/断开时)
     |<---------------------------|   销毁 MediaTransport1 对象
```

### 数据路径(逐条)

- **① SelectConfiguration**(`media.c:502`,bluetoothd → SPA,3 秒超时见 `media.c:74/463`):AVDTP SEP 协商时,bluetoothd 把远端 GetCapabilities 的结果原样交给 SPA,SPA 的 `endpoint_select_configuration`(`bluez5-dbus.c:893`)用 codec 插件(`a2dp-codecs.c`)选一个双方兼容的 configuration 回复;bluetoothd 把该配置通过 **AVDTP SetConfiguration** 发给远端。
- **④ SetConfiguration**(`media.c:559`):AVDTP 流配置完成后,bluetoothd **先**创建 `MediaTransport1` 对象,再把路径+属性交给 SPA。SPA 的 `endpoint_set_configuration`(`bluez5-dbus.c:5406`)解析属性、创建内部 transport、把 profile 接入 PipeWire graph(media-sink/media-source 节点)。此后 State/Volume 经 PropertiesChanged 增量推送。
- **⑤ Acquire**(`transport.c:792`):同一时刻只允许一个 owner(`media_owner`,`transport.c:87`,否则 `not_authorized`)。流 resume/start 完成后,回复中带 fd —— A2DP 是 AVDTP stream 的 BT socket(`a2dp_resume_complete`,`transport.c:486-542`);LE Audio/BAP 是 ISO socket。SPA 侧 `transport_acquire_reply`(`bluez5-dbus.c:4263`)取 fd/MTU,**Source 角色向 fd 写编码帧,Sink 角色从 fd 读**。
- **⑧/⑨**:`Release`(`transport.c:899`,仅 owner 可调)挂起流;`ClearConfiguration`(`media.c:351`,fire-and-forget)通知 SPA 拆除,随后 bluetoothd 销毁 transport 对象。

---

## 4. HFP/HSP(AG 与 HF)

HFP 的音频协商(AT 命令)和 SCO 音频**不走 MediaEndpoint/Acquire 那套**,而是走 **Profile1**:bluetoothd 负责 SDP 记录与 RFCOMM 通道的建立,建立后把 **RFCOMM fd** 交给 SPA,业务(AT 命令、SCO 建链)全部由 SPA 完成。

- **HFP AG**(`/Profile/HFPAG`,UUID = HFP_AG):本机是音频网关,手机(远端 HF)打进来。
- **HFP HF**(`/Profile/HFPHF`,UUID = HFP_HF):本机是免提单元,向手机(远端 AG)发起/接听。
- HSP AG/HS 同理(无 AT 命令,SCO 直接可用)。

### MSC-3:HFP AG(远端 HF 接入)

```
bluez5 SPA (本机=AG)             bluetoothd                          手机 (远端 HF)
     |                               |                                     |
     | ① RegisterProfile(/Profile/HFPAG, HFP_AG, {Version,Features})        |
     |------------------------------>|  ② SDP 记录发布 + RFCOMM 监听
     |                               |     (ext_start_servers, src/profile.c:1379)
     |                               |◄──── RFCOMM 连接 ─────────────────────|
     |                               |  ③ Profile1.NewConnection(device, fd, {Version,Features})
     |<------------------------------|     fd = RFCOMM socket (src/profile.c:1058)
     | ④ reply                       |     回复错误则断开 RFCOMM
     |------------------------------>|
     |═══════ AT 命令: 在 RFCOMM fd 上收发 (AT+BRSF/AT+BAC/AT+BCS…) ═══════|
     |═══════ 音频: SPA 自建 SCO socket (sco-io.c),经内核 HCI,不经 D-Bus ══|
     |                               |◄──── 断开 ───────────────────────────|
     | ⑤ Profile1.RequestDisconnection(device)
     |<------------------------------|     (send_disconn_req, src/profile.c:1810)
```

### MSC-4:HFP HF(本机向手机发起)

与 MSC-3 的差异只有两点:注册 UUID 换成 HFP_HF(`/Profile/HFPHF`),RFCOMM 连接由**本机策略发起**(outbound,`src/profile.c:1762 ext_connect_dev`)。之后同样 `NewConnection(fd)` → AT 走 fd、音频走自建 SCO。

### 数据路径

- **① RegisterProfile**(`src/profile.c:2494`):存 `ext_profile{owner, path}`,挂 disconnect watch;`ext_start_servers`(`src/profile.c:1379`)按 options(Channel/Version/Features)开 RFCOMM 监听并生成 SDP 记录供远端查询(带 `ServiceRecord` 选项则直接用 SPA 给的记录)。
- **③ NewConnection**(`src/profile.c:1058-1120`):bluetoothd 接受 RFCOMM 连接后,把设备路径 + RFCOMM fd + 属性字典发给 SPA。SPA 侧 `profile_new_connection`(`backend-native.c:3431`)按对象路径识别角色(`path_to_profile`,`backend-native.c:3410`),把 fd 挂进主循环 IO source(`backend-native.c:3539`),开始 AT 交换(`AT+BRSF`、宽频协商,`backend-native.c:3563-3575`);HSP 则直接建 transport。AG 角色下音频走 SPA 自己建立的 SCO socket,与 bluetoothd 无关。
- 断开时 bluetoothd 调 **RequestDisconnection** 通知 SPA 释放 fd 与 transport。

---

## 5. 消息 × 数据路径速查表

| 消息 | 方向 | 参数 | 接收后发生什么(数据路径) |
|---|---|---|---|
| `Media1.RegisterApplication` | SPA→bt | app 根路径, options | GDBusClient 订阅 + GetManagedObjects 枚举端点 → `media_endpoint` 列表;reply 延至发现完成 |
| `ProfileManager1.RegisterProfile` | SPA→bt | Profile1 路径, UUID, options | 存 `ext_profile`;RFCOMM 监听 + SDP 记录 |
| `MediaEndpoint1.SelectConfiguration` | bt→SPA | 远端 capabilities | SPA codec 选配置 → 回复 ay → AVDTP SetConfiguration 发远端 |
| `MediaEndpoint1.SetConfiguration` | bt→SPA | transport 路径, 属性 | bt 先建 MediaTransport1;SPA 建内部 transport + 接入 graph |
| `MediaTransport1.Acquire` | SPA→bt | — | 单 owner 检查 → 流 resume/start → reply 带 fd + MTU;SPA 在 fd 上收发音频 |
| `MediaTransport1.Release` | SPA→bt | — | 流 suspend,State → idle |
| `MediaEndpoint1.ClearConfiguration` | bt→SPA | transport 路径 | SPA 释放内部 transport;bt 销毁对象 |
| `Profile1.NewConnection` | bt→SPA | device, **fd**, dict | fd=RFCOMM;SPA 挂 IO 跑 AT;音频另建 SCO |
| `Profile1.RequestDisconnection` | bt→SPA | device | SPA 释放 RFCOMM fd / transport |

其他要点:

- **超时**:bluetoothd 对端点回复限时 3 秒(`REQUEST_TIMEOUT`,`media.c:74`),刻意小于 AVDTP 4 秒协议超时。
- **生命周期**:双向都挂 disconnect/name watch —— SPA 退出 → bluetoothd 自动清端点/Profile(`media_endpoint_exit` `media.c:319`、`ext_exited` `src/profile.c:2485`);bluetoothd 重启 → SPA 的 `filter_cb` 监听 `NameOwnerChanged`(`bluez5-dbus.c:6885`)自动重注册。
- **Volume**:transport 的 `Volume` 属性由 bluetoothd 维护(`transport.c:1152 set_volume`),SPA 经 PropertiesChanged 感知;A2DP 绝对音量经 AVRCP 转发远端。
- **LE Audio (BAP)**:同一套 RegisterApplication + SetConfiguration + Acquire 模式,端点在 `/MediaEndpointLE/BAPSink|BAPSource/…`,Acquire 返回的是 ISO socket(`bluez5-dbus.c:4335`)。
- PipeWire 只用 RegisterApplication(master,BlueZ ≥ 5.55),旧版用 RegisterEndpoint;bluetoothd 两者都支持。

---

## 6. 代码索引

| 主题 | BlueZ | PipeWire (bluez5) |
|---|---|---|
| 端点注册/发现 | `profiles/audio/media.c`(register_app 3449, create_app 3394, proxy_added_cb 3319, media_endpoint_create 1563) | `bluez5-dbus.c`(register_media_application 6178, adapter_register_application 6291, endpoint_handler 5597) |
| 端点回调 | `media.c`(select_configuration 502, set_configuration 559, clear_configuration 351, async_call 463) | `bluez5-dbus.c`(endpoint_select_configuration 893, endpoint_set_configuration 5406, endpoint_clear_configuration 5552) |
| A2DP 接入点 | `profiles/audio/a2dp.c`(3005, 3174 select;866, 1132 setconfig) | `bluez5-dbus.c`(do_transport_acquire 4407, transport_acquire_reply 4263, do_transport_release 4509) |
| Transport / Acquire | `profiles/audio/transport.c`(acquire 792, release 899, fd 回复 526-530, 状态机 60) | — |
| Profile 注册/回调 | `src/profile.c`(register_profile 2494, ext_profile 648, send_new_connection 1058, ext_connect 1125) | `backend-native.c`(register_profile 3740, profile_new_connection 3431, path_to_profile 3410, vtable 4246) |
| ObjectManager 客户端 | `gdbus/client.c`(new_full 1456, service_connect 1393, get_managed_objects 1401, interfaces_added 1232) | `bluez5-dbus.c`(get_managed_objects 6868, filter_cb 6885, 信号处理 6948/6962) |
| fd passing | `transport.c:526-530`(回复), `src/profile.c:1090`(NewConnection 参数) | `bluez5-dbus.c:4306-4310`(收 fd), `backend-native.c:3501`(收 RFCOMM fd) |
