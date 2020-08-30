---
title: PHP7内核-变量
date: 2020-08-21 16:28:40
tags: ["PHP"]
categories: ["PHP"]
---



#### 变量的内部实现
​		变量是一个语言实现的基础，变量有两个组成部分：变量名、变量值，PHP中可以将其对应为：`zval`、`zend_value`，这两个概念一定要区分开，PHP中变量的内存是通过`引用计数`进行管理的，而且**PHP7中引用计数是在`zend_value`而不是zval上，变量之间的传递、赋值通常也是针对zend_value**。

<!--more-->

PHP中可以通过`$关键词`定义一个变量：`$a;`，在定义的同时可以进行初始化：`$a = "hi~";`<u>注意这实际是两步：定义、初始化</u>，只定义一个变量也是可以的，可以不给它赋值，比如：

```php
$a;
$b = 1;
```
这段代码在执行时会分配两个zval。

#### 变量的基础结构
```c
//zend_types.h
typedef struct _zval_struct     zval;
struct _zval_struct {
    zend_value        value; //变量实际的value
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(  //这个是为了兼容大小字节序，小字节序就是下面的顺序，大字节序则下面4个顺序翻转
                zend_uchar    type,         //变量类型
                zend_uchar    type_flags,  //类型掩码，不同的类型会有不同的几种属性，内存管理会用到
                zend_uchar    const_flags,
                zend_uchar    reserved)     //call info，zend执行流程会用到
        } v;
        uint32_t type_info; //上面4个值的组合值，可以直接根据type_info取到4个对应位置的值
    } u1;
    union {
        uint32_t     var_flags;
        uint32_t     next;                 //哈希表中解决哈希冲突时用到
        uint32_t     cache_slot;           /* literal cache slot 运行时缓存会用到*/ 
        uint32_t     lineno;               /* line number (for ast nodes) */
        uint32_t     num_args;             /* arguments number for EX(This) */
        uint32_t     fe_pos;               /* foreach position foreach遍历时会用到*/
        uint32_t     fe_iter_idx;          /* foreach iterator index */
    } u2; //一些辅助值
};
```
`		zval`结构比较简单，内嵌一个union类型的`zend_value`保存具体变量类型的值或指针，zval中还有两个union：`u1`、`u2`:

- **u1**: 它是联合了一个结构体`v`和一个32位无符号整型`type_info`；ZEND_ENDIAN_LOHI_4是一个宏，用于解决字节序问题的，他会根据系统字节序决定struct v中4个成员的顺序。v定义了4个成员变量，**变量的类型就通过u1.v.type区分**；另外一个值`type_flags`为类型掩码，在变量的内存管理、gc机制中会用到；至于后面两个const_flags、reserved暂且不管。
- **u2**: 这个值纯粹是个辅助值，zval结构中value、u1分别占了8byte、4byte，一共12byte，假如zval只有:value、u1两个值，整个zval的大小也会对齐到16byte，既然不管有没有u2大小都是16byte，把多余的4byte拿出来用于一些特殊用途还是很划算的，比如next在哈希表解决哈希冲突时会用到，还有fe_pos在foreach会用到......  

```c
typedef union _zend_value {
    zend_long         lval;    //int整形
    double            dval;    //浮点型
    zend_refcounted  *counted;
    zend_string      *str;     //string字符串
    zend_array       *arr;     //array数组
    zend_object      *obj;     //object对象
    zend_resource    *res;     //resource资源类型
    zend_reference   *ref;     //引用类型，通过&$var_name定义的
    zend_ast_ref     *ast;     //下面几个都是内核使用的value
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        uint32_t w1;
        uint32_t w2;
    } ww;
} zend_value;

```
​		`zend_value`是一个联合体，各个类型根据自己的类型选择使用不同的成员，**从zend_value可以看出，除long、double类型直接存储值外，其它类型都为指针，指向各自的结构**。zend_value中没有布尔型，这是因为PHP7中将布尔型具体拆分为了true、false两种类型，通过zval.u1.v.type进行区分（注：老版本中，布尔型是通过整型进行区分的）

#### 类型
`zval.u1.type`类型：

```c
/* regular data types */
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10

/* constant expressions */
#define IS_CONSTANT                 11
#define IS_CONSTANT_AST             12

/* fake types */
#define _IS_BOOL                    13
#define IS_CALLABLE                 14

/* internal types */
#define IS_INDIRECT                 15
#define IS_PTR                      17
```

##### 标量类型
- 没有value，直接根据type区分的类型：`true`、`false`、`null`
- 值存于value中，无需额外的value指针：`zend_long`、 `double`

##### 字符串（zend_string）
​		PHP中没有使用`char`来表示字符串，而是为字符串单独定义了一个结构`zend_string`，其中除了存储字符串内容，还存储了其他信息。
```c
struct _zend_string {
    zend_refcounted_h gc; // 变量引用计数信息，用于内存管理。比如当前value的引用数，所有用到引用计数的变量类型都会有这个结构
    zend_ulong        h;  /* hash value 哈希值，数组中计算索引时会用到*/
    size_t            len; // 字符串长度，通过这个值保证二进制安全
    char              val[1]; // 字符串内容，变长struct，分配时按len长度申请内存
};
```
​		字符串内容`val`是一个可变数组，在字符串分配时的操作为`malloc(sizeof(zend_string) + 字符串长度)`。
*注：val中多出一个字节（val[1]而不是val[0]）用于存储字符串的最后一个字符"\0".*

例如：$a="abc"，对应zend_string内存结构如下：
![zend_string内存结构](https://note.youdao.com/yws/api/personal/file/39A4055CBC584591A643CF8855653427?method=download&shareKey=100018cf52486ade9a25e3f4d8227678)


字符串具体分类：
- `IS_STR_PERSISTENT`: 通过malloc分配。
- `IS_STR_INTERNED`: php代码中写的一些字面量，如函数名、变量名。
- `IS_STR_PERMERNENT`:永久值，生命周期大于request。
- `IS_STR_CONSTANT`:常量。
- `IS_STR_CONSTANT_UNQUALIFIED`:这个信息通过flag保存：zval.value->gc.u.flags

##### 数组（array）
​		`Array`是PHP中非常强大的一个数据结构，它的**底层实现为散列表（HashTable 哈希表）**。
​		散列表是根据`key`直接进行访问的数据结构，它的`key-value`之间有一个映射函数，可以根据key通过映射函数直接索引到对应的value值，直接根据`“内存起始地址+偏移值”`进行寻址，加快查找速度。理想情况下，查找的期望时间复杂度为O(1).



HashTable的数据结构如下:

```c
typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc; //引用计数信息，与字符串相同
    // 提供一些辅助的功能，比如，flag用来设置散列表的一些属性，是否持久化、是否已经初始化。
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                zend_uchar    nIteratorsCount,
                zend_uchar    reserve)
        } v;
        uint32_t flags;
    } u;
    // 用于散列函数映射存储元素在arData数组中的下标。其值实际是nTableSize的负数，即nTableMask=-nTableSize（nTableMask=~nTableSize+1）
    uint32_t          nTableMask; //计算bucket索引时的掩码
    // 存储元素数组，每个元素的结构统一为Bucket，其内存是连续的，arData指向第一个Bucket（即指向数组的起始位置）
    Bucket           *arData; //bucket数组
    // 当前已使用的Bucket数，但这些Bucket并不都是有效的，因此再删除一个数组元素时，并不会马上将其从数组中移除，而是将这个元素的类型表位IS_UNDEF，只有在数组容量超过限制，需要扩容时才会删除。
    uint32_t          nNumUsed; 
    // 数组实际存储的元素数（有效元素数）。
    uint32_t          nNumOfElements; //已有元素数，nNumOfElements <= nNumUsed，因为删除的并不是直接从arData中移除
    // 数组的总容量，其大小为2的幂次方，最小为8（即2^3）。
    uint32_t          nTableSize; //数组的大小，为2^n
    uint32_t          nInternalPointer; //数值索引
    // 下一个可用的数值索引，如arr[]=1;arr['a']=2;arr[]=3;则nNextFreeElement=2；该成员是给自动确定数值索引使用的。
    zend_long         nNextFreeElement;
    // 当删除或覆盖数组中的某个元素时，若提供了这个函数句柄，则会回调此函数。
    dtor_func_t       pDestructor;
};
```


Bucket的结构如下,主要用来保存元素的key及value。

```c
typedef struct _Bucket {
    // 存储的具体的value，这里嵌入了一个zval而不是一个指针。
	zval              val;
	// hash code，用来映射元素的存储位置。若元素是数值索引，那么他的值就是数值索引的值；若是字符串，那么这个只就是根据字符串key通过Time33算法计算得到的散列值。
	zend_ulong        h;                /* hash value (or numeric index)   */
	// 存储元素的key。
	zend_string      *key;              /* string key or NULL for numerics */
} Bucket;
```
###### 基本实现
散列表主要由两部分组成：
- 存储元素数组
- 散列函数
一个简单的散列函数可以采用取模的方式，比如散列表的大小为8，那么在散列表初始化数组时就会分配8个元素大小的空间，根据key的hash code与8取模的到的值作为该元素在数组中的下标。其示意图如下：

![散列表的基本实现](https://note.youdao.com/yws/api/personal/file/F2CB6F384907410BB02F7F7411F4E341?method=download&shareKey=fc47b2317eb268543f1440ed7296beb0)

**以散列函数的输出值作为该元素在存储元素数组中的下标的方式有一个问题: **元素在数组中的位置是随机的，它是无序的。



- **问：那么PHP是如何保证元素的顺序与其插入顺序一致？** 
  		为了实现散列表的有序性，PHP在散列函数与元素数组之间加了一层映射表，该映射表也是一个数组，大小与存储元素的数组相同，它存储的元素类型为整型，用于保存实际存储的有序数组中的下标：**元素按照先后顺序依次插入实际存储的数组，然后将其数组下标按照散列函数散列出来的位置存储在新加的映射表中**，如下图所示。

![散列表映射关系](https://note.youdao.com/yws/api/personal/file/A8DBDBE3688A48739E06BF38ACDA300F?method=download&shareKey=6e4969019969485debc917c11a87da4d)

原理如上，但实际上PHP是将这个映射表与arData放在一起，在数组初始化时会分配存储Bucket的内存，同时还会分配相同数量的uint32_t大小的空间，将arData偏移到存储元素数组的位置，这个中间映射表可以通过arData向前访问到。如下图所示：
![HashTable中间映射表](https://note.youdao.com/yws/api/personal/file/358B60D6F6AA407E9B034BACE5E8ACB6?method=download&shareKey=e515e9d30e617ea6c6fa8b9475a9efeb)

###### 散列函数
​		通常散列会数会以取模的方式给出，比如：`key->h%nTableSize`.但是PHP采用了另一种方式，因为散列表的大小为2的幂次方，所以通过**或运算**可以得到`[-1,nTableMask]`之间的散列值。
```
nIndex = h | ht->nTableMask
```


eg：

```
h=18003212
nTableSize=8

nTableMask=-8
nIndex=-4
```



###### 数组的初始化

​		数组初始化的过程主要是对`HashTable`中的成员进行设置，初始化时并不会立即分配`arData`的内存，`arData`的内存在**插入第一个元素时才会分配**。
```c
ZEND_API void ZEND_FASTCALL _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
    // 初始化gc信息
	GC_REFCOUNT(ht) = 1;
	GC_TYPE_INFO(ht) = IS_ARRAY;
	// 设置flags
	ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION | HASH_FLAG_STATIC_KEYS;
	// nTableMask的值是临时的
	ht->nTableMask = HT_MIN_MASK;
	// 临时设置ht->arData
	HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
	ht->nNumUsed = 0;
	ht->nNumOfElements = 0;
	ht->nInternalPointer = HT_INVALID_IDX;
	ht->nNextFreeElement = 0;
	ht->pDestructor = pDestructor;
	// 把数组的大小重置为2的幂次方
	ht->nTableSize = zend_hash_check_size(nSize);
}
```
###### 插入
​		插入时，会检查数组是否已经分配存储空间。PHP会在第一次插入时根据`nTableSize`的大小分配，分配完成后把`HashTable->u.flags`打上`HASH_FLAG_INITIALIZAED`掩码。

- 分配内存  
分配的内存包括映射表及元素数组：
```
nTableSize * (sizeof(Bucket) + sizeof(uint32_t))
```
分配完成后，将`HashTable->arData`指向第一个`Bucket`的位置。

- 插入数据  
将元素按照顺序插入`arData`，然后将其在`arData`数组中的位置存储到根据`key`的`hash code`（即`key->h`）与`nTableMask`计算得到的中间映射表中的对应位置。
```c
zend_hash.c
// _zend_hash_add_or_update_i:
add_to_hash:
	HANDLE_BLOCK_INTERRUPTIONS();
	// idx为Bucket在arData中存储位置
	idx = ht->nNumUsed++;
	ht->nNumOfElements++;
	if (ht->nInternalPointer == HT_INVALID_IDX) {
		ht->nInternalPointer = idx;
	}
	zend_hash_iterators_update(ht, HT_INVALID_IDX, idx);
	if ((zend_long)h >= (zend_long)ht->nNextFreeElement) {
		ht->nNextFreeElement = h < ZEND_LONG_MAX ? h + 1 : ZEND_LONG_MAX;
	}
	// 找到存储Bucket，设置key、value
	p = ht->arData + idx;
	p->h = h;
	p->key = NULL;
	// 计算中间映射表的散列值，idx将保存在映射数组的nIndex位置
	nIndex = h | ht->nTableMask;
	// 将映射表中原来的值保存到新Bucket中，哈希冲突时会用到
	ZVAL_COPY_VALUE(&p->val, pData);
	// 先把旧的值保存到新插入的元素中
	Z_NEXT(p->val) = HT_HASH(ht, nIndex);
	// 再把新元素数组存储位置更新到映射表中
	HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(idx);
	HANDLE_UNBLOCK_INTERRUPTIONS();

	return &p->val;
```
###### 哈希冲突
​		散列表中不同元素的`key`可能计算得到相同的哈希值，这些具有相同哈希值的元素在插入散列表时就会发生冲突，因为映射表只能存储一个元素。  
**常见的解决方式（PHP采用这种方式）：将冲突的Bucket串成链表，查找时需要遍历这个链表，逐个比较`key`，从而找到目标元素。**

- 具体操作：
  		`HashTable`中的`Bucket`会记录与它冲突的元素在`arData`数组中的存储位置。在设置映射值时，如果发现映射表中要设置的位置已经被之前插入的元素占用了（值不等于初始化的-1），那么会把已经存在的值保存到新插入的`Bucket`中，然后将映射表中的值更新为新`Bucket`的存储位置（即每次都会把冲突的元素插到开头）。  
  **冲突元素的保存位置为：**`Bucket.val.u2.next`



**示例**：一个数组有三个元素，按照a、b、c的顺序插入，加入a、c两个key冲突了，则HashTable的结构如下：
$arr = [];
$arr['a'] = 11;
$arr['b'] = 22;
$arr['c'] = 33;
![哈希冲突链表](https://note.youdao.com/yws/api/personal/file/C469391F27FF454698CCD908B98FB2B2?method=download&shareKey=9501d4a7b3029c279f3d545c7f9c18e2)

###### 查找
查找过程如下：
- ①、根据`key`计算出`hash code`（即`zend_string->h`）与`nTableMask`计算得到散列值`nIndex`。
- ②、根据散列值从中间映射表中得到存储元素在有序存储数组中的位置`idx`。
- ③、根据`idx`从有序存储数组（`HashTable->arData`）中取出`Bucket`
- ④、从取出的`Bucket`进行遍历，判断Bucket的key是否是要查找的key，若是则停止遍历，否则继续根据`zval.u2.next`遍历比较。

```c
// zend_hash_find_bucket:
// 根据zend_string *key进行查找
static zend_always_inline Bucket *zend_hash_find_bucket(const HashTable *ht, zend_string *key)
{
	zend_ulong h;
	uint32_t nIndex;
	uint32_t idx;
	Bucket *p, *arData;

	h = zend_string_hash_val(key);
	arData = ht->arData;
	// 计算散列值
	nIndex = h | ht->nTableMask;
	// 获取Bucket存储位置
	idx = HT_HASH_EX(arData, nIndex);
    // 遍历
	while (EXPECTED(idx != HT_INVALID_IDX)) {
		p = HT_HASH_TO_BUCKET_EX(arData, idx);
		if (EXPECTED(p->key == key)) { /* check for the same interned string */
			return p;
		} else if (EXPECTED(p->h == h) && // 先比较hash code
		     EXPECTED(p->key) && 
		     // 在比较key长度，最后按字符比较是否相同
		     EXPECTED(ZSTR_LEN(p->key) == ZSTR_LEN(key)) &&
		     EXPECTED(memcmp(ZSTR_VAL(p->key), ZSTR_VAL(key), ZSTR_LEN(key)) == 0)) {// 比较查找的key与Bucket的key是否匹配
			return p;
		}
		// 不匹配则继续遍历
		idx = Z_NEXT(p->val);
	}
	return NULL;
}
```


###### 扩容

​		数组的容量是有限的，最多可以存储`nTableSize`个元素，那么当数组空间已满还要继续插入时如何处理？  



**问: PHP是怎样实现的自动扩容？**

​		**扩容的过程为**：检查数组中已经删除的元素所占的比例（已经删除但未从存储数组中移除的元素）.若比例达到域值，则触发**重建索引**的操作，这个过程会把删除的Bucket移除，然后把后面的Bucket往前移补上空缺的Bucket；若还没有达到域值，则分配一个原数组大小2倍的新数组，然后把原数组的元素复制到新数组上，重建索引。 


<u>域值判断公式</u>如下，即域值为`nNumOfElement + (nNumElement / 32)`

```
ht->nNumUsed > ht->nNumOfElement + (ht->nNumOfElement >> 5)
```

具体的处理过程：
```c
static void ZEND_FASTCALL zend_hash_do_resize(HashTable *ht)
{

	IS_CONSISTENT(ht);
	HT_ASSERT(GC_REFCOUNT(ht) == 1);

	if (ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)) { // 无序扩容，将删除的Bucket移除，然后把后面的bucket往前补上空缺
		HANDLE_BLOCK_INTERRUPTIONS();
		// 只有到达一定域值才进行rehash操作
		zend_hash_rehash(ht); // 重建索引数组
		HANDLE_UNBLOCK_INTERRUPTIONS();
	} else if (ht->nTableSize < HT_MAX_SIZE) { // 扩容，分配原数组大小2倍的新数组。
		void *new_data, *old_data = HT_GET_DATA_ADDR(ht);
		// 扩大为2倍，加法比乘法快
		uint32_t nSize = ht->nTableSize + ht->nTableSize;
		Bucket *old_buckets = ht->arData;

		HANDLE_BLOCK_INTERRUPTIONS();
		// 新分配arData空间，大小为(sizeof(Bucket) + sizeof(uint32_t)) * nSize;
		new_data = pemalloc(HT_SIZE_EX(nSize, -nSize), ht->u.flags & HASH_FLAG_PERSISTENT);
		ht->nTableSize = nSize;
		ht->nTableMask = -ht->nTableSize;
	    // 将arData指针偏移到Bucket数组起始位置
		HT_SET_DATA_ADDR(ht, new_data);
		// 将旧的Bucket数组复制到新空间（此步只复制存储的元素，即HashTable->arData，不会复制中间映射表，因为扩容后旧的映射表已无法使用，key-value的映射关系需要重新计算，即重建索引）
		memcpy(ht->arData, old_buckets, sizeof(Bucket) * ht->nNumUsed);
		// 释放旧空间
		pefree(old_data, ht->u.flags & HASH_FLAG_PERSISTENT);
		// 重建索引数组：映射表
		zend_hash_rehash(ht);
		HANDLE_UNBLOCK_INTERRUPTIONS();
	} else {
		zend_error_noreturn(E_ERROR, "Possible integer overflow in memory allocation (%zu * %zu + %zu)", ht->nTableSize * 2, sizeof(Bucket) + sizeof(uint32_t), sizeof(Bucket));
	}
}
```
重建索引的过程实际上就是将所有元素重新插入一遍，其处理过程如下：
```
// 遍历数组，重新设置中间映射表（索引表）
    do {
			nIndex = p->h | ht->nTableMask;
			Z_NEXT(p->val) = HT_HASH(ht, nIndex);
			HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(i);
			p++;
		} while (++i < ht->nNumUsed);
```
重建索引会将已删除的bucket移除，移除后会把这个Bucket之后的元素全部向前移动一个位置，所以**重建索引后存储数组中元素全部紧密排列在一起**。



##### 引用

​		引用类型是PHP中比较特殊的一种类型，它实际是指向另外一个PHP变量（*在PHP中通过`&操作符`产生一个引用变量*），对它的修改会直接改动实际指向的zval，<u>可以简单的理解为C中的指针</u>。  

操作步骤：

- 首先为`&`操作的变量分配一个`zend_reference结构`，其内嵌一个`zval`，这个`zval`的`value`指向原来`zval`的`value`(**注: 如果是布尔、整形、浮点则直接复制原来的值**)。
- 然后将原`zval`的类型修改为`IS_REFERENCE`，原`zval`的`value`指向新创建的`zend_reference`结构。
```
struct _zend_reference {
    zend_refcounted_h gc;
    zval              val; // 指向原来的value
};
```
示例1：
```
$a = date('Y-m-d');
$b = &$a;
```
![a与b内存引用关系](https://note.youdao.com/yws/api/personal/file/A5F4C7219117457A88A7D7CC3AFB53A9?method=download&shareKey=3d6950341fd9f21fea74f78f022500fd)

**注：若此时将`$b`复制给其他变量，那么传递给新变量的value将实时及引用的值，而不是引用本身**。PHP中的引用只有一级，不会出现一个引用指向另外一个引用的情况，即没有C语言中多级指针的概念。

```
$a = date('Y-m-d');
$b = &$a;
$c = $b; // 若想让$c也指向$a/$b引用的值，则：$c = &$b或$c = &$a;
```
![a,b与c内存引用关系](https://note.youdao.com/yws/api/personal/file/1E8C8DDDE51D479296CA8F63EA5353B7?method=download&shareKey=43183c8018d3140f9b924508528c2af2)


示例2：
```
$a = "time:" . time();      //$a    -> zend_string_1(refcount=1)
$b = &$a;                   //$a,$b -> zend_reference_1(refcount=2) -> zend_string_1(refcount=1)
```
![zend_ref](https://note.youdao.com/yws/api/personal/file/697F728B851442D4AF142554DDF78333?method=download&shareKey=59d725805b5b7c8bc59c391e87607101)

注意：**引用只能通过`&`产生，无法通过赋值传递**  

例：

```
error:
$a = "time:" . time();      //$a    -> zend_string_1(refcount=1)
$b = &$a;                   //$a,$b -> zend_reference_1(refcount=2) -> zend_string_1(refcount=1)
$c = $b;                    //$a,$b -> zend_reference_1(refcount=2) -> zend_string_1(refcount=2)
                            //$c    -> 
right:
$a = "time:" . time();      //$a       -> zend_string_1(refcount=1)
$b = &$a;                   //$a,$b    -> zend_reference_1(refcount=2) -> zend_string_1(refcount=1)
$c = &$b;/*或$c = &$a*/     //$a,$b,$c -> zend_reference_1(refcount=3) -> zend_string_1(refcount=1)                             
```
这个也表示PHP中的 **引用只可能有一层 ，不会出现一个引用指向另外一个引用的情况** ，也就是没有C语言中指针的指针的概念。



##### 对象/资源
对象比较常见，资源指的是tcp连接、文件句柄等等类型。
```
struct _zend_object {
    zend_refcounted_h gc;
    uint32_t          handle;
    zend_class_entry *ce; //对象对应的class类
    const zend_object_handlers *handlers;
    HashTable        *properties; //对象属性哈希表
    zval              properties_table[1];
};

struct _zend_resource {
    zend_refcounted_h gc;
    int               handle;
    int               type;
    void             *ptr;
};
```

##### 类型转换
​		PHP是弱类型语言，使用时不需要明确定义变量的类型，Zend虚拟机在执行PHP代码时，会根据具体的应用场景进行转换，也就是变量会按照类型转换规则将不合格变量转换给合格的变量，然后进行操作。

例:
```
$a = "100" + 200
```
执行时Zend发现相加的一个值为字符串，就会试图将`字符串100`转为数值类型（整型或浮点型），然后与200相加。  
**注：转换的时候并不会改变原来的值，而是会生成一个新的变量进行处理。**



###### 强制转换

PHP提供了一种强制转换方式：
- (int)/(integer): 转换为整型integer
- (bool)/(boolean):转换为布尔类型boolean
- (flaot)/(double)/(real):转换为浮点型flaot
- (string):转换为字符串string
- (array):转换为数组array
- (object):转换为对象object
- (unset):转换为null

*注：有些类型之间是无法转换的，如：资源类型，无法将任何类型转换为资源类型。*

###### 转换为null
​		任意类型都可以转为null，转换时直接将新的`zval类型`设置为`IS_NULL`。



###### 转换为布尔型

​		当转换为布尔型时，根据原值的`true`、`false`决定转换后的结果，一些值被认为是`false`，除此之外的其他值通常被认为是`true`。

被认为是false的值:

- 布尔值false本身
- 整型0
- 浮点型值0.0
- ==空字符串（‘’），以及字符串‘0’==
- 空数组
- null



###### 转换为整型

从`其他值`转换为`整型`的规则如下：
- null：转换为0
- 布尔型：false转为0，true转为1
- 浮点型：向下取整，比如，(int)2.8 = 2
- 字符串：与C语言strtoll()的规则一致
    - 字符串以合法数值(包含正负数)开始，就使用该数值
    - **否则，其值为0**
- 数组：很多操作不支持将一个数组自动转为整型处理，比如array()+2将报error错误，但可以强制把数组转为整型：
    - 非空数组：1
    - 空数组：0
```php
php > $a = array()+2;
PHP Fatal error:  Unsupported operand types in php shell code on line 1
PHP Stack trace:
PHP   1. {main}() php shell code:0
PHP   2. {main}() php shell code:0
php > 
php > $a = array();
php > $b = (int)$a;
php > echo $b;
0
```
- 对象：与数组类似，很多操作也不支持将兑现个自动转为整型，但有些操作只会抛一个warning警告，还是会把对象转换为1.
- 资源：转为分配给这个资源的唯一编号



###### 转为浮点型

​		除了字符串类型外，其他类型转换规则与整型基本一致，只是在整型转换结果上加了小数位，字符串转为浮点数有`zend_strtod`完成。



###### 转换为字符串

- 强制转换：
    - (string)
    - strval()函数
- 自动转换：
    - 需要字符串的表达式中，比如：函数echo或print时
    - 非string类型变量与一个string变量进行比较时
        - null/fasle:转为空字符串
        - true：转为“1”
        - 整型：原样转为字符串，**转换时将各位一次除10取余**
        - 浮点型：原样转为字符串
        - 资源：转为“Resource id#xxx”
        - 数组：转为“Array”，同时报Notice
        - 对象：不能转换，将报错,如下：
        ```php
        php > class A 
        php > {public $b;}
        php > 
        php > $a = new A();
        php > 
        php > echo 'a= ' . $a;
        PHP Catchable fatal error:  Object of class A could not be converted to string in php shell code on line 1
        PHP Stack trace:
        PHP   1. {main}() php shell code:0
        ```
###### 转换为数组
- 若变量类型为`null`、`integer`、`float`、`string`、`boolean`和`resource`中的一个：将得到一个仅有一个元素的数组，其`下标为0`，即(array)$scalarValue与`array($scalarValue)`完全一样。
- 若变量类型为object：其结果为一个数组，数组的元素为该对象的全部属性（包含public、private、protected），但他们也是有区别的，如下：
    - public的属性：key
    - private的属性：key加类型作为前缀
    - protected的属性：'*'加key作为前缀
    ```
    class test
    {
        public $a = 123;
        private $b = 'bbb';
        protected $c = 'ccc';
    }

    $test = new test();
    print_r((array)$test);
    ```
    以上例子将输出：
    ```
    $php stat.php 
    Array
    (
        [a] => 123
        [testb] => bbb
        [*c] => ccc
    )
    ```

###### 转换为对象
其他任何类型的值被转换为对象，将会创建一个内置类stdClass的实例：
- 若该值为null：新的实例为空
- array：转换成的object将以键名成为属性名，并具有相对应的值
    - 数值索引的元素也将转为属性，但无法通过“->”访问，只能遍历获取
    - 非数值索引：会以‘scalar’作为属性名