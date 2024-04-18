### PartA cache模拟

此部分实验要求模拟一个cache。此cache由多个group组， 每个组由若干cache_line组成。具体的数据结构如下：

```c
typedef struct cache_line {
  int vaild;   //有效位
  int tag;     //标记
  long time;   //时间戳，用于缓冲行淘汰
  void * block;
} cache_line_t;

typedef struct  cache_group{
   cache_line_t * lines;
} cache_group_t;

typedef struct  cache{
   long s;   //标记位数
   long E;   //组中行数
   long b;   //偏移位数
   cache_group_t * groups;
} cache_t;
```

对cache的操作与csapp的描述一样的，唯一的区别是并没有实际的内存操作。比如说，当缓冲不命中时，不需要从内存载入block到缓冲行，只是简单地设置缓冲行的有效位、标记和时间戳。cache核心代码如下 ：

```c
cache_line_t * line = find(tag, &cache->groups[g_idx], cache->E);
        if(line != NULL){     //命中
            setResult(q->type, q->address, q->rw_size, HIT);  
        }else{
            line = find_empty_line(&cache->groups[g_idx], cache->E);
            if(line != NULL){  //未命中但组中有空白缓冲行
                line->vaild = 1;
                line->tag =tag;
                setResult(q->type, q->address, q->rw_size, MISS);  

            }else{            //未命中且没有空白缓冲行，需淘汰一个缓冲行
                line = find_evict_line(&cache->groups[g_idx], cache->E);  //使用时间戳版本的LRU算法选择要淘汰的行
                line->vaild  = 1;
                line->tag = tag;
                setResult(q->type, q->address, q->rw_size, MISS + EVICT);  
            }

        }
        line->time = ++timer;  //访问过的缓冲行，更新时间戳
```

当需要载入block时但组中没有空白缓冲行时，需要使用LRU算法淘汰一个缓存行。本LRU实现是简单暴力的。每一个缓冲行都设置一个时间戳，每一次访问时更新时间戳。当需要淘汰时，遍历group中所有缓冲行，选择时间戳最小的行。LRU实现代码如下：

``` c
cache_line_t * find_evict_line(cache_group_t * group, long lines){
    long min_time = LONG_MAX;
    cache_line_t * evict_line = NULL;
    for(long i = 0; i < lines; i++){
        long time = group->lines[i].time;
        if(time < min_time){
            min_time = time;
            evict_line = &group->lines[i];
        }
    }
    return  evict_line;
}
```

### PartB 矩阵转置函数转置优化

有空再做