# 蓝牙SCO/eSCO HCI命令参数完整手册（0x0028 / 0x003D 最新Spec合规版）

## 一、核心命令总览与精简对比（最新BT 5\.4\+ Spec）

本文适配**蓝牙Core Spec 5\.4及以上最新标准**，修正传统旧版协议误区，明确新旧SCO/eSCO建链命令的能力、参数差异、生效逻辑，适配HFP窄带\(CVSD\)、宽带\(mSBC\)通话场景。

两条核心建链命令：

- **传统命令**：Setup\_Synchronous\_Connection（OCF=0x0028），适配SCO/eSCO、仅窄带语音

- **增强命令**：Enhanced\_Setup\_Synchronous\_Connection（OCF=0x003D），仅eSCO、支持mSBC宽带语音

|对比项目|Setup\_Synchronous\_Connection（OCF=0x0028）|Enhanced\_Setup\_Synchronous\_Connection（OCF=0x003D）|
|---|---|---|
|语音配置方式|单2字节 Voice\_Setting 复合位域（新版Spec仅1字节有效）|废弃Voice\_Setting，拆分为全套独立细粒度参数|
|空中编码配置|Tx/Rx 共用同一套Air Coding Format，无法独立配置|Transmit/Receive\_Coding\_Format 收发完全独立|
|Codec帧尺寸|无独立配置字段，无法适配mSBC固定帧长|独立 Transmit/Receive\_Codec\_Frame\_Size 精准配置|
|PCM精细控制|仅少量压缩bit位，无独立位对齐、传输单元配置|独立MSB位偏移、PCM格式、传输单元大小，硬件适配性极强|
|音频通路DataPath|**新版Spec已移除DataPath位域**，无法动态配置路由，仅兼容旧固件|独立1字节 Input/Output\_Data\_Path，动态选择HCI/硬件通路|
|支持编码格式|CVSD、μ\-law、A\-law、Transparent（无mSBC）|兼容所有传统编码，新增**mSBC宽带编码**，支持HFP WBS|
|支持链路类型|同时支持 传统SCO / 增强eSCO|**仅支持eSCO**，不兼容老式SCO链路|
|链路修改能力|可修改eSCO链路，但带宽、语音参数无法精细化调整|完整支持eSCO链路参数动态修改|

## 二、Enhanced\_Setup\_Synchronous\_Connection（OCF=0x003D）完整规范

### 2\.1 命令基础信息

Opcode：OGF=0x01，OCF=0x003D，完整Opcode=0x043D

用途：蓝牙BR/EDR，建立/修改**eSCO同步语音链路**，专用于HFP宽带\(mSBC\)、窄带通话

### 2\.2 核心强制规则（研发必守）

1. **参数生效开关由DataPath决定，与Coding Format无关**：Input/Output\_Data\_Path 控制本地PCM参数是否生效，空中编码格式仅控制eSCO射频链路，互不干扰。

2. 命令为**固定长度HCI数据包**，所有字段必须完整填充；逻辑忽略的字段需填0占位，不可删减字节，否则报参数错误。

3. Transmit/Receive\_Coding\_Format 仅定义空中射频编码，不参与本地音频通路配置。

### 2\.3 全参数生效明细

|参数名称|生效条件|使用场景|忽略场景|
|---|---|---|---|
|Connection\_Handle|永久生效|绑定ACL/eSCO链路句柄，标识链路载体|永不忽略|
|Transmit\_Bandwidth|永久生效|协商eSCO空中发送带宽，基带时隙资源预留|永不忽略|
|Receive\_Bandwidth|永久生效|协商eSCO空中接收带宽|永不忽略|
|Transmit\_Coding\_Format|永久生效|配置设备向对端发送的空中编码（CVSD/mSBC/透明）|永不忽略|
|Receive\_Coding\_Format|永久生效|配置设备接收对端数据的空中编码|永不忽略|
|Transmit\_Codec\_Frame\_Size|永久生效|空中发送侧单帧编码数据字节长度（mSBC固定30字节）|永不忽略|
|Receive\_Codec\_Frame\_Size|永久生效|空中接收侧单帧编码数据字节长度|永不忽略|
|Input\_Bandwidth|Input\_Data\_Path \!= 0|控制器从本地PCM/I2S硬件采集音频的带宽|Input\_Data\_Path=0x00（音频走HCI同步包）|
|Output\_Bandwidth|Output\_Data\_Path \!= 0|控制器向本地PCM/I2S硬件输出音频的带宽|Output\_Data\_Path=0x00|
|Input\_Coding\_Format|Input\_Data\_Path \!= 0|本地硬件输入音频的编码格式（线性PCM/压缩编码）|Input\_Data\_Path=0x00|
|Output\_Coding\_Format|Output\_Data\_Path \!= 0|本地硬件输出音频的编码格式|Output\_Data\_Path=0x00|
|Input\_Coded\_Data\_Size|Input\_Data\_Path \!= 0|本地输入音频单块数据大小|Input\_Data\_Path=0x00|
|Output\_Coded\_Data\_Size|Output\_Data\_Path \!= 0|本地输出音频单块数据大小|Output\_Data\_Path=0x00|
|Input\_PCM\_Data\_Format|Input\_Data\_Path \!= 0|PCM采样数值格式（2补码/反码/无符号等）|Input\_Data\_Path=0x00|
|Output\_PCM\_Data\_Format|Output\_Data\_Path \!= 0|输出PCM采样数值格式|Output\_Data\_Path=0x00|
|Input\_PCM\_Sample\_Payload\_MSB\_Position|Input\_Data\_Path \!= 0|PCM样本有效位MSB偏移，用于硬件位对齐适配|Input\_Data\_Path=0x00|
|Output\_PCM\_Sample\_Payload\_MSB\_Position|Output\_Data\_Path \!= 0|输出PCM样本有效位偏移|Output\_Data\_Path=0x00|
|Input\_Data\_Path（核心开关）|永久生效|选择音频输入通路：0=HCI通路，非0=本地硬件PCM/I2S通路|永不忽略|
|Output\_Data\_Path（核心开关）|永久生效|选择音频输出通路：0=HCI通路，非0=本地硬件通路|永不忽略|
|Input\_Transport\_Unit\_Size|Input\_Data\_Path \!= 0|本地硬件单次输入传输单元大小|Input\_Data\_Path=0x00|
|Output\_Transport\_Unit\_Size|Output\_Data\_Path \!= 0|本地硬件单次输出传输单元大小|Output\_Data\_Path=0x00|
|Max\_Latency|永久生效|eSCO链路最大端到端延迟，参与LMP链路协商|永不忽略|
|Packet\_Type|永久生效|eSCO允许的数据包类型掩码（EV3/EV5等）|永不忽略|
|Retransmission\_Effort|永久生效|eSCO基带丢器重传策略配置|永不忽略|

### 2\.4 标准业务场景配置

#### 场景1：Host侧编解码（Linux BlueZ/oFono，WBS mSBC通用）

配置：Input\_Data\_Path=0x00，Output\_Data\_Path=0x00

逻辑：Host完成PCM↔mSBC编解码，控制器仅透传eSCO数据包，无本地硬件音频交互

生效参数：所有空中链路参数 \+ 双DataPath \+ 延迟/包类型/重传策略

忽略参数：全部Input\_\*/Output\_\* PCM本地参数（包内填0占位）

推荐空中编码：Transparent\(0x03\)（控制器不做编解码，纯透传Host编码帧）

#### 场景2：控制器硬件编解码（耳机/车载SoC内置Codec）

配置：Input\_Data\_Path≠0，Output\_Data\_Path≠0

逻辑：控制器直接读写PCM/I2S硬件，内部完成CVSD/mSBC编解码，音频不走HCI包

生效参数：**全部参数**，无忽略字段

## 三、Setup\_Synchronous\_Connection（OCF=0x0028）最新Spec完整规范

### 3\.1 命令基础信息

Opcode：OGF=0x01，OCF=0x0028，完整Opcode=0x0428

用途：蓝牙BR/EDR，建立**传统SCO/eSCO窄带语音链路**，仅支持CVSD等老旧编码，不支持mSBC

**重大Spec变更（5\.4\+）**：新版协议已删除Voice\_Setting中的DataPath位域，该命令**无法动态配置音频通路**，仅兼容旧固件；仅0x003D/0x003E支持通路选择。

### 3\.2 新版Voice\_Setting 1字节标准位域（最终定稿）

尺寸：1 octet（有效位bit0\~7，bit8\~15保留无意义）

|Bit域|参数说明|取值定义|
|---|---|---|
|0\~1|Air coding format 空中编码|0=CVSD、1=μ\-law、2=A\-law、3=Transparent透明传输|
|2\~4|Linear PCM bit位置偏移|仅线性PCM生效，标识样本MSB偏移位数|
|5|Input sample size 采样位宽|0=8bit、1=16bit（仅线性PCM生效）|
|6\~7|Input data format PCM格式|0=反码、1=2补码、2=符号幅值、3=无符号|
|8\~15|保留位（新版Spec）|**无DataPath功能**，旧固件兼容解析，新固件全部忽略|

### 3\.3 命令参数与生效规则

|参数名称|字节数|说明|生效规则|
|---|---|---|---|
|Connection\_Handle|2|ACL链路句柄，同步链路挂靠载体|永久生效|
|Transmit\_Bandwidth|4|空中发送带宽|永久生效|
|Receive\_Bandwidth|4|空中接收带宽|永久生效|
|Voice\_Setting|2|复合语音配置，新版仅低8位有效，无DataPath|永久生效|
|Max\_Latency|2|eSCO最大延迟|eSCO生效，SCO忽略（填0占位）|
|Packet\_Type|2|SCO/eSCO数据包类型|永久生效|
|Retransmission\_Effort|1|eSCO重传策略|eSCO生效，SCO忽略（填0占位）|

## 四、最终核心总结（研发极简口诀）

1. **0x0028老命令**：无独立DataPath、无独立收发编码、不支持mSBC，新版Spec彻底移除通路配置能力，仅用于老旧窄带SCO/eSCO。

2. **0x003D增强命令**：全参数精细化配置，收发编码独立、支持mSBC宽带、可自由切换HCI/硬件音频通路，是现代HFP通话唯一标准方案。

3. 所有本地PCM参数（Input\_\*/Output\_\*）**只看DataPath**，与空中Coding Format无关。

4. 透明传输Transparent仅代表**空中链路不编解码**，适配Host侧mSBC硬解场景，必须搭配DataPath=0x00。

5. 所有HCI命令包固定长度，忽略字段必须0占位，严禁删减字节。

> （注：部分内容可能由 AI 生成）
