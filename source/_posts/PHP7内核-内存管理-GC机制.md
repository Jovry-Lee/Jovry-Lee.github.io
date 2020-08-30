---
title: PHP7内核-内存管理-GC机制
date: 2020-08-25 11:20:09
tags: ["PHP"]
categories: ["PHP"]
---

##### 1 简介

C/C++语言中，如果想在`堆`上分配变量，需要手动进行内存的分配与释放，变量的内存管理是见非常繁琐的事，稍有不慎就可能导致不可域值的错误。 PHP实现了自动GC机制，由语言自行管理。PHP中的变量是不需要手动释放的，内核帮我们实现了变量的内存管理，包括内存的分配与回收。

<!--more-->

**自动GC最简单的实现方式**：在函数中定义变量时分配一块内存，用于保存zval及对应的value结构，在函数返回时再将内存释放，若函数执行期间改变来凝固作为参数调用了其他函数或复制给了其他变量，则把变量复制一份，变量之间相互独立，不会出现冲突。
（<u>问题：深拷贝可能造成内存浪费，比如定义一个变量赋值给另一个变量，后面是只读操作的情况</u>）



**解决方式（PHP采用此种方式）:** `引用计数`+`写时复制`。

- 当变量赋值、传递时多个变量公用一个value，引用计数用来记录value有多少个变量在使用；
- 当变量的value发生改变时，进行深拷贝。

注：long、double类型是使用的硬拷贝。



##### 2 引用计数

引用计数用来记录当前有多少`zval`指向同一个`zend_value`.
引用计数：<u>指在value中增加一个字段refcount记录指向当前value的数量，变量复制、函数传参时并不直接硬拷贝一份value数据，而是将refcount++，变量销毁时将refcount--，等到refcount减为0时表示已经没有变量引用这个value，将它销毁即可。</u>

**PHP7中将变量的引用计数保存在zend_value中**（与之前版本不同）

之前变量中不同类型的结构体中都有一个相同的成员：gc，该结构用于保存引用计数的,其定义如下：

```c
// zend_type.h
typedef struct _zend_refcounted_h {
    // 引用计数
	uint32_t         refcount;			/* reference counter 32-bit */
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
			    // 类型
				zend_uchar    type,
				zend_uchar    flags,    /* used for strings & objects */
				// 垃圾回收时用到
				uint16_t      gc_info)  /* keeps GC root number (or 0) and color */
		} v;
		uint32_t type_info;
	} u;
} zend_refcounted_h;
```

例1：

```php
$a = "time:" . time();   //$a       ->  zend_string_1(refcount=1)
$b = $a;                 //$a,$b    ->  zend_string_1(refcount=2)
$c = $b;                 //$a,$b,$c ->  zend_string_1(refcount=3)

unset($b);               //$b = IS_UNDEF  $a,$c ->  zend_string_1(refcount=2)
```

例2：

```php
$a = array();   // $a       -> zend_array(refcount=1)
$b = $a;        // $a,$b    ->zend_array(refcount=2)
$c = $b;        // $a,$b,$c -> zend_array(refcount=3)
unset($b);      // $a,$c    -> zend_array(refcount=2)  $b = IS_UNDEF
```



**问: 是所有变量类型会用到引用计数吗?如果不是,那么那些情况不会用到呢?**

不是, 其中不会用到引用计数的情况如下:

- ①、没有具体value结构的类型是不会用到的，比如：整型、浮点型、布尔型、NULL，他们的值直接通过zval保存，赋值时采用<u>深拷贝</u>。
- ②、interned string：内部字符串，在PHP中写的函数名、类名、变量名、静态字符串等都是这种类型，定义:$a = "hi~";后面的字符串内容是唯一不变的，这些字符串等同于C语言中定义在静态变量区的字符串：char *a = "hi~";，这些字符串的生命周期为request期间，request完成后会统一销毁释放，自然也就无需在运行期间通过引用计数管理内存。
  （注：*内部字符串与普通字符串的类型都是IS_STRING，它们并不是通过type进行区分的，而是通过zend_refcount_h.u.v.flag区分，内部字符串的flag值将包含IS_STR_INTERNED*）
- ③、immutable array：不可变数组，只有在用opcache的时候才会用到这种类型，不清楚具体实现，暂时忽略。

除了以上几种情况, 其余类型将会用到引用计数。



**问: PHP内核是怎样区分value是否支持引用计数的呢?**

使用`zval.u1`中的类型掩码`type_flag`字段， 这个字段除了标识value是否支持引用计数外还有其它几个标识位，按位分割
*注：type_flag与zval.value->gc.u.flag不是一个值。*

支持引用计数的value类型是`zval.u1.type_flag & IS_TYPE_REFCOUNTED`。

```c
// IS_TYPE_REFCOUNTED = 4
#define IS_TYPE_REFCOUNTED          (1<<2)
```

下列类型会使用引用计数机制：

```
|     type       | refcounted |
+----------------+------------+
|simple types    |            |
|string          |      Y     |
|interned string |            |
|array           |      Y     |
|immutable array |            |
|object          |      Y     |
|resource        |      Y     |
|reference       |      Y     |
```



##### 3 写时复制

写时复制机制，它只有在必要的时候（即发生写的时候）才会进行深拷贝，可以很好地提升效率。

具体过程：<u>变量使用了引用计数，当出现其中一个变量修改value的情况，这个时候就需要对value进行分离，发生改变的变量恢复至一份数据出来进行修改，同时断开原来value的指向，指向新的value。</u>

例1：

```php
$a = array(1,2);
$b = &$a;
$c = $a;

//发生分离
$b[] = 3;
```

![zval_sep](https://note.youdao.com/yws/api/personal/file/F0B25584D691443AB81AB2C8EA60C941?method=download&shareKey=98bf888bb661490664329007cb8b939c&ynotemdtimestamp=1598325595262)

例2：

```php
$a = array(1,2);
$b = &$a;
$c = $a;

//发生分离
$c[] = 3;
```

![a、b、c发生写时复制的过程](https://note.youdao.com/yws/api/personal/file/4CDEAFB9332D4EEE9443D527A09934BF?method=download&shareKey=26b4c24edc932c27e427583bc422d2e3&ynotemdtimestamp=1598325595262)

注：不是所有类型都可以copy的，比如<u>对象、资源就无法进行复制，也就是无法进行分离，如果多个变量指向同一个对象，当其中一个变量修改对象时，其修改将反映到所有变量上</u>。事实上**只有string、array两种支持value的分离**，与引用计数相同，也是通过`zval.u1.type_flag`标识value是否可复制的。

```c
// IS_TYPE_COPYABLE = 16
#define IS_TYPE_COPYABLE         (1<<4)
|     type       |  copyable  |
+----------------+------------+
|simple types    |            |
|string          |      Y     |
|interned string |            |
|array           |      Y     |
|immutable array |            |
|object          |            |
|resource        |            |
|reference       |            |
```



##### 4 变量回收

PHP变量回收方式：

- 主动销毁：指的就是 unset。
- 自动销毁：指PHP的自动管理机制（GC机制).
  - 在return时减掉局部变量的refcount，即使没有显式的return，PHP也会自动给加上这个操作;
  - 写时复制时也会断开原来value的指向，这时候也会检查断开后旧value的refcount。



变量回收时机： zval断开value的指向时，若发现refcount=0则会直接释放value。其中发生断开的情况包括：

- 修改变量：修改变量时会断开原有value的指向
- 函数返回：函数返回时会释放所有的局部变量，也就是把所有局部变量的引用计数减1.



##### 5 垃圾回收

PHP用过引用计数实现了变量的自动GC机制，还是会有一种情况GC机制无法解决，从而导致变量无法回首导致内存始终得不到释放，造成内存泄露。

造成内存泄漏的情况：<u>循环引用</u>

**问: 什么是循环引用?**
循环引用就是变量的内部成员引用了变量自身，比如数组中某个元素指向了数组，这样一来数组的引用计数中就有一个来自自身成员，当所有外部引用断开时，数组的refcount仍然大于0而得不到释放，但实际上这种变量不可能再被使用了。

**垃圾收集器收集的时机**：<u>refcount减少时，即每次refcount减少都会试图收集</u>



示例：

```php
$a = array(1);
$a[] = &$a;
unset($a);
```

unset($a)前，变量a的类型为引用，该引用的refcount=2，一个来自于$a,一个来自于$a[1];示意图如下： ![unset($a)前的引用关系](https://note.youdao.com/yws/api/personal/file/8C22AE1A04B249948E9ADECA4F6D9865?method=download&shareKey=90e64014943ead9d4294b592037d0484&ynotemdtimestamp=1598325595262)

unset($a)后，减少了一次该引用的refcount，此时已经没有任何外部引用了，但是数组中仍然有一个元素指向该引用，如下图所示： ![unset($a)后的引用关系](https://note.youdao.com/yws/api/personal/file/74E94E6D9F384C0A9E9F44FF5F086CC9?method=download&shareKey=1ef849b379e45a8729679844f572119d&ynotemdtimestamp=1598325595262)

这种因为循环引用而导致无法释放的变量称之为垃圾，PHP引入了另一种机制来对这些垃圾进行回收，即**垃圾回收器**。

- 若一个变量value的refcount减少到0，那么此vlaue可以被释放掉，不属于垃圾。
- 若一个变量value的refcount减少之后大于0，那么此vlaue还不能被释放，此value<u>可能</u>成为一个垃圾。



垃圾回收器会将可能成为垃圾的value收集起来，等到达一定数量后开始启动垃圾鉴定程序，将真正的垃圾释放掉。

**当前垃圾只会出现在array、object两种类型中**

- array：数组中的某个成员指向了数组
- object：对象的成员属性引用对象本身



**问：垃圾回收器怎样判断当前类型是否可回收？** 

垃圾回收器不是通过变量类型进行判断的，而是通过`zval.u1.type_flag`标识进行判断，只有包含`IS_TYPE_COLLECTABLE`标识的变量类型才会被收集。

```c
// IS_TYPE_COLLECTABLE = 8
# define IS_TYPE_COLLECTABLE  (1<<3)
```

###### 5.1 回收算法

垃圾回收器把收集到的可能垃圾保存到一个buffer缓冲区中，当到达一定数量后就会启动垃圾鉴定、回收程序。
回收算法的原理：<u>既然垃圾是由于成员引用自身导致的，那么就对value的所有成员减一遍引用计数，如果发现value本身refcount变为了0，则表明引用全部来自自身成员。</u>

###### 5.2 具体实现

垃圾回收器主要通过`zend_gc_globals`这个这结构对垃圾进行管理，收集到的可能成为垃圾的vlaue就保存在这个结构的buf中，即`垃圾缓冲区`。

```c
// zend_gc.h
typedef struct _zend_gc_globals {
    // 是否启用gc（php.ini中zend.enable_gc设置是否开启，默认是开启的）
	zend_bool         gc_enabled;
	// 是否在垃圾检查过程中
	zend_bool         gc_active;
	// 缓存区是否已满
	zend_bool         gc_full;
    // 启动时分配的用于保存可能垃圾的缓存区(初始化时，一次性分配了10001个gc_root_buffer,其中第一个被保留)
	gc_root_buffer   *buf;				/* preallocated arrays of buffers   */
	// 指向buf中最新加入的一个可能垃圾（root是一个双向链表的头部）
	gc_root_buffer    roots;			/* list of possible roots of cycles */
	// 指向bug中没有使用的buffer（用于管理buf中开始加入后面又删除的结点，是一个单链表）
	gc_root_buffer   *unused;			/* list of unused buffers           */
	// 指向buf中第一个没有使用的buffer
	gc_root_buffer   *first_unused;		/* pointer to first unused buffer   */
	// 指向buf尾部
	gc_root_buffer   *last_unused;		/* pointer to last unused buffer    */
    // 待释放的垃圾
	gc_root_buffer    to_free;			/* list to free                     */
	gc_root_buffer   *next_to_free;
    // 统计gc运行次数
	uint32_t gc_runs;
	// 统计已回收的垃圾数
	uint32_t collected;

#if GC_BENCH
	uint32_t root_buf_length;
	uint32_t root_buf_peak;
	uint32_t zval_possible_root;
	uint32_t zval_buffered;
	uint32_t zval_remove_from_buffer;
	uint32_t zval_marked_grey;
#endif

	gc_additional_buffer *additional_buffer;

} zend_gc_globals;
```

- buf用于保存收集到的value，他是一个数组，在垃圾回收器初始化时一次性分配了10001个gc_root_buffer，其中第一个buffer被保留，插入vlaue时直接取出可用结点即可。
- roots指向buf中最新加入的一个结点，root是一个双向链表的头部，之所以是一个双向链表，是因为bug数组中保存的只是有可能成为垃圾的vlaue吗，其中有些value在加入之后又被删除了，这样bug数组中就会出现一些空隙。
- first_unused一开始指向bug的第一个位置，有些元素插入roots时如果first_unused还没有到达buf尾部，则返回first_unused给最新的元素，然后执行first_unused++,直到last_unused.

下图为已经加入了2个gc的结构： ![buf缓冲区的可用节点](https://note.youdao.com/yws/api/personal/file/0A2236A0CF6442A0BB985383709DB5B7?method=download&shareKey=6656570e92c3731c6f1ce8572522a2d0&ynotemdtimestamp=1598325595262)

- unused成员，他的含义与first_unused类似，用来管理buf中开始加入后面又删除的结点，是一个单链表结构。也就是说first_unused是一直往后偏移的，直到buf的结尾，buf中间由于value删除而重新空闲的结点则由unused串起来。下次有新的value插入roots时优先使用unused的这些节点，其次才是first_unused的结点。

下图为移除了buf[1]的结构： ![buf[1]移除缓存去后的可用结点](https://note.youdao.com/yws/api/personal/file/B788660EE8EE43DDB952F0DC408264B7?method=download&shareKey=3896ffff3a8a0ab653a5c4fd197426da&ynotemdtimestamp=1598325595262)

- 第一步，初始化 gc_init()初始化垃圾回收器：分配buf数组内存、设置first_unused/unused等

```c
ZEND_API void gc_init(void)
{
	if (GC_G(buf) == NULL && GC_G(gc_enabled)) {
	    // 分配buf缓冲区内存，大小为GC_ROOT_BUFFER_MAX_ENTERIES(10001),其中第1个保留不被使用。
		GC_G(buf) = (gc_root_buffer*) malloc(sizeof(gc_root_buffer) * GC_ROOT_BUFFER_MAX_ENTRIES);
		GC_G(last_unused) = &GC_G(buf)[GC_ROOT_BUFFER_MAX_ENTRIES];
		// 进行GC_G的初始化，其中：GC_G(first_unused) = GC_G(buf) + 1;从第二个开始，保留第一个。
		gc_reset();
	}
}
```

- 第二步，垃圾回收，在Zend执行过程中如果销毁一个变量就会判断是否需要加入垃圾收集器。销毁一个zval会调用i_zval_ptr_dtor()进行处理。

```c
// file:zend_variables.h
static zend_always_inline void i_zval_ptr_dtor(zval *zval_ptr ZEND_FILE_LINE_DC)
{
    // 不使用引用计数的类型不需要进行回收（整型、浮点型、布尔型，null）
	if (Z_REFCOUNTED_P(zval_ptr)) {
	    // refcount减一
		if (!Z_DELREF_P(zval_ptr)) {
		    // refcount减一后变为0，不是垃圾，正常回收
			_zval_dtor_func_for_ptr(Z_COUNTED_P(zval_ptr) ZEND_FILE_LINE_RELAY_CC);
		} else {
		    // refcount减一后仍然大于0，表示该变量可能是一个垃圾，被垃圾回收器回收。
			GC_ZVAL_CHECK_POSSIBLE_ROOT(zval_ptr);
		}
	}
}
// file:zend_gc.h
#define GC_ZVAL_CHECK_POSSIBLE_ROOT(z) \
	gc_check_possible_root((z))

static zend_always_inline void gc_check_possible_root(zval *z)
{
	ZVAL_DEREF(z);
	// 判断是否是可收集（检查变量类型掩码zval.u1.type_flag是否包含IS_TYPE_COLLECTABLE,即数组、对象）以及是否已经收集过了（通过zend_refcount_h.v.u.gc_info来判断，第一次收集后会把这个值设置为GC_PURPLE来避免重复收集）
	if (Z_COLLECTABLE_P(z) && UNEXPECTED(!Z_GC_INFO_P(z))) {
		gc_possible_root(Z_COUNTED_P(z));
	}
}
```

收集时首先会从buf中选择一个空闲节点，然后将vlaue的gc保存到这个结点中，若没有空闲节点则表明回收器已经满了，这个时候就会触发垃圾鉴定、回收程序。

```c
// file:zend_gc.c
ZEND_API void ZEND_FASTCALL gc_possible_root(zend_refcounted *ref)
{
	gc_root_buffer *newRoot;

	if (UNEXPECTED(CG(unclean_shutdown)) || UNEXPECTED(GC_G(gc_active))) {
		return;
	}

	ZEND_ASSERT(GC_TYPE(ref) == IS_ARRAY || GC_TYPE(ref) == IS_OBJECT);
	// 插入的结点必须是GC_BLACK,防止重复插入
	ZEND_ASSERT(EXPECTED(GC_REF_GET_COLOR(ref) == GC_BLACK));
	ZEND_ASSERT(!GC_ADDRESS(GC_INFO(ref)));

	GC_BENCH_INC(zval_possible_root);

    // 首先看一下unused中又没有可用的。
	newRoot = GC_G(unused);
	if (newRoot) {
	    // 有的话先用unused的，然后将GC_G(unused)指向单链表的下一个
		GC_G(unused) = newRoot->prev;
	} else if (GC_G(first_unused) != GC_G(last_unused)) {
	    // unused没有可用的，且bug中还有可用
		newRoot = GC_G(first_unused);
		GC_G(first_unused)++;
	} else {
	    // buf缓存区已满，这时需要启动垃圾鉴定、回收程序。
		...
	}
    // 将插入的red标位紫色，防止重复插入
	GC_TRACE_SET_COLOR(ref, GC_PURPLE);
	// 将该节点在buf中的位置保存到gc_info中，目的在于当后续value的refcount变为了0，需要将其从buf中删除时可以知道该value保存在那个gc_root_buffer中，如果没有这个信息，在删除vlaue时无法获取gc_root_buffer的位置。
	GC_INFO(ref) = (newRoot - GC_G(buf)) | GC_PURPLE;
	newRoot->ref = ref;
    // 插入roots链表头部
	newRoot->next = GC_G(roots).next;
	newRoot->prev = &GC_G(roots);
	GC_G(roots).next->prev = newRoot;
	GC_G(roots).next = newRoot;

	GC_BENCH_INC(zval_buffered);
	GC_BENCH_INC(root_buf_length);
	GC_BENCH_PEAK(root_buf_peak, root_buf_length);
}
```

- 第三步，删除 删除的操作通过GC_REMOVE_FROM_BUFFER()完成。

```c
// file:zend_gc.h
#define GC_REMOVE_FROM_BUFFER(p) do { \
		zend_refcounted *_p = (zend_refcounted*)(p); \
		if (GC_ADDRESS(GC_INFO(_p))) { \
			gc_remove_from_buffer(_p); \
		} \
	} while (0)
	
#define GC_ADDRESS(v) \
	((v) & ~GC_COLOR)
```

删除时，先根据gc_info取到gc_root_buffer，然后再从buf中移除，删除后再把空的gc_root_buffer插入到unused单链表尾部。

```c
// file:zend_gc.c
ZEND_API void ZEND_FASTCALL gc_remove_from_buffer(zend_refcounted *ref)
{
	gc_root_buffer *root;

	ZEND_ASSERT(GC_ADDRESS(GC_INFO(ref)));

	GC_BENCH_INC(zval_remove_from_buffer);

	if (EXPECTED(GC_ADDRESS(GC_INFO(ref)) < GC_ROOT_BUFFER_MAX_ENTRIES)) {
		root = GC_G(buf) + GC_ADDRESS(GC_INFO(ref));
		gc_remove_from_roots(root);
	} else {
		root = gc_find_additional_buffer(ref);
		gc_remove_from_additional_roots(root);
	}
	if (GC_REF_GET_COLOR(ref) != GC_BLACK) {
		GC_TRACE_SET_COLOR(ref, GC_PURPLE);
	}
	GC_INFO(ref) = 0;

	/* updete next root that is going to be freed */
	if (GC_G(next_to_free) == root) {
		GC_G(next_to_free) = root->next;
	}
}
```

当buf缓存区满了执行垃圾回收的过程如下：

```c
ZEND_API int zend_gc_collect_cycles(void)
{
	    ...
	    // (1)遍历roots链表，对当前结点vlaue的所有成员（如数组元素、成员变量）进行深度优先遍历，把成员refcount减1.
		gc_mark_roots();
		GC_TRACE("Scanning roots");
		// (2)再次遍历roots链表，检查各结点当前refcount是否为0，是的话标为white，表示是垃圾，不是的话需要还原（1），吧refcount再加回去
		gc_scan_roots();

#if ZEND_GC_DEBUG
		orig_gc_full = GC_G(gc_full);
		GC_G(gc_full) = 0;
#endif

		GC_TRACE("Collecting roots");
		additional_buffer_snapshot = GC_G(additional_buffer);
		// 将roots链表中的非白色结点删除，之后roots链表中全部是真正的垃圾，将垃圾链表转到to_free等待释放
	  
	  ...
	  
		
		// 释放垃圾
		current = to_free.next;
		while (current != &to_free) {
			next = current->next;
			p = current->ref;
			if (EXPECTED(current >= GC_G(buf) && current < GC_G(buf) + GC_ROOT_BUFFER_MAX_ENTRIES)) {
				current->prev = GC_G(unused);
				GC_G(unused) = current;
			}
			efree(p);
			current = next;
		}

	...
}
```