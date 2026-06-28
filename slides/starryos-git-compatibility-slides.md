# 以 Git 为牵引的 StarryOS Linux 兼容性改进

Utopia-V  
2026 春季开源操作系统训练营

---

## 1. 项目背景

StarryOS 的 Linux 兼容性不只是让命令启动。

真实应用会同时依赖：

- syscall 参数语义
- 用户态内存读写
- 文件系统 rename / ref 更新
- socket / TCP loopback
- TLS / OpenSSL
- CPU feature 与上下文保存恢复

---

## 2. 工作路线

```text
syscall 语义测例
        ↓
Alpine Git 本地工作流
        ↓
git:// remote
        ↓
HTTPS smart HTTP
        ↓
SSH / OpenSSH
```

方法：真实应用失败 -> 缩小成内核语义 -> 修复 -> regression。

---

## 3. 阶段工作概览

| 阶段 | PR | 内容 |
| --- | --- | --- |
| 方案一 | #670 | `eventfd2` 测例 |
| 方案一 | #683 | `signalfd4` 测例与 `ssi_pid` / `ssi_uid` 修复 |
| 方案一 | #763 | `utimensat` 测例与语义修复 |
| 方案二 | #1026 | Git 本地 stress suite |
| 方案二 | #1169 | `git://` remote stress |
| 方案二 | #1178 | HTTPS remote + LASX 修复 |
| 方案二 | #1319 | Git SSH + socket QoS 修复 |

---

## 4. 主线一：syscall 语义

| syscall | 覆盖重点 |
| --- | --- |
| `eventfd2` | flag、计数器/信号量、非阻塞、溢出、fork 继承 |
| `signalfd4` | signal mask、返回信息字段、`ssi_pid` / `ssi_uid` |
| `utimensat` | 时间戳更新、flag 校验、`AT_EMPTY_PATH` |

这部分建立了后续工作的基本方法：

Linux 语义 -> 源码级测例 -> StarryOS 差异 -> 内核修复 -> 回归测试。

---

## 5. 主线二：Git 分层测试

| 层次 | 覆盖操作 | 主要验证点 |
| --- | --- | --- |
| 本地 Git | 13 个 probe | 文件系统、ref/reflog、rename |
| `git://` | `clone/fetch/pull/push` | socket、TCP loopback、pack/ref |
| HTTPS | `clone/fetch/pull/push` | TLS、OpenSSL、Python `ssl` |
| SSH | `clone/fetch/pull/push` | OpenSSH socket option、QoS cmsg |

重点不是“一个命令跑通”，而是把 Git 拆成可复现、可定位的路径。

---

## 6. 本地 Git 与 rename

本地 Git stress suite 覆盖：

- `init/config/add/commit/log/status/diff`
- `branch/checkout/reset/merge/stash/tag`

暴露的典型语义：

- Git ref 更新会移动普通文件。
- VFS 需要区分“目录移入自身子树”和“普通文件移动到子目录”。
- 相关 VFS 修复由 #807 合入；Git 线补充 rename 回归覆盖。

---

## 7. Remote 测试设计

测试都尽量在 guest 内闭环：

```text
git://    -> guest 内 git daemon
HTTPS     -> Python ssl + git http-backend
SSH       -> guest 内 sshd + Git client
```

好处：

- 不依赖公网仓库
- 不依赖外部 writable remote
- 失败更容易归因到 StarryOS 行为

---

## 8. 关键问题：LASX 状态

HTTPS remote 在 loongarch64 上触发 OpenSSL / Python `ssl` 异常。

触发链路：

```text
Git HTTPS -> Python ssl -> OpenSSL -> LSX/LASX 向量路径
```

根因：

- 用户态可能使用 LASX 256-bit 向量寄存器。
- 内核没有完整保存恢复 LASX 状态。

修复：

- 开启 `EUEN.ASXE`
- 保存恢复 `xr0..xr31`
- 同步 `AT_HWCAP` 和 `/proc/cpuinfo`
- 添加 `openssl-loongarch` regression

---

## 9. 关键问题：socket QoS

Git SSH / OpenSSH 暴露 socket option 缺口：

- `IP_TOS`
- `IPV6_TCLASS`
- `SO_PRIORITY`
- `IP_RECVTOS`
- `IPV6_RECVTCLASS`

修复方向：

- socket option 分发
- ax-net per-socket QoS 状态
- 出站 IPv4/IPv6 header 写入
- RX metadata
- UDP receive cmsg

---

## 10. 关键问题：recvmsg cmsg

`msg_controllen` 有输入/输出双重语义：

```text
进入 syscall：用户提供的 control buffer 容量
成功返回：实际写入的 cmsg 长度
```

旧实现问题：

- `CMsgBuilder::new()` 过早清零用户态 `msg_controllen`
- `MSG_DONTWAIT` 第一次 `EAGAIN`
- 用户态复用 `msghdr` retry
- 后续 payload 能收到，但 cmsg 丢失

修复：拆分 `capacity` / `written`，成功路径统一 `finish()` 写回。

---

## 11. 达到的效果

已覆盖：

- 3 组 syscall 语义测例
- Git 本地 13 个 probe
- `git://` remote
- HTTPS smart HTTP remote
- Git SSH app
- loongarch64 LASX regression
- socket QoS regression
- `recvmsg` cmsg regression
- AF_UNIX cmsg 回归

---

## 12. 当前边界

当前覆盖的是 Alpine Git 的主要路径：

- 本地工作流
- `git://` remote
- HTTPS smart HTTP remote
- SSH/OpenSSH 暴露的 socket QoS 语义

后续仍可扩展：

- SSH 认证矩阵
- credential helper
- 完整 CA trust store
- LFS / submodule
- 完整 qdisc/device priority

---

## 13. 经验

1. syscall 测例适合建立基础语义判断。
2. 真实应用适合暴露跨模块问题。
3. 复杂应用要分层测，否则失败很难归因。
4. 修通用路径时要补相邻回归。

---

## 14. 总结

这次工作不是实现 Git 本身。

核心价值是：

- 用 syscall 测例打基础；
- 用 Git 牵引真实 Linux app 路径；
- 修复架构状态、网络 socket、cmsg 等实际兼容问题；
- 把问题沉淀成可持续运行的 regression。
