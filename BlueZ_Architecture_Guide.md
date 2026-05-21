# BlueZ 架构与核心组件指南

## 1. 整体架构

BlueZ 采用了经典的分层架构，从用户态工具到内核态驱动共分为四层：

```
+---------------------------+
|         工具层 (Tool)       |  bluetoothctl / btmgmt / hciconfig / hcitool / btmon / btattach
+---------------------------+
             ^
             | D-Bus / ioctl / HCI socket
             v
+---------------------------+
|     守护进程层 (Daemon)     |  bluetoothd
+---------------------------+
             ^
             | AF_BLUETOOTH socket (Mgmt / HCI raw / HCI user channel)
             v
+---------------------------+
|      内核层 (Kernel)       |  BlueZ kernel subsystem (net/bluetooth/)
+---------------------------+
             ^
             | HCI UART / USB / SDIO
             v
+---------------------------+
|        硬件 (Hardware)     |  Bluetooth Controller
+---------------------------+
```

## 2. 关键 socket 类型

| Socket 类型 | 协议 / 通道 | 用途 |
|-------------|-------------|------|
| Mgmt Socket | `PF_BLUETOOTH + BTPROTO_HCI + HCI_CHANNEL_CONTROL` | bluetoothd 与内核之间的管理通道。所有上层命令（StartDiscovery、配对、连接等）通过此通道发往内核。 |
| HCI Raw Socket | `PF_BLUETOOTH + BTPROTO_HCI + HCI_CHANNEL_RAW` | 应用直接向 controller 发送 HCI 命令、接收 HCI 事件的底层通道。hciconfig、hcitool 使用此方式 |
| HCI User Channel | `PF_BLUETOOTH + BTPROTO_HCI + HCI_CHANNEL_USER` | 将控制器完全暴露给用户态程序，内核不介入 |
| L2CAP / RFCOMM Socket | `PF_BLUETOOTH + BTPROTO_L2CAP / BTPROTO_RFCOMM` | 数据面 socket，由应用直接使用 |

## 3. 可执行文件及对应源码目录

### 3.1 核心守护进程

**bluetoothd** —— Bluetooth 核心守护进程，BlueZ 的"大脑"

- 源码: `src/main.c`（入口）, `src/adapter.c`, `src/device.c`, `src/profile.c` 等
- 主要职责: D-Bus 服务端、适配器管理、设备管理、配对与会话控制、GATT 服务管理、SDP 服务
- 构建目标: `pkglibexec_PROGRAMS += src/bluetoothd`
- 源码文件构成:
  - 入口: `src/main.c` — 解析参数、初始化 D-Bus 连接、加载配置文件
  - 适配器管理: `src/adapter.c/.h` — 适配器发现、配对、属性管理（StartDiscovery、StopDiscovery 等 API 实现在此）
  - 设备管理: `src/device.c/.h` — 远端设备对象管理、profile 连接
  - Profile 管理: `src/profile.c/.h` — 注册/管理 profile
  - GATT: `src/gatt-client.c/.h`, `src/gatt-database.c/.h`
  - SDP 服务: `src/sdpd-*.c`, `src/sdp-client.c/.h`
  - 广告: `src/advertising.c/.h`
  - 后台扫描: `src/adv_monitor.c/.h`
  - Agent（配对代理）: `src/agent.c/.h`
  - 插件系统: `src/plugin.c/.h`
  - D-Bus 通用: `src/dbus-common.c/.h`
  - 设置存储: `src/settings.c/.h`, `src/textfile.c/.h`
  - EIR 解析: `src/eir.c/.h`
  - UUID 辅助: `src/uuid-helper.c/.h`
  - 电池: `src/battery.c/.h`
  - Bearer: `src/bearer.c/.h`
  - 共享库: `src/shared/` 目录 — mgmt、hci、gatt-client/server/db、att、queue、mainloop、timeout、io、util、crypto 等

- 链接库: `lib/libbluetooth-internal.la`, `gdbus/libgdbus-internal.la`, `src/libshared-glib.la`, `libglib`, `libdbus`, `-ldl`, `-lrt`

### 3.2 用户工具

**bluetoothctl** —— 交互式蓝牙管理工具

- 源码: `client/main.c`（入口）, `client/` 下其他文件
- 构建目标: `bin_PROGRAMS += client/bluetoothctl`
- 源码文件构成:
  - 入口: `client/main.c` — 主命令行 shell，所有命令（scan/devices/pair/connect等）在此注册
  - 显示与打印: `client/display.c/.h`, `client/print.c/.h`
  - Agent（配对代理）: `client/agent.c/.h`
  - 广告: `client/advertising.c/.h`
  - 后台扫描监控: `client/adv_monitor.c/.h`
  - GATT 操作: `client/gatt.c/.h`
  - 管理子菜单: `client/admin.c/.h`
  - 媒体播放器: `client/player.c/.h`
  - 辅助功能: `client/assistant.c/.h`
  - 底层 HCI 命令（raw/user channel）: `client/hci.c/.h`
  - 直接 Mgmt socket 操作: `client/mgmt.c/.h`
  - 电话: `client/telephony.c/.h`

- 链接库: `lib/libbluetooth-internal.la`, `gdbus/libgdbus-internal.la`, `src/libshared-glib.la`, `libglib`, `libdbus`, `libreadline`

**btmon** —— HCI/Mgmt 抓包与流量分析工具

- 源码: `monitor/main.c`（入口）, `monitor/` 下其他文件
- 构建目标: `bin_PROGRAMS += monitor/btmon`
- 源码文件构成:
  - 入口: `monitor/main.c` — 打开 socket、启动抓包循环
  - 协议解析器: `monitor/packet.c/.h` — HCI 包解析
  - 协议层解析: `monitor/l2cap.c/.h`, `monitor/rfcomm.c/.h`, `monitor/sdp.c/.h`, `monitor/avctp.c/.h`, `monitor/avdtp.c/.h`, `monitor/a2dp.c/.h`, `monitor/bnep.c/.h`, `monitor/att.c/.h`
  - 链路层: `monitor/ll.c/.h`
  - LMP 协议: `monitor/lmp.c/.h`
  - 显示: `monitor/display.c/.h`
  - 分析: `monitor/analyze.c/.h`
  - 控制: `monitor/control.c/.h`
  - 厂商扩展: `monitor/intel.c/.h`, `monitor/broadcom.c/.h`, `monitor/msft.c/.h`
  - Hardware DB: `monitor/hwdb.c/.h`
  - Keys 解密: `monitor/keys.c/.h`
  - HCI dump: `monitor/hcidump.c/.h`
  - Ellisys 格式: `monitor/ellisys.c/.h`
  - J-Link: `monitor/jlink.c/.h`
  - CRC: `monitor/crc.c/.h`
  - TTY: `monitor/tty.h`

- 链接库: `lib/libbluetooth-internal.la`, `src/libshared-mainloop.la`, `libglib`, `libudev`, `-ldl`

**hciconfig** —— HCI 设备配置工具（已废弃）

- 源码: `tools/hciconfig.c`
- 构建目标: `bin_PROGRAMS += tools/hciconfig` (deprecated)
- 单一 C 文件，通过 HCI raw socket 直接操作控制器
- 链接库: `lib/libbluetooth-internal.la`

**hciattach** —— HCI 控制器串口/UART 连接工具（已废弃）

- 源码: `tools/hciattach.c`, `tools/hciattach_*.c`
- 构建目标: `bin_PROGRAMS += tools/hciattach` (deprecated)
- 厂商特定初始化: `hciattach_st.c`, `hciattach_ti.c`, `hciattach_tialt.c`, `hciattach_ath3k.c`, `hciattach_qualcomm.c`, `hciattach_intel.c`, `hciattach_bcm43xx.c`
- 链接库: `lib/libbluetooth-internal.la`

**btattach** —— 新的控制器串口/UART 连接工具

- 源码: `tools/btattach.c`
- 构建目标: `bin_PROGRAMS += tools/btattach`
- 替代旧版 hciattach，使用用户态主要事件循环（mainloop）
- 链接库: `src/libshared-mainloop.la`

**hcitool** —— 底层 HCI 命令工具（已废弃）

- 源码: `tools/hcitool.c`, `src/oui.c/.h`
- 构建目标: `bin_PROGRAMS += tools/hcitool` (deprecated)
- 通过 HCI raw socket 发送 HCI 命令
- 链接库: `lib/libbluetooth-internal.la`, `libudev`

**btmgmt** —— Mgmt 通道管理工具

- 源码: `tools/btmgmt.c`, `client/display.c/.h`, `client/mgmt.c/.h`, `src/uuid-helper.c`
- 构建目标: `noinst_PROGRAMS += tools/btmgmt`
- 通过 mgmt socket 直接与内核交互
- 链接库: `lib/libbluetooth-internal.la`, `src/libshared-glib.la`, `libglib`, `libreadline`

**sdptool** —— SDP 服务发现工具（已废弃）

- 源码: `tools/sdptool.c`, `src/sdp-xml.c/.h`
- 链接库: `lib/libbluetooth-internal.la`, `libglib`

**其他工具** (`tools/` 目录下):

- `tools/l2ping.c` — L2CAP ping 测试
- `tools/l2test.c` — L2CAP 测试
- `tools/rctest.c` — RFCOMM 测试
- `tools/scotest.c` — SCO 测试
- `tools/isotest.c` — ISO 测试
- `tools/avinfo.c` — AVDTP 信息
- `tools/btgatt-client.c` — GATT 客户端示例
- `tools/btgatt-server.c` — GATT 服务端示例
- `tools/bluemoon.c` — 固件下载工具
- `tools/bdaddr.c` — 修改 BD_ADDR
- `tools/btinfo.c` — 蓝牙信息查询

### 3.3 共享库

| 库名 | 源码位置 | 说明 |
|------|---------|------|
| `lib/libbluetooth-internal.la` | `lib/bluetooth.c`, `lib/hci.c`, `lib/sdp.c`, `lib/uuid.c` 等 | 内部蓝牙基础库 |
| `gdbus/libgdbus-internal.la` | `gdbus/` 目录 | D-Bus 封装库 |
| `src/libshared-glib.la` | `src/shared/` 下 glib 实现 | glib 版本的共享模块（mgmt、hci、gatt 等） |
| `src/libshared-mainloop.la` | `src/shared/` 下 mainloop 实现 | mainloop 版本的共享模块 |
| `src/libshared-ell.la` | `src/shared/` 下 ell 实现 | ELL 版本的共享模块 |

### 3.4 源码目录功能划分

| 目录 | 说明 |
|------|------|
| `src/` | bluetoothd 守护进程 + shared 共享库 |
| `src/shared/` | 与 I/O 后端无关的共享模块（mgmt、hci、gatt、att、queue、mainloop 等） |
| `client/` | bluetoothctl 及其公共显示/打印模块 |
| `monitor/` | btmon 抓包工具及协议解析器 |
| `tools/` | 各种独立工具（hciconfig、hcitool、sdptool、btmgmt、btattach 等） |
| `lib/` | libbluetooth-internal 核心库（bluetooth.h、hci.h、sdp.h、mgmt.h 协议定义） |
| `gdbus/` | D-Bus 消息总线的 C 封装库 |
| `btio/` | Bluetooth socket I/O 封装（bt_io_* API） |
| `attrib/` | ATT/GATT 属性协议底层实现 |
| `gobex/` | OBEX 协议库 |
| `plugins/` | bluetoothd 的内置插件 |
| `profiles/` | 各蓝牙 profile 实现（audio、input、network 等） |
| `emulator/` | 蓝牙模拟器 for 测试（btvirt 等） |
| `unit/` | 单元测试 |
| `test/` | Python 测试脚本 |
| `doc/` | 文档（API、mgmt 协议、btmon 说明等） |
| `mesh/` | Bluetooth Mesh 支持 |

## 4. bluetoothctl 基本用法

```bash
# 启动交互式 Shell
bluetoothctl

# 常用命令（在 bluetoothctl Shell 内）

[bluetooth]# power on              # 打开适配器电源
[bluetooth]# power off             # 关闭适配器电源
[bluetooth]# show                  # 显示适配器详细信息
[bluetooth]# list                  # 列出所有可用适配器
[bluetooth]# select <MAC>          # 选择要操作的适配器

[bluetooth]# scan on               # 开始扫描（内部调用 D-Bus StartDiscovery）
[bluetooth]# scan off              # 停止扫描（内部调用 D-Bus StopDiscovery）
[bluetooth]# devices               # 列出已发现的设备
[bluetooth]# devices <属性>        # 列出已连接/已配对设备
[bluetooth]# paired-devices        # 列出已配对的设备

[bluetooth]# connect <MAC>         # 连接设备
[bluetooth]# disconnect <MAC>      # 断开设备连接
[bluetooth]# remove <MAC>          # 移除设备

[bluetooth]# pair <MAC>            # 配对设备
[bluetooth]# trust <MAC>           # 信任设备（后续自动连接）
[bluetooth]# untrust <MAC>         # 取消信任

[bluetooth]# info <MAC>            # 显示设备详细信息
[bluetooth]# block <MAC>           # 屏蔽设备
[bluetooth]# unblock <MAC>         # 取消屏蔽设备

# 子菜单
[bluetooth]# menu scan             # 进入扫描子菜单
[bluetooth]# menu gatt             # 进入 GATT 子菜单
[bluetooth]# menu advertise        # 进入广告子菜单
[bluetooth]# menu admin            # 进入管理子菜单
[bluetooth]# menu mgmt             # 进入 mgmt 子菜单
[bluetooth]# menu monitor          # 进入 btmon 子菜单
[bluetooth]# menu hci              # 进入 HCI 子菜单（直接发送 HCI 命令）

# 退出
[bluetooth]# quit / exit
```

## 5. Inquiry/Discovery 通信流程与函数调用栈

> **注意**: 在 BlueZ 中，`inquiry`（HCI Inquiry）已经被 `discovery`（LE Scan + BR/EDR Inquiry）机制取代。用户通过 `bluetoothctl scan on` 使用 D-Bus API `StartDiscovery` 来触发设备发现。下面以 `scan on` 为入口，将从 command（命令）到 event（事件）的完整函数调用链串联起来。

### 5.1 Command 通路（bluetoothctl -> D-Bus -> bluetoothd -> Mgmt Socket -> Kernel）

```
bluetoothctl (client/main.c)
  cmd_scan()                                  # 用户输入 "scan on" 触发
    |
    +-> set_discovery_filter(false)           # 先设置发现过滤器
    |
    +-> g_dbus_proxy_method_call(             # 通过 D-Bus 代理调用 org.bluez.Adapter.StartDiscovery
          default_ctrl->proxy, "StartDiscovery",
          NULL, start_discovery_reply, ...)
          |
          ===== D-Bus 消息总线 =====
          |
bluetoothd (src/adapter.c)
  start_discovery()                           # D-Bus 方法处理函数 (GDBus 调度)
    |
    +-> btd_adapter_get_powered(adapter)      # 检查适配器是否已上电
    |
    +-> get_discovery_client(adapter, ...)    # 检查是否已有该客户端的发现会话
    |
    +-> g_new0(struct discovery_client, 1)    # 创建发现客户端
    |
    +-> g_dbus_add_disconnect_watch(...)      # 监听客户端断开
    |
    +-> update_discovery_filter(adapter)      # 更新发现过滤条件
    |     |
    |     +-> trigger_start_discovery(adapter, 0)   # 触发启动发现（delay=0 立即执行）
    |             |
    |             +-> cancel_passive_scanning(adapter)  # 取消被动扫描
    |             |
    |             +-> timeout_add(delay,             # 用延迟 timeout 调度
    |             |       start_discovery_timeout, adapter)
    |             |
    |             ===== timeout 触发后 =====
    |             |
    |             +-> start_discovery_timeout(adapter)
    |                   |
    |                   +-> get_scan_type(adapter)        # 根据过滤条件确定扫描类型
    |                   |
    |                   +-> mgmt_send(adapter->mgmt,      # 通过 Mgmt socket 发送
    |                   |       MGMT_OP_START_DISCOVERY,  #   START_DISCOVERY 命令
    |                   |       adapter->dev_id,
    |                   |       sizeof(cp), &cp,
    |                   |       start_discovery_complete,
    |                   |       adapter, NULL)
    |                   |
    |                   |     (src/shared/mgmt.c)
    |                   |     mgmt_send()
    |                   |       -> mgmt_send_timeout()
    |                   |           -> create_request()     # 创建 mgmt_request 包
    |                   |           -> queue_push_tail(mgmt->request_queue, ...)
    |                   |           -> wakeup_writer(mgmt)
    |                   |                |
    |                   |                +-> io_set_write_handler(mgmt->io,
    |                   |                |       can_write_data, ...)   # 注册写就绪回调
    |                   |                |
    |                   |                ===== I/O 主循环检测到 socket 可写 =====
    |                   |                |
    |                   |                +-> can_write_data()
    |                   |                      -> send_request(mgmt, request)
    |                   |                           -> io_send(mgmt->io, &iov, 1)
    |                   |                              = write() to AF_BLUETOOTH mgmt socket
    |                   |
    |                   ===== 数据写入 AF_BLUETOOTH socket (HCI_CHANNEL_CONTROL) =====
    |                   |
    |                   ===== 内核 net/bluetooth/mgmt.c 接收 MGMT_OP_START_DISCOVERY =====
    |                   Kernel 下发 HCI 命令到 Bluetooth Controller（硬件）
    |
    +-> dbus_message_new_method_return(msg)    # 返回 D-Bus 回复
```

### 5.2 Event 通路（Kernel -> Mgmt Socket -> bluetoothd -> D-Bus -> bluetoothctl）

初始化时的事件注册:

```
bluetoothd (src/adapter.c)
  btd_adapter_new() 调用:
    mgmt_register(adapter->mgmt, MGMT_EV_DISCOVERING, ...)   # 注册 Discovering 状态变化通知
      -> (src/shared/mgmt.c) mgmt_register()
           -> 将 notify 加入 mgmt->notify_list

    mgmt_register(adapter->mgmt, MGMT_EV_DEVICE_FOUND, ...)  # 注册设备发现事件通知
      -> 回调函数: device_found_callback()
```

BT Controller 发现设备后的事件上行链路:

```
===== Hardware 报告扫描结果 =====
===== 内核 mgmt.c 封装为 MGMT_EV_DEVICE_FOUND =====
===== 数据写入 AF_BLUETOOTH socket (HCI_CHANNEL_CONTROL) =====

bluetoothd (src/shared/mgmt.c)
  主循环 I/O 回调: can_read_data()
    |
    +-> read(mgmt->fd, mgmt->buf, mgmt->len)     # 从 mgmt socket 读取数据
    |
    +-> 解析 mgmt_hdr:
    |     event = MGMT_EV_DEVICE_FOUND            # 设备发现事件（或其他事件类型）
    |     index = adapter->dev_id
    |
    +-> process_notify(mgmt, event, index, length, param)
    |     |
    |     +-> queue_foreach(mgmt->notify_list, notify_handler, &match)
    |           |
    |           +-> notify_handler() 遍历匹配的 notify 列表
    |                 -> device_found_callback(index, length, param, user_data)
    |                    (src/adapter.c)
    |                      |
    |                      +-> 解析 mgmt_ev_device_found:
    |                      |     ev->addr, ev->rssi, ev->flags, ev->eir
    |                      |
    |                      +-> btd_adapter_device_found(adapter, &addr, ...)
    |                            |
    |                            +-> btd_adapter_find_device() 或 adapter_create_device()
    |                            |   # 更新/创建设备对象
    |                            |
    |                            +-> device_set_rssi(), device_set_flags(), ...
    |                            |   # 更新设备属性
    |                            |
    |                            +-> g_dbus_emit_property_changed(dbus_conn,
    |                            |       device->path, DEVICE_INTERFACE, ...)
    |                            |   # 通过 D-Bus 通知所有监听者设备属性变更
    |                            |
    |                            +-> 如果是新发现的设备:
    |                                  g_dbus_emit_signal(dbus_conn, adapter->path,
    |                                      ADAPTER_INTERFACE, "DeviceFound", ...)
                                      # 发射 D-Bus 信号: InterfacesAdded (到 org.bluez.Adapter1)
```

D-Bus 属性变更通知:

```
===== D-Bus 信号发出 =====

bluetoothctl (client/main.c)
  通过 GDBus Proxy 属性回调:
    device_added() / property_changed()
      |
      +-> print_device()                # 在终端打印设备信息
      |
      +-> g_dbus_proxy_set_property()   # 更新本地 proxy 属性

  同时 start_discovery_reply() 被执行:
    -> bt_shell_printf("Discovery started")
    -> filter.active = true
```

### 5.3 停止 Discovery 的 Event 通路

```
mgmt.c: can_read_data()
  -> 收到 MGMT_EV_DISCOVERING (值为 0 表示 discovery 已停止)
  -> process_notify() -> discovering_callback()

adapter.c: discovering_callback()
  -> adapter->discovering = false
  -> g_dbus_emit_property_changed(dbus_conn, adapter->path,
        ADAPTER_INTERFACE, "Discovering")   # 通知 D-Bus "Discovering" 属性变为 false

  -> trigger_start_discovery(adapter, idle_timeout)  # 安排下一次扫描
```

### 5.4 关键数据结构与函数速查表

| 编号 | 步骤 | 文件 | 函数 |
|------|------|------|------|
| 1 | 用户输入 "scan on" | `client/main.c` | `cmd_scan()` |
| 2 | D-Bus 代理调用 StartDiscovery | `client/main.c` | `g_dbus_proxy_method_call()` |
| 3 | D-Bus 方法处理（发现会话创建） | `src/adapter.c:2507` | `start_discovery()` |
| 4 | 触发启动发现 | `src/adapter.c:1990` | `trigger_start_discovery()` |
| 5 | 延迟超时后启动 | `src/adapter.c:1902` | `start_discovery_timeout()` |
| 6 | 通过 Mgmt socket 发送命令 | `src/adapter.c:1967` | `mgmt_send(MGMT_OP_START_DISCOVERY)` |
| 7 | 创建请求并入队 | `src/shared/mgmt.c:831` | `mgmt_send()` -> `mgmt_send_timeout()` |
| 8 | 唤醒写处理器 | `src/shared/mgmt.c:269` | `wakeup_writer()` |
| 9 | 主循环检测 socket 可写 | `src/shared/mgmt.c:241` | `can_write_data()` |
| 10 | 发送请求到 socket | `src/shared/mgmt.c:209` | `send_request()` -> `io_send()` |
| 11 | Mgmt socket 写操作（进入内核） | `src/shared/io-glib.c` | `io_send()` -> `write()` |
| 12 | **内核处理命令** | Kernel `net/bluetooth/mgmt.c` | `start_discovery()` |
| 13 | **内核收事件** | Kernel -> mgmt socket | `MGMT_EV_DEVICE_FOUND` |
| 14 | 主循环检测 socket 可读 | `src/shared/mgmt.c:374` | `can_read_data()` -> `read()` |
| 15 | 分发事件通知 | `src/shared/mgmt.c:355` | `process_notify()` |
| 16 | 遍历 notify 列表 | `src/shared/mgmt.c:336` | `notify_handler()` |
| 17 | 设备发现回调 | `src/adapter.c:7575` | `device_found_callback()` |
| 18 | 更新设备对象 | `src/adapter.c:7314` | `btd_adapter_device_found()` |
| 19 | D-Bus 发射属性变更 | `gdbus/` | `g_dbus_emit_property_changed()` |
| 20 | 客户端回调 | `client/main.c` | 属性变更回调 -> `print_device()` |

### 5.5 Inquiry vs Discovery 对比

| 特性 | HCI Inquiry (旧) | Discovery (新) |
|------|------------------|----------------|
| D-Bus API | 无 | `StartDiscovery` / `StopDiscovery` |
| 传输通道 | HCI Raw Socket | Mgmt Socket (`HCI_CHANNEL_CONTROL`) |
| 协议类型 | 仅 BR/EDR | BR/EDR Inquiry + LE Scan 交替 |
| 过滤条件 | 不支持 | 支持 UUID、RSSI、Pathloss、Transport、Pattern 等 |
| 工具入口 | `hcitool scan` (已废弃) | `bluetoothctl scan on` |
| 内核接口 | ioctl(HCIINQUIRY) | `MGMT_OP_START_DISCOVERY` |

### 5.6 官方 D-Bus API 文档位置

BlueZ 的所有 D-Bus API 文档在源码 `doc/` 目录下，以 `.txt` 格式维护，这是**最权威的参考**：

```
doc/adapter-api.txt        — org.bluez.Adapter1       (扫描、发现、配对模式)
doc/device-api.txt         — org.bluez.Device1        (连接、配对、属性)
doc/agent-api.txt          — org.bluez.Agent1         (配对代理/PIN码输入)
doc/gatt-api.txt           — GattService1/Characteristic1/Descriptor1
doc/media-api.txt          — MediaTransport1/Media1   (A2DP音频)
doc/mgmt-api.txt           — 内核 Mgmt 协议命令和事件定义
doc/settings-storage.txt   — 设置文件格式
```

**在线查看**：
- Debian manpages 渲染版: `man org.bluez.Adapter` (需安装 bluez)
- BlueZ 官方 git: https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc

**最常用的 D-Bus 接口速查**：

| 接口 | 关键方法 | 关键属性/信号 |
|------|---------|-------------|
| `org.bluez.Adapter1` | `StartDiscovery`, `StopDiscovery`, `SetDiscoveryFilter` | `Powered`, `Discovering`, `Address` |
| `org.bluez.Device1` | `Connect`, `Disconnect`, `Pair`, `CancelPairing` | `Connected`, `Paired`, `RSSI`, `UUIDs` |
| `org.bluez.GattCharacteristic1` | `ReadValue`, `WriteValue`, `StartNotify`, `StopNotify` | `Value`, `Notifying` |
| `org.bluez.AgentManager1` | `RegisterAgent`, `UnregisterAgent` | — |
| `org.freedesktop.DBus.ObjectManager` | `GetManagedObjects` | `InterfacesAdded`, `InterfacesRemoved` |

---

## 6. 关键源码文件索引

| 组件 | 入口文件 | 关键函数 | 说明 |
|------|---------|---------|------|
| bluetoothd | `src/main.c` | `main()` | 守护进程入口，初始化 D-Bus、mgmt socket、加载插件 |
| bluetoothd 适配器 | `src/adapter.c` | `start_discovery()`, `trigger_start_discovery()`, `device_found_callback()` | 适配器管理的核心文件 |
| bluetoothd 设备 | `src/device.c` | `device_create()`, `device_connect()` | 远端设备对象管理 |
| Mgmt 共享层 | `src/shared/mgmt.c` | `mgmt_new_default()`, `mgmt_send()`, `can_read_data()`, `mgmt_register()` | Mgmt socket 的封装层，命令下发与事件分发 |
| HCI 共享层 | `src/shared/hci.c` | `bt_hci_new_raw_device()`, `bt_hci_send()`, `bt_hci_register()` | Raw HCI socket 封装层 |
| bluetoothctl | `client/main.c` | `cmd_scan()`, `cmd_connect()`, `cmd_pair()` | bluetoothctl 所有用户命令注册与处理 |
| btmon | `monitor/main.c` | `main()` | 抓包入口 |
| btmon 包解析 | `monitor/packet.c` | `packet_hci_*()` | HCI 报文解析与格式化输出 |
| btattach | `tools/btattach.c` | `main()` | UART 控制器 attach |
| hciconfig | `tools/hciconfig.c` | `main()` | HCI raw socket 配置工具（已废弃） |
| 蓝牙基础库 | `lib/bluetooth.c` | `hci_open_dev()`, `hci_inquiry()` 等 | 底层 HCI socket 操作 API |
| 协议头文件 | `lib/mgmt.h` | `MGMT_OP_*`, `MGMT_EV_*` | Mgmt 协议的操作码和事件码定义 |
| 协议头文件 | `lib/hci.h` | `HCI_OP_*`, `HCI_EV_*` | HCI 协议的操作码和事件码定义 |
| 协议头文件 | `lib/bluetooth.h` | `AF_BLUETOOTH`, `BTPROTO_*`, `HCI_CHANNEL_*` | Socket 族、协议和通道常量定义 |
