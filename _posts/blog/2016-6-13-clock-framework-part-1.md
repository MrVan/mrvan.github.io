---
layout: post
title: clock framework part 1
description: clock framework part 1
category: blog
---

clk framework是基于consumer和provider来设计的。consumer故名思议是使用clk的个体。这里我们看一下consumer可以使用的API有哪些

```c
int clk_prepare(struct clk *clk);
void clk_unprepare(struct clk *clk);
struct clk *clk_get(struct device *dev, const char *id);
struct clk *devm_clk_get(struct device *dev, const char *id);
int clk_enable(struct clk *clk);
void clk_disable(struct clk *clk);
unsigned long clk_get_rate(struct clk *clk);
void clk_put(struct clk *clk);
void devm_clk_put(struct device *dev, struct clk *clk);
long clk_round_rate(struct clk *clk, unsigned long rate);
int clk_set_rate(struct clk *clk, unsigned long rate);
bool clk_has_parent(struct clk *clk, struct clk *parent);
int clk_set_rate_range(struct clk *clk, unsigned long min, unsigned long max);
int clk_set_min_rate(struct clk *clk, unsigned long rate);
int clk_set_max_rate(struct clk *clk, unsigned long rate);
int clk_set_parent(struct clk *clk, struct clk *parent);
struct clk *clk_get_parent(struct clk *clk);
struct clk *clk_get_sys(const char *dev_id, const char *con_id);
static inline int clk_prepare_enable(struct clk *clk)
static inline void clk_disable_unprepare(struct clk *clk)
```

clk_prepare:在使用clk前，要先clk_prepare,这主要是因为对于像PLL这种需要一定的时间才能稳定下来的clk，我们一般的用法是`clk_prepare + clk_enable`或者直接`clk_prepare_enable`.
clk_unprepare和clk_prepare不可以用在中断和原子上下文，因为会引发睡眠.
clk_get/devm_clk_get:根据传入的device指针以及clk的名字查找对应的clk结构体，在驱动中我们一般这样用：`clk_ipg = devm_clk_get(&pdev->dev, "ipg");`,在dts里面的描述如下：

```c
clocks = <&clks IMX6UL_CLK_USDHC1>,                                                               
         <&clks IMX6UL_CLK_USDHC1>,                                                               
         <&clks IMX6UL_CLK_USDHC1>;                                                               
clock-names = "ipg", "ahb", "per";
```

clk_round_rate：注释是这样的：

```c
/**                                                                             
 * clk_round_rate - adjust a rate to the exact rate a clock can provide         
 * @clk: clock source                                                           
 * @rate: desired clock rate in Hz                                              
 *                                                                              
 * This answers the question "if I were to pass @rate to clk_set_rate(),        
 * what clock rate would I end up with?" without changing the hardware          
 * in any way.  In other words:                                                 
 *                                                                              
 *   rate = clk_round_rate(clk, r);                                             
 *                                                                              
 * and:                                                                            
 *                                                                              
 *   clk_set_rate(clk, r);                                                       
 *   rate = clk_get_rate(clk);                                                  
 *                                                                              
 * are equivalent except the former does not modify the clock hardware          
 * in any way.                                                                     
 *                                                                              
 * Returns rounded clock rate in Hz, or negative errno.                         
 */
```

clk_round_rate会根据传入的rate值，返回一个当前clk支持的频率值，并且不修改硬件寄存器，当然这里说不touch硬件，但很多provider的实现里面还是会对硬件进行操作.
clk_has_parent：查询某个clk1是否可以作为clk2的parent，当然不是grandparent.
clk_set_rate_range:配置某个clk的最大最小值，同时会修正当前的clk.
clk_get_sys:根据device的名字以及clk的名字，找到对应的clk结构体.

支持devicetree的API:

```c
struct clk *of_clk_get(struct device_node *np, int index);
struct clk *of_clk_get_by_name(struct device_node *np, const char *name);
struct clk *of_clk_get_from_provider(struct of_phandle_args *clkspec);

```
