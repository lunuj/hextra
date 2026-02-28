---
title: Linux pinctrl 子系统 2 pinctrl 驱动
date: 2026-02-19
authors:
  - name: lunuj
    link: https://github.com/lunuj
tags:
  - 2025
  - Linux
  - pinctrl
---
**Linux** 内核 **pinctrl** 系统驱动框架。
<!--more-->  

## 一、前言
对于一个嵌入式软件工程师，软件模块经常和硬件打交道，pin control subsystem也不例外，被它驱动的硬件叫做pin controller，主要功能包括：
1. pin multiplexing。基于ARM core的嵌入式处理器一般会提供丰富的功能，例如camera interface、LCD interface、USB、I2C、SPI等等。虽然处理器有几百个pin，但是这些pin还是不够分配，因此有些pin需要复用。例如：127号GPIO可以做一个普通的GPIO控制LED，也可以配置成I2C的clock信号，也可以配置成SPI的data out信号。当然，这些功能不可能同时存在，因为硬件信号只有一个。
2. pin configuration。这些配置参数包括：pull-up/down电阻的设定， tri-state设定，drive-strength的设定。
3. enumerating and naming。引脚的命名和管理。

device tree的相关内容等待后续补充。

## 二、pinctrl DTS
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

## 三、 pinctrl driver
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
