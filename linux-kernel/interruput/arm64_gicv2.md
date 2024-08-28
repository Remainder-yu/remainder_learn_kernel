
以串口0设备为例，设备名称为“uart-pl011",该设备的硬件中断是GIC-33，Linux内核分配的中断号IRQ是39，122表示已经发生了这么多次中断。
```cpp
benshushu:~# cat /proc/interrupts 
           CPU0       CPU1       CPU2       CPU3       
  3:       7807       6971       6448       7202     GICv3  27 Level     arch_timer
 36:       2649          0          0          0     GICv3  79 Edge      virtio0
 38:          0          0          0          0     GICv3 106 Edge      arm-smmu-v3-evtq
 41:          0          0          0          0     GICv3 109 Edge      arm-smmu-v3-gerror
 42:          0          0          0          0     GICv3  34 Level     rtc-pl031
 43:        122          0          0          0     GICv3  33 Level     uart-pl011
 44:          0          0          0          0     GICv3  23 Level     arm-pmu
 46:          0          0          0          0   ITS-MSI 16384 Edge      virtio1-config
 47:         19          0          0          0   ITS-MSI 16385 Edge      virtio1-input.0
 48:          1          0          0          0   ITS-MSI 16386 Edge      virtio1-output.0
 50:          0          0          0          0   ITS-MSI 32768 Edge      virtio2-config
 51:          6          0          0          0   ITS-MSI 32769 Edge      virtio2-requests
IPI0:      1900       1805       1805       1172       Rescheduling interrupts
IPI1:       261        210        238        276       Function call interrupts
IPI2:         0          0          0          0       CPU stop interrupts
IPI3:         0          0          0          0       CPU stop (for crash dump) interrupts
IPI4:         0          0          0          0       Timer broadcast interrupts
IPI5:         0          0          0          0       IRQ work interrupts
IPI6:         0          0          0          0       CPU wake-up interrupts

```

linux中断分层
1. 硬件层：如CPU与中断控制器的连接
2. 处理器架构管理层，如CPU中断异常处理
3. 中断控制器管理层，如IRQ号的映射
4. Linux内核通用中断处理层，如中断注册和中断处理

## 硬件中断号和linux中断号的映射
注册中断接口函数：request_irq\request_threaded_irq(),使用linux内核软件中断号（俗称软件中断号或IRQ号），而不是硬件中断号。

```cpp
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
```

以串口为例：
```cpp
	pl011@9000000 {
		clock-names = "uartclk\0apb_pclk";
		interrupts = <0x00 0x01 0x04>;
		clocks = <0x8000 0x8000>;
		compatible = "arm,pl011\0arm,primecell";
		reg = <0x00 0x9000000 0x00 0x1000>;
	};

```

interrupts域描述的是3个属性：
1. 中断类型： PPI私有中断中断，该值为1，SPI共享外设中断。该值在设备树中为0；
2. 中断ID
3. 触发类型

linux内核有多种触发类型：
```cpp
include/linux/irq.h
enum {
	IRQ_TYPE_NONE		= 0x00000000,
	IRQ_TYPE_EDGE_RISING	= 0x00000001,
	IRQ_TYPE_EDGE_FALLING	= 0x00000002,
	IRQ_TYPE_EDGE_BOTH	= (IRQ_TYPE_EDGE_FALLING | IRQ_TYPE_EDGE_RISING),
	IRQ_TYPE_LEVEL_HIGH	= 0x00000004,
	IRQ_TYPE_LEVEL_LOW	= 0x00000008,
	IRQ_TYPE_LEVEL_MASK	= (IRQ_TYPE_LEVEL_LOW | IRQ_TYPE_LEVEL_HIGH),
	IRQ_TYPE_SENSE_MASK	= 0x0000000f,
	IRQ_TYPE_DEFAULT	= IRQ_TYPE_SENSE_MASK,

```

因此通过上述就可以知道这个串口设备的中断属性，它属于共享SPI类型，中断ID为1，中断触发类型为高电平触发.
