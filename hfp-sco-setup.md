# HFP SCO 建立完整流程(用户态 → 内核 → 芯片)

> 场景:本机 = 手机(AG 角色,modem 由 oFono 管理),远端 = 蓝牙耳机(HF 角色)。
> 音频框架 = PipeWire + WirePlumber,蓝牙协议栈 = BlueZ(bluetoothd)+ 内核。
>
> 路径约定:
> - `src/...` = bluez 仓库(本仓库)
> - `../ofono/...`、`../pipewire/...`、`../wireplumber/...`、`../linux-next/...` = 相邻的源码树

## 目录

- [0. 关键结论(先读)](#0-关键结论先读)
- [1. 完整流程](#1-完整流程)
  - 1.0 流程地图;1.1 来电与振铃 → 1.2 接听与 codec 协商 → 1.3 方式 A(HF 发起)→ 1.4 方式 B(AG 发起)→ 1.5 内核建链 → 1.6 完成回报与音频流动
- [2. 参数是如何设置的](#2-参数是如何设置的)
  - 2.1 参数全景
  - 2.2 enhanced 还是 legacy:三层决策 · 2.3 各驱动的默认设置(QCA 等)
  - 2.4 用户态 sockopt 入口 · 2.5 默认值
  - 2.6 内核 sockopt 处理
  - 2.7 内核组包(enhanced / legacy / 参数表)
  - 2.8 Accept 侧参数(非 defer / defer / 纯 SCO / 两家映射)
  - 2.9 PipeWire 配置链 · 2.10 offload 全场景 · 2.11 backend-ofono 差异
- [3. 相关测试设施对照](#3-相关测试设施对照)

## 0. 关键结论(先读)

1. **bluetoothd 不参与 SCO 数据面**:全程只做了两件事——耳机连 RFCOMM 时把 fd 经
   `Profile1.NewConnection` 转交 oFono(`src/profile.c:1083`);提供 D-Bus 路由。
   建 SCO、传音频都不经过 bluetoothd。
2. **SCO socket 由 oFono 或 PipeWire 直接创建**,`connect()` 是普通系统调用直接进内核,
   无中间人。
3. **enhanced 还是 legacy 由内核决定**,只看芯片 HCI 命令支持表 `commands[29] & 0x08`
   与 BROKEN quirk(`include/net/bluetooth/hci_core.h:2076`)。

## 1. 完整流程

### 1.0 流程地图

```
           ┌────────────── 控制面(RFCOMM,bluetoothd 只转 fd)──────────────┐
   1.1 来电 →  1.2 接听 + codec 协商(AT+BAC / +BCS)
           └─────────────────────────┬───────────────────────────────────┘
                                     ▼
                         数据面:SCO 由谁主动发起?
               ┌─────────────────────┴─────────────────────┐
       1.3 方式 A:HF 发起                       1.4 方式 B:AG 发起
       oFono listen + BT_DEFER_SETUP           PipeWire Acquire
       → PipeWire 读 1 字节授权                → oFono connect()
               │                                       │
               └───────────────────┬───────────────────┘
                                   ▼
       1.5 内核建链 hci_connect_sco → hci_sco_setup
           enhanced: HCI_OP_ENHANCED_SETUP_SYNC_CONN(0x043D)
           legacy:   HCI_OP_SETUP_SYNC_CONN(0x0428)/ HCI_OP_ADD_SCO(0x0411)
                                   ▼
       1.6 Synchronous Connection Complete → SCO 音频流动
```

### 1.1 来电与振铃(RFCOMM 控制面)

```
1. modem 上报来电
   ../ofono/src/voicecall.c:749        notify_emulator_call_status(vc)
   (通话状态变化,广播给所有挂接的 HFP emulator)

2. oFono 经 RFCOMM 通知耳机
   ../ofono/src/emulator.c:723         emulator_call_status_cb()
   ../ofono/src/emulator.c:374         g_at_server_send_unsolicited("+CIEV: 2,1")  callsetup=1
   ../ofono/src/emulator.c:427         g_at_server_send_unsolicited("RING")

   ┌─ 这条 RFCOMM 的来源:
   │  耳机连 RFCOMM → bluetoothd 收到 → 转发给 oFono
   │  src/profile.c:1083               向 oFono 发 org.bluez.Profile1.NewConnection(RFCOMM fd)
   │  ../ofono/plugins/hfp_ag_bluez5.c:227   收到 fd → 建 ofono_emulator(g_at_server)
   │  底层就是普通 RFCOMM socket(kernel net/bluetooth/rfcomm/sock.c)
```

### 1.2 接听与 codec 协商

```
3. 用户接听 → oFono voicecall Answer
   ../ofono/src/emulator.c:374         再发 "+CIEV: 1,1"(call=1)

4. codec 协商(同一 RFCOMM 通道,AT+BAC / +BCS)
   ../ofono/src/emulator.c             g_at_server 响应 BAC/BCS
   ../ofono/src/handsfree-audio.c      card->selected_codec 记录协商结果
```

### 1.3 方式 A:耳机(HF)主动发起 SCO —— oFono listen + PipeWire 授权

```
5. 耳机对自己本地的 SCO socket 发起 connect
   (空中消息:芯片收到对端的 Synchronous Connection Request)

6. 手机内核收到请求,先 hold 住(defer):
   ../ofono/src/handsfree-audio.c:198   sco_init(): socket(BTPROTO_SCO) → bind(BDADDR_ANY)
   ../ofono/src/handsfree-audio.c:221   setsockopt(BT_DEFER_SETUP)      ← 延迟接受
   ../ofono/src/handsfree-audio.c:243   listen(sk, 5) + watch(sco_accept)
   kernel net/bluetooth/hci_event.c     连接请求事件:defer 时不发 Accept,conn → BT_CONNECT2
   kernel net/bluetooth/sco.c:1504      sco_connect_cfm() → sco_conn_ready()
   kernel net/bluetooth/sco.c:1406      sco_conn_ready(): listening socket 收到通知

7. oFono accept,把 fd 经 D-Bus 推给 PipeWire:
   ../ofono/src/handsfree-audio.c:~240  sco_accept() → accept() 拿到子 socket
   ../ofono/src/handsfree-audio.c:97    send_new_connection():
                                        D-Bus 调 org.ofono.HandsfreeAudioAgent.NewConnection
                                        (card path, SCO fd, selected codec)

8. PipeWire 收到 fd,读 1 字节完成授权,内核此时才发 Accept:
   ../pipewire/spa/plugins/bluez5/backend-ofono.c:522   ofono_new_audio_connection()
   ../pipewire/spa/plugins/bluez5/backend-ofono.c:497   enable_sco_socket(): read(sock, &c, 1)
   kernel net/bluetooth/sco.c:893       sco_conn_defer_accept(): 发 HCI_OP_ACCEPT_SYNC_CONN_REQ
   kernel net/bluetooth/hci_conn.c      hci_send_cmd → hdev->send → btusb → 芯片

9. 芯片回 Synchronous Connection Complete → 链路建立
   kernel net/bluetooth/sco.c:1504      sco_connect_cfm() → sco_conn_ready()
```

### 1.4 方式 B:手机(AG)主动发起 SCO —— PipeWire acquire + oFono connect

```
10. WirePlumber 决定把通话音频路由到 SCO transport,启动节点:
    ../wireplumber/src/scripts/monitors/bluez/create-node.lua:31  (api.bluez5.sco.* factory)
    ../wireplumber/src/scripts/policy-node.lua                     link 策略
    → PipeWire transport 的 .acquire 被调用

11. PipeWire 同步调 oFono Acquire:
    ../pipewire/spa/plugins/bluez5/backend-ofono.c:312   .acquire = ofono_audio_acquire
    ../pipewire/spa/plugins/bluez5/backend-ofono.c:199   ofono_audio_acquire()
    ../pipewire/spa/plugins/bluez5/backend-ofono.c:156   _audio_acquire():
                                        D-Bus 调 org.ofono.HandsfreeAudioCard.Acquire(阻塞等回复)

12. oFono 建 SCO:
    ../ofono/src/handsfree-audio.c:397   card_methods: "Acquire" → card_connect
    ../ofono/src/handsfree-audio.c:355   card_connect()
    ../ofono/src/handsfree-audio.c:475   ofono_handsfree_card_connect_sco()
    ../ofono/src/handsfree-audio.c:482   socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_SCO)
    ../ofono/src/handsfree-audio.c:490   bind(BDADDR_ANY)
    ../ofono/src/handsfree-audio.c:507   connect(sk, &addr)          ★ 直接系统调用,不经 bluetoothd
    ../ofono/src/handsfree-audio.c:512   g_io_add_watch(G_IO_OUT, sco_connect_cb)
```

### 1.5 内核建链(方式 A 的 Accept 与方式 B 的 connect 汇合于此)

```
13. connect() 路径(方式 B):
    kernel net/bluetooth/sco.c:691       sco_sock_connect()
    kernel net/bluetooth/sco.c:312       sco_connect():
    kernel net/bluetooth/sco.c:332       lmp_esco_capable(hdev) && !disable_esco → ESCO_LINK
    kernel net/bluetooth/sco.c:351       hci_connect_sco(hdev, type, dst, setting, codec, timeout)

14. kernel net/bluetooth/hci_conn.c:1848 hci_connect_sco():
    - hci_connect_acl()                  先保证 ACL 存在(没有就先建 ACL)
    - hci_conn_add_unset(type, dst)      创建 SCO/ESCO 的 hci_conn
    - hci_conn_link(acl, sco)            挂到 ACL 下
    - sco->setting / sco->codec          保存 voice setting 与 codec
    - ACL 已连接 → hci_conn.c:1891       hci_sco_setup(acl, 0x00)
    - ACL 在 mode change → 置 HCI_CONN_SCO_SETUP_PEND,完成后补发(hci_event.c:2698/2725)

15. kernel net/bluetooth/hci_conn.c:612  hci_sco_setup():
    - lmp_esco_capable → hci_setup_sync(link->conn, handle)
    - 否则             → hci_add_sco()   (hci_conn.c:197,HCI_OP_ADD_SCO 0x0411,最老 SCO 1.1)

16. kernel net/bluetooth/hci_conn.c:463  hci_setup_sync():
    kernel net/bluetooth/hci_conn.c:468  ★ enhanced vs legacy 决策点
        enhanced_sync_conn_capable(hdev)  (include/net/bluetooth/hci_core.h:2076)
        = (commands[29] & 0x08) && !HCI_QUIRK_BROKEN_ENHANCED_SETUP_SYNC_CONN
        ├─ 支持: hci_enhanced_setup_sync()  hci_conn.c:278
        │         → hci_send_cmd(HCI_OP_ENHANCED_SETUP_SYNC_CONN, 0x043d)
        └─ 不支持: hci_setup_sync_conn()    hci_conn.c:402
                  → hci_send_cmd(HCI_OP_SETUP_SYNC_CONN, 0x0428)
    kernel drivers/bluetooth/btusb.c     btusb_send_frame() → USB bulk/interrupt → 芯片 → 耳机
```

### 1.6 完成回报与音频流动

```
17. 芯片回 Synchronous Connection Complete:
    kernel net/bluetooth/hci_event.c     事件处理 → hci_connect_cfm(conn, status)
    kernel net/bluetooth/sco.c:1504      sco_connect_cfm()(.connect_cfm 注册在 sco.c:1572)
    kernel net/bluetooth/sco.c:1406      sco_conn_ready(): sk_state = BT_CONNECTED
    → 用户态 connect() 返回 / accept 出的 socket 就绪

18. 方式 B 的收尾:
    ../ofono/src/handsfree-audio.c:293   sco_connect_cb(): 把 fd 作为 Acquire 的 D-Bus 回复返回
    ../pipewire/spa/plugins/bluez5/backend-ofono.c:210  transport->fd = ret

19. 音频流动:
    PipeWire write(fd) → kernel sco_sock_sendmsg → sco_send_frame
    → kernel net/bluetooth/sco.c         hci_send_sco → hdev->send → btusb → 芯片 → 耳机
    反向: 耳机 → 芯片 → btusb_rx_complete → hci_recv_frame
    → kernel net/bluetooth/sco.c:1537    sco_recv_scodata → 投递 SCO socket → PipeWire read

20. 挂断:
    oFono voicecall hangup → emulator.c 发 "+CIEV"(call=0/callsetup=0)
    → 用户态 close(fd) → kernel sco_sock_release → sco_disconn_cfm
    → HCI_OP_DISCONNECT → 芯片 → 链路拆除
```

## 2. 参数是如何设置的

### 2.1 参数全景:谁在设、谁能设

先看每个 HCI 命令参数最终由谁决定:

| HCI 命令参数 | 来源 |
|---|---|
| handle | SCO hci_conn 的 handle |
| voice_setting / content_format | 用户态 `BT_VOICE` sockopt(PipeWire 设 TRANSPARENT;默认内核芯片值) |
| codec(coding format / bandwidth) | 用户态 `BT_CODEC` sockopt(offload),或由 airmode 派生 TRANSPARENT |
| data_path | `BT_CODEC` sockopt 的 data_path_id + `hdev->get_data_path_id` 映射 |
| pkt_type / max_latency / retrans_effort | 内核写死的参数表(2.7.3),按 codec + attempt 选择 |
| tx/rx_bandwidth | 内核写死 0x1f40 |
| enhanced 还是 legacy | 芯片 `commands[29] & 0x08` + quirk(hci_core.h:2076) |

**用户态可设置(sockopt):**

| sockopt | 影响 |
|---|---|
| `BT_VOICE` | airmode(CVSD/TRANSP)→ legacy 参数表选择、accept 侧 latency/retrans |
| `BT_CODEC` | enhanced 的 coding format/bandwidth/frame size/data_path(需内核 offload 支持) |
| `BT_DEFER_SETUP` | 延迟 accept,授权时机(配合 read 1 字节) |
| connect 超时 | `sk_sndtimeo` → `hci_connect_sco` 的 timeout |

**内核写死(用户无法设置):**

- tx/rx_bandwidth = 0x1f40
- pkt_type / max_latency / retrans_effort(参数表 esco_param_*/sco_param_cvsd)
- 各 codec 的 coding format id、frame size(60)、coded data size(16)、
  pcm data format(2)、transport unit size(1/16)
- enhanced/legacy/ADD_SCO 的选择(芯片属性 + quirk)
- ESCO_LINK vs SCO_LINK(`lmp_esco_capable && !disable_esco`,sco.c:332;
  `disable_esco` 是内核模块参数)
- accept 侧 content_format = `hdev->voice_setting`(芯片默认,非 socket 值)

### 2.2 enhanced 还是 legacy:三层决策

决策全在内核,依据是**芯片属性**,用户态无法选择:

| LMP eSCO 能力 | commands[29] & 0x08 | quirk | 实际命令 |
|---|---|---|---|
| ✗ | — | — | `HCI_OP_ADD_SCO` 0x0411(hci_conn.c:197 `hci_add_sco`) |
| ✓ | ✗ | — | `HCI_OP_SETUP_SYNC_CONN` 0x0428(legacy,hci_conn.c:402) |
| ✓ | ✓ | `HCI_QUIRK_BROKEN_ENHANCED_SETUP_SYNC_CONN` | 同上 legacy |
| ✓ | ✓ | 无 | `HCI_OP_ENHANCED_SETUP_SYNC_CONN` 0x043D(enhanced,hci_conn.c:278) |

- 第一重门:`hci_sco_setup` 里 `lmp_esco_capable(hdev)`(hci_conn.c:621)
- 第二重门:`hci_setup_sync` 里 `enhanced_sync_conn_capable(hdev)`(hci_conn.c:468,
  宏 hci_core.h:2076)
- quirk 由内核驱动(如 btusb)探测芯片时设置,用户不能改
- 用户态与 enhanced 的唯一关联:airmode TRANSP 派生 `BT_CODEC_TRANSPARENT`
  要求 enhanced capable(sco.c:1025);offload `BT_CODEC` 走 data_path 也只有
  enhanced 分支用(`configure_datapath_sync`,hci_conn.c:286)

### 2.3 各驱动的默认设置(QCA 等)

enhanced/legacy 的 quirk 由**驱动**设置,按芯片/接口分派:

| 驱动 | 芯片 | 默认 |
|---|---|---|
| btusb | QCA USB(ATH3012/ROME/WCN6855) | **强制 legacy** |
| hci_qca | QCA UART(SoC) | 无默认,看固件命令表 |
| hci_ll | TI | 强制 legacy(hci_ll.c:664) |
| btusb | MediaTek(MTK) | 强制 legacy(btusb.c:4521) |

**QCA USB(btusb)**:`btusb_setup_qca`(btusb.c:3865)初始化时无条件打 quirk:

```c
/* Mark HCI_OP_ENHANCED_SETUP_SYNC_CONN as broken as it doesn't seem to
 * work with the likes of HSP/HFP mSBC.
 */
hci_set_quirk(hdev, HCI_QUIRK_BROKEN_ENHANCED_SETUP_SYNC_CONN);  /* btusb.c:3937 */
```

挂接该 setup 的:`BTUSB_ATH3012`(btusb.c:4540)、`BTUSB_QCA_ROME`(btusb.c:4547)、
`BTUSB_QCA_WCN6855`(btusb.c:4554)。即使固件命令表声称支持 enhanced,
quirk 使 `enhanced_sync_conn_capable` 恒 false → 永远 `HCI_OP_SETUP_SYNC_CONN`(legacy)。

**QCA UART(hci_qca)**:全文不设 BROKEN_ENHANCED_SETUP_SYNC_CONN,
enhanced/legacy 由固件对 HCI_Read_Local_Supported_Commands 的应答
(commands[29] & 0x08)决定。但 hci_qca 对**宽频语音**有按 SoC 写死的默认:

```c
/* Wideband speech support must be set per driver since it can't
 * be queried via hci. Same with the valid le states quirk.
 */
if (data->capabilities & QCA_CAP_WIDEBAND_SPEECH)
    hci_set_quirk(hdev, HCI_QUIRK_WIDEBAND_SPEECH_SUPPORTED);   /* hci_qca.c:2566 */
```

`HCI_QUIRK_WIDEBAND_SPEECH_SUPPORTED` 的内核效果:

- mgmt.c:830  Read Controller Info 上报 `MGMT_SETTING_WIDEBAND_SPEECH`
- mgmt.c:4494 `MGMT_OP_SET_WIDEBAND_SPEECH` 可用(否则 NOT_SUPPORTED),
  bluetoothd 据此在 HFP codec 协商里启用 mSBC/transparent

SoC capability 表(hci_qca.c:2108-2222 `qca_soc_data_*`):

| SoC | WIDEBAND_SPEECH | HFP_HW_OFFLOAD |
|---|---|---|
| qca2066, qca6390, wcn3990, wcn6855, wcn7850 | ✓ | ✓ |
| wcn3950, wcn3988, wcn3991, wcn3998, wcn6750 | ✓ | ✗ |

`HFP_HW_OFFLOAD` 不是 quirk,是 `qcadev->support_hfp_hw_offload` 驱动内部标志
(hci_qca.c:2572-2573)。

hci_qca 设置的全部 quirk 清单:

| quirk | 位置 | 条件 |
|---|---|---|
| `SIMULTANEOUS_DISCOVERY` | hci_qca.c:1939 | 固件打补丁后,无条件 |
| `BDADDR_PROPERTY_BROKEN` | hci_qca.c:1990 | 按芯片型号 |
| `NON_PERSISTENT_SETUP` | hci_qca.c:2557 | power_ctrl_enabled |
| `WIDEBAND_SPEECH_SUPPORTED` | hci_qca.c:2566 | QCA_CAP_WIDEBAND_SPEECH |
| `BROKEN_LE_STATES` | hci_qca.c:2570 | 无 QCA_CAP_VALID_LE_STATES |

### 2.4 用户态设置入口:SCO socket 的三个 sockopt

| sockopt | 设置方 | 内容 | 内核存储 |
|---|---|---|---|
| `BT_VOICE` | PipeWire / 用户程序 | `struct bt_voice { setting }` | `sco_pi(sk)->setting` |
| `BT_CODEC` | PipeWire(offload 时) | `struct bt_codecs { id, data_path_id }` | `sco_pi(sk)->codec` |
| `BT_DEFER_SETUP` | oFono(listen 侧) | 0/1,延迟发 Accept | `BT_SK_DEFER_SETUP` flag |

- PipeWire backend-native: mSBC 等编码时设 `BT_VOICE_TRANSPARENT`
  `../pipewire/spa/plugins/bluez5/backend-native.c:2616-2655` sco_create_socket 内
  `voice_config.setting = BT_VOICE_TRANSPARENT; setsockopt(SOL_BLUETOOTH, BT_VOICE, ...)`
- PipeWire offload 场景额外设 `BT_CODEC`:
  `../pipewire/spa/plugins/bluez5/backend-native.c:306` sco_offload_btcodec()
  `codecs->codecs[0].id = BT_CODEC_MSBC/CVSD; data_path_id = hfphsp_sco_datapath;`
  `setsockopt(SOL_BLUETOOTH, BT_CODEC, ...)`(backend-native.c:327)
- oFono listen 侧: `BT_DEFER_SETUP = 1`(`../ofono/src/handsfree-audio.c:221`),
  并 `getsockopt(BT_VOICE)` 探测是否 transparent(`handsfree-audio.c:232`)
- oFono connect 侧(方式 B)不设 voice,用内核默认

### 2.5 默认值(用户态什么都不设时)

`net/bluetooth/sco.c:615 sco_sock_create()` 创建 socket 时内核预填:

```c
sco_pi(sk)->setting = BT_VOICE_CVSD_16BIT;   /* sco.c:626 */
sco_pi(sk)->codec.id = BT_CODEC_CVSD;        /* sco.c:627 */
sco_pi(sk)->codec.cid = 0xffff;              /* 厂商 codec 未指定 */
sco_pi(sk)->codec.vid = 0xffff;
sco_pi(sk)->codec.data_path = 0x00;          /* 不 offload */
```

即:默认 CVSD 16bit、enhanced 分支按 `BT_CODEC_CVSD` 组包(16000/format 2)。

### 2.6 内核 sockopt 处理(net/bluetooth/sco.c sco_sock_setsockopt)

- `case BT_VOICE`(sco.c:1004):
  `sco_pi(sk)->setting = voice.setting;`
  且若 `(setting & SCO_AIRMODE_MASK) == SCO_AIRMODE_TRANSP` **且** 芯片 enhanced capable:
  `sco_pi(sk)->codec.id = BT_CODEC_TRANSPARENT`(sco.c:1025)
- `case BT_CODEC`(sco.c:1037):
  要求 `HCI_OFFLOAD_CODECS_ENABLED` 与 `hdev->get_data_path_id` 存在,
  校验 `struct bt_codecs` 后写入 `sco_pi(sk)->codec`(含 data_path)

### 2.7 内核组包(outgoing 建链方向)

发起方建链时,内核按 codec 选择参数并组包,enhanced 与 legacy 分派如下。
incoming(accept 侧)见 2.8。

#### 2.7.1 enhanced(hci_conn.c:278 hci_enhanced_setup_sync)

按 `conn->codec.id` 分派(参数表见 2.7.3):

| codec.id | 参数表 | coding format | bandwidth | frame size | transport unit |
|---|---|---|---|---|---|
| `BT_CODEC_MSBC` | esco_param_msbc | 0x05(mSBC) | 32000 | 60 | 1 |
| `BT_CODEC_TRANSPARENT` | esco_param_msbc | 0x03 | 0x1f40 | 60 | 1 |
| `BT_CODEC_CVSD` | esco_param_cvsd(对端 eSCO 能力) / sco_param_cvsd(纯 SCO) | 2(μ-law) | 16000 | 60 | 16 |

公共字段: `cp.tx_bandwidth = cp.rx_bandwidth = 0x00001f40`(:291-292);
`in/out_data_path = conn->codec.data_path`(offload 数据通路);
`pkt_type / max_latency / retrans_effort` 取自参数表(:393-395);
`configure_datapath_sync(hdev, &conn->codec)`(:286,offload 时配置厂商数据通路)。

#### 2.7.2 legacy(hci_conn.c:402 hci_setup_sync_conn)

- `tx_bandwidth = rx_bandwidth = 0x00001f40`(:415-416)
- `voice_setting = conn->setting`(用户态 BT_VOICE 值,:417)
- 按 `conn->setting & SCO_AIRMODE_MASK` 选表:
  TRANSP → esco_param_msbc;CVSD → esco_param_cvsd / sco_param_cvsd
- `pkt_type / max_latency / retrans_effort` 取自表(:445-447)
- `hci_send_cmd(HCI_OP_SETUP_SYNC_CONN)`

#### 2.7.3 内核参数表(hci_conn.c:38-67)

```c
struct sco_param { u16 pkt_type; u16 max_latency; u8 retrans_effort; };  // :38-42

esco_param_cvsd[]  // :49  S3~D0 共 5 档, retrans_effort 全部 0x01
  { EDR_ESCO_MASK & ~ESCO_2EV3, 0x000a, 0x01 }  /* S3 */
  ...
sco_param_cvsd[]   // :57  纯 SCO,2 档, retrans 0xff
  { EDR_ESCO_MASK | ESCO_HV3, 0xffff, 0xff }    /* D1 */
  ...
esco_param_msbc[]  // :62  T2/T1, retrans 0x02
  { EDR_ESCO_MASK & ~ESCO_2EV3, 0x000d, 0x02 }  /* T2 */
  { EDR_ESCO_MASK | ESCO_EV3,   0x0008, 0x02 }  /* T1 */
```

重试用 `find_next_esco_param()`(:216)——建链失败重试时按 `conn->attempt`
依次尝试下一档参数(如先 T2 后 T1)。

### 2.8 Accept 侧参数:非 defer 与 defer 两条路

incoming 方向 accept 时,内核也要组 `HCI_OP_ACCEPT_SYNC_CONN_REQ` 的参数,
取决于 defer 与否;纯 SCO 另有分支。

#### 2.8.1 非 defer

**非 defer**——`hci_conn_request_evt`(hci_event.c:3380-3419,eSCO 分支 :3411),
事件一到立即发 accept,用户态零参与:

```c
cp.pkt_type       = conn->pkt_type;
cp.tx_bandwidth   = 0x00001f40;           /* 写死 */
cp.rx_bandwidth   = 0x00001f40;           /* 写死 */
cp.max_latency    = 0xffff;               /* 写死,最宽松 */
cp.content_format = hdev->voice_setting;  /* ★ 芯片默认值 */
cp.retrans_effort = 0xff;                 /* 写死 */
```

`hdev->voice_setting` 来源:内核 init 发 `HCI_OP_READ_VOICE_SETTING`
(hci_read_voice_setting_sync,hci_sync.c:3920)→ 芯片回默认值 →
`hci_cc_read_voice_setting` 存入(hci_event.c:573)。
**与 codec 协商无关**——不用 defer 的 HFP,mSBC 协商等于白谈。

#### 2.8.2 defer

**defer**——`sco_conn_defer_accept`(sco.c:893-935),授权触发时传入
socket 自己的 voice setting(sco.c:956):

```c
cp.pkt_type       = conn->pkt_type;
cp.tx_bandwidth   = 0x00001f40;           /* 写死 */
cp.rx_bandwidth   = 0x00001f40;           /* 写死 */
cp.content_format = setting;              /* ★ socket 的 BT_VOICE,codec 映射而来 */

switch (setting & SCO_AIRMODE_MASK) {     /* ★ airmode 决定延迟/重传 */
case SCO_AIRMODE_TRANSP:                  /* mSBC */
    cp.max_latency    = 0x0008(2EV3)/0x000D;
    cp.retrans_effort = 0x02;             /* T1/T2 参数 */
    break;
case SCO_AIRMODE_CVSD:
    cp.max_latency    = 0xffff;
    cp.retrans_effort = 0xff;
}
```

| accept 参数 | 非 defer(hci_event.c:3399-3410) | defer(sco.c:909-935) |
|---|---|---|
| content_format | `hdev->voice_setting`(芯片默认) | `pi->setting`(socket BT_VOICE,codec 映射) |
| max_latency | 0xffff 写死 | TRANSP→0x0008/0x000D;CVSD→0xffff |
| retrans_effort | 0xff 写死 | TRANSP→0x02;CVSD→0xff |
| tx/rx_bandwidth | 0x1f40 写死 | 0x1f40 写死 |

结论:defer 的本质是**给用户态一个"accept 发出前"的窗口**,把协商好的
codec(BT_VOICE_TRANSPARENT)设到 socket 上再授权。

#### 2.8.3 纯 SCO(无 eSCO 能力)

纯 SCO(!lmp_esco_capable): 发 `HCI_OP_ACCEPT_CONN_REQ`(hci_event.c:3386)。

#### 2.8.4 codec → setting 映射的两家实现

| | oFono | PipeWire native |
|---|---|---|
| 映射 | `codec2setting()`(handsfree-audio.c:70,switch) | `media_codec->id != CVSD` 布尔 |
| connect 侧 | `apply_settings_from_codec`(:502) | `sco_create_socket`(:2637-2644) |
| listen 侧 | `apply_settings_from_codec`(:181) | `sco_listen_event`(:3035-3043) |
| defer listen | 总开(:221) | 尝试开,失败降级普通模式(:3096-3100) |
| 授权 read 1 字节 | PipeWire(ofono backend)读 | 自己读(:3047) |

PipeWire native listen 侧(sco_listen,:3075-3104;AG 角色、对端主动):
`socket + bind(BDADDR_ANY) + BT_DEFER_SETUP + listen(1)`,accept 后 defer
模式下先设 BT_VOICE_TRANSPARENT(非 CVSD 时)再 read 1 字节授权;
降级为普通模式时 accept 走内核非 defer 路径,mSBC incoming 表达不了。

### 2.9 PipeWire 侧配置链:codec 与 data path 怎么设的

**codec 选择(CVSD vs mSBC)= RFCOMM 上 HFP 协商出来的:**

1. 编译期 codec 表(按"best"顺序):`media-codecs.c` 的 `monitor->media_codecs`,
   `backend->codecs = spa_bt_get_media_codecs(monitor)`(backend-native.c:4266)
2. 运行时过滤:CVSD 强制可用,mSBC 需配置 enabled
   (`is_media_codec_enabled`,bluez5-dbus.c:510)
3. SLC 协商(backend-native.c:1120-1210):
   - `AT+BRSF` 双方 feature bit 确认支持 codec negotiation(:1123-1140)
   - `AT+BAC=<list>` 耳机报支持集 → 取交集 `supported_codec_list`(:1149-1173)
   - `AT+CMER` 后 `codec_list_best()` 按表序取最优(:1189-1192,表序即优先级 :291-295)
     → 发 `+BCS: n` 要求切换(:1201-1206);耳机回 `AT+BCS=n` 确认(:1228)
4. 结果落到 transport:`rfcomm_new_transport(codec)` → `t->media_codec = codec`
   (backend-native.c:404)

**建 SCO 时翻译成 sockopt(backend-native.c:2657-2675):**

```c
bool encoded = t->media_codec->id != SPA_BLUETOOTH_AUDIO_CODEC_CVSD;  /* mSBC? */
sock = sco_create_socket(backend, d->adapter, encoded);
```

`sco_create_socket()`(backend-native.c:2616-2655):
- mSBC:`voice_config.setting = BT_VOICE_TRANSPARENT; setsockopt(BT_VOICE)`(:2640-2644)
- CVSD:不设,用内核默认 `BT_VOICE_CVSD_16BIT`
- 然后 `sco_offload_btcodec(backend, sock, transparent)`(:2652)

**BT_CODEC —— 只有 offload 才设:**

```
parse_sco_datapath()(backend-native.c:4208-4213):
    hfphsp_sco_datapath = HFP_SCO_DEFAULT_DATAPATH;   /* 默认:不 offload */
    spa_atou32(spa_dict_lookup(info, "bluez5.hw-offload-datapath"), ...)

sco_offload_btcodec()(backend-native.c:306-329):
    if (hfphsp_sco_datapath == HFP_SCO_DEFAULT_DATAPATH) return;   /* :312 */
    codecs->codecs[0].id           = msbc ? BT_CODEC_MSBC : BT_CODEC_CVSD;  /* :320-322 */
    codecs->codecs[0].data_path_id = hfphsp_sco_datapath;                  /* :324 */
    setsockopt(sock, SOL_BLUETOOTH, BT_CODEC, ...)                         /* :327 */
```

即:非 offload 时上层用 `BT_VOICE` 表达 codec 选择,`BT_CODEC` 由内核从
airmode 派生;offload 时上层才直接设 `BT_CODEC{id, data_path_id}`。

**内核最终映射(PipeWire 设置 → HCI 命令字段):**

| PipeWire 实际做的 | 内核得到 | HCI 命令字段 |
|---|---|---|
| 协商出 CVSD,啥都不设 | 默认 setting=CVSD_16BIT,codec.id=BT_CODEC_CVSD(sco.c:626-627) | coding format 2,bandwidth 16000 |
| mSBC 非 offload,设 `BT_VOICE_TRANSPARENT` | airmode TRANSP → codec.id=BT_CODEC_TRANSPARENT(sco.c:1025) | coding format 0x03,mSBC 帧透传 |
| mSBC + offload,设 `BT_CODEC{MSBC, data_path_id}` | sco_pi(sk)->codec 原样(sco.c:1093) | coding format 0x05,bandwidth 32000,in/out_data_path=配置值 |

data_path 两个 id 的分工:命令里的 `in/out_data_path` = PipeWire 配置值;
`HCI_Configure_Data_Path` 用的 `data_path_id` = 驱动回调(QCA 固定 1,
hci_qca.c:1893;且 QCA 不发该命令,hci_qca.c:1904-1907)。

### 2.10 offload 全场景对照与 data_path 语义

**四宫格(coding format 由内核按 codec 分支写死,data_path 由用户态设):**

| 场景 | data_path | coding format | 谁做 encode | 数据走哪 |
|---|---|---|---|---|
| CVSD,非 offload | 0 | 0x02 | controller(CVSD 编码) | HCI SCO 包 |
| CVSD,offload | 1 | 0x02(不变) | controller(CVSD 编码) | 非 HCI data path |
| mSBC,非 offload | 0 | 0x03(transparent) | PipeWire(msbc 插件) | HCI SCO 包,controller 透传 |
| mSBC,offload | 1 | 0x05(mSBC) | controller(硬件 mSBC) | 非 HCI data path |

注意:PipeWire 的 CVSD 插件其实不做 CVSD 编码——`codec_encode` 是纯拷贝
(`spa_memmove`,hfp-codec-cvsd.c:127)。**CVSD 编码始终由 controller 做**,
PipeWire 只送 8kHz S16 PCM。真正的软件编码只发生在 mSBC 非 offload。

**"data_path = 1" 的两处来源(别混):**

| 出现位置 | 谁设 | 怎么设 |
|---|---|---|
| HCI 命令 `in/out_data_path` 字段 | 用户(PipeWire 配置) | `bluez5.hw-offload-datapath=1` → parse_sco_datapath(backend-native.c:4208-4213)→ BT_CODEC sockopt(backend-native.c:324-327)→ 内核透传(hci_conn.c 各分支) |
| `HCI_Configure_Data_Path` 的 `data_path_id` | 驱动写死 | QCA:`qca_get_data_path_id` 恒返 1(hci_qca.c:1893-1896);其他厂商驱动各自实现 |

默认值 `HFP_SCO_DEFAULT_DATAPATH = 0`(defs.h:141)= 不 offload,走 HCI。

**非 HCI 通路的物理含义:**

- data_path=0:音频作为 SCO 数据包经 HCI(USB/UART)进出主机,主机 CPU 搬运
- data_path≠0:数据不经 HCI,在 controller 与音频子系统间直连——典型为
  I2S/PCM/SLIMbus(手机 SoC 上蓝牙直连音频 DSP/modem),或集成 SoC 内部总线
- **具体走哪条、id 对应什么,由 controller/厂商决定,主机不感知**:
  内核与 HCI 协议层面 id 都是裸 `u8` 透传,无枚举(hci.h:1017-1018, 1408);
  内核只透传用户值 + 调驱动回调取硬件 id(hci_conn.c:256)
- QCA 不需要 `HCI_Configure_Data_Path`(`get_codec_config_data = NULL`,
  hci_qca.c:1904-1907):controller 看到命令里的 data_path 字段自己接上内部通路

```
非 offload: PipeWire → 主机 → HCI(USB/UART) → controller → 空中
                                          ↑ 主机参与每一字节
offload:     PipeWire(仅协商) → controller 内部通路 ↔ DSP/modem → 空中
             主机只发"用通路 n"的命令,音频数据不进主机
```

### 2.11 backend-ofono 的差异与 offload 缺失

**backend-native vs backend-ofono 对比(内核侧完全相同,只有用户态分工不同):**

| | backend-native | backend-ofono |
|---|---|---|
| SCO socket 谁建 | PipeWire | oFono(connect 或 listen) |
| `BT_VOICE` 谁设 | PipeWire(mSBC 时 TRANSPARENT,backend-native.c:2640) | oFono(`apply_settings_from_codec`) |
| `BT_CODEC` / offload | 支持(`bluez5.hw-offload-datapath`) | **完全没有** |
| codec 协商 | PipeWire 自己的 AT 引擎(BRSF/BAC/BCS) | oFono 的 g_at_server,PipeWire 被动拿结果 |
| `BT_DEFER_SETUP` | 不用 | oFono listen 侧设(handsfree-audio.c:221),PipeWire read 1 字节授权 |

**oFono 的 codec → BT_VOICE 映射:**

```c
static uint16_t codec2setting(uint8_t codec)              /* handsfree-audio.c:70 */
{
    case HFP_CODEC_CVSD: return BT_VOICE_CVSD_16BIT;
    default:             return BT_VOICE_TRANSPARENT;     /* mSBC 等 */
}

static ofono_bool_t apply_settings_from_codec(int fd, uint8_t codec)  /* :80 */
{
    /* CVSD is the default, no need to set BT_VOICE. */
    if (voice.setting == BT_VOICE_CVSD_16BIT)
        return TRUE;
    setsockopt(fd, SOL_BLUETOOTH, BT_VOICE, ...);          /* :91 只有 mSBC 才设 */
}
```

调用点:accept 侧(handsfree-audio.c:181)、connect 侧(handsfree-audio.c:502)。
PipeWire backend-ofono.c 一个 setsockopt 都没有。
mSBC 编码仍由 PipeWire 的 msbc 插件做(codec 结果经 D-Bus 的 codec byte 传回,
backend-ofono.c:550 `spa_bt_get_hfp_codec` 映射)。

**oFono 场景走不了 offload 的三层原因:**

1. oFono 全树无 offload/BT_CODEC/data_path 代码(它只认 `BT_VOICE`/`BT_DEFER_SETUP`)
2. **Acquire 方向**:fd 到手时已 BT_CONNECTED,`BT_CODEC` 要求状态
   BT_OPEN/BT_BOUND/BT_CONNECT2(sco.c:1045-1047)→ 设了被内核拒绝;
   设置窗口在 oFono 手里,oFono 不设
3. **NewConnection 方向**:fd 在 BT_CONNECT2 时到手,状态允许设,但
   incoming 方向内核发的是 `HCI_ACCEPT_SYNC_CONN_REQ`(sco_conn_defer_accept,
   sco.c:893),参数只有 bdaddr/pkt_type/bandwidth/max_latency/content_format/
   retrans_effort——**没有 coding format、没有 data_path 字段**,
   设了也没地方用

结论:offload 是**发起方(outgoing)专属**功能,必须由建 socket 的一方在
connect 前设 `BT_CODEC`。backend-ofono 架构里 connect 前 socket 在 oFono
手里,所以 offload 只存在于 backend-native 生态。

## 3. 相关测试设施对照

- btdev 模拟芯片同样声明 enhanced 支持:`emulator/btdev.c:3648`(`commands[29] |= 0x08`),
  并实现命令处理:`emulator/btdev.c:3002 cmd_enhanced_setup_sync_conn`(参数校验:
  `tx_coding_format[0] > 5` 返回 INVALID_PARAMETERS)
- sco-tester 用 hciemu(vhci + bthost)端到端验证上文内核路径,参数选择逻辑
  (enhanced/legacy、TRANSP/CVSD)全部由真实内核代码执行,只有"芯片"被 btdev 替换。
