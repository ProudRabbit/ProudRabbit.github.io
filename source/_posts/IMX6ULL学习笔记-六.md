---
title: IMX6ULL学习笔记(六)
date: 2020-09-18 23:11:20
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 移植NXP官方的linux和设备树到开发板

1. 首先使用默认配置文件，编译下测试linux能否在板子上运行。配置文件所在路径`arch/arm/configs/imx_v7_mfg_defconfig`

2. 通过修改NXP官方的默认配置文件和dtb配置文件，来适配开发板。

   > imx_v7_mfg_defconfig
   >
   > imx6ull-14x14-evk-emmc.dtb   
   >

<!--more-->

3. 修改`arch/arm/boot/dts` 下的 `Makefile` 文件，将修改后的dtb文件，添加进去。

   编译设备树文件，`make dtbs`。

4. 修改主频和网络驱动（需要保证linux系统可以正常运行，因此需要暂时使用根文件系统） 

   - 设置`bootcmd` 环境变量，使用 的是SD卡启动，镜像和设备树存放在SD卡中， `setenv bootcmd 'fatload mmc 0:1 80800000 zimage;fatload mmc 0:1 83000000 imx6ull-14x14-myboard.dtb;bootz 80800000 - 83000000;'`  
   - 设置`bootargs`，根文件系统存放在EMMC的分区2中，命令如下：`setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'`  
   - 将`imx6ull-14x14-myboard.dts`中的`usdhc2`节点，改为以下内容。  

   ```c
   pinctrl-names = "default", "state_100mhz", "state_200mhz";
   pinctrl-0 = <&pinctrl_usdhc2_8bit>;
   pinctrl-1 = <&pinctrl_usdhc2_8bit_100mhz>;
   pinctrl-2 = <&pinctrl_usdhc2_8bit_200mhz>;
   bus-width = <8>;
   non-removable;
   status = "okay";
   ```

   - 然后使用`boot` 命令启动`linux`，至此启动完成。

5. 修改网络驱动。

   修改dts文件对应位置代码如下

   ```c
   pinctrl_spi4: spi4grp {
   	fsl,pins = <
   		MX6ULL_PAD_BOOT_MODE0__GPIO5_IO10 0x70a1
   		MX6ULL_PAD_BOOT_MODE1__GPIO5_IO11 0x70a1
   		/*MX6ULL_PAD_SNVS_TAMPER7__GPIO5_IO07 0x70a1
   		MX6ULL_PAD_SNVS_TAMPER8__GPIO5_IO08 0x80000000*/
   		>;
   };
   ```

   ```c
   spi4 {
       compatible = "spi-gpio";
       pinctrl-names = "default";
       pinctrl-0 = <&pinctrl_spi4>;
       /* pinctrl-assert-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>; */
       status = "okay";
       gpio-sck = <&gpio5 11 0>;
       gpio-mosi = <&gpio5 10 0>;
       /* cs-gpios = <&gpio5 7 0>; */
       num-chipselects = <1>;
       #address-cells = <1>;
       #size-cells = <0>;
   
       gpio_spi: gpio_spi@0 {
           compatible = "fairchild,74hc595";
           gpio-controller;
           #gpio-cells = <2>;
           reg = <0>;
           registers-number = <1>;
           registers-default = /bits/ 8 <0x57>;
           spi-max-frequency = <100000>;
       };
   };
   ```

   ```c
   pinctrl_enet1: enet1grp {
       fsl,pins = <
           MX6UL_PAD_ENET1_RX_EN__ENET1_RX_EN	0x1b0b0
           MX6UL_PAD_ENET1_RX_ER__ENET1_RX_ER	0x1b0b0
           MX6UL_PAD_ENET1_RX_DATA0__ENET1_RDATA00	0x1b0b0
           MX6UL_PAD_ENET1_RX_DATA1__ENET1_RDATA01	0x1b0b0
           MX6UL_PAD_ENET1_TX_EN__ENET1_TX_EN	0x1b0b0
           MX6UL_PAD_ENET1_TX_DATA0__ENET1_TDATA00	0x1b0b0
           MX6UL_PAD_ENET1_TX_DATA1__ENET1_TDATA01	0x1b0b0
           MX6UL_PAD_ENET1_TX_CLK__ENET1_REF_CLK1	0x4001b031
           MX6UL_PAD_SNVS_TAMPER7__GPIO5_IO07 0x10b0	/* ENET1_RESET */
           >;
   };
   
   pinctrl_enet2: enet2grp {
       fsl,pins = <
           MX6UL_PAD_GPIO1_IO07__ENET2_MDC		0x1b0b0
           MX6UL_PAD_GPIO1_IO06__ENET2_MDIO	0x1b0b0
           MX6UL_PAD_ENET2_RX_EN__ENET2_RX_EN	0x1b0b0
           MX6UL_PAD_ENET2_RX_ER__ENET2_RX_ER	0x1b0b0
           MX6UL_PAD_ENET2_RX_DATA0__ENET2_RDATA00	0x1b0b0
           MX6UL_PAD_ENET2_RX_DATA1__ENET2_RDATA01	0x1b0b0
           MX6UL_PAD_ENET2_TX_EN__ENET2_TX_EN	0x1b0b0
           MX6UL_PAD_ENET2_TX_DATA0__ENET2_TDATA00	0x1b0b0
           MX6UL_PAD_ENET2_TX_DATA1__ENET2_TDATA01	0x1b0b0
           MX6UL_PAD_ENET2_TX_CLK__ENET2_REF_CLK2	0x4001b031
           MX6UL_PAD_SNVS_TAMPER8__GPIO5_IO08 0x10b0	/* ENET2_RESET */
           >;
   };
   ```

   ```c
   &fec1 {
   	pinctrl-names = "default";
   	pinctrl-0 = <&pinctrl_enet1>;
   	phy-mode = "rmii";
   	phy-handle = <&ethphy0>;
   	phy-reset-gpios = <&gpio5 7 GPIO_ACTIVE_LOW>;
   	phy-reset-duration = <200>;
   	status = "okay";
   };
   
   &fec2 {
   	pinctrl-names = "default";
   	pinctrl-0 = <&pinctrl_enet2>;
   	phy-mode = "rmii";
   	phy-handle = <&ethphy1>;
   	phy-reset-gpios = <&gpio5 8 GPIO_ACTIVE_LOW>;
   	phy-reset-duration = <200>;
   	status = "okay";
   
   	mdio {
   		#address-cells = <1>;
   		#size-cells = <0>;
   
   		ethphy0: ethernet-phy@0 {
   			compatible = "ethernet-phy-ieee802.3-c22";
   			reg = <0>;
   		};
   
   		ethphy1: ethernet-phy@1 {
   			compatible = "ethernet-phy-ieee802.3-c22";
   			reg = <1>;
   		};
   	};
   };
   ```

   修改`drivers/net/ethernet/freescale/fec_main.c`中的`fec_probe`函数，添加如下代码。

   ```c
   /* 设置 MX6UL_PAD_ENET1_TX_CLK 和 MX6UL_PAD_ENET2_TX_CLK
   * 这两个 IO 的复用寄存器的 SION 位为 1。
   */
   void __iomem *IMX6U_ENET1_TX_CLK;
   void __iomem *IMX6U_ENET2_TX_CLK;
   
   IMX6U_ENET1_TX_CLK = ioremap(0X020E00DC, 4);
   writel(0X14, IMX6U_ENET1_TX_CLK);
   
   IMX6U_ENET2_TX_CLK = ioremap(0X020E00FC, 4);
   writel(0X14, IMX6U_ENET2_TX_CLK);
   ```

   然后编译下设备树文件，并且在图形化界面中使能`LAN8720A`的驱动。

   > 1. Device Drivers
   >
   > 2. Network device support
   >
   > 3. PHY Device support and infrastructure
   >
   > 4. Drivers for SMSC PHYs
   
   最后编译下Linux的内核文件。
   
   然后使用如下命令加载Linux镜像到内存中。

   ```c
   fatload mmc 0:1 80800000 zimage 
   fatload mmc 0:1 83000000 imx6ull-14x14-myboard.dts 
   bootz 80800000 - 83000000
   ```

   

   