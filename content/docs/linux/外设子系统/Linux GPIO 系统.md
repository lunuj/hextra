---
title: Linux GPIO 系统
date: 2026-02-19
authors:
  - name: lunuj
    link: https://github.com/lunuj
tags:
  - 2025
  - Linux
  - GPIO
---
**Linux** 内核 **GPIO** 系统驱动框架。
<!--more-->  

# 软件框架

## 一、前言
作为一个刚入门的嵌入式工程师，刚开始的时候会学习一些简单的驱动，例如GPIO driver、LED driver。往往CPU datasheet的关于GPIO或者IO ports的章节都是比较简单的，非常适合刚入行的工程师。虽然GPIO子系统相关的硬件比较简单，没有复杂的协议，不过，对于软件抽象而言，其分层次的软件思想是每个嵌入式软件工程师需要掌握的内容。
GPIO driver仅仅包含了pin signal状态控制和读取的内容，而GPIO系统包括了pin multiplexing、pin configuration、GPIO control、GPIO interrupt control等内容。本文主参考wowo博客上的几篇文件，讲述linux kernel中GPIO系统的软件框架。

## 二、GPIO 硬件差异
嵌入式工程师总是要处理各种各样的target board，每个target board上的GPIO总是存在不同。

### 1 和CPU的连接方式不同
对于ARM的嵌入式硬件平台，SOC本身可以提供大量的IO port，SOC上的GPIO controller是通过SOC的总线连接到CPU的。对于嵌入式系统而言，除了SOC的IO port，一些外设芯片也可能会提供IO port，例如：
1. 有些key controller芯片、codec或者PMU的芯片会提供I/O port
2. 有些专用的IO expander芯片可以扩展16个或者32个GPIO
从硬件角度看，这些IO和SOC提供的那些IO完全不同，CPU和IO expander是通过I2C（也有可能是SPI等其他类型的bus）连接的，在这种情况下，访问这些SOC之外的GPIO需要I2C的操作，而控制SOC上的GPIO只需要写寄存器的操作。
不要小看这个不同，写一个SOC memory map的寄存器非常快，但是通过I2C来操作IO就不是那么快了，甚至，如果总线繁忙有可能阻塞当前进程，这种情况下，内核同步机制必须有所区别（如果操作GPIO可能导致sleep，那么同步机制不能采用spinlock）。

### 2 访问方式不同
SOC片内的GPIO controller和SOC片外的IO expander的访问当然不一样，不过，即便都是SOC片内的GPIO controller，不同的ARM芯片，其访问方式也不完全相同，例如：有些SOC的GPIO controller会提供一个寄存器来控制输出电平。向寄存器写1就是set high，向寄存器写0就是set low。但是有些SOC的GPIO controller会提供两个寄存器来控制输出电平。向其中一个寄存器写一就是set high，向另外一个寄存器写一就是set low。

### 3 配置方式不同
即便是使用了同样的硬件（例如都使用同样的某款SOC），不同硬件系统上GPIO的配置不同。在一个系统上配置为输入，在另外的系统上可能配置为输出。

### 4 GPIO特性不同
这些特性包括：
1. 是否能触发中断。对一个SOC而言，并非所有的IO port都支持中断功能，可能某些处理器只有一两组GPIO有中断功能。
2. 如果能够触发中断，那么该GPIO是否能够将CPU从sleep状态唤醒
3. 有些有软件可控的上拉或者下拉电阻的特性，有的GPIO不支持这种特性。在设定为输入的时候，有的GPIO可以设定debouce的算法，有的则不可以。

### 5 多功能复用
有的GPIO就是单纯的作为一个GPIO出现，有些GPIO有其他的复用的功能。例如IO expander上的GPIO只能是GPIO，但是SOC上的某个GPIO除了做普通的IO pin脚，还可以是SPI上clock信号线。
  
## 三、硬件功能分类
虽然GPIO controller的硬件描述中充满了大量的寄存器的描述，但是这些寄存器的功能大概分成下面三个类别。
### 1 pin controller
有些硬件逻辑是和IO port本身的功能设定相关的，我们称这个HW block为pin controller。软件通过设定pin controller这个硬件单元的寄存器可以实现：
1. 引脚功能配置。例如该I/O pin是一个普通的GPIO还是一些特殊功能引脚（例如memeory bank上CS信号）。
2. 引脚特性配置。例如pull-up/down电阻的设定，drive-strength的设定等。
### 2 GPIO controller
如果一组GPIO被配置成SPI，那么这些pin脚被连接到了SPI controller，如果配置成GPIO，那么控制这些引脚的就是GPIO controller。通过访问GPIO controller的寄存器，软件可以：
1. 配置GPIO的方向
2. 如果是输出，可以配置high level或者low level
3. 如果是输入，可以获取GPIO引脚上的电平状态
### 3 GPIO interrupt
如果一组gpio有中断控制器的功能，虽然控制寄存器在datasheet中的I/O ports章节描述，但是实际上这些GPIO已经被组织成了一个interrupt controller的硬件block，它更像是一个GPIO type的中断控制器，通过访问GPIO type的中断控制器的寄存器，软件可以：
1. 中断的enable和disable（mask和unmask）
2. 触发方式
3. 中断状态清除
## 四、通过软件抽象来掩盖硬件差异
传统的GPIO driver是负责上面三大类的控制，而新的linux kernel中的GPIO subsystem则用三个软件模块来对应上面三类硬件功能：
1. pin control subsystem。驱动pin controller硬件的软件子系统。
2. GPIO subsystem。驱动GPIO controller硬件的软件子系统。
3. GPIO interrupt chip driver。这个模块是作为一个interrupt subsystem中的一个底层硬件驱动模块存在的。本文主要描述前两个软件模块，具体GPIO interrupt chip driver以及interrupt subsystem请参考其他相关文档。
### 1 pin control subsystem block diagram
下图描述了pin control subsystem的模块图：
![pinctrl](http://www.wowotech.net/content/uploadfile/201407/613bd8f3a9ae41cb9511fd4d0bd55c4620140721064054.gif "pinctrl")
底层的pin controller driver是硬件相关的模组，初始化的时候会向pin control core模块注册pin control设备（通过pinctrl_register这个bootom level interface）。pin control core模块是一个硬件无关模块，它抽象了所有pin controller的硬件特性，仅仅从用户（各个driver就是pin control subsystem的用户）角度给出了top level的接口函数，这样，各个driver不需要关注pin controller的底层硬件相关的内容。
### 2 GPIO subsystem block diagram
下图描述了GPIO subsystem的模块图：
![gpio](http://www.wowotech.net/content/uploadfile/201407/490a9605e1250e088b8b2cc6e8201f2020140721064057.gif "gpio")
基本上这个软件框架图和pin control subsystem是一样的，其软件抽象的思想也是一样的，当然其内部具体的实现不一样。

# pin controller driver
## 一、前言
对于一个嵌入式软件工程师，软件模块经常和硬件打交道，pin control subsystem也不例外，被它驱动的硬件叫做pin controller，主要功能包括：
1. pin multiplexing。基于ARM core的嵌入式处理器一般会提供丰富的功能，例如camera interface、LCD interface、USB、I2C、SPI等等。虽然处理器有几百个pin，但是这些pin还是不够分配，因此有些pin需要复用。例如：127号GPIO可以做一个普通的GPIO控制LED，也可以配置成I2C的clock信号，也可以配置成SPI的data out信号。当然，这些功能不可能同时存在，因为硬件信号只有一个。
2. pin configuration。这些配置参数包括：pull-up/down电阻的设定， tri-state设定，drive-strength的设定。
3. enumerating and naming。引脚的命名和管理。

device tree的相关内容等待后续补充。

## 二、pin controller相关的DTS描述
类似其他的硬件，pin controller这个HW block需要是device tree中的一个节点。此外，各个其他的HW block在驱动之前也需要先配置其引脚复用功能，因此，这些device（我们称pin controller是host，那么这些使用pin controller进行引脚配置的device叫做client device）也需要在它自己的device tree node中描述pin control的相关内容

### 1 pin controller DTS结构
下面的伪代码描述了S3C2416 pin controller 的DTS结构：
```c
iomuxc: pinctrl@20e0000 {
    compatible = "fsl,imx6ul-iomuxc";               // 属性标签，驱动匹配加载
    reg = <0x020e0000 0x4000>;                      // 定义iomuxc管理寄存器范围
    pinctrl_led_improve: led-improve {
        fsl,pins = <
            MX6UL_PAD_GPIO1_IO03__GPIO1_IO03        0x40017059
        >;    
    };
    // 此处省略
};
```

每个pin configuration都是pin controller的child node，描述了client device要使用到的一组pin的配置信息。具体如何定义pin configuration是和具体的pin controller相关的。

在pin controller node中定义pin configuration其目的是为了让client device引用。所谓client device其实就是使用pin control subsystem提供服务的那些设备，例如串口设备。在使用之前，我们一般会在初始化代码中配置相关的引脚功能是串口功能。有了device tree，我们可以通过device tree来传递这样的信息。也就是说，各个device可以通过自己节点的属性来指向pin controller的某个child node，也就是pin configuration了。

### 2 pin configuration定义
`fsl,pins`中的信息就是寄存器的偏移地址和对应的值，哟过来配置某个引脚的电器属性和复用功能，以`#define MX6UL_PAD_GPIO1_IO03__GPIO1_IO03 0x40017059`为例：
- 宏展开后为`0x0068 0x02f4 0x0000 5 0 0x40017059`
- mux_reg：0x0068 寄存器偏移地址，寄存器用于定义引脚的复用模式，寄存器地址(0x020e0000+0x0068)
- conf_reg：0x02f4 配置寄存器偏移地址，寄存器用于配置引脚的性能，寄存器地址(0x02290000+0x02f4)
- input_reg：0 寄存器偏移地址，为0表示不存在
- mux_mode：5 配置mux寄存器的值，0x5表示为引脚复用模式(详细看手册)
- input_val: 0 寄存器不存在，配置0即可
- config_val: 0x40017059 定义config寄存器的值。

### 3 client device的DTS
一个典型的device tree中的外设node定义如下：
```c
usr_led {
    //.....                     
    pinctrl-names = "default", "improve";          //用于pinctrl查找state的名称，默认是default
    pinctrl-0 = <&pinctrl_gpio_led>;               //第一组引脚复用功能指定的结构
    pinctrl-1 =<&pinctrl_led_improve>;             //第二组引脚复用功能指定的结构
};
```

1. pinctrl-names定义了一个state列表。那么什么是state呢？具体说应该是pin state，对于一个client device，它使用了一组pin，这一组pin应该同时处于某种状态，毕竟这些pin是属于一个具体的设备功能。state的定义和电源管理关系比较紧密，例如当设备active的时候，我们需要pin controller将相关的一组pin设定为具体的设备功能，而当设备进入sleep状态的时候，需要pin controller将相关的一组pin设定为普通GPIO，并精确的控制GPIO状态以便节省系统的功耗。state有两种，标识，一种就是pinctrl-names定义的字符串列表，另外一种就是ID。ID从0开始，依次加一。根据例子中的定义，state ID等于0（名字是sleep）的state对应pinctrl-0属性，state ID等于1（名字是active）的state对应pinctrl-1属性。具体设备state的定义和各个设备相关，具体参考在自己的device bind。
2. pinctrl-x的定义。pinctrl-x是一个句柄（phandle）列表，每个句柄指向一个pin configuration。有时候，一个state对应多个pin configure。

## 三、 pin controller driver初始化
### 1 注册pin control device
在系统初始化的时候，dts描述的device node会形成一个树状结构，在machine初始化的过程中，会scan device node的树状结构，将真正的硬件device node变成一个个的设备模型中的device结构（比如struct platform_device）并加入到系统中。
```c
iomuxc: pinctrl@20e0000 {
    compatible = "fsl,imx6ul-iomuxc";               // 属性标签，驱动匹配加载
    reg = <0x020e0000 0x4000>;                      // 定义iomuxc管理寄存器范围
    pinctrl_led_improve: led-improve {
        fsl,pins = <
            MX6UL_PAD_GPIO1_IO03__GPIO1_IO03        0x40017059
        >;    
    };
    // 此处省略
};
```
reg属性描述pin controller硬件的地址信息，开始地址是0x020e0000 ，地址长度是0x4000
compatible属性用来描述pin controller的programming model。该属性的值是string list，定义了一系列的modle（每个string是一个model）。这些字符串列表被操作系统用来选择用哪一个pin controller driver来驱动该设备。
pin control subsystem要想进行控制，必须首先了解自己控制的对象，也就是说软件需要提供一个方法将各种硬件信息（total有多少可控的pin，有多少bank，pin的复用情况以及pin的配置情况）注册到pin control subsystem中，这也是pin controller driver的初始化的主要内容。这些信息一般通过定义静态的表格，该表格定义了一个大数组来描述每一个pin，也可以通过dts加上静态表格的方式。

### 2 注册pin controller driver
当然，这个device node也会变成一个platform device加入系统。有了device，那么对应的platform driver是如何注册到系统中的呢？代码如下：
```c
static struct platform_driver imx6ul_pinctrl_driver = {
	.driver = {
		.name = "imx6ul-pinctrl",
		.of_match_table = imx6ul_pinctrl_of_match,
		.suppress_bind_attrs = true,
	},
	.probe = imx6ul_pinctrl_probe,
};

static int __init imx6ul_pinctrl_init(void)
{
	return platform_driver_register(&imx6ul_pinctrl_driver);
}

arch_initcall(imx6ul_pinctrl_init);
```
系统初始化的时候，该函数会执行，向系统注册了pinctrl_driver。

### 3 driver初始化过程
在linux kernel引入统一设备模型之后，bus、driver和device形成了设备模型中的铁三角。对于platform这种类型的bus，其铁三角数据是platform_bus_type（表示platform这种类型的bus）、struct platform_device（platform bus上的device）、struct platform_driver（platform bus上的driver）。统一设备模型大大降低了驱动工程师的工作量，驱动工程师只要将driver注册到系统即可，剩余的事情交给统一设备模型来完成。每次系统增加一个platform_driver，platform_bus_type都会启动scan过程，让新加入的driver扫描整个platform bus上的device的链表，看看是否有device让该driver驱动。同样的，每次系统增加一个platform_device，platform_bus_type也会启动scan过程，遍历整个platform bus上的driver的链表，看看是否有适合驱动该device的driver。具体匹配的代码是platform bus上的match函数，如下：
```c
static int platform_match(struct device *dev, struct device_driver *drv)   
{   
    struct platform_device *pdev = to_platform_device(dev);   
    struct platform_driver *pdrv = to_platform_driver(drv);

    /* Attempt an OF style match first */   
    if (of_driver_match_device(dev, drv))   
        return 1;

    /* Then try ACPI style match */   
    if (acpi_driver_match_device(dev, drv))   
        return 1;

    /* Then try to match against the id table */   
    if (pdrv->id_table)   
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    /* fall-back to driver name match */   
    return (strcmp(pdev->name, drv->name) == 0);   
}
```
旧的的platform的匹配函数就是简单的比较device和driver的名字，多么简单，多么清晰，真是有点怀念过去单纯而美好的生活。当然，事情没有那么糟糕，我们这里只要关注OF style的匹配过程即可，思路很简单，解铃还需系铃人，把匹配过程推给device tree模块，代码如下：
```c
/**
 * of_driver_match_device - Tell if a driver's of_match_table matches a device.
 * @drv: the device_driver structure to test
 * @dev: the device structure to match against
 */
static inline int of_driver_match_device(struct device *dev,
					 const struct device_driver *drv)
{
	return of_match_device(drv->of_match_table, dev) != NULL;
}

const struct of_device_id *of_match_device(const struct of_device_id *matches,   
                       const struct device *dev)   
{   
    if ((!matches) || (!dev->of_node))   
        return NULL;   
    return of_match_node(matches, dev->of_node);   
}
```
platform driver提供了match table（struct device_driver 中的of_match_table的成员）。platform device提供了device tree node（struct device中的of_node成员）。
对于我们这个场景，match table是 imx6ul_pinctrl_of_match，代码如下：
```c
static const struct of_device_id imx6ul_pinctrl_of_match[] = {
	{ .compatible = "fsl,imx6ul-iomuxc", .data = &imx6ul_pinctrl_info, },
	{ .compatible = "fsl,imx6ull-iomuxc-snvs", .data = &imx6ull_snvs_pinctrl_info, },
	{ /* sentinel */ }
};
```
再去看看dts中pin controller的节点compatible属性的定义，你会禁不住感慨：啊，终于遇到对的人。这里还要特别说明的是data成员被设定为s3c2416_pin_ctrl ，它描述了2416的HW pin controller，我们称之samsung pin controller的描述符，初始化的过程中需要这个数据结构，后面还会详细介绍它。一旦pin controller这个device遇到了适当的driver，就会调用probe函数进行具体的driver初始化的动作了，代码如下：
```c
static int imx6ul_pinctrl_probe(struct platform_device *pdev)
{
	const struct imx_pinctrl_soc_info *pinctrl_info;

	pinctrl_info = of_device_get_match_data(&pdev->dev);
	if (!pinctrl_info)
		return -ENODEV;

	return imx_pinctrl_probe(pdev, pinctrl_info);
}

int imx_pinctrl_probe(struct platform_device *pdev,
		      const struct imx_pinctrl_soc_info *info)
{
	struct regmap_config config = { .name = "gpr" };
	struct device_node *dev_np = pdev->dev.of_node;
	struct pinctrl_desc *imx_pinctrl_desc;
	struct device_node *np;
	struct imx_pinctrl *ipctl;
	struct regmap *gpr;
	int ret, i;

	if (!info || !info->pins || !info->npins) {
		dev_err(&pdev->dev, "wrong pinctrl info\n");
		return -EINVAL;
	}

	if (info->gpr_compatible) {
		gpr = syscon_regmap_lookup_by_compatible(info->gpr_compatible);
		if (!IS_ERR(gpr))
			regmap_attach_dev(&pdev->dev, gpr, &config);
	}

	/* Create state holders etc for this driver */
	ipctl = devm_kzalloc(&pdev->dev, sizeof(*ipctl), GFP_KERNEL);
	if (!ipctl)
		return -ENOMEM;

	if (!(info->flags & IMX_USE_SCU)) {
		ipctl->pin_regs = devm_kmalloc_array(&pdev->dev, info->npins,
						     sizeof(*ipctl->pin_regs),
						     GFP_KERNEL);
		if (!ipctl->pin_regs)
			return -ENOMEM;

		for (i = 0; i < info->npins; i++) {
			ipctl->pin_regs[i].mux_reg = -1;
			ipctl->pin_regs[i].conf_reg = -1;
		}

		ipctl->base = devm_platform_ioremap_resource(pdev, 0);
		if (IS_ERR(ipctl->base))
			return PTR_ERR(ipctl->base);

		if (of_property_read_bool(dev_np, "fsl,input-sel")) {
			np = of_parse_phandle(dev_np, "fsl,input-sel", 0);
			if (!np) {
				dev_err(&pdev->dev, "iomuxc fsl,input-sel property not found\n");
				return -EINVAL;
			}

			ipctl->input_sel_base = of_iomap(np, 0);
			of_node_put(np);
			if (!ipctl->input_sel_base) {
				dev_err(&pdev->dev,
					"iomuxc input select base address not found\n");
				return -ENOMEM;
			}
		}
	}

	imx_pinctrl_desc = devm_kzalloc(&pdev->dev, sizeof(*imx_pinctrl_desc),
					GFP_KERNEL);
	if (!imx_pinctrl_desc)
		return -ENOMEM;

	imx_pinctrl_desc->name = dev_name(&pdev->dev);
	imx_pinctrl_desc->pins = info->pins;
	imx_pinctrl_desc->npins = info->npins;
	imx_pinctrl_desc->pctlops = &imx_pctrl_ops;
	imx_pinctrl_desc->pmxops = &imx_pmx_ops;
	imx_pinctrl_desc->confops = &imx_pinconf_ops;
	imx_pinctrl_desc->owner = THIS_MODULE;

	/* for generic pinconf */
	imx_pinctrl_desc->custom_params = info->custom_params;
	imx_pinctrl_desc->num_custom_params = info->num_custom_params;

	/* platform specific callback */
	imx_pmx_ops.gpio_set_direction = info->gpio_set_direction;

	mutex_init(&ipctl->mutex);

	ipctl->info = info;
	ipctl->dev = &pdev->dev;
	platform_set_drvdata(pdev, ipctl);
	ret = devm_pinctrl_register_and_init(&pdev->dev,
					     imx_pinctrl_desc, ipctl,
					     &ipctl->pctl);
	if (ret) {
		dev_err(&pdev->dev, "could not register IMX pinctrl driver\n");
		return ret;
	}

	ret = imx_pinctrl_probe_dt(pdev, ipctl);
	if (ret) {
		dev_err(&pdev->dev, "fail to probe dt properties\n");
		return ret;
	}

	dev_info(&pdev->dev, "initialized IMX pinctrl driver\n");

	return pinctrl_enable(ipctl->pctl);
}
```
#### devm_kzalloc
devm_kzalloc 函数是为struct samsung_pinctrl_drv_data数据结构分配内存。每当driver probe一个具体的device实例的时候，都需要建立一些私有的数据结构来保存该device的一些具体的硬件信息（本场景中，这个数据结构就是struct samsung_pinctrl_drv_data）。

在过去，驱动工程师多半使用kmalloc或者kzalloc来分配内存，但这会带来一些潜在的问题。例如：在初始化过程中，有各种各样可能的失败情况，这时候就依靠driver工程师小心的撰写代码，释放之前分配的内存。当然，初始化过程中，除了memory，driver会为probe的device分配各种资源，例如IRQ 号，io memory map、DMA等等。当初始化需要管理这么多的资源分配和释放的时候，很多驱动程序都出现了资源管理的issue。而且，由于这些issue是异常路径上的issue，不是那么容易测试出来，更加重了解决这个issue的必要性。

内核解决这个问题的模式（所谓解决一类问题的设计方法就叫做设计模式）是Devres，即device resource management软件模块。更细节的内容就不介绍了，其核心思想就是资源是设备的资源，那么资源的管理归于device，也就是说不需要driver过多的参与。当device和driver detach的时候，device会自动的释放其所有的资源。

#### pinctrl_desc
分配了struct pinctrl_desc数据结构的内存，下一步就是初始化这个数据结构了。struct pinctrl_desc和struct pinctrl_dev 都是pin control subsystem中core driver的概念。各个具体硬件的pin controller可能会各不相同，但是可以抽取其共同的部分来形成一个HW independent的数据结构，这个数据就是pin controller描述符，在core driver中用struct pinctrl_desc表示。
```c
/**
 * struct pinctrl_desc - pin controller descriptor, register this to pin
 * control subsystem
 * @name: name for the pin controller
 * @pins: an array of pin descriptors describing all the pins handled by
 *	this pin controller
 * @npins: number of descriptors in the array, usually just ARRAY_SIZE()
 *	of the pins field above
 * @pctlops: pin control operation vtable, to support global concepts like
 *	grouping of pins, this is optional.
 * @pmxops: pinmux operations vtable, if you support pinmuxing in your driver
 * @confops: pin config operations vtable, if you support pin configuration in
 *	your driver
 * @owner: module providing the pin controller, used for refcounting
 * @num_custom_params: Number of driver-specific custom parameters to be parsed
 *	from the hardware description
 * @custom_params: List of driver_specific custom parameters to be parsed from
 *	the hardware description
 * @custom_conf_items: Information how to print @params in debugfs, must be
 *	the same size as the @custom_params, i.e. @num_custom_params
 * @link_consumers: If true create a device link between pinctrl and its
 *	consumers (i.e. the devices requesting pin control states). This is
 *	sometimes necessary to ascertain the right suspend/resume order for
 *	example.
 */
struct pinctrl_desc {
	const char *name;
	const struct pinctrl_pin_desc *pins;
	unsigned int npins;
	const struct pinctrl_ops *pctlops;
	const struct pinmux_ops *pmxops;
	const struct pinconf_ops *confops;
	struct module *owner;
#ifdef CONFIG_GENERIC_PINCONF
	unsigned int num_custom_params;
	const struct pinconf_generic_params *custom_params;
	const struct pin_config_item *custom_conf_items;
#endif
	bool link_consumers;
};
```
其实整个初始化过程的核心思想就是low level的driver定义一个pinctrl_desc ，设定pin相关的定义和callback函数，注册到pin control subsystem中。我们用引脚描述符（pin descriptor）来描述一个pin。在pin control subsystem中，struct pinctrl_pin_desc用来描述一个可以控制的引脚，我们称之引脚描述符，代码如下：
```c
/**
 * struct pinctrl_pin_desc - boards/machines provide information on their
 * pins, pads or other muxable units in this struct
 * @number: unique pin number from the global pin number space
 * @name: a name for this pin
 * @drv_data: driver-defined per-pin data. pinctrl core does not touch this
 */
struct pinctrl_pin_desc {
	unsigned number;
	const char *name;
	void *drv_data;
};
```
冰冷的pin ID是给机器用的，而name是给用户使用的，是ascii字符。

struct pinctrl_dev在pin control subsystem的core driver中抽象一个pin control device。其实我们可以通过多个层面来定义一个device。在这个场景下，pin control subsystem的core driver关注的是一类pin controller的硬件设备，具体其底层是什么硬件连接方式，使用什么硬件协议它不关心，它关心的是pin controller这类设备所有的通用特性和功能。当然imx的pin controller是通过platform bus连接的，因此，在low level的层面，需要一个platform device来标识imx的pin controller（需要注意的是：pin controller class device和platform device都是基于一个驱动模型中的device派生而来的，这里struct device是基类，struct pinctrl_dev和struct platform_device都是派生类，当然c本身不支持class，但面向对象的概念是同样的）。

为了充分理解class这个概念，我们再举一个例子。对于audio的硬件抽象层，它应该管理所有的audio设备，因此这个软件模块应该有一个audio class的链表，连接了所有的系统中的audio设备。但这些具体的audio设备可能是PCI接口的audio设备，也可能是usb接口的audio设备，从具体的总线层面来看，也会有PCI或者USB设备来抽象对应的声卡设备。

对于SOC而言，其引脚除了配置成普通GPIO之外，若干个引脚还可以组成一个pin group，形成特定的功能。例如pin number是{ 0, 8, 16, 24 }这四个引脚组合形成一个pin group，提供SPI的功能。既然有了pin group的概念，为何又有function这个概念呢？什么是function呢？SPI是function，I2C也是一个function，当然GPIO也是一个function。一个function有可能对应一组或者多组pin。

例如：为了设计灵活，芯片内部的SPI0的功能可能引出到pin group { A8, A7, A6, A5 }，也可能引出到另外一个pin group{ G4, G3, G2, G1 }，但毫无疑问，这两个pin group不能同时active，毕竟芯片内部的SPI0的逻辑功能电路只有一个。 从这个角度看，pin control subsystem要进行功能设定的时候必须要给出function以及function的pin group才能确定所有的物理pin的位置。

#### imx_pinctrl
关于pin controller有两个描述符，一个是struct pinctrl_desc，是具体硬件无关的pin controller的描述符。struct imx_pinctrl 描述的具体imx pin controller硬件相关的信息，比如说：pin bank的信息，不是所有的pin controller都是分bank的，因此pin bank的信息只能封装在low level的samsung pin controller driver中。这个数据结构定义如下：
```c
/**
 * @dev: a pointer back to containing device
 * @base: the offset to the controller in virtual memory
 */
struct imx_pinctrl {
	struct device *dev;
	struct pinctrl_dev *pctl;
	void __iomem *base;
	void __iomem *input_sel_base;
	const struct imx_pinctrl_soc_info *info;
	struct imx_pin_reg *pin_regs;
	unsigned int group_index;
	struct mutex mutex;
};
```
关于上面的base可以多说两句。实际上，系统支持多个pin controller设备，这时候，系统要管理多个pin controller控制下的多个pin。每个pin有自己的pin ID，是唯一的，假设系统中有两个pin controller，一个是A，控制32个，另外一个是B，控制64个pin，我们可以统一编号，对A，pin ID从0～31，对于B，pin ID是从32～95。对于B，其pin base就是32。

### 4 获取pin group的信息

具体的代码如下：
```c

static int imx_pinctrl_probe_dt(struct platform_device *pdev,
				struct imx_pinctrl *ipctl)
{
	struct device_node *np = pdev->dev.of_node;
	struct device_node *child;
	struct pinctrl_dev *pctl = ipctl->pctl;
	u32 nfuncs = 0;
	u32 i = 0;
	bool flat_funcs;

	if (!np)
		return -ENODEV;

	flat_funcs = imx_pinctrl_dt_is_flat_functions(np);
	if (flat_funcs) {
		nfuncs = 1;
	} else {
		nfuncs = of_get_child_count(np);
		if (nfuncs == 0) {
			dev_err(&pdev->dev, "no functions defined\n");
			return -EINVAL;
		}
	}

	for (i = 0; i < nfuncs; i++) {
		struct function_desc *function;

		function = devm_kzalloc(&pdev->dev, sizeof(*function),
					GFP_KERNEL);
		if (!function)
			return -ENOMEM;

		mutex_lock(&ipctl->mutex);
		radix_tree_insert(&pctl->pin_function_tree, i, function);
		mutex_unlock(&ipctl->mutex);
	}
	pctl->num_functions = nfuncs;

	ipctl->group_index = 0;
	if (flat_funcs) {
		pctl->num_groups = of_get_child_count(np);
	} else {
		pctl->num_groups = 0;
		for_each_child_of_node(np, child)
			pctl->num_groups += of_get_child_count(child);
	}

	if (flat_funcs) {
		imx_pinctrl_parse_functions(np, ipctl, 0);
	} else {
		i = 0;
		for_each_child_of_node(np, child)
			imx_pinctrl_parse_functions(child, ipctl, i++);
	}

	return 0;
}
```
pin controller的device node有若干个child node，每个child node都描述了一个pin configuration。of_get_child_count函数可以获取pin configuration的数目。

一个pin configuration的device tree node被解析成两个描述符，一个是pin group的描述符，另外一个是pin mux function描述符。这两个描述符的名字都是根据dts file中的pin configuration的device node name生成，只不过pin group的名字附加-grp的后缀，而function描述符的名字后面附加-mux的后缀。

对于ping group描述符解释如下：
```c
/**
 * struct group_desc - generic pin group descriptor
 * @name: name of the pin group
 * @pins: array of pins that belong to the group
 * @num_pins: number of pins in the group
 * @data: pin controller driver specific data
 */
struct group_desc {
	const char *name;
	int *pins;
	int num_pins;
	void *data;
};
```

对于pin function描述符解释如下：
```c
/**
 * struct function_desc - generic function descriptor
 * @name: name of the function
 * @group_names: array of pin group names
 * @num_group_names: number of pin group names
 * @data: pin controller driver specific data
 */
struct function_desc {
	const char *name;
	const char **group_names;
	int num_group_names;
	void *data;
};
```

# pin control subsystem
## 一、前言
在linux2.6内核上工作的嵌入式软件工程师在pin control上都会遇到这样的状况：
1. 启动一个新的项目后，需要根据硬件平台的设定进行pin control相关的编码。例如：在bootloader中建立一个大的table，描述各个引脚的配置和缺省状态。此外，由于SOC的引脚是可以复用的，因此在各个具体的driver中，也可能会对引脚进行的配置。这些工作都是比较繁琐的工作，需要极大的耐心和细致度。
2. 发现某个driver不能正常工作，辛辛苦苦debug后发现仅仅是因为其他的driver在初始化的过程中修改了引脚的配置，导致自己的driver无法正常工作
3. 即便是主CPU是一样的项目，但是由于外设的不同，我们也不能使用一个kernel image，而是必须要修改代码（这些代码主要是board-specific startup code）
4. 代码不是非常的整洁，cut-and-pasted代码满天飞，linux中的冗余代码太多
作为一个嵌入式软件工程师，项目做多了，接触的CPU就多了，摔的跤就多了，之后自然会去思考，我们是否可以解决上面的问题呢？此外，对于基于ARM core那些SOC，虽然表面上看起来各个SOC各不相同，但是在pin control上还有很多相同的内容的，是否可以把它抽取出来，进行进一步的抽象呢？内核提出了pin control subsystem来解决这些问题。

## 二、pin control subsystem的文件列表
### 1 源文件列表
整理linux/drivers/pinctrl目录下pin control subsystem的源文件列表如下：

|                                 |                                                                         |
| ------------------------------- | ----------------------------------------------------------------------- |
| 文件名                             | 描述                                                                      |
| core.c core.h                   | pin control subsystem的core driver                                       |
| pinctrl-utils.c pinctrl-utils.h | pin control subsystem的一些utility接口函数                                     |
| pinmux.c pinmux.h               | pin control subsystem的core driver(pin muxing部分的代码，也称为pinmux driver)     |
| pinconf.c pinconf.h             | pin control subsystem的core driver(pin config部分的代码，也称为pin config driver) |
| devicetree.c devicetree.h       | pin control subsystem的device tree代码                                     |
| pinctrl-xxxx.c                  | 各种pin controller的low level driver。                                      |

### 2 和其他内核模块接口头文件
很多内核的其他模块需要用到pin control subsystem的服务，这些头文件就定义了pin control subsystem的外部接口以及相关的数据结构。我们整理linux/include/linux/pinctrl目录下pin control subsystem的外部接口头文件列表如下：

|            |                                                                                                                                                                                    |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 文件名        | 描述                                                                                                                                                                                 |
| consumer.h | 其他的driver要使用pin control subsystem的下列接口：   <br>a、设置引脚复用功能   <br>b、配置引脚的电气特性   <br>这时候需要include这个头文件                                                                                 |
| devinfo.h  | 这是for linux内核的驱动模型模块（driver model）使用的接口。struct device中包括了一个struct dev_pin_info    *pins的成员，这个成员描述了该设备的引脚的初始状态信息，在probe之前，driver model中的core driver在调用driver的probe函数之前会先设定pin state |
| machine.h  | 和machine模块的接口。                                                                                                                                                                     |

### 3 Low level pin controller driver接口
我们整理linux/include/linux/pinctrl目录下pin control subsystem提供给底层specific pin controller driver的头文件列表如下：

|                   |                                              |
| ----------------- | -------------------------------------------- |
| 文件名               | 描述                                           |
| pinconf-generic.h | 这个接口主要是提供给各种pin controller driver使用的，不是外部接口。 |
| pinconf.h         | pin configuration 接口                         |
| pinctrl-state.h   | pin control state状态定义                        |
| pinmux.h          | pin mux function接口                           |

## 三、pin control subsystem的软件框架图
### 1、功能和接口概述
一般而言，学习复杂的软件组件或者软件模块是一个痛苦的过程。我们可以把我们要学习的那个软件block看成一个黑盒子，不论里面有多么复杂，第一步总是先了解其功能和外部接口特性。如果你愿意，你可以不去看其内部实现，先自己思考其内部逻辑，并形成若干问题，然后带着这些问题去看代码，往往事半功倍。
#### 功能规格
pin control subsystem的主要功能包括：
1. 管理系统中所有可以控制的pin。在系统初始化的时候，枚举所有可以控制的pin，并标识这些pin。
2. 管理这些pin的复用（Multiplexing）。对于SOC而言，其引脚除了配置成普通GPIO之外，若干个引脚还可以组成一个pin group，形成特定的功能。例如pin number是{ 0, 8, 16, 24 }这四个引脚组合形成一个pin group，提供SPI的功能。pin control subsystem要管理所有的pin group。
3. 配置这些pin的特性。例如配置该引脚上的pull-up/down电阻，配置drive strength等
#### 接口规格
linux内核的某个软件组件必须放回到linux系统中才容易探讨它的接口以及在系统中的位置，因此，在本章的第二节会基于系统block上描述各个pin control subsystem和其他内核模块的接口。
#### 内部逻辑
要研究一个subsystem的内部逻辑，首先要打开黑盒子，细分模块，然后针对每一个模块进行功能分析、外部接口分析、内部逻辑分析。如果模块还是比较大，难于掌握，那么就继续细分，拆成子模块，重复上面的分析过程。在本章的第三节中，我们打开pin control subsystem的黑盒子进行进一步的分析。

### 2 pin control subsystem
pin control subsystem在和其他linux内核模块的接口关系图如下图所示：
![pcb](http://www.wowotech.net/content/uploadfile/201407/ed00896a96bea67a76b5d8a901aee43320140726102404.gif "pcb")
pin control subsystem会向系统中的其他driver提供接口以便进行该driver的pin config和pin mux的设定。理想的状态是GPIO controll driver也只是象UART,SPI这样driver一样和pin control subsystem进行交互，但是，实际上由于各种源由（后文详述），pin control subsystem和GPIO subsystem必须有交互。

### 3 pin control subsystem内部block diagram
![pccore](http://www.wowotech.net/content/uploadfile/201407/07edbb347ae8fb535f78f09a782bd36e20140726102406.gif "pccore")
起始理解了接口部分内容，阅读和解析pin control subsystem的内部逻辑已经很简单，本文就不再分析了。

## 四、pin control subsystem向其他driver提供的接口
当你准备撰写一个普通的linux driver（例如串口驱动）的时候，你期望pin control subsystem提供的接口是什么样子的？简单，当然最好是简单的，最最好是没有接口，当然这是可能的。
### 1 概述
普通driver调用pin control subsystem的主要目标是：
#### 设定该设备的功能复用
设定设备的功能复用需要了解两个概念，一个是function，另外一个pin group。function是功能抽象，对应一个HW逻辑block，例如SPI0。虽然给定了具体的gunction name，我们并不能确定其使用的pins的情况。例如：为了设计灵活，芯片内部的SPI0的功能可能引出到pin group { A8, A7, A6, A5 }，也可能引出到另外一个pin group{ G4, G3, G2, G1 }，但毫无疑问，这两个pin group不能同时active，毕竟芯片内部的SPI0的逻辑功能电路只有一个。 因此，只有给出function selector（所谓selector就是一个ID或者index）以及function的pin group selector才能进行function mux的设定。

#### 设定该device对应的那些pin的电气特性
此外，由于电源管理的要求，某个device可能处于某个电源管理状态，例如idle或者sleep，这时候，属于该device的所有的pin就会需要处于另外的状态。综合上述的需求，我们把定义了pin control state的概念，也就是说设备可能处于非常多的状态中的一个，device driver可以切换设备处于的状态。

为了方便管理pin control state，我们又提出了一个pin control state holder的概念，用来管理一个设备的所有的pin control状态。因此普通driver调用pin control subsystem的接口从逻辑上将主要是：

1. 获取pin control state holder的句柄
2. 设定pin control状态
3. 释放pin control state holder的句柄

pin control state holder的定义如下：
```c
/**
 * struct pinctrl - per-device pin control state holder
 * @node: global list node
 * @dev: the device using this pin control handle
 * @states: a list of states for this device
 * @state: the current state
 * @dt_maps: the mapping table chunks dynamically parsed from device tree for
 *	this device, if any
 * @users: reference count
 */
struct pinctrl {
	struct list_head node;
	struct device *dev;
	struct list_head states;
	struct pinctrl_state *state;
	struct list_head dt_maps;
	struct kref users;
};
```
系统中的每一个需要和pin control subsystem进行交互的设备在进行设定之前都需要首先获取这个句柄。而属于该设备的所有的状态都是挂入到一个链表中，链表头就是pin control state holder的states成员，一个state的定义如下：
```c
/**
 * struct pinctrl_state - a pinctrl state for a device
 * @node: list node for struct pinctrl's @states field
 * @name: the name of this state
 * @settings: a list of settings for this state
 */
struct pinctrl_state {
	struct list_head node;
	const char *name;
	struct list_head settings;
};
```
一个pin state包含若干个setting，所有的settings被挂入一个链表中，链表头就是pin state中的settings成员，定义如下：
```c
/**
 * struct pinctrl_setting - an individual mux or config setting
 * @node: list node for struct pinctrl_settings's @settings field
 * @type: the type of setting
 * @pctldev: pin control device handling to be programmed. Not used for
 *   PIN_MAP_TYPE_DUMMY_STATE.
 * @dev_name: the name of the device using this state
 * @data: Data specific to the setting type
 */
struct pinctrl_setting {
	struct list_head node;
	enum pinctrl_map_type type;
	struct pinctrl_dev *pctldev;
	const char *dev_name;
	union {
		struct pinctrl_setting_mux mux;
		struct pinctrl_setting_configs configs;
	} data;
};
```
当driver设定一个pin state的时候，pin control subsystem内部会遍历该state的settings链表，将一个一个的setting进行设定。这些settings有各种类型，定义如下：
```c
enum pinctrl_map_type {
	PIN_MAP_TYPE_INVALID,
	PIN_MAP_TYPE_DUMMY_STATE,
	PIN_MAP_TYPE_MUX_GROUP,
	PIN_MAP_TYPE_CONFIGS_PIN,
	PIN_MAP_TYPE_CONFIGS_GROUP,
};
```
有pin mux相关的设定（PIN_MAP_TYPE_MUX_GROUP），定义如下：
```c
/**
 * struct pinctrl_setting_mux - setting data for MAP_TYPE_MUX_GROUP
 * @group: the group selector to program
 * @func: the function selector to program
 */
struct pinctrl_setting_mux {
	unsigned group;
	unsigned func;
};
```
有了function selector以及属于该functiong的roup selector就可以进行该device和pin mux相关的设定了。设定电气特性的settings定义如下：
```c
/**
 * struct pinctrl_setting_configs - setting data for MAP_TYPE_CONFIGS_*
 * @group_or_pin: the group selector or pin ID to program
 * @configs: a pointer to an array of config parameters/values to program into
 *	hardware. Each individual pin controller defines the format and meaning
 *	of config parameters.
 * @num_configs: the number of entries in array @configs
 */
struct pinctrl_setting_configs {
	unsigned group_or_pin;
	unsigned long *configs;
	unsigned num_configs;
};
```
### 2 具体的接口
#### devm_pinctrl_get和 pinctrl_get
devm_pinctrl_get是Resource managed版本的pinctrl_get，核心还是pinctrl_get函数。这两个接口都是获取设备（设备模型中的struct device）的pin control state holder（struct pinctrl）。pin control state holder不是静态定义的，一般在第一次调用该函数的时候会动态创建。
```c

/**
 * pinctrl_get() - retrieves the pinctrl handle for a device
 * @dev: the device to obtain the handle for
 */
struct pinctrl *pinctrl_get(struct device *dev)
{
	struct pinctrl *p;

	if (WARN_ON(!dev))
		return ERR_PTR(-EINVAL);

	/*
	 * See if somebody else (such as the device core) has already
	 * obtained a handle to the pinctrl for this device. In that case,
	 * return another pointer to it.
	 */
	p = find_pinctrl(dev);
	if (p) {
		dev_dbg(dev, "obtain a copy of previously claimed pinctrl\n");
		kref_get(&p->users);
		return p;
	}

	return create_pinctrl(dev, NULL);
}
EXPORT_SYMBOL_GPL(pinctrl_get);

static struct pinctrl *create_pinctrl(struct device *dev,
				      struct pinctrl_dev *pctldev)
{
	struct pinctrl *p;
	const char *devname;
	struct pinctrl_maps *maps_node;
	int i;
	const struct pinctrl_map *map;
	int ret;

	/*
	 * create the state cookie holder struct pinctrl for each
	 * mapping, this is what consumers will get when requesting
	 * a pin control handle with pinctrl_get()
	 */
	p = kzalloc(sizeof(*p), GFP_KERNEL);
	if (!p)
		return ERR_PTR(-ENOMEM);
	p->dev = dev;
	INIT_LIST_HEAD(&p->states);
	INIT_LIST_HEAD(&p->dt_maps);

	ret = pinctrl_dt_to_map(p, pctldev);
	if (ret < 0) {
		kfree(p);
		return ERR_PTR(ret);
	}

	devname = dev_name(dev);

	mutex_lock(&pinctrl_maps_mutex);
	/* Iterate over the pin control maps to locate the right ones */
	for_each_maps(maps_node, i, map) {
		/* Map must be for this device */
		if (strcmp(map->dev_name, devname))
			continue;
		/*
		 * If pctldev is not null, we are claiming hog for it,
		 * that means, setting that is served by pctldev by itself.
		 *
		 * Thus we must skip map that is for this device but is served
		 * by other device.
		 */
		if (pctldev &&
		    strcmp(dev_name(pctldev->dev), map->ctrl_dev_name))
			continue;

		ret = add_setting(p, pctldev, map);
		/*
		 * At this point the adding of a setting may:
		 *
		 * - Defer, if the pinctrl device is not yet available
		 * - Fail, if the pinctrl device is not yet available,
		 *   AND the setting is a hog. We cannot defer that, since
		 *   the hog will kick in immediately after the device
		 *   is registered.
		 *
		 * If the error returned was not -EPROBE_DEFER then we
		 * accumulate the errors to see if we end up with
		 * an -EPROBE_DEFER later, as that is the worst case.
		 */
		if (ret == -EPROBE_DEFER) {
			pinctrl_free(p, false);
			mutex_unlock(&pinctrl_maps_mutex);
			return ERR_PTR(ret);
		}
	}
	mutex_unlock(&pinctrl_maps_mutex);

	if (ret < 0) {
		/* If some other error than deferral occurred, return here */
		pinctrl_free(p, false);
		return ERR_PTR(ret);
	}

	kref_init(&p->users);

	/* Add the pinctrl handle to the global list */
	mutex_lock(&pinctrl_list_mutex);
	list_add_tail(&p->node, &pinctrl_list);
	mutex_unlock(&pinctrl_list_mutex);

	return p;
}
```
#### devm_pinctrl_put和pinctrl_put
devm_pinctrl_get和pinctrl_get获取句柄的时候申请了很多资源，在devm_pinctrl_put和pinctrl_put可以释放。需要注意的是多次调用get函数不会重复分配资源，只会reference count加一，在put中referrenct count减一，当count＝＝0的时候才释放该device的pin control state holder持有的所有资源。

#### pinctrl_lookup_state
根据state name在pin control state holder找到对应的pin control state。具体的state是各个device自己定义的，不过pin control subsystem自己定义了一些标准的pin control state，定义在pinctrl-state.h文件中：
```c
/**
 * @PINCTRL_STATE_DEFAULT: the state the pinctrl handle shall be put
 *	into as default, usually this means the pins are up and ready to
 *	be used by the device driver. This state is commonly used by
 *	hogs to configure muxing and pins at boot, and also as a state
 *	to go into when returning from sleep and idle in
 *	.pm_runtime_resume() or ordinary .resume() for example.
 * @PINCTRL_STATE_INIT: normally the pinctrl will be set to "default"
 *	before the driver's probe() function is called.  There are some
 *	drivers where that is not appropriate becausing doing so would
 *	glitch the pins.  In those cases you can add an "init" pinctrl
 *	which is the state of the pins before drive probe.  After probe
 *	if the pins are still in "init" state they'll be moved to
 *	"default".
 * @PINCTRL_STATE_IDLE: the state the pinctrl handle shall be put into
 *	when the pins are idle. This is a state where the system is relaxed
 *	but not fully sleeping - some power may be on but clocks gated for
 *	example. Could typically be set from a pm_runtime_suspend() or
 *	pm_runtime_idle() operation.
 * @PINCTRL_STATE_SLEEP: the state the pinctrl handle shall be put into
 *	when the pins are sleeping. This is a state where the system is in
 *	its lowest sleep state. Could typically be set from an
 *	ordinary .suspend() function.
 */
#define PINCTRL_STATE_DEFAULT "default"
#define PINCTRL_STATE_INIT "init"
#define PINCTRL_STATE_IDLE "idle"
#define PINCTRL_STATE_SLEEP "sleep"
```
#### pinctrl_select_state
设定一个具体的pin control state接口。

## 五、和驱动模型的接口
前文已经表述过，最好是让统一设备驱动模型（Driver model）来处理pin 的各种设定。与其自己写代码调用devm_pinctrl_get、pinctrl_lookup_state、pinctrl_select_state等pin control subsystem的接口函数，为了不让linux内核自己的框架处理呢。
本章将分析具体的代码，这些代码实例对自己driver调用pin control subsystem的接口函数来设定本device的pin control的相关设定也是有指导意义的。 linux kernel中的驱动模型提供了driver和device的绑定机制，一旦匹配会调用probe函数如下：
```c
static int really_probe(struct device *dev, struct device_driver *drv)
{
	bool test_remove = IS_ENABLED(CONFIG_DEBUG_TEST_DRIVER_REMOVE) &&
			   !drv->suppress_bind_attrs;
	int ret;

	if (defer_all_probes) {
		/*
		 * Value of defer_all_probes can be set only by
		 * device_block_probing() which, in turn, will call
		 * wait_for_device_probe() right after that to avoid any races.
		 */
		dev_dbg(dev, "Driver %s force probe deferral\n", drv->name);
		return -EPROBE_DEFER;
	}

	ret = device_links_check_suppliers(dev);
	if (ret)
		return ret;

	pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
		 drv->bus->name, __func__, drv->name, dev_name(dev));
	if (!list_empty(&dev->devres_head)) {
		dev_crit(dev, "Resources present before probing\n");
		ret = -EBUSY;
		goto done;
	}

re_probe:
	dev->driver = drv;

	/* If using pinctrl, bind pins now before probing */
	ret = pinctrl_bind_pins(dev);
	if (ret)
		goto pinctrl_bind_failed;

	if (dev->bus->dma_configure) {
		ret = dev->bus->dma_configure(dev);
		if (ret)
			goto probe_failed;
	}

	ret = driver_sysfs_add(dev);
	if (ret) {
		pr_err("%s: driver_sysfs_add(%s) failed\n",
		       __func__, dev_name(dev));
		goto probe_failed;
	}

	if (dev->pm_domain && dev->pm_domain->activate) {
		ret = dev->pm_domain->activate(dev);
		if (ret)
			goto probe_failed;
	}

	ret = call_driver_probe(dev, drv);
	if (ret) {
		/*
		 * Return probe errors as positive values so that the callers
		 * can distinguish them from other errors.
		 */
		ret = -ret;
		goto probe_failed;
	}

	ret = device_add_groups(dev, drv->dev_groups);
	if (ret) {
		dev_err(dev, "device_add_groups() failed\n");
		goto dev_groups_failed;
	}

	if (dev_has_sync_state(dev)) {
		ret = device_create_file(dev, &dev_attr_state_synced);
		if (ret) {
			dev_err(dev, "state_synced sysfs add failed\n");
			goto dev_sysfs_state_synced_failed;
		}
	}

	if (test_remove) {
		test_remove = false;

		device_remove_file(dev, &dev_attr_state_synced);
		device_remove_groups(dev, drv->dev_groups);

		if (dev->bus->remove)
			dev->bus->remove(dev);
		else if (drv->remove)
			drv->remove(dev);

		devres_release_all(dev);
		arch_teardown_dma_ops(dev);
		kfree(dev->dma_range_map);
		dev->dma_range_map = NULL;
		driver_sysfs_remove(dev);
		dev->driver = NULL;
		dev_set_drvdata(dev, NULL);
		if (dev->pm_domain && dev->pm_domain->dismiss)
			dev->pm_domain->dismiss(dev);
		pm_runtime_reinit(dev);

		goto re_probe;
	}

	pinctrl_init_done(dev);

	if (dev->pm_domain && dev->pm_domain->sync)
		dev->pm_domain->sync(dev);

	driver_bound(dev);
	pr_debug("bus: '%s': %s: bound device %s to driver %s\n",
		 drv->bus->name, __func__, dev_name(dev), drv->name);
	goto done;

dev_sysfs_state_synced_failed:
	device_remove_groups(dev, drv->dev_groups);
dev_groups_failed:
	if (dev->bus->remove)
		dev->bus->remove(dev);
	else if (drv->remove)
		drv->remove(dev);
probe_failed:
	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_DRIVER_NOT_BOUND, dev);
pinctrl_bind_failed:
	device_links_no_driver(dev);
	devres_release_all(dev);
	arch_teardown_dma_ops(dev);
	kfree(dev->dma_range_map);
	dev->dma_range_map = NULL;
	driver_sysfs_remove(dev);
	dev->driver = NULL;
	dev_set_drvdata(dev, NULL);
	if (dev->pm_domain && dev->pm_domain->dismiss)
		dev->pm_domain->dismiss(dev);
	pm_runtime_reinit(dev);
	dev_pm_set_driver_flags(dev, 0);
done:
	return ret;
}
```
pinctrl_bind_pins的代码如下：
```c

/**
 * pinctrl_bind_pins() - called by the device core before probe
 * @dev: the device that is just about to probe
 */
int pinctrl_bind_pins(struct device *dev)
{
	int ret;

	if (dev->of_node_reused)
		return 0;

	dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
	if (!dev->pins)
		return -ENOMEM;

	dev->pins->p = devm_pinctrl_get(dev);
	if (IS_ERR(dev->pins->p)) {
		dev_dbg(dev, "no pinctrl handle\n");
		ret = PTR_ERR(dev->pins->p);
		goto cleanup_alloc;
	}

	dev->pins->default_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_DEFAULT);
	if (IS_ERR(dev->pins->default_state)) {
		dev_dbg(dev, "no default pinctrl state\n");
		ret = 0;
		goto cleanup_get;
	}

	dev->pins->init_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_INIT);
	if (IS_ERR(dev->pins->init_state)) {
		/* Not supplying this state is perfectly legal */
		dev_dbg(dev, "no init pinctrl state\n");

		ret = pinctrl_select_state(dev->pins->p,
					   dev->pins->default_state);
	} else {
		ret = pinctrl_select_state(dev->pins->p, dev->pins->init_state);
	}

	if (ret) {
		dev_dbg(dev, "failed to activate initial pinctrl state\n");
		goto cleanup_get;
	}

#ifdef CONFIG_PM
	/*
	 * If power management is enabled, we also look for the optional
	 * sleep and idle pin states, with semantics as defined in
	 * <linux/pinctrl/pinctrl-state.h>
	 */
	dev->pins->sleep_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_SLEEP);
	if (IS_ERR(dev->pins->sleep_state))
		/* Not supplying this state is perfectly legal */
		dev_dbg(dev, "no sleep pinctrl state\n");

	dev->pins->idle_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_IDLE);
	if (IS_ERR(dev->pins->idle_state))
		/* Not supplying this state is perfectly legal */
		dev_dbg(dev, "no idle pinctrl state\n");
#endif

	return 0;

	/*
	 * If no pinctrl handle or default state was found for this device,
	 * let's explicitly free the pin container in the device, there is
	 * no point in keeping it around.
	 */
cleanup_get:
	devm_pinctrl_put(dev->pins->p);
cleanup_alloc:
	devm_kfree(dev, dev->pins);
	dev->pins = NULL;

	/* Return deferrals */
	if (ret == -EPROBE_DEFER)
		return ret;
	/* Return serious errors */
	if (ret == -EINVAL)
		return ret;
	/* We ignore errors like -ENOENT meaning no pinctrl state */

	return 0;
}

```
1. struct device数据结构有一个pins的成员，它描述了和该设备相关的pin control的信息。
```c
/**
 * struct dev_pin_info - pin state container for devices
 * @p: pinctrl handle for the containing device
 * @default_state: the default state for the handle, if found
 * @init_state: the state at probe time, if found
 * @sleep_state: the state at suspend time, if found
 * @idle_state: the state at idle (runtime suspend) time, if found
 */
struct dev_pin_info {
	struct pinctrl *p;
	struct pinctrl_state *default_state;
	struct pinctrl_state *init_state;
#ifdef CONFIG_PM
	struct pinctrl_state *sleep_state;
	struct pinctrl_state *idle_state;
#endif
};
```
2. 调用devm_pinctrl_get获取该device对应的 pin control state holder句柄。
3. 搜索default state，sleep state，idle state并记录在本device中
4. 将该设备设定为pin default state

## 六、和device tree或者machine driver相关的接口
### 1 概述
device tree或者machine driver这两个模块主要是为 pin control subsystem提供pin mapping database的支持。这个database的每个entry用下面的数据结构表示：
```c
/**
 * struct pinctrl_map - boards/machines shall provide this map for devices
 * @dev_name: the name of the device using this specific mapping, the name
 *	must be the same as in your struct device*. If this name is set to the
 *	same name as the pin controllers own dev_name(), the map entry will be
 *	hogged by the driver itself upon registration
 * @name: the name of this specific map entry for the particular machine.
 *	This is the parameter passed to pinmux_lookup_state()
 * @type: the type of mapping table entry
 * @ctrl_dev_name: the name of the device controlling this specific mapping,
 *	the name must be the same as in your struct device*. This field is not
 *	used for PIN_MAP_TYPE_DUMMY_STATE
 * @data: Data specific to the mapping type
 */
struct pinctrl_map {
	const char *dev_name;
	const char *name;
	enum pinctrl_map_type type;
	const char *ctrl_dev_name;
	union {
		struct pinctrl_map_mux mux;
		struct pinctrl_map_configs configs;
	} data;
};

```
### 2 machine driver
machine driver通过静态定义的数据来建立pin mapping database

machine driver定义一个巨大的mapping table，描述，然后在machine初始化的时候，调用pinctrl_register_mappings将该table注册到pin control subsystem中。

### 3 通过device tree来建立pin mapping database
pin mapping信息定义在dts中，主要包括两个部分，一个是定义在各个具体的device node中，另外一处是定义在pin controller的device node中。

一个典型的device tree中的外设node定义如下：
```c
usr_led {
    //.....                     
    pinctrl-names = "default", "improve";          //用于pinctrl查找state的名称，默认是default
    pinctrl-0 = <&pinctrl_gpio_led>;               //第一组引脚复用功能指定的结构
    pinctrl-1 =<&pinctrl_led_improve>;             //第二组引脚复用功能指定的结构
};
```
对普通device的dts分析在函数pinctrl_dt_to_map中，代码如下：
```c
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
{
	struct device_node *np = p->dev->of_node;
	int state, ret;
	char *propname;
	struct property *prop;
	const char *statename;
	const __be32 *list;
	int size, config;
	phandle phandle;
	struct device_node *np_config;

	/* CONFIG_OF enabled, p->dev not instantiated from DT */
	if (!np) {
		if (of_have_populated_dt())
			dev_dbg(p->dev,
				"no of_node; not parsing pinctrl DT\n");
		return 0;
	}

	ret = dt_gpio_assert_pinctrl(p);
	if (ret) {
		dev_dbg(p->dev, "failed to assert pinctrl setting: %d\n", ret);
		return ret;
	}

	/* We may store pointers to property names within the node */
	of_node_get(np);

	/* For each defined state ID */
	for (state = 0; ; state++) {
		/* Retrieve the pinctrl-* property */
		propname = kasprintf(GFP_KERNEL, "pinctrl-%d", state);
		prop = of_find_property(np, propname, &size);
		kfree(propname);
		if (!prop) {
			if (state == 0) {
				of_node_put(np);
				return -ENODEV;
			}
			break;
		}
		list = prop->value;
		size /= sizeof(*list);

		/* Determine whether pinctrl-names property names the state */
		ret = of_property_read_string_index(np, "pinctrl-names",
						    state, &statename);
		/*
		 * If not, statename is just the integer state ID. But rather
		 * than dynamically allocate it and have to free it later,
		 * just point part way into the property name for the string.
		 */
		if (ret < 0)
			statename = prop->name + strlen("pinctrl-");

		/* For every referenced pin configuration node in it */
		for (config = 0; config < size; config++) {
			phandle = be32_to_cpup(list++);

			/* Look up the pin configuration node */
			np_config = of_find_node_by_phandle(phandle);
			if (!np_config) {
				dev_err(p->dev,
					"prop %s index %i invalid phandle\n",
					prop->name, config);
				ret = -EINVAL;
				goto err;
			}

			/* Parse the node */
			ret = dt_to_map_one_config(p, pctldev, statename,
						   np_config);
			of_node_put(np_config);
			if (ret < 0)
				goto err;
		}

		/* No entries in DT? Generate a dummy state table entry */
		if (!size) {
			ret = dt_remember_dummy_state(p, statename);
			if (ret < 0)
				goto err;
		}
	}

	return 0;

err:
	pinctrl_dt_free_maps(p);
	return ret;
}

```
1. pinctrl-0 pinctrl-1 pinctrl-2……表示了该设备的一个个的状态，这里我们定义了两个pinctrl-0和pinctrl-1分别对应sleep和default状态。这里每次循环分析一个pin state。
2. 代码执行到这里，size和list分别保存了该pin state中所涉及pin configuration phandle的数目以及phandle的列表
3. 读取从pinctrl-names属性中获取state name
4. 如果没有定义pinctrl-names属性，那么我们将pinctrl-0 pinctrl-1 pinctrl-2……中的那个ID取出来作为state name
5. 遍历一个pin state中的pin configuration list，这里的pin configuration实际应该是pin controler device node中的sub node，用phandle标识。
6. 用phandle作为索引，在device tree中找他该phandle表示的那个pin configuration
7. 分析一个pin configuration，具体下面会仔细分析
8. 如果该设备没有定义pin configuration，那么也要创建一个dummy的pin state。
这里我们已经进入对pin controller node下面的子节点的分析过程了。分析一个pin configuration的代码如下：
```C

static int dt_to_map_one_config(struct pinctrl *p,
				struct pinctrl_dev *hog_pctldev,
				const char *statename,
				struct device_node *np_config)
{
	struct pinctrl_dev *pctldev = NULL;
	struct device_node *np_pctldev;
	const struct pinctrl_ops *ops;
	int ret;
	struct pinctrl_map *map;
	unsigned num_maps;
	bool allow_default = false;

	/* Find the pin controller containing np_config */
	np_pctldev = of_node_get(np_config);
	for (;;) {
		if (!allow_default)
			allow_default = of_property_read_bool(np_pctldev,
							      "pinctrl-use-default");

		np_pctldev = of_get_next_parent(np_pctldev);
		if (!np_pctldev || of_node_is_root(np_pctldev)) {
			of_node_put(np_pctldev);
			ret = driver_deferred_probe_check_state(p->dev);
			/* keep deferring if modules are enabled */
			if (IS_ENABLED(CONFIG_MODULES) && !allow_default && ret < 0)
				ret = -EPROBE_DEFER;
			return ret;
		}
		/* If we're creating a hog we can use the passed pctldev */
		if (hog_pctldev && (np_pctldev == p->dev->of_node)) {
			pctldev = hog_pctldev;
			break;
		}
		pctldev = get_pinctrl_dev_from_of_node(np_pctldev);
		if (pctldev)
			break;
		/* Do not defer probing of hogs (circular loop) */
		if (np_pctldev == p->dev->of_node) {
			of_node_put(np_pctldev);
			return -ENODEV;
		}
	}
	of_node_put(np_pctldev);

	/*
	 * Call pinctrl driver to parse device tree node, and
	 * generate mapping table entries
	 */
	ops = pctldev->desc->pctlops;
	if (!ops->dt_node_to_map) {
		dev_err(p->dev, "pctldev %s doesn't support DT\n",
			dev_name(pctldev->dev));
		return -ENODEV;
	}
	ret = ops->dt_node_to_map(pctldev, np_config, &map, &num_maps);
	if (ret < 0)
		return ret;
	else if (num_maps == 0) {
		/*
		 * If we have no valid maps (maybe caused by empty pinctrl node
		 * or typing error) ther is no need remember this, so just
		 * return.
		 */
		dev_info(p->dev,
			 "there is not valid maps for state %s\n", statename);
		return 0;
	}

	/* Stash the mapping table chunk away for later use */
	return dt_remember_or_free_map(p, statename, pctldev, map, num_maps);
}
```
1. 首先找到该pin configuration node对应的parent node（也就是pin controler对应的node），如果找不到或者是root node，则进入出错处理。
2. 获取pin control class device
3. 一旦找到pin control class device则跳出for循环
4. 调用底层的callback函数处理pin configuration node。这也是合理的，毕竟很多的pin controller bindings是需要自己解析的。
5. 将该pin configuration node的mapping entry信息注册到系统中

## 七、core driver和low level pin controller driver的接口规格
pin controller描述符。每一个特定的pin controller都用一个struct pinctrl_desc来抽象，具体如下：
```c
/**
 * struct pinctrl_desc - pin controller descriptor, register this to pin
 * control subsystem
 * @name: name for the pin controller
 * @pins: an array of pin descriptors describing all the pins handled by
 *	this pin controller
 * @npins: number of descriptors in the array, usually just ARRAY_SIZE()
 *	of the pins field above
 * @pctlops: pin control operation vtable, to support global concepts like
 *	grouping of pins, this is optional.
 * @pmxops: pinmux operations vtable, if you support pinmuxing in your driver
 * @confops: pin config operations vtable, if you support pin configuration in
 *	your driver
 * @owner: module providing the pin controller, used for refcounting
 * @num_custom_params: Number of driver-specific custom parameters to be parsed
 *	from the hardware description
 * @custom_params: List of driver_specific custom parameters to be parsed from
 *	the hardware description
 * @custom_conf_items: Information how to print @params in debugfs, must be
 *	the same size as the @custom_params, i.e. @num_custom_params
 * @link_consumers: If true create a device link between pinctrl and its
 *	consumers (i.e. the devices requesting pin control states). This is
 *	sometimes necessary to ascertain the right suspend/resume order for
 *	example.
 */
struct pinctrl_desc {
	const char *name;
	const struct pinctrl_pin_desc *pins;
	unsigned int npins;
	const struct pinctrl_ops *pctlops;
	const struct pinmux_ops *pmxops;
	const struct pinconf_ops *confops;
	struct module *owner;
#ifdef CONFIG_GENERIC_PINCONF
	unsigned int num_custom_params;
	const struct pinconf_generic_params *custom_params;
	const struct pin_config_item *custom_conf_items;
#endif
	bool link_consumers;
};
```
pin controller描述符需要描述它可以控制多少个pin（成员npins），每一个pin的信息为何？（成员pins）。这两个成员就确定了一个pin controller所能控制的引脚的信息。

pin controller描述符中包括了三类操作函数：pctlops是一些全局的控制函数，pmxops是复用引脚相关的操作函数，confops操作函数是用来配置引脚的特性（例如：pull-up/down）。
### struct pinctrl_ops
struct pinctrl_ops 中各个callback函数的具体的解释如下：

|   |   |
|---|---|
|callback函数|描述|
|get_groups_count|该pin controller支持多少个pin group。pin group的定义可以参考本文关于pin controller的功能规格中的描述。注意不要把pin group和IO port的硬件分组搞混了。例如：S3C2416有138个I/O 端口，分成11组，分别是gpa～gpl，这个组并不叫pin group，而是叫做pin bank。pin group是和特定功能（例如SPI、I2C）相关的一组pin。|
|get_group_name|给定一个selector（index），获取指定pin group的name|
|get_group_pins|给定一个selector（index），获取该pin group中pin的信息（该pin group包括多少个pin，每个pin的ID是什么）|
|pin_dbg_show|debug fs的callback接口|
|dt_node_to_map|分析一个pin configuration node并把分析的结果保存成mapping table entry，每一个entry表示一个setting（一个功能复用设定，或者电气特性设定）|
|dt_free_map|上面函数的逆函数|
### 复用引脚
复用引脚相关的操作函数的具体解释如下：

|   |   |
|---|---|
|call back函数|描述|
|request|pin control core进行具体的复用设定之前需要调用该函数，主要是用来请底层的driver判断某个引脚的复用设定是否是OK的。|
|free|是request的逆函数。调用request函数请求占用了某些pin的资源，调用free可以释放这些资源|
|get_functions_count|就是返回pin controller支持的function的数目|
|get_function_name|给定一个selector（index），获取指定function的name|
|get_function_groups|给定一个selector（index），获取指定function的pin groups信息|
|enable|enable一个function。当然要给出function selector和pin group的selector|
|disable|enable的逆函数|
|gpio_request_enable|request并且enable一个单独的gpio pin|
|gpio_disable_free|gpio_request_enable的逆函数|
|gpio_set_direction|设定GPIO方向的回调函数|
### 配置引脚
配置引脚的特性的struct pinconf_ops数据结构的各个成员定义如下：

|   |   |
|---|---|
|call back函数|描述|
|pin_config_get|给定一个pin ID以及config type ID，获取该引脚上指定type的配置。|
|pin_config_set|设定一个指定pin的配置|
|pin_config_group_get|以pin group为单位，获取pin上的配置信息|
|pin_config_group_set|以pin group为单位，设定pin group的特性配置|
|pin_config_dbg_parse_modify|debug接口|
|pin_config_dbg_show|debug接口|
|pin_config_group_dbg_show|debug接口|
|pin_config_config_dbg_show|debug接口|

# pinctrl驱动的理解和总结

## 一、前言
介绍了pin controller（对应的pin controller subsystem）、gpio controller（对应的GPIO subsystem）有关的基本概念，包括pin multiplexing、pin configuration等等。本文将基于这些文章，单纯地从pin controller driver的角度（屏蔽掉pinctrl core的实现细节），理解pinctrl subsystem的设计思想，并掌握pinctrl驱动的移植和实现方法。

## 二、pin controller的概念和软件抽象

相信每一个嵌入式从业人员，都知道“pin（管脚）”是什么东西（就不赘述了）。由于SoC系统越来越复杂、集成度越来越高，SoC中pin的数量也越来越多、功能也越来越复杂，这就对如何管理、使用这些pins提出了挑战。因此，用于管理这些pins的硬件模块（pin controller）就出现了。相应地，linux kernel也出现了对应的驱动（pin controller driver）。

Kernel pinctrl core使用struct pinctrl_desc抽象一个pin controller，该结构的定义如下（先贴在这里，后面会围绕这个抽象一步步展开）：
```c
struct pinctrl_desc {   
        const char *name;   
        const struct pinctrl_pin_desc *pins;   
        unsigned int npins;   
        const struct pinctrl_ops *pctlops;   
        const struct pinmux_ops *pmxops;   
        const struct pinconf_ops *confops;   
        struct module *owner;   
#ifdef CONFIG_GENERIC_PINCONF   
        unsigned int num_custom_params;   
        const struct pinconf_generic_params *custom_params;   
        const struct pin_config_item *custom_conf_items;   
#endif   
```
### 1 Pin
kernel的pin controller子系统要想管理好系统的pin资源，第一个要搞明白的问题就是：系统中到底有多少个pin？用软件语言来表述就是：要把系统中所有的pin描述出来，并建立索引。这由上面struct pinctrl_desc结构中pins和npins来完成。

对pinctrl core来说，它只关心系统中有多少个pin，并使用自然数为这些pin编号，后续的操作，都是以这些编号为操作对象。至于编号怎样和具体的pin对应上，完全是pinctrl driver自己的事情。

因此，pinctrl driver需要根据实际情况，将系统中所有的pin组织成一个struct pinctrl_pin_desc类型的数组，该类型的定义为：
```c
/**   
* struct pinctrl_pin_desc - boards/machines provide information on their   
  * pins, pads or other muxable units in this struct   
  * @number: unique pin number from the global pin number space   
  * @name: a name for this pin   
  * @drv_data: driver-defined per-pin data. pinctrl core does not touch this   
  */   
struct pinctrl_pin_desc {   
        unsigned number;   
        const char *name;   
        void *drv_data;   
};
```

number和name完全由driver自己决定，不过要遵循有利于代码编写、有利于理解等原则。另外，为了便于driver的编写，可以在drv_data中保存driver的私有数据结构（可以包含相关的寄存器偏移等信息）。
### 2 Pin groups
在SoC系统中，有时需要将很多pin组合在一起，以实现特定的功能，例如SPI接口、I2C接口等。因此pin controller需要以group为单位，访问、控制多个pin，这就是pin groups。相应地，pin controller subsystem需要提供一些机制，来获取系统中到底有多少groups、每个groups包含哪些pins、等等。

因此，pinctrl core在struct pinctrl_ops中抽象出三个回调函数，用来获取pin groups相关信息，如下：
```c
struct pinctrl_ops {   
        int (*get_groups_count) (struct pinctrl_dev *pctldev);   
        const char *(*get_group_name) (struct pinctrl_dev *pctldev,   
                                        unsigned selector);   
        int (*get_group_pins) (struct pinctrl_dev *pctldev,   
                               unsigned selector,   
                               const unsigned **pins,   
                               unsigned *num_pins);   
        void (*pin_dbg_show) (struct pinctrl_dev *pctldev, struct seq_file *s,   
                          unsigned offset);   
        int (*dt_node_to_map) (struct pinctrl_dev *pctldev,   
                               struct device_node *np_config,   
                               struct pinctrl_map **map, unsigned *num_maps);   
        void (*dt_free_map) (struct pinctrl_dev *pctldev,   
                             struct pinctrl_map *map, unsigned num_maps);   
};
```
- get_groups_count，获取系统中pin groups的个数，后续的操作，将以相应的索引为单位（类似数组的下标，个数为数组的大小）。
- get_group_name，获取指定group（由索引selector指定）的名称。
- get_group_pins，获取指定group的所有pins（由索引selector指定），结果保存在pins（指针数组）和num_pins（指针）中。

当然，最终的group信息是由pinctrl driver提供的，至于driver怎么组织这些group，那是driver自己的事情了。  

### 3 Pin configuration
2.1和2.2中介绍了pinctrl subsystem中的操作对象（pin or pin group）以及抽象方法。嵌入式系统的工程师都知道，SoC中的管脚有些属性可以配置，例如上拉、下拉、高阻、驱动能力等。pinctrl subsystem使用pin configuration来封装这些功能，具体体现在struct pinconf_ops数据结构中，如下：
```c
struct pinconf_ops {   
#ifdef CONFIG_GENERIC_PINCONF   
         bool is_generic;   
#endif   
        int (*pin_config_get) (struct pinctrl_dev *pctldev,   
                               unsigned pin,   
                               unsigned long *config);   
        int (*pin_config_set) (struct pinctrl_dev *pctldev,   
                               unsigned pin,   
                                unsigned long *configs,   
                                unsigned num_configs);   
        int (*pin_config_group_get) (struct pinctrl_dev *pctldev,   
                                      unsigned selector,   
                                      unsigned long *config);   
        int (*pin_config_group_set) (struct pinctrl_dev *pctldev,   
                                      unsigned selector,   
                                      unsigned long *configs,   
                                     unsigned num_configs);   
        int (*pin_config_dbg_parse_modify) (struct pinctrl_dev *pctldev,   
                                            const char *arg,   
                                            unsigned long *config);   
        void (*pin_config_dbg_show) (struct pinctrl_dev *pctldev,   
                                      struct seq_file *s,   
                                      unsigned offset);   
        void (*pin_config_group_dbg_show) (struct pinctrl_dev *pctldev,   
                                            struct seq_file *s,   
                                            unsigned selector);   
        void (*pin_config_config_dbg_show) (struct pinctrl_dev *
```
- pin_config_get，获取指定pin（管脚的编号）当前配置，保存在config指针中（配置的具体含义，只有pinctrl driver自己知道，下同）。
- pin_config_set，设置指定pin的配置（可以同时配置多个config，具体意义要由相应pinctrl driver解释）。
- pin_config_group_get、pin_config_group_set，获取或者设置指定pin group的配置项。

kernel pinctrl subsystem并不关心configuration的具体内容是什么，它只提供pin configuration get/set的通用机制，至于get到的东西，以及set的东西，到底是什么，是pinctrl driver自己的事情。

### 4 Pin multiplexing
为了照顾不同类型的产品、不同的应用场景，SoC中的很多管脚可以配置为不同的功能，例如A2和B5两个管脚，既可以当作普通的GPIO使用，又可以配置为I2C0的的SCL和SDA，也可以配置为UART5的TX和RX，这称作管脚的复用（pin multiplexing，简称为pinmux）。kernel pinctrl subsystem使用struct pinmux_ops来抽象pinmux有关的操作，如下：
```c
struct pinmux_ops {   
         int (*request) (struct pinctrl_dev *pctldev, unsigned offset);   
        int (*free) (struct pinctrl_dev *pctldev, unsigned offset);   
        int (*get_functions_count) (struct pinctrl_dev *pctldev);   
        const char *(*get_function_name) (struct pinctrl_dev *pctldev,   
                                           unsigned selector);   
        int (*get_function_groups) (struct pinctrl_dev *pctldev,   
                                  unsigned selector,   
                                  const char * const **groups,   
                                  unsigned *num_groups);   
        int (*set_mux) (struct pinctrl_dev *pctldev, unsigned func_selector,   
                        unsigned group_selector);   
        int (*gpio_request_enable) (struct pinctrl_dev *pctldev,   
                                    struct pinctrl_gpio_range *range,   
                                     unsigned offset);   
        void (*gpio_disable_free) (struct pinctrl_dev *pctldev,   
                                   struct pinctrl_gpio_range *range,   
                                    unsigned offset);   
        int (*gpio_set_direction) (struct pinctrl_dev *pctldev,   
                                   struct pinctrl_gpio_range *range,   
                                    unsigned offset,   
                                   bool input);   
   
```
- 为了管理SoC的管脚复用，pinctrl subsystem抽象出function的概念，用来表示I2C0、UART5等功能。pin（或者pin group）所对应的function一经确定，它（们）的管脚复用状态也就确定了
- pinctrl core不关心function的具体形态，只要求pinctrl driver将SoC的所有可能的function枚举出来（格式自行定义，不过需要有编号、名称等内容，可参考[5]中的例子），并注册给pinctrl core。后续pinctrl core将会通过function的索引，访问、控制相应的function
- 另外，有一点大家应该很熟悉：在SoC的设计中，同一个function（如I2C0），可能可以map到不同的pin（或者pin group）上

理解了function的概念之后，struct pinmux_ops中的API就简单了：
- get_functions_count，获取系统中function的个数。
- get_function_name，获取指定function的名称。
- get_function_groups，获取指定function所占用的pin group（可以有多个）。
- set_mux，将指定的pin group（group_selector）设置为指定的function（func_selector）。
- request，检查某个pin是否已作它用，用于管脚复用时的互斥（避免多个功能同时使用某个pin而不知道，导致奇怪的错误）。
- free，request的反操作。
- gpio_request_enable、gpio_disable_free、gpio_set_direction，gpio有关的操作。
- strict，为true时，说明该pin controller不允许某个pin作为gpio和其它功能同时使用。

## 三、pinctrl subsystem的控制逻辑
之前介绍了pinctrl subsystem中有关pin controller的概念抽象，包括pin、pin group、pinconf、pinmux、pinmux function、等等，相当于从provider的角度理解pinctrl subsystem。那么，问题来了，怎么使用pinctrl subsystem提供的功能控制管脚的配置以及功能复用呢？这看似需要由consumer（例如各个外设的驱动）自行处理，实际上却不尽然：
- 由于pinctrl subsystem的特殊性，对于pin configuration以及pin multiplexing而言，要怎么配置、怎么复用，只有pinctrl driver自己知道。同理，各个consumer也是云里雾里，啥都搞不清楚。
- 对一个确定的产品来说，某个设备所使用的pinctrl功能（function）、以及所对应的pin（或者pin group）、还有这些pin（或者pin group）的属性配置，基本上在产品设计的时候就确定好了，consumer没必要关心技术细节。因此pinctrl driver就要多做一些事情，帮助consumer厘清pin有关资源的使用情况，并在这些设备需要使用的时候（例如probe时），一声令下，将资源准备好。
- pinctrl subsystem的设计理念就是：不需要consumer关心pin controller的技术细节，只需要在特定的时候向pinctrl driver发出一些简单的指令，告诉pinctrl driver自己的需求即可（例如我在运行时需要使用这样一组配置，在休眠时使用那样一组配置）。
- 最后，需求的细节（例如需要使用哪些pin、配置为什么功能、等等），要怎么确定呢？一般是通过machine的配置文件、具体版型的device tree等，告诉pinctrl subsystem，以便在需要的时候使用。
### 1 pin state
根据前面的描述，pinctrl driver抽象出来了一些离散的对象：pin（pin group）、function、configuration，并实现了这些对象的控制和配置方式。

pin（pin group）以及相应的function和configuration的组合，可以确定一个设备的一个“状态”。

这个状态在pinctrl subsystem中就称作pin state。而pinctrl driver和具体板型有关的部分，需要负责枚举该板型下所有device（当然，特指那些需要pin资源的device）的所有可能的状态，并详细定义这些状态需要使用的pin（或pin group），以及这些pin（或pin group）需要配置为哪种function、哪种配置项。这些状态确定之后，consumer（device driver）就好办了，直接发号施令给pinctrl subsystem，帮忙将我的xxx state激活。

pinctrl subsystem接收到指令后，找到该state的相关信息（pin、function和configuration），并调用pinctrl driver提供的相应API，控制pin controller即可。

### 2 pin map
在pinctrl subsystem中，pin state有关的信息是通过pin map收集，相关的数据结构如下：
```c
struct pinctrl_map {   
        const char *dev_name;   
        const char *name;   
        enum pinctrl_map_type type;   
        const char *ctrl_dev_name;   
        union {   
                struct pinctrl_map_mux mux;   
                 struct pinctrl_map_configs configs;   
   
```
- dev_name，device的名称。
- name，pin state的名称。
- ctrl_dev_name，pin controller device的名字。
- type，该map的类型，包括PIN_MAP_TYPE_MUX_GROUP（配置管脚复用）、PIN_MAP_TYPE_CONFIGS_PIN（配置pin）、PIN_MAP_TYPE_CONFIGS_GROUP（配置pin group）、PIN_MAP_TYPE_DUMMY_STATE（不需要任何配置，仅仅为了表示state的存在。
- data，该map需要用到的数据项，是一个联合体，如果map的类型是PIN_MAP_TYPE_CONFIGS_GROUP，则为struct pinctrl_map_mux类型的变量；如果map的类型是PIN_MAP_TYPE_CONFIGS_PIN或者PIN_MAP_TYPE_CONFIGS_GROUP，则为struct pinctrl_map_configs类型的变量。

struct pinctrl_map_mux的定义如下：
```c
struct pinctrl_map_mux {   
         const char *group;   
        const char *function;
};
```
- group，group的名字，指明该map所涉及的pin group。
- function，function的名字，表示该map需要将group配置为哪种function。

struct pinctrl_map_configs的定义如下：
```c
struct pinctrl_map_configs {   
        const char *group_or_pin;   
        unsigned long *configs;   
        unsigned num_configs;   
};
```
- group_or_pin，pin或者pin group的名字。
- configs，configuration数组，指明要将该group_or_pin配置成“神马样子”。
- num_configs，配置项的个数。

最后，某一个device的某一种pin state，可以由多个不同类型的map entry组合而成，举例如下：
```c
static struct pinctrl_map mapping[] __initdata = {   
        PIN_MAP_MUX_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0", "i2c0"),   
        PIN_MAP_CONFIGS_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0", i2c_grp_configs),   
        PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0scl", i2c_pin_configs),   
        PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT, "pinctrl-foo", "i2c0sda", i2c_pin_configs),   
};
```
这是一个mapping数组，包含4个map entry，定义了"foo-i2c.0"设备的一个pin state（PINCTRL_STATE_DEFAULT，"default"），该state由一个PIN_MAP_TYPE_MUX_GROUP entry、一个PIN_MAP_TYPE_CONFIGS_GROUP entry以及两个PIN_MAP_TYPE_CONFIGS_PIN entry组成。

### 3 通过dts生成pin map
在旧时代，kernel的bsp工程师需要在machine有关的代码中，静态的定义pin map数组（类似于3.2小节中的例子），这一个非常繁琐且不易维护的过程。不过当kernel引入device tree之后，事情就简单了很多：
- pinctrl driver确定了pin map各个字段的格式之后，就可以在dts文件中维护pin state以及相应的mapping table。pinctrl core在初始化的时候，会读取并解析dts，并生成pin map。
- 而各个consumer，可以在自己的dts node中，直接引用pinctrl driver定义的pin state，并在设备驱动的相应的位置，调用pinctrl subsystem提供的API，active或者deactive这些state。

至于dts中pin map描述的格式是什么，则完全由pinctrl driver自己决定，因为，最终的解析工作（dts to map）也是它自己做的。

## 四、pinctrl subsystem的整体流程
通过前面几章的分析，我们对pinctrl subsystem有了一个比较全面的认识，这里以pinctrl整个使用流程为例，简单的总结一下。
- pinctrl driver根据pin controller的实际情况，实现struct pinctrl_desc（包括pin/pin group的抽象，function的抽象，pinconf、pinmux的operation API实现，dt_node_to_map的实现，等等），并注册到kernel中。
- pinctrl driver在pin controller的dts node中，根据自己定义的格式，描述每个device的所有pin state。大致的形式如下（具体可参考kernel中的代码，照葫芦总能画出来瓢。
```c

&iomuxc {
	pinctrl-names = "default";

	pinctrl_camera_clock: cameraclockgrp {
		fsl,pins = <
			MX6UL_PAD_CSI_MCLK__CSI_MCLK		0x1b088
		>;
	};

	pinctrl_csi1: csi1grp {
		fsl,pins = <
			MX6UL_PAD_CSI_PIXCLK__CSI_PIXCLK	0x1b088
			MX6UL_PAD_CSI_VSYNC__CSI_VSYNC		0x1b088
			MX6UL_PAD_CSI_HSYNC__CSI_HSYNC		0x1b088
			MX6UL_PAD_CSI_DATA00__CSI_DATA02	0x1b088
			MX6UL_PAD_CSI_DATA01__CSI_DATA03	0x1b088
			MX6UL_PAD_CSI_DATA02__CSI_DATA04	0x1b088
			MX6UL_PAD_CSI_DATA03__CSI_DATA05	0x1b088
			MX6UL_PAD_CSI_DATA04__CSI_DATA06	0x1b088
			MX6UL_PAD_CSI_DATA05__CSI_DATA07	0x1b088
			MX6UL_PAD_CSI_DATA06__CSI_DATA08	0x1b088
			MX6UL_PAD_CSI_DATA07__CSI_DATA09	0x1b088
		>;
	};
	// ......
};

```

- 相应的consumer driver可以在自己的dts node中，引用pinctrl driver所定义的pin state，例如：
```c
usr_led {
    //.....                     
    pinctrl-names = "default", "improve";          //用于pinctrl查找state的名称，默认是default
    pinctrl-0 = <&pinctrl_gpio_led>;               //第一组引脚复用功能指定的结构
    pinctrl-1 =<&pinctrl_led_improve>;             //第二组引脚复用功能指定的结构
};
```
- consumer driver在需要的时候，可以调用pinctrl_get/devm_pinctrl_get接口，获得一个pinctrl handle（struct pinctrl类型的指针）。pinctrl subsystem在pinctrl get的过程中，解析consumer device的dts node，找到相应的pin state，进行调用pinctrl driver提供的dt_node_to_map API，解析pin state并转换为pin map。以driver probe时为例，调用过程如下：
```c
probe   
    devm_pinctrl_get or pinctrl_get   
        create_pinctrl(drivers/pinctrl/core.c)   
            pinctrl_dt_to_map(drivers/pinctrl/devicetree.c)   
                dt_to_map_one_config   
                    pctlops->dt_node_to_map
```
- consumer获得pinctrl handle之后，可以调用pinctrl subsystem提供的API（例如pinctrl_select_state），使自己的某个pin state生效。pinctrl subsystem进而调用pinctrl driver提供的各种回调函数，配置pin controller的硬件。   

