---
title: imx6ull 移植 2023 uboot
date: 2025-10-23
authors:
  - name: lunuj
    link: https://github.com/lunuj
tags:
  - 2025
  - uboot
---
在正点原子阿尔法开发板上移植 **NXP** 官方 2023 年的 **uboot**。
<!--more-->

## 1. 序言
这次记录 **imx6ull** 的 **uboot** 移植过程，将 **NXP** 官方发布的 **uboot** 移植到正点原子 imx6ull-**alpha** 开发板上。
正点原子教程中的 **uboot** 版本存在以下问题：
1. 不支持设备树，对以后学习移植帮助不太。
2. 仅支持 **nfs2** ，**WSL** 环境下的 linux 版本不支持 nfs2，无法挂载文件系统。
要使用新版的 **Linux** 版本进行开发，就需要移植较新版本的 **uboot**。

## 2. 开发环境
1. windows11_x86 下的 **WSL2** 虚拟机，后续也可能在 macos_aarch64 平台上开发。
2. uboot 使用的是 [NXP官方仓库](https://github.com/nxp-imx)的 **imx-uboot**，版本是 2023.04，[直达链接](https://github.com/nxp-imx/uboot-imx)。
3. 编译器使用的是 [linaro](https://snapshots.linaro.org)提供的 13.0-2022.11-1 版本的 [gcc]([Linaro Snapshots](https://snapshots.linaro.org/gnu-toolchain/13.0-2022.11-1/))。
4. IDE 使用的是 VSCode + Remote-SSH。

## 3. 编译官方 uboot
下载官方 **uboot** 版本：
```shell
git clone https://github.com/nxp-imx/uboot-imx.git --single -branch lf_v2023.04 uboot-2023
```
编译官方 **uboot** 版本：
```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_evk_emmc_defconfig
make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j32
```
编译过程可能提示各种缺失库，安装后即可编译成功。
编译完成后会在目录下得到一下目标文件：
- uboot：包含debug信息的 **elf** 文件。
- uboot.bin：去除调试信息和符号表的二进制文件。
- uboot.dtb：描述硬件信息的设备树文件。
- uboot-dtb.bin：包含 **uboot** 和设备树的二进制文件。
- uboot-dtb.imx：在 uboot-dtb.bin 添加 **NXP IMX** 系列处理头部信息的二进制文件。
VSCode 下可以通过 setting.json 屏蔽无关文件，放入 .vscode 文件夹：
```json
{
    "search.exclude": {
        "**/node_modules": true,
        "**/bower_components":true,
        "**/*.o":true,
        "**/*.su":true,
        "**/*.cmd":true,
        "Documentation":false,
        ".vscode/*":true,
        ".github/*":true,
         /* 屏蔽不用的架构相关的文件 */
        "arch/alpha":true,
        "arch/arc":true,
        "arch/arm64":true,
        "arch/avr32":true,
        "arch/[b-z]*":true,
        "arch/arm/plat*":true, 
        "arch/arm/mach-[a-h]*":true, 
        "arch/arm/mach-[n-z]*":true, 
        "arch/arm/mach-i[n-z]*":true,
        "arch/arm/mach-imx/i*":true,
        "arch/arm/mach-m[e-v]*":true,
        "arch/arm/mach-k*":true,
        "arch/arm/mach-l*":true,
        /* 屏蔽排除不用的配置文件 */
        "arch/arm/configs/[a-h]*":true,
        "arch/arm/configs/[j-z]*":true,
        "arch/arm/configs/imo*":true,
        "arch/arm/configs/in*":true,
        "arch/arm/configs/io*":true,
        "arch/arm/configs/ix*":true,

         /* 屏蔽掉不用的 DTB 文件 */
        "arch/arm/boot/dts/[a-h]*":true,
        "arch/arm/boot/dts/[k-z]*":true,
        "arch/arm/boot/dts/in*":true,
        "arch/arm/boot/dts/imx1*":true,
        "arch/arm/oot/dts/imx7*":true,
        "arch/arm/boot/dts/imx2*":true,
        "arch/arm/boot/dts/imx3*":true,
        "arch/arm/boot/dts/imx5*":true,
        "arch/arm/boot/dts/imx6d*":true,
        "arch/arm/boot/dts/imx6q*":true,
        "arch/arm/boot/dts/imx6s*":true,
        "arch/arm/boot/dts/imx6ul-*":false,
        "arch/arm/boot/dts/imx6ull-9x9*":true,
        "arch/arm/boot/dts/imx6ull-14x14-ddr*":true, 

        /*屏蔽不相关开发板*/
        "board/[a-e]*":true,
        "board/[g-z]*":true,
        "board/[A-Z]*":true,
        "board/freescale/[a-l]*":true,
        "board/freescale/[n-z]*":true,
        "board/freescale/m5*":true,
        "board/freescale/mx6s*":true,
        "board/freescale/mx7*":true,
        "board/freescale/mx[1-5]*":true,

        /*屏蔽不用的芯片配置文件*/
        "configs/[a-l]*":true,
        "configs/[n-z]*":true,
        "configs/[A-Z]*":true,
        "configs/m[a-w]*":true,
        "configs/mx[^6]*":true,
        "configs/mx6[^u]*":true,
        "configs/mx6ulz*":true,
        "configs/mx6ul_*":true,
        /*屏蔽不用的设备树文件*/
        "arch/arm/dts/[0-9]*":true,
        "arch/arm/dts/[^i]*":true,
        "arch/arm/dts/im[^x]*":true,
        "arch/arm/dts/imx[^6]*":true,
        "arch/arm/dts/imx6[^u]*":true,
        "arch/arm/dts/imx6ull-[^h]*":true,
        "arch/arm/dts/imx6ul-[a-z]*":true,
    },

    "files.exclude": {
        "**/.git": true,
        "**/.svn": true,
        "**/.hg": true,
        "**/CVS": true,
        "**/.DS_Store": true, 
        "**/*.o":true,
        "**/*.su":true,
        "**/*.cmd":true,
        "Documentation":false,
        ".vscode":true,
        ".github":true,
        /* 屏蔽不用的架构相关的文件 */
        "arch/alpha":true,
        "arch/arc":true,
        "arch/arm64":true,
        "arch/avr32":true,
        "arch/[b-z]*":true,
        "arch/arm/plat*":true, 
        "arch/arm/mach-[a-h]*":true, 
        "arch/arm/mach-[n-z]*":true, 
        "arch/arm/mach-i[n-z]*":true,
        "arch/arm/mach-m[e-v]*":true,
        "arch/arm/mach-imx/i*":true,
        "arch/arm/mach-k*":true,
        "arch/arm/mach-l*":true,
        /* 屏蔽排除不用的配置文件 */
        "arch/arm/configs/[^i]*":true,
        "arch/arm/configs/imo*":true,
        "arch/arm/configs/in*":true,
        "arch/arm/configs/io*":true,
        "arch/arm/configs/ix*":true,
        /* 屏蔽掉不用的 DTB 文件 */
        "arch/arm/boot/dts/[a-h]*":true,
        "arch/arm/boot/dts/[k-r]*":true,
        "arch/arm/boot/dts/s[0-9]*":true,
        "arch/arm/boot/dts/s[a-i]*":true,
        "arch/arm/boot/dts/s[l-z]*":true,
        "arch/arm/boot/dts/[t-z]*":true,
        "arch/arm/boot/dts/in*":true,
        "arch/arm/boot/dts/imx1*":true,
        "arch/arm/boot/dts/imx7*":true,
        "arch/arm/boot/dts/imx2*":true,
        "arch/arm/boot/dts/imx3*":true,
        "arch/arm/boot/dts/imx5*":true,
        "arch/arm/boot/dts/imx6d*":true,
        "arch/arm/boot/dts/imx6q*":true,
        "arch/arm/boot/dts/imx6s*":true,
        "arch/arm/boot/dts/imx6ul-*":false,
        "arch/arm/boot/dts/imx6ull-9x9*":true,
        "arch/arm/boot/dts/imx6ull-14x14-ddr*":true,
        /*屏蔽不相关开发板*/
        "board/[a-e]*":true,
        "board/[g-z]*":true,
        "board/[A-Z]*":true,
        "board/freescale/[a-l]*":true,
        "board/freescale/[n-z]*":true,
        "board/freescale/m5*":true,
        "board/freescale/mx6s*":true,
        "board/freescale/mx7*":true,
        "board/freescale/mx[1-5]*":true,
        /*屏蔽不用的芯片配置文件*/
        "configs/[a-l]*":true,
        "configs/[n-z]*":true,
        "configs/[A-Z]*":true,
        "configs/m[a-w]*":true,
        "configs/mx[^6]*":true,
        "configs/mx6[^u]*":true,
        "configs/mx6ulz*":true,
        "configs/mx6ul_*":true,
        /*屏蔽不用的芯片头文件*/
        "include/configs/[0-9]*":true,
        "include/configs/[^m]*.h":true,
        "include/configs/m[^x]*.h":true,
        "include/configs/mx[^6]*.h":true,
        "include/configs/mx6[^u]*.h":true,
        /*屏蔽不用的设备树文件*/
        "arch/arm/dts/[0-9]*":true,
        "arch/arm/dts/[^i]*":true,
        "arch/arm/dts/im[^x]*":true,
        "arch/arm/dts/imx[^6]*":true,
        "arch/arm/dts/imx6[^u]*":true,
        "arch/arm/dts/imx6ull-[^h]*":false,
        "arch/arm/dts/imx6ul-[a-t]*":true,
        },
        "files.associations": {
            "panel.h": "c",
            "video.h": "c",
            "mxsfb.h": "c",
            "fb.h": "c",
            "device-internal.h": "c"
        }
}
```

## 4. 更改 uboot 配置
官方的 **uboot** 编译成功后，要对正点原子的开发板进行适配。
### 修改默认配置文件
复制 defconfig 配置文件：
```shell
cp configs/mx6ull_14x14_evk_emmc_defconfig configs/mx6ull_lunuj_defconfig
```
修改一下内容：
```c
# CONFIG_TARGET_MX6ULL_14X14_EVK=y
CONFIG_TARGET_MX6ULL_LUNUJ=y
```
### 修改板级配置文件
复制 board 目录和头文件：
```shell
cp board/freescale/mx6ullevk/ -r board/freescale/mx6ull_lunuj
cp include/configs/mx6ullevk.h include/configs/mx6ull_lunuj.h
```
mx6ullev 文件夹下是一些板级的配置函数，修改 c 文件名称：
```shell
mv mx6ullevk.c mx6ull_lunuj.c
```
在 c 文件中，修改以下两个函数：
```c
static int setup_lcd(void)
{
	enable_lcdif_clock(LCDIF1_BASE_ADDR, 1);

	imx_iomux_v3_setup_multiple_pads(lcd_pads, ARRAY_SIZE(lcd_pads));
#if 0 // 正点原子的开发板中屏幕不需要复位
	/* Reset the LCD */
	gpio_request(IMX_GPIO_NR(5, 9), "lcd reset");
	gpio_direction_output(IMX_GPIO_NR(5, 9) , 0);
	udelay(500);
	gpio_direction_output(IMX_GPIO_NR(5, 9) , 1);
#endif
	/* Set Brightness to high */
	gpio_request(IMX_GPIO_NR(1, 8), "backlight");
	gpio_direction_output(IMX_GPIO_NR(1, 8) , 1);

	return 0;
}
```
```c
int checkboard(void)
{
	if (is_mx6ull_9x9_evk())
		puts("Board: MX6ULL 9x9 EVK\n");
	else if (is_cpu_type(MXC_CPU_MX6ULZ))
		puts("Board: MX6ULZ 14x14 EVK\n");
	else
		puts("Board: MX6ULL LUNUJ\n"); # 需改为开发板名称
	return 0;
}
```
修改 Makefile：
```Makefile
obj-y  := mx6ull_lunuj.o
```
修改 Kconfig：
```
if TARGET_MX6ULL_LUNUJ

config SYS_BOARD
	default "mx6ull_lunuj"

config SYS_VENDOR
	default "freescale"

config SYS_CONFIG_NAME
	default "mx6ull_lunuj"

config IMX_CONFIG
	default "board/freescale/mx6ull_lunuj/imximage.cfg"

config TEXT_BASE
	default 0x87800000
endif
```
修改 imximage.cfg：
```c
#ifdef CONFIG_USE_IMXIMG_PLUGIN
/*PLUGIN    plugin-binary-file    IRAM_FREE_START_ADDR*/
PLUGIN	board/freescale/mx6ull_lunuj/plugin.bin 0x00907000
#else
```
MAINTAINERS 文件存放文件对应维护者，imximage_lpddr2.cfg 是 lpddr2 的初始化头部信息可以删去。mx6ull_lunuj.h 默认的就是14x14_evk和alpha开发板上的芯片一致，ddr默认512MB，默认 EMMC 没有必要修改。其他版本如NAND版则要根据自己的开发板型号具体修改。
### 修改图新配置文件
在 arch/arm/mach-imx/mx6/Kconfig 文件中增加开发板：
```c
config TARGET_MX6ULL_14X14_EVK
	bool "Support mx6ull_14x14_evk"
	depends on MX6ULL
	select BOARD_LATE_INIT
	select DM
	select DM_THERMAL
	select IOMUX_LPSR
	select IMX_MODULE_FUSE
	select OF_SYSTEM_SETUP
	imply CMD_DM
# 仿照上文增加以下内容
config TARGET_MX6ULL_LUNUJ
	bool "Support lunuj"
	depends on MX6ULL
	select BOARD_LATE_INIT
	select DM
	select DM_THERMAL
	select IOMUX_LPSR
	select IMX_MODULE_FUSE
	select OF_SYSTEM_SETUP
	imply CMD_DM
......
source "board/freescale/mx6ullevk/Kconfig"
# 仿照上文增加以下内容
source "board/freescale/mx6ull_lunuj/Kconfig"
```

## 5. 修改 uboot 设备树
### 复制官方设备树
**NXP** 官方使用的默认设备树文件是 arch/arm/dts/imx6ull-14x14-evk.dts：
```dts
#include "imx6ull.dtsi"
#include "imx6ul-14x14-evk.dtsi"
#include "imx6ul-14x14-evk-u-boot.dtsi"

/ {
	model = "i.MX6 ULL 14x14 EVK Board";
	compatible = "fsl,imx6ull-14x14-evk", "fsl,imx6ull";
};

&clks {
	assigned-clocks = <&clks IMX6UL_CLK_PLL3_PFD2>,
			  <&clks IMX6UL_CLK_PLL4_AUDIO_DIV>;
	assigned-clock-rates = <320000000>, <786432000>;
};

&csi {
	status = "okay";
};

&ov5640 {
	status = "okay";
};

/delete-node/ &sim2;
```
其中 imx6ull.dtsi 存放了代表imx6ull共性的节点，如cpu，usdhc存储接口等，可以直接引用；imx6ul-14x14-evk.dtsi 存放了代表evk开发板特性的节点信息，需要复制一份；imx6ul-14x14-evk-u-boot.dtsi 存放了代表dm总线相关的配置信息，是u-boot特性节点，也需要复制一份。
```shell
cp arch/arm/dts/imx6ull-14x14-evk.dts arch/arm/dts/imx6ull-lunuj.dts
cp arch/arm/dts/imx6ul-14x14-evk-u-boot.dtsi arch/arm/dts/imx6ul-lunuj-u-boot.dtsi 
```
对于 EMMC 版本需要从 arch/arm/dts/imx6ull-14x14-evk-emmc.dts 中复制emmc的存储节点信息：
```dts
&usdhc2 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc2_8bit>;
	pinctrl-1 = <&pinctrl_usdhc2_8bit_100mhz>;
	pinctrl-2 = <&pinctrl_usdhc2_8bit_200mhz>;
	bus-width = <8>;
	non-removable;
	status = "okay";
};

```
最后修改如下:
```dts
#include "imx6ull.dtsi"
#include "imx6ul-lunuj.dtsi"
#include "imx6ul-lunuj-u-boot.dtsi"

/ {
	model = "i.MX6 ULL LUNUJ Board";
	compatible = "fsl,imx6ull-14x14-evk", "fsl,imx6ull";
};

&clks {
	assigned-clocks = <&clks IMX6UL_CLK_PLL3_PFD2>,
			  <&clks IMX6UL_CLK_PLL4_AUDIO_DIV>;
	assigned-clock-rates = <320000000>, <786432000>;
};

&csi {
	status = "okay";
};

&ov5640 {
	status = "okay";
};
// EMMC 版本需要添加
&usdhc2 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc2_8bit>;
	pinctrl-1 = <&pinctrl_usdhc2_8bit_100mhz>;
	pinctrl-2 = <&pinctrl_usdhc2_8bit_200mhz>;
	bus-width = <8>;
	non-removable;
	status = "okay";
};
/delete-node/ &sim2;
```
要使用设备树还要修改 configs/mx6ull_lunuj_defconfig 中的设备树配置：
```c
CONFIG_DEFAULT_DEVICE_TREE="imx6ull-lunuj"
```
这个时候就可以编译运行，简单测试一下。

### 移植 LCD 驱动
可以在 arch/arm/dts/imx6ull-lunuj.dtsi 中找到 lcdif 节点配置信息，修改为 7 寸屏的时许：
```dtb
&lcdif {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lcdif_dat
		     &pinctrl_lcdif_ctrl>;

	display = <&display0>;
	status = "okay";

	display0: display@0 {
		bits-per-pixel = <24>;
		bus-width = <24>;

		display-timings {
			native-mode = <&timing0>;
			timing0: timing0 {
			clock-frequency = <51200000>;
			hactive = <1024>;
			vactive = <600>;
			hfront-porch = <160>;
			hback-porch = <140>;
			hsync-len = <20>;
			vback-porch = <20>;
			vfront-porch = <12>;
			vsync-len = <3>;

			hsync-active = <0>;
			vsync-active = <0>;
			de-active = <1>;
			pixelclk-active = <0>;
			};
		};
	};
};

```
之前修改板级配置文件时，已经屏蔽过 LCD 复位，接下来还要修改 lcdif_dat 修改引脚的电气属性，将 lcdif_dat 节点里所有 0x79 改成 0x49 这样是为了降低 IO 驱动能力，防止 alpha 板上的模拟开关影响网络芯片。
### 移植网络驱动
开发板将GPIO5_IO07和GPIO5_IO08分别用作enet1和enet2的复位引脚，需要修改复位。
首先屏蔽这两个引脚复用：
```dtb
	pinctrl_spi4: spi4grp {
		fsl,pins = <
			MX6UL_PAD_BOOT_MODE0__GPIO5_IO10	0x70a1
			MX6UL_PAD_BOOT_MODE1__GPIO5_IO11	0x70a1
			// MX6UL_PAD_SNVS_TAMPER7__GPIO5_IO07	0x70a1
			// MX6UL_PAD_SNVS_TAMPER8__GPIO5_IO08	0x80000000
		>;
	};
```
和 spi-4 节点相关引脚的使用：
```
	spi-4 {
		compatible = "spi-gpio";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_spi4>;
		status = "okay";
		gpio-sck = <&gpio5 11 0>;
		gpio-mosi = <&gpio5 10 0>;
		// cs-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
		num-chipselects = <1>;
		#address-cells = <1>;
		#size-cells = <0>;

		gpio_spi: gpio@0 {
			compatible = "fairchild,74hc595";
			gpio-controller;
			#gpio-cells = <2>;
			reg = <0>;
			registers-number = <1>;
			registers-default = /bits/ 8 <0x57>;
			spi-max-frequency = <100000>;
			// enable-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>;
		};
	};
```
之后在 fec 节点下增加复位信息：
```dtb
&fec1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_enet1>;
	phy-mode = "rmii";
	phy-handle = <&ethphy0>;
	phy-supply = <&reg_peri_3v3>;
	phy-reset-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
	phy-reset-duration = <200>;
	phy-reset-post-delay = <200>;
	status = "okay";
};

&fec2 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_enet2>;
	phy-mode = "rmii";
	phy-handle = <&ethphy1>;
	phy-supply = <&reg_peri_3v3>;
	phy-reset-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>;
	phy-reset-duration = <200>;
	phy-reset-post-delay = <200>;
	status = "okay";

	mdio {
		#address-cells = <1>;
		#size-cells = <0>;
		ethphy0: ethernet-phy@0 {
			compatible = "ethernet-phy-id0022.1560";
			reg = <0>;
			micrel,led-mode = <1>;
			clocks = <&clks IMX6UL_CLK_ENET_REF>;
			clock-names = "rmii-ref";

		};
		ethphy1: ethernet-phy@1 {
			compatible = "ethernet-phy-id0022.1560";
			reg = <1>;
			micrel,led-mode = <1>;
			clocks = <&clks IMX6UL_CLK_ENET2_REF>;
			clock-names = "rmii-ref";
		};
	};
};
```
在 2.4 版本以前用的 **PHY** 芯片是 LAN8720A，还需要修改其他地方：
1. 之前增加的 configs/mx6ull_lunuj_defconfig 配置文件中需要删去 **NXP** 官方使用的芯片
```c
CONFIG_PHY_MICREL=y
CONFIG_PHY_MICREL_KSZ8XXX=y
```
2. board/freescale/mx6ull_lunuj/mx6ull_lunuj.c 文件中是复制 **NXP** 官方的内容，其中有
```c
int board_phy_config(struct phy_device *phydev)
{
	phy_write(phydev, MDIO_DEVAD_NONE, 0x1f, 0x8190);

	if (phydev->drv->config)
		phydev->drv->config(phydev);

	return 0;
}
```
函数配置了 0x1f 寄存器，在 **NXP** 使用的 **KSZ8081** 芯片可能是配置页寄存器的（具体需要看芯片数据手册）。
但是在 **LAN8720A** 中这个寄存器中 **PHY** 状态控制寄存器，其中 11:5 位要求写入固定值，如果写入其他数值，**PHY** 芯片可能工作异常。
这个寄存器也不需要额外去配置，默认值就是要求写入的固定值，只是不能在 `board_phy_config` 函数中写入其他值，直接删去整个函数即可，驱动会去调用 drivers/net/phy/phy.c 中的版本。
```c
__weak int board_phy_config(struct phy_device *phydev)
{
	if (phydev->drv->config)
		return phydev->drv->config(phydev);
	return 0;
}
```

## 6. 测试 uboot
### 增加编译脚本
**uboot** 修改完成后可以增加以下配置脚本来简化编译：
```shell
#!/bin/bash
ARCH=arm
CROSS_COMPILE=/home/lunuj/project/toolchain/arm/gcc-linaro-13.0.0-2022.11-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-
while getopts "ach" opt; do
    case "$opt" in
        c)
            echo "清除配置"
            make ARCH=${ARCH} CROSS_COMPILE=${CROSS_COMPILE} distclean
            exit 0
            ;;
        a)
            echo "完全编译"
            make ARCH=${ARCH} CROSS_COMPILE=${CROSS_COMPILE} distclean
            ;;
        h)
            echo -e "默认:  根据当前配置文件编译\n-a    :完全编译\n-c    :清除配置\n-h    :帮助信息\n"
            exit 0
            ;;
        *)
            echo "无效选项"
            exit 1
            ;;
    esac
done

make ARCH=${ARCH} CROSS_COMPILE=${CROSS_COMPILE} mx6ull_lunuj_defconfig
make ARCH=${ARCH} CROSS_COMPILE=${CROSS_COMPILE} -j32

if [ $? == 0 ]
then
    mv u-boot* ./tmp
    cp ./tmp/u-boot-dtb.imx ../share
    echo "编译完成"
else
    echo "编译失败"
fi
```
编译命令：
```shell
# 首次编译
./build.sh -a
# 继续编译
./build.sh
# 清除编译
./build.sh -c
```
### 烧录 uboot
**NXP** 提供的烧录方式有很多，可以通过 **OTG** 烧录，也可以通过 **SD** 卡启动。**OTG** 烧录可以使用 **NXP** 提供的工具，直接烧写到内部存储。
这里选择 **NXP** 提供的 imx_usb_loader 工具，imx_usb_loader 工具会将文件直接写入 ddr 中，而不是内部存储，对于测试比较方便，缺点是掉电会丢失，但也可以通过 **uboot** 手动写入对应的内置存储设备中。
从[imx_usb_loader 官方仓库](https://github.com/boundarydevices/imx_usb_loader.git)下载下来，从 README 中可以看到具体的使用方法，这里使用windows 下 MinGW 方式编译:
```shell
mingw32-make -f Makefile.mingw LIBUSBPATH=C:\Msys2\mingw64\lib\
```
注意需要指定 libusb 的库路径，编译的时候需要。这个库是 **MinGW** 安装自带的，之后可能会出一篇 **Msys2** 的安装文档。
编译完成后会生成两个文件：
- imx_usb.exe：USB 下载方式。
- mx_uart.exe：UART 下载方式。
正常使用 imx_usb.exe 就好，连接电脑和开发板，调整开发板跳线为 **USB** 启动（不能插入 **SD** 卡），确认设备管理器中正确设备到 **USB**，通过 **Msys2** 在编译目录下运行：
```shell
$ ./imx_usb.exe u-boot.imx
Trying to open device vid=0x15a2 pid=0x0080
Interface 0 claimed
HAB security state: development mode (0x56787856)
== work item
filename u-boot.imx
load_size 0 bytes
load_addr 0x00000000
dcd 1
clear_dcd 0
plug 1
jump_mode 3
jump_addr 0x00000000
== end work item
loading DCD table @0x910000

<<<488, 488 bytes>>>
succeeded (security 0x56787856, status 0x128a8a12)
clear dcd_ptr=0x877ff42c

loading binary file(u-boot.imx) to 877ff400, skip=0, fsize=68c00 type=aa

<<<429056, 429056 bytes>>>
succeeded (security 0x56787856, status 0x88888888)
jumping to 0x877ff400
```
启动 **uboot** 时，系统会提示ethernet地址没有设置，这是网口的地址。这个error是不影响使用的，**uboot** 仍然能正常运行，只要设置好参数就可以正常运行了。
```shell
setenv ipaddr 172.25.20.100
setenv eth1addr 00:04:9f:04:d2:36
setenv gatewayip 172.25.0.1
setenv netmask 255.255.0.0
setenv serverip 172.25.1.1
saveenv
```
**uboot** 启动内核是会解析设备树文件，如果要启动旧版内核，版本不匹配的会会有报错，在 boot/image-fdt.c 文件中修改宏 CONFIG_OF_SYSTEM_SETUP 即可。
```c
	if (IS_ENABLED(CONFIG_OF_SYSTEM_SETUP) && 0) {
		fdt_ret = ft_system_setup(blob, gd->bd);
		if (fdt_ret) {
			printf("ERROR: system-specific fdt fixup failed: %s\n",
			       fdt_strerror(fdt_ret));
			goto err;
		}
	}
```
### 启动 Linux
要启动 **Linux**，需要有以下文件：
- linux 内核镜像。
- linux 设备树。
- rootfs 跟文件系统。

这里先使用正点原子提供的文件用于测试，步骤相对简单：
使用 tftp 下载内核镜像和设备树到内存：
```shell
tftp 80800000 zImage
tftp 83000000 imx6ull.dtb
```
使用 nfs 在启动参数中挂载根文件系统：
```shell
setenv bootargs console=ttymxc0,115200 root=/dev/nfs nfsroot=172.25.1.1:/home/share/rootfs,nfsvers=3,tcp rw ip=172.25.20.100:172.25.0.1:172.25.1.1:255.255.0.0::eth0:off
```
跳转至内核：
```shell
bootz 80800000 - 83000000
```
之后在补充新版本 **linux** 和根文件系统的内容。