# gaokun3 双仓库迁移记录

这是迁移候选，不能作为已验证发行版发布。两个目标仓库已建立，内核 `gaokun3-next` 分支已发布下表的 Iris/GSI 候选精确提交，buildbot 迁移位于 `next`。原仓库 PR #7 保留迁移评审记录。

## 仓库边界

- `gaokun3/linux`：从 `gregkh/linux` fork，`gaokun3` 维护设备提交，`gaokun3-next` 审查 Iris/GSI 候选；DTS、驱动和 `gaokun3_defconfig` 都在内核树内。
- `gaokun3/buildbot`：Bash 构建入口、打包、镜像、固件及用户空间工具。GitHub Actions 负责调度。镜像组装仍包含工作流内联步骤，尚未全部抽成脚本。
- 移除 buildbot 的 `patches/`、`drivers/`、`dts/`、`defconfig/`。原件保留在迁移前提交 `315528c028843794ccd0f3d9dab373b03033150f`，不在构建时再次应用。

## 固定输入

`build.env` 是本地与 CI 共用的版本来源：

| 输入 | 当前候选 |
| --- | --- |
| 上游分支 | gregkh/linux `linux-rolling-stable` |
| 上游提交 | `d396b05e7e39b0ed6f6d5553fbaf174228e18bdf`，Merge v7.2.6 |
| 下游提交 | `3a9f5e6c574c096d12558093167a7ca0f85b431b` |
| Fedora / Ubuntu | 44 / 26.04 |
| EL2 | 未设置提交；显式请求会报错 |

`KERNEL_TAG` 目前只是产物命名标签，并不代表 GitHub 已存在该 tag；真正检出依据为 `KERNEL_COMMIT`。内核精确提交已推送；正式发布前仍需建立不可变 tag，并保留上游 base SHA。Ubuntu rootfs 下载失败直接停止，不再回退 beta。

软件包清单记录内核 SHA 与 buildbot SHA；镜像组装核对内核 SHA，防止复用不同源码生成的软件包。

## 本地构建

安装内核构建依赖（Git、make、GCC、bc、bison、flex、OpenSSL/libelf 开发包、pahole、rsync、kmod）；x86 主机还需 AArch64 交叉工具链。DEB 打包需 dpkg-dev，RPM 打包需 rpmbuild 及相应发行版工具。打包和镜像完整流程以原生 arm64 CI 为验证目标。

```bash
./build.sh kernel
./build.sh debs
./build.sh rpms
# 也可使用提交匹配且干净的本地内核仓库：
KERN_SRC=/absolute/path/to/linux ./build.sh kernel
```

已有源码目录必须与固定 SHA 一致且干净；脚本不会 reset、覆盖或打补丁。`WORKDIR` 可更改输出目录，`JOBS` 可限制并发。`scripts/local/build_kernel.sh` 保留设备上的交互式构建安装入口，也使用相同固定源码。

## 迁移审查

- 导入原补丁作者信息。PDC 映射补丁通过反向应用确认已在基线中，单独移除；不以“冲突”判定补丁已上游。
- EC 设备树保留上游 GPIO 103 修正，对应 PDC 215。
- 根据原重启规划纠正视频路线：next 去除旧 Venus 系列，移植上游 Iris DTS 并启用 stable Iris 驱动；解码及编码均未实测。补入 right-0903 force-GSI 实现，触摸算法替换仍待审查。详见 [内核审计](kernel-audit.md)。
- `CONFIG_INPUT_UINPUT=m` 已在配置中；PR #2 的用户空间部分未在本轮引入。
- `9420138` 删除的旧 UCSI、q6apm 改动与新基线冲突，尚待语义审查；不能宣称已上游或功能等价。
- EL2 仅有部分移植工作：remoteproc 异步 attach 与 q6v5 running 状态变更需要继续审查，不能发布。
- systemd-boot 使用 `fedora` / `ubuntu` entry token。旧 machine-id 条目保留作回退；详见 [启动布局](boot-layout.md)。此轮参考 PeronGH 的方向，未整体引入其发行版策略。

## 已验证与发布门槛

已验证：gaokun3 defconfig 生成、内核 Kbuild 设备树编译，以及上一版 Himax 触摸、EC、电池驱动对象交叉编译；当前 Iris 全目录对象与 SPI GENI 对象交叉编译通过；systemd 255 的实际 kernel-install / BLS 插件测试通过。

尚未完成：完整 Image/modules 链接、DEB/RPM 打包、Fedora/Ubuntu 镜像构建及设备启动。还需检查 Fedora SELinux 与当前 AppArmor 配置的兼容性，并实测触摸、60/120 Hz、音频、无线、蓝牙、充电、USB-C、休眠唤醒、视频解码、升级和回退。

先完成上述检查，再合并迁移 PR、发布 release。EL2 单独推进，不作为普通内核已完成的功能。

## 构建验证

`Validate kernel packages` 在 `next` 的构建相关改动后运行，也可手动触发。它复用现有 DEB/RPM 工作流，在原生 ARM64 runner 上编译完整 Image、modules 和 DTB，然后打包。包及源码清单保存在 Actions artifacts 中 7 天。

两个打包工作流新增 `publish_release`，默认 `false`。仅显式开启时才创建 GitHub release；镜像发布流程需要下载软件包，因此明确传入 `true`。构建成功不代表设备启动和升级测试通过。
