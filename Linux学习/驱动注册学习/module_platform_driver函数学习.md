# 驱动学习：`module_platform_driver`函数学习

## 起因

`drivers/soc/hisilicon/kunpeng_hccs.c` 中 `hccs_probe()` 函数有报错，该函数主要是利用 `platform_device` 结构体将 `hccs_dev` 和 `acpi_dev` 进行关联注册，该函数为结构体 `hccs_driver` 的成员，注意这里是驱动不再是传入的设备：

```C
static struct platform_driver hccs_driver = {
    .probe = hccs_probe,
    .remove_new = hccs_remove,
    .driver = {
        .name = "kunpeng_hccs",
        .acpi_match_table = hccs_acpi_match,
    },
};
```

可以看到，驱动的结构体中包含了 `probe` 、`remove`以及 `driver` 基础结构体，定义驱动结构之后要做的就是将驱动进行注册。

## 驱动注册

```C
// drivers/soc/hisilicon/kunpeng_hccs.c
module_platform_driver(hccs_driver);

// include/linux/platform_device.h
#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
            platform_driver_unregister)
```

先不看 `platform_driver_register` `platform_driver_unregister` 的具体实现，先看 `module_platform_driver` ，可以发现 `module_platform_driver` 依然是一个宏定义：

```C
// include/linux/device/driver.h
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
```

这里 `module_driver` 宏定义下，首先定义 `__driver##_init` 函数然后传入 `module_init()` 函数，其次定义 `__exit __driver##_exit()` 函数传入 `module_exit()` 函数。

`module_init()` 和 `module_exit()` 定义为：

```C
// include/linux/module.h
#define module_init(x) __initcall(x);
#define module_exit(x) __exitcall(x);

// include/linux/init.h
#define __initcall(fn) device_initcall(fn)
#define __exitcall(fn)      \
    static exitcall_t __exitcall_##fn __exit_call = fn

#define device_initcall(fn) __define_initcall(fn, 6)
```

其中 exit 部分将一个函数传给一个特定函数名的函数，init 中 `device_initcall` 又为 `device_initcall` 的宏定义，这里我们看到多了一个数字（初始化的优先级），`__define_initcall` 定义为：

```C
// include/linux/init.h
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)
```

该函数的作用是将一开始定义的注册函数放在 ELF 指定的 `.initcall##id` 段内，这样就保证了初始化时候的执行顺序，这里用到 [`__attribute__((section(__sec))`](https://blog.csdn.net/seven_feifei/article/details/95947358) 来实现。

简单讲，驱动注册的过程是：

1. 将驱动结构体传给 `module_platform_driver` 进行注册；
2. `module_platform_driver`宏会获取驱动结构体和驱动的 `register` 和 `unregister` 函数；
3. `unregister` 函数会被赋值给特性名称的变量，而 `register` 函数 + 优先级 会被写入 ELF `.initcall##id` 段来保证初始化时候的注册顺序。

## `platform_driver_register` 函数

上面跳过了对 `module_platform_driver` 获得的 `register` 和 `unregister` 函数的分析，这里先看看 `platform_driver_register` 函数：

```C
// include/linux/platform_device.h
#define platform_driver_register(drv) \
 __platform_driver_register(drv, THIS_MODULE)
extern int __platform_driver_register(struct platform_driver *,
     struct module *);

// drivers/base/platform.c
/**
 * __platform_driver_register - register a driver for platform-level devices
 * @drv: platform driver structure
 * @owner: owning module/driver
 */
int __platform_driver_register(struct platform_driver *drv,
    struct module *owner)
{
 drv->driver.owner = owner;
 drv->driver.bus = &platform_bus_type;

 return driver_register(&drv->driver);
}
```

其中，`.ownwer` 为 `module` 结构体，代表的是一个内核模块，当使用 `insmod` 的时候编写的内核模块与一个 `module` 结构体关联。

平台驱动注册中，在给基础驱动结构体赋值 `owner` 和 `bus` 之后进行驱动的注册：

```C
// drivers/base/driver.c
int driver_register(struct device_driver *drv)
{
    int ret;
    struct device_driver *other;

    // 通过驱动名在内核中查找，找到则说明已经注册过
    other = driver_find(drv->name, drv->bus);
    // 根据总线类型添加驱动
    ret = bus_add_driver(drv);
    // 将驱动添加到对应组
    ret = driver_add_groups(drv, drv->groups);
    // 注册 uevent 事件
    kobject_uevent(&drv->p->kobj, KOBJ_ADD);

    return ret;
}
```

## `platform_driver_unregister` 函数

如果看过了 register，unregister 相对就简单很多：

```C
// drivers/base/platform.c
void platform_driver_unregister(struct platform_driver *drv)
{
 driver_unregister(&drv->driver);
}
EXPORT_SYMBOL_GPL(platform_driver_unregister);

// drivers/base/driver.c
void driver_unregister(struct device_driver *drv)
{
 if (!drv || !drv->p) {
  WARN(1, "Unexpected driver unregister!\n");
  return;
 }
 driver_remove_groups(drv, drv->groups);
 bus_remove_driver(drv);
}

```

## TODO

看驱动过程中，对于 `driver_register` 函数还是需要进一步的探索，这里涉及一些概念，后续了解清楚：

- [ ] 总线知识
- [ ] kobject知识

后续先看 `driver_find` 函数了解 kobject 相关结构。
