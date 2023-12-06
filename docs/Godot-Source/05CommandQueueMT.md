---
title: CommandQueueMT 解析
comments: true
---

# CommandQueueMT 解析
## 宏定义
### COMMA
COMMA(N)：根据传入的整数N返回一个逗号。如果N为0，则不返回任何内容。
```cpp
#define COMMA(N) _COMMA_##N
#define _COMMA_0
#define _COMMA_1 ,
#define _COMMA_2 ,
#define _COMMA_3 ,
#define _COMMA_4 ,
#define _COMMA_5 ,
#define _COMMA_6 ,
#define _COMMA_7 ,
#define _COMMA_8 ,
#define _COMMA_9 ,
#define _COMMA_10 ,
#define _COMMA_11 ,
#define _COMMA_12 ,
#define _COMMA_13 ,
#define _COMMA_14 ,
#define _COMMA_15 ,
```
### COMMA_SEP_LIST
COMMA_SEP_LIST(ITEM, LENGTH)：根据传入的整数LENGTH和参数ITEM，生成一个以逗号分隔的ITEM列表。

例如，当ITEM为X，LENGTH为5时，输出为X(1), X(2), X(3), X(4), X(5)
```cpp
// 1-based comma separated list of ITEMs
#define COMMA_SEP_LIST(ITEM, LENGTH) _COMMA_SEP_LIST_##LENGTH(ITEM)
#define _COMMA_SEP_LIST_15(ITEM) \
	_COMMA_SEP_LIST_14(ITEM)     \
	, ITEM(15)
#define _COMMA_SEP_LIST_14(ITEM) \
	_COMMA_SEP_LIST_13(ITEM)     \
	, ITEM(14)
#define _COMMA_SEP_LIST_13(ITEM) \
	_COMMA_SEP_LIST_12(ITEM)     \
	, ITEM(13)
#define _COMMA_SEP_LIST_12(ITEM) \
	_COMMA_SEP_LIST_11(ITEM)     \
	, ITEM(12)
#define _COMMA_SEP_LIST_11(ITEM) \
	_COMMA_SEP_LIST_10(ITEM)     \
	, ITEM(11)
#define _COMMA_SEP_LIST_10(ITEM) \
	_COMMA_SEP_LIST_9(ITEM)      \
	, ITEM(10)
#define _COMMA_SEP_LIST_9(ITEM) \
	_COMMA_SEP_LIST_8(ITEM)     \
	, ITEM(9)
#define _COMMA_SEP_LIST_8(ITEM) \
	_COMMA_SEP_LIST_7(ITEM)     \
	, ITEM(8)
#define _COMMA_SEP_LIST_7(ITEM) \
	_COMMA_SEP_LIST_6(ITEM)     \
	, ITEM(7)
#define _COMMA_SEP_LIST_6(ITEM) \
	_COMMA_SEP_LIST_5(ITEM)     \
	, ITEM(6)
#define _COMMA_SEP_LIST_5(ITEM) \
	_COMMA_SEP_LIST_4(ITEM)     \
	, ITEM(5)
#define _COMMA_SEP_LIST_4(ITEM) \
	_COMMA_SEP_LIST_3(ITEM)     \
	, ITEM(4)
#define _COMMA_SEP_LIST_3(ITEM) \
	_COMMA_SEP_LIST_2(ITEM)     \
	, ITEM(3)
#define _COMMA_SEP_LIST_2(ITEM) \
	_COMMA_SEP_LIST_1(ITEM)     \
	, ITEM(2)
#define _COMMA_SEP_LIST_1(ITEM) \
	_COMMA_SEP_LIST_0(ITEM)     \
	ITEM(1)
#define _COMMA_SEP_LIST_0(ITEM)
```

### SEMIC_SEP_LIST

SEMIC_SEP_LIST(ITEM, LENGTH)：类似地，这个宏根据传入的整数LENGTH和参数ITEM，生成一个以分号分隔的ITEM列表。

例如，当ITEM为Y，LENGTH为5时，输出为Y(1); Y(2); Y(3); Y(4); Y(5)
```cpp 
// 1-based semicolon separated list of ITEMs
#define SEMIC_SEP_LIST(ITEM, LENGTH) _SEMIC_SEP_LIST_##LENGTH(ITEM)
#define _SEMIC_SEP_LIST_15(ITEM) \
	_SEMIC_SEP_LIST_14(ITEM);    \
	ITEM(15)
#define _SEMIC_SEP_LIST_14(ITEM) \
	_SEMIC_SEP_LIST_13(ITEM);    \
	ITEM(14)
#define _SEMIC_SEP_LIST_13(ITEM) \
	_SEMIC_SEP_LIST_12(ITEM);    \
	ITEM(13)
#define _SEMIC_SEP_LIST_12(ITEM) \
	_SEMIC_SEP_LIST_11(ITEM);    \
	ITEM(12)
#define _SEMIC_SEP_LIST_11(ITEM) \
	_SEMIC_SEP_LIST_10(ITEM);    \
	ITEM(11)
#define _SEMIC_SEP_LIST_10(ITEM) \
	_SEMIC_SEP_LIST_9(ITEM);     \
	ITEM(10)
#define _SEMIC_SEP_LIST_9(ITEM) \
	_SEMIC_SEP_LIST_8(ITEM);    \
	ITEM(9)
#define _SEMIC_SEP_LIST_8(ITEM) \
	_SEMIC_SEP_LIST_7(ITEM);    \
	ITEM(8)
#define _SEMIC_SEP_LIST_7(ITEM) \
	_SEMIC_SEP_LIST_6(ITEM);    \
	ITEM(7)
#define _SEMIC_SEP_LIST_6(ITEM) \
	_SEMIC_SEP_LIST_5(ITEM);    \
	ITEM(6)
#define _SEMIC_SEP_LIST_5(ITEM) \
	_SEMIC_SEP_LIST_4(ITEM);    \
	ITEM(5)
#define _SEMIC_SEP_LIST_4(ITEM) \
	_SEMIC_SEP_LIST_3(ITEM);    \
	ITEM(4)
#define _SEMIC_SEP_LIST_3(ITEM) \
	_SEMIC_SEP_LIST_2(ITEM);    \
	ITEM(3)
#define _SEMIC_SEP_LIST_2(ITEM) \
	_SEMIC_SEP_LIST_1(ITEM);    \
	ITEM(2)
#define _SEMIC_SEP_LIST_1(ITEM) \
	_SEMIC_SEP_LIST_0(ITEM)     \
	ITEM(1)
#define _SEMIC_SEP_LIST_0(ITEM)
```
### SPACE_SEP_LIST
SPACE_SEP_LIST(ITEM, LENGTH)：同样地，这个宏根据传入的整数LENGTH和参数ITEM，生成一个以空格分隔的ITEM列表。

例如，当ITEM为Z，LENGTH为5时，输出为Z(1) Z(2) Z(3) Z(4) Z(5)
```cpp
// 1-based space separated list of ITEMs
#define SPACE_SEP_LIST(ITEM, LENGTH) _SPACE_SEP_LIST_##LENGTH(ITEM)
#define _SPACE_SEP_LIST_15(ITEM) \
	_SPACE_SEP_LIST_14(ITEM)     \
	ITEM(15)
#define _SPACE_SEP_LIST_14(ITEM) \
	_SPACE_SEP_LIST_13(ITEM)     \
	ITEM(14)
#define _SPACE_SEP_LIST_13(ITEM) \
	_SPACE_SEP_LIST_12(ITEM)     \
	ITEM(13)
#define _SPACE_SEP_LIST_12(ITEM) \
	_SPACE_SEP_LIST_11(ITEM)     \
	ITEM(12)
#define _SPACE_SEP_LIST_11(ITEM) \
	_SPACE_SEP_LIST_10(ITEM)     \
	ITEM(11)
#define _SPACE_SEP_LIST_10(ITEM) \
	_SPACE_SEP_LIST_9(ITEM)      \
	ITEM(10)
#define _SPACE_SEP_LIST_9(ITEM) \
	_SPACE_SEP_LIST_8(ITEM)     \
	ITEM(9)
#define _SPACE_SEP_LIST_8(ITEM) \
	_SPACE_SEP_LIST_7(ITEM)     \
	ITEM(8)
#define _SPACE_SEP_LIST_7(ITEM) \
	_SPACE_SEP_LIST_6(ITEM)     \
	ITEM(7)
#define _SPACE_SEP_LIST_6(ITEM) \
	_SPACE_SEP_LIST_5(ITEM)     \
	ITEM(6)
#define _SPACE_SEP_LIST_5(ITEM) \
	_SPACE_SEP_LIST_4(ITEM)     \
	ITEM(5)
#define _SPACE_SEP_LIST_4(ITEM) \
	_SPACE_SEP_LIST_3(ITEM)     \
	ITEM(4)
#define _SPACE_SEP_LIST_3(ITEM) \
	_SPACE_SEP_LIST_2(ITEM)     \
	ITEM(3)
#define _SPACE_SEP_LIST_2(ITEM) \
	_SPACE_SEP_LIST_1(ITEM)     \
	ITEM(2)
#define _SPACE_SEP_LIST_1(ITEM) \
	_SPACE_SEP_LIST_0(ITEM)     \
	ITEM(1)
#define _SPACE_SEP_LIST_0(ITEM)
```

### 参数宏
#define ARG(N) p##N：这个宏定义将传入的参数N与字符p拼接在一起，形成一个新的标识符。

例如，当N为2时，ARG(2)会被替换为p2
```cpp
#define ARG(N) p##N
```
#define PARAM(N) P##N p##N：这个宏定义将传入的参数N与字符P和p分别拼接在一起，形成两个新的标识符。

例如，当N为3时，PARAM(3)会被替换为P3 p3。

```cpp
#define PARAM(N) P##N p##N
```
#define TYPE_PARAM(N) class P##N：这个宏定义将传入的参数N与关键字class和字符P拼接在一起，形成一个新的类型参数声明。

例如，当N为4时，TYPE_PARAM(4)会被替换为class P4。

```cpp
#define TYPE_PARAM(N) class P##N

```

```cpp
#define PARAM_DECL(N) typename GetSimpleTypeT<P##N>::type_t p##N
```