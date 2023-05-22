> 本书描述的 v5.1.4 的 Lua 源码，而笔者使用的是 v5.4.2，内容虽大相径庭，笔者会根据自己理解阐述总结  
> Lua 的源码设计思路清晰，结构分明，是不可多得的短小精悍的源代码，只能说相见恨晚  
# Chap 1. 概述
- 名字含义  
葡萄牙语中 SOL（simple object language，Lua的前身）的含义是太阳，Lua 的意思是月亮，可能这也是 Lua icon 样式的由来 🌕🌔  
- Lua目录结构（v5.4.2）  
``` c
/* 虚拟机 */
// LUA接口辅助库
lapi.c
lapi.h
// 调试辅助函数
ldebug.c
ldebug.h
// LUA虚拟机
lvm.c
lvm.h
// 函数调用和栈管理
ldo.c
ldo.h
// 函数原型和闭包的辅助库
lfunc.c
lfunc.h
// LUA全局表，记录虚拟机的状态
lstate.c
lstate.h
// 元表tag方法
ltm.c
ltm.h
// 核心API
lua.c
lua.h
lua.hpp

/* 数据结构 */
// LUA数据类型定义 lu_type
lctype.c
lctype.h
// LUA对象定义
lobject.c
lobject.h
// LUA字符串
lstring.c
lstring.h
// LUA表
ltable.c
ltable.h

/* 编译原理 */
// 源码生成器
lcode.c
lcode.h
// 代码预编译
ldump.c
// 加载预编译代码
lundump.c
lundump.h
// LUA编译器
luac.c
// 词法分析器
llex.c
llex.h
// 语法语义分析器
lparser.c
lparser.h
// make
Makefile 

/* 内存 */
// 内存管理
lmem.c
lmem.h
// 垃圾回收
lgc.c
lgc.h

// 定义了虚拟机的操作跳转
ljumptab.h 
// LUA虚拟机操作码
lopcodes.c
lopcodes.h
// LUA操作码数组定义，有序
lopnames.h 
// 跨平台兼容
lprefix.h
// 一些限制，基础类型，安装依赖的定义
llimits.h

/* LIBRARY */
linit.c     // 负责库的初始化
lbaselib.c  // 基础库
lcorolib.c  // 协程库
ldblib.c    // 调式库
lutf8lib.c  // 标准UTF-8库
liolib.c    // I/O库
lmathlib.c  // 数学库
loadlib.c   // 动态扩展库加载器
loslib.c    // 标准系统调用库
lstrlib.c   // 字符串库
ltablib.c   // 表库
lualib.h    // LUA标准库
//辅助函数库：主要提供了错误处理接口和虚拟机状态维护
lauxlib.c
lauxlib.h

// 配置文件
luaconf.h

// 字节流
lzio.c
lzio.h
```
# Chap 2. Lua 中的数据类型
> 本章的数据结构基本都在 lobject.h 定义，读者可以根据自己本地的 Lua 版本阅读源码  
- Lua基本数据类型（v5.4.2）  
``` c
#define LUA_TNONE		(-1)        // 无
#define LUA_TNIL		0           // nil值
#define LUA_TBOOLEAN		1       // 布尔值
#define LUA_TLIGHTUSERDATA	2   // void*
#define LUA_TNUMBER		3         // lua_Number
#define LUA_TSTRING		4         // TString
#define LUA_TTABLE		5         // Table
#define LUA_TFUNCTION		6       // C闭包 L(ua)闭包
#define LUA_TUSERDATA		7       // void*
#define LUA_TTHREAD		8         // lua_State
#define LUA_NUMTYPES		9       // 用于下面的定义
// Extra types for collectable non-values 用于可回收的“无值”类型
#define LUA_TUPVAL	LUA_NUMTYPES  /* upvalues */ 上值
#define LUA_TPROTO	(LUA_NUMTYPES+1)  /* function prototypes */ 函数原型
#define LUA_TDEADKEY	(LUA_NUMTYPES+2)  /* removed keys in tables */ table里移除了的key
// number of all possible types (including LUA_TNONE but excluding DEADKEY)
#define LUA_TOTALTYPES	(LUA_TPROTO + 2)  // 除了 LUA_TDEADKEY 的所有类型
```

- Lua不同数据"类"的实现（v5.4.2）   
> 因为 Lua 用标准C实现，所以没有用到cpp的class，用struct配合union实现了类的“多态”  
> 这里的 TValue 就是一个标准的 variant可变体 允许在运行时决定变量的类型  
> CPP17里也提供了 std::variant 库实现类似的行为  
``` c
typedef unsigned char lu_byte;
#define CommonHeader	struct GCObject *next; lu_byte tt; lu_byte marked
#define TValuefields	Value value_; lu_byte tt_

typedef struct GCObject {
  CommonHeader;
} GCObject;

typedef union Value {
  struct GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;

typedef struct TValue {
  TValuefields;
} TValue;
```
对上述代码进行宏展开，我们可以发现，TValue 是 Lua 中最通用的数据结构，  
根据 tt_ 字段的不同（lctype），可以用一个结构体表述各种类型的数据，  
value_ 字段则描述了数据详情，因为用的是 union，控制内存大小（不为其他类型分配多余内存）且固定大小效率更好  

# Chap 3. 字符串
> 类似 const char* 的字符串，Lua也为自己维护了一份字符串“常量”表，对于相同的字符串，设计成访问相同的地址诚然是较好的  
> 书中称之为 internalization 内化，相同的数据不使用副本，而是复用它的引用  
- 字符串数据结构（v5.4.2）  
``` c
#define LUA_VSHRSTR	makevariant(LUA_TSTRING, 0)  /* short strings */
#define LUA_VLNGSTR	makevariant(LUA_TSTRING, 1)  /* long strings */

typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */ 长字符串专属，长字符串的长度
    struct TString *hnext;  /* linked list for hash table */ 短字符串专属，hash开链法
  } u;
  char contents[1];
} TString;
```
从官方备注可知，字符串类型内还分了长短两种类型，它们根据长度区分，内容显然是存储在 contents 数组了，  
那么短字符串呢，我们接着往下看  
``` c
#define LUAI_MAXSHORTLEN	40 
#define getstr(ts)  ((ts)->contents)

TString *luaS_newlstr (lua_State *L, const char *str, size_t l) {
  if (l <= LUAI_MAXSHORTLEN)  /* short string? */
    return internshrstr(L, str, l);
  else {
    TString *ts;
    if (unlikely(l >= (MAX_SIZE - sizeof(TString))/sizeof(char)))
      luaM_toobig(L);
    ts = luaS_createlngstrobj(L, l);
    memcpy(getstr(ts), str, l * sizeof(char));
    return ts;
  }
}
```
从初始化函数可知，大于40字节的被归类为长字符串，下面分别看看两种字符串的初始化方式  
- LUA_VLNGSTR
``` c
static TString *createstrobj (lua_State *L, size_t l, int tag, unsigned int h) {
  TString *ts;
  GCObject *o; // Lua对象基本都需要垃圾回收，所以都会由o这个万能union转化
  size_t totalsize;  /* total size of TString object */
  totalsize = sizelstring(l);
  o = luaC_newobj(L, tag, totalsize); // 初始化GCObject
  ts = gco2ts(o); // GCObject -> TString
  ts->hash = h;
  ts->extra = 0;
  getstr(ts)[l] = '\0';  /* ending 0 */
  return ts;
}

TString *luaS_createlngstrobj (lua_State *L, size_t l) {
  TString *ts = createstrobj(L, l, LUA_VLNGSTR, G(L)->seed);
  ts->u.lnglen = l;
  return ts;
}
```
可见长字符串的创建以外的简单，不过是创建了 GC对象（为了垃圾回收，下面再细说）然后强转为字符串类型  
最后把 str 的内容通过 memcpy 复制到 contents 里面  

- LUA_VSHRSTR
``` c
#define check_exp(c,e)		(lua_assert(c), (e))
#define lmod(s,size) \   
	(check_exp((size&(size-1))==0, (cast_int((s) & ((size)-1)))  ))
/* 前面对size和size-1做了个and处理，实际上是判断其是否为2的指数，是哈希算法的要求 */  
typedef struct stringtable {
  TString **hash; //开链表处理哈希冲突，*hash[key]为相同key值的list
  int nuse;  /* number of elements */ 短字符串个数
  int size;
} stringtable;

static TString *internshrstr (lua_State *L, const char *str, size_t l) {
  TString *ts;
  //获取全局的stringtable，实现内化的方法，后面讨论虚拟机的时候会提及
  global_State *g = G(L);
  stringtable *tb = &g->strt;
  //对当前的短字符串hash取值,在list中查询有无此值
  unsigned int h = luaS_hash(str, l, g->seed);
  TString **list = &tb->hash[lmod(h, tb->size)];
  lua_assert(str != NULL);  /* otherwise 'memcmp'/'memcpy' are undefined */ 保证str不为空指针，否则内存比较/复制是未定义的行为
  for (ts = *list; ts != NULL; ts = ts->u.hnext) {
    if (l == ts->shrlen && (memcmp(str, getstr(ts), l * sizeof(char)) == 0)) {
      /* found! */
      if (isdead(g, ts))  /* dead (but not collected yet)? */ 未垃圾回收的字符串
        changewhite(ts);  /* resurrect it */ 把它从ow（otherwhite）改为white，GC相关，后面提及
      return ts;
    }
  }
  /* else must create a new string */ 找不到就创建一个短字符串
  if (tb->nuse >= tb->size) {  /* need to grow string table? */
    growstrtab(L, tb); //now use >= size，扩容咯
    list = &tb->hash[lmod(h, tb->size)];  /* rehash with new size */
  }
  ts = createstrobj(L, l, LUA_VSHRSTR, h);
  memcpy(getstr(ts), str, l * sizeof(char));
  ts->shrlen = cast_byte(l);
  // 头插
  ts->u.hnext = *list; //新对象的next指向链表头
  *list = ts; //链表头节点指针指向新对象
  tb->nuse++; // now use +1
  return ts;
}
```
短字符串的处理则较为复杂，首先对 str 进行 luaS_hash，在对哈希值根据哈希数组长度 tb->size 求余数  
算出在数组中的位置后，就可以遍历该位置的链表了，查找等长且内存相同（const char*）的短字符串  
在找不到的时候，其实也是正常创建一个新对象然后 memcpy，但因为hash开链表需要把新对象头插到 list 里  
并且短字符因为有 stringtable 这一层缓存，还可能存在扩容的情况，所以可以关心一下扩容的方式  
``` c
#define MAX_SIZET	((size_t)(~(size_t)0)) //对0求反码，获取全1的最大值
#define MAX_INT		INT_MAX  /* maximum value of an int */
#define luaM_limitN(n,t)  \ //把INT_MAX强转为size_t，然后与 MAX_SIZET和sizeof(TSTRING*) 的余数比较，返回较小的一个
  ((cast_sizet(n) <= MAX_SIZET/sizeof(t)) ? (n) : cast_uint((MAX_SIZET/sizeof(t))))
#define MAXSTRTB	cast_int(luaM_limitN(MAX_INT, TString*)) //将上述结果转回INT值

static void growstrtab (lua_State *L, stringtable *tb) {
  if (unlikely(tb->nuse == MAX_INT)) {  /* too many strings? */
    luaC_fullgc(L, 1);  /* try to free some... */
    if (tb->nuse == MAX_INT)  /* still too many? */
      luaM_error(L);  /* cannot even create a message... */
  }
  if (tb->size <= MAXSTRTB / 2)  /* can grow string table? */
    luaS_resize(L, tb->size * 2);
}

static void tablerehash (TString **vect, int osize, int nsize) {
  int i;
  for (i = osize; i < nsize; i++)  /* clear new elements */
    vect[i] = NULL;
  for (i = 0; i < osize; i++) {  /* rehash old part of the array */
    TString *p = vect[i];
    vect[i] = NULL;
    while (p) {  /* for each string in the list */
      TString *hnext = p->u.hnext;  /* save next */
      unsigned int h = lmod(p->hash, nsize);  /* new position */
      p->u.hnext = vect[h];  /* chain it into array */
      vect[h] = p;
      p = hnext;
    }
  }
}

void luaS_resize (lua_State *L, int nsize) {
  stringtable *tb = &G(L)->strt;
  int osize = tb->size;
  TString **newvect; //声明新hash数组
  if (nsize < osize)  /* shrinking table? */ 缩小hash表时可以先做rehash
    tablerehash(tb->hash, osize, nsize);  /* depopulate shrinking part */
  newvect = luaM_reallocvector(L, tb->hash, osize, nsize, TString*);
  if (unlikely(newvect == NULL)) {  /* reallocation failed? */
    // 没想到缩表时内存居然分配失败了，把刚刚做的rehash回退吧。。。
    if (nsize < osize)  /* was it shrinking table? */ 
      tablerehash(tb->hash, nsize, osize);  /* restore to original size */
    /* leave table as it was */ 
  }
  else {  /* allocation succeeded */
    tb->hash = newvect;
    tb->size = nsize;
    if (nsize > osize) // 扩容成了，重新hash每个短字符串的位置吧
      tablerehash(newvect, osize, nsize);  /* rehash for new size */
  }
}
```
在扩容时，hash数组每次都会以 2倍 的速度进行，直至达到 MAXSTRTB 定义的最大值  
扩容完成后，会做一次rehash；缩容进行前，会做一次rehash，根据内存分配是否成功决定是否回退  
> 在本文件内相互引用的逻辑基本就是这样，后面GC相关还会再提及字符串的删除等操作  

# Chap 4. 表
> Lua 的招牌菜——表，使用极其便利，几乎支持任意类型的 k-v 键值对  
> 但使用不当可能会造成效率低性能差的问题，通过研究源代码，让我们逐步揭开它方便好使的背后逻辑  
- 表的数据结构（v5.4.2）
	+ TValuefields  
描述哈希键对应的值，对应例子中 value_:"hello", tt_:LUA_VSHRSTR  
	+ key_tt：lu_byte  
描述哈希键的类型，对应例子中 LUA_VSHRSTR  
	+ key_val：Value  
描述哈希键的值，对应例子中 "hi"   
	+ next：int  
指向相同hash值的下一个节点，这里用int表示 offset偏移量 因为是数组结构，对指针进行加减就可以找到对应位置  
> 举例说明：假设有 *local t = { ["hi"] = "hello" }*  
``` c
typedef union Node {
  struct NodeKey {
    TValuefields;  /* fields for value */
    lu_byte key_tt;  /* key type */
    int next;  /* for chaining */
    Value key_val;  /* key value */
  } u;
  TValue i_val;  /* direct access to node's value as a proper 'TValue' */
} Node;

typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */ 标记表的元方法是否提供
  lu_byte lsizenode;  /* log2 of size of 'node' array */ log2(size)的值，说明size是2的指数，上面提到的 size&(size-1)
  unsigned int alimit;  /* "limit" of 'array' array */ 数组大小，有2种含义
  TValue *array;  /* array part */ 普通数组
  Node *node; // 指向hash数组起始位
  Node *lastfree;  /* any free position is before this position */ 指向hash数组空闲位
  struct Table *metatable; // 元表
  GCObject *gclist; // 表内所有元素的GCObject的链表
} Table;
```
- alimit是什么  
> 看看官方注释 About 'alimit': if 'isrealasize(t)' is true, then 'alimit' is the real size of 'array'.   
> Otherwise, the real size of 'array' is the smallest power of two not smaller than 'alimit' (or zero iff 'alimit'is zero);   
> 'alimit' is then used as a hint for #t.  
``` c
#define limitequalsasize(t)	(isrealasize(t) || ispow2((t)->alimit))
LUAI_FUNC unsigned int luaH_realasize (const Table *t) {
  if (limitequalsasize(t))
    return t->alimit;  /* this is the size */
  else {
    unsigned int size = t->alimit;
    /* compute the smallest power of 2 not smaller than 'n' */
    size |= (size >> 1);
    size |= (size >> 2);
    size |= (size >> 4);
    size |= (size >> 8);
    size |= (size >> 16);
#if (UINT_MAX >> 30) > 3
    size |= (size >> 32);  /* unsigned int has more than 32 bits */
#endif
    size++;
    lua_assert(ispow2(size) && size/2 < t->alimit && t->alimit < size);
    return size;
  }
}
```
> iff：当且仅当。翻译一下，就是当 isrealasize 返回 true 时alimit是array的真实大小  
> 否则 array 的大小是不小于 alimit 的最小的 2<sup>n</sup>，当且仅当 alimit=0 时，array大小是0  
> 比如说alimit=7，那大小其实就是8，alimit=25，大小其实就是32，保证数组大小是 pow(2) 是有益的  

- 表的一些访问宏  
``` c
#define gnode(t,i)	(&(t)->node[i])
#define gval(n)		(&(n)->i_val)
#define gnext(n)	((n)->u.next)
```

在介绍 table 的构造之前，需要先介绍一下2个辅助数据结构，分别是 dummyNode 和 mainposition 
- dummyNode  
``` c
#define dummynode		(&dummynode_)
static const Node dummynode_ = {
  {{NULL}, LUA_VEMPTY,  /* value's value and type */
   LUA_VNIL, 0, {NULL}}  /* key type, next, and key value */
};
// 为方便理解，上述代码可以转化为
dummynode_.u.value_ = {NULL};		// Value(union)
dummynode_.u.tt_ = LUA_VEMPTY;		// lu_byte
dummynode_.u.key_tt = LUA_VNIL;		// lu_byte
dummynode_.u.next = 0;			// int
dummynode_.u.key_val = {NULL};		// Value(union)
```
声明一个全局静态常量节点，并且将其各种变量定义为空指针/空值  
这个空节点其实是用了判断Hash数组是否为空的，也是一种常见的编程方式（头前节点？）  

- mainposition与哈希算法  
因为LUA表使用开地址法处理哈希冲突，导致一个问题就是：后来的新key经过hash后，发现自己的节点被占用  
一个 key 经过 hash 后得到的未处理过的值，称为这个 key 的 mainposition首要位置  
``` c
#define twoto(x)	(1<<(x))
#define sizenode(t)	(twoto((t)->lsizenode))
// 默认数据类型
#define hashpow2(t,n)		(gnode(t, lmod((n), sizenode(t))))
#define hashstr(t,str)		hashpow2(t, (str)->hash)
#define hashboolean(t,p)	hashpow2(t, p)
#define hashint(t,i)		hashpow2(t, i)
// 自定义指针
#define hashmod(t,n)		(gnode(t, ((n) % ((sizenode(t)-1)|1))))
#define hashpointer(t,p)	hashmod(t, point2uint(p))
#define point2uint(p)		((unsigned int)((size_t)(p) & UINT_MAX))

static Node *mainposition (const Table *t, int ktt, const Value *kvl) {
  switch (withvariant(ktt)) {
    case LUA_VNUMINT: // int
      return hashint(t, ivalueraw(*kvl));
    case LUA_VNUMFLT: // float
      return hashmod(t, l_hashfloat(fltvalueraw(*kvl)));
    case LUA_VSHRSTR: // short str
      return hashstr(t, tsvalueraw(*kvl));
    case LUA_VLNGSTR:  // long str
      return hashpow2(t, luaS_hashlongstr(tsvalueraw(*kvl)));
    case LUA_VFALSE: // bool false
      return hashboolean(t, 0);
    case LUA_VTRUE: // bool true
      return hashboolean(t, 1);
    case LUA_VLIGHTUSERDATA: // bool lightudata
      return hashpointer(t, pvalueraw(*kvl));
    case LUA_VLCF: // bool
      return hashpointer(t, fvalueraw(*kvl));
    default:
      return hashpointer(t, gcvalueraw(*kvl));
  }
}
```
先不管hash的2参数各种繁杂的宏和函数，他们前面的实现都是清晰简洁的  
- 对一般数据类型而言  
因为 hashpow2 的存在，前面 lsizenode 才需要对2取对数  
因为 sizenode 会将 1 左移 lsizenode 位，获得当前 hash 数组长度  
然后对处理好的数值进行mod运算后，得出的最终值就是他的主位置  
- 对自定义指针而言  
唯一的区别就在 (sizenode(t)-1)|1 这里，对数组长度减1后对末位补1  
> 这是编程的巧思也是数学的美，GPT告诉我：  
> 在哈希取模的过程中，通常会使用模数的一种特殊形式，即模数为2的幂次方加1（这里是-1）  
> 比如 3、5、17、257 等都是这种形式。而在这种情况下，模数的末尾需要补1（3、7、15、255）  
> 这是因为，对于模数为2的幂次方加1的形式，可以使用一种更高效的计算方法——使用位运算来代替除法运算  
> 具体地说，我们可以使用按位与操作（&）来代替取模运算（%），这样可以大大提高计算速度  
> 但是，为了使用这种方法，我们需要保证模数的二进制表示中，除了最高位以外，其他位都是1  
> 因此，我们需要在模数的末尾补1，以满足这个条件。  

简单来说，就是将 pow2的数组长度 转为一个质数，因为被mod数中可能存在大量2的因子（前提）  
用 pow2数 去取模，hash冲突会十分严重，±1后补1，就可以避免这一点  

> [可参考](https://www.zhihu.com/question/20806796)  

- 表的创建
``` c
int luaO_ceillog2 (unsigned int x) {
  static const lu_byte log_2[256] = {  /* log_2[i] = ceil(log2(i - 1)) */
    0,1,2,2,3,3,3,3, 4,4,4,4,4,4,4,4, 5,5,5,5,5,5,5,5, 5,5,5,5,5,5,5,5,
    6,6,6,6,6,6,6,6, 6,6,6,6,6,6,6,6, 6,6,6,6,6,6,6,6, 6,6,6,6,6,6,6,6,
    7,7,7,7,7,7,7,7, 7,7,7,7,7,7,7,7, 7,7,7,7,7,7,7,7, 7,7,7,7,7,7,7,7,
    7,7,7,7,7,7,7,7, 7,7,7,7,7,7,7,7, 7,7,7,7,7,7,7,7, 7,7,7,7,7,7,7,7,
    8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8,
    8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8,
    8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8,
    8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8, 8,8,8,8,8,8,8,8
  };
  int l = 0;
  x--; // 修正数组下标，兼容边界情况
  while (x >= 256) { l += 8; x >>= 8; } // 大于等于256，log2的值每次可以+8，原值除256，进入下一轮
  return l + log_2[x]; //空间换时间
}

static void setnodevector (lua_State *L, Table *t, unsigned int size) {
  if (size == 0) {  /* no elements to hash part? */
    t->node = cast(Node *, dummynode);  /* use common 'dummynode' */
    t->lsizenode = 0;
    t->lastfree = NULL;  /* signal that it is using dummy node */
  }
  else {
    int i;
    int lsize = luaO_ceillog2(size); // 对log2的值向上取整
    if (lsize > MAXHBITS || (1u << lsize) > MAXHSIZE)
      luaG_runerror(L, "table overflow");
    size = twoto(lsize);
    t->node = luaM_newvector(L, size, Node);
    for (i = 0; i < (int)size; i++) {
      Node *n = gnode(t, i);
      gnext(n) = 0;
      setnilkey(n);
      setempty(gval(n));
    }
    t->lsizenode = cast_byte(lsize);
    t->lastfree = gnode(t, size);  /* all positions are free */
  }
}

Table *luaH_new (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_VTABLE, sizeof(Table));
  Table *t = gco2t(o);
  t->metatable = NULL; // 元表为空
  t->flags = cast_byte(maskflags);  /* table has no metamethod fields */ 无任何元方法
  t->array = NULL; // 默认数值为空
  t->alimit = 0; // 默认哈希表大小为0
  // 初始化啥也没有，走size为0的逻辑：node指向&dummyNode_，lsizenode为0，lastfree为空
  setnodevector(L, t, 0); 
  return t;
}
```
无论是什么自定义类型，在Lua中都是需要先生成 GCObject 然后再强转为对应类型，Table也不例外  
新表创建出来，啥也没有十分干净，非0即NULL，于是我们看看如何为它增添一些成员  
> 除了 int float 和 自定义指针，他们的回收比较不一样  

- 表 k-v 的插入
``` c
TValue *luaH_set (lua_State *L, Table *t, const TValue *key) {
  const TValue *p = luaH_get(t, key);
  if (!isabstkey(p))
    return cast(TValue *, p);
  else return luaH_newkey(L, t, key);
}

TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp; // 主位置节点
  TValue aux; // 辅助值，浮点型专用
  if (unlikely(ttisnil(key)))
    luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {
    lua_Number f = fltvalue(key);
    lua_Integer k;
    if (luaV_flttointeger(f, &k, F2Ieq)) {  /* does key fit in an integer? */
      setivalue(&aux, k); // 假设float能转为int，用转化后的新值代替
      key = &aux;  /* insert it as an integer */
    }
    else if (unlikely(luai_numisnan(f)))
      luaG_runerror(L, "table index is NaN");
  }
  mp = mainpositionTV(t, key); // 计算当前新值的主位置
  if (!isempty(gval(mp)) || isdummy(t)) {  /* main position is taken? */ 主位置非空 或 空表
    Node *othern; // 临时节点用于比较
    Node *f = getfreepos(t);  /* get a free place */
    if (f == NULL) {  /* cannot find a free place? */ 假设是空表，这里肯定会走到扩容，否则就是表被塞满的情况
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      return luaH_set(L, t, key);  /* insert key into grown table */ 扩容后重新走一遍newKey
    }
    lua_assert(!isdummy(t)); // 断言此时已经不可能为空表，因为前面刚刚做了扩容
    othern = mainposition(t, keytt(mp), &keyval(mp)); // 获取主位置节点k-v的主位置
    if (othern != mp) {  /* is colliding node out of its main position? */ 你是本来就该在这的，还是被赶过来的
      /* yes; move colliding node into free position */ 如果是被赶过来的，我才是应该在这的，你让个位置吧
      while (othern + gnext(othern) != mp)  /* find previous */ 从你应该在的主位置节点一直往下找，找到你的上一个节点（链表）
        othern += gnext(othern); // 因为是偏移量，对指针进行运算即可
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */ 找到你的上一个节点了（现在的othern），修改它的偏移量吧
      // 因为你会被移到lastfree所在位置，所以上一个节点(othern)和你(mp)的偏移量就是 f - othern 
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */ mp当前还是指向你这个被赶过来的，把你复制到lastfree位置吧
      if (gnext(mp) != 0) { // 如果你后面还有别的节点，那就修正一下他的偏移量吧
        gnext(f) += cast_int(mp - f);  /* correct 'next' */ 你现在来到 lastfree 的位置了，修正你和你下一个节点的偏移量
        gnext(mp) = 0;  /* now 'mp' is free */ mp自由了
      }
      setempty(gval(mp)); // 清空mp节点 i_val 的 tt_ 为 LUA_VEMPTY
    }
    else {  /* colliding node is in its own main position */ 你本来就在这的，我主位置和你冲突了，我自己走
      /* new node will go into free position */
      if (gnext(mp) != 0) // 头插我进链表
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else 
      	lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f; //修改mp指针到lastfree位置
    }
  }
  setnodekey(L, mp, key); //更新key对应的TValue到mp所指位置
  luaC_barrierback(L, obj2gco(t), key); //内存写屏障，防止在GC时对象被错误地释放
  lua_assert(isempty(gval(mp))); // 断言这个值为 LUA_TNIL 类型，因为新 Node 里面的 i_val 还未初始化
  return gval(mp); // 返回mp所指节点的 i_val 值，方便后续赋值
}
```
在 luaH_newkey 中，比较容易令人迷惑的就是 mp != othern 这个不等的比较  
前面提到，因为用开地址法处理哈希冲突，所以新节点算出来的主位置是没错  
但可能占用了这个位置的节点，是因为它的主节点也被占用（mp所指节点产生了哈希冲突）  
不得已才过来的；因此，对mp的 k-v 计算它实际的主节点是很必要的  
如果发现这不是你地盘，你就该立马让个位出来，顺着 lastfree 找个位置呆着吧  
详细过程看代码注释，如果还是比较抽象就可以看看他人的图文总结 [参考资料](https://blog.csdn.net/chqj_163/article/details/118050890)  

- 空闲节点的获取
``` c
static Node *getfreepos (Table *t) {
  if (!isdummy(t)) { // 非空表
    while (t->lastfree > t->node) { // 非头结点
      t->lastfree--; // 前移
      if (keyisnil(t->lastfree)) // 空节点返回
        return t->lastfree;
    }
  }
  return NULL;  /* could not find a free place */
}
```
这部分逻辑比较简单，从上面创建表的过程中，我们可以发现 lastfree 的指向并没有改版  
实际上，他是在 创表luaH_new 或者 扩/缩容luaH_resize 的时候指向了哈希数组的末尾(通过setnodevector)  
然后在后续使用的时候通过 自减 的方式找空位，找到了此时的指向就是从后往前第一个空节点了  

- 哈希节点的获取
``` c
static const TValue absentkey = {ABSTKEYCONSTANT};
// value_ = {NULL}; tt_ = LUA_VABSTKEY;

const TValue *luaH_get (Table *t, const TValue *key) {
  switch (ttypetag(key)) { // key->tt_
    case LUA_VSHRSTR: return luaH_getshortstr(t, tsvalue(key));
    case LUA_VNUMINT: return luaH_getint(t, ivalue(key));
    case LUA_VNIL: return &absentkey;
    case LUA_VNUMFLT: {
      lua_Integer k;
      if (luaV_flttointeger(fltvalue(key), &k, F2Ieq)) /* integral index? */
        return luaH_getint(t, k);  /* use specialized version */
      /* else... */
    }  /* FALLTHROUGH */
    default:
      return getgeneric(t, key, 0);
  }
}

// LUA_NUMBER 包括 FLOAT 和 INT 都是最终走 getint，只有不带整数值的float能走到通用接口
const TValue *luaH_getint (Table *t, lua_Integer key) {
  if (l_castS2U(key) - 1u < t->alimit)  /* 'key' in [1, t->alimit]? */
    return &t->array[key - 1]; // 整形值在数组范围内，返回数组对应位置引用
  else if (!limitequalsasize(t) &&  /* key still may be in the array part? */
           (l_castS2U(key) == t->alimit + 1 ||
            l_castS2U(key) - 1u < luaH_realasize(t))) {
// 另一种情况，因为alimit并非实际数组大小，这里会可能修正alimit == realsize
// 假设alimit=5，此时size=8，如果key为8，第一个判断会false
// 因此将 key == alimit+1 和 key-1 < realsize 的情况考虑进来
// 那么按照刚刚的例子 (8 == 5+1) false || (8-1 < 8) true 
    t->alimit = cast_uint(key);  /* probably '#t' is here now */ alimit修正为8 刚好 limitequalsasize
    return &t->array[key - 1]; 
  }
  else {
    Node *n = hashint(t, key);
    for (;;) {  /* check whether 'key' is somewhere in the chain */
      if (keyisinteger(n) && keyival(n) == key)
        return gval(n);  /* that's it */
      else {
        int nx = gnext(n);
        if (nx == 0) break;
        n += nx;
      }
    }
    return &absentkey;
  }
}

// LUA_TSTRING 的 LUA_VSHRSTR 使用，LUA_VLNGSTR 走通用接口
const TValue *luaH_getshortstr (Table *t, TString *key) {
  Node *n = hashstr(t, key);
  lua_assert(key->tt == LUA_VSHRSTR);
  for (;;) {  /* check whether 'key' is somewhere in the chain */
    if (keyisshrstr(n) && eqshrstr(keystrval(n), key))
      return gval(n);  /* that's it */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return &absentkey;  /* not found */
      n += nx;
    }
  }
}

// 其余数据类型通用接口
static const TValue *getgeneric (Table *t, const TValue *key, int deadok) {
  Node *n = mainpositionTV(t, key);
  for (;;) {  /* check whether 'key' is somewhere in the chain */
    if (equalkey(key, n, deadok))
      return gval(n);  /* that's it */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return &absentkey;  /* not found */
      n += nx;
    }
  }
}
```
> 在对浮点型处理时，根据 mode 可以使用不同的 floor 函数，可以参考 [floor, floorf, floorl](https://en.cppreference.com/w/c/numeric/math/floor)  

- Node.i_val 和 table.array
这二者的赋值，都可以在 luaH_setint 的时候找到例子，但前者更为广泛  
仔细看 Node 其实是个 union，而 i_val 又是 TValue 类型变量，因为内存共用  
修改 i_val 的值实际上就是修改 Node.u 中的 TValuefield，这也是数据结构上设计的巧思  
``` c
void luaH_setint (lua_State *L, Table *t, lua_Integer key, TValue *value) {
  const TValue *p = luaH_getint(t, key); // 找不到时返回 absentKey，否则返回 array的位置 或者 hashtable里的i_val
  TValue *cell;
  if (!isabstkey(p)) // 数组和哈希表任意一个里面找到了
    cell = cast(TValue *, p); // 解const，准备修改节点的 TValuefields
  else { // 找不到这个 int 的节点
    TValue k; // 构造一个TV准备用于插入新节点
    setivalue(&k, key);
    cell = luaH_newkey(L, t, &k); // 记录返回的 i_val，此时cell指向对应Node的 i_val
  }
  setobj2t(L, cell, value); // cell->value_ = value->value_; cell->tt_ = value->tt_; 修改了i_val的内容
}
```
通过设置数组和哈希表，均摊了部分整数到数组中，降低了哈希冲突实际上也提高了效率  
数组大小范围内的整数key都会保存到array，范围外的就进入哈希表  
那么当他们大小改变了，会发生什么呢？接下来看看 hash resize 的做法  

- 表的扩容缩容
``` c
void luaH_resize (lua_State *L, Table *t, unsigned int newasize, unsigned int nhsize) {
  unsigned int i; // 用于数组遍历，记录数组大小
  Table newt;  /* to keep the new hash part */ 新的表
  unsigned int oldasize = setlimittosize(t); // 将alimit设为实际大小，并更改flags位使元方法 isrealasize 返回为真 
  TValue *newarray; // 新的array数组
  /* create new hash part with appropriate size into 'newt' */
  setnodevector(L, &newt, nhsize); // luaH_new 的最后一步，初始化表的字段，设置哈希表大小，区别是这里没有加入全局的gc管理
  
  // 缩容时
  if (newasize < oldasize) {  /* will array shrink? */
    t->alimit = newasize;  /* pretend array has new size... */ 假装旧array已经是新的长度了（变小）
    exchangehashpart(t, &newt);  /* and new hash */ 将 t 和 newt 的 lsizenode node lastfree 交换（方便失败时还原）
    /* re-insert into the new hash the elements from vanishing slice */
    // 对于缩容的部分，因为alimit已经改了，所以这里调用 luaH_setint 会把超长的部分转移到哈希表里
    for (i = newasize; i < oldasize; i++) {
      if (!isempty(&t->array[i])) 
        luaH_setint(L, t, i + 1, &t->array[i]);
    }
    t->alimit = oldasize;  /* restore current size... */ 把alimit还原回去
    exchangehashpart(t, &newt);  /* and hash (in case of errors) */ 再把哈希部分的字段交换回来，这时新的哈希表
  }
  
  // 为新array数组分配空间，这里可能会失败，要对异常处理
  newarray = luaM_reallocvector(L, t->array, oldasize, newasize, TValue); 
  if (unlikely(newarray == NULL && newasize > 0)) {  /* allocation failed? */
    freehash(L, &newt);  /* release new hash part */ 释放新表
    luaM_error(L);  /* raise error (with array unchanged) */ 抛出 ERROR，可能在这里就abort了（no handle时）
  }
  
  // 新array数组分配完成，要对其赋值了（刚刚只操作了哈希表没操作array）
  exchangehashpart(t, &newt);  /* 't' has the new hash ('newt' has the old) */ 再次进行哈希部分字段交换，此时新的哈希表在 t 这里
  t->array = newarray;  /* set new array part */  因为是 realloc 所以直接指向新数组即可
  t->alimit = newasize; // 更改alimit，此时不是假装了是真的变了，且 isrealsize(t) 为 true
  for (i = oldasize; i < newasize; i++)  /* clear new slice of the array */ 缩容其实啥也不做
     setempty(&t->array[i]);	// 扩容时将多出来的部分初始化为 LUA_VEMPTY
  /* re-insert elements from old hash part into new parts */
  reinsert(L, &newt, t);  /* 'newt' now has the old hash */ 将旧哈希表中的Node设进新哈希表
  freehash(L, &newt);  /* free old hash part */
}

static void reinsert (lua_State *L, Table *ot, Table *t) {
  int j;
  int size = sizenode(ot);
  for (j = 0; j < size; j++) {
    Node *old = gnode(ot, j);
    if (!isempty(gval(old))) { // i_val非LUA_VEMPTY（初始化节点）
      /* doesn't need barrier/invalidate cache, as entry was already present in the table */
      TValue k;
      getnodekey(L, &k, old); // k->value_ = old->u.key_val k->tt_ = old->u.key_tt
      setobjt2t(L, luaH_set(L, t, &k), gval(old)); // 在t中根据k(old Key)重新获取节点位置，将i_val赋值过去
    }
  }
}

void luaH_resizearray (lua_State *L, Table *t, unsigned int nasize) {
  int nsize = allocsizenode(t);
  luaH_resize(L, t, nasize, nsize);
}
```
我们观察表对数组部分缩容的操作，是十分高效的  
因为他没有任何的内存拷贝，只是单纯把指针swap了一下  
就把 t 的 array 超长部分转移到了 newt 的 node 里面

- 表的扩容缩容条件 与 表size计算  
	+ 哈希表扩容  
		- luaH_newkey 获取 lastfree 发现已经指向 NULL 时  
	+ 哈希表缩容  
		- 哈希表内有大量int类型key被转移进array里  
	+ 数组扩容
		- 哈希表内有大量int类型key被转移进array里
	+ 数组缩容
		- 暂未发现主动缩容情况
``` c
#define MAXABITS	cast_int(sizeof(int) * CHAR_BIT - 1) // CHAR_BIT是每个字节的位数，通常1字节8位，但某些os可能不一样，具体见limit.h
#define MAXASIZE	luaM_limitN(1u << MAXABITS, TValue)

// 统计数组部分中的数值
static unsigned int numusearray (const Table *t, unsigned int *nums) {
  int lg; // 2的lg次幂用位表示，对应MAXABITS第几位，以0开始，对应nums[0]
  unsigned int ttlg;  /* 2^lg */ 2的lg次幂用值表示，每次乘2，以2的0次幂，即1开始
  unsigned int ause = 0;  /* summation of 'nums' */ 数组总元素个数
  unsigned int i = 1;  /* count to traverse all array keys */
  unsigned int asize = limitasasize(t);  /* real array size */ 当前的t->alimit
  /* traverse each slice */ MAXBITS有做-1操作，所以这里遍历是 <= MAXBITS
  
  /* 
  第一轮 遍历 t->array[0 ~ 0] 中非 LUA_VEMPTY，记录到 nums[0]，最多1个;
  第二轮 遍历 t->array[1 ~ 1] 中非 LUA_VEMPTY，记录到 nums[1]，最多1个;
  第三轮 遍历 t->array[2 ~ 4] 中非 LUA_VEMPTY，记录到 nums[2]，最多3个;
  第四轮 遍历 t->array[5 ~ 8] 中非 LUA_VEMPTY，记录到 nums[3]，最多4个;
  第五轮 遍历 t->array[9 ~ 16] 中非 LUA_VEMPTY，记录到 nums[4]，最多8个;
  第六轮 遍历 t->array[17 ~ 32] 中非 LUA_VEMPTY，记录到 nums[5]，最多16个;
  第七轮 遍历 t->array[33 ~ 64] 中非 LUA_VEMPTY，记录到 nums[6]，最多32个;
  ...
  */
  for (lg = 0, ttlg = 1; lg <= MAXABITS; lg++, ttlg *= 2) {
    unsigned int lc = 0;  /* counter */
    unsigned int lim = ttlg;
    if (lim > asize) { // 2^lg 大于 t->alimit 说明此时已经是最后一轮，可以回看alimit的定义结合理解
      lim = asize;  /* adjust upper limit */ 减少遍历次数，(lim,asize]的值此时都是初始值，无意义
      if (i > lim) // 发现游标已经过了遍历上限了，直接跳出循环
        break;  /* no more elements to count */
    }
    /* count elements in range (2^(lg - 1), 2^lg] */
    for (; i <= lim; i++) {
      if (!isempty(&t->array[i-1]))
        lc++;
    }
    nums[lg] += lc;
    ause += lc;
  }
  return ause;
}

// 哈希表部分
static unsigned int arrayindex (lua_Integer k) {
  if (l_castS2U(k) - 1u < MAXASIZE)  /* 'k' in [1, MAXASIZE]? */
    return cast_uint(k);  /* 'key' is an appropriate array index */
  else
    return 0;
}

static int countint (lua_Integer key, unsigned int *nums) {
  unsigned int k = arrayindex(key); // key数值在array范围内
  if (k != 0) {  /* is 'key' an appropriate array index? */
    nums[luaO_ceillog2(k)]++;  /* count as such */ 对cast_uint后的key对2求对数，向上取整
    return 1; // 返回1，表示成功
  }
  else
    return 0;
}

static int numusehash (const Table *t, unsigned int *nums, unsigned int *pna) {
  int totaluse = 0;  /* total number of elements */ 哈希表元素总值
  int ause = 0;  /* elements added to 'nums' (can go to array part) */ 哈希表中可以放到t->array的元素个数
  int i = sizenode(t); // t->lsizenode
  while (i--) {
    Node *n = &t->node[i];
    if (!isempty(gval(n))) {
      if (keyisinteger(n))
        ause += countint(keyival(n), nums); //根据key值大小是否在[1, MAXASIZE]，判断这个值能否放nums数组，能就放进去且计数+1
      totaluse++;
    }
  }
  *pna += ause; // 更新外层的na值
  return totaluse;
}

static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
  int i; // 对应nums[i]，数组下标，默认0
  unsigned int twotoi;  /* 2^i (candidate for optimal size) */ 2的i次幂，每次乘2指数自增，默认1
  unsigned int a = 0;  /* number of elements smaller than 2^i */ 
  unsigned int na = 0;  /* number of elements to go to array part */
  unsigned int optimal = 0;  /* optimal size for array part */
  /* loop while keys can fill more than half of total size */
  for (i = 0, twotoi = 1;
       twotoi > 0 && *pna > twotoi / 2; // 数组总元素个数大于当前 2^(i-1) 但小于 2^i，回看alimit定义
       i++, twotoi *= 2) {
    a += nums[i];
    if (a > twotoi/2) {  /* more than half elements present? */
      optimal = twotoi;  /* optimal size (till now) */ 最理想的大小
      na = a;  /* all elements up to 'optimal' will go to array part */
    }
  }
  /* na = 8
  假设有 nums[7] = { 0, 0, 0, 0, 4, 4, 0, 0 }
  对应的 twotoi 为： 1  2  4  8  16 32 64 128
  对应的 twotoi/2为：0  1  2  4  8  16 32 64
  对应的 a 为：      0  0  0  0  4  8  8  8
  当 i=4 时，因为 a(4) <= twotoi/2(8)，optimal值未能更新，后续也不可能更新
  此时alimit的值就会被改为0，后续在 luaH_setint 的时候，数值都会被放到哈希表直至下次rehash
  这样子的设计，应该是考虑到了 array 数组的空间分配出来得充分利用（至少得用到一半），否则干脆不用array
  毕竟，在定义 local t = { [100000] = 1 } 的时候，声明一个巨大的 array 是十分浪费的行为
  
  而如果将例子稍微修改 na = 13
  nums[7] = { 0, 0, 1, 4, 4, 4, 0, 0 }
  twotoi/2为：0  1  2  4  8  16 32 64
  a 为：      0  0  1  5  9  13 13 13
  当 i=4 时，因为 a(9) > twotoi/2(8)，optimal = 16，na = 9（少了的部分[13-9]就去哈希表吧）
  当 i=5 时，因为 a(13) > twotoi/2(16)，后续也不可能更新
  于是 alimit 可以修改为 4，数组的过半元素都可以被赋非空值
  */
  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
  *pna = na;
  return optimal;
}

static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */ 假设sizeof(int)为4，1字节8位，那么这个值是31(32-1)
  setlimittosize(t); // 更新alimit值为数组实际大小
  na = numusearray(t, nums);  /* count keys in array part */ 数组非空元素总和
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */ 表元素总和
  /* count extra key */ ek为触发此次rehash操作的元素，还未在表内，单独计算
  if (ttisinteger(ek))
    na += countint(ivalue(ek), nums);
  totaluse++;
  /* compute new size for array part */
  asize = computesizes(nums, &na); // 算出新的alimit，然后更新na
  /* resize the table to new computed sizes */
  luaH_resize(L, t, asize, totaluse - na); // newhashsize = totaluse - na
}
```
- 表的遍历（迭代）  
区别于传统迭代器的模式，因为表这个容器实际上由 数组 和 哈希表 两部分组成  
所以它的迭代是用 Key 来实现的，通过循环调用 luaH_next 实现对表的遍历  
``` c
#define s2v(o)	(&(o)->val)
typedef union StackValue {
  TValue val;
} StackValue;
typedef StackValue *StkId;

static unsigned int findindex (lua_State *L, Table *t, TValue *key, unsigned int asize) {
  unsigned int i;
  if (ttisnil(key)) return 0;  /* first iteration */
  i = ttisinteger(key) ? arrayindex(ivalue(key)) : 0;
  if (i - 1u < asize)  /* is 'key' inside array part? */
    return i;  /* yes; that's the index */ 
  else {
    const TValue *n = getgeneric(t, key, 1); // 获取key在哈希表内对应的值
    if (unlikely(isabstkey(n)))
      luaG_runerror(L, "invalid key to 'next'");  /* key not found */
    i = cast_int(nodefromval(n) - gnode(t, 0));  /* key index in hash table */ 获取这个值对应Node 和 t->node[0] 的差值，就是这个节点的下标
    /* hash elements are numbered after array ones */
    return (i + 1) + asize; // 加1就是下一个节点的下标，即next
  }
}

int luaH_next (lua_State *L, Table *t, StkId key) {
  unsigned int asize = luaH_realasize(t); // 获取数组部分长度
  unsigned int i = findindex(L, t, s2v(key), asize);  /* find original key */
  // 如果 i 在数组部分
  for (; i < asize; i++) {  /* try first array part */
    if (!isempty(&t->array[i])) {  /* a non-empty entry? */
      setivalue(s2v(key), i + 1); // 更新栈内key的value_=i+1 和 tt_=LUA_VNUMINT
      setobj2s(L, key + 1, &t->array[i]); // 值压栈到 key+1 的位置 
      return 1;
    }
  }
  // 如果 i 不在数组部分，i肯定是大于 asize 的（见findIndex else返回）故上一个for不执行
  for (i -= asize; cast_int(i) < sizenode(t); i++) {  /* hash part */ 开始就得先把 asize 减掉获取正确哈希表下标
    if (!isempty(gval(gnode(t, i)))) {  /* a non-empty entry? */
      Node *n = gnode(t, i);
      getnodekey(L, s2v(key), n); // 更新栈内key的value_和tt_为Node n对应值
      setobj2s(L, key + 1, gval(n)); // 值压栈到 key+1 的位置 
      return 1;
    }
  }
  return 0;  /* no more elements */
}
```

- 表的长度获取
	- 如果 t[limit] 是空，边界很可能在前面
		+ 如果 limit-1 有值，那他就是边界（特例）
		+ 否则，从 [0,limit] 调用 binsearch 二分搜索边界
	- 如果 t[limit] 非空，数组后面可能还有很多值
		+ 如果 limit+1 无值，那 limit 就是边界（特例）
		+ 否则，从 [limit,realsize] 调用 binsearch 二分搜索边界
	- 如果 limit 为0，且非空表，表示数值元素被存放在哈希表而非数组（前面算optimal的例子）
		+ 调用 hash_search 搜索边界（也是二分思想）  

核心思想：通过二分搜素，找到“边界”，即找到最后一个有值的下标 和 第一个无值的下标  
> 这块算法反而是比较复杂的部分，算法考虑很多特例情况来加快运行  
> 代价是可读性特别差，所以注释特别多，我就不瞎翻译了（其实是懒）  
``` c
lua_Unsigned luaH_getn (Table *t) {
  unsigned int limit = t->alimit;
  if (limit > 0 && isempty(&t->array[limit - 1])) {  /* (1)? */
    /* there must be a boundary before 'limit' */
    if (limit >= 2 && !isempty(&t->array[limit - 2])) {
      /* 'limit - 1' is a boundary; can it be a new limit? */
      if (ispow2realasize(t) && !ispow2(limit - 1)) {
        t->alimit = limit - 1;
        setnorealasize(t);  /* now 'alimit' is not the real size */
      }
      return limit - 1;
    }
    else {  /* must search for a boundary in [0, limit] */
      unsigned int boundary = binsearch(t->array, 0, limit);
      /* can this boundary represent the real size of the array? */
      if (ispow2realasize(t) && boundary > luaH_realasize(t) / 2) {
        t->alimit = boundary;  /* use it as the new limit */
        setnorealasize(t);
      }
      return boundary;
    }
  }
  /* 'limit' is zero or present in table */
  if (!limitequalsasize(t)) {  /* (2)? */
    /* 'limit' > 0 and array has more elements after 'limit' */
    if (isempty(&t->array[limit]))  /* 'limit + 1' is empty? */
      return limit;  /* this is the boundary */
    /* else, try last element in the array */
    limit = luaH_realasize(t);
    if (isempty(&t->array[limit - 1])) {  /* empty? */
      /* there must be a boundary in the array after old limit,
         and it must be a valid new limit */
      unsigned int boundary = binsearch(t->array, t->alimit, limit);
      t->alimit = boundary;
      return boundary;
    }
    /* else, new limit is present in the table; check the hash part */
  }
  /* (3) 'limit' is the last element and either is zero or present in table */
  lua_assert(limit == luaH_realasize(t) &&
             (limit == 0 || !isempty(&t->array[limit - 1])));
  if (isdummy(t) || isempty(luaH_getint(t, cast(lua_Integer, limit + 1))))
    return limit;  /* 'limit + 1' is absent */
  else  /* 'limit + 1' is also present */
    return hash_search(t, limit);
}
```
# Chap 5. Lua虚拟机
> 脚本语言的虚拟机扮演了一个中间层的角色，作为底层操作系统的上层抽象  
> 对上而言，它负责解释执行字节码；对下而言，它屏蔽了平台相关的内容，使得脚本代码跨平台运行  

- 虚拟机（线程）的数据结构
``` c
struct lua_State {
  CommonHeader;
  lu_byte status;	// 线程状态
  lu_byte allowhook;	// 是否允许挂hook
  unsigned short nci;  /* number of items in 'ci' list */ CallInfo的数量
  StkId top;  /* first free slot in the stack */  栈顶
  global_State *l_G; 	// 全局虚拟机指针
  CallInfo *ci;  /* call info for current function */ 当前函数调用信息
  StkId stack_last;  /* end of stack (last element + 1) */ 栈底的
  StkId stack;  /* stack base */ Lua虚拟栈
  UpVal *openupval;  /* list of open upvalues in this stack */
  GCObject *gclist;
  struct lua_State *twups;  /* list of threads with open upvalues */
  struct lua_longjmp *errorJmp;  /* current error recover point */
  CallInfo base_ci;  /* CallInfo for first level (C calling Lua) */
  volatile lua_Hook hook;
  ptrdiff_t errfunc;  /* current error handling function (stack index) */
  l_uint32 nCcalls;  /* number of nested (non-yieldable | C)  calls */
  int oldpc;  /* last pc traced */
  int basehookcount;
  int hookcount;
  volatile l_signalT hookmask;
};
```

# Chap 6. 指令的解析与执行[略]

# Chap 7. GC算法

# Chap 8. 环境与模块

# Chap 9. 调试器工作原理

# Chap 10. 异常处理

# Chap 11. 协程























