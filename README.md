# mk20-flasher

**Waveshare MK20 ScreenKey 固件 macOS 烧录工具** — 无需 Windows/PhoenixCard。

官方烧录工具 PhoenixCard 仅 Windows,且固件 `ScreenKey-mk20-AI-80M-*.img` 是全志
**IMAGEWTY** 量产容器格式,不能直接 dd/balenaEtcher。本工具链在 Mac 上完成:
解包 → 重组为裸镜像 → balenaEtcher 写卡。

2026-08 实测通过:MK20 (Allwinner T113-S3) + macOS 26 + balenaEtcher 2.1.6。

## 快速开始

```sh
# 1. 获取固件 zip (微雪官方渠道), 解压出 .img
# 2. 解包 (一次编译, 到处可用)
git clone --recurse-submodules https://github.com/YuzukiTsuru/OpenixIMG.git
cmake -B OpenixIMG/build -S OpenixIMG && cmake --build OpenixIMG/build
OpenixIMG/build/app/openiximgcli unpack -i firmware.img -o unpacked-full --format imgrepacker

# 3. 把布局元数据拷进 unpacked/ (本仓库自带官方 20250929 版本的这些小文件)
#    需要从 unpacked-full 再拷 5 个大 fex: boot0_sdcard/boot_package/boot-resource/boot/rootfs

# 4. 重组精简镜像 (1GB, 含全部有效数据)
python3 repack.py --out mk20-slim.img --size-mb 1064

# 5. balenaEtcher 烧入 TF 卡 → 插回 MK20 上电
```

## 固件布局 (实测 ScreenKey-mk20-AI-80M-20250929)

**裸机区(cardscript.fex 指定,不走 GPT):**

| 文件 | 扇区 | 说明 |
|---|---|---|
| boot0_sdcard.fex | LBA 16 | eGON.BT0, BROM 第一级 |
| boot_package.fex | LBA 32800 | sunxi-package (U-Boot + DTB) |

**GPT 分区(sunxi_gpt.fex):**

| 分区 | 起 LBA | 大小 | 内容 |
|---|---|---|---|
| boot-resource | 504 | 32MB | FAT16 配置区 (config.json / mk20-plus.bin / bootlogo) |
| env | 66040 | 258KB | U-Boot env (`mmc_root=/dev/mmcblk0p5`, 板名 t113-pro) |
| env-redund | 66544 | 258KB | 冗余 env |
| boot | 67048 | 6.2MB | ANDROID bootimg → Linux 5.4.61 |
| rootfs | 79648 | 1GB | ext4 出厂 rootfs |
| UDISK | 2176800 | 剩余 | overlayfs upper (设备持久化数据) |

## macOS 写卡的关键坑 (重要)

**macOS 26 (Tahoe) 起 root 也无法 dd raw 设备** — `Operation not permitted`,
SIP + DDM 对外置盘 raw 访问的新限制:

| 通道 | 结果 |
|---|---|
| `sudo dd of=/dev/rdiskN` | ✗ Operation not permitted |
| `asr restore` | ✗ 只认 HFS+ 源 |
| `diskutil eraseDisk` | ✓ 但只能写零/格式化 |
| **balenaEtcher** | ✓ **唯一可行的任意镜像写通道**(自带特权助手) |

速度建议:USB 2.0 读卡器 ≈ 16MB/s。全量 64GB 镜像需 68 分钟 ×2(写+校验);
**精简镜像(1GB)约 2 分钟** — 本工具默认产出精简镜像,UDISK 空区不写,烧完后
`diskutil repairDisk diskN` 把备份 GPT 修到卡真实末尾(可选)。

## 仓库内容

```
├── repack.py        重组脚本: 解包产物 → 裸镜像 (GPT 双 CRC 重算/UDISK 动态扩容)
├── unpacked/        官方 20250929 固件的布局元数据 (GPT/MBR/分区表/cardscript/dlinfo/env)
│                    ⚠ 大体积固件本体 (.img/rootfs.fex 等) 不入库, 见 .gitignore
└── README.md        本文档
```

`unpacked/` 里的元数据文件让 repack.py 开箱即用 — 你只需从官方固件补齐 5 个大 fex。
repack.py 亦可用 OpenixIMG 解包任意新版固件后的产物直接运行。

## 免责声明

仅供设备所有者为自己的 MK20 刷机使用。固件版权归 Waveshare/SYK 所有,
本仓库不分发固件本体。刷机有变砖风险,动卡前先
`hdiutil create -srcdevice /dev/diskN -format UDBZ full-backup.dmg` 整卡备份。
