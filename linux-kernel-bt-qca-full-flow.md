# Linux 蓝牙子系统工作流程 — 从零开始讲

> 通俗语言 + 技术细节。重点讲**模块之间怎么连起来的**。
>
> 节与节之间大量前后呼应——每节结尾告诉你"这些后面哪里用到"，每节开头帮你"回顾前面讲了什么"。

---

## 目录

1. [先理解大图：分几层，谁跟谁说话](#一先理解大图)
2. [怎么找到蓝牙芯片 — Probe](#二probe)
3. [Power On — 从 probe 排队到芯片跑起来](#三power-on)
4. [数据通路 — App 数据怎么下去，芯片数据怎么上来](#四数据通路)
5. [附录：关键数据结构速查](#附录)

---

## 一、先理解大图

先搞清楚内核里蓝牙代码分哪几块、它们怎么互相说话。不管后面讲多细，回来对着这张图看就不会迷路。

### 1.1 三层结构

```
┌──────────────────────────────────────────────┐
│  net/bluetooth/                              │
│  (hci_core.c, hci_sock.c, l2cap_core.c ...) │  ← 核心层
│  跟芯片型号无关。管理协议、提供 socket。     │
│                                              │
│  代表数据结构: struct hci_dev                │
├──────────────────────────────────────────────┤
│  drivers/bluetooth/hci_ldisc.c               │  ← 传输层
│  drivers/bluetooth/hci_serdev.c              │     把 HCI 包搬到串口
│                                              │
│  代表数据结构: struct hci_uart               │
├──────────────────────────────────────────────┤
│  drivers/bluetooth/hci_qca.c                 │  ← 驱动层
│  drivers/bluetooth/btqca.c                   │     控制 QCA 芯片
│                                              │
│  代表数据结构: struct qca_data               │
│               struct qca_serdev              │
└──────────────────────────────────────────────┘
```

### 1.2 怎么连？层与层不直接认识，通过"函数指针"说话

核心层**不知道也不关心**下面是 QCA 芯片还是其他什么芯片。

它只跟 `struct hci_dev` 里的几个**函数指针**说话：

```c
struct hci_dev {
    int  (*open)(struct hci_dev *hdev);     // "请你开机"
    int  (*close)(struct hci_dev *hdev);    // "请你关机"
    int  (*send)(struct hci_dev *hdev, struct sk_buff *skb); // "把数据发出去"
    int  (*setup)(struct hci_dev *hdev);    // "请你初始化（下载固件等）"
};
```

这就是**核心层和驱动层之间的"合同"**。驱动在 probe 阶段把自己的函数填进这些指针（第二节讲细节），之后核心层调用 `hdev->send(skb)` 时，实际跑的就是驱动提供的函数。

传输层夹在中间，做一个转发器：

```
hdev->send(skb)               ← 核心层调这个
  = hci_uart_send_frame()     ← 先到传输层
    → hu->proto->enqueue()    ← 再从传输层到驱动层
      = qca_enqueue()         ← 最终到 QCA 驱动
```

为什么多这一层？因为 UART 串口传输有一些通用逻辑（波特率、流控、write_work 调度），所有 UART 蓝牙芯片都一样，提取到 `hci_ldisc.c` / `hci_serdev.c` 里复用，QCA 驱动只管自己特有的部分（IBS 休眠、固件下载）。

### 1.3 完整路径速览（后面的章节都在解释这张图）

拿"用户 App 发一包数据给蓝牙耳机"举例，数据穿过以下模块：

```
App write(sock, data)
  → [L2CAP socket]    l2cap_sock_sendmsg()       ← BTPROTO_L2CAP socket (第4.2节)
  → [L2CAP core]      l2cap_chan_send()           ← 加 L2CAP header (第4.3节)
  → [HCI core]        hci_send_acl()              ← 加 HCI ACL header (第4.3节)
  → [HCI core]        hci_tx_work()               ← 异步 work 调度发送 (第4.3节)
  → [HCI core]        hci_send_frame()            ← 统一发送入口 (第4.3节)
  → [callback]        hdev->send(skb)             ← 函数指针 ★ 第2节注册,这里调用
  → [HCI UART]        hci_uart_send_frame()       ← 传输层转发 (第4.3节)
  → [callback]        hu->proto->enqueue(hu, skb) ← 函数指针 ★ 第2节注册,这里调用
  → [QCA driver]      qca_enqueue()               ← 加 H4 header, IBS 处理 (第4.3节)
  → [serdev]          serdev_device_write_buf()   ← 写串口 (第4.3节)
  → [硬件]            UART TX → 芯片 RX
```

**上行同理（芯片 → App）：**

```
[硬件] UART RX
  → [serdev callback]  hci_uart_receive_buf()    ← 第2节注册的串口回调
  → [callback]         hu->proto->recv() = qca_recv() ← ★ 跳到驱动层
  → [QCA driver]       h4_recv_buf() → 组包
  → [call HCI core]    hci_recv_frame(hdev, skb) ← ★★ 驱动主动调核心层的函数
  → [HCI core]         rx_q 入队 → rx_work 异步处理
  → [HCI core]         hci_acldata_packet() → l2cap_recv_acldata()
  → [L2CAP core]       l2cap_recv_frame() → l2cap_data_channel()
  → [L2CAP socket]     l2cap_sock_recv_cb() → __sock_queue_rcv_skb()
  → App: read() 被唤醒, 拿到数据
```

**注意看三个 ★ 处的"函数指针"**——这就是层与层之间的连接点。

---

## 二、Probe：怎么找到蓝牙芯片

> **这节干了什么**：把芯片和内核"对上号"，分配软件结构体，注册函数指针（填合同），最后在 workqueue 里排队一个 power_on 任务。
>
> **这节的东西后面哪里用到**：
> - 注册的回调函数 → 第 3 节 power_on 时 `hdev->open/close/setup` 被调用
> - 注册的回调函数 → 第 4 节数据通路时 `hdev->send` 被调用, `qca_recv` 被调用
> - 末尾的 `queue_work(power_on)` → 第 3 节整个流程的起点

### 2.1 设备树匹配

内核启动时，你的板子有一个 **设备树 (.dts)** 描述了硬件。里面有一项大概是这样的：

```
serial@12345 {
    bluetooth {
        compatible = "qcom,wcn3990-bt";
        enable-gpios = <&gpio 10 0>;
        firmware-name = "qca/crbtfw21.tlv";
        max-speed = <3000000>;
    };
};
```

内核的串口子系统发现串口上挂了一个设备，拿着 `compatible = "qcom,wcn3990-bt"` 去找驱动。

在 `drivers/bluetooth/hci_qca.c:2813`，驱动声明了它能处理的型号：

```c
static struct serdev_device_driver qca_serdev_driver = {
    .probe = qca_serdev_probe,            // 匹配成功就调这个函数
    .driver = {
        .name = "hci_uart_qca",
        .of_match_table = qca_bluetooth_of_match,  // 支持的 compatible 列表
    },
};
```

匹配上 → 内核调用 `qca_serdev_probe()`。

### 2.2 `qca_serdev_probe()` 内部——读配置 + 拿电源

**第 1 步：读出配置**

从设备树读出并保存：
- `soc_type` — 芯片型号（WCN3990 / QCA6390 / ROME ...）
- `firmware-name` — 固件名列表（后面第 3 节 `qca_uart_setup` 下载固件时用）
- `max-speed` — 通信波特率（后面第 3 节 `qca_setup` 设置速度时用）

**第 2 步：获取电源引脚和时钟**

```c
switch (soc_type) {
case QCA_WCN3990:
    qcadev->bt_power->pwrseq = devm_pwrseq_get(&serdev->dev, "bluetooth");
    if (!pwrseq) {
        qca_init_regulators(qcadev);         // 获取电压调节器
        qcadev->bt_en = devm_gpiod_get_optional(..., "enable", ...); // BT_EN 引脚
    }
    qcadev->susclk = devm_clk_get_optional(...); // 32KHz 休眠时钟
    break;
case QCA_ROME:
    qcadev->bt_en = devm_gpiod_get_optional(..., "enable", ...);
    break;
}
```

**probe 阶段只是"获取这些资源"的引用（指针），并没有操作它们**。真正上电、拉高 GPIO、开时钟的代码在后面第 3 节的 `qca_regulator_init()` 和 `qca_regulator_enable()` 里。

### 2.3 注册到核心层 ★ 最关键的连接步骤

```c
hci_uart_register_device(&qcadev->serdev_hu, &qca_proto);
```

这行代码展开后，就是下面这个函数。**每一行注释说明了它与后面的哪个章节有关**：

```c
// drivers/bluetooth/hci_serdev.c:303
int hci_uart_register_device_priv(struct hci_uart *hu,
                                   const struct hci_uart_proto *p)
{
    // (a) 设置串口接收回调 —— 串口来数据时调谁
    //     第 4.4 节: 芯片发数据上来 → 中断触发 → 这个回调被调用
    serdev_device_set_client_ops(hu->serdev, &hci_serdev_client_ops);
    //   hci_serdev_client_ops = {
    //       .receive_buf = hci_uart_receive_buf,  ← 数据上行入口
    //       .write_wakeup = hci_uart_write_wakeup, ← 串口可写时通知
    //   };

    // (b) 打开串口 —— 只是建立连接,还没配速度
    serdev_device_open(hu->serdev);

    // (c) 调 qca_open() —— 分配 qca_data, 初始化 IBS 要用到的 wq/timer
    //     见下方 2.4 节
    err = p->open(hu);   // p = &qca_proto → 调 qca_open()

    hu->proto = p;       // 保存协议指针

    // (d) 分配 hci_dev —— 这个函数内部初始化了 power_on work
    //     第 3.2 节: INIT_WORK(&hdev->power_on, hci_power_on) 在这做的
    hdev = hci_alloc_dev_priv(sizeof(...));

    // (e) ★★★ 填函数指针 —— 核心层和驱动签"合同" ★★★
    //     第 3 节: hdev->open 被 power_on 流程调用
    //     第 3 节: hdev->setup 被 power_on 流程调用,触发 qca_setup
    //     第 3 节: hdev->close 被 power_off 流程调用
    //     第 4 节: hdev->send 被每次发送数据时调用
    hdev->open  = hci_uart_open;          // 核心层调 open → 到传输层
    hdev->close = hci_uart_close;         // 核心层调 close → 到传输层
    hdev->send  = hci_uart_send_frame;    // 核心层调 send → 到传输层
    hdev->setup = hci_uart_setup;         // 核心层调 setup → 到传输层

    // (f) 把 hci_uart 附在 hci_dev 上
    //     传输层函数（hci_uart_open 等）内部通过这个拿回 hu,进而拿到 qca_data
    hci_set_drvdata(hdev, hu);

    // (g) ★★★ 注册 —— 从此核心层知道有这个设备了 ★★★
    //     内部末尾调用 queue_work(power_on), 见下方 2.3.1
    return hci_register_dev(hdev);
}
```

#### 2.3.1 probe → power_on 的连接点

```c
// net/bluetooth/hci_core.c:2672 (hci_register_dev() 的最后一行)
queue_work(hdev->req_workqueue, &hdev->power_on);
```

**翻译**："往待办队列里加一条——跑 `hci_power_on` 这个函数。" 注册完就排队，但不立刻执行。等调度器安排上才跑。

**这行代码就是第二节到第三节的连接点。** probe 到此结束，后续是异步的 power_on 流程。

### 2.4 probe 阶段辅助：`qca_open()` —— 只分配软件,不动机器

`qca_open()` 在 probe 的 (c) 步被调，但它**不操作硬件**,只分配驱动层自己用的软件资源：

```c
// drivers/bluetooth/hci_qca.c:577
qca_open(hu) {
    qca = kzalloc(sizeof(*qca));          // 分配 qca_data

    // 这些队列在第 4 节数据通路中用到：
    skb_queue_head_init(&qca->txq);       // 发送队列——存待发数据
    skb_queue_head_init(&qca->tx_wait_q); // 等待队列——芯片睡着时暂存

    qca->qca_wq = alloc_ordered_workqueue("qca_wq", 0);

    // 这些 work item 在第 4 节的 IBS 休眠协议中用到：
    INIT_WORK(&qca->ws_awake_device, qca_wq_awake_device);  // 唤醒芯片
    INIT_WORK(&qca->ws_awake_rx,     qca_wq_awake_rx);      // 响应芯片唤醒

    // timer 在第 4 节用到：
    timer_setup(&qca->wake_retrans_timer, hci_ibs_wake_retrans_timeout, 0);
    timer_setup(&qca->tx_idle_timer, hci_ibs_tx_idle_timeout, 0);

    // 保存速度配置，第 3 节 qca_setup 里会用它设波特率
    hu->init_speed = qcadev->init_speed;
    hu->oper_speed = qcadev->oper_speed;

    // IBS 初始状态——双方都睡着,发数据前要唤醒
    qca->tx_ibs_state = HCI_IBS_TX_ASLEEP;
    qca->rx_ibs_state = HCI_IBS_RX_ASLEEP;
}
```

**怎么从核心层拿到这些私有数据？** `hci_uart_open()` / `hci_uart_send_frame()` 先通过 `hci_get_drvdata(hdev)` 拿到 `struct hci_uart *hu`，再从 `hu` 关联拿到 `qca_data`。

### 2.5 probe 的全貌：你现在知道了什么

```
probe 完成时，已经做了什么：

✅ 解析了设备树 (芯片型号, 电源引脚, 固件名)
✅ 打开了串口
✅ 分配了软件结构体:
     核心层的 hci_dev (hci_alloc_dev_priv)
     传输层的 hci_uart
     驱动层的 qca_data (qca_open)
✅ 填好了函数指针 (核心层 → 传输层, 传输层 → 驱动层)
✅ 排队了 power_on work — 等待执行

还缺什么（后面章节的事）：

❌ 芯片还没通电 (第 3 节 power_on)
❌ 固件还没下载 (第 3 节 qca_setup)
❌ 芯片还没初始化 (第 3 节 hci_init_sync)
❌ 没数据收发 (第 4 节)
```

---

## 三、Power On——从 probe 排队到芯片跑起来

> **回顾前面**：第二节 probe 末尾调了 `queue_work(hdev->req_workqueue, &hdev->power_on)`。这节就是从那行代码开始，讲清楚这个排队的任务到底做了什么。
>
> **这节干了什么**：给芯片通电 → 下载固件 → 发标准 HCI 命令初始化 → 通知用户空间"好了"。
>
> **这节的东西后面哪里用到**：
> - `hdev->open/setup/close` 这些回调 → 都是在第 2 节注册的,这节第一次被调用
> - 固件下载时走 `hci_send_frame()` → `hdev->send()` → 这个数据路径在第 4 节会被 App 大量使用
> - `mgmt_power_on` 通知用户空间 → bluetoothd 收到后会通过 HCI socket 发控制命令,走第 4 节的数据通路

### 3.1 触发：三种入口,汇入同一条路

| 触发来源 | 场景 | 入口函数 |
|---------|------|---------|
| **probe 自动** | 驱动 probe 完，`hci_register_dev` 末尾 auto-trigger | `hci_power_on()` |
| **用户打开蓝牙开关** | bluetoothd 发送 `MGMT_OP_SET_POWERED` | `set_powered()` → `hci_power_on_sync()` |
| **旧 ioctl** | 老工具调用 | `hci_dev_open()` |

三条路最终汇合到 **`hci_dev_open_sync()`**。

### 3.2 锁：保证不会同时开关

`power_on` work 跑在 `hdev->req_workqueue` 上（第 2.3.1 节排的队）。进入 `hci_dev_open_sync` 之前要拿锁：

```c
// net/bluetooth/hci_core.c:423
hci_dev_do_open(hdev) {
    hci_req_sync_lock(hdev);     // = mutex_lock(&hdev->req_lock)

    ret = hci_dev_open_sync(hdev);  // 核心逻辑

    hci_req_sync_unlock(hdev);
}
```

**为什么这安全？** `power_on`、`power_off`、`cmd_sync_work` 都在同一个 `req_workqueue`（ordered）上排队，又共用同一把锁，天然串行。

### 3.3 `hci_dev_open_sync()` 四大步

```c
// net/bluetooth/hci_sync.c:5144
int hci_dev_open_sync(struct hci_dev *hdev)
{
    // ============ 第 1 步: 调驱动的 open ============
    // 对 QCA 来说就是 hci_uart_open() → 只是恢复 flush 回调,返回 0
    // (真正的硬件上电在 setup 里)
    ret = hdev->open(hdev);    // ← 这是第 2 节注册的 hci_uart_open
    if (ret) goto done;

    set_bit(HCI_RUNNING, &hdev->flags);

    // ============ 第 2 步: 芯片初始化 ============
    // 这是最核心的部分,下面 3.4 展开
    ret = hci_dev_init_sync(hdev);

    // ============ 第 3 步: 通知所有人 ============
    if (!ret) {
        set_bit(HCI_UP, &hdev->flags);
        hci_sock_dev_event(hdev, HCI_DEV_UP);      // 通知 HCI socket 监听者
        mgmt_power_on(hdev);                        // 通知 bluetoothd ★
        // bluetoothd 收到通知后,就知道 hci0 上线了,可以开始扫描/配对
    }

    // ============ 第 4 步: 失败则关 ============
    if (ret) {
        hdev->close(hdev);  // ← 第 2 节注册的 hci_uart_close
                             //   → qca_close() → qca_power_off()
                             //     → 拉低 bt_en GPIO → 关 regulators
        clear_bit(HCI_RUNNING, &hdev->flags);
    }
    return ret;
}
```

### 3.4 `hci_dev_init_sync()` 内部——分两段

```
hci_dev_init_sync(hdev)               hci_sync.c:5092
  │
  ├─ set HCI_INIT (初始化进行中标志)
  │
  ├─ ★ 第一段: 驱动 setup（下固件,芯片特有的）
  │   hci_dev_setup_sync(hdev)
  │     └─ hdev->setup(hdev)          ← 第 2 节注册的回调, 见 3.5
  │         = hci_uart_setup()         传输层设速度
  │           └─ hu->proto->setup(hu)  跳转 ★
  │               = qca_setup()        驱动层, 见 3.5
  │
  ├─ ★ 第二段: 标准 HCI 初始化（所有蓝牙芯片通用的）
  │   hci_init_sync(hdev)
  │     ├─ HCI_Read_Local_Features      ← "你支持啥?"
  │     ├─ HCI_Read_BD_ADDR             ← "你蓝牙地址是啥?"
  │     ├─ HCI_Read_Buffer_Size         ← "你缓冲区多大?"
  │     ├─ HCI_LE_Read_Local_Features   ← "BLE 支持啥?"
  │     └─ ... 等等
  │     这些命令走: hci_send_cmd() → cmd_q → cmd_work
  │                 → hci_send_frame() → hdev->send
  │                 这个 hdev->send 就是第 2 节注册的,
  │                 跟第 4 节 App 发数据走同一个函数！
  │
  └─ clear HCI_INIT
```

**关键联系**：第一段是"芯片特有"的（不同芯片固件不同），第二段是"蓝牙标准"的（所有蓝牙 4.0+ 芯片都要回答这些问题）。第一段跑完后芯片才从一块硅片变成一个能应答 HCI 命令的蓝牙控制器。

### 3.5 `qca_setup()` 展开——驱动层怎么把芯片搞起来

**这里就是第 2 节注册的 `hu->proto->setup`（即 `qca_setup`）被调用的地方。**

```c
// drivers/bluetooth/hci_qca.c:1909
static int qca_setup(struct hci_uart *hu)
{
    // (1) 关 IBS — 固件下载期间不用休眠
    set_bit(QCA_IBS_DISABLED, &qca->flags);
    // 这个标志在第 4 节 qca_enqueue 里被检查：
    //   如果置位 → 不管芯片睡没睡,直接入 txq 发送
    //   如果清位 → 走完整的 IBS 休眠/唤醒流程

    // (2) ★ 硬件上电 — 操作 probe 时获取的那些引脚/regulator ★
    ret = qca_power_on(hdev);
    //   以 WCN3990 为例,实际路径:
    //     qca_regulator_init(hu)         hci_qca.c:1767
    //       ├─ serdev_device_close()        先关串口
    //       ├─ qca_regulator_enable()       ★ 开电!
    //       │   └─ pwrseq_power_on() 或 regulator_bulk_enable()
    //       │       ↑ probe 时 devm_pwrseq_get / qca_init_regulators 拿到的
    //       ├─ serdev_device_open()         重开串口
    //       ├─ gpiod_set_value_cansleep(bt_en, 0)  msleep(50)
    //       ├─ gpiod_set_value_cansleep(bt_en, 1)  msleep(50)
    //       │   ↑ probe 时 devm_gpiod_get_optional("enable") 拿到的
    //       └─ qca_port_reopen()           重开以同步 RTS/CTS

    // (3) 读芯片 ROM 版本 → 决定用哪个固件文件
    qca_read_soc_version(hdev, &soc_ver, ...);

    // (4) 设操作速度
    qca_set_baudrate(hdev, qcadev->oper_speed);
    //   用 probe 时从设备树读的 max-speed

    // (5) ★ 下载固件 — 就是用 hdev->send 发数据 ★
    qca_uart_setup(hdev, qca->soc_type, soc_ver, ...);
    //   → drivers/bluetooth/btqca.c:766
    //   → 构造文件名: "qca/crbtfw21.tlv"
    //      (用 probe 时从设备树读的 firmware-name)
    //   → qca_download_firmware(hdev, TLV_TYPE_PATCH, "qca/crbtfw21.tlv")
    //       → request_firmware(&fw, filename, &hdev->dev)
    //           ↑ 从文件系统 /lib/firmware/ 读固件
    //       → 循环: qca_tlv_send_segment()
    //           → __hci_cmd_send(hdev, ...)
    //               → hci_send_frame(hdev, skb)
    //                   → hdev->send(skb)    ★ 第 2 节注册的！
    //                       → hci_uart_send_frame()
    //                         → qca_enqueue()
    //                           (IBS_DISABLED 置位,直接入 txq)
    //                           → hci_uart_write_work()
    //                             → serdev_device_write_buf() → UART → 芯片
    //       → qca_inject_cmd_complete_event()  模拟"下载完成"事件
    //   → 同理下载 NVM config: "qca/crnv21.bin"
    //   → qca_send_reset(hdev)  发 HCI_RESET,芯片重启执行新固件

    // (6) 开 IBS — 固件完了,可以正常休眠了
    clear_bit(QCA_IBS_DISABLED, &qca->flags);
    // 从这以后,第 4 节的 qca_enqueue 走完整的 IBS 逻辑
}
```

**固件下载走的是 `hdev->send`**，和第 4 节 App 发数据走同一条路。唯一区别：下载时 `IBS_DISABLED` 置位，`qca_enqueue` 看到这标志直接入 txq，跳过休眠唤醒逻辑。

### 3.6 Power On 完整调用链

```
hci_register_dev()[2.3节末尾]
  └─ queue_work(req_workqueue, &hdev->power_on)  ← probe→power_on 的连接点
       │                                            INIT_WORK 在 hci_alloc_dev_priv 里
       ▼
     hci_power_on()                    hci_core.c:944
       └─ hci_dev_do_open()            hci_core.c:423
            ├─ mutex_lock(&hdev->req_lock)
            └─ hci_dev_open_sync()     hci_sync.c:5144
                 │
                 ├─ hdev->open(hdev)   ① ──→ hci_uart_open()  [2.3节填的指针]
                 │
                 ├─ hci_dev_init_sync() hci_sync.c:5092
                 │    │
                 │    ├─ hci_dev_setup_sync()
                 │    │    └─ hdev->setup(hdev) ② ──→ hci_uart_setup() [2.3节填的指针]
                 │    │         └─ hu->proto->setup(hu) ③ ──→ qca_setup()
                 │    │              [hci_qca.c:1909]         [2.4节注册的qca_proto]
                 │    │              ├─ qca_power_on()        ← 通电
                 │    │              └─ qca_uart_setup()      ← 下固件
                 │    │                   └─ 固件段
                 │    │                       → hci_send_frame() → hdev->send
                 │    │                         这个 hdev->send ④ 就是第 2.3 节填的
                 │    │                         跟第 4 节用户数据共用
                 │    │
                 │    └─ hci_init_sync()        ← 标准 HCI 初始化
                 │         └─ 各命令 → hci_send_cmd → cmd_work
                 │                                  → hci_send_frame → hdev->send ④
                 │
                 ├─ set HCI_UP
                 └─ mgmt_power_on(hdev)          ← 通知 bluetoothd ★
```

**关键跳转**：
- ① `hdev->open` ← 第 2 节注册, 这节第一次调
- ② `hdev->setup` ← 第 2 节注册, 这节第一次调, 这是 power_on 流程的"主菜"
- ③ `hu->proto->setup` ← 第 2 节 `qca_proto.setup = qca_setup`, 进入驱动层
- ④ `hdev->send` ← 第 2 节注册, 固件下载和标准 HCI init 内部都用到, 第 4 节 App 发数据也用它

### 3.7 Power Off 流程（逆过程）

```
用户关蓝牙 / suspend
  → hci_power_off(hdev)              hci_core.c:1013
    → hci_dev_do_close(hdev)
        mutex_lock(&hdev->req_lock)   同一把锁
        → hci_dev_close_sync(hdev)
            ├─ mgmt_power_off(hdev)   ← 通知 bluetoothd
            ├─ clear HCI_UP
            └─ hdev->close(hdev)     → hci_uart_close [2.3节注册]
                → hu->proto->close(hu) = qca_close()
                    → qca_power_off(hu)  hci_qca.c:2217
                         ├─ bt_en GPIO = 0      ← probe 时拿的引脚
                         ├─ qca_regulator_disable() ← probe 时拿的 regulator
                         └─ set QCA_BT_OFF
        mutex_unlock(&hdev->req_lock)
```

---

## 四、数据通路——App 数据怎么下去，芯片数据怎么上来

> **回顾前面**：
> - 第 2 节注册的回调函数（`hdev->send`, `hdev->open`, `qca_enqueue`, `qca_recv` 等）——这节全在调用它们
> - 第 2 节分配的队列（`txq`, `tx_wait_q`, `rx_q`）——这节的数据全在这些队列里流转
> - 第 2 节初始化的 IBS work/timer——这节的 `qca_enqueue` 和 `qca_recv` 用它们管理休眠
> - 第 3 节 power_on 完成后芯片已就绪——这节的所有操作都发生在这之后
>
> **这节干了什么**：从 App 的 write() 追到芯片的 UART TX；从芯片的 UART RX 追到 App 的 read()。中间经过每一层,标注每一次函数指针跳转。

### 4.1 先画地图——上下行全貌

```
  App (用户空间)
    ↕  socket API (sendmsg / recvmsg)
  BTPROTO 层 (BTPROTO_L2CAP / BTPROTO_HCI / ...)
    ↕  bt_proto[] 数组分派
  L2CAP 核心 / HCI sock
    ↕  hci_send_acl() / hci_acldata_packet()
  HCI 核心层 (hci_dev)
    ↕  hdev->send()  函数指针 ★ 第2节注册
  HCI UART 传输层 (hci_ldisc / hci_serdev)
    ↕  hu->proto->enqueue() / recv()  函数指针 ★ 第2节注册
  QCA 驱动层 (hci_qca)
    ↕  serdev 接口  串口回调 ★ 第2节注册
  串口硬件
    ↕  UART TX/RX 线
  蓝牙芯片
```

### 4.2 蓝牙有哪些 socket（数据的"门"）

蓝牙子系统的 socket 族是 `PF_BLUETOOTH`，下面注册了这些协议（`net/bluetooth/af_bluetooth.c` 的 `bt_init()` 里做的）：

| Protocol | Socket 类型 | 使用场景 | 实际数据传输路径 |
|----------|------------|---------|----------------|
| BTPROTO_HCI | SOCK_RAW | bluetoothd 控制芯片 | 直接写入 `hdev->cmd_q` 或 `raw_q` |
| BTPROTO_L2CAP | SOCK_SEQPACKET/STREAM/DGRAM | 几乎所有的蓝牙数据和音频 | 走 L2CAP → HCI → 驱动 |
| BTPROTO_SCO | (用户指定) | 通话语音 | 走 SCO → HCI → 驱动 |
| BTPROTO_RFCOMM | SOCK_STREAM | 串口模拟(SPP) | RFCOMM socket 内部创建了一个 L2CAP 内核 socket,最终还是走 L2CAP → HCI |
| BTPROTO_ISO | (用户指定) | LE Audio | 走 ISO → HCI → 驱动 |

**重要细节**：RFCOMM、HIDP、BNEP 这些"上层协议"有一个共同特点——它们自己创建了一个 L2CAP 内核 socket（`sock_create_kern(BTPROTO_L2CAP, ...)`），最终数据都是从 L2CAP 通道走的。所以下面以 **L2CAP 数据**为例跟踪全路径。

### 4.3 下行（TX）：App write() → 芯片收到

拿最常见的 L2CAP 数据发送为例：

**第一段——从 App 到 HCI 队列（同步,在 App 的进程上下文里执行）**：

```
App: write(sock_fd, buf, len)
  → VFS → sock_sendmsg()

    → l2cap_sock_sendmsg()           l2cap_sock.c:1129
        │                              ↑ 通过 bt_proto[BTPROTO_L2CAP] 分派到这
        ├─ sock → chan (L2CAP 通道)
        │   连接: chan 在蓝牙建连时创建,socket 指针存在 chan->data 里
        │   第 4.4 节上行时就是通过 chan->data 找到 socket 的
        │
        └─ l2cap_chan_send(chan, msg, len)
             l2cap_core.c:2559
             ├─ 加 L2CAP header (CID + 长度)
             └─ l2cap_do_send(chan, skb)
                  l2cap_core.c:999
                    └─ hci_send_acl(chan->conn->hchan, skb, flags)
                         hci_core.c:3275
                         │
                         ├─ hci_queue_acl(chan, &chan->data_q, skb)
                         │   ├─ 设 pkt_type = HCI_ACLDATA_PKT (0x02)
                         │   ├─ 加 HCI ACL header (handle | pb | bc | len)
                         │   └─ skb_queue_tail(&chan->data_q, skb)
                         │        ↑ 数据暂存在 HCI 通道的 data_q 里
                         │
                         └─ queue_work(hdev->workqueue, &hdev->tx_work)
                              ↑ 触发异步发送,然后 App 的 write() 就返回了
```

**为什么这里要切到异步 work？** App 的 write() 不应该等数据真的通过串口发送完（串口慢）。write() 只负责把数据组装好、丢进队列，然后立刻返回给 App。

**第二段——异步 work：`hci_tx_work` 调度发送**：

```
hci_tx_work()                        hci_core.c:3806
  这个 work 在 hdev->workqueue 上跑 (第 2.3 节创建的)
  │
  ├─ hci_sched_sco(hdev, SCO_LINK)   SCO 语音优先
  ├─ hci_sched_sco(hdev, ESCO_LINK)
  ├─ hci_sched_acl(hdev)             ACL 数据
  │   hci_core.c:3719
  │   └─ hci_sched_acl_pkt(hdev)
  │       ├─ 检查流控 (acl_cnt > 0? 芯片的缓冲区还有空吗?)
  │       ├─ dequeue from chan->data_q   ← 从之前入队的队列里取
  │       └─ hci_send_conn_frame(hdev, conn, skb)
  │           └─ hci_send_frame(hdev, skb)
  │               hci_core.c:3039  ← 从这里开始是所有发送的统一入口
  │               ├─ hci_send_to_monitor(skb) ← 拷贝给 hcidump 等调试工具
  │               └─ ★ hdev->send(hdev, skb)  ★ 第 2 节注册的函数指针！
  │
  └─ hci_sched_le(hdev)             LE 数据
```

**第三段——进入传输层和驱动层**：

```
hdev->send(hdev, skb)                ← 第2节: hdev->send = hci_uart_send_frame
  = hci_uart_send_frame()            hci_ldisc.c:274
      │
      ├─ hu->proto->enqueue(hu, skb) ← 第2节: hu->proto->enqueue = qca_enqueue
      │   = qca_enqueue()            hci_qca.c:897
      │       │
      │       ├─ skb_push(skb, 1); *skb->data = 0x02;
      │       │    ↑ 在数据前面加 H4 packet type 字节
      │       │      ACL data = 0x02, EVENT = 0x04, SCO = 0x08
      │       │
      │       ├─ if (QCA_IBS_DISABLED)         ← 第 3 节固件下载时置的位
      │       │     → skb_queue_tail(&qca->txq, skb)
      │       │       ↑ 直接入队发送, 不管芯片睡没睡
      │       │
      │       └─ switch (qca->tx_ibs_state):   ← IBS 状态机 (第2节qca_open初始化的)
      │           case AWAKE: 芯片醒着
      │             → skb_queue_tail(&qca->txq, skb) 直接入队
      │             → mod_timer(&qca->tx_idle_timer, 2000ms)
      │                ↑ 2秒没新数据就发 SLEEP_IND 让芯片休眠
      │           case ASLEEP: 芯片睡着
      │             → skb_queue_tail(&qca->tx_wait_q, skb) 暂存等待队列
      │             → tx_ibs_state = WAKING
      │             → queue_work(qca->qca_wq, &qca->ws_awake_device)
      │                ↑ 触发唤醒芯片流程 (见下)
      │           case WAKING: 正在唤醒
      │             → skb_queue_tail(&qca->tx_wait_q, skb) 继续等
      │
      └─ hci_uart_tx_wakeup(hu)
           → schedule_work(&hu->write_work)    触发真正的串口写
```

**IBS 唤醒流程——芯片睡着要先叫醒**：

```
qca_wq_awake_device()                hci_qca.c:389
  这个 work 在 qca->qca_wq 上跑 (第2节 qca_open 创建的)
  ├─ send_hci_ibs_cmd(HCI_IBS_WAKE_IND)
  │   构造 1 字节 skb: [0xFE] = HCI_IBS_WAKE_IND
  │   → 入 txq → write_work 发到串口 → 芯片收到
  └─ mod_timer(&qca->wake_retrans_timer, 100ms)
       ↑ 如果 100ms 内没收到 WAKE_ACK,重传
         重试 3 次都失败 → report hardware error
```

芯片回复 WAKE_ACK 后（通过上行路径进来,见 4.4）：

```
qca_recv() → h4_recv_buf() → 检测到 0xFE = IBS_WAKE_ACK
  → device_woke_up()       hci_qca.c
      ├─ cancel wake_retrans_timer    芯片回了,不重传了
      ├─ tx_ibs_state = AWAKE        标记已醒
      ├─ 把 tx_wait_q 全移到 txq     ← ★ 之前暂存的数据现在可以发了
      │   while ((skb = skb_dequeue(&qca->tx_wait_q)))
      │       skb_queue_tail(&qca->txq, skb);
      └─ hci_uart_tx_wakeup(hu)      触发发送
```

**第四段——真正写串口**：

```
hci_uart_write_work()                hci_serdev.c:57
  这个 work 在 system_wq 上跑
  │
  └─ while (1) {
        skb = hci_uart_dequeue(hu)   → qca_dequeue()
               → skb_dequeue(&qca->txq)  从 txq 取待发数据

        len = serdev_device_write_buf(serdev, skb->data, skb->len);
              ↑ ★ 最终物理写入——调用串口驱动把字节发到 UART TX 线

        if (len < skb->len) break;   串口缓冲区满了,剩余等 write_wakeup 再发
        kfree_skb(skb);
     }
```

### 4.4 上行（RX）：芯片发数据 → App read() 读到

**第一段——中断上下文,只做最少的事**：

```
[硬件中断: 蓝牙芯片通过 UART TX 线发数据给 CPU]
  串口驱动收到 → 触发 serdev 回调 ★ 第 2.3 节 (a) 注册的
    = hci_uart_receive_buf()          hci_serdev.c:274
        │
        └─ hu->proto->recv(hu, data, count)  ← 第2节: qca_proto.recv = qca_recv
            = qca_recv()              hci_qca.c:1278
                │
                └─ h4_recv_buf(hdev, qca_recv_pkts, data, count)
                     │
                     ├─ 第一个字节判断类型:
                     │   0x02 → ACL 数据  → qca_recv_acl_data()
                     │   0x04 → HCI 事件  → qca_recv_event()
                     │   0x08 → SCO 数据
                     │   0x0A → ISO 数据
                     │   0xFE → IBS 协议包 → 内部消化,不往上发
                     │
                     └─ 组包完成后:
                          ★ hci_recv_frame(hdev, skb) ★
                          这是驱动层主动调用核心层！
                          hci_core.c:2918
                            ├─ 验证 pkt_type
                            ├─ bt_cb(skb)->incoming = 1
                            ├─ skb_queue_tail(&hdev->rx_q, skb)
                            └─ queue_work(hdev->workqueue, &hdev->rx_work)
                                 ↑ 跟 tx_work 共用一个 workqueue (第2节创建的)
```

**这里有个重要区别**：
- 驱动 → 核心层的连接**不是函数指针**,而是**直接函数调用** `hci_recv_frame()`
- 因为驱动层代码可以直接 `#include` 核心层的头文件,直接调用
- TX 方向用函数指针是因为核心层不能依赖具体的驱动

**第二段——进程上下文,rx_work 分派和处理**：

```
hci_rx_work()                        hci_core.c:4027
  这个 work 在 hdev->workqueue 上跑
  │
  └─ while ((skb = skb_dequeue(&hdev->rx_q))) {
        │
        ├─ hci_send_to_monitor(skb)    拷贝给监控 socket
        │
        ├─ switch (hci_skb_pkt_type(skb)) {
        │
        │   case HCI_EVENT_PKT:
        │     → hci_event_packet(hdev, skb)
        │         ├─ command_complete → 匹配 hdev->sent_cmd
        │         │   唤醒 wait_queue → 发该命令的调用者拿到结果
        │         ├─ connection_request → 通知 mgmt 层
        │         └─ ...
        │
        │   case HCI_ACLDATA_PKT:      ← ★ 最主要的数据通道
        │     → hci_acldata_packet(hdev, skb)
        │          hci_core.c:3833
        │          └─ l2cap_recv_acldata(hdev, handle, skb, flags)
        │               l2cap_core.c:7629
        │               │
        │               ├─ hcon = hci_conn_hash_lookup_handle(hdev, handle)
        │               │   根据 ACL handle 找到 HCI 连接
        │               │
        │               ├─ conn = hcon->l2cap_data
        │               │   ↑ HCI 连接 → L2CAP 连接的桥梁
        │               │    struct l2cap_conn 嵌在 HCI 连接的 hcon->l2cap_data 里
        │               │
        │               ├─ 处理 ACL 分片重组(如果需要)
        │               │
        │               └─ l2cap_recv_frame(conn, skb)
        │                    l2cap_core.c:6920
        │                      ├─ 读 L2CAP header (CID + 长度)
        │                      └─ 按 CID 分派:
        │                          CID=0x0001 → l2cap_sig_channel()  信令
        │                          CID=0x0002 → l2cap_conless_channel()
        │                          CID=0x0005 → l2cap_le_sig_channel()
        │                          其他 CID  → l2cap_data_channel()  ← 数据
        │                                      l2cap_core.c:6813
        │                                        ├─ chan = l2cap_get_chan_by_scid(conn, cid)
        │                                        │   根据 CID 找 L2CAP 通道
        │                                        │
        │                                        └─ ★ chan->ops->recv(chan, skb) ★
        │                                             = l2cap_sock_recv_cb()
        │                                             l2cap_sock.c:1528
        │                                               │
        │                                               ├─ sk = chan->data ★★★
        │                                               │   ↑ 通道里存着 socket 指针！
        │                                               │   建连时存进去的
        │                                               │
        │                                               └─ __sock_queue_rcv_skb(sk, skb)
        │                                                   数据挂到 sk->sk_receive_queue
        │                                                   → sk->sk_data_ready(sk)
        │                                                      ↑ 唤醒在 poll/epoll 等的 App
        │
        │   case HCI_SCODATA_PKT:
        │     → sco_recv_scodata() → sco_recv_frame()
        │         → sock_queue_rcv_skb(conn->sk, skb)
        │
        │   case HCI_ISODATA_PKT:
        │     → hci_isodata_packet() → iso_recv()
        │   }
     }
```

**App 被唤醒**：
```
App: 正在 poll(sock_fd, ...) / read(sock_fd, ...)
  → sk->sk_data_ready 触发 → poll_wait 唤醒
  → read(sock_fd, buf, len)
    → l2cap_sock_recvmsg()
      → skb_recv_datagram(sk, ...)  从 sk->sk_receive_queue 取数据
      → copy_to_user(buf, skb->data, len)
  → App 拿到数据
```

### 4.5 六个桥接点汇总

| # | 位置 (代码) | 从 | 到 | 桥接方式 |
|---|-----------|----|----|---------|
| ① | `hci_send_frame()` → `hdev->send(skb)` | HCI 核心层 | 传输层 | 函数指针 `hdev->send` (第2节注册) |
| ② | `hci_uart_send_frame()` → `hu->proto->enqueue()` | 传输层 | QCA 驱动层 | 函数指针 `hu->proto->enqueue` (第2节 `qca_proto` 注册) |
| ③ | `serdev receive_buf` → `hci_uart_receive_buf()` | 串口硬件 | 传输层 | `serdev_device_ops` (第2.3(a)设置) |
| ④ | `hci_uart_receive_buf()` → `hu->proto->recv()` | 传输层 | QCA 驱动层 | 函数指针 `hu->proto->recv` (第2节 `qca_proto` 注册) |
| ⑤ | `qca_recv()` → `hci_recv_frame()` | QCA 驱动层 | HCI 核心层 | 直接函数调用（不是指针,因为驱动可见核心层） |
| ⑥ | `l2cap_data_channel()` → `l2cap_sock_recv_cb()` | L2CAP 核心 | socket | `chan->ops->recv` 函数指针 + `chan->data = sk` |
| ⑦ | `hci_register_dev()` → `queue_work(power_on)` | probe | power_on | work 排队 (第2→3节的连接) |

### 4.6 TX + RX 全景融合图

```
============ 下行 (TX): App write() → 芯片 ============

App: write(fd, buf, len)
  → l2cap_sock_sendmsg()                    [BTPROTO_L2CAP socket]
     → l2cap_chan_send()                    [L2CAP: 加 L2CAP header]
       → l2cap_do_send()
         → hci_send_acl()                   [HCI: 加 ACL header]
           → hci_queue_acl()                [入队 chan->data_q]
           → queue_work(tx_work)            [排队 → App write() 返回]
                ↓ 异步执行
           hci_tx_work()                    [HCI: 按流控调度]
             → hci_sched_acl()
               → hci_send_frame()           [统一发送入口]
                 → hdev->send(skb)      ① [函数指针 → 传输层]
                   → hci_uart_send_frame()
                     → qca_enqueue()    ② [函数指针 → 驱动层]
                       (加 H4 type byte)
                       (IBS: AWAKE→入txq, ASLEEP→入tx_wait_q+唤醒)
                       → hci_uart_write_work() [异步写]
                         → serdev_device_write_buf()
                           → UART TX → 芯片 RX


============ 上行 (RX): 芯片 → App read() ============

芯片 TX → UART RX (硬件中断)
  → hci_uart_receive_buf()              ③ [serdev 回调 → 传输层]
    → qca_recv()                        ④ [函数指针 → 驱动层]
      → h4_recv_buf()                      [H4 解帧,去 IBS 包]
      → hci_recv_frame(hdev, skb)      ⑤ [直接调用,回到核心层]
        → skb_queue_tail(&hdev->rx_q)      [入队 rx_q]
        → queue_work(rx_work)              [排队 → 中断返回]
             ↓ 异步执行
        hci_rx_work()                      [HCI: 从 rx_q 取数据]
          → hci_acldata_packet()           [去 ACL header]
            → l2cap_recv_acldata()         [HCI连接 → L2CAP连接]
              → l2cap_recv_frame()         [按 CID 分派]
                → l2cap_data_channel()     [找 L2CAP 通道]
                  → l2cap_sock_recv_cb()⑥ [chan->ops->recv + chan->data = sk]
                    → __sock_queue_rcv_skb()
                      → sk->sk_data_ready()
                        → App 被唤醒, read() 拿到数据
```

### 4.7 中断 vs 进程上下文——为什么收要分两段

```
中断上下文（qca_recv → hci_recv_frame）:

  只能做: 组包, 入队 rx_q, 触发 rx_work
  不能做: 查连接表, 查 L2CAP 通道, 通知 socket, 拷贝到用户空间

  原因: 中断里不能做耗时操作,不能拿某些锁,不能调可能休眠的函数。

进程上下文（hci_rx_work）:

  可以做: 所有的事——解析协议, 路由数据, 通知 socket, 唤醒 App

  原因: workqueue handler 跑在普通进程上下文, 可以休眠, 可以拿锁。
```

---

## 附录：关键数据结构速查

### 核心结构体的关系

```
struct hci_dev {                     ← 核心层看到的蓝牙设备
    struct workqueue_struct *workqueue;     ← rx_work/cmd_work/tx_work 在这跑
    struct workqueue_struct *req_workqueue; ← power_on/power_off 在这跑

    struct sk_buff_head rx_q;        ← 上行数据暂存
    struct sk_buff_head cmd_q;       ← 待发 HCI 命令暂存
    struct sk_buff_head raw_q;       ← 待发 raw 数据暂存

    int (*open)(struct hci_dev *hdev);     ← → hci_uart_open
    int (*close)(struct hci_dev *hdev);    ← → hci_uart_close
    int (*send)(struct hci_dev *hdev, struct sk_buff *skb); ← → hci_uart_send_frame
    int (*setup)(struct hci_dev *hdev);    ← → hci_uart_setup

    struct work_struct power_on;     ← handler = hci_power_on
    struct delayed_work power_off;   ← handler = hci_power_off

    struct mutex req_lock;           ← power on/off/sync 共用锁
};

struct hci_uart {                    ← 传输层的 per-device 数据
    struct serdev_device *serdev;    ← 指向串口设备
    struct hci_uart_proto *proto;    ← 指向 qca_proto

    struct work_struct write_work;   ← 实际写串口的 work

    unsigned int init_speed;         ← 初始波特率 (qca_open 设)
    unsigned int oper_speed;         ← 工作波特率 (qca_open 设)
};

struct qca_data {                    ← QCA 驱动层的 per-device 数据
    struct sk_buff_head txq;         ← 待发队列
    struct sk_buff_head tx_wait_q;   ← 等芯片醒来的等待队列

    struct workqueue_struct *qca_wq; ← IBS 相关 work 在这跑
    struct work_struct ws_awake_device;  ← 叫芯片醒
    struct work_struct ws_awake_rx;      ← 芯片叫主机醒的回应

    struct timer_list wake_retrans_timer; ← WAKE_IND 超时重传 (100ms)
    struct timer_list tx_idle_timer;      ← TX 空闲超时 → SLEEP_IND (2000ms)

    u8 tx_ibs_state;                 ← AWAKE / ASLEEP / WAKING
    u8 rx_ibs_state;
};

struct qca_serdev {                  ← QCA serdev 平台的 per-device 数据
    struct hci_uart serdev_hu;       ← 内嵌传输层结构
    enum qca_soc_type soc_type;      ← WCN3990 / QCA6390 / ROME ...
    struct qca_power *bt_power;      ← 电源管理
    struct gpio_desc *bt_en;         ← BT_EN 引脚 (probe 时从 DT 获取)
    struct clk *susclk;              ← 休眠时钟
    const char *firmware_name;       ← 固件文件名 (probe 时从 DT 获取)
    u32 init_speed;                  ← 初始波特率
    u32 oper_speed;                  ← 操作波特率
};
```

### 关键文件及对应关系

| 文件 | 核心数据结构 | 做什么 | 与谁连接 |
|------|------------|--------|---------|
| `drivers/bluetooth/hci_qca.c` | `qca_data`, `qca_serdev` | probe, open, setup, enqueue, recv | 通过 `qca_proto` 注册到传输层 |
| `drivers/bluetooth/btqca.c` | (无私有,hci_dev) | 固件下载 TLV 处理 | 被 qca_setup 调用,内部走 hdev->send |
| `drivers/bluetooth/hci_serdev.c` | `hci_uart` | 注册,写串口,收串口回调 | 桥接 serdev 和 hci_uart |
| `drivers/bluetooth/hci_ldisc.c` | `hci_uart` | open/close/send/setup/dequeue | 桥接 hci_dev 函数指针和 hci_uart_proto |
| `net/bluetooth/hci_core.c` | `hci_dev` | register_dev, recv_frame, rx_work, tx_work, send_frame, power_on | 核心调度中心 |
| `net/bluetooth/hci_sync.c` | `hci_dev` | dev_open_sync, dev_init_sync, dev_setup_sync, init_sync | power on 主流程 |
| `net/bluetooth/hci_event.c` | `hci_dev` | 处理芯片上报的事件 | 被 rx_work 调用 |
| `net/bluetooth/l2cap_core.c` | `l2cap_conn`, `l2cap_chan` | 数据收发, 建连拆连 | 通过 hci_send_acl / l2cap_recv_acldata 连接 HCI 层 |
| `net/bluetooth/l2cap_sock.c` | `sock` | socket sendmsg/recvmsg | 通过 bt_proto[] 注册, chan->data = sk |
| `net/bluetooth/af_bluetooth.c` | `bt_proto[]` | PF_BLUETOOTH 注册, 协议分派 | 连接 socket 系统和蓝牙协议 |
| `net/bluetooth/mgmt.c` | `hci_dev` | bluetoothd 管理接口 | 通过 HCI socket 或 cmd_sync 与核心层交互 |

### 一句话记各层连接方式

- **socket → L2CAP**: `bt_proto[]` 数组按 protocol 分派; `chan->data = sk` 反向定位
- **L2CAP → HCI core**: `hci_send_acl()` 下行; `l2cap_recv_acldata()` 上行; `chan->conn->hchan` 桥接 HCI 连接
- **HCI core → 传输层**: `hdev->open/close/send/setup` 函数指针 (第 2 节注册)
- **传输层 → 驱动层**: `hu->proto->open/close/enqueue/recv/setup` 函数指针 (第 2 节 `qca_proto` 注册)
- **驱动 → 硬件**: `serdev_device_write_buf()` 写; 中断 + `serdev receive_buf` 回调 读
- **probe → power_on**: `queue_work(hdev->req_workqueue, &hdev->power_on)` (第 2 → 3 节)
- **power_on → 数据通路**: 同一条 `hdev->send()` (第 3 节固件下载, 第 4 节 App 数据, 走同一个函数)

---

## 附录二：实用信息速查

### 如何找蓝牙子系统的 maintainer

```bash
# 方法1：查 MAINTAINERS 文件
grep -A10 "BLUETOOTH SUBSYSTEM" MAINTAINERS
# 输出：
# BLUETOOTH SUBSYSTEM
# M:  Marcel Holtmann <marcel@holtmann.org>
# M:  Johan Hedberg <johan.hedberg@gmail.com>
# M:  Luiz Augusto von Dentz <luiz.dentz@gmail.com>
# L:  linux-bluetooth@vger.kernel.org

# 方法2：用 get_maintainer.pl 查你改的文件
./scripts/get_maintainer.pl -f drivers/bluetooth/hci_qca.c
./scripts/get_maintainer.pl -f net/bluetooth/hci_core.c
```

### 内核版本演进（2025年值得注意的变更）

| 变更 | 说明 |
|------|------|
| **HCI_DRV_PKT** (2025.04) | 新增 `HCI_DRV_PKT` 包类型，用户态程序可以直接调用驱动特定回调（如 btusb 的 USB alternate setting 切换），用于 SCO 音频 |
| **RCU-safe RX path** (2025.11) | `l2cap_recv_acldata()` 签名从 `(struct hci_conn *hcon, ...)` 改为 `(struct hci_dev *hdev, u16 handle, ...)`，修复了 use-after-free。conn 查找和使用在同一 RCU 临界区 |
| **hci_drv 结构体** (2025.04) | `struct hci_dev` 新增 `hci_drv` 成员，指向 HCI Driver Protocol 实现 |

### 官方资源

| 资源 | 位置 |
|------|------|
| 内核蓝牙源码 | `net/bluetooth/`, `drivers/bluetooth/`, `include/net/bluetooth/` |
| BlueZ 官网 | http://www.bluez.org/ |
| BlueZ 源码 | git://git.kernel.org/pub/scm/bluetooth/bluez.git |
| 蓝牙子系统维护树 | git://git.kernel.org/pub/scm/linux/kernel/git/bluetooth/bluetooth-next.git |
| 邮件列表 | linux-bluetooth@vger.kernel.org |
| lore 归档 | https://lore.kernel.org/linux-bluetooth/ |
| 内核提交规则 | `Documentation/process/submitting-patches.rst` |
| 编码风格 | `Documentation/process/coding-style.rst` |
| 蓝牙 maintainer | `MAINTAINERS` 文件 `BLUETOOTH SUBSYSTEM` 条目 |
