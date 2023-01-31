---
title: Hixxxx PCIe + SMMU bad performance debug
tags:
  - SMMU
  - 软件性能
description: 记录一个SMMU相关的性能问题的调试过程
abbrlink: c8b64683
date: 2021-07-11 23:16:26
categories:
---

When we enabled SMMU in Dxx board, we found that the performance of 82599 pluged
in PCIe slot is very bad. LeiZhen and I spent some to debug this problem. This
document just shares the related results and information.

test scenarios and results
-----------------------------
```
     +----------------+        +---------------+
     |   Dxx   82599  |<------>|  82599   Dxx  |
     +----------------+        +---------------+

           +-----+         +-----+
           | cpu |         | ddr |
           +--+--+         +--+--+
              |               |
      --------+-----+---------+------- bus
                    |
                 +--+---+
                 | smmu |
                 +--+---+
                    |                   
                 +--+---+
                 |  rp  |
                 +--+---+
                    |
                 +--+---+
                 | 82599|
                 +------+
```
Hardware topology as showed above. In order to use SMMU to translate data from
82599 to DDR, we need enable SMMU node in ACPI table.[1]

Then boot up two Dxx boards connected by two 82599 networking cards. When using
iperf to test the performance between two 82599 networking cards, it is very bad,
nearly 100Mbps.[2]

analysis
-----------

The only difference is disable SMMU and enable SMMU. So the difference is we use
diffferent DMA callbacks.

We can see arch/arm64/mm/dma-mapping.c, when configuring CONFIG_IOMMU_DMA,
callbacks in struct dma_map_ops iommu_dma_ops will be used.

And we also know when 82599 sending/receiving packages, its driver will call
ixgbe_tx_map/ixgbe_alloc_mapped_page to allocate DMA memory function which will
finally call map_page in iommu_dma_ops when SMMU is enable. So we guess there
is something wrong with the map_page here.

So we should analyze related function to find the hot point. Here we firstly use
ftrace to confirm our idea, then use perf to locate the hot point explicitly.

 * ftrace

   We can use function profiling in ftrace to see durations of related function.
   please refer[3] to know how to use ftrace function profiling.

   Here we get:
```
10)<...>-4313   |               |   ixgbe_xmit_frame_ring() {
10)<...>-4313   |               |     __iommu_map_page() {
10)<...>-4313   |   0.080 us    |       dma_direction_to_prot();
10)<...>-4313   |               |       iommu_dma_map_page() {
10)<...>-4313   |               |         __iommu_dma_map() {
10)<...>-4313   |   0.480 us    |           iommu_get_domain_for_dev();
10)<...>-4313   |               |           __alloc_iova() {
10)<...>-4313   |               |             alloc_iova() {
10)<...>-4313   |               |               kmem_cache_alloc() {
10)<...>-4313   |               |                 __slab_alloc.isra.21()
10)<...>-4313   |   0.040 us    |                 memcg_kmem_put_cache();
10)<...>-4313   | + 16.160 us   |               }
[...]
10)<...>-4313   |   0.120 us    |               _raw_spin_lock_irqsave();
10)<...>-4313   |               |               _raw_spin_unlock_irqrestore() {
10)<...>-4313   |   ==========> |
10)<...>-4313   |               |                 gic_handle_irq()
10)<...>-4313   |   <========== |
10)<...>-4313   | + 88.620 us   |               }
10)<...>-4313   | ! 679.760 us  |             }
10)<...>-4313   | ! 680.480 us  |           }
```
   Most time has been spent in alloc_iova.

 * perf

   Sadly, there was no perf(PMU hardware event) support in ACPI in plinth
   kernel :( So we directly set PMU related registers to get how many CPU cycles
   each function has spent.
```
	/* Firstly call this init function to init PMU */
	static void pm_cycle_init(void)
	{
		u64 val;

		asm volatile("mrs %0, pmccfiltr_el0" : "=r" (val));
		if (val & ((u64)1 << 31)) {
			val &= ~((u64)1 << 31);
			asm volatile("msr pmccfiltr_el0, %0" :: "r" (val));
			dsb(sy);
			asm volatile("mrs %0, pmccfiltr_el0" : "=r" (val));
		}

		asm volatile("mrs %0, pmcntenset_el0" : "=r" (val));
		if (!(val & ((u64)1 << 31))) {
			val |= ((u64)1 << 31);
			asm volatile("msr pmcntenset_el0, %0" :: "r" (val));
			dsb(sy);
			asm volatile("mrs %0, pmcntenset_el0" : "=r" (val));
		}

		asm volatile("mrs %0, pmcr_el0" : "=r" (val));
		if (!(val & ((u64)1 << 6))) {
			val |= ((u64)1 << 6) | 0x1;
			asm volatile("msr pmcr_el0, %0" :: "r" (val));
			dsb(sy);
			asm volatile("mrs %0, pmcr_el0" : "=r" (val));
		}
	}

	/* Get the CPU cycles in PMU counter */
	u64 pm_cycle_get(void)
	{
		u64 val;

		asm volatile("mrs %0, pmccntr_el0" : "=r" (val));

		return val;
	}
	EXPORT_SYMBOL(pm_cycle_get);
```
   Using above debug functions, we found almost 600000 CPU cycles will happen in
   a while loop in function __alloc_and_insert_iova_range. If CPU frequency is
   2G Hz, then 600000 CPU cycles is 300us! This is the hot point.

   We found it will loop almost 10000 times in above while loop!


 * Code analysis 

   Firstly, this DMA software modules is like this:
``` 
   VA = kmalloc();
   IOVA = dma_map_function(PA = fun(VA)); 
```
   Firstly, allocate memory for DMA memory and map a VA which can be used by CPU,
   Then, build map between IOVA and PA in dma map function. In the case of SMMU
   enable, .map_page(__iommu_map_page) in iommu_dma_ops will be call to build
   above map.

   Then common function iommu_dma_map_page in drivers/iommu/dma-iommu.c will be
   called. There are two steps in above function: 1. allocate iova, this is a
   common function; 2. build map between IOVA and PA, this is SMMU specific
   function.   

   The hot point is in the point 1 above, so we need understand the module of
   how to allocate iova. A red black tree in iova_domain is used to store all iova
   range in system, after allocating or freeing an iova range, an iova range
   should be inserted or remove from above red black tree. Now we allocate the
   iova range from the end of the iova domain, for 32 DMA mask, it is 0xffffffff,
   for 64bit DMA mask it is 0xffffffff_ffffffff. There is a cache for 32bit DMA
   mask to store the iova range in last time, but for 64bit DMA MASK, there is
   no such cache. so for 64bit DMA, when we want to allocate a DMA range in
   iova domain, we have to search from the 0xffffffff_ffffffff. If we already allocate
   a lot iova range, then we have to search all iova range allocated before.


solution
-----------

So we can fix this bug like this:

```
diff --git a/drivers/iommu/iova.c b/drivers/iommu/iova.c
index 080beca..1e582d8 100644
--- a/drivers/iommu/iova.c
+++ b/drivers/iommu/iova.c
@@ -46,6 +46,7 @@ init_iova_domain(struct iova_domain *iovad, unsigned long granule,
 	spin_lock_init(&iovad->iova_rbtree_lock);
 	iovad->rbroot = RB_ROOT;
 	iovad->cached32_node = NULL;
+	iovad->cached64_node = NULL;
 	iovad->granule = granule;
 	iovad->start_pfn = start_pfn;
 	iovad->dma_32bit_pfn = pfn_32bit;
@@ -56,13 +57,19 @@ EXPORT_SYMBOL_GPL(init_iova_domain);
 static struct rb_node *
 __get_cached_rbnode(struct iova_domain *iovad, unsigned long *limit_pfn)
 {
-	if ((*limit_pfn > iovad->dma_32bit_pfn) ||
-		(iovad->cached32_node == NULL))
+	struct rb_node *cached_node;
+
+	if (*limit_pfn < iovad->dma_32bit_pfn)
+		cached_node = iovad->cached32_node;
+	else
+		cached_node = iovad->cached64_node;
+
+	if (cached_node == NULL)
 		return rb_last(&iovad->rbroot);
 	else {
-		struct rb_node *prev_node = rb_prev(iovad->cached32_node);
+		struct rb_node *prev_node = rb_prev(cached_node);
 		struct iova *curr_iova =
-			container_of(iovad->cached32_node, struct iova, node);
+			container_of(cached_node, struct iova, node);
 		*limit_pfn = curr_iova->pfn_lo - 1;
 		return prev_node;
 	}
@@ -72,9 +79,10 @@ static void
 __cached_rbnode_insert_update(struct iova_domain *iovad,
 	unsigned long limit_pfn, struct iova *new)
 {
-	if (limit_pfn != iovad->dma_32bit_pfn)
-		return;
-	iovad->cached32_node = &new->node;
+	if (limit_pfn <= iovad->dma_32bit_pfn)
+		iovad->cached32_node = &new->node;
+	else
+		iovad->cached64_node = &new->node;
 }
 
 static void
@@ -82,21 +90,26 @@ __cached_rbnode_delete_update(struct iova_domain *iovad, struct iova *free)
 {
 	struct iova *cached_iova;
 	struct rb_node *curr;
+	struct rb_node **cached_node;
+
+	if (free->pfn_hi <= iovad->dma_32bit_pfn)
+		cached_node = &iovad->cached32_node;
+	else
+		cached_node = &iovad->cached64_node;
 
-	if (!iovad->cached32_node)
+	curr = *cached_node;
+	if(!curr)
 		return;
-	curr = iovad->cached32_node;
 	cached_iova = container_of(curr, struct iova, node);
 
 	if (free->pfn_lo >= cached_iova->pfn_lo) {
 		struct rb_node *node = rb_next(&free->node);
 		struct iova *iova = container_of(node, struct iova, node);
 
-		/* only cache if it's below 32bit pfn */
-		if (node && iova->pfn_lo < iovad->dma_32bit_pfn)
-			iovad->cached32_node = node;
+		if (node)
+			*cached_node = node;
 		else
-			iovad->cached32_node = NULL;
+			*cached_node = NULL;
 	}
 }
 
diff --git a/include/linux/iova.h b/include/linux/iova.h
index f27bb2c..d4670c1 100644
--- a/include/linux/iova.h
+++ b/include/linux/iova.h
@@ -41,6 +41,7 @@ struct iova_domain {
 	spinlock_t	iova_rbtree_lock; /* Lock to protect update of rbtree */
 	struct rb_root	rbroot;		/* iova domain rbtree root */
 	struct rb_node	*cached32_node; /* Save last alloced node */
+	struct rb_node	*cached64_node; /* Save last 64bit alloced node */
 	unsigned long	granule;	/* pfn granularity for this domain */
 	unsigned long	start_pfn; 	/* Lower limit for this domain */
 	unsigned long	dma_32bit_pfn;
```

above solution just adds another cache for 64bit DMA Mask.

But now Linux kernel community just merged a PATCH:
        iommu/dma: Implement PCI allocation optimisation
into mainline kernel.

This patch just castes 64bit DMA mask to 32bit DMA mask, so we can still use
32bit DMA cache to improve the performance.

NOTE: but if this we can not allocate a 64bit iova to a PCIe device's DMA target
      address. This is a problem :(


problem
----------

 * Performance

   After SMMU enable and applying above patch, 82599 performance is 7.5Gbps,
   only 80% performance comparing SMMU disable. We need check if this is correct
   considering both hardware and software limitation.
 
 * NIC panic

   After SMMU enable and applying above patch, xxx net will panic :( should fix this.
   (p.s. already find where the problem is, xxx net dma map once, but unmap multiple
    times)

reference
---------
[1] xxx
[2] JIRA bug
[3] https://lwn.net/Articles/370423/
    
    cd /sys/kernel/debug/tracing
    echo ixgbe_* > set_graph_function
    echo function_graph > current_tracer
    cat trace > ~/smmu_test
