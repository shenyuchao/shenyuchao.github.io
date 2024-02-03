+++
title = 'SDS动态字符串'
date = 2024-01-03T17:22:27+08:00
draft = false
categories = [
    "Redis"
]
tags = [
    "redis",
    "源码",
]
+++

## 对比 C 语言字符串
1.存储方式

- C 语言字符串创建后长度固定，使用 NULL 作为结尾。不能自动扩容，如果存储的数据超过了字符串的长度，会造成缓冲区溢出。

- Reids 字符串是二进制安全的可以存储任意二进制数据，而不仅仅是文本。Redis 使用字节数组来表示字符串，并且支持多种编码方式（如 raw、int、embstr 等）来优化存储。内置 len 属性记录字符串的长度，free 属性记录字节数组未使用的字节数量，支持自动扩容，并且不会造成缓冲区的溢出。

  2.字符串操作

- C 语言提供了一系列字符串操作函数，如 strlen、strcpy、strcat、strcmp 等。

- Redis 为了复用 C 语言的字符串函数也会在字符串结尾添加一个 NULL，但是字符串的长度不包含空字符。

  3.内存管理

- C 语言字符串需要手动分配，并在使用完后手动释放内存，以避免内存泄漏。

- Redis 会自动管理字符串的内存分配和释放，并且使用了预分配机制，减少内存分配的次数，提高内存使用率。字符串缩容后内存并不会被释放，而是被记录到 free 属性中，以便下次使用。

## 源码分析

### C基础知识回顾

##### __attribute__

``` c
__attribute__ ((attribute-list))
```

attribute是GCC编译器（GNU Compiler Collection）提供的一个特性，用于向编译器提供额外的指示或属性。它可以用于修改变量、函数、结构体等的行为或特性。

在redis源码中使用\_\_attribute\_\_ ((\*\*packed\*\*))将告诉编译器使用1个字节对齐，而不是编译器默认的字节数。



##### 灵活的数组成员

> 也被称为 "struct hack"。允许结构的最后一个成员是长度为零的数组，如 `int` `foo[];`。这种结构一般用作访问 malloc 内存的头文件。
>
> 例如，在结构 `struct s { int n; double d[]; } S;` 中，数组 `d` 是不完整数组类型。对于 `S` 的该成员，C 编译器不对任何内存偏移进行计数。换句话说，`sizeof(struct s)` 与 `S.n` 的偏移相同。
>
> 可以像使用任何普通数组成员一样使用 `d`。`S.d[10] = 0;`。

参考：[oracle C99文档](https://docs.oracle.com/cd/E19205-01/820-1210/bjazj/index.html)

##### 指针回顾

C语言数组名可以作为指针使用。当数组名出现在表达式中时，它会被隐式地转换为指向数组第一个元素的指针。这意味着数组名可以被用作指针来访问数组元素。

数组名作为指针:

- 例如，如果有一个整型数组 `int arr[5]`，你可以使用 `arr` 来访问数组的第一个元素，就像使用指针一样：`*arr` 或 `arr[0]`。

- 数组名做为指针是一个只读指针，无法被修改。如果使用arr= &(variables)进行赋值操作，将会报错。

数组名作为数组：

- 当数组名出现在 `sizeof` 运算符的操作数中时，它表示整个数组的大小。
- 当时用取地址符时，arr代表数组。

在C语言中，指针和数组之间有一种等价关系。

``` c
#include <stdio.h>

int main(int argc, char *argv[]) {
	char buf[] = "hello world!";
	char *s;
	s = buf;
	printf("buf pointer: %p\n", buf); 
	printf("s pointer: %p\n", s);	
	printf("buf+1 pointer: %p\n", buf+1);
	printf("buf[1] pointer: %p\n", buf[1]);
	printf("s+1 pointer: %p\n", s+1);
	printf("s[1] pointer: %p\n", s[1]);
}

// output
// buf pointer: 0x16b7c3388
// s pointer: 0x16b7c3388
// buf+1 pointer: 0x16b7c3389
// buf[1] pointer: 0x16b7c3389
// s+1 pointer: 0x16b7c3389
// s[1] pointer: 0x16b7c3389
```

#### sdshdr结构体 

``` c title="sds.h" linenums="1"
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 低3位存储类型，高5位存储长度。低三位对应的值为SDS_TYPE_8*/
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* buf中已使用长度 */
    uint8_t alloc; /* buf分配内存总长度 */
    unsigned char flags; /* 只有低三位被使用，用来标记类型。低三位对应的值为SDS_TYPE_8*/
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* buf中已使用长度 */
    uint16_t alloc; /* buf分配内存总长度 */
    unsigned char flags; /* 只有低三位被使用，用来标记类型。低三位对应的值为SDS_TYPE_16 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* buf中已使用长度 */
    uint32_t alloc; /* buf分配内存总长度 */
    unsigned char flags; /* 只有低三位被使用，用来标记类型。低三位对应的值为SDS_TYPE_32 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* buf中已使用长度 */
    uint64_t alloc; /* buf分配内存总长度 */
    unsigned char flags; /* 只有低三位被使用，用来标记类型。低三位对应的值为SDS_TYPE_64*/
    char buf[];
};

// sdshdr中flags的低3位存储的值
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7 // 使用&运算获取低3位的值
#define SDS_TYPE_BITS 3
```

### 获取sdshdr\*结构体地址

1.因为\_\_attribute\_\_ ((\*\*packed\*\*))修饰结构体使用一个字节对齐，s-1 就可以指向flags

2.通过flags低三位获取到具体类型

3.使用s-sizeof(struct shshdr\*)获取到结构体指针

``` c title="sds.c"
static inline int sdsHdrSize(char type) {
    switch(type&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5); // sizeof(sdshdr4) = flags(1) = 1
        case SDS_TYPE_8:
            return sizeof(struct sdshdr8); // sizeof(sdshdr8) = len(1) + alloc(1) + 1(flags)= 3
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16); // sizeof(sdshdr16) = len(2) + alloc(2) + 1(flags)= 5
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32); // sizeof(sdshdr32) = len(4) + alloc(4) + 1(flags)= 9
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64); // sizeof(sdshdr64) = len(8) + alloc(8) + 1(flags)= 17
    }
    return 0;
}

void *sdsAllocPtr(sds s) {
    return (void*) (s-sdsHdrSize(s[-1]));
}
```

### 创建字符串

- 创建空字符串时，将 SDS_TYPE_5 强制转换成 SDS_TYPE_8(注释中说：空字符串常用于追加，SDS_TYPE_5 的效果没有 SDS_TYPE_8 效果好，猜测是可能追加的字节一般都大于 31 个字节)
- 内存分配时，会多分配一个字节，用于存储字符串的结束符 '\0'，但是 len 和alloc不会计算末尾null
- 返回值是指向是 sds(char \*), 而不是 sdshdr 结构体的指针，这样返回的字符指针可以方便的使用 C 语言字符串函数，如果想要访问结构体其他字段可以使用指针偏移的方式访问

``` c title="sds.c" linenums="1"
typedef char *sds;

sds sdsnewlen(const void *init, size_t initlen) {
    return _sdsnewlen(init, initlen, 0);
}

// 计算不同sdshder类型允许的最大长度
static inline size_t sdsTypeMaxSize(char type) {
    if (type == SDS_TYPE_5)
        return (1<<5) - 1;
    if (type == SDS_TYPE_8)
        return (1<<8) - 1;
    if (type == SDS_TYPE_16)
        return (1<<16) - 1;
#if (LONG_MAX == LLONG_MAX)
    if (type == SDS_TYPE_32)
        return (1ll<<32) - 1;
#endif
    return -1; /* this is equivalent to the max SDS_TYPE_64 or SDS_TYPE_32 */
}

sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    /* 空字符串通常用于追加。使用类型8，因为类型5在这方面效果不好. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
    size_t usable;

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        // init == "SDS_NOINIT"时，不重置内存中的数据为0，节省时间
        init = NULL;
    else if (!init)
        // init指针为空时，重置内存中的数据为0
        memset(sh, 0, hdrlen+initlen+1);
  	// 结构体buf指向的地址 
    s = (char*)sh+hdrlen;
  	// 结构体flags指向的地址
    fp = ((unsigned char*)s)-1;
    // 分配的内存字节数-结构体占用字节数-NUll字节数=结构体中buf可用的字节数
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
    }
    // 将init指针指向的数据拷贝到sds字符串中
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    // 返回sds的指针，而不是SDSHDR的指针
    return s;
}
```

### 删除sds
1.如果字符串是 NULL 直接返回

2.通过 s-1 获取到 flags，得到对应的结构体类型长度，s-sizeof(struct sdddr)获取到结构体地址

3.使用 free 释放结构体地址

``` c title="sds.c" linenums="1"
/* 释放一个sds字符串，如果s是NULL直接返回 */
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```

### 重置sds长度

- 老样子 sds-1 获取 flags ，flags&SDS_TYPE_MASK 获取到类型，s-sizeof(struct sdddr)获取到结构体地址
- 通过获取到结构体指针，然后将 sdshdr->len 设置为 0

```c title="sds.c"
void sdsclear(sds s) {
  	// 设置结构体len字段
    sdssetlen(s, 0);
    s[0] = '\0';
}
```

### 拼接字符串(扩容)

```c title="sds.c" linenums="1"
sds sdscatsds(sds s, const sds t) {
    return sdscatlen(s, t, sdslen(t));
}

sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);// 获取当前长度
    s = sdsMakeRoomFor(s,len);	// 检查新增长度是否能装下
    if (s == NULL) return NULL;// 分配失败直接返回
    memcpy(s+curlen, t, len);// 直接拷贝追加
    sdssetlen(s, curlen+len);// 设置新的长度
    s[curlen+len] = '\0';// 结尾设置null
    return s;
}

/* 扩大 sds 字符串末尾的可用空间,
 * 这有助于避免在重复追加 sds 时重复重新分配. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    return _sdsMakeRoomFor(s, addlen, 1);
}

sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);// 获取可以长度， alloc - len
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;// 初始化新的type,并获取旧的type，后面进行比较
    int hdrlen; // 结构体长度
    size_t usable; // buf长度

    /* 如果有足够的空间，直接返回*/
    if (avail >= addlen) return s;

    len = sdslen(s); //旧的len
    sh = (char*)s-sdsHdrSize(oldtype); //结构体地址
    reqlen = newlen = (len+addlen); // 旧的len + 需要增加的长度  = 新的长度
    assert(newlen > len);   /* Catch size_t overflow */
    if (greedy == 1) {
        if (newlen < SDS_MAX_PREALLOC) // SDS_MAX_PREALLOC = 1024 * 1024 = 1M，新的长度小于1M，则扩容两倍
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC; // 新的长度大于1M，则额外分配1M
    }

    type = sdsReqType(newlen); //根据长度得出新的类型，因为长度可能变化导致无法承载

    //在Redis的SDS实现中，SDS类型5是一种优化的字符串类型，称为"embstr"（embedded string）。
    //它的特点是将短字符串直接存储在				SDS结构体中，而不是动态分配额外的缓冲区。
    //然而，这种类型的字符串并不适用于需要频繁进行字符串追加操作的场景。
  	//因为类型5的SDS字符串不会记住空闲空间，也就是说，每次进行字符串追加操作时，都需要调用sdsMakeRoomFor()函数来确保有足够的空间。
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    if (oldtype==type) {
        // 类型没有改变，使用realloc
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* 类型改变，使用malloc分配内存，调整字符串的位置 */
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh); // 释放旧的
        s = (char*)newsh+hdrlen; // 新的buf地址
        s[-1] = type; // 新的flags
        sdssetlen(s, len); // 设置len
    }
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable); // 设置alloc
    return s;
}
```

**参考资料**

- [Redis设计与实现](https://book.douban.com/subject/34804798/)
- [Redis5设计与源码分析](https://book.douban.com/subject/34968025/)