---
title: Linux GPIO子系统
date: 2026-02-28
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


## 一、GPIO 介绍
“通用输入/输出”(GPIO) 是一种灵活的软件控制数字信号。 它们由多种芯片提供，并且嵌入式和定制硬件开发的 Linux 开发人员对此很熟悉。 每个 GPIO 代表连接到特定引脚或球栅阵列 (BGA) 封装上的“球”的一位。 电路板原理图显示了哪些外部硬件连接到哪些 GPIO。 可以通用地编写驱动程序，以便电路板设置代码将此类引脚配置数据传递给驱动程序。

片上系统 (SOC) 处理器严重依赖 GPIO。 在某些情况下，每个非专用引脚都可以配置为 GPIO； 大多数芯片至少有几十个。 可编程逻辑设备（如 FPGA）可以轻松提供 GPIO； 多功能芯片（如电源管理器和音频编解码器）通常有一些此类引脚，以帮助解决 SOC 上的引脚稀缺问题； 并且还有使用 I2C 或 SPI 串行总线连接的“GPIO 扩展器”芯片。 大多数 PC 南桥都有几十个支持 GPIO 的引脚（只有 BIOS 固件知道它们是如何使用的）。

GPIO 的确切功能因系统而异。 常见选项
- 输出值可写（高=1，低=0）。 有些芯片还具有关于如何驱动该值的选项，因此例如，可能只驱动一个值，从而支持“线或”以及类似的其他值方案（特别是“开漏”信令）。
- 输入值同样是可读的（1, 0）。 有些芯片支持回读配置为“输出”的引脚，这在“线或”情况下非常有用（以支持双向信令）。 GPIO 控制器可能具有输入去毛刺/防抖动逻辑，有时具有软件控制。
- 输入通常可以用作 IRQ 信号，通常是边沿触发，但有时是电平触发。 此类 IRQ 可以配置为系统唤醒事件，以将系统从低功耗状态唤醒。
- 通常，GPIO 可以根据不同产品板的需要配置为输入或输出； 也存在单向 GPIO。
- 大多数 GPIO 可以在持有自旋锁时访问，但通过串行总线访问的 GPIO 通常无法访问。 有些系统同时支持这两种类型。

在给定的板上，每个 GPIO 都用于一个特定目的，例如监视 MMC/SD 卡的插入/移除、检测卡写保护状态、驱动 LED、配置收发器、位敲击串行总线、探测硬件看门狗、感应开关等等。

## 二、 软件框架
不同芯片厂家对 GPIO 的实现驱动都各有不同，为了提供统一接口给其他驱动使用，Linux 中使用 GPIO lib 屏蔽底层硬件差异，来实现统一管理。
```txt
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Deivcer0    │  │ Deivcer1    │  │ Deivcer2    │  │ Deivcer3    │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
┌──────┴────────────────┴────────────────┴────────────────┴──────┐
│                            GPIO Lib                            │
└──────┬────────────────┬────────────────┬────────────────┬──────┘
┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
│ platform    │  │ platform    │  │ platform    │  │ platform    │
│ GPIO driver0│  │ GPIO driver1│  │ GPIO driver2│  │ GPIO driver3│
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
│ platform    │  │ platform    │  │ platform    │  │ platform    │
│ GPIO device0│  │ GPIO device1│  │ GPIO device2│  │ GPIO device3│
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```
1. Deivcer：使用 GPIO lib 中提供的函数接口，对 GPIO 进行申请配置。
2. GPIO Lib：为上层驱动提供接口，并为下层提供 GPIO chip 结构体。
3. platform GPIO driver：底层的特定平台驱动，初始化底层硬件，注册 GPIO chip 到 GPIO Lib 适配层。
4. platform GPIO device：底层 GPIO 硬件。

## 三、GPIO 驱动过程
作为驱动开发人员，主要关注子系统提供的上层接口，对于SOC片上外设的驱动，了解大致流程就好，不用过于关注底层细节。

### 1 initcall

linux系统初始化分为几个阶段，在不同阶段调用不同的函数，这些就是驱动和设备创建的基础。

```c
/*
 * initcalls are now grouped by functionality into separate
 * subsections. Ordering inside the subsections is determined
 * by link order. 
 * For backwards compatibility, initcall() puts the call in 
 * the device init subsection.
 *
 * The `id' arg to __define_initcall() is needed so that multiple initcalls
 * can point at the same handler without causing duplicate-symbol build errors.
 *
 * Initcalls are run by placing pointers in initcall sections that the
 * kernel iterates at runtime. The linker can do dead code / data elimination
 * and remove that completely, so the initcall sections have to be marked
 * as KEEP() in the linker script.
 */

/* Format: <modname>__<counter>_<line>_<fn> */
#define __initcall_id(fn)					\
	__PASTE(__KBUILD_MODNAME,				\
	__PASTE(__,						\
	__PASTE(__COUNTER__,					\
	__PASTE(_,						\
	__PASTE(__LINE__,					\
	__PASTE(_, fn))))))

/* Format: __<prefix>__<iid><id> */
#define __initcall_name(prefix, __iid, id)			\
	__PASTE(__,						\
	__PASTE(prefix,						\
	__PASTE(__,						\
	__PASTE(__iid, id))))

#ifdef CONFIG_LTO_CLANG
/*
 * With LTO, the compiler doesn't necessarily obey link order for
 * initcalls. In order to preserve the correct order, we add each
 * variable into its own section and generate a linker script (in
 * scripts/link-vmlinux.sh) to specify the order of the sections.
 */
#define __initcall_section(__sec, __iid)			\
	#__sec ".init.." #__iid

/*
 * With LTO, the compiler can rename static functions to avoid
 * global naming collisions. We use a global stub function for
 * initcalls to create a stable symbol name whose address can be
 * taken in inline assembly when PREL32 relocations are used.
 */
#define __initcall_stub(fn, __iid, id)				\
	__initcall_name(initstub, __iid, id)

#define __define_initcall_stub(__stub, fn)			\
	int __init __cficanonical __stub(void);			\
	int __init __cficanonical __stub(void)			\
	{ 							\
		return fn();					\
	}							\
	__ADDRESSABLE(__stub)
#else
#define __initcall_section(__sec, __iid)			\
	#__sec ".init"

#define __initcall_stub(fn, __iid, id)	fn

#define __define_initcall_stub(__stub, fn)			\
	__ADDRESSABLE(fn)
#endif

#ifdef CONFIG_HAVE_ARCH_PREL32_RELOCATIONS
#define ____define_initcall(fn, __stub, __name, __sec)		\
	__define_initcall_stub(__stub, fn)			\
	asm(".section	\"" __sec "\", \"a\"		\n"	\
	    __stringify(__name) ":			\n"	\
	    ".long	" __stringify(__stub) " - .	\n"	\
	    ".previous					\n");
#else
#define ____define_initcall(fn, __unused, __name, __sec)	\
	static initcall_t __name __used 			\
		__attribute__((__section__(__sec))) = fn;
#endif

#define __unique_initcall(fn, id, __sec, __iid)			\
	____define_initcall(fn,					\
		__initcall_stub(fn, __iid, id),			\
		__initcall_name(initcall, __iid, id),		\
		__initcall_section(__sec, __iid))

#define ___define_initcall(fn, id, __sec)			\
	__unique_initcall(fn, id, __sec, __initcall_id(fn))

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

/*
 * Early initcalls run before initializing SMP.
 *
 * Only for built-in code, not modules.
 */
#define early_initcall(fn)		__define_initcall(fn, early)

/*
 * A "pure" initcall has no dependencies on anything else, and purely
 * initializes variables that couldn't be statically initialized.
 *
 * This only exists for built-in code, not for modules.
 * Keep main.c:initcall_level_names[] in sync.
 */
#define pure_initcall(fn)		__define_initcall(fn, 0)

#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)

#define __initcall(fn) device_initcall(fn)

#define __exitcall(fn)						\
	static exitcall_t __exitcall_##fn __exit_call = fn

#define console_initcall(fn)	___define_initcall(fn, con, .con_initcall)

```

以platform为例，讲解
```c
// kernel/linux-5.10/drivers/of/platform.c
static int __init of_platform_default_populate_init(void)
{
	struct device_node *node;

	device_links_supplier_sync_state_pause();

	if (!of_have_populated_dt())
		return -ENODEV;

	/*
	 * Handle certain compatibles explicitly, since we don't want to create
	 * platform_devices for every node in /reserved-memory with a
	 * "compatible",
	 */
	for_each_matching_node(node, reserved_mem_matches)
		of_platform_device_create(node, NULL, NULL);

	node = of_find_node_by_path("/firmware");
	if (node) {
		of_platform_populate(node, NULL, NULL, NULL);
		of_node_put(node);
	}

	/* Populate everything else. */
	of_platform_default_populate(NULL, NULL, NULL);

	return 0;
}
arch_initcall_sync(of_platform_default_populate_init);
```

arch init 初始化阶段会调用 of_platform_default_populate_init 函数。

### 2 添加设备
这个函数将设备树中根节点的子设备描述为 platform device，并调用 device_add 将其添加到 platform bus中。

device_add 函数会调用 driver_match_device 匹配驱动，调用 device_driver_attach 匹配设备，匹配成功会调用驱动对应的 probe 函数。

有些平台的设备树中，片上外设子节点是挂载在 soc/aips1 节点下，而不是根节点。对于这种情况，有些节点指定了 simple-bus 属性。
```c
const struct of_device_id of_default_bus_match_table[] = {
	{ .compatible = "simple-bus", },
	{ .compatible = "simple-mfd", },
	{ .compatible = "isa", },
#ifdef CONFIG_ARM_AMBA
	{ .compatible = "arm,amba-bus", },
#endif /* CONFIG_ARM_AMBA */
	{} /* Empty terminated list */
};

int of_platform_default_populate(struct device_node *root,
				 const struct of_dev_auxdata *lookup,
				 struct device *parent)
{
	return of_platform_populate(root, of_default_bus_match_table, lookup,
				    parent);
}
EXPORT_SYMBOL_GPL(of_platform_default_populate);


/**
 * of_platform_populate() - Populate platform_devices from device tree data
 * @root: parent of the first level to probe or NULL for the root of the tree
 * @matches: match table, NULL to use the default
 * @lookup: auxdata table for matching id and platform_data with device nodes
 * @parent: parent to hook devices from, NULL for toplevel
 *
 * Similar to of_platform_bus_probe(), this function walks the device tree
 * and creates devices from nodes.  It differs in that it follows the modern
 * convention of requiring all device nodes to have a 'compatible' property,
 * and it is suitable for creating devices which are children of the root
 * node (of_platform_bus_probe will only create children of the root which
 * are selected by the @matches argument).
 *
 * New board support should be using this function instead of
 * of_platform_bus_probe().
 *
 * Returns 0 on success, < 0 on failure.
 */
int of_platform_populate(struct device_node *root,
			const struct of_device_id *matches,
			const struct of_dev_auxdata *lookup,
			struct device *parent)
{
	struct device_node *child;
	int rc = 0;

	root = root ? of_node_get(root) : of_find_node_by_path("/");
	if (!root)
		return -EINVAL;

	pr_debug("%s()\n", __func__);
	pr_debug(" starting at: %pOF\n", root);

	device_links_supplier_sync_state_pause();
	for_each_child_of_node(root, child) {
		rc = of_platform_bus_create(child, matches, lookup, parent, true);
		if (rc) {
			of_node_put(child);
			break;
		}
	}
	device_links_supplier_sync_state_resume();

	of_node_set_flag(root, OF_POPULATED_BUS);

	of_node_put(root);
	return rc;
}
EXPORT_SYMBOL_GPL(of_platform_populate);
```

of_default_bus_match_table 这个表决定哪些节点被当成 bus 继续递归创建子设备。
```c
/**
 * of_platform_bus_create() - Create a device for a node and its children.
 * @bus: device node of the bus to instantiate
 * @matches: match table for bus nodes
 * @lookup: auxdata table for matching id and platform_data with device nodes
 * @parent: parent for new device, or NULL for top level.
 * @strict: require compatible property
 *
 * Creates a platform_device for the provided device_node, and optionally
 * recursively create devices for all the child nodes.
 */
static int of_platform_bus_create(struct device_node *bus,
				  const struct of_device_id *matches,
				  const struct of_dev_auxdata *lookup,
				  struct device *parent, bool strict)
{
	const struct of_dev_auxdata *auxdata;
	struct device_node *child;
	struct platform_device *dev;
	const char *bus_id = NULL;
	void *platform_data = NULL;
	int rc = 0;

	/* Make sure it has a compatible property */
	if (strict && (!of_get_property(bus, "compatible", NULL))) {
		pr_debug("%s() - skipping %pOF, no compatible prop\n",
			 __func__, bus);
		return 0;
	}

	/* Skip nodes for which we don't want to create devices */
	if (unlikely(of_match_node(of_skipped_node_table, bus))) {
		pr_debug("%s() - skipping %pOF node\n", __func__, bus);
		return 0;
	}

	if (of_node_check_flag(bus, OF_POPULATED_BUS)) {
		pr_debug("%s() - skipping %pOF, already populated\n",
			__func__, bus);
		return 0;
	}

	auxdata = of_dev_lookup(lookup, bus);
	if (auxdata) {
		bus_id = auxdata->name;
		platform_data = auxdata->platform_data;
	}

	if (of_device_is_compatible(bus, "arm,primecell")) {
		/*
		 * Don't return an error here to keep compatibility with older
		 * device tree files.
		 */
		of_amba_device_create(bus, bus_id, platform_data, parent);
		return 0;
	}

	dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
	if (!dev || !of_match_node(matches, bus))
		return 0;

	for_each_child_of_node(bus, child) {
		pr_debug("   create child: %pOF\n", child);
		rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
		if (rc) {
			of_node_put(child);
			break;
		}
	}
	of_node_set_flag(bus, OF_POPULATED_BUS);
	return rc;
}
```

就这样，所有设备树的平台设备都创建了对应的platform_device，并添加到了platform_bus当中，具体函数调用链如下：
```text
of_platform_default_populate_init
of_platform_default_populate
of_platform_populate
of_platform_bus_create
of_platform_device_create_pdata
of_device_alloc
​	platform_device_alloc
​		device_initialize
of_device_add
​	device_add
​		bus_probe_device
​			device_initial_probe
​				__device_attach
​					__device_attach_driver
​						driver_match_device
​						driver_probe_device
```

### 3 注册驱动
说完了device，接下来该driver了，以spi驱动为例，ecspi1是 /soc/aips1/spba-bus 节点下的子节点，可以看到父亲节点都具备 compatible = "simple-bus"，所以会被创建对应的 platform device，加入到platform bus中。

```c
/ {
    soc {
		#address-cells = <1>;
		#size-cells = <1>;
		compatible = "simple-bus";
		interrupt-parent = <&gpc>;
		ranges;
        aips1: aips-bus@02000000 {
			compatible = "fsl,aips-bus", "simple-bus";
			#address-cells = <1>;
			#size-cells = <1>;
			reg = <0x02000000 0x100000>;
			ranges;

			spba-bus@02000000 {
				compatible = "fsl,spba-bus", "simple-bus";
				#address-cells = <1>;
				#size-cells = <1>;
				reg = <0x02000000 0x40000>;
				ranges;

				ecspi1: ecspi@02008000 {
					#address-cells = <1>;
					#size-cells = <0>;
					compatible = "fsl,imx6ul-ecspi", "fsl,imx51-ecspi";
					reg = <0x02008000 0x4000>;
					interrupts = <GIC_SPI 31 IRQ_TYPE_LEVEL_HIGH>;
					clocks = <&clks IMX6UL_CLK_ECSPI1>,
						 <&clks IMX6UL_CLK_ECSPI1>;
					clock-names = "ipg", "per";
					status = "disabled";
				};
           	};
        };
    };
}
```

在对应的驱动当中，

```c
static const struct of_device_id spi_imx_dt_ids[] = {
	{ .compatible = "fsl,imx1-cspi", .data = &imx1_cspi_devtype_data, },
	{ .compatible = "fsl,imx21-cspi", .data = &imx21_cspi_devtype_data, },
	{ .compatible = "fsl,imx27-cspi", .data = &imx27_cspi_devtype_data, },
	{ .compatible = "fsl,imx31-cspi", .data = &imx31_cspi_devtype_data, },
	{ .compatible = "fsl,imx35-cspi", .data = &imx35_cspi_devtype_data, },
	{ .compatible = "fsl,imx51-ecspi", .data = &imx51_ecspi_devtype_data, },
	{ .compatible = "fsl,imx53-ecspi", .data = &imx53_ecspi_devtype_data, },
	{ /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, spi_imx_dt_ids);

static struct platform_driver spi_imx_driver = {
	.driver = {
		   .name = DRIVER_NAME,
		   .of_match_table = spi_imx_dt_ids,
		   .pm = &imx_spi_pm,
	},
	.id_table = spi_imx_devtype,
	.probe = spi_imx_probe,
	.remove = spi_imx_remove,
};
module_platform_driver(spi_imx_driver);
```

module_platform_driver是一个宏定义，具体代码如下：
```c
/* module_platform_driver() - Helper macro for drivers that don't do
 * anything special in module init/exit.  This eliminates a lot of
 * boilerplate.  Each module may only use this macro once, and
 * calling it replaces module_init() and module_exit()
 */
#define module_platform_driver(__platform_driver) \
	module_driver(__platform_driver, platform_driver_register, \
			platform_driver_unregister)

/**
 * module_driver() - Helper macro for drivers that don't do anything
 * special in module init/exit. This eliminates a lot of boilerplate.
 * Each module may only use this macro once, and calling it replaces
 * module_init() and module_exit().
 *
 * @__driver: driver name
 * @__register: register function for this driver type
 * @__unregister: unregister function for this driver type
 * @...: Additional arguments to be passed to __register and __unregister.
 *
 * Use this macro to construct bus specific macros for registering
 * drivers, and do not use it on its own.
 */
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ \
	return __register(&(__driver) , ##__VA_ARGS__); \
} \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ \
	__unregister(&(__driver) , ##__VA_ARGS__); \
} \
module_exit(__driver##_exit);

/**
 * module_init() - driver initialization entry point
 * @x: function to be run at kernel boot time or module insertion
 *
 * module_init() will either be called during do_initcalls() (if
 * builtin) or at module insertion time (if a module).  There can only
 * be one per module.
 */
#define module_init(x)	__initcall(x);

#define __initcall(fn) device_initcall(fn)
```

可以看到，在 device_initcall 中会调用对应宏生成的init函数，展开后如下
```c
static int __init rockchip_spi_driver_init(void)
{
	return platform_driver_register(&(rockchip_spi_driver) , ##__VA_ARGS__);
}
module_init(rockchip_spi_driver_init);
static void __exit rockchip_spi_driver_exit(void)
{
	platform_driver_unregister(&(rockchip_spi_driver) , ##__VA_ARGS__);
}
module_exit(rockchip_spi_driver_exit);
```

platform_driver_register 会调用 driver_register 来注册驱动，在注册过程中同样会调用 driver_match_device 来匹配设，调用 device_driver_attach 执行驱动的probe回调函数。

具体函数调用链如下：
```c
module_platform_driver(rockchip_spi_driver);
module_init(__driver##\_init);
static int __init __driver##\_init(void)
platform_driver_register
__platform_driver_register
driver_register
bus_add_driver
driver_attach
__driver_attach
​	driver_match_device
​		drv->bus->match
​		platform_match
​			of_driver_match_device
​				of_match_device
​					of_match_node
​						__of_match_node
​							__of_device_is_compatible
​	device_driver_attach
​		driver_probe_device
​			really_probe
​				drv->probe
```

GPIO 平台驱动也是类似，就不在过多介绍，驱动匹配到对应设备后会调用驱动的 probe 回调函数
```c
static int mxc_gpio_probe(struct platform_device *pdev)
{
	struct device_node *np = pdev->dev.of_node;
	struct mxc_gpio_port *port;
	int irq_count;
	int irq_base;
	int err;

	mxc_gpio_get_hw(pdev);

	port = devm_kzalloc(&pdev->dev, sizeof(*port), GFP_KERNEL);
	if (!port)
		return -ENOMEM;

	port->dev = &pdev->dev;

	port->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(port->base))
		return PTR_ERR(port->base);

	irq_count = platform_irq_count(pdev);
	if (irq_count < 0)
		return irq_count;

	if (irq_count > 1) {
		port->irq_high = platform_get_irq(pdev, 1);
		if (port->irq_high < 0)
			port->irq_high = 0;
	}

	port->irq = platform_get_irq(pdev, 0);
	if (port->irq < 0)
		return port->irq;

	/* the controller clock is optional */
	port->clk = devm_clk_get_optional(&pdev->dev, NULL);
	if (IS_ERR(port->clk))
		return PTR_ERR(port->clk);

	err = clk_prepare_enable(port->clk);
	if (err) {
		dev_err(&pdev->dev, "Unable to enable clock.\n");
		return err;
	}

	if (of_device_is_compatible(np, "fsl,imx7d-gpio"))
		port->power_off = true;

	/* disable the interrupt and clear the status */
	writel(0, port->base + GPIO_IMR);
	writel(~0, port->base + GPIO_ISR);

	if (mxc_gpio_hwtype == IMX21_GPIO) {
		/*
		 * Setup one handler for all GPIO interrupts. Actually setting
		 * the handler is needed only once, but doing it for every port
		 * is more robust and easier.
		 */
		irq_set_chained_handler(port->irq, mx2_gpio_irq_handler);
	} else {
		/* setup one handler for each entry */
		irq_set_chained_handler_and_data(port->irq,
						 mx3_gpio_irq_handler, port);
		if (port->irq_high > 0)
			/* setup handler for GPIO 16 to 31 */
			irq_set_chained_handler_and_data(port->irq_high,
							 mx3_gpio_irq_handler,
							 port);
	}

	err = bgpio_init(&port->gc, &pdev->dev, 4,
			 port->base + GPIO_PSR,
			 port->base + GPIO_DR, NULL,
			 port->base + GPIO_GDIR, NULL,
			 BGPIOF_READ_OUTPUT_REG_SET);
	if (err)
		goto out_bgio;

	port->gc.request = gpiochip_generic_request;
	port->gc.free = gpiochip_generic_free;
	port->gc.to_irq = mxc_gpio_to_irq;
	port->gc.base = (pdev->id < 0) ? of_alias_get_id(np, "gpio") * 32 :
					     pdev->id * 32;

	err = devm_gpiochip_add_data(&pdev->dev, &port->gc, port);
	if (err)
		goto out_bgio;

	irq_base = devm_irq_alloc_descs(&pdev->dev, -1, 0, 32, numa_node_id());
	if (irq_base < 0) {
		err = irq_base;
		goto out_bgio;
	}

	port->domain = irq_domain_add_legacy(np, 32, irq_base, 0,
					     &irq_domain_simple_ops, NULL);
	if (!port->domain) {
		err = -ENODEV;
		goto out_bgio;
	}

	/* gpio-mxc can be a generic irq chip */
	err = mxc_gpio_init_gc(port, irq_base);
	if (err < 0)
		goto out_irqdomain_remove;

	list_add_tail(&port->node, &mxc_gpio_ports);

	platform_set_drvdata(pdev, port);

	return 0;

out_irqdomain_remove:
	irq_domain_remove(port->domain);
out_bgio:
	clk_disable_unprepare(port->clk);
	dev_info(&pdev->dev, "%s failed with errno %d\n", __func__, err);
	return err;
}
```

主要做了一下几件事：
- 解析设备树，获取gpio对应的基址和偏移地址。
- 初始化gpio中断。
- 初始化gpio时钟。
- 初始化 struct gpio_chip 结构体
- 将gpio_chip添加到gpiolib当中。
- 将irq_chip_generic添加irq当中。

关键是实现了 struct gpio_chip 结构体，并将其添加到gpiolib当中，这样通过gpiolib，其他驱动就可以调用到对应gpio chip实现的回调函数。


## 四、GPIO 驱动使用

### 1 设备树
在设备树中使用gpio
```c
//创建LED节点，在platform总线管理下
gpiosgrp {
    compatible = "simple-bus";
    //..

    usr_led {
        compatible = "rmk,usr-led";
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_gpio_led>;
        led-gpios = <&gpio1 3 GPIO_ACTIVE_LOW>;
        status = "okay";
    };

    //......
};
```

驱动根据 led-gpios 从设备树得到 struct of_phandle_args 结构体
```c
/**
 * struct of_phandle_args - structure to hold phandle and arguments
 *
 * This is used when decoding a phandle in a device tree property. Typically
 * these look like this:
 *
 * wibble {
 *    phandle = <5>;
 * };
 *
 * ...
 * some-prop = <&wibble 1 2 3>
 *
 * Here &node is the phandle of the node 'wibble', i.e. 5. There are three
 * arguments: 1, 2, 3.
 *
 * So when decoding the phandle in some-prop, np will point to wibble,
 * args_count will be 3 and the three arguments will be in args.
 *
 * @np: Node that the phandle refers to
 * @args_count: Number of arguments
 * @args: Argument values
 */
struct of_phandle_args {
	struct device_node *np;
	int args_count;
	uint32_t args[OF_MAX_PHANDLE_ARGS];
};
```

而 np 是一个指向 struct gpio_chip 的设备节点，有了这个节点就可以从已经注册了的gpio device 中找到与之对应的gpio chip，具体代码如下：
```c
/**
 * of_get_named_gpiod_flags() - Get a GPIO descriptor and flags for GPIO API
 * @np:		device node to get GPIO from
 * @propname:	property name containing gpio specifier(s)
 * @index:	index of the GPIO
 * @flags:	a flags pointer to fill in
 *
 * Returns GPIO descriptor to use with Linux GPIO API, or one of the errno
 * value on the error condition. If @flags is not NULL the function also fills
 * in flags for the GPIO.
 */
static struct gpio_desc *of_get_named_gpiod_flags(struct device_node *np,
		     const char *propname, int index, enum of_gpio_flags *flags)
{
	struct of_phandle_args gpiospec;
	struct gpio_chip *chip;
	struct gpio_desc *desc;
	int ret;

	ret = of_parse_phandle_with_args_map(np, propname, "gpio", index,
					     &gpiospec);
	if (ret) {
		pr_debug("%s: can't parse '%s' property of node '%pOF[%d]'\n",
			__func__, propname, np, index);
		return ERR_PTR(ret);
	}

	chip = of_find_gpiochip_by_xlate(&gpiospec);
	if (!chip) {
		desc = ERR_PTR(-EPROBE_DEFER);
		goto out;
	}

	desc = of_xlate_and_get_gpiod_flags(chip, &gpiospec, flags);
	if (IS_ERR(desc))
		goto out;

	if (flags)
		of_gpio_flags_quirks(np, propname, flags, index);

	pr_debug("%s: parsed '%s' property of node '%pOF[%d]' - status (%d)\n",
		 __func__, propname, np, index,
		 PTR_ERR_OR_ZERO(desc));

out:
	of_node_put(gpiospec.np);

	return desc;
}

static struct gpio_chip *of_find_gpiochip_by_xlate(
					struct of_phandle_args *gpiospec)
{
	return gpiochip_find(gpiospec, of_gpiochip_match_node_and_xlate);
}

static int of_gpiochip_match_node_and_xlate(struct gpio_chip *chip, void *data)
{
	struct of_phandle_args *gpiospec = data;

	return chip->gpiodev->dev.of_node == gpiospec->np &&
				chip->of_xlate &&
				chip->of_xlate(chip, gpiospec, NULL) >= 0;
}

/**
 * gpiochip_find() - iterator for locating a specific gpio_chip
 * @data: data to pass to match function
 * @match: Callback function to check gpio_chip
 *
 * Similar to bus_find_device.  It returns a reference to a gpio_chip as
 * determined by a user supplied @match callback.  The callback should return
 * 0 if the device doesn't match and non-zero if it does.  If the callback is
 * non-zero, this function will return to the caller and not iterate over any
 * more gpio_chips.
 */
struct gpio_chip *gpiochip_find(void *data,
				int (*match)(struct gpio_chip *gc,
					     void *data))
{
	struct gpio_device *gdev;
	struct gpio_chip *gc = NULL;
	unsigned long flags;

	spin_lock_irqsave(&gpio_lock, flags);
	list_for_each_entry(gdev, &gpio_devices, list)
		if (gdev->chip && match(gdev->chip, data)) {
			gc = gdev->chip;
			break;
		}

	spin_unlock_irqrestore(&gpio_lock, flags);

	return gc;
}
EXPORT_SYMBOL_GPL(gpiochip_find);


```

同样，有了gpio chip 和对应的gpio 编号，就可以调用gpio chip对应的回调函数获取 gpio desc：
```c
static struct gpio_desc *of_xlate_and_get_gpiod_flags(struct gpio_chip *chip,
					struct of_phandle_args *gpiospec,
					enum of_gpio_flags *flags)
{
	int ret;

	if (chip->of_gpio_n_cells != gpiospec->args_count)
		return ERR_PTR(-EINVAL);

	ret = chip->of_xlate(chip, gpiospec, flags);
	if (ret < 0)
		return ERR_PTR(ret);

	return gpiochip_get_desc(chip, ret);
}

/**
 * gpiochip_get_desc - get the GPIO descriptor corresponding to the given
 *                     hardware number for this chip
 * @gc: GPIO chip
 * @hwnum: hardware number of the GPIO for this chip
 *
 * Returns:
 * A pointer to the GPIO descriptor or ``ERR_PTR(-EINVAL)`` if no GPIO exists
 * in the given chip for the specified hardware number.
 */
struct gpio_desc *gpiochip_get_desc(struct gpio_chip *gc,
				    unsigned int hwnum)
{
	struct gpio_device *gdev = gc->gpiodev;

	if (hwnum >= gdev->ngpio)
		return ERR_PTR(-EINVAL);

	return &gdev->descs[hwnum];
}
EXPORT_SYMBOL_GPL(gpiochip_get_desc);
```

如此这边，将设备树中gpio描述转换为了gpio_desc，并获取到了对应的gpio_chip，就可以根据 gpio_chip 回调函数调用具体的功能函数。

函数调用链如下：
```text
gpiod_get_index
​	of_find_gpio
​		of_get_named_gpiod_flags
​			of_find_gpiochip_by_xlate
​				gpiochip_find
​			of_xlate_and_get_gpiod_flags
​				gpiochip_get_desc
```

### 2 GPIO 使用指南
不能在没有标准 GPIO 调用的情况下工作的驱动程序应该具有依赖于 GPIOLIB 或选择 GPIOLIB 的 Kconfig 条目。 通过包含以下文件，可以使用允许驱动程序获取和使用 GPIO 的函数
```
#include <linux/gpio/consumer.h>
```
如果禁用 GPIOLIB，则头文件中的所有函数都有静态内联存根。 当调用这些存根时，它们将发出警告。 这些存根用于两个用例
- 简单的编译覆盖，例如 COMPILE_TEST - 当前平台是否启用或选择 GPIOLIB 并不重要，因为无论如何我们都不会执行该系统。
- 真正的可选 GPIOLIB 支持 - 在某些系统的某些编译时配置中，驱动程序实际上并不使用 GPIO，但在其他编译时配置下将使用它。 在这种情况下，消费者必须确保不调用这些函数，否则用户将遇到可能被认为具有威胁性的控制台警告。 将真正可选的 GPIOLIB 用法与调用 `[devm_]gpiod_get_optional()` 结合使用是一个*坏主意*，并且会导致奇怪的错误消息。 将可选 GPIOLIB 与普通的 getter 函数一起使用：当您这样做时，应该预料到一些开放式的错误处理。
所有使用基于描述符的 GPIO 接口的函数都以 `gpiod_` 作为前缀。 `gpio_` 前缀用于旧版接口。 内核中的任何其他函数都不应使用这些前缀。 强烈建议不要使用旧版函数，新代码应专门使用 <linux/gpio/consumer.h> 和描述符。

### 3 获取和释放 GPIO
使用基于描述符的接口，GPIO 通过不透明的、不可伪造的处理程序来标识，该处理程序必须通过调用 [`gpiod_get()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get) 函数之一来获取。 与许多其他内核子系统一样，[`gpiod_get()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get) 采用将使用 GPIO 的设备和所请求的 GPIO 应该履行的功能

```
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id,
                            enum gpiod_flags flags)
```

如果一个函数是通过一起使用多个 GPIO 来实现的（例如，一个简单的 LED 设备，用于显示数字），则可以指定一个额外的索引参数

```
struct gpio_desc *gpiod_get_index(struct device *dev,
                                  const char *con_id, unsigned int idx,
                                  enum gpiod_flags flags)
```

有关 DeviceTree 情况下 con_id 参数的更详细描述，请参阅 [GPIO 映射](https://docs.linuxkernel.org.cn/driver-api/gpio/board.html)

flags 参数用于可选地指定 GPIO 的方向和初始值。 值可以是

- GPIOD_ASIS 或 0 以完全不初始化 GPIO。 方向必须稍后使用其中一个专用函数进行设置。
- GPIOD_IN 将 GPIO 初始化为输入。
- GPIOD_OUT_LOW 将 GPIO 初始化为输出，值为 0。
- GPIOD_OUT_HIGH 将 GPIO 初始化为输出，值为 1。
- GPIOD_OUT_LOW_OPEN_DRAIN 与 GPIOD_OUT_LOW 相同，但也强制该线路在电气上与开漏一起使用。
- GPIOD_OUT_HIGH_OPEN_DRAIN 与 GPIOD_OUT_HIGH 相同，但也强制该线路在电气上与开漏一起使用。

请注意，初始值是*逻辑*值，物理线路电平取决于线路是配置为高电平有效还是低电平有效（请参阅 [低电平有效和开漏语义](https://docs.linuxkernel.org.cn/driver-api/gpio/consumer.html#active-low-semantics)）。

最后两个标志用于开漏是强制性的用例，例如 I2C：如果线路尚未在映射中配置为开漏（请参阅 [GPIO 映射](https://docs.linuxkernel.org.cn/driver-api/gpio/board.html)），则无论如何都会强制使用开漏，并且会打印一条警告，指出需要更新板配置以匹配用例。

这两个函数都返回有效的 GPIO 描述符，或者返回一个可以使用 [`IS_ERR()`](https://docs.linuxkernel.org.cn/core-api/kernel-api.html#c.IS_ERR) 检查的错误代码（它们永远不会返回 NULL 指针）。 仅当且仅当没有将 GPIO 分配给设备/功能/索引三元组时，才会返回 -ENOENT，其他错误代码用于已分配 GPIO 但尝试获取时发生错误的情况。 这对于区分仅仅的错误和可选 GPIO 参数缺少 GPIO 非常有用。 对于 GPIO 可选的常见模式，可以使用 [`gpiod_get_optional()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_optional) 和 [`gpiod_get_index_optional()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_index_optional) 函数。 如果没有将 GPIO 分配给请求的函数，这些函数将返回 NULL 而不是 -ENOENT

```
struct gpio_desc *gpiod_get_optional(struct device *dev,
                                     const char *con_id,
                                     enum gpiod_flags flags)

struct gpio_desc *gpiod_get_index_optional(struct device *dev,
                                           const char *con_id,
                                           unsigned int index,
                                           enum gpiod_flags flags)
```

请注意，gpio_get*_optional() 函数（及其托管变体），与 gpiolib API 的其余部分不同，当禁用 gpiolib 支持时，也会返回 NULL。 这对驱动程序作者很有帮助，因为他们不需要专门处理 -ENOSYS 返回代码。 但是，系统集成商应注意在需要它的系统上启用 gpiolib。

对于使用多个 GPIO 的函数，可以使用一次调用来获取所有这些 GPIO

```
struct gpio_descs *gpiod_get_array(struct device *dev,
                                   const char *con_id,
                                   enum gpiod_flags flags)
```

此函数返回一个 struct gpio_descs，其中包含一个描述符数组。 它还包含一个指向 gpiolib 私有结构的指针，如果将其传递回 get/set 数组函数，则可能会加快 I/O 处理

```
struct gpio_descs {
        struct gpio_array *info;
        unsigned int ndescs;
        struct gpio_desc *desc[];
}
```

如果未将 GPIO 分配给请求的函数，则以下函数将返回 NULL 而不是 -ENOENT

```
struct gpio_descs *gpiod_get_array_optional(struct device *dev,
                                            const char *con_id,
                                            enum gpiod_flags flags)
```

这些函数的设备托管变体也已定义

```
struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id,
                                 enum gpiod_flags flags)

struct gpio_desc *devm_gpiod_get_index(struct device *dev,
                                       const char *con_id,
                                       unsigned int idx,
                                       enum gpiod_flags flags)

struct gpio_desc *devm_gpiod_get_optional(struct device *dev,
                                          const char *con_id,
                                          enum gpiod_flags flags)

struct gpio_desc *devm_gpiod_get_index_optional(struct device *dev,
                                                const char *con_id,
                                                unsigned int index,
                                                enum gpiod_flags flags)

struct gpio_descs *devm_gpiod_get_array(struct device *dev,
                                        const char *con_id,
                                        enum gpiod_flags flags)

struct gpio_descs *devm_gpiod_get_array_optional(struct device *dev,
                                                 const char *con_id,
                                                 enum gpiod_flags flags)
```

可以使用 [`gpiod_put()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_put) 函数释放 GPIO 描述符

```
void gpiod_put(struct gpio_desc *desc)
```

对于 GPIO 数组，可以使用此函数

```
void gpiod_put_array(struct gpio_descs *descs)
```

在调用这些函数后，严禁使用描述符。 也不允许从使用 [`gpiod_get_array()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array) 获取的数组中单独释放描述符（使用 [`gpiod_put()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_put)）。

不出所料，设备托管变体是

```
void devm_gpiod_put(struct device *dev, struct gpio_desc *desc)

void devm_gpiod_put_array(struct device *dev, struct gpio_descs *descs)
```

### 4 使用 GPIO

#### 设置方向

驱动程序必须对 GPIO 做的第一件事是设置其方向。 如果没有为 gpiod_get*() 提供任何方向设置标志，则通过调用 gpiod_direction_*() 函数之一来完成此操作

```
int gpiod_direction_input(struct gpio_desc *desc)
int gpiod_direction_output(struct gpio_desc *desc, int value)
```

返回值成功时为零，否则为负 errno。 应该检查它，因为 get/set 调用不返回错误，并且可能会出现配置错误。 通常，您应该从任务上下文中发出这些调用。 但是，对于自旋锁安全的 GPIO，在启用任务处理之前，作为早期板设置的一部分，可以使用它们是可以的。

对于输出 GPIO，提供的值将成为初始输出值。 这有助于避免系统启动期间的信号毛刺。

驱动程序还可以查询 GPIO 的当前方向

```
int gpiod_get_direction(const struct gpio_desc *desc)
```

此函数对于输出返回 0，对于输入返回 1，如果发生错误，则返回错误代码。

请注意，GPIO 没有默认方向。 因此，**在未先设置 GPIO 方向的情况下使用 GPIO 是非法的，并且会导致未定义的行为！**

#### 自旋锁安全的 GPIO 访问

大多数 GPIO 控制器都可以使用内存读取/写入指令进行访问。 这些不需要休眠，并且可以安全地从硬（非线程）IRQ 处理程序和类似上下文中完成。

使用以下调用从原子上下文中访问 GPIO

```
int gpiod_get_value(const struct gpio_desc *desc);
void gpiod_set_value(struct gpio_desc *desc, int value);
```

这些值是布尔值，非活动时为零，活动时为非零。 读取输出引脚的值时，返回的值应该是引脚上看到的值。 由于包括开漏信号和输出延迟在内的问题，这并不总是与指定的输出值匹配。

get/set 调用不返回错误，因为应该更早地从 gpiod_direction_*() 报告“无效 GPIO”。 但是，请注意，并非所有平台都可以读取输出引脚的值； 那些不能读取的平台应始终返回零。 此外，对于无法在没有休眠的情况下安全访问的 GPIO 使用这些调用（请参见下文）是一个错误。

#### 可能休眠的 GPIO 访问

必须使用基于消息的总线（如 I2C 或 SPI）来访问某些 GPIO 控制器。 读取或写入这些 GPIO 值的命令需要等待到达队列的头部以传输命令并获取其响应。 这需要休眠，而这无法从 IRQ 处理程序内部完成。

支持此类型 GPIO 的平台通过从此调用返回非零值来将其与其他 GPIO 区分开

```
int gpiod_cansleep(const struct gpio_desc *desc)
```

要访问此类 GPIO，需要定义一组不同的访问器

```
int gpiod_get_value_cansleep(const struct gpio_desc *desc)
void gpiod_set_value_cansleep(struct gpio_desc *desc, int value)
```

访问此类 GPIO 需要可能休眠的上下文，例如线程化的 IRQ 处理程序，并且必须使用这些访问器而不是没有 cansleep() 名称后缀的自旋锁安全访问器。

除了这些访问器可能会休眠并且可以在无法从 hardIRQ 处理程序访问的 GPIO 上工作之外，这些调用与自旋锁安全调用的作用相同。



#### 低电平有效和开漏语义

由于消费者不应关心物理线路电平，因此所有 gpiod_set_value_xxx() 或 gpiod_set_array_value_xxx() 函数都使用*逻辑*值进行操作。 通过这种方式，它们会考虑低电平有效属性。 这意味着它们会检查 GPIO 是否配置为低电平有效，如果是，则在驱动物理线路电平之前，它们会操作传递的值。

这同样适用于开漏或开路源输出线路：它们不会主动将其输出驱动为高电平（开漏）或低电平（开路源），它们只是将其输出切换为高阻抗值。 消费者不需要关心。（有关详细信息，请阅读 [GPIO 驱动程序接口](https://docs.linuxkernel.org.cn/driver-api/gpio/driver.html) 中的开漏。）

通过这种方式，所有 gpiod_set_(array)_value_xxx() 函数都将参数“value”解释为“活动”（“1”）或“非活动”（“0”）。 物理线路电平将相应地驱动。

例如，如果设置了专用 GPIO 的低电平有效属性，并且 gpiod_set_(array)_value_xxx() 传递了“活动”（“1”），则物理线路电平将被驱动为低电平。

总结一下

```
Function (example)                 line property          physical line
gpiod_set_raw_value(desc, 0);      don't care             low
gpiod_set_raw_value(desc, 1);      don't care             high
gpiod_set_value(desc, 0);          default (active high)  low
gpiod_set_value(desc, 1);          default (active high)  high
gpiod_set_value(desc, 0);          active low             high
gpiod_set_value(desc, 1);          active low             low
gpiod_set_value(desc, 0);          open drain             low
gpiod_set_value(desc, 1);          open drain             high impedance
gpiod_set_value(desc, 0);          open source            high impedance
gpiod_set_value(desc, 1);          open source            high
```

可以使用 set_raw/get_raw 函数覆盖这些语义，但应尽可能避免这样做，特别是对于不应关心实际物理线路电平而应担心逻辑值的与系统无关的驱动程序。

### 5 访问原始 GPIO 值

存在一些消费者，他们需要管理 GPIO 线路的逻辑状态，即他们的设备实际接收的值，无论它与 GPIO 线路之间有什么。

以下调用集忽略 GPIO 的低电平有效或开漏属性，并处理原始线路值

```
int gpiod_get_raw_value(const struct gpio_desc *desc)
void gpiod_set_raw_value(struct gpio_desc *desc, int value)
int gpiod_get_raw_value_cansleep(const struct gpio_desc *desc)
void gpiod_set_raw_value_cansleep(struct gpio_desc *desc, int value)
int gpiod_direction_output_raw(struct gpio_desc *desc, int value)
```

GPIO 的低电平有效状态也可以使用以下调用来查询和切换

```
int gpiod_is_active_low(const struct gpio_desc *desc)
void gpiod_toggle_active_low(struct gpio_desc *desc)
```

请注意，这些函数应谨慎使用； 驱动程序不应关心物理线路电平或开漏语义。

### 6 使用单个函数调用访问多个 GPIO

以下函数获取或设置 GPIO 数组的值

```
int gpiod_get_array_value(unsigned int array_size,
                          struct gpio_desc **desc_array,
                          struct gpio_array *array_info,
                          unsigned long *value_bitmap);
int gpiod_get_raw_array_value(unsigned int array_size,
                              struct gpio_desc **desc_array,
                              struct gpio_array *array_info,
                              unsigned long *value_bitmap);
int gpiod_get_array_value_cansleep(unsigned int array_size,
                                   struct gpio_desc **desc_array,
                                   struct gpio_array *array_info,
                                   unsigned long *value_bitmap);
int gpiod_get_raw_array_value_cansleep(unsigned int array_size,
                                   struct gpio_desc **desc_array,
                                   struct gpio_array *array_info,
                                   unsigned long *value_bitmap);

int gpiod_set_array_value(unsigned int array_size,
                          struct gpio_desc **desc_array,
                          struct gpio_array *array_info,
                          unsigned long *value_bitmap)
int gpiod_set_raw_array_value(unsigned int array_size,
                              struct gpio_desc **desc_array,
                              struct gpio_array *array_info,
                              unsigned long *value_bitmap)
int gpiod_set_array_value_cansleep(unsigned int array_size,
                                   struct gpio_desc **desc_array,
                                   struct gpio_array *array_info,
                                   unsigned long *value_bitmap)
int gpiod_set_raw_array_value_cansleep(unsigned int array_size,
                                       struct gpio_desc **desc_array,
                                       struct gpio_array *array_info,
                                       unsigned long *value_bitmap)
```

该数组可以是任意一组 GPIO。 如果相应的芯片驱动程序支持，这些函数将尝试同时访问属于同一组或芯片的 GPIO。 在这种情况下，可以预期性能会得到显着提高。 如果无法同时访问，则将按顺序访问 GPIO。

这些函数采用四个参数

> - array_size - 数组元素的数量
> - desc_array - GPIO 描述符数组
> - array_info - 从 [`gpiod_get_array()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array) 获取的可选信息
> - value_bitmap - 用于存储 GPIO 值的位图（get）或用于分配给 GPIO 的值的位图（set）

可以使用 [`gpiod_get_array()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array) 函数或其变体之一获取描述符数组。 如果该函数返回的描述符组与所需的 GPIO 组匹配，则可以通过简单地使用 [`gpiod_get_array()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array) 返回的 struct gpio_descs 来访问这些 GPIO

```
struct gpio_descs *my_gpio_descs = gpiod_get_array(...);
gpiod_set_array_value(my_gpio_descs->ndescs, my_gpio_descs->desc,
                      my_gpio_descs->info, my_gpio_value_bitmap);
```

也可以访问完全任意的描述符数组。 可以使用 [`gpiod_get()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get) 和 [`gpiod_get_array()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array) 的任意组合来获取描述符。 之后，必须手动设置描述符数组，然后才能将其传递给上述函数之一。 在这种情况下，应将 array_info 设置为 NULL。

请注意，为了获得最佳性能，属于同一芯片的 GPIO 应在描述符数组中是连续的。

如果描述符的数组索引与单个芯片的硬件引脚编号匹配，则可以实现更好的性能。 如果传递给 get/set 数组函数的数组与从 [`gpiod_get_array()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array) 获取的数组匹配，并且还传递与该数组关联的 array_info，则该函数可能会采用快速位图处理路径，将 value_bitmap 参数直接传递给芯片的相应 .get/set_multiple() 回调。 这允许利用 GPIO 组作为数据 I/O 端口，而不会损失太多性能。

[`gpiod_get_array_value()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_array_value) 及其变体的返回值成功时为 0，发生错误时为负值。 请注意与 [`gpiod_get_value()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_get_value) 的区别，后者在成功时返回 0 或 1 以传达 GPIO 值。 使用数组函数，GPIO 值存储在 value_array 中，而不是作为返回值传递回来。

### 7 映射到 IRQ 的 GPIO

GPIO 线路通常可以用作 IRQ。 您可以使用以下调用获取与给定 GPIO 对应的 IRQ 编号

```
int gpiod_to_irq(const struct gpio_desc *desc)
```

它将返回一个 IRQ 编号，如果无法完成映射，则返回一个负的 errno 代码（最可能的原因是该特定 GPIO 无法用作 IRQ）。 使用未设置为使用 [`gpiod_direction_input()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_direction_input) 的输入的 GPIO，或者使用最初不是来自 [`gpiod_to_irq()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_to_irq) 的 IRQ 编号，是一个未检查的错误。 不允许 [`gpiod_to_irq()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_to_irq) 休眠。

从 [`gpiod_to_irq()`](https://docs.linuxkernel.org.cn/driver-api/gpio/index.html#c.gpiod_to_irq) 返回的非错误值可以传递给 [`request_irq()`](https://docs.linuxkernel.org.cn/core-api/genericirq.html#c.request_irq) 或 [`free_irq()`](https://docs.linuxkernel.org.cn/core-api/genericirq.html#c.free_irq)。 它们通常会由特定于板的初始化代码存储到平台设备的 IRQ 资源中。 请注意，IRQ 触发选项是 IRQ 接口的一部分，例如 IRQF_TRIGGER_FALLING，系统唤醒功能也是如此。
