本实验要求实现mm_malloc、mm_free和mm_realloc三个函数。它们的语义与c标准库中malloc、free和realloc函数的语义是相同。

csapp一书己经提供了大部分的函数实现，只需要补全find_fit、replace和mm_realloc三个函数即可。

### find_fit  寻找合适的空闲块

堆中内存以块的形式进行分配。每一个块由头部、数据和脚部三个字段顺序排列而成。头部中记录了整个块的长度以及是否分配的信息，而脚部是头部的副本。从块的定义不难看出，这是个隐式链表。己知当前块的地址及长度，可以计算出下一个块的地址。

find_fit函数使用首次适配法寻找合适的空闲块。用指针遍历隐式链表，若找一块足够大的空闲块，立即返回此块。

```c
//首次适配法。遍历隐式空闲链表，找到第一个足够大的空闲就返回。
static void * find_fit(size_t asize){
    void * ptr = NEXT_BLKP(heap_listp);
    size_t bsize = GET_SIZE(HDRP(ptr));
    size_t balloc  = GET_ALLOC(HDRP(ptr));
    while(!(bsize  == 0  &&  balloc == 1)){   //遇到结束块结束遍历
        if(bsize >=  asize && balloc == 0 ){
            return ptr;
        }c
        ptr = NEXT_BLKP(ptr);        //指针移动到下一块
        bsize = GET_SIZE(HDRP(ptr));
        balloc  = GET_ALLOC(HDRP(ptr));
    }

    return NULL;
}
```

### replace 

在将一个新分配的块放置到某个空闲块之后，如何处理这个空闲块中的剩余部分？只有剩余部分的大小大于或等于最小块的大小，才会分割。

```c
static void place(void * bp, size_t asize){
      size_t bsize = GET_SIZE(HDRP(bp));
      size_t rsize = bsize - asize;
      if(rsize < 0){
        printf("Fail!,allocate size less than block size\n");
      }
      //最小块的大小为两个双字。也就是说最小块应由4字节头部、8字节数据和4字节的尾部构成。
      if(rsize < 2 * DSIZE){ 
         //不分割
         PUT(HDRP(bp), PACK(bsize, 1));
         PUT(FTRP(bp), PACK(bsize, 1));
      }else{
        //分割
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(rsize, 0));
        PUT(FTRP(bp), PACK(rsize, 0));
      }

}
```

### mm_realloc

此函数将原有内存块重分配成指定大小。例如，如果旧块是 8 个字节，新块是 12 个字节，则新块的前 8 个字节与旧块的前 8 个 2字节相同，最后 4 个字节未初始化。同样，如果旧块是 8 个字节，新块是 4 个字节，则新块的内容与旧块的前 4 个字节相同。实现代码如下：

```c
void *mm_realloc(void *ptr, size_t size)
{

    if(ptr == NULL){
        return mm_malloc(size);
    }
    if(ptr != NULL &&  size == 0){
        mm_free(ptr);
        return NULL;
    }

    void *oldptr = ptr;
    void *newptr;
    size_t copySize;
    //mm_malloc申请一个size大小的新块，将旧块内容尽可能地复制到新块，最后用mm_free释放旧块。
    newptr = mm_malloc(size);
    if (newptr == NULL)
      return NULL;
    copySize = GET_SIZE(HDRP(oldptr));
    if (size < copySize)
      copySize = size;
    memcpy(newptr, oldptr, copySize);
    mm_free(oldptr);
    return newptr;
    
}
```

这个函数是是可以进行优化的。 比如说，如果旧块的实际数据大小加上内部碎片大小大于size，那么可将旧块直接用做新块，不用去malloc和free。

在本实验中，使用“隐式空闲链表+首次适配”方法得到的分数是70分。还可以使用书中提到的分离适配法获取更好的分数。

```txt
trace  valid  util     ops      secs  Kops
 0       yes   99%    5694  0.004494  1267
 1       yes   99%    5848  0.005111  1144
 2       yes   99%    6648  0.005844  1138
 3       yes  100%    5380  0.004160  1293
 4       yes   66%   14400  0.000083172455
 5       yes   92%    4800  0.005221   919
 6       yes   92%    4800  0.004687  1024
 7       yes   55%   12000  0.090710   132
 8       yes   51%   24000  0.138216   174
 9       yes   27%   14401  0.033946   424
10       yes   34%   14401  0.001656  8698
Total          74%  112372  0.294128   382
Perf index = 44 (util) + 25 (thru) = 70/100
```



### 注:

怎么找不到wirteup中描述的11个trace文件？

官方提供的tar文件中没有包含这11个，在 [这里](https://github.com/joe00joe/csapp/tree/master/malloclab-handout/traces)可找到。



