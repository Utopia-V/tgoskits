# 以 Git 为牵引的 StarryOS Linux 兼容性改进

Utopia-V  
2026 春季开源操作系统训练营

---

## 1. 目标

以 Alpine Git 作为真实 Linux app，推动 StarryOS Linux 兼容性改进。

重点不是实现 Git 本身，而是验证 Git 运行依赖的系统语义：

- syscall 行为
- 文件系统 rename/ref 更新
- 进程与 shell 环境
- socket / TCP loopback
- TLS / OpenSSL
- socket QoS option

---

## 2. 为什么选择 Git

Git 是一个足够复杂但可确定化测试的 Linux app：

- 本地操作会频繁创建、重命名、删除文件
- branch/ref/reflog 会压测 VFS rename 语义
- remote 操作会经过 socket、TCP、pack/ref 传输
- HTTPS 会引入 OpenSSL / Python ssl 路径
- SSH 会引入 OpenSSH 的 socket option 行为

---

## 3. 总体路线

```text
syscall 基础语义
        ↓
Git 本地工作流
        ↓
git:// remote
        ↓
HTTPS smart HTTP
        ↓
SSH / OpenSSH 暴露的 socket QoS 语义
```

每一步只推进一个确定路径：先跑通，再把失败缩小成具体内核语义，最后补 regression。

---

## 4. 贡献概览

| 方向 | PR |
| --- | --- |
| syscall 测例 | #670, #683, #763 |
| Git 本地 | #1025, #1026 |
| Git `git://` remote | #1169 |
| Git HTTPS / LASX | #1178 |
| Git SSH / QoS | #1319 |
| 方案三探索 | #1248 |

---

## 5. 方案一：syscall 基础语义

| PR | syscall | 重点 |
| --- | --- | --- |
| #670 | `eventfd2` | flag、计数器/信号量、非阻塞、溢出、fork 继承 |
| #683 | `signalfd4` | 信号接收、mask、`ssi_pid` / `ssi_uid` |
| #763 | `utimensat` | 时间戳更新、flag 校验、`AT_EMPTY_PATH` |

这部分的价值：

- 直接补 StarryOS 的 syscall 兼容测试面。
- 形成“Linux 语义 -> 源码级测例 -> 内核修复 -> regression”的工作方法。
- 为后续定位 Git 暴露的问题打基础。

---

## 6. Git 覆盖矩阵

| 层次 | 覆盖操作 | 主要验证点 |
| --- | --- | --- |
| 本地 Git | 13 个 probe | 文件系统、ref/reflog、rename |
| `git://` | `clone/fetch/pull/push` | socket、TCP loopback、pack/ref |
| HTTPS | `clone/fetch/pull/push` | TLS、OpenSSL、Python `ssl` |
| SSH | `clone/fetch/pull/push` | OpenSSH socket option、QoS cmsg |

这不是用一个命令证明“Git 已支持”，而是把 Git 拆成几层可复现路径。

---

## 7. Git 本地工作流

新增 `stress/git`：

- `init`
- `config`
- `add`
- `commit`
- `log`
- `status`
- `diff`
- `branch`
- `checkout`
- `reset`
- `merge`
- `stash`
- `tag`

四架构 QEMU 覆盖，使用完成标记和失败正则避免误报。

---

## 8. rename 语义问题

Git ref 操作会触发 rename 路径。

暴露的问题：

- VFS 的祖先检查本意是禁止目录移入自身子树。
- 旧逻辑过宽，普通文件 rename 到子目录也会被拒绝。

准确表述：

- 相关 VFS 修复由 #807 合入。
- 我的工作在 #1025/#1026 补充 Git rename 回归覆盖，使 Git 路径不再只停留在手工复现。

---

## 9. `git://` remote

在 guest 内启动 `git daemon`：

```text
git://127.0.0.1:9418/src.git
```

覆盖：

- `git ls-remote`
- `git clone`
- clone 到已有空目录
- 关闭端口失败路径
- `git fetch`
- `git pull --ff-only`
- `git push`

验证 TCP loopback、socket、Git client/server、pack/ref 更新。

---

## 10. HTTPS remote

在 guest 内启动本地 HTTPS smart Git 服务：

- Python `ThreadingHTTPServer`
- `ssl`
- `git http-backend`
- 自签名证书
- `GIT_SSL_NO_VERIFY=true`

覆盖 `ls-remote`、`clone`、`fetch`、`pull`、`push`。

不测公网 CA，也不依赖外部 writable remote。

---

## 11. LASX bug

HTTPS 路径在 loongarch64 上触发 OpenSSL / Python `ssl` 异常。

触发链路：

```text
Git HTTPS -> Python ssl -> OpenSSL -> LASX 向量路径
```

真正的问题：

- 用户态可能使用 LSX/LASX 向量能力。
- 内核暴露了能力，但没有完整保存恢复 LASX 256-bit 状态。

修复：

- 启用 `EUEN.ASXE`
- 保存恢复 `xr0..xr31`
- 同步 `AT_HWCAP` 和 `/proc/cpuinfo`
- 添加 `openssl-loongarch` regression

---

## 12. SSH / QoS bug

OpenSSH client 会设置 `IP_TOS`。

暴露的问题：

- Starry 没有完整处理 `IP_TOS` / `IPV6_TCLASS` / `SO_PRIORITY`
- 出站 TOS 没有进入 IP header
- 收包侧 `IP_RECVTOS` / `IPV6_RECVTCLASS` 不会返回 cmsg

#1319 补齐 socket option、ax-net 状态、出站 header 写入和 UDP receive cmsg。

---

## 13. `recvmsg` cmsg 细节

review 后发现的细节 bug：

- `msg_controllen` 进入 syscall 时表示用户 buffer 容量。
- 成功返回时才应该写回实际 cmsg 长度。
- 原实现 `CMsgBuilder::new()` 过早清零。
- `MSG_DONTWAIT` 第一次 `EAGAIN` 后，用户态复用 `msghdr` retry，容量变 0，后续 cmsg 丢失。

修复：

```text
capacity = 用户传入的 buffer 容量
written  = 本次成功写入的 cmsg 长度
```

成功路径统一 `finish()` 写回，失败路径不破坏用户态 `msghdr`。

---

## 14. 验证体系

测试不是只跑 `git --version`：

- Git 本地 13 个 probe
- `git://` remote stress
- HTTPS remote stress
- Git SSH app
- `openssl-loongarch`
- `bugfix-bug-socket-qos-options`
- `bugfix-bug-recv-qos-cmsg`
- AF_UNIX cmsg 回归

目标是“真实应用失败 -> 缩小根因 -> 内核修复 -> regression”。

---

## 15. 当前边界

已覆盖：

- 本地 Git 主要工作流
- `git://` remote 核心操作
- HTTPS smart HTTP remote 核心操作
- Git SSH 暴露的 socket QoS 兼容语义

未覆盖：

- SSH 认证矩阵
- credential helper
- 完整 CA/TLS 边界
- 外部 writable remote
- LFS / submodule

---

## 16. 总结

这条线的价值：

- 用真实 Linux app 牵引 StarryOS 兼容性改进。
- 通过 Git 路径修复了架构状态和网络 socket 语义问题。
- 每个修复都有 regression。
- 明确当前覆盖范围，也保留后续扩展边界。

后续方向：

- Git 方向后续可以继续补 SSH 认证、credential helper、LFS/submodule 等场景。
- 方案三继续按“真实路径 -> 最小闭环 -> 回归验证”推进。
