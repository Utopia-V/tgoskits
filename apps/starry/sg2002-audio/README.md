# SG2002 录音诊断

`sg2002-audio-check` 通过 Linux ALSA ioctl 检查板载单声道采集的参数、状态和资源释放。WAV 录制使用现成的 `arecord`。StarryOS 的 `sg2002-audio` feature 接入 ADC、I²S 和循环 DMA。

## 1. 构建

`audio-check.c` 使用目标 Linux 的 `sound/asound.h` 和 C 运行库；`CMakeLists.txt` 将程序安装到 `bin/sg2002-audio-check`。源码针对 64 位小端平台，不能用裸机 Rust 工具链链接 Linux 用户态程序。

### 1.1 宿主检查

在仓库根目录执行以下命令。`--abi-check` 只验证 Linux C 头文件的大小、偏移和 ioctl 编码，不访问声卡。

```sh
cmake -S apps/starry/sg2002-audio -B /tmp/sg2002-audio-check -DCMAKE_BUILD_TYPE=Release
cmake --build /tmp/sg2002-audio-check
/tmp/sg2002-audio-check/sg2002-audio-check --abi-check
```

成功标记为 `ALSA_LP64_LAYOUT_OK`，与 `drivers/interface/alsa-pcm-uapi` 的 Rust 布局测试相互对照。

### 1.2 板端程序

板端需要 RISC-V64 Linux 用户态交叉编译器及匹配的 sysroot。板测目录 `test-suit/starryos/board-aka-00-sg2002/audio` 复用本目录的 CMake 目标，构建后通过会话文件部署，不预装到 Linux 根文件系统。手动构建音频内核可使用：

```sh
cargo xtask starry build -c os/StarryOS/configs/board/aka-00-sg2002.toml
```

同一音频 feature 也接入了 `licheerv-nano-sg2002.toml`。关闭该 feature 时不探测音频硬件、不创建 `/dev/snd` 节点。

## 2. 板上检查

`open_capture()` 访问 `/dev/snd/pcmC0D0c`，`check_gain()` 访问 `/dev/snd/controlC0` 的单值 `ADC Capture Volume`。运行期间应独占声卡；原厂 Linux 的双值增益控件不满足此检查的前提。

### 2.1 录音文件

使用 `arecord` 验证 alsa-lib 协商和实际 WAV 输出，并回放检查声音、削顶和采样速率。输出文件名应选择尚不存在的路径。

```sh
arecord -D hw:0,0 -c 1 -f S16_LE -r 16000 -d 5 -t wav mic-16k.wav
arecord -D hw:0,0 -c 1 -f S16_LE -r 48000 -d 5 -t wav mic-48k.wav
```

### 2.2 生命周期

`lifecycle()` 检查增益改变及恢复、立体声拒绝、不完整帧和非交错 `readv` 拒绝、第二次打开返回 `EBUSY`、停止重启，以及 `dup/fork` 共享会话后由进程退出释放活动采集。`check_blocking_and_overrun()` 检查非阻塞 `EAGAIN`、信号打断、读取量小于 `avail_min` 时的进展、参数变更后阻塞读与 `select` 的唤醒，以及溢出的 `POLLERR/EPIPE` 与重新配置。`check_drain()` 区分已在进行的读取和排空后新发起的读取，检查停止、溢出后的数据不被恢复。`check_geometry()` 验证最小可用 DMA 环的多次回绕。`capture_one_second()` 处理短读、非阻塞等待和超时，再检查 `SYNC_PTR/STATUS`；失败立即退出，由进程退出路径释放描述符。

```sh
timeout 30 ./sg2002-audio-check --lifecycle 16000
timeout 30 ./sg2002-audio-check --lifecycle 48000
```

全部步骤成功才输出 `SG2002_AUDIO_PASSED`；断言失败输出 `SG2002_AUDIO_FAILED` 并返回非零。阻塞录音由进程定时信号限时，外层超时拦截内核调用挂起；任一非零退出均由板卡运行器判失败。缺少设备节点时必须失败。上述命令要求板端提供 `timeout`。生命周期诊断不检查音质；持续采集、冷启动瞬态和受控声源仍需单独验证。

需要定向诊断时，可将 `--lifecycle` 替换为 `--blocking`、`--drain` 或 `--geometry`，分别只运行对应检查；采样率参数保持不变。

### 2.3 板卡运行器

`board-aka-00-sg2002/audio` 连续检查两档采样率，只有两次均通过才匹配总成功标记。下载诊断程序需要板端网络；`requirements.toml` 要求在本机环境设置 Wi-Fi 参数，勿将凭据写入仓库。

```sh
cargo xtask starry test board --list
cargo xtask starry test board -c board-aka-00-sg2002/audio --board aka-00-sg2002
```

运行器检查生命周期，`arecord` 另外验证实际 alsa-lib 协商与 WAV 输出。自动测试通过后仍需试听并检查真实采样率，不能将字节数正确当作麦克风已经正常工作。
