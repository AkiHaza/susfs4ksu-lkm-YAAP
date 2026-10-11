# susfs4ksu-lkm

SUSFS 的**可加载内核模块（LKM）移植版**：让 susfs 可以在不用重编译内核的情况下运行 。

## YAAP / OnePlus 12 构建

YAAP `seventeen` 使用定制的 6.1 内核，不能仅凭 `android14-6.1` 标签选择通用 DDK 产物。
本仓库新增手动触发的 **Build SUSFS for YAAP** 工作流
（`.github/workflows/build-yaap.yml`），参考 `LKM-PathMask-YAAP` 的构建方式：

- 使用 `AkiHaza/android_kernel_oneplus_sm8650` 和配套 `-modules` 仓库的 `seventeen` 分支。
- 使用 clang-r596125，合并 `gki_defconfig` 与两个厂商 `pineapple_GKI.config`，保留正常的 CFI/LTO 配置。
- 完整构建内核，使用同次构建的 `Module.symvers`、`System.map` 和生成头文件编译 SUSFS。
- 校验必需配置、引用及运行时 hook 符号、CFI 调用点、vermagic、真实符号 CRC 和命名空间；strip 前后均检查模块元数据。

在 GitHub Actions 选择该工作流并点击 **Run workflow**。默认只上传 artifact；
只有勾选 `publish_release` 才会发布 Release。产物包含 `susfs_guard_lkm.ko`、
加载器及诊断工具、接口文档、完整配置、源码提交记录和 SHA256 校验值，也提供 `.tar.gz`。
这不是可直接刷入 KernelSU 管理器的模块 ZIP。

模块仅适用于产物 `README.txt` 记录的内核、厂商模块提交及配置。
分支更新后应重新构建；修改 vermagic 不能证明 ABI 兼容。
构建检查不等于设备加载测试，仍须在匹配的设备上验证。

SUSFS 与 PathMask 的区别是它直接引用部分**未导出的内核符号**，因此构建时使用
`KBUILD_MODPOST_WARN=1`，随后强制检查所有引用是否存在于同次构建的 `System.map`。
保留真实 `__versions` 和 `module_layout` CRC，不使用清空版本段或伪造符号表的补丁。
加载仍需下文的专用加载器；兼容的 `ksud insmod` 也可使用。

本地 Linux / WSL 构建时，先按工作流生成匹配的内核输出目录，再执行：

```sh
make -C /path/to/yaap-kernel O=/path/to/yaap-out \
  ARCH=arm64 LLVM=1 LLVM_IAS=1 M=/path/to/susfs4ksu-lkm-YAAP/kernel \
  KBUILD_MODPOST_WARN=1 modules
```

只有 `modules_prepare` 不会产生 `CONFIG_MODVERSIONS=y` 所需的完整符号 CRC。
发布本地产物前，应执行工作流中的符号、CFI 和元数据检查。

## 加载

`.ko` 要选与设备内核对应的那一份，例如 5.15 设备用 `susfs_guard_lkm-android13-5.15.ko`。

**KernelSU 设备**：

```sh
ksud insmod /data/adb/loader/susfs_guard_lkm.ko
```

**非 KernelSU 设备**：用 release 附带的加载器 `susfs_insmod`：

```sh
susfs_insmod /data/adb/loader/susfs_guard_lkm.ko
```

卸载：

```sh
rmmod susfs_guard_lkm
```

## 接口文档

全部 `/proc` 控制节点，见 **[PROC_INTERFACES.md](PROC_INTERFACES.md)**。

## License

GPL-2.0，见 [LICENSE](LICENSE)。移植来源：[susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu)（功能逻辑与 hook 点）、
[SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra)（`patch_memory` / `lsm_hook` / `symbol_resolver`）。

## 免责声明

仅用于技术研究与学习。在自担风险的前提下使用。
