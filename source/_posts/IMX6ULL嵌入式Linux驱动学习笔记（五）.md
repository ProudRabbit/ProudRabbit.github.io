---
title: IMX6ULL嵌入式Linux驱动学习笔记（五）
date: 2020-09-19 21:51:40
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 一、什么是设备树

1. 设备树：设备和树。
2. 描述设备树的文件叫做`DTS`(Device Tree Source)，这个 `DTS` 文件采用树形结构描述本板级设备，也就是开发板上的设备信息。比如CPU数量、内存基地址、`IIC` 接口上外接了哪些设备等等。
3. 由于以前板级信息都是写到 `.c` 文件里面，导致 `linux` 内核臃肿。因此将板级信息做成独立的格式，文件扩展名为 `.dts` 。一个平台或者机器对应一个 `.dts` 。

<!--more-->

## 二、DTS、DTB和DTC的关系

`.dts` 相当于 `.c` ，就是DTS源码文件。`DTC` 工具相当于 `gcc` 编译器，将 `.dts` 编译成 `.dtb` 。`dtb` 相当于 `bin` 文件或可执行文件。

通过 `make dtbs` 命令来编译所有的 `.dts` 文件，通过 `make xxxxx.dtb` 来编译对应的 `.dts` 文件，需要在 `makefile` 中添加 `.dts` 文件所在路径。编译 `dts` 文件的 `Makefile` 在 `arch/arm/boot/dts` 下。

## 三、DTS基本语法  

这篇[DTS入门知识](https://blog.csdn.net/u014717231/article/details/53139968)博客对`DTS`进行了入门的讲解。

1. DTS是 `/` 开始。

2. 从 `/` 根节点开始描述设备信息。

3. 在 `/` 根节点外有一些 `&cpu0` 的语句是”追加“。

4. 节点名字的要求

   `label: node-name@unit-address` 标签: 节点名字@地址。

   - `label` ：为了方便的访问节点。
   - `node-name` ：可使用的字符 [0\~9] [a\~z] [A~Z] [ , . + - _ ]。约定使用小写。
   - `unit-address` ：一般是外设寄存器的起始地址。有的是外设的设备地址或者其他含义，需要根据情况来分析。

## 四、设备树在系统中的体现

系统启动以后可以在根文件系统里面看到设备树的节点信息。在 `/proc/device-tree/` `->` `/sys/firmware/devicetree/base` 目录下存放着设备树信息。

内核启动的时候会解析设备树，然后在 `/proc/device-tree/`目录下呈现出来。

## 五、特殊节点  

1. `aliases` 节点，对节点进行取别名。
2. `chosen` 节点，主要目的就是将 `uboot` 里面的 `bootargs` 环境变量值，传递给 `linux` 内核作为命令行参数 `cmd line` 。`uboot` 会将 `bootargs` 环境变量写入 `chosen` 节点中，通过 `fdt_chosen` 函数。

## 六、属性   

| 嵌入式不常用或弃用属性 | 实例                                                         | 作用                                                         |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ranges                 | ranges = <child-bus-address,parent-bus-address,length> / `ranges;` | ranges是一个地址映射/转换表,ranges 属性每个项目由子地址、父地址和地址空间长度这三部分组成: <br/>`child-bus-address`:子总线地址空间的物理地址,由父节点的#address-cells 确定此物理地址所占用的字长。<br/>`parent-bus-address`:父总线地址空间的物理地址,同样由父节点的#address-cells 确定此物理地址所占用的字长。<br/>`length`:子地址空间的长度,由父节点的#size-cells 确定此地址长度所占用的字长。<br/>如果 ranges 属性值为空值，说明子地址空间和父地址空间完全相同，不需要进行地址转换。 |
| name                   |                                                              | name 属性用于记录节点名字,name 属性已经被弃用,不推荐使用name 属性,一些老的设备树文件可能会使用此属性。 |
| device_type            | device_type = "cpu";                                         | 用于描述设备的 FCode,但是设备树没有 FCode,所以此属性也被抛弃了。此属性只能用于 cpu 节点或者 memory 节点。imx6ull.dtsi 的 cpu0 节点用到了此属性。 |

```c
/* ranges属性不为空时 */
soc {
	compatible = "simple-bus";
	#address-cells = <1>;
	#size-cells = <1>;
	ranges = <0x0 0xe0000000 0x00100000>;
	
	serial {
		device_type = "serial";
		compatible = "ns16550";
		reg = <0x4600 0x100>;
		clock-frequency = <0>;
		interrupts = <0xA 0x8>;
		interrupt-parent = <&ipic>;
	};
};
 
第 6 行，节点 soc 定义的 ranges 属性，值为<0x0 0xe0000000 0x00100000>，此属性值指定
了一个 1024KB(0x00100000)的地址范围，子地址空间的物理起始地址为 0x0，父地址空间的物
理起始地址为 0xe0000000。
第 11 行， serial 是串口设备节点， reg 属性定义了 serial 设备寄存器的起始地址为 0x4600，
寄存器长度为 0x100。经过地址转换， serial 设备可以从 0xe0004600 开始进行读写操作，
0xe0004600 = 0x4600 + 0xe0000000。
```

---

| 属性           | 实例/可选值                                                  | 作用                                                   |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| compatible     | compatible = "fsl,imx6ul-evk-wm8960","fsl,imx-audio-wm8960"; | 设备兼容属性                                           |
| mode           | model = "wm8960-audio";                                      | 描述设备模块信息                                       |
| status         | `okay`/`disabled`/`fail`/`fail-sss`                          | 描述设备状态                                           |
| #address-cells | #address-cells = <1>;                                        | 描述了**子节点**`reg`属性中地址信息所占用的字长(32 位) |
| #size-cells    | #size-cells = <1>;                                           | 描述了**子节点**`reg`属性中长度信息所占的字长(32 位)   |
| reg            | reg = \<address1 length1 address2 length2......\>            | 描述设备起始地址和长度                                 |

## 七、特殊的属性  

`compatible`属性，值是字符串。

根节点`/`下的`compatible`属性，内核在启动的时候会检查是否支持此平台，在以前不使用设备树的时候会通过`machine id`来判断内核是否支持此机器。使用设备数后不再使用机器ID，而是使用根节点`/`下的`compatible`属性。

未使用设备树的结构体

```c
#define MACHINE_START(_type,_name)						\
static const struct machine_desc __mach_desc_##_type	\
__used													\
__attribute__((__section__(".arch.info.init"))) = {		\
	.nr = MACH_TYPE_##_type,							\
	.name = _name,										\
    
#define MACHINE_END										\
};
```

使用设备树的结构体

```c
#define DT_MACHINE_START(_name, _namestr)				\
static const struct machine_desc __mach_desc_##_name	\
__used 													\
__attribute__((__section__(".arch.info.init"))) = { 	\
	.nr = ~0,											\
	.name = _namestr,									\

#define MACHINE_END										\
};
```

## 八、Linux内核的OF操作函数  

1. 驱动如何获取到设备树中的设备信息。在驱动中使用`OF`函数获取设备树属性内容。

  2. 驱动要想获取到设备树节点内容，首先要找到节点。

  3. 查找节点的of函数：

     - `of_find_node_by_name`函数

       of_find_node_by_name 函数通过节点名字查找指定的节点,函数原型如下:

       ```c
       struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
       ```

       函数参数和返回值含义如下:
       from：开始查找的节点,如果为`NULL`表示从根节点开始查找整个设备树。
       name：要查找的节点名字。
       返回值：找到的节点,如果为`NULL`表示查找失败。

     - `of_find_node_by_type`函数

       of_find_node_by_type 函数通过`device_type`属性查找指定的节点,函数原型如下:

       ```c
       struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
       ```

       函数参数和返回值含义如下:
       from：开始查找的节点,如果为 `NULL` 表示从根节点开始查找整个设备树。
       type：要查找的节点对应的 `type` 字符串,也就是`device_type`属性值。
       返回值：找到的节点,如果为`NULL`表示查找失败。

     - `of_find_compatible_node`函数

       of_find_compatible_node 函数根据`device_type`和`compatible`这两个属性查找指定的节点,
       函数原型如下:

       ```c
       struct device_node *of_find_compatible_node(struct device_node *from,
       							const char *type, const char *compatible)
       ```

       函数参数和返回值含义如下:
       from：开始查找的节点,如果为`NULL`表示从根节点开始查找整个设备树。
       type：要查找的节点对应的`type`字符串,也就是`device_type`属性值,可以为`NULL`,表示忽略掉`device_type`属性。
       compatible：要查找的节点所对应的`compatible`属性列表。
       返回值：找到的节点,如果为`NULL`表示查找失败。

     - `of_find_matching_node_and_match`函数

       of_find_matching_node_and_match 函数通过`of_device_id`匹配表来查找指定的节点,函数原
       型如下:

       ```c
       struct device_node *of_find_matching_node_and_match(struct device_node *from,
                                                           const struct of_device_id *matches,
                                                           const struct of_device_id **match)
       ```

       函数参数和返回值含义如下:

       from：开始查找的节点,如果为`NULL`表示从根节点开始查找整个设备树。
       matches：`of_device_id`匹配表,也就是在此匹配表里面查找节点。
       match：找到的匹配的`of_device_id`。

       返回值：找到的节点,如果为`NULL`表示查找失败。
       
     - `of_find_node_by_path`函数  
       of_find_node_by_path 函数通过路径来查找指定的节点,函数原型如下:
     
       ```c
       struct device_node *of_find_node_by_path(const char *path)
       ```
     
       函数参数和返回值含义如下:
       path：带有全路径的节点名,可以使用节点的别名,比如`/backlight`就是`backlight`这个节点的全路径。
       返回值：找到的节点,如果为`NULL`表示查找失败。    

4. 查找父/子节点的of函数

   - `of_get_parent`函数

     of_get_parent 函数用于获取指定节点的父节点(如果有父节点的话),函数原型如下:

     ```c
     struct device_node *of_get_parent(const struct device_node *node)
     ```

     函数参数和返回值含义如下:

     node：要查找的父节点的节点。
     返回值：找到的父节点。

   - `of_get_next_child`函数

     of_get_next_child 函数用迭代的查找子节点,函数原型如下:

     ```c
     struct device_node *of_get_next_child(const struct device_node *node,
     										struct device_node *prev)
     ```

     函数参数和返回值含义如下:
     node：父节点。
     prev：前一个子节点,也就是从哪一个子节点开始迭代的查找下一个子节点。可以设置为`NULL`,表示从第一个子节点开始。
     返回值：找到的下一个子节点。

5. 提取属性值的of函数

   节点的属性信息里面保存了驱动所需要的内容,因此对于属性值的提取非常重要, Linux 内
   核中使用结构体`property`表示属性,此结构体同样定义在文件`include/linux/of.h`中,内容如下:

   ```c
   struct property {
   	char *name;				/* 属性名字 	*/
   	int length; 			/* 属性长度		*/
   	void *value;			/* 属性值		 */
   	struct property *next;	/* 下一个属性	*/
   	unsigned long _flags; 
   	unsigned int unique_id;
       struct bin_attribute attr; 
   };
   ```

   - `of_find_property`函数

     of_find_property 函数用于查找指定的属性,函数原型如下:

     ```c
     property *of_find_property(const struct device_node *np,
     							const char *name,int *lenp)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     name：属性名字。
     lenp：属性值的字节数
     返回值：找到的属性。

   - `of_property_count_elems_of_size`函数

     of_property_count_elems_of_size 函数用于获取属性中元素的数量,比如`reg`属性值是一个
     数组,那么使用此函数可以获取到这个数组的大小,此函数原型如下:

     ```c
     int of_property_count_elems_of_size(const struct device_node *np,
     									const char *propname, int elem_size)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     proname：需要统计元素数量的属性名字。
     elem_size：元素长度。
     返回值：得到的属性元素数量。

   - `of_property_read_u32_index`函数

     of_property_read_u32_index 函数用于从属性中获取指定标号的`u32`类型数据值(无符号 32
     位),比如某个属性有多个`u32`类型的值,那么就可以使用此函数来获取指定标号的数据值,此
     函数原型如下:

     ```c
     int of_property_read_u32_index(const struct device_node *np,
     								const char *propname, u32 index,
                                    u32 *out_value)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     proname：要读取的属性名字。
     index：要读取的值标号。
     out_value：读取到的值
     返回值：0 读取成功,负值,读取失败,`-EINVAL`表示属性不存在,`-ENODATA`表示没有
     要读取的数据,`-EOVERFLOW`表示属性值列表太小。

   - `of_property_read_u8_array` 函数
     `of_property_read_u16_array` 函数
     `of_property_read_u32_array` 函数
     `of_property_read_u64_array` 函数

     这 4 个函数分别是读取属性中 `u8`、`u16`、`u32` 和 `u64` 类型的数组数据,比如大多数的`reg`属
     性都是数组数据,可以使用这 4 个函数一次读取出`reg`属性中的所有数据。这四个函数的原型
     如下:

     ```c
     int of_property_read_u8_array(const struct device_node *np,
                                     const char *propname,
                                     u8 *out_values, size_t sz)
         
     int of_property_read_u16_array(const struct device_node *np,
                                     const char *propname,
                                     u16 *out_values, size_t sz)
         
     int of_property_read_u32_array(const struct device_node *np,
                                     const char *propname,
                                     u32 *out_values, size_t sz)
         
     int of_property_read_u64_array(const struct device_node *np,
                                     const char *propname,
                                     u64 *out_values,size_t sz)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     proname：要读取的属性名字。
     out_value：读取到的数组值,分别为`u8`、`u16`、`u32`和`u64`。
     sz：要读取的数组元素数量。
     返回值：0,读取成功,负值,读取失败,`-EINVAL`表示属性不存在,`-ENODATA`表示没
     有要读取的数据,`-EOVERFLOW`表示属性值列表太小。

   - `of_property_read_u8` 函数
     `of_property_read_u16` 函数
     `of_property_read_u32` 函数
     `of_property_read_u64` 函数

     有些属性只有一个整形值,这四个函数就是用于读取这种只有一个整形值的属性,分别用
     于读取 u8、u16、u32 和 u64 类型属性值,函数原型如下:

     ```c
     int of_property_read_u8(const struct device_node *np,
     						const char *propname, u8*out_value)
         
     int of_property_read_u16(const struct device_node *np,
                              const char *propname, u16 *out_value)
         
     int of_property_read_u32(const struct device_node *np,
                              const char*propname, u32 *out_value)
         
     int of_property_read_u64(const struct device_node *np,
                              const char *propname, u64 *out_value)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     proname：要读取的属性名字。
     out_value：读取到的数组值。
     返回值：0,读取成功,负值,读取失败,`-EINVAL`表示属性不存在,`-ENODATA`表示没
     有要读取的数据,`-EOVERFLOW`表示属性值列表太小。

   - `of_property_read_string` 函数

     of_property_read_string 函数用于读取属性中字符串值,函数原型如下:

     ```c
     int of_property_read_string(struct device_node *np,const char *propname,
     							const char **out_string)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     proname：要读取的属性名字。
     out_string：读取到的字符串值。
     返回值：0,读取成功,负值,读取失败。

   - `of_n_addr_cells` 函数

     of_n_addr_cells 函数用于获取`#address-cells`属性值,函数原型如下:

     ```c
     int of_n_addr_cells(struct device_node *np)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     返回值：获取到的`#address-cells`属性值。

   - `of_n_size_cells` 函数

     of_size_cells 函数用于获取`#size-cells`属性值,函数原型如下:

     ```c
     int of_n_size_cells(struct device_node *np)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     返回值：获取到的`#size-cells`属性值。

6. 其他常用of函数

   - `of_device_is_compatible`函数

     of_device_is_compatible 函数用于查看节点的`compatible`属性是否有包含`compat`指定的字
     符串,也就是检查设备节点的兼容性,函数原型如下:

     ```c
     int of_device_is_compatible(const struct device_node *device,
                                 const char *compat)
     ```

     函数参数和返回值含义如下:
     device：设备节点。
     compat：要查看的字符串。
     返回值： 0,节点的`compatible`属性中不包含`compat`指定的字符串;正数,节点的 `compatible`
     属性中包含`compat`指定的字符串。

   - `of_get_address` 函数

     of_get_address 函数用于获取地址相关属性,主要是`reg`或者`assigned-addresses`属性
     值,函数原型如下:

     ```c
     const __be32 *of_get_address(struct device_node *dev,int index,
                                  u64 *size, unsigned int *flags)
     ```

     函数参数和返回值含义如下:
     dev：设备节点。
     index：要读取的地址标号。
     size：地址长度。
     flags：参数,比如`IORESOURCE_IO`、`IORESOURCE_MEM`等
     返回值：读取到的地址数据首地址,为`NULL`的话表示读取失败。

   - `of_translate_address` 函数

     of_translate_address 函数负责将从设备树读取到的地址转换为物理地址,函数原型如下:

     ```c
     u64 of_translate_address(struct device_node *dev,const __be32 *in_addr)
     ```

     函数参数和返回值含义如下:
     dev：设备节点。
     in_addr：要转换的地址。
     返回值：得到的物理地址,如果为`OF_BAD_ADDR`的话表示转换失败。

   - `of_address_to_resource` 函数

     `IIC`、`SPI`、`GPIO` 等这些外设都有对应的寄存器,这些寄存器其实就是一组内存空间,Linux
     内核使用`resource`结构体来描述一段内存空间,“ resource”翻译出来就是“资源”,因此用`resource`结构体描述的都是设备资源信息,`resource`结构体定义在文件`include/linux/ioport.h`中,定义如下:

     ```c
     struct resource {
         resource_size_t start;
         resource_size_t end;
         const char *name;
         unsigned long flags;
         struct resource *parent, *sibling, *child;
     };
     ```

     对于 32 位的 SOC 来说,`resource_size_t`是 `u32` 类型的。其中`start`表示开始地址,`end` 表示结束地址,`name`是这个资源的名字,`flags`是资源标志位,一般表示资源类型,可选的资源标志
     定义在文件`include/linux/ioport.h `中,如下所示:

     ```c
     #define IORESOURCE_BITS 		0x000000ff
     #define IORESOURCE_TYPE_BITS 	0x00001f00
     #define IORESOURCE_IO 			0x00000100
     #define IORESOURCE_MEM 			0x00000200
     #define IORESOURCE_REG 			0x00000300
     #define IORESOURCE_IRQ 			0x00000400
     #define IORESOURCE_DMA 			0x00000800
     #define IORESOURCE_BUS 			0x00001000
     #define IORESOURCE_PREFETCH 	0x00002000
     #define IORESOURCE_READONLY 	0x00004000
     #define IORESOURCE_CACHEABLE 	0x00008000
     #define IORESOURCE_RANGELENGTH 	0x00010000
     #define IORESOURCE_SHADOWABLE 	0x00020000
     #define IORESOURCE_SIZEALIGN 	0x00040000
     #define IORESOURCE_STARTALIGN 	0x00080000
     #define IORESOURCE_MEM_64 		0x00100000
     #define IORESOURCE_WINDOW 		0x00200000
     #define IORESOURCE_MUXED 		0x00400000
     #define IORESOURCE_EXCLUSIVE 	0x08000000
     #define IORESOURCE_DISABLED 	0x10000000
     #define IORESOURCE_UNSET 		0x20000000
     #define IORESOURCE_AUTO 		0x40000000
     #define IORESOURCE_BUSY 		0x80000000
     ```

     一般最常见的资源标志就是`IORESOURCE_MEM`、 `IORESOURCE_REG`和`IORESOURCE_IRQ`等。接下来我们回到`of_address_to_resource`函数,此函数看名字像是从设备树里面提取资源值,但是本质上就是将`reg`属性值,然后将其转换为`resource`结构体类型,函数原型如下所示:

     ```c
     int of_address_to_resource(struct device_node *dev,int index,
                                struct resource *r)
     ```

     函数参数和返回值含义如下:
     dev：设备节点。
     index：地址资源标号。
     r：得到的`resource`类型的资源值。
     返回值：0,成功;负值,失败。

   - `of_iomap`函数

     of_iomap 函数用于直接内存映射,以前我们会通过`ioremap`函数来完成物理地址到虚拟地
     址的映射,采用设备树以后就可以直接通过`of_iomap`函数来获取内存地址所对应的虚拟地址,
     不需要使用`ioremap`函数了。当然了,你也可以使用`ioremap`函数来完成物理地址到虚拟地址
     的内存映射,只是在采用设备树以后,大部分的驱动都使用`of_iomap`函数了，`of_iomap`函数本
     质上也是将`reg`属性中地址信息转换为虚拟地址,如果`reg`属性有多段的话,可以通过`index`参
     数指定要完成内存映射的是哪一段,`of_iomap`函数原型如下:

     ```c
     void __iomem *of_iomap(struct device_node *np, int index)
     ```

     函数参数和返回值含义如下:
     np：设备节点。
     index：`reg`属性中要完成内存映射的段,如果`reg`属性只有一段的话`index`就设置为 0。
     返回值：经过内存映射后的虚拟内存首地址,如果为`NULL`的话表示内存映射失败。

## 九、设备树添加内容（开发中一般不使用这种方式）

```c
/* 自己添加的节点 2020-9-17 */
alphaled {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "atkalpha-led";
    status = "okay";
    reg = <	0X020C406C 0X04	/* CCM_CCGR1_BASE */
        0X020E0068 0X04	/* SW_MUX_GPIO1_IO03_BASE */
        0X020E02F4 0X04	/* SW_PAD_GPIO1_IO03_BASE */
        0X0209C000 0X04	/* GPIO1_DR_BASE */
        0X0209C004 0X04	/* GPIO1_GPIR_BASE */
        >;
};
```

## 十、驱动使用设备树例子

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_irq.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

/*
 * 入口和出口函数
 */
static int __init dtsof_init(void)
{
	int ret = 0;
	struct device_node *np = NULL; // 节点
	struct property *comppro = NULL;
	const char *str;
	u32 def_value = 0;
	int elesize = 0;
	u32 *brival;
	int i;

	/* 找到backlight节点 */
	np = of_find_node_by_path("/backlight");
	if (np == NULL)
	{
		ret = -EINVAL;
		goto fail_findnd;
	}

	/* 获取属性 */
	comppro = of_find_property(np, "compatible", NULL);
	if (comppro == NULL)
	{
		ret = -EINVAL;
		goto fail_findpro;
	}
	else
	{
		printk("compatible=%s\r\n", (char *)comppro->value);
	}

	ret = of_property_read_string(np, "status", &str);
	if (ret < 0)
	{
		goto fail_rs;
	}
	else
	{
		printk("status=%s\r\n", str);
	}

	/* 3 获取数字属性值 */
	ret = of_property_read_u32(np, "default-brightness-level", &def_value);
	if (ret < 0)
	{
		goto fail_read_u32;
	}
	else
	{
		printk("default-brightness-level = %d\r\n", def_value);
	}

	/* 获取数组类型的属性 */
	elesize = of_property_count_elems_of_size(np, "brightness-levels", sizeof(u32));
	if (elesize < 0)
	{
		ret = -EINVAL;
		goto fail_readele;
	}
	else
	{
		printk("brightness-levels elems size = %d\r\n", elesize);
	}

	/* 申请内存 */
	brival = kmalloc(elesize * sizeof(u32), GFP_KERNEL);
	if (!brival)
	{
		ret = -EINVAL;
		goto fail_mem;
	}
	
	/* 获取数组 */
	ret = of_property_read_u32_array(np, "brightness-levels", brival, elesize);
	if (ret < 0)
	{
		goto fail_read32array;
	}
	else
	{
		for ( i = 0; i < elesize; i++)
		{
			printk("brightness-levels[%d] = %d \r\n", i, brival[i]);
		}
	}
	kfree(brival);
	
	return ret;

fail_read32array:
	kfree(brival);	/* 释放内存 */	
fail_mem:
fail_readele:
fail_read_u32:
fail_rs:
	str = NULL;
fail_findpro:
fail_findnd:
	return ret;
}

static void __exit dtsof_exit(void)
{

}

/**
 * 注册入口和出口函数
 * */
module_init(dtsof_init);
module_exit(dtsof_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");
```

## 十一、设备树下的LED驱动实验

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_irq.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

#define DTSLED_CNT	1
#define DTSLED_NAME	"dtsled"

#define LEDOFF	0
#define LEDON 	1

/* 虚拟地址的指针 */
static void __iomem *CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_GDIR;
static void __iomem *GPIO1_DR;

/* 设备结构体 */
struct dtsled_dev {
	dev_t devid;	/* 设备号 */
	int major;		/* 主设备号 */
	struct cdev cdev;
	struct class *class;	/* 类 */
	struct device *device;	/* 设备 */
	struct device_node *nd;	/* 设备节点 */
};

struct dtsled_dev dtsled;	/* LED设备 */

void led_toggle(u8 state)
{
	u32 val = 0;

	if(state == LEDON)
	{
		val = readl(GPIO1_DR);
		val &= ~(1 << 3);
		writel(val, GPIO1_DR);	// 打开LED
	}
	else
	{
		val = readl(GPIO1_DR);
		val |= (1 << 3);
		writel(val, GPIO1_DR);	// 关闭LED
	}
	
}

static int dtsled_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &dtsled;
	
	return 0;
}	

static int dtsled_release(struct inode *inode, struct file *filp)
{
	struct dtsled_dev *dev = filp->private_data;
	return 0;
}

ssize_t dtsled_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
	int retvalue;
	unsigned char databuff[1];
	struct dtsled_dev *dev = filp->private_data;

	retvalue = copy_from_user(databuff, buf, count);
	if (retvalue < 0)
	{
		printk("kernel write failed!\r\n");
		return -EFAULT;
	}

	if (databuff[0] == LEDON)
	{
		led_toggle(LEDON);
	}
	else
	{
		led_toggle(LEDOFF);
	}

	return 0;
}

/* 字符操作设备集 */
static const struct file_operations dtsled_fops = {
	.owner = THIS_MODULE,
	.write = dtsled_write,
	.open = dtsled_open,
	.release = dtsled_release,
};

/* 入口和出口 */
static int __init dtsled_init(void)
{
	int ret = 0;
	int i;
	const char *str;
	u32 regdata[10];
	unsigned int val;

	/* 1.注册字符设备 */
	/* 1.1 申请设备号 */
	dtsled.major = 0;	/* 设备号由内核分配 */
	if (dtsled.major)
	{
		/* 定义了设备号 */
		dtsled.devid = MKDEV(dtsled.major,0);
		ret = register_chrdev_region(dtsled.devid, DTSLED_CNT, DTSLED_NAME);
	}
	else
	{
		/* 没有给定设备号,向内核申请*/
		ret = alloc_chrdev_region(&dtsled.devid, 0, DTSLED_CNT, DTSLED_NAME);
	}
	
	if (ret < 0)
	{
		goto fail_devid;
	}

	/* 1.1 添加字符设备 */
	dtsled.cdev.owner = THIS_MODULE;
	cdev_init(&dtsled.cdev, &dtsled_fops);
	ret = cdev_add(&dtsled.cdev, dtsled.devid, DTSLED_CNT);
	if (ret < 0)
	{
		goto fail_cdev;
	}
	
	/* 3.自动创建设备节点 */
	/* 3.1 创建类 */
	dtsled.class = class_create(THIS_MODULE,DTSLED_NAME);
	if (IS_ERR(dtsled.class))
	{
		ret = PTR_ERR(dtsled.class);
		goto fail_class;
	}

	/* 3.2创建设备节点 */
	dtsled.device = device_create(dtsled.class, NULL, dtsled.devid, NULL, DTSLED_NAME);
	if (IS_ERR(dtsled.device))
	{
		ret = PTR_ERR(dtsled.device);
		goto fail_device;
	}

	/* 获取设备树属性内容 */
	dtsled.nd = of_find_node_by_path("/alphaled");
	if (dtsled.nd == NULL)
	{
		ret = -EINVAL;
		goto fail_findnd;
	}

	ret = of_property_read_string(dtsled.nd, "status", &str);
	if (ret < 0)
	{
		goto fail_rs;
	}
	else
	{
		printk("status = %s\r\n", str);
	}

	ret = of_property_read_string(dtsled.nd, "compatible", &str);
	if (ret < 0)
	{
		goto fail_rs;
	}
	else
	{
		printk("compatible = %s\r\n", str);
	}
	
#if 0
	ret = of_property_read_u32_array(dtsled.nd, "reg", regdata, 10);
	if (ret < 0)
	{
		goto fail_rs;
	}
	else 
	{
		printk("red data：");
		for (i = 0; i < 10; i++)
		{
			printk("%#x ", regdata[i]);
		}
		printk("\r\n");
	}

	/* LED初始化 */
	/* 地址映射 */

	CCM_CCGR1 = ioremap(regdata[0], regdata[1]);
	SW_MUX_GPIO1_IO03 = ioremap(regdata[2], regdata[3]);
	SW_PAD_GPIO1_IO03 = ioremap(regdata[4], regdata[5]);
	GPIO1_DR = ioremap(regdata[6], regdata[7]);
	GPIO1_GDIR = ioremap(regdata[8], regdata[9]);
#endif
	/* 使用of_iomap直接获取地址映射 */
	CCM_CCGR1 = of_iomap(dtsled.nd, 0);
	SW_MUX_GPIO1_IO03 = of_iomap(dtsled.nd, 1);
	SW_PAD_GPIO1_IO03 = of_iomap(dtsled.nd, 2);
	GPIO1_DR = of_iomap(dtsled.nd, 3);
	GPIO1_GDIR = of_iomap(dtsled.nd, 4);

	/* 初始化 */
	val = readl(CCM_CCGR1);
	val &= ~(3 << 26);
	val |= (3 << 26);
	writel(val, CCM_CCGR1);	// 使能时钟

	writel(0x5, SW_MUX_GPIO1_IO03);		// 设置复用
	writel(0x10b0, SW_PAD_GPIO1_IO03);	// 设置电气属性
	
	val = readl(GPIO1_GDIR);
	val |= (1 << 3);
	writel(val, GPIO1_GDIR);	// 设置输出

	val = readl(GPIO1_DR);
	val &= ~(1 << 3);
	writel(val, GPIO1_DR);	// 默认打开LED

	return 0;

fail_rs:
fail_findnd:
	device_destroy(dtsled.class, dtsled.devid);
fail_device:
	class_destroy(dtsled.class);
fail_class:
	cdev_del(&dtsled.cdev);
fail_cdev:
	unregister_chrdev_region(dtsled.devid, DTSLED_CNT);
fail_devid:
	return ret;
}

static void __exit dtsled_exit(void)
{
	/* 取消地址映射 */
	iounmap(CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_DR);
	iounmap(GPIO1_GDIR);

	/* 摧毁设备 */
	device_destroy(dtsled.class, dtsled.devid);
	/* 摧毁类 */
	class_destroy(dtsled.class);
	/* 删除字符设备 */
	cdev_del(&dtsled.cdev);
	/* 释放设备号 */
	unregister_chrdev_region(dtsled.devid, DTSLED_CNT);
}

/* 注册驱动和卸载驱动 */
module_init(dtsled_init);
module_exit(dtsled_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");
```

## 十二、测试应用程序  

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

/**
 * ./ledAPP <filename> <0:1> 1 表示开灯 0 表示关灯
 * @param argc 应用程序参数个数
 * @param argv 保存的参数，字符串形式。
 * */
int main(int argc, char *argv[])
{
	int ret = 0;
	int fd = 0;
	char *filename;
	unsigned char databuff[1];

	if ( argc !=3)
	{
		printf("输入错误\r\n");
	}
	
	filename = argv[1];

	fd = open(filename, O_RDWR);
	if(fd < 0) {
		printf("Error: %s\r\n",filename);
		return -1;
	}

	/* 读 */
/* 	if (atoi(argv[2]) ==1 )		// 传递过来的是字符串，需要转换成数字
	{
		ret = read(fd, readbuf, 10);
		if (ret < 0)
		{
			printf("read file %s failed\r\n", filename);
		}
		else
		{
			
		}
	} */
	
	databuff[0] = atoi(argv[2]);

	/* 写 */
	ret = write(fd, databuff, sizeof(databuff));
	if (ret < 0)
	{
		printf("LED control failed!\r\n");
		close(fd);
		return -1;
	}

	/* 关闭 */
	ret = close(fd);
	
	return 0;
}
```

