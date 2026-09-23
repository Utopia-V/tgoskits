# SG200x 麦克风采集

`sg200x-audio` 提供板载 ADC 左声道的单声道 S16_LE 采集，支持 16 kHz、48 kHz。硬件核心不依赖 StarryOS；设备树、IRQ 注册和用户接口分别由 `ax-driver::audio`、StarryOS `pseudofs::dev::audio` 接入。

## 1. 硬件接口

`Capture` 持有控制与采集队列，`Interrupt` 只确认中断并锁存错误。平台通过 `Resources` 传入 MMIO 映射，通过 `DeviceDma` 提供 DMA 内存，不在驱动中固定板级物理地址。

### 1.1 时钟与输入

`Frontend::prepare` 从固件留下的 MIPIMPLL、APLL 和 synthesizer 寄存器计算实际父时钟，只调整 AUD0/AUD3 分频，不重编程共享 PLL。无法精确产生目标时钟时返回 `Error::Clock`。ADC 先输出 BCLK，再等待 I²S 接收器复位完成；复位超时返回错误。

`set_gain` 接受 0..=24，对应 0..48 dB、每步 2 dB。零表示最低增益，不表示静音。输入接 ADC 左路，I²S0 使用单 slot、16 位内存格式，I²S3 只提供 ADC 所需的 MCLK。

### 1.2 DMA 所有权

`DmaRoute` 区分物理通道和握手请求号。`Ring` 使用 64 字节对齐的描述符、128 字节硬件块和一致性音频缓冲区；一个通知周期可以含多个硬件块。`Config` 要求周期为 64 帧的倍数，范围 64..8192 帧，缓冲区包含 2..16 个完整周期且不超过 32768 帧。

`Cursor` 根据目的地址累计进度，不把一次 IRQ 当作一个周期。无法排除漏过整圈时报告溢出；消费方在复制后再次检查进度，拒绝已被覆盖的数据。DMA 内存通过易失访问读取，不建立指向设备正在写入区域的 Rust 引用。

`stop` 只中止本通道，并有界等待 `CH_EN` 清零。停机失败会保留 DMA 内存，将对象置为不可重用状态；不会复位共享控制器或释放设备仍可能访问的内存。探测阶段设置共享路由，要求尚无活动 DMA 通道。

## 2. 系统接入

StarryOS 的 `sg2002-audio` feature 启用设备探测和 ALSA 兼容入口。只有硬件资源与 IRQ 注册成功后才发布设备节点；没有 SG2002 音频资源的机器不会出现假声卡。

### 2.1 录音会话

`AudioFile` 是打开文件描述的所有者：一个独立采集打开占用设备，另一个返回 `EBUSY`；`dup` 和 `fork` 共享同一个 `Arc`，最后引用关闭时停止采集。`Stream` 维护参数、状态、硬件进度和应用进度，支持参数协商、准备、启动、读取、停止、排空和溢出后的重新准备。

`pcmC0D0c` 提供交错帧读取、状态、`SYNC_PTR` 和 `poll`；`controlC0` 提供声卡查询及单值 `ADC Capture Volume`。不提供非交错 `readv`、音频 mmap、播放、混音、重采样或 control 事件订阅。硬中断只通知固定服务线程，读者唤醒在任务上下文执行。用法和板测入口见 [录音诊断](../../../apps/starry/sg2002-audio/README.md)。

### 2.2 资料与验证边界

寄存器配置参考 Sipeed SDK 提交 `d4003f15b35d43ad4842f427050ab2bba0114fa5` 的 [ADC](https://github.com/sipeed/LicheeRV-Nano-Build/blob/d4003f15b35d43ad4842f427050ab2bba0114fa5/linux_5.10/sound/soc/cvitek/cv181xadc.c)、[I²S](https://github.com/sipeed/LicheeRV-Nano-Build/blob/d4003f15b35d43ad4842f427050ab2bba0114fa5/linux_5.10/sound/soc/cvitek/cv1835_i2s.c) 和 [DMA](https://github.com/sipeed/LicheeRV-Nano-Build/blob/d4003f15b35d43ad4842f427050ab2bba0114fa5/linux_5.10/drivers/dma/cvitek/cvitek-dma.c)。原实现版权归 CVITEK 等原作者；本软件包采用 GPL-2.0-or-later。Linux ABI 以 v6.6 为基准，消费者调用链对照 alsa-lib v1.2.14。

构建和逻辑测试不能确认模拟前端、真实采样率、16 位样本打包、IRQ 时序或音质。实板验收需要冷启动、两档采样率、增益变化、持续录音、故意溢出恢复，以及停止和进程退出后的再次录音；DMA 和 unsafe 边界在合入前仍需领域审查。

LicheeRV Nano 的两档 `arecord` 采集和生命周期诊断已有实板结果。首次采集存在约两秒的启动瞬态，原有 Linux 对照也出现相同现象；本驱动不额外丢弃这些样本。受控声源的采样率与增益测量、主观音质仍需核验。
