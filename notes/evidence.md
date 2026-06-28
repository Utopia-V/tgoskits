# 贡献证据表

## 直接合入 PR

| PR | 状态 | 主题 | 报告定位 |
| --- | --- | --- | --- |
| [#670](https://github.com/rcore-os/tgoskits/pull/670) | merged | `eventfd2` syscall 测例 | 方案一，syscall 语义测试 |
| [#683](https://github.com/rcore-os/tgoskits/pull/683) | merged | `signalfd4` 测例，修复 `ssi_pid` / `ssi_uid` | 方案一，信号语义修复 |
| [#763](https://github.com/rcore-os/tgoskits/pull/763) | merged | `utimensat` 测例与内核语义修复 | 方案一，时间戳 syscall 兼容 |
| [#1025](https://github.com/rcore-os/tgoskits/pull/1025) | merged | loongarch64 `to_bin` 支持与 rename 测例 | Git rename 回归补齐 |
| [#1026](https://github.com/rcore-os/tgoskits/pull/1026) | merged | Git 本地 stress suite | Git 本地工作流覆盖 |
| [#1169](https://github.com/rcore-os/tgoskits/pull/1169) | merged | Git `git://` remote stress probes | Git 网络 remote 第一层 |
| [#1178](https://github.com/rcore-os/tgoskits/pull/1178) | merged | Git HTTPS remote 与 loongarch64 LASX 修复 | Git HTTPS + 架构状态管理 |
| [#1248](https://github.com/rcore-os/tgoskits/pull/1248) | merged | Rockchip RGA dry-run command buffer | 方案三探索，不作为 Git 主线 |
| [#1319](https://github.com/rcore-os/tgoskits/pull/1319) | merged | Git SSH 触发的 socket QoS 兼容修复 | Git/OpenSSH socket 语义补齐 |

## 相关协作 PR

| PR | 作者 | 关系 | 报告表述 |
| --- | --- | --- | --- |
| [#807](https://github.com/rcore-os/tgoskits/pull/807) | `aptacc2421` | 修复 VFS rename 祖先检查和 ext4 dentry 删除 | Git 路径暴露 rename 语义缺口；相关修复由 #807 合入，后续通过 #1025/#1026 补回归覆盖 |

## Issue

- [#579](https://github.com/rcore-os/tgoskits/issues/579)：`[StarryOS] 方案一：Syscall 测例与语义完善 + 方案二：支持 git`
  - 当前状态为 open，但 body 已标注方案一完成、方案二 Git app 支持完成并合入。
  - Git 方向明确 deferred：SSH 认证矩阵、credential helper、完整 CA/TLS 边界、外部 writable remote、LFS/submodule。

## 可讲的核心 bug

### VFS rename 语义缺口

- 触发方式：Git 本地操作中 `git branch -m` / ref rename 相关路径。
- 问题本质：目录防循环检查用于防止“目录移入自身子树”，但过宽地拦截了普通文件移动到子目录。
- 修复归属：相关 VFS 修复在 #807 合入；Git 线通过 #1025/#1026 补充 rename 回归和 Git stress 覆盖。
- 公开表述：这是 Git 真实应用路径暴露的文件系统语义缺口；#807 是相关协作修复，Git 线贡献在于补充回归覆盖。

### loongarch64 LASX 用户态状态保存恢复

- 触发方式：Git HTTPS remote 依赖 Python `ssl` / OpenSSL，loongarch64 上触发用户态异常。
- 问题本质：内核暴露/启用了用户态向量能力，但没有完整保存恢复 LASX 256-bit 状态。
- 修复内容：开启 `EUEN.ASXE`，扩展 `FpuState` 保存恢复 `xr0..xr31`，同步 `AT_HWCAP` 和 `/proc/cpuinfo`。
- 验证：`openssl-loongarch` regression 覆盖 OpenSSL rand/genrsa/req、Python ssl、`AT_HWCAP`、`/proc/cpuinfo`。

### socket QoS 与 `recvmsg` cmsg

- 触发方式：Git SSH remote / OpenSSH client 设置 `IP_TOS`，并暴露 `IPV6_TCLASS`、`SO_PRIORITY`、`IP_RECVTOS`、`IPV6_RECVTCLASS` 兼容缺口。
- 问题本质：
  - 出站 `IP_TOS` 没有真正写入 IP header。
  - Starry socket option 分发和 ax-net 状态缺少 QoS 选项。
  - `recvmsg` 失败路径提前清零 `msg_controllen`，导致非阻塞 retry 后丢失 cmsg。
- 修复内容：补 socket option 分发、ax-net per-socket QoS 状态、出站 IPv4/IPv6 header 写入、RX metadata、UDP receive cmsg、`CMsgBuilder` 延迟写回。
- 验证：`bugfix-bug-socket-qos-options`、`bugfix-bug-recv-qos-cmsg`、Git SSH app、AF_UNIX cmsg 回归。

## 当前覆盖范围

可以说：

- 已覆盖 Alpine Git 本地主要工作流。
- 已覆盖 `git://` remote 和 HTTPS smart HTTP remote 的核心 clone/fetch/pull/push。
- 已修复或协同推动修复 Git 路径暴露的文件系统、架构状态、网络 socket 语义问题。
- 已补齐对应 regression，避免后续回归。

后续边界：

- Git 仍可继续扩展 SSH 认证矩阵、credential helper、LFS/submodule 等场景。
- HTTPS/TLS 方向仍可继续覆盖完整 CA trust store、代理认证和更多边界条件。
- QoS 方向当前聚焦 socket option 可见语义，未扩展到完整 qdisc/device priority 调度模型。
- #807 作为相关协作修复引用，不归为本人直接代码修改。
