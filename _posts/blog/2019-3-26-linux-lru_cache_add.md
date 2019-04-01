---
layout: post
title: Linux lru_cache_add
description: Linux LRU
category: blog
---

lru\_cache\_add Queue the page for addition to the LRU via pagevec.
是否加入\[in\]active \[file|anon\] 列表会被推迟到pagevec被drain. 这样就
可以给lru\_cache\_add的调用者一个机会通过mark\_page\_accessed来把page
重新加入active list.

```
 436 void lru_cache_add(struct page *page)
 437 {
 438         VM_BUG_ON_PAGE(PageActive(page) && PageUnevictable(page), page);
 439         VM_BUG_ON_PAGE(PageLRU(page), page);
 440         __lru_cache_add(page);
 441 }
```

438行，如果page是active和unevitable状态，表明出现了bug，unevictable表明不
可以被回收

439行，page已经在LRU list里面，重复加入也表明出现bug

`__lru_cache_add`是把page加入pagevec,如果pagevec满了，把pagevec加入LRU,
如果page是PageCompound则不判断pagevec是否已满直接drain pagevc

```
 398 static void __lru_cache_add(struct page *page)
 399 {
 400         struct pagevec *pvec = &get_cpu_var(lru_add_pvec);
 401
 402         get_page(page);
 403         if (!pagevec_add(pvec, page) || PageCompound(page))
 404                 __pagevec_lru_add(pvec);
 405         put_cpu_var(lru_add_pvec);
 406 }
```

400,405行是获取和释放lru\_add\_pvec percpu变量, get\_cpu\_var 和 put\_cpu\_var
会禁止当前cpu的调度，这是必须的，不然进程迁移会导致错误的percpu变量访问.

403行，把page加入到pagevec中，如果是compound page或者pagevec已满则直接加入lru

404行\_\_pagevec\_lru\_add,遍历pvec中的page加入page所在的pgdata的lrurec list

```c
 914 /*
 915  * Add the passed pages to the LRU, then drop the caller's refcount
 916  * on them.  Reinitialises the caller's pagevec.
 917  */
 918 void __pagevec_lru_add(struct pagevec *pvec)
 919 {
 920         pagevec_lru_move_fn(pvec, __pagevec_lru_add_fn, NULL);
 921 }
 922 EXPORT_SYMBOL(__pagevec_lru_add);
```

这里我们不考虑memcg的话,`mem_cgroup_page_lruvec(page, pgdat);`直接返回
当前pgdat的lrurec list/array,核心函数当然是\_\_pagevec\_lru\_add\_fn

```c
 859 static void __pagevec_lru_add_fn(struct page *page, struct lruvec *lruvec,
 860                                  void *arg)
 861 {
 862         enum lru_list lru;
 863         int was_unevictable = TestClearPageUnevictable(page);
 864
 865         VM_BUG_ON_PAGE(PageLRU(page), page);
 866
 867         SetPageLRU(page);
 868         /*
 869          * Page becomes evictable in two ways:
 870          * 1) Within LRU lock [munlock_vma_pages() and __munlock_pagevec()].
 871          * 2) Before acquiring LRU lock to put the page to correct LRU and then
 872          *   a) do PageLRU check with lock [check_move_unevictable_pages]
 873          *   b) do PageLRU check before lock [clear_page_mlock]
 874          *
 875          * (1) & (2a) are ok as LRU lock will serialize them. For (2b), we need
 876          * following strict ordering:
 877          *
 878          * #0: __pagevec_lru_add_fn             #1: clear_page_mlock
 879          *
 880          * SetPageLRU()                         TestClearPageMlocked()
 881          * smp_mb() // explicit ordering        // above provides strict
 882          *                                      // ordering
 883          * PageMlocked()                        PageLRU()
 884          *
 885          *
 886          * if '#1' does not observe setting of PG_lru by '#0' and fails
 887          * isolation, the explicit barrier will make sure that page_evictable
 888          * check will put the page in correct LRU. Without smp_mb(), SetPageLRU
 889          * can be reordered after PageMlocked check and can make '#1' to fail
 890          * the isolation of the page whose Mlocked bit is cleared (#0 is also
 891          * looking at the same page) and the evictable page will be stranded
 892          * in an unevictable LRU.
 893          */
 894         smp_mb();
 895
 896         if (page_evictable(page)) {
 897                 lru = page_lru(page);
 898                 update_page_reclaim_stat(lruvec, page_is_file_cache(page),
 899                                          PageActive(page));
 900                 if (was_unevictable)
 901                         count_vm_event(UNEVICTABLE_PGRESCUED);
 902         } else {
 903                 lru = LRU_UNEVICTABLE;
 904                 ClearPageActive(page);
 905                 SetPageUnevictable(page);
 906                 if (!was_unevictable)
 907                         count_vm_event(UNEVICTABLE_PGCULLED);
 908         }
 909
 910         add_page_to_lru_list(page, lruvec, lru);
 911         trace_mm_lru_insertion(page, lru);
 912 }
```

863行判断当前page was\_evictable, was表示之前,不表示现在, 并且清掉这个标志位.

865行如果page已经设置LRU状态,表明有bug,因为page已经在LRU链表里了,再次加入LRU
肯定有bug.

867行设置page的LRU标志

894行smp\_mb,为什么要加呢？防止编译器和处理器乱序访问.根据注释,如果\#1没有
观察到\#0处的LRU置位,显示的barrier会确保page\#evictable 检查可以把page放入
正确的LRU.如果没有smp\#mb,SetPageLRU可能会被重排到PageMlocked之后,这样\#1
处page ioslation会失败,这是\#0处也同时在访问这个page的flags,那么就会把
evictable page放入unevictable LRU,所以需要显示的smp\_mb.

896行如果page是evictable,page\_lru是根据page来找到这个page应该被放在哪个LRU list

902行else代码段,表示page不能被evict,清active标志,置位Unevictable标志

910行把page加入对应的lru链表
