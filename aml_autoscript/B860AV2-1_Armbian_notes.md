# B860AV2-1（Meson8m2 / n200C）跑 Armbian 笔记

## 硬件

- SoC：Amlogic Meson8m2（S812 同系，32 位 4 核 Cortex-A9），1GB DDR，4GB eMMC
- WiFi：RTL8189ES（SDIO，`024c:8179`），主线内核没有驱动，需自己编
- 有线网：内置 MAC。蓝牙：没有内置设备
- 原厂 U-Boot 的设备号：**mmc 0 = SD 卡，mmc 1 = eMMC**
- 原厂 Android 4.4，内核 3.10.33，串口 115200

## 备份（eMMC 分区）

```sh
dd if=/dev/block/<名字> of=<目录>/<名字>.img bs=1M
# 名字：bootloader reserved env conf logo recovery misc boot system
# 另有：mmcblk1boot0 mmcblk1boot1
```

预期大小（字节）：

| 文件 | 大小 |
|---|---|
| bootloader / env / conf / misc / mmcblk1boot0 / mmcblk1boot1 | 4194304 |
| reserved / logo | 16777216 |
| recovery / boot | 20971520 |
| system | 805306368 |

`env.img` 和 boot0/boot1 最重要，丢了无法重建。

## SD 卡启动用的文件（Armbian 6.12.28）

卡里 FAT 分区：

- `uImage`、`uInitrd`
- `dtb/meson8m2-b860a.dtb`
- `armbianEnv.txt`
- `s905_autoscript`、`s805_autoscript`、`aml_autoscript`
这几个都要有，cp s805_autoscript s905_autoscript
## U-Boot 手动启动

串口 115200，一通电就连按回车（`bootdelay=1`，只有 1 秒）。

```
mmcinfo
fatls mmc 0
run start_mmc_autoscript
```

等价于：

```
fatload mmc 0 1020000 s905_autoscript
autoscr 1020000
```

## 自动启动（写入 env 分区，会写 eMMC）

中兴代码开机会把 `preboot` 改回去，`bootcmd` 也不起作用，所以挂在 `update_key` 上。

```
printenv update_key        # 先记下原值
setenv update_key 'saradc open 0; if saradc get_in_range 0 0x50; then msleep 50; if saradc get_in_range 0 0x50; then echo update by key...; run update; fi;fi; if mmcinfo; then run start_mmc_autoscript; fi;'
run update_key             # 先在内存里测试
saveenv
reset
```

原值：

```
saradc open 0; if saradc get_in_range 0 0x50; then msleep 50; if saradc get_in_range 0 0x50; then echo update by key...; run update; fi;fi
```

- 插卡进 Armbian，拔卡回 Android。
- 状态：`run update_key` 在内存里测试成功。`saveenv` 之后断电自启是否生效、`update_key` 会不会被中兴代码改回去，**还没验证**。

## 恢复

- 把 `update_key` 改回原值再 `saveenv`
- 或把备份写回去：`dd if=env.img of=/dev/block/env bs=1M`
- 拔掉 SD 卡，原厂 Android 就回来了

## 要避开的坑

- `aml_autoscript` 里有 `saveenv`，执行时会写 eMMC。以后任何 autoscript 先用 `strings` 读一遍，看到 `saveenv`、`store`、`mmc write`、`nand` 就不要用
- 没有卡槽时用 U 盘，`start_usb_autoscript` 会查 usb 0~3
- 复位键走 `update` 流程，会执行卡里的 `aml_autoscript`
- 不要运行 `armbian-install`，它会覆盖 eMMC
- `mmcinfo` 要先于 `fatload`，否则报 `Can't read from device 0`
- 关机用 `poweroff` 或断电，不要用待机，`reboot_mode=suspend_off` 会让 preboot 直接 suspend
- 不要改 `bootcmd`、`bootdelay`、`storeboot`

## 待办

1. 确认 `saveenv` 后断电重启能否自启，并用 `printenv update_key` 确认没被改回
2. Armbian 里检查有线网、WiFi：
   ```sh
   ip -br a
   dmesg | grep -iE "eth|sdio|mmc|8189|rtl"
   ls /sys/bus/sdio/devices/
   lsmod
   ```
3. RTL8189ES：用 DKMS 编 `rtl8189ES_linux`，必要时对照原厂 `dtb6.dts` 补 SDIO 节点
4. Armbian 首次进入后设置 root 密码：`passwd`
