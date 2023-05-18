> 本书描述的 v5.1.4 的 Lua 源码，而笔者使用的是 v5.4.2，内容虽大相径庭，笔者会根据自己理解阐述总结  
> Lua 的源码设计思路清晰，结构分明，是不可多得的短小精悍的源代码，只能说相见恨晚  
# Chap 1. 概述
- 名字含义  
葡萄牙语中 SOL（simple object language，Lua的前身）的含义是太阳，Lua 的意思是月亮，可能这也是 Lua icon 样式的由来 🌕🌔  
- Lua目录结构（v5.4.2）  
```
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
```
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
```
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
```
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
```
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
```
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
```
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
```
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
```
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
  GCObject *gclist;
} Table;
```
> About 'alimit': if 'isrealasize(t)' is true, then 'alimit' is the real size of 'array'.   
> Otherwise, the real size of 'array' is the smallest power of two not smaller than 'alimit' (or zero iff 'alimit'is zero);   
> 'alimit' is then used as a hint for #t.  
> iff：当且仅当。翻译一下，就是当 isrealasize 返回 true 时alimit是array的真实大小  
> 否则 array 的大小是不小于 alimit 的最小的 2<sup>n</sup>，当且仅当 alimit=0 时，array大小是0  
> 比如说alimit=7，那大小其实就是8，alimit=25，大小其实就是32，保证数组大小是 pow(2) 是有益的  

短短几行定义，就可以实现那么复杂的功能，不得不说招牌菜就是香，









