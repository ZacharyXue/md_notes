# 驱动学习：`driver_register`函数学习

`driver_register` 函数主要步骤为：

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

## `driver_find` 函数

`driver_find` 函数是通过驱动名在总线中查找是否存在指定驱动，如果存在，则返回 `device_driver` 结构体。在`driver_register`函数中，如果 `driver_find` 查找到则会直接返回，不会继续进行注册。

``` C
// drivers/base/driver.c
struct device_driver *driver_find(const char *name, struct bus_type *bus)
{
    // 遍历 kset 对比 kset 成员 name 和 输入name 之间是否一致
    struct kobject *k = kset_find_obj(bus->p->drivers_kset, name); 
    struct driver_private *priv;

    if (k) {
        /* Drop reference added by kset_find_obj() */
        kobject_put(k);  // 使用完驱动指针后减少引用次数防止内存泄漏
        // 通过 kobject 对象获取 driver_private 对象
        priv = to_driver(k); 
        return priv->driver;
    }
 return NULL;
}
```

`kset_find_obj` 函数流程为：加自旋锁 --> 遍历 `kset` 比对 `kobject` `name` 与输入 `name` --> 释放锁。

`driver_private` 结构体用于在**同一个驱动支持多个相同设备**时，为各个设备准备不冲突的数据结构。

```C
// drivers/base/base.h
struct driver_private {
    struct kobject kobj;  // 驱动内核对象
    struct klist klist_devices;  // 驱动相关设备链表
    struct klist_node knode_bus;  // 将驱动添加到总线的链表中
    struct module_kobject *mkobj;  // 指向驱动所属模块的指针
    struct device_driver *driver;  // 指向驱动实体
};
```

## `bus_add_driver` 函数

```C
// drivers/base/bus.c
int bus_add_driver(struct device_driver *drv)
{
    struct bus_type *bus;
    struct driver_private *priv;
    int error = 0;
    // 获取总线类型的指针，并增加总线的引用计数
    bus = bus_get(drv->bus);
    // 申请内核空间
    priv = kzalloc(sizeof(*priv), GFP_KERNEL);
    // 初始化 klist
    klist_init(&priv->klist_devices, NULL, NULL);
    // 初始化 driver_private 的 driver
    // 同时，将 driver 的 p 指向 driver_private
    priv->driver = drv;
    drv->p = priv;

    // 初始化 driver_private 的 kobject --> kset 部分
    priv->kobj.kset = bus->p->drivers_kset;
    // 初始化 kobject --> kref、entry…… + 加入 kset
    // from lib/kobject.c 
    error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
            "%s", drv->name);

    // 加入 klist 队列末尾
    klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
    // 遍历总线上设备与驱动进行匹配，匹配后使用驱动对设备进行初始化
    if (drv->bus->p->drivers_autoprobe) {
        error = driver_attach(drv);
    }
    // 创建 module 和 driver 的关联
    module_add_driver(drv->owner, drv);
    // 创建驱动 sysfs 文件
    error = driver_create_file(drv, &driver_attr_uevent);
    //  creates a bunch of attribute groups
    error = driver_add_groups(drv, bus->drv_groups);

    return 0;
}
```

### `klist_init` 函数

`klist_init` 函数初始化 `klist` 结构体的时候初始化其链表和自旋锁：

```C
//lib/klist.c
void klist_init(struct klist *k, void (*get)(struct klist_node *),
    void (*put)(struct klist_node *))
{
    INIT_LIST_HEAD(&k->k_list);
    spin_lock_init(&k->k_lock);
    k->get = get;
    k->put = put;
}
```

### `kobject_init_and_add` 函数

`kobject` 的初始化包括：

1. 初始化 `klist` 之后赋值 `driver_private` 成员 `kobject` 的 `kset`；
2. 在 `va_start` 的 `kobject_init_and_add` 中对 `kobject` 进行初始化：
    - `kobject_init_internal()`:

        ```C
        // lib/kobject.c 
        static void kobject_init_internal(struct kobject *kobj)
        {
            if (!kobj)
                return;
            kref_init(&kobj->kref);
            INIT_LIST_HEAD(&kobj->entry);
            kobj->state_in_sysfs = 0;
            kobj->state_add_uevent_sent = 0;
            kobj->state_remove_uevent_sent = 0;
            kobj->state_initialized = 1;
        }
        ```

    - `kobject_add_varg` 设置 `kobject` 的 `name` `parent`，然后调用 `kobject_add_internal` 加入 `kset` ，创建目录文件

        ```C
        // lib/kobject.c/kobject_set_name_vargs
        kobj->name = s;
        // lib/kobject.c/kobject_add_varg
        kobj->parent = parent;
        // lib/kobject.c/kobject_add_internal
        /* join kset if set, use it as parent if we do not already have one */
        if (kobj->kset) {
        if (!parent)
            parent = kobject_get(&kobj->kset->kobj);
            kobj_kset_join(kobj);
            kobj->parent = parent;
        }
        error = create_dir(kobj);
        kobj->state_in_sysfs = 1;
        ```

### `driver_attach` 函数

当调用 `driver_attach` 函数时，它会**遍历给定总线上的所有设备**，并尝试将传入的驱动程序与这些设备进行匹配。如果找到匹配的设备，内核会尝试使用该驱动程序来初始化设备。

```C
// drivers/base/dd.c
int driver_attach(struct device_driver *drv)
{
    return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}
```

`driver_attach` 函数对 `bus` 上的 `klist` 中每一个 `device` 进行遍历，调用 `__driver_attach`函数。注意这里注册驱动的时候是对已有的设备进行遍历进行匹配，当有新设备的时候是对所有驱动进行遍历与设备进行匹配。

`__driver_attach` 函数做了两件事情：

```C
// drivers/base/dd.c
// 驱动与设备进行匹配
ret = driver_match_device(drv, dev);
// 初始化设备
driver_probe_device(drv, dev);
// ----> __driver_probe_device
// --------> really_probe
```

### `module_add_driver` 函数

`module_add_driver` 函数将 `module` 结构体与 `device_driver` 结构体建立关联（两个结构体都有对应成员指针指向对方），同时 `module_kobject` 对象初始化 `drivers_dir` 成员，建立 sysfs 链接关系。

```C
// drivers/base/module.c
void module_add_driver(struct module *mod, struct device_driver *drv)
{
    char *driver_name;
    int no_warn;
    struct module_kobject *mk = NULL;

    if (mod)
        mk = &mod->mkobj;


    /* Lookup built-in module entry in /sys/modules */
    mkobj = kset_find_obj(module_kset, drv->mod_name);

    mk = container_of(mkobj, struct module_kobject, kobj);
    /* remember our module structure */
    drv->p->mkobj = mk;

    /* Don't check return codes; these calls are idempotent */
    no_warn = sysfs_create_link(&drv->p->kobj, &mk->kobj, "module");

    driver_name = make_driver_name(drv);  
    // 创建 mk->drivers_dir kobject 对象，其 parent 为 mk 的 kobject
    module_create_drivers_dir(mk);
    no_warn = sysfs_create_link(mk->drivers_dir, &drv->p->kobj,
                    driver_name);
}
```
