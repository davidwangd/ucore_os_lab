## Lab2 物理内存管理 实验报告
-----------------------------
计65 王元炜 2016011383

### 练习1：连续物理内存分配算法

除去一些非常小的页的属性并不完整之外。在default_ppm.c中的函数已经帮我们实现了一个类似随机fit的分配算法，在添加上一些比如property和refd的属性修改之外，只需要将free_area所维护的链表中的内容按照地址从小到大进行排序即可。这里主要讲维护链表顺序的操作（分配和释放）。

这里注意一点就是在分配内存的时候需要为所有分配出去的内存打上property的reserved的标记。
```C
for (struct Page* cur = page; cur != page + n; cur++){
    ClearPageProperty(cur);
    ClearPageReserved(cur);
    set_page_ref(cur, 0);
}
```

#### 分配内存

分配内存只有在将一大块内存块分配出去一小部分的时候会遇到顺序问题，我们从头线性查找，找到剩下部分应该插入的位置进行插入即可，注意对于循环链表，list_add_after(&head)相当于插入最后的位置。

```C
    if (page != NULL) {
        list_del(&(page->page_link));
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;

            int flag = 0;
            le = &free_list;
            while ((le = list_next(le)) != &free_list){
                struct Page *now = le2page(le, page_link);
                if ((uint32_t)now > (uint32_t)p){
                    list_add_before(&(now->page_link), &(p->page_link));
                    flag = 1;
                    break;
                }
            }
            if (!flag) list_add_before(&free_list, &(p->page_link));
    ...
```

#### 释放内存
同理，找到合适的位置即可。这个需要对已有的框架进行一点点的修改。

具体来说就是如果找到第一个后面的，在合并之前先插入，然后再直接合并之后退出即可，就是一个考虑线段合并（最多连续两次）的插入排序。依然需要注意如果没有找到合并的位置需要放到链表的最后的位置。

```C
    while (le != &free_list) {
        p = le2page(le, page_link);
        //cprintf("[%d,%d]", (uint32_t)p/sizeof(struct Page), p->property);
        le = list_next(le);
        if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            base = p;
            list_del(&(p->page_link));
        }
        if ((uint32_t)p > (uint32_t)base){
            list_add_before(&(p->page_link), &(base->page_link));
            flag = 1;
            if (base + base->property == p) {
                base->property += p->property;
                ClearPageProperty(p);
                list_del(&(p->page_link));
            }
            break;
        }
    }
    nr_free += n;
    if(!flag){
        list_add_before(&free_list, &(base->page_link));
    }
```

### 寻找虚拟地址对应的页表项

给出一个虚拟地址，需要周到该虚拟地址的二级页表对应项，如果没有根据create参数创建一个
+ @pgdir 是内核虚拟内存的PDT表的起始位置。
+ @la 是线性地址
+ @create 是如果不存在是否创建一个页。

可以通过PDX(la)来得到线性地址所对应的PDT的数组下标。从而找到我们需要的二级页表对应项的地址指针。以地址的高8位加上偏移量就是我们需要的二级页表地址。

与之同时我们需要考虑不存在的情况，创建一个页面，然后有
```
页目录项内容 = (页表起始物理地址 & ~0x0FFF) | PTE_U | PTE_W | PTE_P
```
以这个决定实际额页表内容。

```C
    uint32_t* pd = pgdir + PDX(la);     //找到虚拟页地址            
    if (!(*pd & PTE_P)){                //判断是否present，不present的话根据create判断是否创建
        if (!create) return NULL;
        struct Page *page = alloc_page();   
        if (page == NULL) return NULL;
        uint32_t pa = page2pa(page);        //找到撞见的起始物理位置
        set_page_ref(page, 1);              // 
        memset(KADDR(pa), 0, sizeof(struct Page));  //初始化内核虚拟内存
        *pd = (pa) | PTE_P | PTE_W | PTE_U;     //计算页目录项内容
    }
    // 返回二级页表
    return ((pte_t *)KADDR(PDE_ADDR(*pd))) + PTX(la);
```

### 练习3： 释放虚地址所在页并取消对应二级页表映射

这里我并没有特别弄明白一些问题，大概是指在二级页表对应的映射为0的时候需要将其释放（为什么是invalid_tlb）

但是按照注释给出的提示还是能够比较理想的完成代码

代码如下

```C
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    uint32_t* pd = pgdir + PDX(la);
    // 检测参数合法性
    if (&((pte_t *)KADDR(PDE_ADDR(*pd)))[PTX(la)] != ptep){
        cprintf("Wrong with the parameter!\n");
        return;
    }
    // 检测是否present
    if (*ptep & PTE_P){
        // 找到页表
        struct Page* page = pte2page(*ptep);
        if (page == NULL) return;
        page_ref_dec(page);
        //将ref--， 如果为0则invalid_tlb
        if (page_ref(page) == 0){
            tlb_invalidate(pgdir, la);
        }
    }
```