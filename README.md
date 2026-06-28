# 以 Git 为牵引的 StarryOS Linux 兼容性改进

本仓库用于整理 2026 春季开源操作系统训练营总结报告、博客稿和汇报 slide。

## 内容

- `source/_posts/20260627-Utopia-V-以Git为牵引的StarryOS-Linux兼容性改进.md`
  - 准备提交到 `rcore-os/blog` 的博客稿。
- `slides/starryos-git-compatibility-slides.md`
  - 6 月 28 日晚汇报 slide 大纲。
- `notes/evidence.md`
  - PR、issue、测试和边界范围证据表。

## 核心主线

本次工作先补齐 `eventfd2`、`signalfd4`、`utimensat` 等 syscall 语义测试，再以 Alpine Git 作为真实 Linux app，推动 StarryOS 在文件系统、网络 remote、TLS/OpenSSL、socket QoS 等路径上的兼容性补齐。当前材料聚焦已经验证过的范围：

- `eventfd2` / `signalfd4` / `utimensat` 的源码级 syscall 测例和相关语义修复；
- 本地 Git 主要工作流；
- `git://` remote 的核心 `ls-remote` / `clone` / `fetch` / `pull` / `push`；
- HTTPS smart HTTP remote 的核心 `ls-remote` / `clone` / `fetch` / `pull` / `push`；
- SSH remote 场景中 OpenSSH/Git 暴露出的 socket QoS option 兼容问题；
- x86_64、aarch64、riscv64、loongarch64 QEMU 上的回归验证。

## 主要证据

- 跟踪 issue：<https://github.com/rcore-os/tgoskits/issues/579>
- `eventfd2` syscall 测例：<https://github.com/rcore-os/tgoskits/pull/670>
- `signalfd4` syscall 测例与信号信息修复：<https://github.com/rcore-os/tgoskits/pull/683>
- `utimensat` syscall 测例与语义修复：<https://github.com/rcore-os/tgoskits/pull/763>
- Git 本地：<https://github.com/rcore-os/tgoskits/pull/1026>
- Git `git://` remote：<https://github.com/rcore-os/tgoskits/pull/1169>
- Git HTTPS / loongarch64 LASX：<https://github.com/rcore-os/tgoskits/pull/1178>
- Git SSH / socket QoS：<https://github.com/rcore-os/tgoskits/pull/1319>
- 方案三 RGA 探索：<https://github.com/rcore-os/tgoskits/pull/1248>
