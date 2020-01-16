# 内存分配
## 前序课程
操作系统接口：https://dreamerjonson.com/2020/01/04/6-s081-1/

## 系统编程（Systems programming）
[wiki参考](https://en.wikipedia.org/wiki/Systems_programming)
* 与应用程序编程相比，系统编程的主要区别在于，应用程序编程旨在产生直接向用户提供服务的软件。
* 系统编程主要为其他应用程序提供服务，直接操作操作系统。它的目标是实现对可用资源的有效利用。

例如：
* unix utilities
* K/V servers
* ssh
* bigint library

挑战：
*  低级编程环境
   -  数字是64位（不是无限的整数）
   -  分配/释放内存
   * 并发
     - 允许并行处理请求
* 崩溃
* 性能
* 为应用程序提供硬件支持

动态内存分配是一项基本的系统服务
*  底层编程： 指针操作，类型转换
* 使用底层操作系统请求内存块（memory chunks）
* 性能非常重要
* 支持广泛类型的应用程序负载

## 应用程序结构
  * text
  * data  (静态存储)
  * stack (栈存储)
  * heap (动态内存分配)
    使用 sbrk() o或 mmap() 操作系统接口扩展堆。
  [text |  data | heap ->  ...    <- stack]
  0                                       top of address space

* data段的内存分配是静态的，始终存在。
* 栈的分配在函数中，随着函数消亡而释放。
* 堆的分配与释放，调用接口：
    — malloc(int sz)
    — free(p)

## 堆分配的目标
* 快速分配和释放
* 内存开销小
* 想要使用所有内存
* 避免碎片化

下面介绍几种malloc实现的方式

##  方式1：K&R malloc
又叫做first-fit规则, 即查找第一个可用的匹配块。与之相对应的是查找第一个最符合（best-fit）的可用块。
K&R malloc的实现来自书籍  the C programming language by Kernighan and Ritchie (K&R) Section 8.7


### 维持一个链表
维持的free list是一个环。 第一个元素是base。
```
#define NALLOC  1024  /* minimum #units to request */

struct header {
  struct header *ptr;
  size_t size;
};

typedef struct header Header;

static Header base;
static Header *freep = NULL;
```


### 内存分配
  指定分配的内存大小为sizeof(Header)的倍数，且一定大于nbytes。nunits就是这个倍数。
*  循环free list。 找到第一个符合即大于等于nbytes的块。
   — 如果刚好合适，则链表删除此块并返回此块。
   — 如果大于，则截断此元素
   — 如果没有找到合适的块，则调用moreheap新分配一个。
```
/* malloc: general-purpose storage allocator */
void *
kr_malloc(size_t nbytes)
{
  Header *p, *prevp;
  unsigned nunits;

  nunits = (nbytes + sizeof(Header) - 1) / sizeof(Header) + 1;
  // base作为第一个元素。
  if ((prevp = freep) == NULL) {	/* no free list yet */
    base.ptr = freep = prevp = &base;
    base.size = 0;
  }

  for (p = prevp->ptr; ; prevp = p, p = p->ptr) {
    if (p->size >= nunits) {	/* big enough */
      if (p->size == nunits)	/* exactly */
	prevp->ptr = p->ptr;
      else {	/* allocate tail end */
	p->size -= nunits;
	p += p->size;
	p->size = nunits;
      }
      freep = prevp;
      return (void *) (p + 1);
    }
    if (p == freep) {	/* wrapped around free list */
      if ((p = (Header *) moreheap(nunits)) == NULL) {
	return NULL;	/* none left */
      }
    }
  }
}
```


###  sbrk

sbrk是unix增加内存的操作系统调用。
[wiki](https://en.wikipedia.org/wiki/Sbrk)的解释是：
```
The brk and sbrk calls dynamically change the amount of space allocated for the data segment of the calling process. The change is made by resetting the program break of the process, which determines the maximum space that can be allocated. The program break is the address of the first location beyond the current end of the data region. The amount of available space increases as the break value increases. The available space is initialized to a value of zero, unless the break is lowered and then increased, as it may reuse the same pages in some unspecified way. The break value can be automatically rounded up to a size appropriate for the memory management architecture.[4]
Upon successful completion, the brk subroutine returns a value of 0, and the sbrk subroutine returns the prior value of the program break (if the available space is increased then this prior value also points to the start of the new area). If either subroutine is unsuccessful, a value of −1 is returned and the errno global variable is set to indicate the error.
```

由于这里是增加内存，sbrk成功时会返回新增加区域的开始地址，如果失败则会返回-1。
新生成一个块之后，调用free函数将其加入到freelist当中。
注意这里的up+1 是什么意思。其代表的是新分配的区域的开头。以为新分配的区域之前有Header大小，用于标识大小和下一个区域。
```
static Header *moreheap(size_t nu)
{
  char *cp;
  Header *up;

  if (nu < NALLOC)
    nu = NALLOC;
  cp = sbrk(nu * sizeof(Header));
  if (cp == (char *) -1)
    return NULL;
  up = (Header *) cp;
  up->size = nu;
  kr_free((void *)(up + 1));
  return freep;
}
```

###  free
 ap是要释放的区域，其前面还有Header大小。bp 指向了块的开头。
  * 遍历free list，找到要插入的中间区域。
  * 如果前后的区域正好是连在一起的，则进行合并。
```
/* free: put block ap in free list */
void
kr_free(void *ap)
{
  Header *bp, *p;

  if (ap == NULL)
    return;

  bp = (Header *) ap - 1;	/* point to block header */
  for (p = freep; !(bp > p && bp < p->ptr); p = p->ptr)
    if (p >= p->ptr && (bp > p || bp < p->ptr))
      break;	/* freed block at start or end of arena */

  if (bp + bp->sizfe == p->ptr) {	/* join to upper nbr */
    bp->size += p->ptr->size;
    bp->ptr = p->ptr->ptr;
  } else {
    bp->ptr = p->ptr;
  }

  if (p + p->size == bp) {	/* join to lower nbr */
    p->size += bp->size;
    p->ptr = bp->ptr;
  } else {
    p->ptr = bp;
  }

  freep = p;
}
```

## 方式2：Region-based allocator, a special-purpose allocator.
* malloc 与free 快速
* 内存开销低
* 内存碎片严重
* 不通用，用于特定应用程序。
```
struct region {
  void *start;
  void *cur;
  void *end;
};
typedef struct region Region;

static Region rg_base;

Region *
rg_create(size_t nbytes)
{
  rg_base.start = sbrk(nbytes);
  rg_base.cur = rg_base.start;
  rg_base.end = rg_base.start + nbytes;
  return &rg_base;
}

void *
rg_malloc(Region *r, size_t nbytes)
{
  assert (r->cur + nbytes <= r->end);
  void *p = r->cur;
  r->cur += nbytes;
  return p;
}

// free all memory in region resetting cur to start
void
rg_free(Region *r) {
  r->cur = r->start;
}

```

## 方式3：Buddy allocator
我觉得wiki的解释挺好的：
[wiki解析](https://en.wikipedia.org/wiki/Buddy_memory_allocation)
提示：
* 对于2^k 大小的空间，我们可以将其分割为大小为2^0, 2^1, 2^2, ... 2^k的多种可能。
*  malloc(17) 会分配 32 bytes，因此其会一定程度上浪费空间。
* 数据结构带来的格外内存开销
* malloc 和free快速。

下面介绍一种代码实现。

基本参数
* ROUNDUP(n,sz) 求出要分配n哥字节时，希望分配的实际内存是大于等于n 并且是sz的倍数。
```
#define LEAF_SIZE     16 // The smallest allocation size (in bytes)
#define NSIZES        15 // Number of entries in bd_sizes array
#define MAXSIZE       (NSIZES-1) // Largest index in bd_sizes array
#define BLK_SIZE(k)   ((1L << (k)) * LEAF_SIZE) // Size in bytes for size k
#define HEAP_SIZE     BLK_SIZE(MAXSIZE)
#define NBLK(k)       (1 << (MAXSIZE-k))  // Number of block at size k
#define ROUNDUP(n,sz) (((((n)-1)/(sz))+1)*(sz))  // Round up to the next multiple of sz
```
* NBLK求出在第k位置有多少块。

每一个大小k维护一个`sz_info`, `sz_info`中都有一个free list, alloc 是一个char数组用于记录块是否分配。split 是一个char数组用于块是否割裂。
这里要注意，使用的是bit数组来记录。  char有8位，如第n位代表的是当前k大小的第5个区块。

```
// The allocator has sz_info for each size k. Each sz_info has a free
// list, an array alloc to keep track which blocks have been
// allocated, and an split array to to keep track which blocks have
// been split.  The arrays are of type char (which is 1 byte), but the
// allocator uses 1 bit per block (thus, one char records the info of
// 8 blocks).
struct sz_info {
  struct bd_list free;
  char *alloc;
  char *split;
};
typedef struct sz_info Sz_info;

```


每一个级别的双链表 for free list
```
// A double-linked list for the free list of each level
struct bd_list {
  struct bd_list *next;
  struct bd_list *prev;
};

```

###  bit数组操作
```
// Return 1 if bit at position index in array is set to 1
int bit_isset(char *array, int index) {
  char b = array[index/8];
  char m = (1 << (index % 8));
  return (b & m) == m;
}

// Set bit at position index in array to 1
void bit_set(char *array, int index) {
  char b = array[index/8];
  char m = (1 << (index % 8));
  array[index/8] = (b | m);
}

// Clear bit at position index in array
void bit_clear(char *array, int index) {
  char b = array[index/8];
  char m = (1 << (index % 8));
  array[index/8] = (b & ~m);
}

```
### 初始化
首先调用mmap分配一个非常大的区域。此大小为2 ^ MAXSIZE * 16,16为分配的最小块。

```
// Allocate memory for the heap managed by the allocator, and allocate
// memory for the data structures of the allocator.
void
bd_init() {
  bd_base = mmap(NULL, HEAP_SIZE, PROT_READ | PROT_WRITE,
		 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (bd_base == MAP_FAILED) {
    fprintf(stderr, "couldn't map heap; %s\n", strerror(errno));
    assert(bd_base);
  }
  // printf("bd: heap size %d\n", HEAP_SIZE);
  for (int k = 0; k < NSIZES; k++) {
    lst_init(&bd_sizes[k].free);
    int sz = sizeof(char)*ROUNDUP(NBLK(k), 8)/8;
    bd_sizes[k].alloc = malloc(sz);
    memset(bd_sizes[k].alloc, 0, sz);
  }
  for (int k = 1; k < NSIZES; k++) {
    int sz = sizeof(char)*ROUNDUP(NBLK(k), 8)/8;
    bd_sizes[k].split = malloc(sz);
    memset(bd_sizes[k].split, 0, sz);
  }
  lst_push(&bd_sizes[MAXSIZE].free, bd_base);
}

```
### 找到第一个k， 使得2^k >= n

```
// What is the first k such that 2^k >= n?
int
firstk(size_t n) {
  int k = 0;
  size_t size = LEAF_SIZE;

  while (size < n) {
    k++;
    size *= 2;
  }
  return k;
}
```

###  查找位置p ， 如果以2^k 大小为区块，那么其位于第几个区块。

```

// Compute the block index for address p at size k
int
blk_index(int k, char *p) {
  int n = p - (char *) bd_base;
  return n / BLK_SIZE(k);
}

```

###  将k大小，序号为bi的区块的首地址计算出来
```
// Convert a block index at size k back into an address
void *addr(int k, int bi) {
  int n = bi * BLK_SIZE(k);
  return (char *) bd_base + n;
}
```


###  malloc
找到一个大小k，块2^k是大于等于要分配的大小。
如果db_size[k].free 不为空，说明当前有大小为k的空闲空间。
如果没有找到，则让k+1，继续找到更大的空间有无空闲。

如果找到，则lst_pop(&bd_sizes[k].free)获取第一个块。 blk_index(k, p)获取以k为衡量指标，p位于的第n个块。  并通过bit_set将bd_sizes[k].alloc 在第n号位置设置为1。
如果找到的k是比较大的空间，这时候需要对此空间进行分割，一分为2。 一直到找到一个比较符合的空间。

```
void *
bd_malloc(size_t nbytes)
{
  int fk, k;

  assert(bd_base != NULL);

  // Find a free block >= nbytes, starting with smallest k possible
  fk = firstk(nbytes);
  for (k = fk; k < NSIZES; k++) {
    if(!lst_empty(&bd_sizes[k].free))
      break;
  }
  if(k >= NSIZES)  // No free blocks?
    return NULL;

  // Found one; pop it and potentially split it.
  char *p = lst_pop(&bd_sizes[k].free);
  bit_set(bd_sizes[k].alloc, blk_index(k, p));
  for(; k > fk; k--) {
    // 第2半的空间
    char *q = p + BLK_SIZE(k-1);
    // 对于大小k来说，其在位置p处是分割的。
    bit_set(bd_sizes[k].split, blk_index(k, p));
    // 对于大小k-1来说，其在位置p处是分配的。
    bit_set(bd_sizes[k-1].alloc, blk_index(k-1, p));
    // 对于大小k-1来说，其在位置q处是空闲的。
    lst_push(&bd_sizes[k-1].free, q);
  }
  // printf("malloc: %p size class %d\n", p, fk);
  return p;
}
```

###  free
 当要free位置p时，size找到第一个k，当k+1在p位置是分割的，则返回k。这时候说明在k处区域是可以合并的。这是一种优化。
 ```
 // Find the size of the block that p points to.
int
size(char *p) {
  for (int k = 0; k < NSIZES; k++) {
    if(bit_isset(bd_sizes[k+1].split, blk_index(k+1, p))) {
      return k;
    }
  }
  return 0;
}
 ```


```
void
bd_free(void *p) {
  void *q;
  int k;

  for (k = size(p); k < MAXSIZE; k++) {
    int bi = blk_index(k, p);
    bit_clear(bd_sizes[k].alloc, bi);
    int buddy = (bi % 2 == 0) ? bi+1 : bi-1;
    if (bit_isset(bd_sizes[k].alloc, buddy)) {
      break;
    }
    // budy is free; merge with buddy
    q = addr(k, buddy);
    lst_remove(q);
    if(buddy % 2 == 0) {
      p = q;
    }
    bit_clear(bd_sizes[k+1].split, blk_index(k+1, p));
  }

  // 放入freelist当中。
  // printf("free %p @ %d\n", p, k);
  lst_push(&bd_sizes[k].free, p);
}

```

###  环形双链表的基本操作
```
// Implementation of lists: double-linked and circular. Double-linked
// makes remove fast. Circular simplifies code, because don't have to
// check for empty list in insert and remove.

void
lst_init(Bd_list *lst)
{
  lst->next = lst;
  lst->prev = lst;
}

int
lst_empty(Bd_list *lst) {
  return lst->next == lst;
}

void
lst_remove(Bd_list *e) {
  e->prev->next = e->next;
  e->next->prev = e->prev;
}

void*
lst_pop(Bd_list *lst) {
  assert(lst->next != lst);
  Bd_list *p = lst->next;
  lst_remove(p);
  return (void *)p;
}

void
lst_push(Bd_list *lst, void *p)
{
  Bd_list *e = (Bd_list *) p;
  e->next = lst->next;
  e->prev = lst;
  lst->next->prev = p;
  lst->next = e;
}

void
lst_print(Bd_list *lst)
{
  for (Bd_list *p = lst->next; p != lst; p = p->next) {
    printf(" %p", p);
  }
  printf("\n");
}
```


## 其他顺序分配方式
    * dlmalloc
    * slab allocator
## 其他目标

* 内存开销小
* 例如buddy的元数据很大
* 良好的内存位置
* cpu核心增加时，扩展性好
* 并发malloc / free

##  参考资料
[源码](https://en.wikipedia.org/wiki/Buddy_memory_allocation)
[讲义](https://pdos.csail.mit.edu/6.828/2019/lec/l-allocator.txt)

## 技术交流
技术交流2群：713385260