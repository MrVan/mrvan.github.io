---
layout: post
title: Linux reserved memory part 2
description: Linux reserved memory
category: blog
---

首先来分析`drivers/base/dma-contiguous.c`中的cma代码。

```c
243 static int __init rmem_cma_setup(struct reserved_mem *rmem)
244 {
245         phys_addr_t align = PAGE_SIZE << max(MAX_ORDER - 1, pageblock_order);
246         phys_addr_t mask = align - 1;
247         unsigned long node = rmem->fdt_node;
248         struct cma *cma;
249         int err;
250
251         if (!of_get_flat_dt_prop(node, "reusable", NULL) ||
252             of_get_flat_dt_prop(node, "no-map", NULL))
253                 return -EINVAL;
254
255         if ((rmem->base & mask) || (rmem->size & mask)) {
256                 pr_err("Reserved memory: incorrect alignment of CMA region\n");
257                 return -EINVAL;
258         }
259
260         err = cma_init_reserved_mem(rmem->base, rmem->size, 0, &cma);
261         if (err) {
262                 pr_err("Reserved memory: unable to setup CMA region\n");
263                 return err;
264         }
265         /* Architecture specific contiguous memory fixup. */
266         dma_contiguous_early_fixup(rmem->base, rmem->size);
267
268         if (of_get_flat_dt_prop(node, "linux,cma-default", NULL))
269                 dma_contiguous_set_default(cma);
270
271         rmem->ops = &rmem_cma_ops;
272         rmem->priv = cma;
273
274         pr_info("Reserved memory: created CMA memory pool at %pa, size %ld MiB\n",
275                 &rmem->base, (unsigned long)rmem->size / SZ_1M);
276
277         return 0;
278 }
279 RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup);
```

在part1中`__reserved_mem_init_node`会调用到`rmem_cma_setup`,我们知道cma中的内存其实是可以用来做dma，也可以用来在normal memory的，所以251行,如果dts结点中没有`reusable`或者有`no-map`，说明这个结点不是用来做cma的，直接返回`-EINVAL`。

255行，检测rmem的base和size是否符合对齐。这里暂且不表。

261行，`cma_init_reserved_mem`，主要是把base，size存入`cma_areas`数组。

268行，如果有`linux,cma-default`属性，就把当前cma赋值给`dma_contiguous_default_area`

271和272行，主要是给device使用reserved memory时使用。下面的dts展示的是per device的reserved memory做dma使用，这里给的是dma-coherent.c的例子，cma是同样的原理:

```
 27         reserved-memory {
 28                 #address-cells = <1>;
 29                 #size-cells = <1>;
 30                 ranges;
 31 
 32                 usdhc1_reserved: usdhc_dma@90000000 {
 33                         compatible = "shared-dma-pool";
 34                         no-map;
 35                         reg = <0x90000000 0x4000000>;
 36                 };
 37         };
714 &usdhc1 {
715         pinctrl-names = "default", "state_100mhz", "state_200mhz";
716         pinctrl-0 = <&pinctrl_usdhc1>;
717         pinctrl-1 = <&pinctrl_usdhc1_100mhz>;
718         pinctrl-2 = <&pinctrl_usdhc1_200mhz>;
719         cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>;
720         keep-power-in-suspend;
721         enable-sdio-wakeup;
722         vmmc-supply = <&reg_sd1_vmmc>;
723         status = "okay";
724         memory-region=<&usdhc1_reserved>;
725         dma-coherent;
726 };
```

在`include/linux/dma-contiguous.h`中`dev_get_cma_area`会首先查看dev结构体是否有`cma_area`，如果没有则使用`dma_contiguous_default_area`,那什么时候`dev->cma_area`会被赋值呢？

`rmem_cma_device_init`会给`dev->cma_area`赋值。

当cma ready之后，`arch/arm[64]/mm/dma-mapping.c`中的函数就可以使用cma来分配dma区域了。

`dma-contiguous.c`中的API主要为:

```c
dma_contiguous_reserve
dma_contiguous_reserve_area
dma_alloc_from_contiguous
dma_release_from_contiguous
···

`dma_alloc_from_contiguous`从per device的cma区域或者总体的cma区域`dma_contiguous_default_cma`中分配memory，给dma使用。

`drivers/base/dma-coherent.c`中的代码也是类似的方法，只是通过` dev->dma_mem`来进行dma内存分配，并且是per device的，没有global的。当然可以通过修改代码(hack)来做成global coherent dma，供所有的device使用。