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

