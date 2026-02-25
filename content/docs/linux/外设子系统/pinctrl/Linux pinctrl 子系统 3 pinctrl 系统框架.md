---
title: Linux GPIO 系统
date: 2026-02-19
authors:
  - name: lunuj
    link: https://github.com/lunuj
tags:
  - 2025
  - Linux
  -  pinctrl
---
**Linux** 内核 **pinctrl** 系统驱动框架。
<!--more-->  

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
![pcb](http://www.wowotech.net/content/uploadfile/201407/ed00896a96bea67a76b5d8a901aee43320140726102404.gif)
pin control subsystem会向系统中的其他driver提供接口以便进行该driver的pin config和pin mux的设定。理想的状态是GPIO controll driver也只是象UART,SPI这样driver一样和pin control subsystem进行交互，但是，实际上由于各种源由（后文详述），pin control subsystem和GPIO subsystem必须有交互。

### 3 pin control subsystem内部block diagram
![pccore](http://www.wowotech.net/content/uploadfile/201407/07edbb347ae8fb535f78f09a782bd36e20140726102406.gif)
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
