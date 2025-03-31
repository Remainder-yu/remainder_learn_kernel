## arm64反编译设备树
```cpp
        pl011@9000000 {
                clock-names = "uartclk\0apb_pclk";
                clocks = <0x8000 0x8000>;
                interrupts = <0x00 0x01 0x04>;
                reg = <0x00 0x9000000 0x00 0x1000>;
                compatible = "arm,pl011\0arm,primecell";
        };

```
arm,pl011和arm,primecell是该外设的兼容字符串,用于和驱动程序进行匹配工作。

interrupts域描述相关属性，这里分别使用3个属性来表示。
1. 中断类型。对于GIC，它主要分成两种类型的中断，分别如下。
* GIC_SPI：共享外设中断，该值在设备树中用0表示
* GIC_PPI:私有外设中断。该值在设备树中用1来表示。
2. 中断ID
3. 触发类型：对应,中断触发类型为高电平触发(IRQ_TYPE_LEVEL_HIGH)。


分析：

首先从根目录下查找“simple-bus”，从设备树可以看出指向smb设备。

用of_platform_bus_create中遍历所有设备。

```cpp
const struct of_device_id of_default_bus_match_table[] = {
	{ .compatible = "simple-bus", },
	{ .compatible = "simple-mfd", },
	{ .compatible = "isa", },
#ifdef CONFIG_ARM_AMBA
	{ .compatible = "arm,amba-bus", },
#endif /* CONFIG_ARM_AMBA */
	{} /* Empty terminated list */
};

static int __init customize_machine(void)
{
	/*
	 * customizes platform devices, or adds new ones
	 * On DT based machines, we fall back to populating the
	 * machine from the device tree, if no callback is provided,
	 * otherwise we would always need an init_machine callback.
	 */
	if (machine_desc->init_machine)
		machine_desc->init_machine();

	return 0;
}
arch_initcall(customize_machine);

// !TODO : 什么时候init_machine完成初始化
DT_MACHINE_START(IMX6UL, "Freescale i.MX6 Ultralite (Device Tree)")
	.init_irq	= imx6ul_init_irq,
	.init_machine	= imx6ul_init_machine,  // 完成初始化
	.init_late	= imx6ul_init_late,
	.dt_compat	= imx6ul_dt_compat,
MACHINE_END


static void __init imx6ul_init_machine(void)
{
	imx_print_silicon_rev(cpu_is_imx6ull() ? "i.MX6ULL" : "i.MX6UL",
		imx_get_soc_revision());

	of_platform_default_populate(NULL, NULL, NULL);
	imx6ul_enet_init();
	imx_anatop_init();
	imx6ul_pm_init();
}

int of_platform_default_populate(struct device_node *root,
				 const struct of_dev_auxdata *lookup,
				 struct device *parent)
{
	return of_platform_populate(root, of_default_bus_match_table, lookup,
				    parent);
}
EXPORT_SYMBOL_GPL(of_platform_default_populate);

of_platform_default_populate

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
		rc = of_platform_bus_create(child, matches, lookup, parent, true); // 这里的root执行设备树根目录
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


of_amba_device_create(bus, bus_id, platform_data, parent)
-> dev = amba_device_alloc(NULL, 0, 0);
-> ret = amba_device_add(dev, &iomem_resource);



```

## 如何解析设备树中的节点--带有中断属性

```cpp
unsigned int irq_of_parse_and_map(struct device_node *dev, int index)
{
	struct of_phandle_args oirq;

	if (of_irq_parse_one(dev, index, &oirq))
		return 0;

	return irq_create_of_mapping(&oirq);
}
EXPORT_SYMBOL_GPL(irq_of_parse_and_map);

unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data)
{
	struct irq_fwspec fwspec;

	of_phandle_args_to_fwspec(irq_data->np, irq_data->args,
				  irq_data->args_count, &fwspec);

	return irq_create_fwspec_mapping(&fwspec);
}
EXPORT_SYMBOL_GPL(irq_create_of_mapping);

return irq_create_fwspec_mapping(&fwspec);
-> virq = irq_find_mapping(domain, hwirq);
-> virq = irq_domain_alloc_irqs(domain, 1, NUMA_NO_NODE, fwspec);
    -> irq_domain_alloc_descs
    -> return __irq_domain_alloc_irqs(domain, -1, nr_irqs, node, arg, false,
				       NULL);

return domain->ops->alloc(domain, irq_base, nr_irqs, arg);
static const struct irq_domain_ops gic_irq_domain_hierarchy_ops = {
	.translate = gic_irq_domain_translate,
	.alloc = gic_irq_domain_alloc,
	.free = irq_domain_free_irqs_top,
};

static int gic_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
				unsigned int nr_irqs, void *arg)
{
	int i, ret;
	irq_hw_number_t hwirq;
	unsigned int type = IRQ_TYPE_NONE;
	struct irq_fwspec *fwspec = arg;

	ret = gic_irq_domain_translate(domain, fwspec, &hwirq, &type); // 获取args翻译出来中断号和中断类型
	if (ret)
		return ret;

	for (i = 0; i < nr_irqs; i++) {
		ret = gic_irq_domain_map(domain, virq + i, hwirq + i); // 执行软硬件的映射，并根据中断类型设置struct irq_desc->handle_irq处理函数。
		if (ret)
			return ret;
	}

	return 0;
}

int __irq_domain_alloc_irqs(struct irq_domain *domain, int irq_base,
			    unsigned int nr_irqs, int node, void *arg,
			    bool realloc, const struct irq_affinity_desc *affinity)
{
	int i, ret, virq;

	if (domain == NULL) {
		domain = irq_default_domain;
		if (WARN(!domain, "domain is NULL; cannot allocate IRQ\n"))
			return -EINVAL;
	}

	if (realloc && irq_base >= 0) {
		virq = irq_base;
	} else {
		virq = irq_domain_alloc_descs(irq_base, nr_irqs, 0, node, // 从allocated_irqs位图中查找第一个nr_irqs个空闲的比特位，最终调用__irq_alloc_descs
					      affinity);
		if (virq < 0) {
			pr_debug("cannot allocate IRQ(base %d, count %d)\n",
				 irq_base, nr_irqs);
			return virq;
		}
	}

	if (irq_domain_alloc_irq_data(domain, virq, nr_irqs)) { // 分配struct irq_data
		pr_debug("cannot allocate memory for IRQ%d\n", virq);
		ret = -ENOMEM;
		goto out_free_desc;
	}

	mutex_lock(&irq_domain_mutex);
	ret = irq_domain_alloc_irqs_hierarchy(domain, virq, nr_irqs, arg); // 调用struct irq_domain中的alloc回调函数进行硬件中断号和软件中断号的映射
	if (ret < 0) {
		mutex_unlock(&irq_domain_mutex);
		goto out_free_irq_data;
	}

	for (i = 0; i < nr_irqs; i++) {
		ret = irq_domain_trim_hierarchy(virq + i);
		if (ret) {
			mutex_unlock(&irq_domain_mutex);
			goto out_free_irq_data;
		}
	}

	for (i = 0; i < nr_irqs; i++)
		irq_domain_insert_irq(virq + i);
	mutex_unlock(&irq_domain_mutex);

	return virq;

out_free_irq_data:
	irq_domain_free_irq_data(virq, nr_irqs);
out_free_desc:
	irq_free_descs(virq, nr_irqs);
	return ret;
}
EXPORT_SYMBOL_GPL(__irq_domain_alloc_irqs);
```
以上完成了中断设备树的解析，各数据结构的初始化，以及最主要的硬件中断号到linux中断号的映射。

### ARM底层中断处理
ARM底层中断处理的范围是从中断异常触发，到irq_handler。


参考文档：
[Linux中断管理 (1)Linux中断管理机制](https://www.cnblogs.com/arnoldlu/p/8659981.html)


## 中断控制器初始化
qemu环境学习：
```cpp
        intc@8000000 {
                phandle = <0x8001>;
                interrupts = <0x01 0x09 0x04>;
                reg = <0x00 0x8000000 0x00 0x10000 0x00 0x8010000 0x00 0x10000 0x00 0x8030000 0x00 0x10000 0x00 0x8040000 0x00 0x10000>;
                compatible = "arm,cortex-a15-gic";
                ranges;
                #size-cells = <0x02>;
                #address-cells = <0x02>;
                interrupt-controller;
                #interrupt-cells = <0x03>;

                v2m@8020000 {
                        phandle = <0x8002>;
                        reg = <0x00 0x8020000 0x00 0x1000>;
                        msi-controller;
                        compatible = "arm,gic-v2m-frame";
                };
        };


        pl011@9000000 {
                clock-names = "uartclk\0apb_pclk";
                clocks = <0x8000 0x8000>;
                interrupts = <0x00 0x01 0x04>;
                reg = <0x00 0x9000000 0x00 0x1000>;
                compatible = "arm,pl011\0arm,primecell";
        };

root@qemuarm64:~# cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3
 11:       4260       5007       5102       4536     GIC-0  27 Level     arch_timer
 43:       1212          0          0          0     GIC-0  78 Edge      virtio0
 44:         35          0          0          0     GIC-0  79 Edge      virtio1
 46:          0          0          0          0     GIC-0  34 Level     rtc-pl031
 47:         22          0          0          0     GIC-0  33 Level     uart-pl011
 48:          0          0          0          0     GIC-0  23 Level     arm-pmu
 51:          0          0          0          0  9030000.pl061   3 Edge      GPIO Key Poweroff
IPI0:       563       1015       1351       1041       Rescheduling interrupts
IPI1:      1279        479        460        375       Function call interrupts
IPI2:         0          0          0          0       CPU stop interrupts
IPI3:         0          0          0          0       CPU stop (for crash dump) interrupts
IPI4:         0          0          0          0       Timer broadcast interrupts
IPI5:       138        559        616        562       IRQ work interrupts
IPI6:         0          0          0          0       CPU wake-up interrupts
Err:          0

```
struct irq_domain用于描述一个中断控制器。GIC中断控制器在初始化时解析DTS信息中定义了几个GIC控制器，每个GIC控制器注册一个struct irq_domain数据结构。
