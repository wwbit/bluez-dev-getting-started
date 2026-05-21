# Yocto / QCLinux 构建体系详解（小白友好版）

---

## 一、Yocto 在做什么？—— 用装电脑来理解

你想从零装一台电脑并装好系统，大步骤是：

| 步骤 | 装电脑 | Yocto 对应的是什么 |
|------|--------|-------------------|
| 1. 买零件 | 下单 CPU、内存、硬盘、显卡 | **fetch（下载）** — 从网上下载各种软件的源代码 |
| 2. 拆包装 | 拆盒子、零件分类放好 | **unpack（解压）** — 把下载的压缩包解压到工作目录 |
| 3. 组装 | 把零件插到主板上 | **compile（编译）** — 把源代码变成可运行的程序（二进制） |
| 4. 接线 | 接好电源线、数据线 | **install（安装）** — 把编译产物放到正确的位置（如 `/usr/bin`） |
| 5. 装系统 | 装 Linux、驱动、应用软件 | **rootfs（根文件系统）** — 把所有程序汇到一个目录，打包成镜像文件 |

**Yocto 的终极产物就是一个"刷机包"**（比如 `.ext4` 或 `.qcomflash` 文件），烧录到开发板上通电就能跑完整的 Linux 系统。

**但是**，以上每个步骤都不是 Yocto 手动做的——它有一个"总指挥"叫 BitBake，有一个"基础零件库"叫 oe-core，有一堆"专业供应商"叫 layer，还有一个"一键部署脚本"叫 KAS。下面逐个说清楚它们是什么，以及它们之间**怎么串起来的**。

---

## 二、核心概念一：Recipe（配方）—— "一道菜的菜谱"

### 2.1 Recipe 是什么

recipe（`.bb` 文件）告诉 Yocto 四个关键信息，这四个信息直接对应了**第一节的流水线**：

```bitbake
# hello-world.bb — 描述怎么构建"hello-world"这个程序

SUMMARY = "一个简单的 hello world"                        # 简介
LICENSE = "MIT"                                          # 许可证
SRC_URI = "https://example.com/hello-world.tar.gz"      # 1️⃣ 去哪下载 → 对应 fetch

do_compile() {
    make                                                  # 2️⃣ 怎么编译 → 对应 compile
}

do_install() {
    install -d ${D}/usr/bin                              # 3️⃣ 装到哪 → 对应 install
    install -m 0755 hello ${D}/usr/bin/
}
```

**Yocto 还自动多做了一件事：打包。** `do_compile` 编译出二进制之后，`do_install` 把它装到临时目录，然后 `do_package` 会把它拆成 `.rpm`（或 `.deb`/`.ipk`）包。这些包稍后会全部汇入最终的 rootfs 镜像。

### 2.2 Recipe 的四种形态

| 类型 | 文件名 | 类比 | 做什么 |
|------|--------|------|--------|
| 普通 recipe | `.bb` | 一道菜的菜谱 | 构建**一个**软件（openssh、python、Linux 内核） |
| packagegroup | `.bb` | 一份购物清单 | 自己不产出文件，只是列了一堆包名，用来"一站式引用" |
| image | `.bb` | 一整桌宴席的菜单 | 声明最终镜像里包含哪些菜（包），产出刷机包 |
| bbappend | `.bbappend` | 在别人菜谱上加批注 | 不改原 recipe 文件，而是追加/覆盖其中的内容 |

### 2.3 Recipe 之间怎么建立联系？

Recipe 之间通过两个关键变量建立**依赖关系**：

```bitbake
DEPENDS = "libfoo libbar"           # "编译我之前，先把 libfoo 和 libbar 编译好"
RDEPENDS:${PN} = "libfoo"          # "运行我的时候，系统里必须有 libfoo"
```

**这就是依赖链的起点。** 后面第四节会展示，BitBake 如何从 image recipe 出发，沿着 `DEPENDS` + `RDEPENDS` 一层层往下挖，挖出几百个包的完整依赖图。

**本项目示例：**

```bitbake
# packagegroup-qcom-benchmark.bb — 购物清单
RDEPENDS:${PN} = "coremark dhrystone glmark2"
# 意思是：装了这个 packagegroup，就等于装了以上三个测试工具

# qcom-multimedia-image.bb — 宴席菜单
CORE_IMAGE_BASE_INSTALL += "gstreamer1.0 pipewire weston libcamera ..."
# 意思是：这个镜像里必须包含 gstreamer、pipewire、weston 这些菜
```

> **节间关联**：Recipe 里写的 `SRC_URI` 决定了 fetch 去哪下载 → `do_compile` 怎么编译 → `do_install` 装到哪 → 而 `DEPENDS`/`RDEPENDS` 把成千上万个 recipe 串成一张网 → 这张网最终被 **第四节** 的 BitBake DAG 所调度。

---

## 三、核心概念二：Layer（层）—— "分类存放 recipe 的文件夹供应商"

### 3.1 Layer 是什么

Layer 就是把一堆 recipe、配置文件、class 文件**按功能分类**放在一起。你可以理解为：

> **Layer = 一个供应商。每个供应商提供某一类零件（recipe）。**

每个 layer 必须有一个 `conf/layer.conf`，相当于供应商的"营业执照"。它告诉 Yocto：

```bitbake
# meta-qcom-distro/conf/layer.conf
BBFILE_COLLECTIONS += "qcom-distro"                         # 我叫什么
BBFILE_PATTERN_qcom-distro := "^${LAYERDIR}/"               # 我的领土范围
BBFILE_PRIORITY_qcom-distro = "10"                          # 我有最高优先级
LAYERDEPENDS_qcom-distro = "core qcom openembedded-layer ..."  # 我依赖这些其他 layer
```

### 3.2 Layer 之间怎么依赖？（LAYERDEPENDS 链）

**这是 layer 之间的第一条联系。** 一个 layer 不能独立存在，它建立在其他 layer 的基础上：

```
meta-qcom-distro  ← 最上层：产品镜像 + 发行策略
  │  LAYERDEPENDS = "core, qcom, openembedded-layer, networking-layer,
  │                   multimedia-layer, security, selinux, sota, tpm-layer,
  │                   virtualization-layer, xfce-layer, meta-python"
  │
  ├── meta-qcom  ← BSP 层：内核 + 驱动 + firmware
  │     LAYERDEPENDS = "core"
  │
  ├── meta-openembedded/*  ← 社区生态层（Python/网络/桌面/文件系统...）
  │     这些子 layer 也各自依赖 core
  │
  └── oe-core/meta  ← 最底层：gcc、glibc、busybox、systemd 等基础
        （没有 LAYERDEPENDS，它是根）
```

**为什么需要声明 LAYERDEPENDS？** 比如 `meta-qcom-distro` 的 image recipe 里写了 `weston`，但 `weston` 的 recipe 不在 `meta-qcom-distro` 里，它在 `oe-core/meta` 里。如果构建时没有把 `oe-core/meta` 加到 `BBLAYERS`，BitBake 就找不到 weston 的 recipe，直接报错。`LAYERDEPENDS` 确保所有"供应商"都到位了。

### 3.3 两个 Layer 有同名 recipe 怎么办？（BBFILE_PRIORITY 机制）

**这是 layer 之间的第二条联系——优先级覆盖。**

如果 `meta-qcom`（优先级 6）和 `meta-qcom-distro`（优先级 10）都有一个 `linux-qcom_6.18.bb`：

```
BBFILE_PRIORITY_qcom-distro = 10  ← 更高，胜出
BBFILE_PRIORITY_qcom = 6          ← 更低，被覆盖
```

**数字越大越优先。** 所以 distro 层的版本生效。这很常用：BSP 层提供默认版本，distro 层用 `.bbappend` 或同名 recipe 覆盖特定配置。

### 3.4 Layer 和 Recipe 是什么关系？

```
Layer（文件夹）
  └── recipes-*/*/  （按功能分子目录）
        ├── foo_1.0.bb      ← 普通 recipe
        ├── foo_1.0.bbappend ← 对上层 layer 的 foo 做定制
        └── images/
              └── my-image.bb ← image recipe，产出刷机包
```

**所以 layer 是容器，recipe 是内容。** 每个 layer 通过 `conf/layer.conf` 里的 `BBFILES` 声明"我的哪些文件是 recipe"。

> **节间关联**：Layer 之间通过 `LAYERDEPENDS` 串成依赖链 → 这个依赖链决定了**第五节** bblayers.conf 的内容 → 而 KAS（下一节）就是自动生成 bblayers.conf 和 local.conf 的工具。

---

## 四、核心概念三：KAS —— "一键部署所有 layer 并生成配置"

### 4.1 没有 KAS 的话，你需要手动做什么？

传统 Yocto 搭建需要手动完成：
1. `git clone` 十几个仓库（oe-core、bitbake、meta-qcom、meta-openembedded...）
2. 手写 `build/conf/bblayers.conf`（列出所有 layer 的路径）
3. 手写 `build/conf/local.conf`（设置 MACHINE、DISTRO、内核版本、镜像格式...）
4. `source oe-init-build-env build` 初始化环境
5. 最后 `bitbake <target>`

**KAS 把以上 5 步压缩成一条命令。**

### 4.2 KAS YAML 文件长什么样？

```yaml
# 示例文件：展示了 KAS 的五个核心能力

header:
  version: 14
  includes:                          # ① 积木式组合：引用其他 YAML
    - ci/base.yml

machine: iq-9075-evk                 # ② 选择硬件平台

distro: qcom-distro                  # ③ 选择发行版策略

repos:                               # ④ 声明要 clone 哪些仓库（即哪些 layer）
  meta-qcom:
    url: https://github.com/qualcomm-linux/meta-qcom
    branch: master
    layers:                          # 这个仓库里的哪些子目录是 layer
      .:                             # 根目录本身就是 layer

local_conf_header:                   # ⑤ 注入到 local.conf 的配置
  myconfig: |
    PREFERRED_VERSION_foo = "1.0"

target:                              # ⑥ 构建目标
  - qcom-multimedia-image
```

### 4.3 KAS 文件的积木搭法：include 链

**这是 KAS 最重要的机制：多个 YAML 文件通过 `includes` 层层嵌套，像乐高一样拼在一起。**

本项目中，一条典型命令：

```bash
kas build ci/iq-9075-evk.yml:ci/qcom-distro.yml:ci/linux-qcom-rt-6.18.yml
```

冒号分隔 = **从左到右优先级递增（右边的覆盖左边的）**。

**但实际展开的文件远超 3 个**，因为每个文件内部还有 `includes`。真实展开链路：

```
命令行：iq-9075-evk.yml : qcom-distro.yml : linux-qcom-rt-6.18.yml
          │                  │                   │
          │ includes         │ includes          │ (叶子文件，没有 includes)
          ▼                  ▼                   │
    iq-9075-evk.yml     qcom-distro.yml          │
    (meta-qcom-distro)  (meta-qcom-distro)       │
          │                  │                   │
          │ includes         │ includes          │
          ▼                  ▼                   │
      ci/base.yml        qcom-distro.yml        │
    (meta-qcom-distro)  (meta-qcom)             │
          │                                      │
          │ includes                             │
          ▼                                      │
      meta-qcom/ci/base.yml ←── 最终叶子层      │
          │                                      │
          │ 提供：oe-core、bitbake、基础配置       │
          │                                      │
          └──────────────────────────────────────┘
```

**展开后总共有 ~10 个 YAML 文件参与合并。** 最终合并成一份 `local.conf`（包含所有 `local_conf_header` 拼在一起）+ 一份 repo clone 清单 + 一个 `BBLAYERS` 列表。

### 4.4 KAS 如何连接 Layer、Recipe、以及最终的构建？

```
KAS YAML 文件
  │
  ├── repos: 告诉 KAS clone 哪些 git 仓库
  │     → 这些仓库就是 Layer 的物理存储位置
  │     → 仓库 clone 完之后，layers: 指定哪些子目录加入 BBLAYERS
  │
  ├── machine: / distro: / local_conf_header:
  │     → 这些生成 local.conf 中的变量
  │     → 这些变量会在 BitBake 解析 recipe 时被读取
  │
  └── target:
        → 传给 bitbake 的命令行参数
        → 告诉 BitBake "给我构建这些 image"
```

> **节间关联总结**：KAS 文件声明"用哪些 Layer"（第三节）→ KAS clone 代码并生成 bblayers.conf → BitBake（下一节）读取所有 layer 里的 Recipe（第二节）→ 解析 recipe 之间的 DEPENDS/RDEPENDS → 构建 DAG → 执行 fetch/compile/install/rootfs。

---

## 五、BitBake 调度引擎 —— "施工队总指挥"

### 5.1 BitBake 是什么

BitBake 是一个**纯调度器**——它自己不编译任何代码，而是：
1. 解析所有 layer 里的所有 recipe
2. 根据 recipe 之间的依赖关系画出 DAG（有向无环图）
3. 按拓扑顺序调度每一个 task
4. 决定哪些 task 可以**并行执行**，哪些必须**串行等待**

```
oe-core      ← 提供 400+ bbclass（task 模板），定义了 cmake/autotools/meson 等编译流程
bitbake      ← 读取 recipe → 套用 bbclass 模板 → 调度 task 执行
meta-qcom    ← 提供内核、GPU、固件等具体 recipe
meta-qcom-distro ← 提供发行策略和产品 image recipe
```

### 5.2 它怎么工作——以 `qcom-multimedia-image` 为例

你敲下 `bitbake qcom-multimedia-image` 之后发生了什么：

```
① 解析阶段（Parsing）
   ├── 读取 BBLAYERS 中所有 layer 的 recipe 文件
   ├── 把每个 recipe 展开（require、include、inherit 全部内联）
   └── 构建全局 recipe 名字空间

② DAG 构建
   ├── 从 qcom-multimedia-image.bb 出发
   │     它 require了 qcom-console-image.bb
   │     它 require了 qcom-minimal-image.bb
   │     它 inherit了 core-image（oe-core 提供的 bbclass）
   │     它写了 CORE_IMAGE_BASE_INSTALL += "weston gstreamer pipewire ..."
   │
   ├── 针对 CORE_IMAGE_BASE_INSTALL 里每个名字，找到对应的 recipe
   │     比如 "weston" → oe-core/meta/recipes-graphics/wayland/weston_*.bb
   │
   ├── 每个 recipe 又有自己的 DEPENDS 和 RDEPENDS
   │     比如 weston → DEPENDS = "wayland libinput libxkbcommon ..."
   │          wayland → DEPENDS = "libffi expat libxml2"
   │              libffi → DEPENDS = "gcc-cross"
   │                  ...
   │
   └── 最终得到一张包含 ~500+ 节点的 DAG

③ 按拓扑顺序执行
   对 DAG 中的每个节点，按序执行：
     do_fetch → do_unpack → do_patch → do_configure → do_compile → do_install → do_package

④ Rootfs 组装（只有 image recipe 有这个阶段）
   ├── do_rootfs：安装所有 rpm 到临时目录（模拟目标系统的 /）
   ├── do_image：打包成 ext4 / qcomflash 等格式
   └── 输出到 build/tmp/deploy/images/<MACHINE>/
```

### 5.3 oe-core 的 bbclass 如何介入？

oe-core 里的 `meta/classes-recipe/core-image.bbclass` 定义了 `do_rootfs` 的标准流程。`qcom-minimal-image.bb` 里写的 `inherit core-image` 就是继承了这套流程。

**oe-core 还提供整个 toolchain：**
- `gcc-cross` — 交叉编译器（在 x86 上编译 arm64 代码）
- `glibc` — C 标准库
- `binutils-cross` — 链接器、汇编器

这三个是**所有 recipe 的隐式依赖**。任何一个软件包的编译，BitBake 都会自动先确保 toolchain 就绪。

> **节间关联**：BitBake 的 DAG 节点是什么？就是**第二节**那些 recipe 的 task → 这些 recipe 分散在**第三节**各个 layer 里 → 哪些 layer 参与构建由**第四节**的 KAS 决定 → 最终生成的 DAG 决定了谁先编译谁后编译。

---

## 六、实战拆解：一条 KAS 命令的完整链路

现在我们把前面所有概念串起来，追踪一条真实命令：

```bash
kas build ci/iq-9075-evk.yml:ci/qcom-distro.yml:ci/linux-qcom-rt-6.18.yml
```

### 6.1 第 1 层：`meta-qcom/ci/base.yml`（最底层的骨架）

通过 `includes` 链展开后最先被加载的。它做了最基础的事：

```yaml
distro: nodistro                 # 临时设为"无发行版"，等着被后面覆盖
machine: unset                   # 临时设为"无平台"，等着被后面覆盖
target: [core-image-base]        # 临时目标：最基础镜像

repos:
  oe-core:                       # ← clone 基础零件库
    url: https://github.com/openembedded/openembedded-core
  bitbake:                       # ← clone 构建引擎
    url: https://github.com/openembedded/bitbake

local_conf_header:
  base: |
    INHERIT += "rm_work"          # 编译完就删工作目录，省磁盘
  clo-mirrors: |
    MIRRORS:append = "git://github.com/ git://git.codelinaro.org/clo/yocto-mirrors/github/"
    # ↑ 国内镜像加速
  qcomflash: |
    IMAGE_CLASSES += "image_types_qcom"  # 引入高通专有镜像格式
```

**这一层解决的核心问题：** 没有 oe-core 和 bitbake，后面什么都没法做——它们提供了 gcc、glibc、cmake.bbclass、kernel.bbclass 等一切基础。

### 6.2 第 2 层：`meta-qcom-distro/ci/qcom-distro.yml`（叠加上层生态 + 产品目标）

```yaml
distro: qcom-distro              # ← 覆盖 base 的 nodistro！现在发行版正式定为 qcom-distro

repos:
  meta-openembedded:             # ← 新增：社区大卖场（Python/网络/桌面/...）
  meta-virtualization:           # ← 新增：Docker/LXC
  meta-security:                 # ← 新增：安全
  meta-selinux:                  # ← 新增：SELinux
  meta-updater:                  # ← 新增：OTA
  meta-audioreach:               # ← 新增：音频 DSP 框架

target:                          # ← 覆盖 base 的 core-image-base
  - qcom-multimedia-image        #    现在是这四个产品镜像
  - qcom-multimedia-proprietary-image
  - qcom-container-orchestration-image
  - qcom-networking-image
```

**当 `distro: qcom-distro` 被设置后，发生了什么？**

Yocto 自动去搜索 `conf/distro/qcom-distro.conf`。它找到了 `meta-qcom-distro/conf/distro/qcom-distro.conf`，这个文件只有一行：

```bitbake
require conf/distro/include/qcom-base.inc
```

而这个 `.inc` 文件里定义了完整的发行版策略：

```bitbake
DISTRO_FEATURES:append = " efi pam virtualization wifi x11 ..."
INIT_MANAGER = "systemd"
PACKAGE_CLASSES = "package_rpm"
IMAGE_FSTYPES += "qcomflash"
```

**这些变量如何影响后续构建？**
- `DISTRO_FEATURES` 里的 `wayland` → image recipe 中就可以用 `${@bb.utils.contains('DISTRO_FEATURES', 'wayland', ...)}` 做条件判断
- `INIT_MANAGER = "systemd"` → rootfs 会自动引入 systemd 而不是 sysvinit
- `PACKAGE_CLASSES = "package_rpm"` → 所有包都打成 `.rpm` 格式

### 6.3 第 3 层：`ci/iq-9075-evk.yml`（锁定硬件平台）

```yaml
machine: iq-9075-evk              # ← 覆盖 base.yml 的 unset
```

**当 `machine: iq-9075-evk` 被设置后，发生了什么？**

Yocto 搜索 `conf/machine/iq-9075-evk.conf`，找到了 `meta-qcom/conf/machine/iq-9075-evk.conf`。这个文件通过 `require` 串起了一个**完整的机器配置链**：

```
iq-9075-evk.conf
  │
  ├── require qcom-qcs9100.inc        ← SoC 族配置
  │     ├── require qcom-base.inc     ← 设置内核默认 provider + 串口 + ext4 参数
  │     └── require qcom-common.inc   ← 设置 MACHINE_FEATURES + EFI + DTB 策略
  │
  ├── 本板特有的 DTB 列表
  │     KERNEL_DEVICETREE = "qcom/lemans-evk.dtb ..."
  │
  ├── 本板特有的固件包
  │     MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += "packagegroup-iq-9075-evk-firmware ..."
  │
  └── 本板特有的分区/烧录配置
        UBOOT_CONFIG = "iq-9075-evk"
```

**MACHINE 的设置如何影响内核 recipe？**

看 `meta-qcom/recipes-kernel/linux/linux-qcom_6.18.bb` 的第一行关键代码：

```bitbake
COMPATIBLE_MACHINE = "(qcom)"  # 只有 MACHINE 属于 qcom 族的才编译这个内核
```

以及 `qcom-common.inc` 中有：

```bitbake
MACHINEOVERRIDES =. "qcom:"  # 给所有 qcom 族板子打上 "qcom" 标签
```

**所以链路是：** `KAS machine: iq-9075-evk` → 加载 `iq-9075-evk.conf` → include `qcom-common.inc` → `MACHINEOVERRIDES` 打上 `qcom` 标签 → `COMPATIBLE_MACHINE = "(qcom)"` 匹配通过 → 内核 recipe 可以编译 → 内核使用的 DTB 来自 `iq-9075-evk.conf` 的 `KERNEL_DEVICETREE`。

### 6.4 第 4 层：`ci/linux-qcom-rt-6.18.yml`（选择内核）

```yaml
local_conf_header:
  kernelprovider: |
    PREFERRED_PROVIDER_virtual/kernel = "linux-qcom-rt"  # ← 使用 RT 内核
    PREFERRED_VERSION_virtual/kernel = "6.18%"           # ← 锁定 6.18 版本
```

**这是最高优先级的覆盖。** `PREFERRED_PROVIDER_virtual/kernel` 是 Yocto 的一个特殊机制——多个 recipe 都可以声称自己"提供内核"（`PROVIDES = "virtual/kernel"`），BitBake 通过 `PREFERRED_PROVIDER` 决定选哪个。

**RT 内核 recipe 的连接：**

```bitbake
# linux-qcom-rt_6.18.bb
require linux-qcom_6.18.bb          # ← 继承标准内核的所有配置！

KBUILD_CONFIG_EXTRA:append = " rt.config"  # ← 只多了 RT 抢占配置
```

**所以 RT 内核 = 标准 `linux-qcom` 的全部 + 一个 RT config fragment。** 它复用了同一个 git 仓库、同一个 SRCREV、同一个补丁集——极致简洁。

### 6.5 四层合并后的全景图

```
最终生效的配置（右边覆盖左边）：
                      base.yml + qcom-distro.yml + iq-9075-evk.yml + linux-qcom-rt-6.18.yml
MACHINE        unset  ───────────────────────────→ iq-9075-evk
DISTRO         nodistro ──→ qcom-distro
KERNEL         linux-qcom-next ──────────────────────────────────────→ linux-qcom-rt 6.18
TARGET         core-image-base ──→ 4个产品镜像
EXTRA LAYERS   (无) ────────────→ meta-openembedded/meta-virt/meta-security/...
```

**此时 KAS 执行：**
1. Clone 所有 `repos` 中声明的 git 仓库到本地
2. 生成 `build/conf/bblayers.conf`（列出 15+ 层的绝对路径）
3. 生成 `build/conf/local.conf`（合并所有 `local_conf_header` + `machine` + `distro`）
4. 执行 `bitbake qcom-multimedia-image qcom-multimedia-proprietary-image ...`

> **节间关联**：KAS 的机器配置决定了**第六节**中 kernel recipe 的选择 → distro 配置决定了 rootfs 的特性集 → target image recipe 的 RDEPENDS 决定了下游几百个包的编译顺序。

---

## 七、完整构建流水线：从一条命令到刷机包

### 7.1 五个阶段的端到端流程

```
┌────────────────────────────────────────────────────────────┐
│ 阶段 1：KAS 环境部署                                        │
│   kas build ci/*.yml                                        │
│   ├── git clone oe-core → oe-core/                          │
│   ├── git clone bitbake → bitbake/                          │
│   ├── git clone meta-qcom → meta-qcom/                      │
│   ├── git clone meta-openembedded → meta-openembedded/      │
│   ├── git clone meta-virtualization, meta-security, ...     │
│   ├── 生成 build/conf/bblayers.conf  ← BBLAYERS = 15+ 路径  │
│   └── 生成 build/conf/local.conf      ← MACHINE/DISTRO/... │
├────────────────────────────────────────────────────────────┤
│ 阶段 2：BitBake 解析                                         │
│   ├── 遍历 bblayers.conf 中所有 layer 的 .bb/.bbappend 文件  │
│   ├── 展开 require/include/inherit                            │
│   ├── 应用 PREFERRED_PROVIDER / PREFERRED_VERSION            │
│   └── 构建全局 namespace（每个包名 → recipe 路径映射）       │
├────────────────────────────────────────────────────────────┤
│ 阶段 3：DAG 构建                                             │
│   ├── 从 target image recipe 出发                            │
│   ├── 递归展开 DEPENDS / RDEPENDS                             │
│   ├── 加上隐式依赖（gcc-cross、glibc 对一切包的依赖）        │
│   └── 拓扑排序，确定执行顺序                                  │
├────────────────────────────────────────────────────────────┤
│ 阶段 4：逐 recipe 构建（~500 个包，可并行）                    │
│   对每个 recipe:                                              │
│     do_fetch     ← 去 SRC_URI 指定的地址下载源码到 downloads/
│     do_unpack    ← 解压到 tmp/work/<arch>/<recipe>/<version>/
│     do_patch     ← 打补丁 + bbappend 追加的补丁               │
│     do_configure ← cmake/autoconf/meson 配置                  │
│     do_compile   ← make / ninja 编译                          │
│     do_install   ← make install 到临时 image 目录             │
│     do_package   ← 拆成 .rpm 包（主包、-dev、-dbg、-doc）    │
│     do_package_write_rpm  ← 写入 rpm 文件                     │
├────────────────────────────────────────────────────────────┤
│ 阶段 5：Rootfs 组装 + 镜像生成（仅 image recipe）              │
│   ├── do_rootfs     ← 把所有 .rpm 安装到临时 rootfs 目录     │
│   ├── do_image_ext4 ← 用 mkfs.ext4 打包                      │
│   ├── do_image_qcomflash ← 用高通工具生成线刷包               │
│   └── do_image_complete → 输出到 tmp/deploy/images/iq-9075-evk/
│         ├── qcom-multimedia-image-*.rootfs.ext4    ← 根文件系统
│         ├── qcom-multimedia-image-*.qcomflash      ← 高通线刷包
│         ├── Image                                  ← 内核
│         ├── dtb-qcom/lemans-evk.dtb                ← 设备树
│         └── modules-*.tgz                          ← 内核模块
└────────────────────────────────────────────────────────────┘
```

### 7.2 Image recipe 的继承链全景

了解最终镜像里有什么东西，需要看这条 require 链：

```
core-image.bbclass (oe-core)
  │  定义 CORE_IMAGE_BASE_INSTALL 默认值 = "packagegroup-core-boot"
  │  定义了 do_rootfs 的标准流程
  │
  └── qcom-minimal-image.bb (meta-qcom-distro)
        inherit core-image
        += kernel-modules
        += packagegroup-qcom-utilities-*
        += resize-rootfs
        │
        └── qcom-console-image.bb (meta-qcom-distro)
              require qcom-minimal-image.bb
              += openssh ssh-server
              += packagegroup-qcom-security
              += packagegroup-qcom-virtualization
              │
              └── qcom-multimedia-image.bb (meta-qcom-distro)
                    require qcom-console-image.bb
                    += weston wayland
                    += gstreamer1.0 + plugins
                    += pipewire wireplumber
                    += libcamera
                    += tensorflow-lite
                    += packagegroup-qcom-benchmark
                    += packagegroup-qcom-test-pkgs
```

**每一层叠加，下层不会丢。** 最终镜像包含从 `core-image` 到 `qcom-multimedia-image` 全部 5 层声明的所有包。理解这个链就能回答"为什么我的镜像里有 XXX 包"——去查这 5 层里谁写的 `CORE_IMAGE_BASE_INSTALL`。

### 7.3 Packagegroup 的展开

`qcom-multimedia-image.bb` 里一句：

```bitbake
CORE_IMAGE_BASE_INSTALL += "packagegroup-qcom-benchmark"
```

在运行时展开为：

```
packagegroup-qcom-benchmark
  └── RDEPENDS = "coremark dhrystone glmark2"
        ├── coremark    → 自己的 DEPENDS 链
        ├── dhrystone   → 自己的 DEPENDS 链
        └── glmark2     → DEPENDS = "mesa libpng libjpeg-turbo ..."
                            └── mesa → DEPENDS = "wayland libdrm llvm ..."
                                         └── ... 继续展开 ...
```

**Packagegroup 就是"快捷方式"**——你不用在 image 里列 20 个包名，只需引用一个 packagegroup，它帮你展开。

---

## 八、QCLinux 目录全景图（每个目录的角色和联系）

```
qclinux/                         ← 项目根目录
│
├── oe-core/                     ← 【地基】基础建材市场
│   ├── meta/classes/            ← 400+ bbclass（cmake、autotools、kernel、image...）
│   │     ↑ 被所有 recipe 的 "inherit xxx" 所引用
│   ├── meta/conf/bitbake.conf   ← 全局默认变量（所有 recipe 的隐含父配置）
│   ├── meta/recipes-core/       ← glibc、busybox、systemd、util-linux
│   ├── meta/recipes-devtools/   ← gcc-cross、binutils-cross、make、cmake
│   └── scripts/                 ← oe-init-build-env、wic、devtool
│
├── bitbake/                     ← 【引擎】施工队总指挥（纯 Python）
│   ├── lib/bb/parse/            ← .bb 文件解析器
│   ├── lib/bb/runqueue.py       ← DAG 构建 + 并行调度
│   └── lib/bb/fetch2/           ← git/http/ftp 下载器
│
├── meta-qcom/                   ← 【BSP 层】高通芯片专有零件
│   ├── conf/machine/            ← 19 个板子的 machine config
│   │     iq-9075-evk.conf ──require──→ qcom-qcs9100.inc
│   │                                   └──→ qcom-base.inc ──→ qcom-common.inc
│   ├── recipes-kernel/linux/    ← 4 个内核 recipe
│   │     linux-qcom_6.18.bb ←── require ── linux-qcom-rt_6.18.bb
│   │     标准内核                          RT 内核（只多一个 rt.config）
│   ├── recipes-bsp/firmware-boot/ ← 每个板子的启动固件
│   ├── recipes-graphics/        ← GPU 用户态驱动（Adreno/Mesa/KGSL）
│   └── classes-recipe/image_types_qcom.bbclass ← qcomflash 格式
│
├── meta-qcom-distro/            ← 【产品层】在 BSP 之上定义发行版策略 + 产品镜像
│   ├── conf/distro/             ← 5 个 distro 变体
│   │     qcom-distro.conf ──require──→ qcom-base.inc
│   │     qcom-distro-kvm.conf ──require──→ qcom-base.inc + qcom-distro-kvm.inc
│   ├── conf/layer.conf          ← LAYERDEPENDS = "core qcom openembedded-layer ..."
│   ├── recipes-products/images/ ← 5 个产品 image（继 承 链）
│   │     qcom-minimal-image.bb  ← 基础
│   │       └── qcom-console-image.bb ← +SSH +工具
│   │             ├── qcom-multimedia-image.bb ← +Wayland +GStreamer +PipeWire
│   │             ├── qcom-networking-image.bb
│   │             └── qcom-container-orchestration-image.bb
│   ├── recipes-products/packagegroups/ ← 6 个包组（购物清单）
│   └── ci/                      ← 40+ 个 KAS 配置文件
│         base.yml、qcom-distro.yml、iq-9075-evk.yml、linux-qcom-rt-6.18.yml...
│         这些文件互相 include，组合出所有 MACHINE×KERNEL×DISTRO 的构建矩阵
│
├── meta-openembedded/           ← 【综合超市】社区生态（600+ 个 recipe）
│   ├── meta-python/             ← Python 库
│   ├── meta-networking/         ← iperf3、curl、openssl...
│   ├── meta-multimedia/         ← 编解码器
│   ├── meta-oe/                 ← 杂项工具
│   └── meta-xfce/               ← XFCE 桌面（qcom-xfce-demo-image 依赖它）
│
├── meta-virtualization/         ← Docker、runc、Kubernetes、QEMU
├── meta-security/ + meta-tpm/   ← TPM2.0、dm-verity、IMA/EVM、Suricata
├── meta-selinux/                ← SELinux 用户态工具 + 策略
├── meta-updater/                ← OTA 更新（Uptane/TUF/Aktualizr）
├── meta-audioreach/             ← 高通 AudioReach DSP 音频框架
│
└── build/                       ← 【工地】所有构建输出
    ├── conf/
    │   ├── local.conf           ← KAS 生成的构建配置
    │   └── bblayers.conf        ← KAS 生成的 layer 路径列表
    ├── downloads/               ← 源码缓存（SRC_URI 下载到这里）
    ├── sstate-cache/            ← 编译缓存（没变的 recipe 下次直接复用）
    └── tmp/
        ├── work/                ← 每个 recipe 的编译工作区
        │     .../linux-qcom-rt/6.18.21/  ← 内核源码解压 + 编译
        │     .../weston/13.0/            ← weston 源码解压 + 编译
        ├── deploy/
        │   ├── images/iq-9075-evk/  ← 最终产物（镜像、内核、DTB）
        │   └── rpm/                  ← 所有编译好的 .rpm 包
        └── sysroots/             ← 交叉编译 sysroot（目标系统头文件/库）
```

---

## 九、总结：所有模块之间的连线图

用一张 ASCII 连线图呈现全部模块的关系：

```
                    ┌──────────────┐
                    │   开发者     │
                    └──────┬───────┘
                           │ 写 KAS YAML 文件
                           ▼
              ┌────────────────────────┐
              │         KAS            │  ← 配置管理层
              │  (ci/*.yml 互相 include)│
              │  机器 + 发行版 + 内核   │
              └────┬───────┬───────────┘
                   │       │
         clone 仓库 │       │ 生成 local.conf + bblayers.conf
                   ▼       ▼
   ┌─────────────────┐   ┌─────────────────┐
   │  15+ 个 Layer    │   │  build/conf/    │
   │  (git 仓库)      │   │  local.conf     │
   │                  │   │  bblayers.conf  │
   └────────┬────────┘   └────────┬────────┘
            │                     │
            │ BBLAYERS 指向       │ MACHINE/DISTRO/...
            ▼                     ▼
   ┌─────────────────────────────────────────┐
   │              BitBake                     │  ← 调度引擎
   │   ① 解析所有 layer 里几千个 recipe       │
   │   ② 根据 DEPENDS/RDEPENDS 构建 DAG       │
   │   ③ 拓扑排序，并行调度 task               │
   └──────────────────┬──────────────────────┘
                      │ 调用
                      ▼
   ┌─────────────────────────────────────────┐
   │        oe-core 的 bbclass               │  ← 任务模板
   │  cmake.bbclass    → do_configure        │
   │  kernel.bbclass   → do_compile (内核)    │
   │  image.bbclass    → do_rootfs           │
   │  base.bbclass     → do_fetch/unpack     │
   └──────────────────┬──────────────────────┘
                      │ 按模板执行
                      ▼
   ┌─────────────────────────────────────────┐
   │        逐 recipe 执行 task               │  ← 实际构建
   │  fetch → unpack → patch → configure     │
   │  → compile → install → package          │
   │                                          │
   │  meta-qcom:   内核 + GPU + firmware      │
   │  meta-qcom-distro: image + packagegroup  │
   │  meta-openembedded: 社区软件              │
   │  meta-virt/security/selinux/...: 专项    │
   └──────────────────┬──────────────────────┘
                      │ 所有包打完 rpm
                      ▼
   ┌─────────────────────────────────────────┐
   │           do_rootfs                     │  ← 组装
   │  安装所有 rpm → 打包 ext4/qcomflash     │
   └──────────────────┬──────────────────────┘
                      │
                      ▼
   ┌─────────────────────────────────────────┐
   │   build/tmp/deploy/images/iq-9075-evk/  │  ← 最终产物
   │   *.rootfs.ext4  *.qcomflash            │
   │   Image  dtb/  modules-*.tgz            │
   └─────────────────────────────────────────┘
```

**三句话记住整个体系：**

1. **KAS 拼积木**决定用哪些 layer、什么硬件、什么发行版 → 生成 Yocto 环境
2. **BitBake 画 DAG**：从 image recipe 的 `RDEPENDS` 出发，递归展开所有依赖，得到 ~500 个节点的有向无环图
3. **按图施工**：拓扑顺序执行 fetch→compile→install，最后 `do_rootfs` 把全部 rpm 装进一个目录，打包成刷机包
