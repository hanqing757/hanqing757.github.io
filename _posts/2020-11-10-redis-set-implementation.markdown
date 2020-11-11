---
layout: post
title:  "redis之set实现"
date:   2020-11-10 15:00:00
categories: redis-set
tags: redis set intset
excerpt: redis之集合set实现
mathjax: true
---

#### set常用命令
set是无序且不重复的元素集合，命令如下
```shell
sadd key member [member]
sismember key member
spop key
srandmember key [count]
srem key member [member]
smove src dst memeber
scard key
smembers key
sinter key [key]
sinterstore dst key [key]
sunion key [key]
sunionstore dst key [key]
sdiff key [key]
sdiffstore dst key [key]
```
几点说明
1. spop是移除并返回集合中的一个随机元素，srandmember返回一个随机元素但是并不移除
2. 可以计算多个集合的交集，并集，差集，并存储到另外一个集合中

#### 数据结构（intset）
为了节省存储空间，当set的元素均为整数且均小于64bit的时候，如果元素的个数在以下配置的范围内的时候，set底层使用intset这种结构存储，否则使用dict存储。使用dict存储的时候，key就是要存储的元素，value是null。
```shell
# Sets have a special encoding in just one case: when a set is composed
# of just strings that happen to be integers in radix 10 in the range
# of 64 bit signed integers.
# The following configuration setting sets the limit in the size of the
# set in order to use this special memory saving encoding.
set-max-intset-entries 512
```
intset实际是一个有序整数数组，二分查找的平均时间复杂度在O(log N)，所以当元素个数很多的时候使用intset不适合，而使用dict进行存储的时候可以以
O(1)的时间复杂度进行查询。下面我们分析一下intset的数据结构实现。
```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;

/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. */
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
字段解释如下
1. encoding表示set中的元素使用几个字节存储。根据元素大小有三种长度编码，INTSET_ENC_INT16表示每个元素最大用2个字节存储，INTSET_ENC_INT32
表示用4个字节存储，INTSET_ENC_INT64表示用8个字节存储。每次向set中添加一个元素的时候都会判断当前set的编码长度是否满足添加元素的编码长度。
2. length表示set的长度，就是元素个数
3. content就是实际元素存储地址，是一个柔性数组，代表一个偏移量。
下图表示向一个set中添加元素的例子，

![intset](/img/redis-set/intset.png)

从图中我们可以看出
1. 初始化的时候set不包含元素，length=0，encoding=2，表示元素用2个字节的编码。
2. 在添加元素13和5之后，length的长度变为2
3. 在添加了元素32768，10，100000之后，由于2个字节表示的整数范围（-32768，32767）不满足，需要将encoding变为4个字节，length也变为5。
同时，可以看到，在set不要求元素以插入顺序存储的时候，我们可以选择插入的位置从而使元素的排列是有序的，这样可以提高查询的速度。

#### 常用操作分析
##### sadd命令的实现
set命令的实现方法主要在t_set.c这个文件中，
```c
/* Factory method to return a set that *can* hold "value". When the object has
 * an integer-encodable value, an intset will be returned. Otherwise a regular
 * hash table. */
robj *setTypeCreate(sds value) {
    if (isSdsRepresentableAsLongLong(value,NULL) == C_OK)
        return createIntsetObject();
    return createSetObject();
}

/* Add the specified value into a set.
 *
 * If the value was already member of the set, nothing is done and 0 is
 * returned, otherwise the new element is added and 1 is returned. */
int setTypeAdd(robj *subject, sds value) {
    long long llval;
    if (subject->encoding == OBJ_ENCODING_HT) {
        dict *ht = subject->ptr;
        dictEntry *de = dictAddRaw(ht,value,NULL);
        if (de) {
            dictSetKey(ht,de,sdsdup(value));
            dictSetVal(ht,de,NULL);
            return 1;
        }
    } else if (subject->encoding == OBJ_ENCODING_INTSET) {
        if (isSdsRepresentableAsLongLong(value,&llval) == C_OK) {
            uint8_t success = 0;
            subject->ptr = intsetAdd(subject->ptr,llval,&success);
            if (success) {
                /* Convert to regular set when the intset contains
                 * too many entries. */
                if (intsetLen(subject->ptr) > server.set_max_intset_entries)
                    setTypeConvert(subject,OBJ_ENCODING_HT);
                return 1;
            }
        } else {
            /* Failed to get integer from object, convert to regular set. */
            setTypeConvert(subject,OBJ_ENCODING_HT);

            /* The set *was* an intset and this value is not integer
             * encodable, so dictAdd should always work. */
            serverAssert(dictAdd(subject->ptr,sdsdup(value),NULL) == DICT_OK);
            return 1;
        }
    } else {
        serverPanic("Unknown set encoding");
    }
    return 0;
}

/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

```
对于sadd命令，主要是将元素加入到set集合中
1. 当插入的key不存在的时候，根据插入元素类型是整数还是字符串，创建不同的底层数据结构。
2. 循环插入元素。 当set的编码为hashtable的时候，插入的key就是元素，插入的value为null。
3. 当set的编码为intset的时候，首先判断当前插入元素的编码是否大于set当前设置的编码长度，不足则升级set的编码长度并插入元素，否则，查找当前元素在set中的位置。
4. 如果元素存在则直接返回，否则将插入位置之后的元素全部向后移动，并插入新的元素（保证intset的有序性）。
5. 在向intset中插入新元素之后判断元素个数是否大于set-max-intset-entries，从而决定是否要将intset转为dict存储。

我们看一下在intset中搜索元素的方法intsetSearch，
```c
/* Search for the position of "value". Return 1 when the value was found and
 * sets "pos" to the position of the value within the intset. Return 0 when
 * the value is not present in the intset and sets "pos" to the position
 * where "value" can be inserted. */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```
由于intset是有序的，那我们必须进行二分查找
1. 若当前集合set中没有元素，则直接返回。
2. 若元素比set中最小元素还小或者比最大元素还大（set的有序性只需要与端点处的元素比较），则直接返回未找到元素，并标记插入位置是在起点还是终点
3. 若当前元素在set元素的范围之内，则使用二分查找，返回查找结果，并标记元素位置（找到），或者即将插入位置（未找到）。

set的其余命令无非是建立在查找或者删除的基础之上。对于查找操作，如果是dict，则复杂度为O(1)；如果是intset，则复杂度是O(log N)。对于删除操作，在找到元素（位置）之后删除操作就很简单了。

set的交并差集计算则比较常规，对几个集合中的元素进行遍历计算即可，此处不再赘述。

#### 参考文章
1. http://zhangtielei.com/posts/blog-redis-intset.html