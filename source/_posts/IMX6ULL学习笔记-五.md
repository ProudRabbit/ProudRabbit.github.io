---
title: IMX6ULL学习笔记(五)
date: 2020-09-18 23:08:01
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 1. 移植NXP官方 uboot 到 alpha 开发板

1. 添加板子默认配置文件

   借鉴NXP官方6ull evk 开发板，修改NXP官方6ull evk开发板配置文件`configs/mx6ull_14x14_evk_emmc_defconfig`并重命名。

   ```
   CONFIG_SYS_EXTRA_OPTIONS="IMX_CONFIG=board/freescale/mx6ull_myboard_emmc/imximage.cfg,MX6ULL_EVK_EMMC_REWORK"
   CONFIG_ARM=y
   CONFIG_ARCH_MX6=y
   CONFIG_TARGET_MX6ULL_MYBOARD_EMMC=y
   CONFIG_CMD_GPIO=y
   ```

   <!--more-->

2. 添加板子对应的头文件  

   不同的板子，有一些需要配置的信息，一般是在一个头文件里配置，每个板子有一个。对于NXP官方的6ULL EVK板子，头文件是`include/configs/mx6ullevk.h`，复制该文件为`mx6ull_myboard_emmc.h`，然后修改`mx6ull_myboard_emmc.h`该文件内的条件编译为  

   ```
   #ifndef __MX6ULL_MYBOARD_EMMC_CONFIG_H
   #define __MX6ULL_MYBOARD_EMMC_CONFIG_H
   ```

3. 添加板子对应的板级文件夹

   每个板子都有特有的文件，也叫板级文件。这里我们将6ULL EVK的板级文件夹`board/freescale/mx6ullevk`直接拷贝一份，并重命名为`mx6ull_myboard_emmc` 。修改`mx6ull_myboard_emmc`文件夹中的`mx6ullevk.c` `Makefile` `imximage.cfg` `Kconfig`  `MAINTAINERS`文件。  最后需要修改`arch/arm/cpu/armv7/mx6/Kconfig`文件，添加以下代码  

   ```
   config TARGET_MX6ULL_MYBOARD_EMMC
   	bool "Support mx6ull_myboard_emmc"
   	select MX6ULL
   	select DM
   	select DM_THERMAL
   	
   source "board/freescale/mx6ull_myboard_emmc/Kconfig"
   ```

4. 修改uboot图形配置界面

5. LCD驱动修改

   修改`board/mx6ull_myboard_emmc/mx6ull_myboard_emmc.c` `include/configs/mx6ull_myboard_emmc.h` 文件来修改驱动。

   - 确认LCD IO 初始化正确，修改`mx6ull_myboard_emmc.c`中的LCD_PADS
   - 修改LCD 参数，`mx6ull_myboard_emmc.c`中的displays。fb_videomode表示RGB LCD参数。
   - 修改`mx6ull_myboard_emmc.h`的环境变量panel=TFT43AB 为`mx6ull_myboard_emmc.c`中`display_info_t const displays[]`中的`name`值。  

6. 网络驱动修改

   LAN8720有一个管理接口，叫做MDIO，有两根线 `MDIO`、`MDC`，一个MDIO接口可以管理32个PHY芯片。MDIO通过`PHY ADDR`来确定访问哪个PHY芯片。ALPHA开发板的`ENET1`的`PHY ADDR`是`0x0`, `ENET2`的PHY ADDR是`0x1`；每个LAN8720都有一个复位引脚，ENET1是`SNVS_TAMPER8`。  

   LAN8720驱动，因为所有的PHY芯片的前16个寄存器一模一样，因此uboot里面已经写好了通用PHY驱动，所以理论上不需要修改。

- 修改`mx6ull_myboard_emmc.h`文件中的`PHY ADDR`  

```C
#if (CONFIG_FEC_ENET_DEV == 0)
#define IMX_FEC_BASE			ENET_BASE_ADDR
#define CONFIG_FEC_MXC_PHYADDR          0x0
#define CONFIG_FEC_XCV_TYPE             RMII
#elif (CONFIG_FEC_ENET_DEV == 1)
#define IMX_FEC_BASE			ENET2_BASE_ADDR
#define CONFIG_FEC_MXC_PHYADDR		0x1
#define CONFIG_FEC_XCV_TYPE		RMII
#endif	
```

   - 删除原有的74LV595的驱动代码 

```C
#define IOX_SDI IMX_GPIO_NR(5, 10)
#define IOX_STCP IMX_GPIO_NR(5, 7)
#define IOX_SHCP IMX_GPIO_NR(5, 11)
#define IOX_OE IMX_GPIO_NR(5, 8)
```

为

```c
#define ENET1_RESET IMX_GPIO_NR(5, 7)
#define ENET2_RESET IMX_GPIO_NR(5, 8)
```

然后删除和74LV595有关的代码，添加ALPHA开发板驱动代码。

分别在`mx6ull_myboard_emmc.c`中的`fec1_pads[]` 和`fec2_pads[]`中添加以下代码

```c
//fec1_pads[]
MX6_PAD_SNVS_TAMPER7__GPIO5_IO07 | MUX_PAD_CTRL(NO_PAD_CTRL),

//fec2_pads[]
MX6_PAD_SNVS_TAMPER8__GPIO5_IO08 | MUX_PAD_CTRL(NO_PAD_CTRL),
```

修改`setup_iomux_fec(int fec_id)`函数为以下代码

```c
static void setup_iomux_fec(int fec_id)
{
	if (fec_id == 0)
	{
		imx_iomux_v3_setup_multiple_pads(fec1_pads, ARRAY_SIZE(fec1_pads));
		// 添加ENET1_RESET复位代码
		gpio_direction_output(ENET1_RESET, 1);
		gpio_set_value(ENET1_RESET, 0);
		mdelay(20);
		gpio_set_value(ENET1_RESET, 1);
	}
	else
	{
		imx_iomux_v3_setup_multiple_pads(fec2_pads, ARRAY_SIZE(fec2_pads));
		// 添加ENET2_RESET复位代码
		gpio_direction_output(ENET2_RESET, 1);
		gpio_set_value(ENET2_RESET, 0);
		mdelay(20);
		gpio_set_value(ENET2_RESET, 1);
	}
}
```

在`int genphy_update_link(struct phy_device *phydev)`中的最上面添加以下代码

```c
#ifdef CONFIG_PHY_SMSC	// CONFIG_PHY_SMSC 需要在 mx6ull_myboard_emmc.h 中定义
	static int lan8720_flag = 0;
	int bmcr_reg = 0;
	if (lan8720_flag == 0)
	{
		bmcr_reg = phy_read(phydev, MDIO_DEVAD_NONE, MII_BMCR);
		phy_write(phydev, MDIO_DEVAD_NONE, MII_BMCR, BMCR_RESET);
		while (phy_read(phydev, MDIO_DEVAD_NONE, MII_BMCR) & 0X8000)
		{
			udelay(100);
		}
		phy_write(phydev, MDIO_DEVAD_NONE, MII_BMCR, bmcr_reg);
		lan8720_flag = 1;
	}
#endif
```

## 2. Uboot命令

| 命令    | 格式                                                         | 说明                                                         |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| md      | md[.b, .w, .l] address [# of objects]                        | 用于显示内存值                                               |
| nm      | nm [.b, .w, .l] address                                      | 用于修改指定地址的值，q退出                                  |
| mm      | mm [.b, .w, .l] address                                      | 如上，地址会自增                                             |
| cp      | cp [.b, .w, .l] source target count                          | 数据拷贝命令，用于将 DRAM 中的数据从一段内存拷贝到另一段内存中，或者把 Nor Flash 中的数据拷贝到 DRAM 中 |
| mmc     | mmc 是一系列的命令。？mmc 查询                               | uboot 支持 EMMC 和 SD 卡，因此也要提供 EMMC 和 SD 卡的操作命令。一般认为 EMMC 和 SD 卡是同一个东西。 |
| fatinfo | fatinfo \<interface\> [\<dev[:part]>]                        | 查询指定 MMC 设置指定分区的文件系统信息,例如`fatinfo mmc 1:1` |
| fatls   | fatls \<interface> [\<dev[:part]>] [directory]               | 用于查询 FAT 格式设备的目录和文件信息，如`fatls mmc 1:1`，查询 EMMC 分区 1 中的所有的目录和文件。 |
| fstype  | fstype \<interface> \<dev>:\<part>                           | 查看 MMC 设备某个分区的文件系统格式，如`fstype mmc 1:0`      |
| fatload | fatload \<interface> [\<dev[:part]> [\<addr> [\<filename> [bytes [pos]]]]] | 用于将指定的文件读取到 DRAM 中，如`fatload mmc 1:1 80800000 zImage` |
| bootz   | bootz [addr [initrd[:size]] [fdt]]                           | 引导[启动]Linux(zImage)，如`bootz 80800000 – 83000000`，80800000存放着Linux内核，83000000是设备树，不使用initrd时，使用－代替 |
| go      | go addr [arg ...]                                            | 用于跳到指定的地址处执行应用，如`tftp 87800000 printf.bin`   `go 87800000` |



## 3. Uboot 图形化配置方法

1. 通过终端配置

2. 首先进入到uboot的源码路径下

3. 然后使用默认配置 `make mx6ull_myboard_emmc_defconfig` 进行默认配置

4. 输入`make menuconfig` 打开图形化界面。**注意：**如果出现错误，需要安装`ncurses`库。  

   ```shell
   sudo apt install build-essential
   sudo apt install libncurses5
   sudo apt install libncurses5-dev
   ```

5. 图形化配置界面对于一个功能的编译或者叫做选择有三种模式。

  - Y：对应的功能编译到uboot里面。
  - N：对应的功能不编译到uboot里面。
  - M：将对应的功能编译成模块，linux内常用，uboot不支持。  

6. 当我们配置好后，因为只是写入到`.config`文件中，清理工程后会丢失，因此需要保存自己的配置文件。在图形配置界面，选择`save`选项来保存，使用`load`选项来加载配置文件。

