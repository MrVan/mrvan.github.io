---
layout: post
title: Linux reserved memory
description: Linux reserved memory
category: blog
---

在kernel启动时，`early_init_fdt_Scan_reserved_mem`负责构建rmem数组。

```c
 682 /**
 683  * early_init_fdt_scan_reserved_mem() - create reserved memory regions
 684  *
 685  * This function grabs memory from early allocator for device exclusive use
 686  * defined in device tree structures. It should be called by arch specific code
 687  * once the early allocator (i.e. memblock) has been fully activated.
 688  */
 689 void __init early_init_fdt_scan_reserved_mem(void)
 690 {
 691         int n;
 692         u64 base, size;
 693
 694         if (!initial_boot_params)
 695                 return;
 696
 697         /* Process header /memreserve/ fields */
 698         for (n = 0; ; n++) {
 699                 fdt_get_mem_rsv(initial_boot_params, n, &base, &size);
 700                 if (!size)
 701                         break;
 702                 early_init_dt_reserve_memory_arch(base, size, 0);
 703         }
 704
 705         of_scan_flat_dt(__fdt_scan_reserved_mem, NULL);
 706         fdt_init_reserved_mem();
 707 }
```

此处initial_boot_params其实是device tree所在的虚拟地址。
699行，查询memreserve结点，然后702行会把相应的address space写入一个memblock结构体，
memblock结构主要是在用在kernel boot阶段，我们知道kernel使用的是buddy page系统，
在kernel启动过程中，会有一个从memblock到buddy page的切换过程，所有没有被标记为reserve的
memblock会被释放然后迁移到buddy page系统，这也就意味着，所以被reserve的memblock，不会
进入buddy page系统，也就不会被kernel的mm子系统所管理。
705行，遍历所有的dts节点，执行`__fdt_scan_reserved_mem`函数，这里面核心的代码主要是：

```c
 674         err = __reserved_mem_reserve_reg(node, uname);
 675         if (err == -ENOENT && of_get_flat_dt_prop(node, "size", NULL))
 676                 fdt_reserved_mem_save_node(node, uname, 0, 0);
```

`__reserved_mem_reserve_reg`，获取reg里面的address和size，以及no-map属性。然后在
memblock系统里面reserve这部分memory，同时`fdt_reserved_mem_save_node`把相关的信息
存入`reserved_mem`数组，方便后面使用。如果没有提供reg属性，那么就只在`reserved_mem`中
创建一个entry存储相关信息，但这里的base和size给的值都为0，和dts里面我们一般给一个size不一致。
这里的主要目的是为了和reg区分。如果这里的size被赋值，那么后面就没法区分是否有提供reg属性了。
在`fdt_init_reserved_mem`中会检测size，如果size为0，那么表示这个rmem不是基于reg的，就会
重新从dts里面获取size值。
`fdt_init_reserved_mem` 分配和初始化所有的reserved memory regions.
1. 首先检查是否有overlap
2. 遍历rmem数组
   - 获取phandle或者linux,phandle
   - 如果`mem->size`0，表明dts里面没有提供reg，而是提供的size,这样就需要在memblock系统中分配一段内存，reserve出来，如果分配成功，会填充rmem数组中的base和size，到这里就和提供reg属性的rmem没有区别了。
   - `__reserved_mem_init_node`，遍历`__reservedmem_of_table`，执行initfn函数。

所有`__reservedmem_of_table`中的项都是通过`ESERVEDMEM_OF_DECLARE`来注册的。
