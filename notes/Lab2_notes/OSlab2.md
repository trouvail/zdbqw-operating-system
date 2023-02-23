# `Lab2`——物理内存管理

> 学号：2013921  
姓名：周延霖  
专业：信息安全



实验一做出来了一个可以启动的系统，实验二主要涉及操作系统的物理内存管理。操作系统为了使用内存，还需高效地管理内存资源。在实验二中会了解并且动手完成一个简单的物理内存管理系统。

***实验目的如下：***

- 理解基于段页式内存地址的转换机制
- 理解页表的建立和使用方法
- 理解物理内存的管理方法



## 问题一：不太清楚各个部分之间的关系

> 因为实验在理论课之前，所以做实验的时候并不熟悉页目录项，页表项等等之间的关系


***解决：***


- 首先需要知道PDT(页目录表),PDE(页目录项),PTT(页表),PTE(页表项)之间的关系:页表保存页表项，页表项被映射到物理内存地址；页目录表保存页目录项，页目录项映射到页表。
  - PDE和PTE都是4B大小的一个元素，其高20bit被用于保存索引，低12bit用于保存属性，但是由于用处不同，内部具有细小差异，如以下两图所示：

![pde.png](https://i.loli.net/2020/11/04/7DvHaPVIACfyjbN.png)

![pte.png](https://i.loli.net/2020/11/04/Yqdz7hTnWtEBsSc.png)



- 其各个标识位的含义如下：

| bit  | PDE                                                          | PTE                                                        |
| ---- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| 0    | Present位，0不存在，1存在下级页表                            | 同                                                         |
| 1    | Read/Write位，0只读，1可写                                   | 同                                                         |
| 2    | User/Supervisor位，0则其下页表/物理页用户无法访问，1可以访问 | 同                                                         |
| 3    | Page level Write Through，1则开启页层次的写回机制，0不开启   | 同                                                         |
| 4    | Page level Cache Disable， 1则禁止页层次缓存，0不禁止        | 同                                                         |
| 5    | Accessed位，1代表在地址翻译过程中曾被访问，0没有             | 同                                                         |
| 6    | 忽略                                                         | 脏位，判断是否有写入                                       |
| 7    | PS，当且仅当PS=1且CR4.PSE=1，页大小为4M，否则为4K            | 如果支持 PAT 分页，间接决定这项访问的页的内存类型，否则为0 |
| 8    | 忽略                                                         | Global 位。当 CR4.PGE 位为 1 时,该位为1则全局翻译          |
| 9    | 忽略                                                         | 忽略                                                       |
| 10   | 忽略                                                         | 忽略                                                       |
| 11   | 忽略                                                         | 忽略                                                       |






## 问题二：不太清楚各个CR寄存器的作用






***解决：***


- 首先需要将发生错误的线性地址la保存在CR2寄存器中
  - 这里说一下控制寄存器CR0-4的作用
  - CR0的0位是PE位，如果为1则启动保护模式，其余位也有自己的作用
  - CR1是未定义控制寄存器，留着以后用
  - CR2是**页故障线性地址寄存器**，保存最后一次出现页故障的全32位线性地址
  - CR3是**页目录基址寄存器**，保存PDT的物理地址
  - CR4在Pentium系列处理器中才实现，它处理的事务包括诸如何时启用虚拟8086模式等
- 之后需要往中断时的栈中压入EFLAGS,CS,EIP,ERROR CODE，如果这页访问异常很不巧发生在用户态，还需要先压入SS,ESP并切换到内核态
- 最后根据IDT表查询到对应的也访问异常的ISR，跳转过去并将剩下的部分交给软件处理。


## 问题三：不太清楚获取页表项的具体方法



***解决：***

- 在查询课本及实验指导数后发现：
  1. 抹去低12位，只保留对应的PTT的起始基地址
  2. 用PTX(la)获得PTT对应的PTE的索引
  3. 用数组和对应的索引得到PTE并返回

具体实现如下：

~~~c
return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)]; 
~~~


la,PDX,PTX,PTE_ADDR,PDE_ADDR的定义可以在mmu.h下看到，具体内容如下

~~~c
// A linear address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
//
// The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).

// page directory index
#define PDX(la) ((((uintptr_t)(la)) >> PDXSHIFT) & 0x3FF)

// page table index
#define PTX(la) ((((uintptr_t)(la)) >> PTXSHIFT) & 0x3FF)

// offset in page
#define PGOFF(la) (((uintptr_t)(la)) & 0xFFF)

// address in page table or page directory entry
#define PTE_ADDR(pte)   ((uintptr_t)(pte) & ~0xFFF)
#define PDE_ADDR(pde)   PTE_ADDR(pde)
~~~

其高10位用作PDT的索引，中10位用作PTT的索引，末12位用作物理页偏移量

KADDR，page2pa可以在pmm.h中看到

```c
/* *
 * KADDR - takes a physical address and returns the corresponding kernel virtual
 * address. It panics if you pass an invalid physical address.
 * */
#define KADDR(pa) ({                                                    \
            uintptr_t __m_pa = (pa);                                    \
            size_t __m_ppn = PPN(__m_pa);                               \
            if (__m_ppn >= npage) {                                     \
                panic("KADDR called with invalid pa %08lx", __m_pa);    \
            }                                                           \
            (void *) (__m_pa + KERNBASE);                               \
        })

static inline ppn_t
page2ppn(struct Page *page) {
    return page - pages;
}
// PGSHIFT = log2(PGSIZE) = 12
static inline uintptr_t
page2pa(struct Page *page) {
    return page2ppn(page) << PGSHIFT;
}
```


> PDE,PDT,PTT,PTE真的很容易让人弄混，而且他们的发音真的很像，一不留神就弄混了。搞清楚关系和结构看一下注释就能很快地搞定。


## 问题四：Page的全局变量的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？




***解决：***


- 所有的物理页都有一个描述它的Page结构，所有的页表都是通过alloc_page()分配的，每个页表项都存放在一个Page结构描述的物理页中；如果 PTE 指向某物理页，同时也有一个Page结构描述这个物理页。所以有两种对应关系：

- 可以通过 PTE 的地址计算其所在的页表的Page结构：
  - 将虚拟地址向下对齐到页大小，换算成物理地址(减 KERNBASE), 再将其右移 PGSHIFT(12)位获得在pages数组中的索引PPN，&pages[PPN]就是所求的Page结构地址。  

- 可以通过 PTE 指向的物理地址计算出该物理页对应的Page结构：
  - PTE 按位与 0xFFF获得其指向页的物理地址，再右移 PGSHIFT(12)位获得在pages数组中的索引PPN，&pages[PPN]就 PTE 指向的地址对应的Page结构。






## 最后：`Challenge1——buddy system`实现



**分配内存：**

1.寻找大小合适的内存块（大于等于所需大小并且最接近2的幂，比如需要27，实际分配32）

​    1.1 如果找到了，分配给应用程序。
​    1.2 如果没找到，分出合适的内存块。

​       1.2.1 对半分离出高于所需大小的空闲内存块
​       1.2.2 如果分到最低限度，分配这个大小。
​       1.2.3 回溯到步骤1（寻找合适大小的块）
​       1.2.4 重复该步骤直到一个合适的块

**释放内存：**

1.释放该内存块

   1.1 寻找相邻的块，看其是否释放了。
   1.2 如果相邻块也释放了，合并这两个块，重复上述步骤直到遇上未释放的相邻块，或者达到最高上限（即所有内存都释放了）。

​        在此定义之下，我们使用数组分配器来模拟构建这样完全二叉树结构而不是真的用指针建立树结构——树结构中向上或向下的指针索引都通过数组分配器里面的下标偏移来实现。在这个“完全二叉树”结构中，二叉树的节点用于标记相应内存块的使用状态，高层节点对应大的块，低层节点对应小的块，在分配和释放中我们就通过这些节点的标记属性来进行块的分离合并。

​        在分配阶段，首先要搜索大小适配的块——这个块所表示的内存大小刚好大于等于最接近所需内存的2次幂；通过对树深度遍历，从左右子树里面找到最合适的，将内存分配。

​        在释放阶段，我们将之前分配出去的内存占有情况还原，并考察能否和同一父节点下的另一节点合并，而后递归合并，直至不能合并为止。

![image-20201110000901267.png](https://i.loli.net/2020/11/11/rVJYyaZokCqDPT1.png)

​       基于上面的理论准备，我们可以开始写代码了。因为buddy system替代的是之前的first fit算法，所以可以仿照default_pmm的格式来写。

​       首先，编写buddy.h（仿照default_pmm.h），唯一修改的地方是引入的pmm_manager不一样，要改成buddy system所使用的buddy_pmm_manager

```C
#ifndef __KERN_MM_BUDDY_PMM_H__
#define  __KERN_MM_BUDDY_PMM_H__

#include <pmm.h>

extern const struct pmm_manager buddy_pmm_manager;

#endif /* ! __KERN_MM_DEFAULT_PMM_H__ */
```

​        而后，进入buddy.c文件

   1. 因为这里使用数组来表示二叉树结构，所以需要建立正确的索引机制：每level的第一左子树的下标为2^level-1，所以如果我们得到[index]节点的所在level，那么offset的计算可以归结为(index-2^level+1) * node_size = (index+1)*node_size – node_size*2^level。其中size的计算为2^(max_depth-level)，所以node_size * 2^level = 2^max_depth = size。综上所述，可得公式offset=(index+1)*node_size – size。
      PS：式中索引的下标均从0开始，size为内存总大小，node_size为内存块对应大小。

      由上，完成宏定义。

```C++
#define LEFT_LEAF(index) ((index) * 2 + 1)
#define RIGHT_LEAF(index) ((index) * 2 + 2)
#define PARENT(index) ( ((index) + 1) / 2 - 1)
```

          2.  因为buddy system的块大小都是2的倍数，所以我们对于输入的所需的块大小，要先计算出最接近于它的2的倍数的值以匹配buddy system的最合适块的查找。

```C
static unsigned fixsize(unsigned size) {
  size |= size >> 1;
  size |= size >> 2;
  size |= size >> 4;
  size |= size >> 8;
  size |= size >> 16;
  return size+1;
}
```

3. 构造buddy system最基本的数据结构，并初始化一个用来存放二叉树的数组。

```C++
struct buddy {
  unsigned size;//表明管理内存
  unsigned longest; 
};
struct buddy root[10000];//存放二叉树的数组，用于内存分配
```

4. buddy system是需要和实际指向空闲块双链表配合使用的，所以需要先各自初始化数组和指向空闲块的双链表。

```C++
//先初始化双链表
free_area_t free_area;
#define free_list (free_area.free_list)
#define nr_free (free_area.nr_free)

static void buddy_init()
{
    list_init(&free_list);
    nr_free=0;
}

//再初始化buddy system的数组
void buddy_new( int size ) {
  unsigned node_size;    //传入的size是这个buddy system表示的总空闲空间；node_size是对应节点所表示的空闲空间的块数
  int i;
  nr_block=0;
  if (size < 1 || !IS_POWER_OF_2(size))
    return;

  root[0].size = size;
  node_size = size * 2;   //认为总结点数是size*2

  for (i = 0; i < 2 * size - 1; ++i) {
    if (IS_POWER_OF_2(i+1))    //如果i+1是2的倍数，那么该节点所表示的二叉树就要到下一层了
      node_size /= 2;
    root[i].longest = node_size;   //longest是该节点所表示的初始空闲空间块数
  }
  return;
}
```

5. 根据pmm.h里面对于pmm_manager的统一结构化定义，我们需要对buddy system完成如下函数：

```C++
const struct pmm_manager buddy_pmm_manager = {
    .name = "buddy_pmm_manager",      // 管理器的名称
    .init = buddy_init,               // 初始化管理器
    .init_memmap = buddy_init_memmap, // 设置可管理的内存,初始化可分配的物理内存空间
    .alloc_pages = buddy_alloc_pages, // 分配>=N个连续物理页,返回分配块首地址指针 
    .free_pages = buddy_free_pages,   // 释放包括自Base基址在内的，起始的>=N个连续物理内存页
    .nr_free_pages = buddy_nr_free_pages, // 返回全局的空闲物理页数量
    .check = buddy_check,             //举例检测这个pmm_manager的正确性
};
```

​    5.1 初始化管理器（这个已在上面完成）

​    5.2 初始化可管理的物理内存空间

```C++
static void
buddy_init_memmap(struct Page *base, size_t n)
{
    assert(n>0);
    struct Page* p=base;
    for(;p!=base + n;p++)
    {
        assert(PageReserved(p));
        p->flags = 0;
        p->property = 1;
        set_page_ref(p, 0);    //表明空闲可用
        SetPageProperty(p);
        list_add_before(&free_list,&(p->page_link));     //向双链表中加入页的管理部分
    }
    nr_free += n;     //表示总共可用的空闲页数
    int allocpages=UINT32_ROUND_DOWN(n);
    buddy2_new(allocpages);    //传入所需要表示的总内存页大小，让buddy system的数组得以初始化
}
```

​    5.3 分配所需的物理页，返回分配块首地址指针

```C++
//分配的逻辑是：首先在buddy的“二叉树”结构中找到应该分配的物理页在整个实际双向链表中的位置，而后把相应的page进行标识表明该物理页已经分出去了。
static struct Page*
buddy_alloc_pages(size_t n){
  assert(n>0);
  if(n>nr_free)
   return NULL;
  struct Page* page=NULL;
  struct Page* p;
  list_entry_t *le=&free_list,*len;
  rec[nr_block].offset=buddy2_alloc(root,n);//记录偏移量
  int i;
  for(i=0;i<rec[nr_block].offset+1;i++)
    le=list_next(le);
  page=le2page(le,page_link);
  int allocpages;
  if(!IS_POWER_OF_2(n))
   allocpages=fixsize(n);
  else
  {
     allocpages=n;
  }
  //根据需求n得到块大小
  rec[nr_block].base=page;//记录分配块首页
  rec[nr_block].nr=allocpages;//记录分配的页数
  nr_block++;
  for(i=0;i<allocpages;i++)
  {
    len=list_next(le);
    p=le2page(le,page_link);
    ClearPageProperty(p);
    le=len;
  }//修改每一页的状态
  nr_free-=allocpages;//减去已被分配的页数
  page->property=n;
  return page;
}


//以下是在上面的分配物理内存函数中用到的结构和辅助函数
struct allocRecord//记录分配块的信息
{
  struct Page* base;
  int offset;
  size_t nr;//块大小，即包含了多少页
};

struct allocRecord rec[80000];//存放偏移量的数组
int nr_block;//已分配的块数

int buddy2_alloc(struct buddy2* self, int size) { //size就是这次要分配的物理页大小
  unsigned index = 0;  //节点的标号
  unsigned node_size;  //用于后续循环寻找合适的节点
  unsigned offset = 0;

  if (self==NULL)//无法分配
    return -1;

  if (size <= 0)//分配不合理
    size = 1;
  else if (!IS_POWER_OF_2(size))//不为2的幂时，取比size更大的2的n次幂
    size = fixsize(size);

  if (self[index].longest < size)//根据根节点的longest，发现可分配内存不足，也返回
    return -1;

  //从根节点开始，向下寻找左右子树里面找到最合适的节点
  for(node_size = self->size; node_size != size; node_size /= 2 ) {
    if (self[LEFT_LEAF(index)].longest >= size)
    {
       if(self[RIGHT_LEAF(index)].longest>=size)
        {
           index=self[LEFT_LEAF(index)].longest <= self[RIGHT_LEAF(index)].longest? LEFT_LEAF(index):RIGHT_LEAF(index);
         //找到两个相符合的节点中内存较小的结点
        }
       else
       {
         index=LEFT_LEAF(index);
       }  
    }
    else
      index = RIGHT_LEAF(index);
  }

  self[index].longest = 0;//标记节点为已使用
  offset = (index + 1) * node_size - self->size;  //offset得到的是该物理页在双向链表中距离“根节点”的偏移
  //这个节点被标记使用后，要层层向上回溯，改变父节点的longest值
  while (index) {
    index = PARENT(index);
    self[index].longest = 
      MAX(self[LEFT_LEAF(index)].longest, self[RIGHT_LEAF(index)].longest);
  }
  return offset;
}

```

   5.4 释放指定的内存页大小

```C++
void buddy_free_pages(struct Page* base, size_t n) {
  unsigned node_size, index = 0;
  unsigned left_longest, right_longest;
  struct buddy2* self=root;
  
  list_entry_t *le=list_next(&free_list);
  int i=0;
  for(i=0;i<nr_block;i++)  //nr_block是已分配的块数
  {
    if(rec[i].base==base)  
     break;
  }
  int offset=rec[i].offset;
  int pos=i;//暂存i
  i=0;
  while(i<offset)
  {
    le=list_next(le);
    i++;     //根据该分配块的记录信息，可以找到双链表中对应的page
  }
  int allocpages;
  if(!IS_POWER_OF_2(n))
   allocpages=fixsize(n);
  else
  {
     allocpages=n;
  }
  assert(self && offset >= 0 && offset < self->size);//是否合法
  nr_free+=allocpages;//更新空闲页的数量
  struct Page* p;
  for(i=0;i<allocpages;i++)//回收已分配的页
  {
     p=le2page(le,page_link);
     p->flags=0;
     p->property=1;
     SetPageProperty(p);
     le=list_next(le);
  }
  
  //实际的双链表信息复原后，还要对“二叉树”里面的节点信息进行更新
  node_size = 1;
  index = offset + self->size - 1;   //从原始的分配节点的最底节点开始改变longest
  self[index].longest = node_size;   //这里应该是node_size，也就是从1那层开始改变
  while (index) {//向上合并，修改父节点的记录值
    index = PARENT(index);
    node_size *= 2;
    left_longest = self[LEFT_LEAF(index)].longest;
    right_longest = self[RIGHT_LEAF(index)].longest;
    
    if (left_longest + right_longest == node_size) 
      self[index].longest = node_size;
    else
      self[index].longest = MAX(left_longest, right_longest);
  }
  for(i=pos;i<nr_block-1;i++)//清除此次的分配记录，即从分配数组里面把后面的数据往前挪
  {
    rec[i]=rec[i+1];
  }
  nr_block--;//更新分配块数的值
}
```

   5.5 返回全局的空闲物理页数

```C++
static size_t
buddy_nr_free_pages(void) {
    return nr_free;
}
```

​    5.6 检查这个pmm_manager是否正确

```C++
//以下是一个测试函数
static void
buddy_check(void) {
    struct Page  *A, *B;
    A = B  =NULL;

    assert((A = alloc_page()) != NULL);
    assert((B = alloc_page()) != NULL);

    assert( A != B);
    assert(page_ref(A) == 0 && page_ref(B) == 0);
  //free page就是free pages(p0,1)
    free_page(A);
    free_page(B);
    
    A=alloc_pages(500);     //alloc_pages返回的是开始分配的那一页的地址
    B=alloc_pages(500);
    cprintf("A %p\n",A);
    cprintf("B %p\n",B);
    free_pages(A,250);     //free_pages没有返回值
    free_pages(B,500);
    free_pages(A+250,250);
    
}
```