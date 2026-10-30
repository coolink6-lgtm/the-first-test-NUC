# C 语言高阶面试题（10 道 · 大厂三面难度）

> **难度定位**：对标字节跳动、阿里、腾讯、华为、Intel、NVIDIA 等公司 C/C++ 系统方向（基础架构、嵌入式、编译器/运行时、内核、高性能网络）三面及以上的深度。
> 不考八股定义，考的是：**标准条款 + 编译器/ABI 真实行为 + 工程取舍 + 手写代码**。
>
> **默认环境**：x86-64（LP64，SysV AMD64 ABI）+ GCC/Clang，语言标准 C11/C17。
> 涉及实现定义/ABI 相关的地方，答题时必须明确指出「这是标准未规定的」，并给出 32 位、Windows LLP64 的差异。
>
> **使用建议**：先限时 90 分钟独立作答（每题都要写出可编译的代码或明确结论），再看 `answer.md` 对照。

---

## 第 1 题：未定义行为、求值顺序与优化器 ★★★★☆

### 背景

面试官给你一段「看起来能跑」的代码，让你判断哪些行是未定义行为（UB），并解释为什么 `-O0` 和 `-O2` 结果不同。

### 代码

```c
#include <stdio.h>

static int g = 0;
static int f(void) { return ++g; }

int main(void)
{
    int i = 0;
    int a = i++ + i++;          /* ① */

    int b = (i = 3) + (i = 4);  /* ② */

    int arr[2] = {0, 0};
    int k = 0;
    arr[k] = k++;               /* ③ */

    printf("%d %d %d %d\n", a, b, arr[0], k);

    int x = 1;
    x = x++ + 1;                /* ④ */
    printf("%d\n", x);

    printf("%d %d\n", f(), f());/* ⑤ */

    int y = 1;
    y = (y++, y + 1);           /* ⑥ */
    printf("%d\n", y);
    return 0;
}
```

### 问题

1. ①～⑥ 逐条判定：**未定义行为 / 未指定行为 / 实现定义行为 / 完全良定义**，并给出 C11/C17 的具体条款依据（章节号）。
2. 为什么 ⑤ 中两个函数都修改同一个全局变量就是 UB，而 `printf("%d %d", i, i)` 只是「未指定」？
3. 用优化器的视角解释：为什么 `-O2` 下 ① 可能被直接折叠成常数，甚至整段代码被删掉？请说明编译器「利用 UB 做假设」的具体机制。
4. 给出 3 条可落地的工程规范来规避这类问题；并回答：`-fsanitize=undefined` 能否 100% 捕获上述 UB？为什么？
5. 附加：④ 和 ⑥ 只差一个逗号表达式，为什么一个 UB、一个良定义？如果把 ④ 改成 `x = (x++, x + 1);`，结果是多少？

### 考察点

序列点/sequenced-before、副作用（side effect）与值计算（value computation）、函数实参求值顺序、UB 的「时间旅行」性质、Sanitizer 的能力边界。

---

## 第 2 题：整型提升、隐式转换与溢出 ★★★★☆

### 背景

一段在 32 位机器上「正常」的代码，移植到 64 位后行为改变；另一段代码在 `-O2` 下被优化成了死循环。

### 代码

```c
#include <limits.h>
#include <stdio.h>

int main(void)
{
    int a = -1;
    unsigned b = 1;
    printf("%d\n", a < b);              /* ① */
    printf("%d\n", a < sizeof(int));    /* ② */

    for (unsigned i = 10; i >= 0; --i)  /* ③ */
        ;

    char c = 200;                       /* ④ */
    printf("%d\n", c);

    int x = INT_MIN;
    printf("%d\n", -x);                 /* ⑤ */
    printf("%d\n", x / -1);             /* ⑤ */

    unsigned u = 1u << 31;              /* ⑥ */
    int s = 1 << 31;                    /* ⑥ */

    int ch;
    while ((ch = getchar()) != EOF) {   /* ⑦ */
        char cc = ch;
        (void)cc;
    }

    size_t n = 10;
    int m = n;                          /* ⑧ */
    printf("%d\n", m);
    return 0;
}
```

### 问题

1. ①～⑧ 逐条给出在 x86-64 Linux（LP64）下的结果或行为，并说明依据：整型提升（integer promotions）、寻常算术转换（usual arithmetic conversions）、转换等级。
2. ② 中 `-1 < sizeof(int)` 为什么是假？在 32 位（ILP32）和 64 位（LP64）下分别是什么？如果写成 `-1 < (int)sizeof(int)` 呢？
3. ③ 为什么会死循环？给出 4 种正确写法，并比较它们的可读性与边界行为（包括 `i` 为 0 时）。
4. ④ 中 `char` 到底有没有符号？如何写出可移植的「字节」代码？⑦ 中 `getchar()` 为什么返回 `int` 而不是 `char`？如果把 `ch` 声明成 `char` 会发生什么？
5. ⑤ 为什么是 UB 而不是「得到 INT_MIN」？⑥ 中两行的区别是什么？`1u << 31`、`1 << 31`、`1LL << 31` 分别是否合法？
6. 写两个函数（不允许依赖 UB）：
   - `bool mul_ovf(int a, int b, int *out);` 检测有符号乘法溢出；
   - `bool size_to_int(size_t n, int *out);` 带检查的窄化转换。
   并说明 `__builtin_mul_overflow` 在汇编层通常如何实现。
7. ⑧ 在 64 位下 `size_t` 是 64 位、`int` 是 32 位，窄化转换的规则是什么？为什么 `printf("%d", sizeof(int))` 是 UB？给出 `%zu`、`%ld`、`%lld`、`PRIu64` 的正确使用场景，并说明 Windows LLP64 下 `long` 的坑。

### 考察点

6.3.1.1 / 6.3.1.8、无符号回绕、有符号溢出 UB、移位规则、字符类型的符号性、`size_t` 与格式串匹配、跨平台数据模型（LP64/LLP64/ILP32）。

---

## 第 3 题：结构体布局、位域与柔性数组 ★★★★★

### 背景

你要为一个网络协议设计内存结构，同时要兼容内核风格的 `container_of`。面试官要求你精确说出每个字节。

### 代码

```c
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>

struct S1 { char c; int i; short s; };
struct S2 { char c; short s; int i; };
struct S3 { char c; double d; char e; };
struct S4 { unsigned a:4; unsigned b:5; signed c:7; };
struct S5 { uint8_t len; uint8_t data[]; };
union  U  { uint32_t u; float f; uint8_t b[4]; };
```

### 问题

1. 在 x86-64 SysV ABI 下，给出 `S1`、`S2`、`S3` 的 `sizeof`、`_Alignof` 及每个成员的 `offsetof`，并写出计算过程。为什么把 `S1` 的成员重排成 `S2` 就能省 4 字节？举一个「重排也无法再省」的例子。
2. `S4` 的 `sizeof` 是多少？`&s.a` 合法吗？`offsetof(struct S4, a)` 合法吗？位域的分配单元、跨单元、符号性、字节序依赖分别由谁规定？`int b:1` 能存哪些值？`if (s.c < 0)` 这类比较有什么陷阱？
3. `S5` 的 `sizeof` 是多少？`offsetof(struct S5, data)` 是多少？如何正确分配？为什么柔性数组成员不能作为数组元素、不能初始化、不能出现在 `union` 里？`uint8_t data[]` 与 `uint8_t data[1]`、`uint8_t data[0]` 有何区别？`malloc(sizeof(struct S5) + n)` 是精确的还是会多分配？
4. 写出 `offsetof` 与 `container_of` 的宏实现，并回答：
   - `offsetof` 的经典实现 `((size_t)&((T*)0)->m)` 是否 UB？为什么全世界都在用？
   - `container_of` 里的指针减法是否越界？如何用 `uintptr_t` 规避指针运算的 UB？这样做的代价是什么？
   - 如何用 `_Static_assert` + `__builtin_types_compatible_p` 给 `container_of` 加类型检查？
5. 讨论 `#pragma pack(1)` / `__attribute__((packed))` 的代价：非对齐访问在 x86 上「能跑」但在 ARM 上可能 SIGBUS；与 `_Atomic`、DMA、缓存行伪共享的关系；网络协议解析时如何既保证布局又安全访问（`memcpy` 到本地变量）。
6. 附加：`union U` 通过 `u.f` 写、`u.u` 读，是类型双关（type punning）还是 UB？C11 与 C++ 的规则有何不同？如何可移植地查看浮点数的二进制表示？

### 考察点

对齐与填充规则、位域的实现定义行为、柔性数组（FAM）、`offsetof`/`container_of` 的 UB 边界、packed 的代价、类型双关。

---

## 第 4 题：指针、数组与复杂声明符 ★★★★☆

### 背景

面试官递给你一张纸，上面只有几行声明和一个 `sizeof` 输出，要求你「用人话解释每个标识符到底是什么」。

### 代码

```c
int a[5] = {1, 2, 3, 4, 5};
int (*p)[5] = &a;
int *q = a;

int m[3][4];

char *(*(*x[3])(void))[5];
int  (*(*f)(int))[10];

void g(int arr[10]);
```

### 问题

1. 在 x86-64 下，`sizeof(a)`、`sizeof(p)`、`sizeof(q)`、`sizeof(*p)` 分别是多少？`a`、`&a`、`&a[0]` 的类型和值分别是什么？`a + 1` 与 `&a + 1` 的字节偏移各是多少？为什么 `p + 1` 和 `q + 1` 的步长不同？
2. 把 `a` 传给 `void g(int arr[10])` 后，函数内 `sizeof(arr)` 是多少？为什么？C 标准在哪一条规定了「数组形参退化为指针」？`int arr[static 10]` 又是什么意思？
3. `int m[3][4]` 中，`m`、`m[0]`、`&m[0][0]` 的类型分别是什么？为什么不能把 `m` 直接传给 `int **`？如果函数签名写成 `void h(int **m)`，调用 `h(m)` 会发生什么？给出至少 3 种正确写法（含 VLA 版本）。
4. 用「右左法则」解析 `char *(*(*x[3])(void))[5];` 和 `int (*(*f)(int))[10];`，并用 `typedef` 改写，要求改写后可读性显著提升。
5. `void *` 能做算术运算吗？`void *vp; vp + 1;` 在 GCC 下是什么行为？为什么标准禁止？如何可移植地做字节级偏移？
6. `const char * const *p`、`char * const *p`、`const char **p` 分别表示什么？`int *const p` 与 `const int *p` 的区别？`restrict` 修饰指针时，编译器能做什么优化？违反 `restrict` 契约的后果是什么？
7. 写一个函数指针表驱动的命令分发器：`typedef struct { const char *name; int (*handler)(int argc, char **argv); } cmd_t;`，要求支持前缀匹配、参数校验，并说明为什么函数指针表比 `if/else if` 链更好（以及不好在哪）。

### 考察点

数组退化、指针步长、多维数组的内存模型、声明符解析、`const` 的层次、`restrict` 契约、函数指针。

---

## 第 5 题：动态内存、对齐与 strict aliasing ★★★★★

### 背景

你要实现一个「只依赖 `malloc/free`」的通用对齐分配器，并解释为什么有些「能跑」的内存操作其实是 UB。

### 代码

```c
#include <stdlib.h>
#include <string.h>

void demo(void)
{
    char *p = malloc(0);            /* ① */
    free(p);

    char *buf = malloc(16);
    free(buf + 4);                  /* ② */

    char *q = malloc(16);
    q = realloc(q, 1024);           /* ③ */
    if (!q) { /* ... */ }

    int i = 0x3f800000;
    float f = *(float *)&i;         /* ④ */

    unsigned char raw[4];
    memcpy(raw, &i, 4);
    float g;
    memcpy(&g, raw, 4);             /* ⑤ */
}
```

### 问题

1. ① `malloc(0)` 的返回值有哪几种可能？必须 `free` 吗？② `free(buf + 4)` 为什么是 UB？glibc 会怎么死？③ `realloc` 失败时原指针还有效吗？`q = realloc(q, ...)` 的写法错在哪？`realloc(p, 0)` 的行为是什么？
2. `calloc(nmemb, size)` 内部如何做溢出检查？写一个 `void *xcalloc(size_t n, size_t size)`，要求检测 `n * size` 溢出且不依赖 UB。
3. `malloc` 返回的指针对齐到多少？`max_align_t` 是什么？为什么 `malloc` 的对齐保证对 `__m256`、`long double`（x86 下 16 字节）、`_Atomic` 类型可能不够？C11 的 `aligned_alloc` 和 POSIX 的 `posix_memalign` 有何区别（尤其是 size 必须是 alignment 的整数倍这一条）？
4. 只用 `malloc/free` 实现：
   ```c
   void *aligned_alloc_impl(size_t alignment, size_t size);
   void  aligned_free_impl(void *ptr);
   ```
   要求：`alignment` 为 2 的幂；支持 `alignment < sizeof(void *)`；`size` 任意；不得溢出；`aligned_free_impl` 能正确归还原始指针。写出完整代码并解释每个边界条件。
5. ④ 为什么是 UB？C11 的「有效类型（effective type）」规则（6.5p6、6.5p7）怎么描述 `malloc` 出来的内存？⑤ 为什么是合法的？除了 `memcpy`，还有哪些写法（`union`、`may_alias` 属性）？各自的移植性如何？
6. 讨论：`alloca` 与 VLA 的风险；`free` 后使用与 double free 的检测手段（ASan、Valgrind、`MALLOC_CHECK_`、glibc 的 tcache 检测）；为什么「崩溃在现场」不等于「bug 在现场」。

### 考察点

分配器契约、对齐、`realloc` 语义、有效类型/严格别名、溢出安全、调试工具链。

---

## 第 6 题：memcpy / memmove 与字符串库函数 ★★★★★

### 背景

让你手写 `memcpy`、`memmove`、`strlen`，并且要求「比朴素版本快一个数量级，同时不触发 UB」。

### 问题

1. 为什么 `memcpy` 在源/目的区间重叠时是 UB，而 `memmove` 不是？画出 `dst` 落在 `[src, src+n)` 内、`src` 落在 `[dst, dst+n)` 内两种情形，说明 `memmove` 应该在什么时候从后向前拷贝。写出完整实现。
2. 实现 `void *my_memcpy(void *dst, const void *src, size_t n)`：
   - 朴素字节循环为什么慢？
   - 按 8 字节机器字拷贝时，把 `char *` 强转成 `uint64_t *` 有哪些 UB？（提示：对齐、严格别名、越界读）
   - 如何写出既快又合法的版本？请给出「头部对齐 + 主体按字 + 尾部」的实现，并说明 `memcpy(&v, p, 8)` 这种「用 memcpy 读一个 word」为什么不会被优化拖慢。
3. Duff's device：
   ```c
   void duff_copy(char *to, const char *from, size_t count);
   ```
   解释 `switch` 为什么能跳进 `do { } while` 的循环体中间（这在标准里合法吗？），写出实现，并指出 `count == 0` 时的经典 bug 及修复。
4. `strncpy`、`strncat`、`strlcpy` 各有什么坑？`strncpy(dst, src, n)` 在 `strlen(src) >= n` 时会怎样？`strncat` 的 `n` 到底是「源的长度」还是「剩余空间」？
5. 写一个字长优化的 `my_strlen`，解释 `haszero` 技巧：
   ```c
   #define HASZERO(x) (((x) - 0x01010101UL) & ~(x) & 0x80808080UL)
   ```
   它为什么能检测「某个字节为 0」？这种实现有没有越界读的风险？ASan 会不会报错？
6. `memcmp` 与 `strcmp` 的返回值语义有何不同？为什么 `if (strcmp(a, b))` 表示「不相等」？`strtok` 为什么不可重入？`strtok_r` 如何解决？
7. `memcpy` 原型中的 `restrict` 意味着什么？如果调用者传了重叠区间，编译器「有权」做什么？举一个在 `-O2` 下真实会出错的例子。

### 考察点

重叠语义、字长优化、对齐与别名、Duff's device、C 标准库的坑、`restrict` 契约。

---

## 第 7 题：链接属性、inline 与多文件工程 ★★★★☆

### 背景

一个三人小组的 C 项目，头文件里放了「定义」，在 `-O0` 下链接失败、在 `-O2` 下又能过；换台机器又报 `multiple definition`。

### 代码

```c
/* a.h */
int counter;                                  /* ① */
inline int add(int a, int b) { return a + b; }/* ② */
static int helper(void) { return 42; }        /* ③ */
const int LIMIT = 100;                        /* ④ */

/* a.c */
#include "a.h"
int counter = 0;                              /* ⑤ */

/* b.c */
#include "a.h"
int use(void) { return add(1, 2) + helper() + LIMIT; }
```

构建命令：

```sh
gcc -std=c17 -O0 a.c b.c -o app
gcc -std=c17 -O2 a.c b.c -o app
```

### 问题

1. ② 在 C99/C11 下的 `inline` 语义是什么？为什么 `-O0` 下会报 `undefined reference to 'add'`，而 `-O2` 下又能链接成功？这算不算「海森堡 bug」？
2. 给出 3 种正确写法，并说明各自的代码膨胀/链接行为：`static inline`、`inline` + 一个 TU 提供外部定义、`extern inline`。C99 的 `extern inline` 与 GNU89 的 `inline` 语义为什么相反？GCC 的 `-fgnu89-inline` 是干什么的？
3. ① 和 ⑤ 同时存在会发生什么？什么是 tentative definition（暂定定义）？为什么 GCC 10 之后默认 `-fno-common` 会让老项目突然链接失败？如何临时规避（`-fcommon`）以及为什么不应该长期这么做？
4. ④ 为什么在 C 中会导致 `multiple definition`，而在 C++ 中不会？C 和 C++ 对文件作用域 `const` 的链接属性规定有何不同？
5. `static` 有哪三种完全不同的含义？③ 放在头文件里有什么后果（代码膨胀、调试符号、每 TU 一份）？`static` 函数能被外部调用吗？能被内联吗？
6. 静态库的链接顺序为什么重要？`libfoo.a` 依赖 `libbar.a` 时，`-lfoo -lbar` 与 `-lbar -lfoo` 有何区别？`--start-group` 解决什么问题？弱符号（`__attribute__((weak))`）的典型用途是什么？
7. 写一个 header-only 的工具库（`static inline` 版本），并说明如何用 `-ffunction-sections -fdata-sections -Wl,--gc-sections` 控制代码膨胀。

### 考察点

链接属性、C99 inline 模型、暂定定义与 common symbol、`const` 链接属性、静态库解析顺序、弱符号。

---

## 第 8 题：setjmp / longjmp 与错误处理 ★★★★☆

### 背景

一个 C 服务用 `setjmp/longjmp` 做「异常」；上线后偶发返回错误的值，且只在 `-O2` 下出现。

### 代码

```c
#include <setjmp.h>
#include <stdio.h>

static jmp_buf env;

static int foo(void)
{
    int x = 1;
    if (setjmp(env) == 0) {
        x = 2;
        longjmp(env, 1);
    }
    return x;
}

int main(void)
{
    printf("%d\n", foo());
    return 0;
}
```

### 问题

1. 这段代码在 `-O0` 和 `-O2` 下分别可能输出什么？把 `int x` 改成 `volatile int x` 后呢？请用「寄存器分配 + `longjmp` 恢复寄存器现场」解释。
2. C 标准对「被 `longjmp` 跨越修改的自动变量」到底怎么规定？哪一条？为什么要求 `volatile`？`volatile` 在这里的作用是「禁止优化」还是「保证原子性」？
3. `longjmp(env, 0)` 会发生什么？`setjmp` 的返回值有哪几种？为什么 `setjmp` 只能出现在特定的上下文（`if`/`switch`/`while` 的条件、比较、取反、表达式语句）？写成 `int r = setjmp(env);` 为什么是 UB？
4. 如果 `longjmp` 的目标函数已经返回，会发生什么？为什么是 UB？栈帧已经被复用的情况下，`longjmp` 具体破坏了什么？
5. 在信号处理函数里 `longjmp` 有什么限制？`sigsetjmp`/`siglongjmp` 与普通版本的区别是什么？什么是 async-signal-safe？为什么 `printf` 不能在信号处理函数里调用？
6. 用 `setjmp/longjmp` 实现一套 `TRY/CATCH/THROW` 宏，要求支持嵌套、能携带错误码。再讨论：与 `goto cleanup`、返回错误码、C++ 异常、`ucontext` 协程相比，工程上如何选型？
7. 附加：C++ 中 `longjmp` 跨越带有非平凡析构的栈帧为什么是 UB？VLA 作用域内 `longjmp` 有什么额外约束？

### 考察点

非局部跳转、`volatile` 的真实语义、`setjmp` 的上下文限制、信号安全、错误处理架构。

---

## 第 9 题：并发、原子操作与内存序 ★★★★★

### 背景

一个「无锁」计数器在 4 核机器上偶尔少计数；一个「用 volatile 做同步」的标志位在 ARM 上永远等不到。

### 代码

```c
/* 版本 A：volatile 同步 */
volatile int flag = 0;
int data = 0;

/* T1 */ data = 42; flag = 1;
/* T2 */ while (!flag) ; assert(data == 42);

/* 版本 B：多线程自增 */
int counter = 0;
/* 4 个线程各执行 1000000 次 */ counter++;
```

### 问题

1. 版本 A 在 C11 内存模型下是什么行为？`volatile` 能提供原子性、可见性、顺序性中的哪些？为什么编译器可以把 `while (!flag)` 优化成 `if (!flag) for(;;);`？CPU 层又可能怎么重排（StoreLoad、StoreStore）？
2. 什么是 data race？C11 对 data race 的后果是怎么规定的（哪一节）？为什么说「data race 不是读到旧值，而是整个程序变成 UB」？
3. 用 `atomic_flag` 实现自旋锁，说明：
   - 加锁为什么用 `memory_order_acquire`，解锁为什么用 `memory_order_release`？
   - `atomic_flag_test_and_set` 默认是什么内存序？
   - 自旋时为什么要加 `pause`（x86）/`yield`（ARM）？`sched_yield()` 适合放在自旋循环里吗？
4. 版本 B 中 `counter++` 在 x86-64 上通常编译成哪三条指令？为什么多线程会丢更新？`atomic_fetch_add(&counter, 1)` 编译成什么？`counter++` 在 `_Atomic int` 上是否原子？它和 `atomic_fetch_add` 有什么区别（返回值、内存序）？
5. 写一个单生产者/单消费者（SPSC）环形队列：
   ```c
   typedef struct { /* ... */ } ring_t;
   bool ring_push(ring_t *r, int v);
   bool ring_pop(ring_t *r, int *v);
   ```
   要求：容量为 2 的幂；使用 acquire/release；不使用锁；正确处理「满/空」判定；说明为什么无符号回绕让 `head - tail` 仍然正确。再解释「伪共享（false sharing）」以及为什么要 `_Alignas(64)`。
6. 无锁栈的 `pop` 用 CAS 实现时，ABA 问题是如何产生的？举一个具体的交错执行序列。如何用 tagged pointer、hazard pointer、epoch reclamation 解决？各自的代价是什么？
7. `memory_order_relaxed` 适合什么场景？举一个「用 relaxed 足够」和一个「用 relaxed 会错」的例子。`memory_order_seq_cst` 在 x86 上需要额外指令吗？在 ARM 上呢？

### 考察点

C11 内存模型、数据竞争、原子类型与内存序、自旋锁、SPSC 队列、ABA、伪共享、x86 TSO 与 ARM 弱内存模型。

---

## 第 10 题：预处理宏与元编程 ★★★★☆

### 背景

面试官说：「C 没有模板，但内核用宏干了很多模板的事。你给我写几个。」

### 代码

```c
#define SQUARE(x) x * x
#define MAX(a, b) ((a) > (b) ? (a) : (b))

#define STR(x) #x
#define CAT(a, b) a##b
```

### 问题

1. `SQUARE(1 + 2)`、`SQUARE(i++)` 分别展开成什么？各有什么问题？给出 4 种改进方案（加括号、`static inline`、`_Generic`、GNU 语句表达式 `({ ... })`），并说明每种方案的代价与可移植性。为什么「加括号」只能解决优先级，解决不了副作用？
2. `MAX(a++, b++)` 有什么问题？如果 `a` 和 `b` 类型不同（`int` 与 `unsigned`），`((a) > (b) ? (a) : (b))` 的结果类型是什么？这会带来什么 bug？如何用 `_Generic` 或 `__typeof__` 修正？
3. 为什么宏体通常写成 `do { ... } while (0)`？请展开分析 `if (cond) MACRO(); else ...` 在宏体为 `{ ... }`、`do {...} while(0)`、裸语句三种情况下的行为。`do {...} while(0)` 后面能不能不加分号？
4. `STR(FOO)` 与 `STR(1 + 2)` 分别得到什么？如果要让 `STR(FOO)` 先展开 `FOO` 的宏定义（比如 `#define FOO 42`），要怎么写？为什么需要两层宏？
5. `CAT(a, b)` 在 `a` 或 `b` 本身是宏时为什么不会展开？写出正确的两层版本。`CAT(1, 2)` 得到什么？`CAT(FOO, BAR)` 在 `FOO`/`BAR` 未定义时得到什么？
6. 写 `offsetof`、`container_of`、`BUILD_BUG_ON`（C11 之前和之后两种写法）。解释旧式 `typedef char __static_assert[(cond) ? 1 : -1];` 为什么能工作，以及它和 `_Static_assert` 的差别。
7. 用 `_Generic` 写一个类型通用的 `print` 宏，支持 `int`、`double`、`const char *`。再写一个 `SWAP(a, b)`，要求：任意类型、只求值一次、不使用 `memcpy` 的临时缓冲区。说明 GNU 语句表达式版本的移植性风险。
8. 用 X-macro 维护一份「错误码 → 字符串」表，要求新增错误码只改一处。再说明变参宏 `__VA_ARGS__`、GNU 的 `##__VA_ARGS__`、C23 的 `__VA_OPT__` 的区别，以及 `#if` 中为什么不能用 `sizeof`。

### 考察点

宏展开规则（参数预扫描、`#`/`##`）、`do-while(0)`、类型泛型、静态断言、X-macro、变参宏的可移植性。

---

## 附：难度对标说明

| 题号 | 主题 | 难度 | 对标岗位/轮次 |
|---|---|---|---|
| 1 | UB 与求值顺序 | ★★★★☆ | 语言基础面、编译器/运行时 |
| 2 | 整型转换与溢出 | ★★★★☆ | 嵌入式、跨平台 SDK |
| 3 | 结构体布局/位域/FAM | ★★★★★ | 内核、驱动、网络协议栈 |
| 4 | 指针与声明符 | ★★★★☆ | 通用 C/C++ 一面深水区 |
| 5 | 内存管理与对齐 | ★★★★★ | 基础架构、分配器/运行时 |
| 6 | memcpy/字符串库 | ★★★★★ | 高性能库、libc 实现 |
| 7 | 链接与 inline | ★★★★☆ | 大型 C 工程、构建系统 |
| 8 | setjmp/longjmp | ★★★★☆ | 服务端、JIT/协程 |
| 9 | 并发与内存序 | ★★★★★ | 基础架构、无锁数据结构 |
| 10 | 宏元编程 | ★★★★☆ | 内核、跨平台抽象层 |

> 说明：上述题目不依赖冷门语法，但每一题都要求「标准条款 + 真实编译器行为 + 可落地代码」三件事同时答对；能完整答出 6 题以上，基本达到大厂 C/C++ 系统方向三面水平。
