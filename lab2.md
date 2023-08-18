[题目要求](https://objectkuan.gitbooks.io/ucore-docs/content/lab2/lab2_3_2_1_phymemlab_exercise.html)
> 本次实验包含三个部分。首先了解如何发现系统中的物理内存；然后了解如何建立对物理内存的初步管理，即了解连续物理内存管理；最后了解页表相关的操作，即如何建立页表来实现虚拟内存到物理内存之间的映射，对段页式内存管理机制有一个比较全面的了解。本实验里面实现的内存管理还是非常基本的，并没有涉及到对实际机器的优化，比如针对 cache 的优化等。如果大家有余力，尝试完成扩展练习。

# 练习1：实现firest-fit连续物理内存分配算法
#### flag标志位的设置
flags 表示此物理页的状态标记，进一步查看 kern/mm/memlayout.h 中的定义，可以看到：

```
/* Flags describing the status of a page frame */
#define PG_reserved                 0       // the page descriptor is reserved for kernel or unusable
#define PG_property                 1       // the member 'property' is valid
```

这表示 flags 目前用到了两个 bit 表示页目前具有的两种属性，bit 0 表示此页是否被保留（reserved），如果是被保留的页，则 bit 0 会设置为 1，且不能放到空闲页链表中，即这样的页不是空闲页，不能动态分配与释放。比如目前内核代码占用的空间就属于这样“被保留”的页。在本实验中，bit 1 表示此页是否是 free 的，如果设置为 1，表示这页是 free 的，可以被分配；如果设置为 0，表示这页已经被分配出去了，不能被再二次分配。
设置函数：
```c
#define SetPageReserved(page) set_bit(PG_reserved, &((page)->flags))
#define ClearPageReserved(page) clear_bit(PG_reserved, &((page)->flags))
#define PageReserved(page) test_bit(PG_reserved, &((page)->flags))
#define SetPageProperty(page) set_bit(PG_property, &((page)->flags))
#define ClearPageProperty(page) clear_bit(PG_property, &((page)->flags))
#define PageProperty(page) test_bit(PG_property, &((page)->flags))
```
#### `default_init()`函数：初始化链表
```c
free_area_t free_area;
#define free_list (free_area.free_list)
#define nr_free (free_area.nr_free)

static void default_init(void)
{
	//初始化双向链表free_list
	list_init(&free_list);
	//设置链表中的空闲页面数为0
	nr_free = 0;
}
```
#### `default_init_memmap()`函数：初始化页面
```c
//对n个页面初始化
static void default_init_memmap(struct Page *base, size_t n)
{
	//确保n>0
	assert(n > 0);
	struct Page *p = base;
	for (; p != base + n; p++)
	{
		//确保页面是预先保留的
		assert(PageReserved(p));
		// 初始化页面的标志位和属性字段
		// flags是描述页框状态的标志数组
		// property是记录地址连续的空闲页的个数
		p->flags = p->property = 0;
		// 设置页面的引用计数ref = 0
		set_page_ref(p, 0);
	}
	// 表示该段连续页面数量为n
	base->property = n;
	//set_bit(PG_property, &((page)->flags))
	//设置flags的bit 0 为0（默认为1）
	//具体原因查看https://learningos.github.io/ucore_os_webdocs/lab2/lab2_3_3_3_phymem_pagelevel.html
	SetPageProperty(base);
	//空闲页面数量+n
	nr_free += n;
	//将base的链接page_link添加到链表中
	list_add(&free_list, &(base->page_link));
}
```
#### `default_alloc_pages()`函数：分配页面
```c
//分配n和页面
static struct Page *default_alloc_pages(size_t n)
{
	//确保要分配的页面大于0,小于空闲页面数
	assert(n > 0);
	if (n > nr_free)
	{
		return NULL;
	}
	
	struct Page *page = NULL;
	//声明一个指向空闲页面链表头节点的的指针le
	list_entry_t *le = &free_list;
	//遍历空闲页面链表
	while ((le = list_next(le)) != &free_list)
	{
		//led2page()函数将le转换为Page指针p
		struct Page *p = le2page(le, page_link);
		//如果当前页面p的空间大于等于要分配的数量n
		if (p->property >= n)
		{
			page = p;
			break;
		}
	}
	if (page != NULL)
	{
		//从空闲页面链表中删除要分配的页面
		list_del(&(page->page_link));
		//如果该页面的空间大于要分配的空间
		if (page->property > n)
		{
			//将剩余空间添加到空闲页面链表中
			struct Page *p = page + n;
			p->property = page->property - n;
			SetPageProperty(p);
			list_add(&free_list, &(p->page_link));
		}
		//减去分配的页面数量
		nr_free -= n;
		//清除页面属性标志位
		ClearPageProperty(page);
	}
	return page;
}
```
#### `default_free_pages()`函数：释放页面
```c
//释放页面
static void default_free_pages(struct Page *base, size_t n)
{
	assert(n > 0);
	struct Page *p = base;
	for (; p != base + n; p++)
	{
		//确保当前页面不是保留页面也不是属性页面
		assert(!PageReserved(p) && !PageProperty(p));
		p->flags = 0;
		//将引用计数ref置0
		set_page_ref(p, 0);
	}
	//表示该段连续页面数量为n
	base->property = n;
	SetPageProperty(base);
	list_entry_t *le = list_next(&free_list);
	//循环遍历链表
	while (le != &free_list)
	{
		//将le转换为Page类型指针
		p = le2page(le, page_link);
		//更新指针，指向下一个节点
		le = list_next(le);
		//当前页面是否是base页面的后继页面
		//base是一个指向页面的指针
		//base->property是base的内存空间块大小（连续页面数量）
		//所以bse + bse->property是将指针移动到base指向的页面的下一个页面
		if (base + base->property == p)
		{
			//合并页面，将当前界面p加到base中
			base->property += p->property;
			//清楚p的属性标志位
			ClearPageProperty(p);
			//从空闲页面链表中删除p
			list_del(&(p->page_link));
		}
		else if (p + p->property == base)
		{
			p->property += base->property;
			ClearPageProperty(base);
			base = p;
			list_del(&(p->page_link));
		}
	}
	//页面数量+n
	nr_free += n;
	le = list_next(&free_list);
	//在链表中查找释放页面应该插入的页面
	//以保持链表中页面的地址排序顺序
	while (le != &free_list)
	{
		p = le2page(le, page_link);
		if (base + base->property <= p)
		{
			//如果改地址等于当前页面p的起始地址使
			//表示页面之间存在重叠，非法情况
			assert(base + base->property != p);
			break;
		}
		le = list_next(le);
	}
	list_add_before(le, &(base->page_link));
}
```
----
# 练习2：实现寻找虚拟地址对应的页表项（补全get_pte函数）
[分段、分页、段页的概念](https://blog.csdn.net/qq_35423154/article/details/111084633)
#### get_pte()函数调用关系图
![](pic/Pasted%20image%2020230817200415.png)
#### 函数功能及参数
[建立二级页表](https://learningos.github.io/ucore_os_webdocs/lab2/lab2_3_3_5_3_setup_paging_map.html)
```c
// get_pte - 获取 pte 并返回该 pte 的内核虚拟地址给 la
// 如果 PT 包含此 pte 不存在，则为 PT 分配一个页面
// 范围：
// pgdir：PDT的内核虚拟基地址
// la：需要映射的线性地址
// create：一个逻辑值来决定是否概念为PT分配页面
// 返回值：该pte的内核虚拟地址
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create);
```
这里涉及到三个类型`pte_t`、`pde_t`和`uintptr_t`。通过参见`mm/mmlayout.h`和`libs/types.h`，可知它们其实都是`unsigned int`类型。在此做区分，是为了分清概念。
`pde_t`全称为`page directory entry`，也就是一级页表的表项（注意：`pgdir`实际不是表项，而是一级页表本身。实际上应该心定义一个类型`pgd_t`来表示一级页表本身）。
`pte_t`全称为`page table entry`，表示二级页表的表项。`uintptr_t`表示为线性地址，由于段式管理只做直接映射，所以它也是逻辑地址。
`pgdir`给出页表起始地址。通过查找这个页表，我们需要给出二级页表中对应项的地址。虽然目前我们只有`boot_pgdir`一个页表，但是引入进程的概念之后每个进程都会有自己的页表。
有可能根本就没有对应的二级页表的情况，所以二级页表不必要一开始就分配，而是等到需要的时候再添加对应的二级页表。如果在查找二级页表项时发现对应的二级页表不存在，则需要根据`create`参数的值来处理是否是创建新的二级页表。如果`create`参数为0，则`get_pte`返回`NULL`；如果`create`参数不为0，则`get_pte`需要申请一个新的物理页（通过`alloc_page`来实现，可在`mm/pnn.h`中找到它的定义），再在一级页表中添加页目录项指向表示二级页表的新物理页（注意：新申请的页必须全部设定为0，因为这个页所代表的虚拟地址都没有被映射）。
当建立从一级页表到二级页表的映射时，需要注意设置控制位。这里应该设置同时设置上 `PTE_U`、`PTE_W` 和 `PTE_P`（定义可在 `mm/mmu.h`）。如果原来就有二级页表，或者新建立了页表，则只需返回对应项的地址即可。
#### 函数实现
```c
/*代码中用到的宏或函数
* PDX(la) = VIRTUAL ADDRESS la 的页目录项的索引。
* KADDR(pa) ：获取物理地址并返回相应的内核虚拟地址。
* set_page_ref(page,1) : 表示被引用一次的页面
* page2pa(page): 获取该(struct Page *)页管理的内存物理地址
* struct Page * alloc_page() : 分配一个页面
* memset(void *s, char c, size_t n) : 设置s指向的内存区域的前n个字节到指定值 c.
*/
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create)
{
	// 查找目录项
	pde_t *pdep = &pgdir[PDX(la)];
	// 检查条目是否不存在
	if (!(*pdep & PTE_P))
	{
		struct Page *page;
		// 检查是否需要申请页，然后为页表分配页面
		if (!create || (page = alloc_page()) == NULL)
		{
			return NULL;
		}
		// 设置页面引用
		set_page_ref(page, 1);
		//获取页的线性地址
		uintptr_t pa = page2pa(page);
		// 使用memset来清楚页面内容，初始化页面
		memset(KADDR(pa), 0, PGSIZE);
		// 设置页目录项的全性
		*pdep = pa | PTE_U | PTE_W | PTE_P;
	}
	// 返回页表项
	return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
}
```
代码逻辑如下：
1. `pde_t *pdep = &pgdir[PDX(la)];`：获取页目录项的地址
	- `pgdir`：`pgdir` 是一个指向页目录（Page Directory）的内核虚拟基地址的指针。
	- `PDX(la)`：`PDX` 是一个宏定义，用于从线性地址 `la` 中提取高 10 位，即获取页目录索引（Page Directory Index）。
	- `pgdir[PDX(la)]`：通过将页目录基地址 `pgdir` 视为一个数组，使用 `PDX(la)` 作为索引，可以得到对应页目录项的内容。这里的数组访问方式相当于对 `pgdir` 指针进行偏移，从而得到指定索引处的页目录项。
	- `&pgdir[PDX(la)]`：在获取到页目录项的内容后，通过在前面加上取地址符 `&`，可以获得页目录项的地址。
2. `if (!(*pdep & PTE_P))`：判断页目录项是否有效，即是否已经存在页表。`PTE_P` 是一个宏定义，表示页表项的存在位（Present bit）。如果页表项无效（不存在页表），则执行下面的代码块。
3.  `struct Page *page;`：定义一个指向物理页结构的指针 `page`，用于存储分配的物理页的信息。
4.  `if (!create || (page = alloc_page()) == NULL)`：根据参数 `create` 的值和物理页的分配情况，判断是否需要创建新的页表项。如果 `create` 为假或者无法分配到物理页，则返回空指针。
5. `set_page_ref(page, 1);`：设置物理页的引用计数为 1，表示该物理页被使用一次。
6. `uintptr_t pa = page2pa(page);`：通过 `page2pa` 函数将物理页结构指针转换为物理地址（Physical Address），将其赋给变量 `pa`。
7. `memset(KADDR(pa), 0, PGSIZE);`：通过 `KADDR` 宏将物理地址转换为内核虚拟地址，然后使用 `memset` 函数将该地址对应的内存清零。这个步骤是可选的，可以根据具体需求初始化页表的内容。
8. `*pdep = pa | PTE_U | PTE_W | PTE_P;`：将页目录项设置为新创建的页表的物理地址，并设置相关标志位，包括用户位（PTE_U）、写入位（PTE_W）和存在位（PTE_P）。
9. `return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];`：通过 `PDE_ADDR(*pdep)` 
	- `*pdep`：`pdep` 是一个指向页目录项（Page Directory Entry）的指针。通过 `*pdep`，访问指针所指向的内存地址，获取该地址处的页目录项的内容。
	- `PDE_ADDR(*pdep)`：`PDE_ADDR` 是一个宏定义，用于从页目录项中提取出物理页框地址（Page Frame Address）。通过执行该宏定义，可以从页目录项的内容中获取物理页框的地址。
	- `KADDR(PDE_ADDR(*pdep))`：`KADDR` 是一个宏定义，用于将物理地址转换为内核虚拟地址。通过执行该宏定义，将物理页框地址转换为对应的内核虚拟地址。
	- `(pte_t *)KADDR(PDE_ADDR(*pdep))`：将转换后的内核虚拟地址强制类型转换为指向页表项类型 `pte_t` 的指针。
	- `PTX(la)`：`PTX` 是一个宏定义，用于从线性地址 `la` 中提取中间 10 位，即获取页表索引（Page Table Index）。
	- `((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)]`：通过将步骤 4 中得到的指针视为数组，使用 `PTX(la)` 作为索引，可以得到对应的页表项的内容。
	- `&((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)]`：在获取到页表项的内容后，通过在前面加上取地址符 `&`，可以获得页表项的地址。
----
# 练习3：释放某虚拟地址所在的页并取消对应二级页表项的映射(完善page_remove_pte函数)
#### TLB
**TLB**是一种高速缓存，用于存储最近访问的虚拟地址与物理地址的映射关系。它位于CPU内部，用于加速虚拟地址到物理地址的转换过程。当CPU执行指令时，会将虚拟地址发送到TLB进行查找，以获取对应的物理地址，从而访问正确的内存位置。
#### 函数参数
```c
// page_remove_pte - 释放与线性地址la相关的Page structct和 clean(invalidate) pte 相关的线性地址 la
// 注意：PT改变了，所以TLB需要失效
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep)
```
#### 函数实现
```c
	  /* 
      * 请检查ptep是否有效，如果映射更新则必须手动更新tlb
      * 也许你需要帮助注释，下面的注释可以帮助你完成代码
      * 一些有用的宏和定义，您可以在下面的实现中使用它们。
      * 宏或函数：
      * struct Page *page pte2page(*ptep): 从 ptep 的值中获取相应的页面
      * free_page : 释放一个页面
      * page_ref_dec(page) : 减少page->ref。 注意： ff page->ref == 0 ，那么这个页面应该是免费的。
      * tlb_invalidate(pde_t *pgdir, uintptr_t la) ：使 TLB 条目无效，但前提是页表是
      * 编辑的是处理器当前正在使用的那些。
      * 定义：
      * PTE_P 0x001 //页表/目录条目标志位：存在
      */
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep)
{
	// 检查页表项是否存在
	if (*ptep & PTE_P)
	{
		// 找到pte对应的页面
		struct Page *page = pte2page(*ptep);
		// 减少页面引用
		if (page_ref_dec(page) == 0)
		{
			// 当页面引用达到0时释放该页面
			free_page(page);
		}
		// 消除第二页表项
		*ptep = 0;
		// 刷新tlb
		tlb_invalidate(pgdir, la);
	}
}
```
代码逻辑：
- 首先，通过检查页表项指针 `ptep` 所指向的值 `*ptep` 是否满足 PTE_P 标志位（存在位）的条件。PTE_P 标志位用于表示页表项指向的页面是否存在于物理内存中。
- 如果满足 PTE_P 标志位的条件，即页面存在于物理内存中，那么执行下面的操作。
- 通过函数 `pte2page(*ptep)` 将页表项转换为对应的页面结构指针 `page`，这个函数用于将页表项的物理地址转换为对应的页面结构的指针。
- 调用函数 `page_ref_dec(page)` 对页面的引用计数进行递减操作，并返回递减后的引用计数值。引用计数用于跟踪页面的使用情况，当引用计数为零时，表示页面没有被任何对象引用，可以被释放。
- 如果递减后的引用计数值为零，即没有其他对象引用该页面，那么调用函数 `free_page(page)` 来释放该页面的内存空间。
- 将页表项 `*ptep` 的值设置为 0，表示页面不再存在于物理内存中。
- 调用函数 `tlb_invalidate(pgdir, la)` 来使 TLB（转换后备缓冲器）中的相关缓存项无效化，以确保下次访问该页面时能够获取最新的映射关系。
