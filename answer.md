# C 语言高阶面试题 · 思路与参考答案

> 对应 `question.md` 的 10 道题。每题按「**思路 → 标准依据 → 详解 → 代码 → 追问/易错点**」展开。
> 环境约定：x86-64（LP64，SysV AMD64），GCC/Clang，C11/C17。凡标「实现定义 / ABI 相关」的，换平台必须重新确认。
> 标准条款编号以 **C11（ISO/IEC 9899:2011）** 为准；C17 只做技术勘误，C23 的变化会单独标注。

---

## 第 1 题：未定义行为、求值顺序与优化器

### 思路

这道题只考一件事：**你脑子里有没有「抽象机器 + 序列点」的模型**。
判定口诀：

1. 找出同一标量对象上的**两个副作用**，或**一个副作用 + 一次值计算**；
2. 看它们之间有没有 `sequence point`（C11 起叫 sequenced-before）；
3. 没有 → `6.5p2` → **UB**，不是「结果不确定」，是「程序已经无效」。

### 逐条判定

| 编号 | 表达式 | 判定 | 依据 |
|---|---|---|---|
| ① | `i++ + i++` | **UB** | 两个副作用对 `i` 未定序，6.5p2 |
| ② | `(i = 3) + (i = 4)` | **UB** | 同上，两个赋值副作用未定序 |
| ③ | `arr[k] = k++` | **UB**（C11/C17；C++17 起为良定义） | 左操作数求值读 `k`，与 `k++` 的副作用未定序，6.5p2 + 6.5.16.1p3 |
| ④ | `x = x++ + 1` | **UB** | `x++` 的副作用与 `=` 的副作用未定序 |
| ⑤ | `printf("%d %d", f(), f())` | **UB** | 两次调用都修改 `g`，调用之间无定序，6.5.2.2p10 + 6.5p2 |
| ⑥ | `y = (y++, y + 1)` | **良定义** | 逗号运算符 `,` 是序列点，6.5.17p2；先 `y++`（y 变 2），再求 `y + 1` 得 3 |

> ⑥ 的精确结果：`y` 初值 1 → `y++` 求值为 1、副作用使 `y=2`；逗号右侧 `y+1` 在副作用完成后求值 → 3；赋给 `y` → **y = 3**。
> 注意：如果写成 `y = (y++, y)`，结果才是 2。

**① 的实际输出为什么「看起来正常」**：GCC `-O0` 下通常先算左 `i++`（得 0，i=1），再算右 `i++`（得 1，i=2），`a=1`；`-O2` 下可能折叠成 `i += 2; a = i` 或 `a = 2*i`，得到 `a=2`。**不同编译器、不同优化级别、不同版本都可能不同，且都「合法」——因为 UB 意味着标准对程序没有任何要求。**

### 2. UB vs 未指定（unspecified）

- **未指定行为**：标准给出一个可能性的集合，实现必须选一个，且**不需要文档化**（6.5p3 注）。例如函数实参的求值顺序、`f() + g()` 中先算哪个。
- **UB**：标准**不对程序施加任何要求**（3.4.3）。一旦执行到 UB，**整个程序**（包括 UB 之前的输出）都不再有保证。

`printf("%d %d", i, i)`：两个 `i` 的读取顺序未指定，但**都没有副作用**，读到的值相同，所以结果确定（不是 UB）。
`f() + f()`：两次调用各自有副作用，且**未定序** → 6.5p2 直接判 UB。关键不是「顺序未知」，而是「两个副作用之间没有 happens-before」。

### 3. 优化器如何「利用」UB

编译器的契约是：**输入程序不含 UB**。于是它可以反推：

```c
int f(int x) { return x + 1 > x; }   // 编译器直接 return 1
```

因为若 `x == INT_MAX`，`x + 1` 溢出是 UB，编译器可以假设这种情况不存在，于是 `x+1 > x` 恒真。这就是所谓的「UB 时间旅行」：**UB 会让它之前、之后的代码一起失去语义**。

对 ① 而言，`i++ + i++` 在 IR 层就是两个无定序的 `load/add/store`；优化器可以任选一种重写（合并成 `i += 2`、只保留一次自增、甚至把整个函数删掉），因为它可以假设「不会有两个副作用撞在同一对象上」。

工业界最著名的案例是 Linux 内核的 `if (ptr + offset < ptr)` 溢出检查被 GCC 优化掉，以及 glibc 里靠有符号溢出做边界检查的代码被删。

### 4. 工程规范与 Sanitizer 的边界

三条规范：

1. **一个表达式里只修改一个对象一次**；需要多次修改就拆语句，用临时变量。
2. 打开 `-Wall -Wextra -Wsequence-point -Wshadow`，把警告当错误（`-Werror`）。
3. CI 里跑 `-fsanitize=undefined,address`（UBSan + ASan）和 `-fno-strict-aliasing` 的对照构建。

**UBSan 不能 100% 捕获**，原因有三：

- 未定序修改这类 UB **没有运行时检查点**，GCC/Clang 的 UBSan 目前不检测 `i++ + i++`；它主要查有符号溢出、移位、空指针解引用、越界、对齐、`vptr` 等。
- UB 可能在插桩前就被优化掉（UB 让代码变成 unreachable）。
- 有些 UB 是「库契约」层面的（违反 `restrict`、`free` 非堆指针），需要 ASan/Valgrind/`_FORTIFY_SOURCE` 才能部分发现。

### 5. ④ vs ⑥

```c
x = x++ + 1;        /* UB：x++ 的副作用 与 = 的副作用 未定序 */
x = (x++, x + 1);   /* OK：逗号是序列点，x++ 完成后才求 x+1，x 最终为 3 */
```

注意 ④ 即使 GCC 给出 `x=2`，也不能依赖；`-ftrapv` 也抓不到它。

### 追问

- `i = i++;` → UB；`i++;` 单独成句 → 良定义。
- `a[i] = i++;` 在 **C++17** 起是良定义的（右操作数先于左操作数求值），但在 C 里仍是 UB。这是 C/C++ 面试的经典分界线。
- `int a = 1; a += (a = 3);` → UB（两个副作用未定序）。

---

## 第 2 题：整型提升、隐式转换与溢出

### 思路

三条规则按顺序套：

1. **整型提升**（6.3.1.1p2）：`char/short/_Bool`、位域等**rank < int** 的类型，若 `int` 能表示其全部值 → 转 `int`，否则转 `unsigned int`。
2. **寻常算术转换**（6.3.1.8）：两边先提升；若类型不同，按 rank 与「能否表示」决定：
   - 无符号 rank ≥ 有符号 rank → 有符号转无符号；
   - 有符号能表示无符号全部值 → 无符号转有符号；
   - 否则两边都转成「有符号类型对应的无符号类型」。
3. **赋值/传参**：转成目标类型；不在目标类型表示范围内 → 实现定义（6.3.1.3p3）或抛信号。

### 逐条结果（x86-64 Linux，LP64）

| 编号 | 结果 | 解释 |
|---|---|---|
| ① | `0`（假） | `int` 与 `unsigned` → `-1` 转 `UINT_MAX`，`4294967295 < 1` 为假 |
| ② | `0`（假） | `sizeof` 是 `size_t`（`unsigned long`，64 位）→ `-1` 转 `ULONG_MAX` → 假 |
| ③ | **死循环** | `i >= 0` 永远为真；`i=0` 时 `--i` 回绕到 `UINT_MAX` |
| ④ | `-56`（GCC/x86） | `char` 默认有符号；200 超范围 → 实现定义，按模 256 解释为 -56 |
| ⑤ | UB / UB | `-INT_MIN` 不可表示；`INT_MIN / -1` 商不可表示（6.5.5p6） |
| ⑥ | 合法 / UB | `1u << 31` 无符号合法；`1 << 31` 有符号结果不可表示（6.5.7p4） |
| ⑦ | 正常读到 EOF | `getchar` 返回 `unsigned char` 转 `int`，或 `EOF(-1)`；存 `char` 会截断 |
| ⑧ | `10`（本例） | `size_t → int` 若不可表示则实现定义；本例 10 可表示 |

**② 的关键**：`sizeof` 的结果类型是 `size_t`，而 `size_t` 是**无符号**的。`-1` 在转换后变成该无符号类型的最大值。32 位（ILP32，`size_t` = `unsigned int`）也是假；64 位仍是假。写成 `-1 < (int)sizeof(int)` 才是真——但这只是把陷阱挪了个位置（`sizeof` 超过 `INT_MAX` 时又会实现定义）。

**⑦ 为什么必须用 `int`**：`getchar()` 的契约是「返回 `unsigned char` 的值（0..255），或 `EOF`（负值，通常是 -1）」。若存进 `char`：
- 若 `char` 有符号，字节 `0xFF` 会变成 `-1`，与 `EOF` 相等 → 提前结束；
- 若 `char` 无符号，`EOF` 会变成 `255`，永远等不到 → 死循环。
两者都错。

**⑤ 的正确写法**：

```c
if (x == INT_MIN) { /* 特判 */ }
else { int nx = -x; }
/* 或者用宽类型 */
long long nx = -(long long)x;   /* long long 一定能表示 INT_MIN 的相反数 */
```

**⑥**：`1 << 31` 在 32 位 `int` 上是 UB，GCC 通常给 `INT_MIN`；`1LL << 31` 合法（`long long` 能表示）；`1u << 31` 合法（无符号回绕/模运算，6.2.5p9）。

### 5. 溢出安全的乘法与窄化

```c
#include <stdbool.h>
#include <limits.h>
#include <stddef.h>

/* 不依赖 UB 的有符号乘法溢出检测 */
bool mul_ovf(int a, int b, int *out)
{
    if (a > 0) {
        if (b > 0) { if (a > INT_MAX / b) return false; }
        else       { if (b < INT_MIN / a) return false; }
    } else if (a < 0) {
        if (b > 0) { if (a < INT_MIN / b) return false; }
        else       { if (a != 0 && b < INT_MAX / a) return false; }
    }
    /* a == 0 时结果必为 0 */
    *out = a * b;
    return true;
}

/* size_t -> int 的带检查转换 */
bool size_to_int(size_t n, int *out)
{
    if (n > (size_t)INT_MAX) return false;   /* size_t 无符号，比较安全 */
    *out = (int)n;
    return true;
}
```

`__builtin_mul_overflow(a, b, &r)` 的原理：编译器把它映射到目标指令集的溢出标志。x86-64 上对 `int` 通常生成 `imul`（32 位）后检查 `OF`，或提升到 64 位做乘法再比较范围；对 `size_t` 用 `mul` 的高半部分判溢出。它比手写分支更快，且没有 UB。

**为什么 `a * b` 本身不能先算再检查**：一旦溢出，UB 已经发生，后面的检查没有意义（优化器可以把检查删掉）。

### 6. 格式串与数据模型

```c
printf("%d", sizeof(int));   /* UB：%d 要 int，实参是 size_t（8 字节） */
printf("%zu", sizeof(int));  /* 正确 */
```

| 类型 | 正确格式 | 备注 |
|---|---|---|
| `size_t` | `%zu` | C99；MSVC 2015+ 支持 |
| `ptrdiff_t` | `%td` | |
| `int64_t` | `PRId64` | Linux LP64 下是 `"ld"`，Windows 下是 `"lld"` |
| `long` | `%ld` | **Windows LLP64 下 `long` 仍是 32 位** |
| `long long` | `%lld` | 各平台都是 64 位 |
| 指针 | `%p` + `(void *)` | |

`int32_t` 在 LP64 下是 `int`，在有些平台可能是 `long`；**永远用 `PRId32`/`PRId64`**，不要手写 `%d`。

Windows LLP64 的坑：`long` 32 位、指针 64 位，所以 `long` 存不下指针；存地址用 `intptr_t`/`uintptr_t`，格式串用 `PRIdPTR`/`PRIuPTR`。另外，用 `%ld` 打印 `size_t` 在 Windows 上会被截断（`size_t` 是 64 位，`long` 是 32 位）。

### 追问

- `unsigned u = 10; int i = -20; printf("%d", u + i);` → `u + i` 中 `i` 转无符号 → 结果 `4294967286`，用 `%d` 打印是实现定义（通常 -10）。
- `for (unsigned i = 10; i-- > 0; )` 与 `for (unsigned i = 10; i >= 0; --i)` 的差别：前者在 `i==0` 时判断 `0 > 0` 为假并**后置自减**，循环体执行 10 次（i 为 9..0）；后者死循环。
- `-fwrapv` 让有符号溢出按二补数回绕（不再是 UB），但会阻止部分优化；不要用它来「修复」逻辑错误。

---

## 第 3 题：结构体布局、位域与柔性数组

### 思路

布局只有两条规则：

1. 每个成员的偏移必须是**该成员对齐数**的整数倍，不够就插 padding；
2. 结构体总大小必须是**最大成员对齐数**的整数倍（尾部 padding），保证数组里每个元素都对齐。

`offsetof` 用「从 0 开始摆」手算即可。

### 1. 三个结构体

```text
S1: char c(0) | pad(1..3) | int i(4..7) | short s(8..9) | pad(10..11)
    sizeof = 12, _Alignof = 4
    offsetof: c=0, i=4, s=8

S2: char c(0) | pad(1) | short s(2..3) | int i(4..7)
    sizeof = 8,  _Alignof = 4
    offsetof: c=0, s=2, i=4

S3: char c(0) | pad(1..7) | double d(8..15) | char e(16) | pad(17..23)
    sizeof = 24, _Alignof = 8
    offsetof: c=0, d=8, e=16
```

`S1 → S2` 省 4 字节，是因为把 `short` 挪到 `char` 后面，`char+short` 刚好占满 4 字节，消除了 `int` 前的 3 字节空洞。

**重排也无法再省**的例子：

```c
struct A { char c; int i; };   /* 8 */
struct B { int i; char c; };   /* 也是 8：4 + 1 + 3 尾部 padding */
```

原因：尾部 padding 由最大对齐数决定，单靠重排消不掉。

### 2. 位域

```c
struct S4 { unsigned a:4; unsigned b:5; signed c:7; };
```

- `sizeof(S4) == 4`（GCC/Clang/MSVC 都是把 4+5+7=16 位放进一个 4 字节单元）。**但这不是标准保证的**：分配单元、能否跨单元、单元内从低位还是高位开始分配，都是**实现定义**（6.7.2.1p11、p12）。
- `&s.a` **非法**：位域没有地址，6.5.3.2p1 明确 `&` 的操作数不能是位域。编译器报 `cannot take address of bit-field`。
- `offsetof(struct S4, a)` **非法**：`offsetof` 要求成员可被 `&` 取址（7.19p3 的约束），GCC 直接报错。
- `int b:1`：若 `int` 位域按有符号处理（GCC 默认），取值范围是 **0 和 -1**。`s.b = 1; printf("%d", s.b)` 打印 **-1**。这是网络协议/寄存器映射里的经典事故。
- 比较陷阱：`if (s.c < 0)` 对 `signed c:7` 是正常的；但对 `unsigned` 位域，`< 0` 恒假（提升为 `int` 后仍非负）。
- 跨平台：位域的字节序和分配方向依赖 ABI，**不要把位域结构体直接映射到网络包/硬件寄存器**，除非你锁定编译器并加 `_Static_assert` 校验。

### 3. 柔性数组

```c
struct S5 { uint8_t len; uint8_t data[]; };
```

- `sizeof(struct S5) == 1`，`offsetof(struct S5, data) == 1`。
- 标准（6.7.2.1p18）：柔性数组成员被**当作不存在**来计算结构体大小，「except that it may have more trailing padding」。所以 `sizeof(S5) + n` 可能**多分配**（尾部 padding），但绝不会少——是安全的上界。精确写法是 `offsetof(struct S5, data) + n`。
- 分配：

```c
struct S5 *p = malloc(sizeof *p + n);   /* 常用、安全 */
if (!p) return NULL;
p->len = (uint8_t)n;
memcpy(p->data, src, n);
```

- 限制（6.7.2.1p3）：含 FAM 的结构体**不能作为另一个结构体的成员，也不能作为数组元素**（大小不确定）；不能初始化；`sizeof` 不含数据。
- `data[]` vs `data[1]` vs `data[0]`：
  - `data[]`：C99 标准写法，`sizeof` 不含数据；
  - `data[1]`：C89 的「struct hack」，`sizeof` 多 1 字节，且严格说访问 `data[1..]` 越界（实际可用）；
  - `data[0]`：GNU 扩展，C 标准不允许，`sizeof` 不含数据。

### 4. `offsetof` 与 `container_of`

```c
#include <stddef.h>
#include <stdint.h>

/* 经典写法：几乎所有的 <stddef.h> 都这么实现或内建 */
#define OFFSETOF(type, member) ((size_t)&(((type *)0)->member))

/* 更稳的写法：走整数运算，避免指针越界算术 */
#define CONTAINER_OF(ptr, type, member) \
    ((type *)((uintptr_t)(ptr) - offsetof(type, member)))
```

**`offsetof` 是否 UB？** 严格按 6.5.3.2，`&(((T*)0)->m)` 对一个「不在任何对象内」的左值取地址，形式上不合法。但：
- C 标准把 `offsetof` 放在 `<stddef.h>` 里，**实现者被允许用魔法**；GCC/Clang 内建 `__builtin_offsetof`；
- 所有主流编译器都保证它按「地址 0 + 偏移」求值，且在常量表达式里可用。

结论：**用标准库的 `offsetof`，不要自己发明**；自己写只在教学/无标准库环境里。

**`container_of` 的问题**：`(char *)ptr - offset` 若 `ptr` 指向某个成员，减完落在结构体内部，没问题；但如果这个「成员指针」本身是伪造的，或者结构体不在同一对象里，指针算术就越界（6.5.6p8 只允许数组内或 one-past-end）。用 `uintptr_t` 做减法可以绕开**指针算术**的 UB，但整数→指针的转换仍是实现定义的，且结果必须指向真实对象才可解引用。

**加类型检查**（GCC/Clang）：

```c
#define container_of(ptr, type, member) ({ \
    void *__mptr = (void *)(ptr); \
    _Static_assert(__builtin_types_compatible_p( \
        __typeof__(*(ptr)), __typeof__(((type *)0)->member)), \
        "container_of: member type mismatch"); \
    (type *)((char *)__mptr - offsetof(type, member)); })
```

Linux 内核的 `container_of` 还加了 `__same_type` 和 `BUILD_BUG_ON`，并且用 `__builtin_offsetof`。

### 5. `packed` 的代价

```c
struct __attribute__((packed)) hdr { uint8_t type; uint32_t len; };
```

- `sizeof == 5`，`len` 偏移 1，**未对齐**。
- x86 允许非对齐访问（慢一点），ARM 默认对某些指令/访问会 `SIGBUS`；编译器对 packed 成员通常生成逐字节访问，性能差。
- 取 `&hdr.len` 得到 `uint32_t *` 后解引用 → **UB**（6.3.2.3 对齐要求），`-Waddress-of-packed-member` 会警告。
- 对 packed 成员做 `_Atomic` 操作 / `atomic_load` → UB（原子类型要求自然对齐）。
- 正确姿势：定义**不 packed 的本地结构体**，用 `memcpy` 从字节流解析；或者用 packed 只做「字节视图」，所有读写走 `memcpy` 或逐字段 accessor。
- 与缓存行：把频繁写的共享变量放同一 cache line 会伪共享；用 `_Alignas(64)` 或填充隔开。packed 反而可能让多个变量挤进一行。

### 6. union 类型双关

```c
union U u; u.f = 1.0f; printf("%08x", u.u);
```

- C11：允许写一个成员、读另一个成员，读出的值按**对象表示**重新解释（6.5.2.3 的脚注，可能产生 trap representation）；GCC/Clang **明确文档化支持** type punning，MSVC 也支持。
- C++：读非活跃成员是 UB（除 common initial sequence），虽然编译器普遍支持。
- 可移植写法：

```c
float f = 1.0f; uint32_t bits;
memcpy(&bits, &f, sizeof bits);
```

`memcpy` 走字符类型，6.5p7 明确允许，且编译器会优化成一条 `movd`。

### 追问

- `_Alignof(double)` 与 `sizeof(double)` 在 x86-64 上都是 8；但 `long double` 是 16 字节、对齐 16。
- `struct { char c; double d; }` 重排后仍是 16，因为 `double` 对齐 8，`1+7+8` 与 `8+1+7` 一样。
- `offsetof` 对数组元素 `a[3]` 也合法：`offsetof(struct S, arr[3])`。
- `_Static_assert(sizeof(struct S1) == 12, "ABI changed");` 是跨平台结构体布局的必备保险。

---

## 第 4 题：指针、数组与复杂声明符

### 思路

- **数组名在大多数表达式里退化为指向首元素的指针**（6.3.2.1p3），例外：`sizeof`、`&`、`_Alignof`、字符串字面量初始化数组。
- `&a` 的类型是「指向整个数组的指针」，步长是**整个数组**。
- 声明符用「右左法则」：从标识符出发，先向右看 `[]`/`()`，再向左看 `*`，遇到括号就掉头。

### 1. sizeof 与指针算术

```c
int a[5];
int (*p)[5] = &a;
int *q = a;
```

| 表达式 | 类型 | 值（x86-64） |
|---|---|---|
| `sizeof(a)` | `size_t` | 20 |
| `sizeof(p)` | `size_t` | 8 |
| `sizeof(q)` | `size_t` | 8 |
| `sizeof(*p)` | `size_t` | 20 |

- `a` 在 `sizeof` 里不退化为指针，所以是 20。
- `a + 1`：`int *`，+4 字节。
- `&a + 1`：`int (*)[5]`，+20 字节。
- `p + 1`：+20；`q + 1`：+4。
- `a`、`&a`、`&a[0]` 的**值相同**（同一地址），但类型不同，这是「值一样、类型不同」的经典考点。

### 2. 数组形参退化

```c
void g(int arr[10]) { printf("%zu\n", sizeof(arr)); }  /* 8 */
```

6.7.6.3p7：形参声明为「数组类型」时，调整为「指向元素类型的限定指针」。所以 `int arr[10]` 就是 `int *arr`，`sizeof` 是指针大小。

`int arr[static 10]`：C99 的数组形参语法，表示**调用者必须保证至少 10 个元素**；编译器可据此做优化/检查。传少了是 UB（契约违反）。

### 3. 二维数组与 `int **`

```c
int m[3][4];
```

- `m` 退化为 `int (*)[4]`（指向「4 个 int 的数组」）；
- `m[0]` 是 `int[4]`，退化为 `int *`；
- `&m[0][0]` 是 `int *`；
- `m[1]` 的地址 = `m + 1*4*sizeof(int)` = +16 字节。

把 `m` 传给 `int **` 是**类型不兼容**，强转后 `pp[1]` 会把「第一行的第一个 int」当成一个指针去解引用 → 崩溃或读到垃圾。因为 `int **` 的语义是「指针的数组」，而 `m` 的内存里根本没有指针。

三种正确写法：

```c
void h1(int (*m)[4], int rows);                  /* 指向数组的指针 */
void h2(int rows, int cols, int m[rows][cols]);  /* C99 VLA */

void h3(int *m, int rows, int cols)              /* 扁平化，最通用 */
{
    for (int i = 0; i < rows; ++i)
        for (int j = 0; j < cols; ++j)
            m[i * cols + j] = 0;
}
```

### 4. 复杂声明

```c
char *(*(*x[3])(void))[5];
```

右左法则：

1. `x` 向右 `[3]` → **数组，3 个元素**；
2. 向左 `*` → 元素是**指针**；
3. 出括号向右 `(void)` → 指向**函数**（无参）；
4. 向左 `*` → 函数返回**指针**；
5. 向右 `[5]` → 指向**数组，5 个元素**；
6. 向左 `*` → 数组元素是**指针**；
7. 向右 `char` → 指向 **char**。

合起来：**x 是「3 个指针组成的数组」，每个指针指向「无参函数」，函数返回「指向 5 个 char* 的数组」的指针。**

```c
int (*(*f)(int))[10];
```

`f` 是**函数指针**，函数接受 `int`，返回「指向 `int[10]` 的指针」。

typedef 改写：

```c
/* x */
typedef char *char_ptr;
typedef char_ptr char_ptr_arr5[5];       /* char *[5] */
typedef char_ptr_arr5 *ptr_to_arr5;      /* char *(*)[5] */
typedef ptr_to_arr5 (*fn_ret_ptr)(void); /* 返回 ptr_to_arr5 的函数指针 */
fn_ret_ptr x[3];

/* f */
typedef int int_arr10[10];
typedef int_arr10 *ptr_to_int_arr10;
typedef ptr_to_int_arr10 (*fn_f)(int);
fn_f f;
```

可读性提升的关键：**从内到外一层一个 typedef，每个 typedef 只做一件事**。

### 5. `void *` 算术

标准不允许：6.5.6 要求指针指向完整对象类型，`void` 是不完整类型。GCC 作为扩展按「大小 1」处理，`vp + 1` 前进 1 字节；`-Wpointer-arith` 会警告。可移植写法：

```c
char *p = (char *)vp + offset;
```

### 6. const 的层次与 restrict

```c
const char * const *p;
```

从右往左读：`p` 是「指向 const 指针的指针」，该指针指向 `const char`。即：**p 本身可改，*p 不可改，**p 不可改**。

```c
char * const *p;    /* 指向「const 指针」的指针；*p 不可改，**p 可改 */
const char **p;     /* 指向「指向 const char 的指针」的指针；**p 不可改 */
```

注意：`char **` 不能隐式转成 `const char **`（否则可以通过后者写入 const 对象），6.5.16.1 有约束；`char **` → `const char * const *` 才是安全的。

`int *const p`：指针本身 const；`const int *p`：指向的内容 const。

`restrict`（6.7.3.1）：承诺在指针的生命周期内，**通过它访问的对象不会通过其他指针访问**。编译器可据此消除冗余 load、向量化、重排。违反契约（比如 `memcpy` 传重叠区间）是 UB，不是「结果可能不对」——优化器可以直接生成错误代码。

### 7. 命令分发器

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    const char *name;
    int (*handler)(int argc, char **argv);
} cmd_t;

static int cmd_help(int argc, char **argv) { (void)argc; (void)argv; puts("help"); return 0; }
static int cmd_quit(int argc, char **argv) { (void)argc; (void)argv; return 1; }

static const cmd_t cmds[] = {
    { "help", cmd_help },
    { "quit", cmd_quit },
};

int dispatch(int argc, char **argv)
{
    if (argc < 2) return -1;

    const char *name = argv[1];
    size_t n = strlen(name);
    const cmd_t *hit = NULL;
    size_t hits = 0;

    for (size_t i = 0; i < sizeof cmds / sizeof cmds[0]; ++i) {
        if (strncmp(cmds[i].name, name, n) == 0) {  /* 前缀匹配 */
            ++hits;
            hit = &cmds[i];
        }
    }
    if (hits != 1) {                                /* 0 个：未知；>1：有歧义 */
        fprintf(stderr, "unknown or ambiguous command: %s\n", name);
        return -1;
    }
    return hit->handler(argc - 1, argv + 1);
}
```

（`"he"` 会同时命中 `help` 和 `hello`，所以上面用 `hits` 计数，命中数不为 1 就报「未知或有歧义」；工程里通常还要求用户输入完整命令，或按注册顺序取第一个。）

函数指针表的优点：O(n) 但常数小、易扩展、可测试、可动态注册；缺点：调试时调用栈不直观、无法内联、需要保证表项生命周期。

### 追问

- `sizeof(int[10])` = 40；`sizeof(int(*)[10])` = 8。
- `int *p = &a;` 类型不兼容（`int(*)[5]` → `int*`），GCC 警告，值相同。
- `char *s = "abc"; s[0] = 'x';` → UB（修改字符串字面量），通常段错误。
- `int a[5]; int *p = a - 1;` → UB（越界），即使不解引用。

---

## 第 5 题：动态内存、对齐与 strict aliasing

### 思路

分配器的契约只有几句话，但每句都有坑：

- `malloc` 返回「 suitably aligned for any object with fundamental alignment」的块，或 NULL；
- `free` 只接受 `malloc/calloc/realloc` 返回的指针或 NULL；
- `realloc` 失败时**原块不变**。

### 1. 三个经典错误

**① `malloc(0)`**：可以返回 NULL，也可以返回一个**唯一的、可 free 的**非空指针（C11 7.22.3p1）。两种都合法，**非空时必须 free**。不能假设它等于 NULL。

**② `free(buf + 4)`**：UB（7.22.3.3p2），参数必须是分配器返回的指针。glibc 会检测到 chunk 头不对，打印 `free(): invalid pointer` 并 `abort()`；在 tcache 下也可能静默破坏堆，稍后才崩。

**③ `q = realloc(q, 1024)`**：失败时返回 NULL，**原块仍然有效**；直接赋给 `q` 就泄漏了原块。正确：

```c
void *tmp = realloc(q, 1024);
if (!tmp) { /* q 仍有效，决定保留还是 free */ free(q); return -1; }
q = tmp;
```

`realloc(p, 0)`：C17 是实现定义（可能 free 并返回 NULL，也可能返回唯一指针）；**C23 改为 UB**，不要依赖。

### 2. 溢出安全的 calloc

```c
#include <stdint.h>
#include <stdlib.h>

void *xcalloc(size_t n, size_t size)
{
    if (size != 0 && n > SIZE_MAX / size)
        return NULL;                 /* n * size 溢出 */
    return calloc(n, size);
}
```

`calloc` 本身也必须做这个检查（7.22.3.2），glibc 内部用 `__builtin_mul_overflow`。另外 `calloc` 可能拿到内核零页，比 `malloc + memset` 快。

### 3. 对齐

- `malloc` 的对齐 = `_Alignof(max_align_t)`。x86-64 glibc 是 **16**（因为 `long double` 16 字节），32 位通常 8。
- `max_align_t`：C11 定义的「对齐要求最严的标量类型」，通常是 `long double` 或 `struct { long long; long double; }`。
- 不够用的场景：`__m256`（32）、`__m512`（64）、cache line 对齐、某些 `_Atomic` 大结构、DMA 缓冲。
- `aligned_alloc(align, size)`（C11）：`align` 必须是实现支持的合法对齐，且 **`size` 必须是 `align` 的整数倍**（违反是 UB）；失败返回 NULL。
- `posix_memalign(&p, align, size)`：`align` 必须是 2 的幂且是 `sizeof(void*)` 的倍数；**不要求 size 是倍数**；返回错误码；得到的指针可以 `free()`。

### 4. 手写对齐分配

```c
#include <stdint.h>
#include <stddef.h>
#include <stdlib.h>

void *aligned_alloc_impl(size_t alignment, size_t size)
{
    /* 1. 只接受 2 的幂；0 和 1 视为无效 */
    if (alignment == 0 || (alignment & (alignment - 1)) != 0)
        return NULL;

    /* 2. 小于指针大小的对齐没有意义，统一抬到 sizeof(void*)，
          这样才能保证返回块前面放得下原始指针 */
    if (alignment < sizeof(void *))
        alignment = sizeof(void *);

    /* 3. 溢出检查：size + (alignment - 1) + sizeof(void*) */
    if (alignment > SIZE_MAX - sizeof(void *))
        return NULL;
    size_t overhead = alignment - 1 + sizeof(void *);
    if (size > SIZE_MAX - overhead)
        return NULL;

    void *raw = malloc(size + overhead);
    if (!raw)
        return NULL;

    /* 4. 在 raw 之后留出 sizeof(void*)，再向上取整到 alignment */
    uintptr_t base = (uintptr_t)raw + sizeof(void *);
    uintptr_t aligned = (base + alignment - 1) & ~(uintptr_t)(alignment - 1);

    /* 5. 在 aligned 前面存原始指针 */
    ((void **)aligned)[-1] = raw;
    return (void *)aligned;
}

void aligned_free_impl(void *ptr)
{
    if (ptr)
        free(((void **)ptr)[-1]);
}
```

边界条件逐条：

- **`alignment < sizeof(void*)`**：若不对齐抬到 8，可能出现 `aligned - 8 < raw`，把原始指针写到分配块外面。
- **`alignment == 1`**：`alignment - 1 == 0`，向上取整是 no-op；抬到 8 后仍正确。
- **`aligned - sizeof(void*) >= raw`**：因为 `base = raw + 8` 且 `aligned >= base`，所以 `aligned - 8 >= raw`，写入安全。
- **`aligned + size <= raw + total`**：`aligned <= base + alignment - 1 = raw + 8 + alignment - 1`，加上 `size` 正好不超过 `raw + total`。
- **溢出**：先算 `overhead` 再比较，避免 `size + overhead` 回绕。
- **配对**：只能由 `aligned_free_impl` 释放；不能 `free()`，不能 `realloc()`。
- **C11 的 `aligned_alloc`** 还要求 `size % alignment == 0`，这个实现不要求，更接近 `posix_memalign` 的语义。

### 5. strict aliasing 与 effective type

```c
int i = 0x3f800000;
float f = *(float *)&i;    /* UB */
```

- 6.5p7：对象的存储值只能通过「兼容类型、限定版本、对应有/无符号类型、聚合类型、字符类型」的左值访问。`float` 与 `int` 不兼容 → UB。
- 6.5p6（effective type）：`malloc` 出来的内存**没有声明类型**；第一次通过非字符类型写入时，该类型成为它的 effective type。之后再用不兼容类型读 → 违反 6.5p7。

合法写法：

```c
float f; memcpy(&f, &i, sizeof f);          /* 字符类型例外，最可移植 */
union { int i; float f; } u = { .i = i };   /* C 中实现定义但 GCC 保证 */
```

`memcpy` 版本在 `-O2` 下通常就是一条 `movd`，没有性能损失。

### 6. alloca / VLA / 调试

- `alloca`：在栈上分配，函数返回自动释放；**失败不返回 NULL，直接栈溢出**；不能用在循环里（栈只增不减）；不可移植（POSIX/BSD 有，标准没有）。
- VLA：C11 起可选（`__STDC_NO_VLA__`），大小不能为 0/负；`longjmp` 跳出 VLA 作用域是 UB；大 VLA 直接爆栈。
- 检测手段：
  - ASan：use-after-free、heap-buffer-overflow、double-free（quarantine 延迟复用）；
  - Valgrind：无需重编译，慢 20~50 倍；
  - glibc：`MALLOC_CHECK_=3`、`MALLOC_PERTURB_`、tcache 的 `key` 字段能抓一部分 double free；
  - `_FORTIFY_SOURCE=2/3`：编译期/运行期检查 `memcpy` 等。
- 「崩溃不在现场」：堆块被 free 后可能立刻被 reused，写越界破坏的是下一个 chunk 的 metadata，直到下一次 `malloc/free` 才崩。所以**第一次崩溃的位置通常不是第一次出错的位置**。

### 追问

- `free(NULL)` 合法，什么都不做。
- `malloc` 返回的指针 `free` 之后立即置 NULL，能防 double free，但防不了别名指针。
- `realloc` 可能原地扩展，也可能搬迁；搬迁后旧指针失效。
- `calloc(0, 0)` 同 `malloc(0)`。

---

## 第 6 题：memcpy / memmove 与字符串库函数

### 思路

- `memcpy` 的契约是「**不重叠**」，重叠是 UB；`memmove` 的契约是「**允许重叠**」。
- 性能优化的核心是「**按机器字/向量拷贝**」，但必须在**对齐、别名、边界**三条约束内做。

### 1. memmove

```c
#include <stddef.h>
#include <stdint.h>

void *my_memmove(void *dst, const void *src, size_t n)
{
    unsigned char *d = (unsigned char *)dst;
    const unsigned char *s = (const unsigned char *)src;

    if (d == s || n == 0)
        return dst;

    /* 用整数比较，避免无关指针之间的 < 运算（严格说也是 UB） */
    if ((uintptr_t)d < (uintptr_t)s || (uintptr_t)d >= (uintptr_t)s + n) {
        /* 不重叠，或 dst 在 src 之前：从前向后 */
        for (size_t i = 0; i < n; ++i)
            d[i] = s[i];
    } else {
        /* dst 落在 (src, src+n) 内：从后向前，避免覆盖尚未读的源字节 */
        for (size_t i = n; i != 0; --i)
            d[i - 1] = s[i - 1];
    }
    return dst;
}
```

两种重叠情形：

```text
情形 A：dst 在 src 之前
  src:  [-----------]
  dst: [-----------]
  从前向后安全（写 dst 不会碰到还没读的 src）

情形 B：dst 在 src 之后且重叠
  src: [-----------]
  dst:    [-----------]
  从前向后会把 src 的尾部提前覆盖 → 必须从后向前
```

### 2. 手写 memcpy

朴素版本：

```c
while (n--) *d++ = *s++;
```

慢的原因：每字节一次 load/store，无法利用 8/16/32 字节的总线宽度；循环控制开销大；编译器**有时**能自动向量化，但遇到别名不确定时不敢。

强行按字拷贝的 UB：

```c
uint64_t *dw = (uint64_t *)d;   /* ① 对齐：d 可能不是 8 的倍数 → UB */
uint64_t *sw = (uint64_t *)s;   /* ② 严格别名：s 指向 char 数组，用 uint64_t 读 → UB */
*dw++ = *sw++;                  /* ③ 越界：n 不是 8 的倍数时读到对象外 → UB */
```

正确写法（对齐头 + 字拷贝 + 尾）：

```c
#include <stdint.h>
#include <string.h>

void *my_memcpy(void *dst, const void *src, size_t n)
{
    unsigned char *d = (unsigned char *)dst;
    const unsigned char *s = (const unsigned char *)src;

    /* 1. 先把 dst 对齐到 8 字节 */
    while (n && ((uintptr_t)d & 7)) {
        *d++ = *s++;
        --n;
    }

    /* 2. 主体：用 memcpy 读写一个 word，规避对齐与别名问题 */
    while (n >= 8) {
        uint64_t w;
        memcpy(&w, s, 8);
        memcpy(d, &w, 8);
        d += 8; s += 8; n -= 8;
    }

    /* 3. 尾部 */
    while (n--) *d++ = *s++;
    return dst;
}
```

为什么 `memcpy(&w, s, 8)` 不会被「拖慢」：编译器认识 `memcpy` 内建，会把它变成一条 `mov`（x86 上未对齐也允许），同时**合法地**完成「从 char 缓冲区读 8 字节」——因为 `memcpy` 走字符类型，不受严格别名约束。

如果坚持用指针强转，需要：

```c
typedef uint64_t __attribute__((may_alias)) u64a;
```

但 `may_alias` 是 GCC/Clang 扩展，MSVC 没有。

### 3. Duff's device

```c
void duff_copy(char *to, const char *from, size_t count)
{
    if (count == 0) return;              /* 关键修复 */

    size_t n = (count + 7) / 8;
    switch (count % 8) {
    case 0: do { *to++ = *from++;
    case 7:      *to++ = *from++;
    case 6:      *to++ = *from++;
    case 5:      *to++ = *from++;
    case 4:      *to++ = *from++;
    case 3:      *to++ = *from++;
    case 2:      *to++ = *from++;
    case 1:      *to++ = *from++;
            } while (--n > 0);
    }
}
```

**合法性**：`case` 标号只是标签，`switch` 允许跳到复合语句内部的标签处；`do { ... } while` 的循环体是一个语句，标签可以出现在其中。C 标准对「跳入语句内部」没有一般性禁止，唯一相关约束是 6.8.6.1p1：`goto` 不能从 VLA 的作用域外跳进作用域内（`switch` 跳到 VLA 声明之后同样受限）。所以这是**合法但反直觉**的代码，属于 K&R 时代流传下来的经典技巧。

**`count == 0` 的 bug**：`n = 0`，`switch (0)` 跳到 `case 0`，而 `case 0` 在 `do` 体**内部**，于是先执行一整轮 8 字节拷贝，再 `--n` 从 0 回绕到 `SIZE_MAX` → 灾难。所以必须先判断 0。

今天 Duff's device 基本只用于教学；现代编译器对简单循环的向量化更好，手写它反而可能阻碍优化。

### 4. strncpy / strncat / strlcpy

```c
char dst[8];
strncpy(dst, "hello world", sizeof dst);
```

- `strncpy(dst, src, n)`：最多拷 n 字节；若 `strlen(src) < n`，**剩余全部补 '\0'**；若 `strlen(src) >= n`，**不补 '\0'** → `dst` 不是字符串。它是为固定长度字段（如早期 UNIX 目录项）设计的，不是「安全版的 strcpy」。
- `strncat(dst, src, n)`：从 `dst` 的 '\0' 开始追加，最多 n 个字符，**总是**补 '\0'。所以 `n` 是「源最多取多少」，而 `dst` 需要 `strlen(dst) + n + 1` 的空间。常见错误是传「dst 剩余空间」。
- `strlcpy(dst, src, size)`：BSD 引入，返回 `strlen(src)`，截断且保证 '\0'；glibc 2.38 才有；不是 ISO C。
- 现代推荐：`snprintf(dst, sizeof dst, "%s", src)`（慢但安全），或自己封装 `strcpy_s`（C11 Annex K，实现支持度差）。

### 5. 字长 strlen 与 haszero

```c
#include <stdint.h>
#include <stddef.h>

#define HASZERO(x) (((x) - 0x0101010101010101ULL) & ~(x) & 0x8080808080808080ULL)

size_t my_strlen(const char *s)
{
    const char *p = s;
    /* 先按字节走到对齐 */
    while ((uintptr_t)p & 7) {
        if (!*p) return (size_t)(p - s);
        ++p;
    }
    while (1) {
        uint64_t w;
        memcpy(&w, p, 8);          /* 可能读到字符串末尾之后 */
        if (HASZERO(w)) {
            for (int i = 0; i < 8; ++i)
                if (p[i] == '\0') return (size_t)(p - s + i);
        }
        p += 8;
    }
}
```

`HASZERO` 原理：对每个字节减 1，若某字节为 0，则变成 0xFF 并**向高位借位**；`~x` 的高位在「原字节 < 0x80」时为 1；两者与 `0x80..` 相与，只有「原字节为 0」的位会同时满足。可以证明它**没有假阳性**（没有 0 字节就不会产生借位链）。

**越界读风险**：按 8 字节读可能越过 `malloc` 块末尾，ASan 会报 `heap-buffer-overflow`；若恰好跨页且下一页未映射，会 SIGSEGV。glibc 的 `strlen` 用「页边界对齐」技巧避免跨页：先读到下一个页边界，再按字读。**这是「可移植性 vs 性能」的经典取舍**：标准不允许越界读，但真实 libc 会做，并靠页对齐规避 fault。

### 6. memcmp / strcmp / strtok

- `memcmp(a, b, n)`：比较 n 字节，按 `unsigned char`，返回 <0/0/>0；
- `strcmp(a, b)`：遇到 '\0' 停止，返回 <0/0/>0；
- **两者相等时都返回 0**，所以 `if (strcmp(a, b))` 表示「不相等」——这是最常见的逻辑反转 bug。
- `strtok` 用**静态内部状态**保存下次起点，不可重入、非线程安全；`strtok_r(str, delim, &saveptr)` 把状态交给调用者。

### 7. restrict 与重叠

```c
void *memcpy(void * restrict s1, const void * restrict s2, size_t n);
```

`restrict` 告诉编译器：在 `memcpy` 执行期间，通过 `s1` 和 `s2` 访问的对象**不会通过其他途径被访问**。于是编译器可以假设两块内存不重叠，放心地「先读一大块、再写一大块」或向量化。

如果调用者传了重叠区间：契约违反 → UB。实际后果可能是「后半段被前半段的写入覆盖」，也可能是向量化后**完全错误**的结果。`memmove` 没有 `restrict`，所以必须自己处理重叠。

### 追问

- `memcpy(dst, src, 0)` 合法，即使指针为 NULL（只要不访问）。
- `memcpy` 返回 `dst`，支持链式调用。
- `memcmp` 不能用于比较含填充字节的结构体（padding 内容不确定）。
- `strdup` 是 POSIX，不是 C 标准；`strndup` 同理。

---

## 第 7 题：链接属性、inline 与多文件工程

### 思路

这道题的核心是**翻译单元（TU）与链接**：

- 头文件被 `#include` 到每个 `.c`，其中的**定义**就会在每个 TU 里各出现一次；
- C 的规则是「一个外部链接对象/函数，**全程序只能有一个定义**」（6.9p5）；
- `inline` 在 C99 里引入了一套「inline definition 不等于 external definition」的模型。

### 1. C99 的 inline 语义

6.7.4p6/p7：

- 若一个 TU 里函数的所有文件作用域声明都带 `inline` 而**不带 `extern`**，则该 TU 里的定义是 **inline definition**；
- **inline definition 不提供外部定义**；
- 若程序里没有任何 TU 提供外部定义，而又有地方需要这个符号（比如 `-O0` 没内联）→ **链接错误**。

所以：

```sh
gcc -O0 a.c b.c   # undefined reference to `add'
gcc -O2 a.c b.c   # 内联了，不需要符号，链接成功
```

这就是「海森堡 bug」：**优化级别改变了程序是否能链接**。

### 2. 三种正确写法

**写法 A：`static inline`（最常用）**

```c
/* a.h */
static inline int add(int a, int b) { return a + b; }
```

- 内部链接，每个 TU 一份；不会多重定义；
- 没被内联时，每个 TU 生成一份本地副本 → 代码膨胀；
- 函数地址在每个 TU 不同（比较函数指针会失败）。

**写法 B：`inline` + 一个 TU 提供外部定义**

```c
/* a.h */
inline int add(int a, int b) { return a + b; }

/* a.c（恰好一个 TU） */
extern inline int add(int a, int b);   /* C99：这是外部定义 */
```

或者更直白：头文件只放 `inline` 定义，某个 `.c` 里写一个**不带 inline** 的定义：

```c
/* a.c */
int add(int a, int b) { return a + b; }
```

但这样同一个函数有两份定义（一份 inline definition、一份 external definition），语义上合法，容易混淆，不推荐。

**写法 C：宏**（类型不安全，不推荐用于函数）。

**C99 vs GNU89 为什么相反**：

| | GNU89 | C99 |
|---|---|---|
| `inline` | 内联 + **发出外部定义** | **只**是 inline definition，不发出 |
| `extern inline` | **只**内联，不发出外部定义 | **发出**外部定义 |

GCC 在 `-std=gnu89` 下用 GNU 语义，`-std=c99` 及以后用 C99 语义；`-fgnu89-inline` 可强制 GNU 语义。老代码迁移时的链接错误大多来自这个反转。

### 3. tentative definition 与 `-fno-common`

```c
/* a.h */
int counter;      /* 暂定定义（tentative definition） */
```

- C11 6.9.2：文件作用域的 `int counter;` 是 tentative definition；若 TU 结束时没有真正的定义，就变成「初值为 0 的定义」。
- 两个 TU 都包含 → 两个定义 → 违反 6.9p5（外部链接只能有一个定义）。
- 旧 GCC 默认 `-fcommon`：把 tentative definition 放进 **common block**，链接器合并成一个，所以「能过」。
- **GCC 10 起默认 `-fno-common`**：每个都变成真正的定义 → `multiple definition of 'counter'`。这就是老项目升级编译器突然链接失败的原因。
- 正确写法：

```c
/* a.h */
extern int counter;   /* 声明 */
/* a.c */
int counter = 0;      /* 唯一定义 */
```

`-fcommon` 只是临时兼容，长期会掩盖「同一变量被定义多次」的真实 bug（尤其是两个 TU 各有一份、各自初始化不同值时）。

另外：`int counter = 0;` 在 `a.c` 里，同时头文件里还有 `int counter;`，在同一个 TU 里是「暂定定义 + 定义」，合法（合并为一个）；问题出在 `b.c` 也有一份暂定定义。

### 4. `const` 的链接属性

- **C**：文件作用域的 `const int LIMIT = 100;` **没有** `static` 时是**外部链接**（6.2.2p5）。头文件里写 → 每个 TU 一个定义 → 多重定义。
- **C++**：命名空间作用域的 `const` 默认**内部链接**（除非 `extern`），所以 C++ 头文件里写 `const int LIMIT = 100;` 没问题。

这是 C/C++ 混编时的经典差异。C 里正确写法：

```c
/* a.h */
extern const int LIMIT;
/* a.c */
const int LIMIT = 100;
```

或者用 `enum { LIMIT = 100 };` / `#define`（无地址，但可用于数组大小）。

### 5. static 的三种含义

1. **文件作用域**：内部链接（6.2.2p3），符号不导出；
2. **块作用域**：静态存储期（6.2.4p3），只初始化一次，函数返回后仍保留；
3. **数组形参**：`void f(int a[static 10])`，表示至少 10 个元素（6.7.6.3p7），与链接无关。

`static` 函数放在头文件里：

- 每个 TU 一份副本 → 代码膨胀；
- 每个 TU 的地址不同 → 函数指针比较失败；
- 调试符号重复；
- 但可以内联、没有链接冲突，适合 header-only 小工具。

`static` 函数**不能被其他 TU 调用**（没有外部链接），但可以被本 TU 内联。

### 6. 静态库链接顺序

链接器从左到右处理：

- 遇到 `.o`：无条件加入；
- 遇到 `.a`：只抽取「当前未定义符号」能匹配的成员。

所以：

```sh
gcc main.o -lfoo -lbar    # foo 依赖 bar → OK
gcc main.o -lbar -lfoo    # bar 先处理，foo 的符号还没出现 → foo 里的 bar 依赖可能未解析
```

`--start-group ... --end-group` 让链接器在这组库之间反复扫描直到不动点，解决循环依赖（代价是变慢）。

弱符号 `__attribute__((weak))`：允许用户提供强定义覆盖库里的默认实现；也可以用来做「可选功能」的钩子。注意弱符号在静态库里的抽取行为很微妙（未定义弱符号不会强制抽取成员）。

### 7. header-only 与 section GC

```c
/* util.h */
#ifndef UTIL_H
#define UTIL_H
#include <stddef.h>
static inline int util_clamp(int v, int lo, int hi) {
    return v < lo ? lo : (v > hi ? hi : v);
}
#endif
```

控制膨胀：

```sh
gcc -ffunction-sections -fdata-sections ... -Wl,--gc-sections
```

每个函数/数据进独立 section，链接时丢弃未引用的 section。对 `static inline` 没被内联的副本也有效（但仍需 `-ffunction-sections`）。

### 追问

- `extern int x;` 在块作用域合法（声明外部变量）；`static int x;` 在块作用域是「静态局部变量」，不是内部链接。
- 头文件里定义变量 + `#pragma once` 不能解决问题：`#pragma once` 只防同一 TU 重复包含，不防多 TU。
- `__attribute__((constructor))` 的初始化顺序在多个 TU 间**未指定**，不要跨 TU 依赖。

---

## 第 8 题：setjmp / longjmp 与错误处理

### 思路

`setjmp` 保存的是「**寄存器现场 + 栈指针 + 返回地址**」，`longjmp` 把 CPU 状态恢复到这个现场。它**不恢复**栈上的局部变量，也不执行任何清理。所以：

- 栈上的非 `volatile` 局部变量，若在 `setjmp` 之后被修改，`longjmp` 后值**不确定**；
- 跳到已经返回的函数 → 栈帧已失效 → UB。

### 1. `-O0` vs `-O2`

```c
static int foo(void)
{
    int x = 1;
    if (setjmp(env) == 0) { x = 2; longjmp(env, 1); }
    return x;
}
```

- `-O0`：`x` 在栈上，`x = 2` 写内存；`longjmp` 只恢复寄存器，不碰栈 → 返回 **2**。
- `-O2`：`x` 被放进 callee-saved 寄存器（如 `%rbx`）；`setjmp` 时 `%rbx = 1` 被存进 `jmp_buf`；`longjmp` 恢复 `%rbx = 1` → 返回 **1**。

改成 `volatile int x`：强制每次读写都走内存，`longjmp` 不恢复内存 → 返回 **2**，行为确定。

### 2. 标准依据

C11 7.13.2.1p3：

> 所有可访问对象的值，在 `longjmp` 调用时都是最新的，**除了**：在包含 `setjmp` 调用的函数内、具有自动存储期、**不是 volatile 限定**、且在 `setjmp` 与 `longjmp` 之间被修改的对象，其值**不确定**。

`volatile` 在这里的作用是**强制对象有稳定的内存位置**，让「恢复寄存器」不会回滚它；它**不保证原子性、不保证线程可见性**。

### 3. 返回值与上下文限制

- `setjmp` 直接返回 → 0；
- `longjmp(env, v)` → `setjmp` 再次返回 `v`；
- `v == 0` 时，标准规定改为 **1**（7.13.2.1p3），所以 `longjmp(env, 0)` 后 `setjmp` 返回 1。

**上下文限制**（7.13.1.1p4）：`setjmp` 只能出现在：

1. 选择/循环语句的**整个控制表达式**：`if (setjmp(env))`、`while (setjmp(env))`；
2. 与整数常量比较的**整个控制表达式**：`if (setjmp(env) == 0)`；
3. `!` 的**整个操作数**，且结果作为整个控制表达式；
4. **表达式语句**的整个表达式（可强转 `void`）。

`int r = setjmp(env);` **不在**上述列表 → UB。原因：`setjmp` 可能是宏，依赖编译器在特定位置保存/恢复现场；赋值语境下编译器无法保证。

### 4. 跳到已返回的函数

7.13.2.1p2：如果包含 `setjmp` 的函数已经终止执行，`longjmp` 的行为 **undefined**。

原因：该函数的栈帧已经被释放/复用；`longjmp` 把栈指针恢复到旧位置，之后任何函数调用/局部变量访问都会踩到别的函数的栈帧 → 随机崩溃或静默数据损坏。

### 5. 信号处理中的 longjmp

- POSIX：如果信号处理函数打断了**非 async-signal-safe** 的函数，然后调用 `longjmp`，行为未定义。`setjmp`/`longjmp` 本身不在 async-signal-safe 列表里。
- `sigsetjmp(env, savemask)` / `siglongjmp(env, val)`：`savemask != 0` 时保存/恢复信号掩码，避免 `longjmp` 后信号屏蔽字处于错误状态。
- async-signal-safe：`write`、`_exit`、`sigaction`、`kill` 等；`printf`、`malloc`、`free`、`pthread_mutex_lock` 都**不是**（stdio 锁、堆锁可能被信号打断时正持有 → 死锁）。

### 6. TRY/CATCH 宏

```c
#include <setjmp.h>

#define TRY do { \
    jmp_buf _env; \
    volatile int _code = 0; \
    if (setjmp(_env) == 0) {

#define CATCH(c) \
    } else { \
        (c) = _code; \
        {

#define END_TRY \
        } \
    } \
} while (0)

#define THROW(c) do { _code = (c); longjmp(_env, 1); } while (0)
```

用法：

```c
int err;
TRY {
    if (bad) THROW(42);
} CATCH(err) {
    printf("caught %d\n", err);
} END_TRY;
```

三个关键点：

1. **`_code` 必须是 `volatile`**：它在 `setjmp` 之后被修改、又被 `longjmp` 跨过，非 `volatile` 的值不确定（正是第 1 小问的规则）。
2. `THROW` 直接引用外层作用域的 `_env`/`_code`，所以**嵌套 `TRY` 时自动绑定最近一层**；每个 `TRY` 有自己的 `jmp_buf`，不需要保存/恢复。
3. `END_TRY` 的三个 `}` 分别关闭用户的花括号块、`else` 块、`do` 块；`_env` 是 `jmp_buf` 数组，`setjmp(_env)` 合法，若想传结构体则要 `memcpy`，因为数组不能赋值。

局限：`longjmp` 的目标 `_env` 必须在栈上仍然有效；`THROW` 只能在本 `TRY` 的词法作用域内使用；不能跨线程。

**选型对比**：

| 方案 | 优点 | 缺点 |
|---|---|---|
| `goto cleanup` | 零开销、可读、可调试 | 只能函数内跳转；清理代码集中 |
| 错误码返回 | 最可移植、最可控 | 每层都要检查，样板代码多 |
| `setjmp/longjmp` | 跨函数跳转、C 里最接近异常 | 绕过清理、`volatile` 陷阱、信号不安全 |
| C++ 异常 | 自动析构、类型安全 | 需要 C++、有运行时开销 |
| `ucontext` 协程 | 可做用户态调度 | 已废弃（POSIX.1-2008 移除）、平台相关 |

### 7. C++ 与 VLA

- C++ 中 `longjmp` 跨越「具有非平凡析构函数的栈帧」是 UB（不调用析构函数，资源泄漏/状态损坏）。
- VLA 作用域内 `longjmp`：如果跳入/跳出 VLA 的作用域且 VLA 大小不同，UB；因为栈指针恢复后 VLA 的边界不再有效（C11 7.13.2.1p3 的脚注/6.2.4 相关约束）。

### 追问

- `setjmp` 保存的信号掩码在 `longjmp` 后是否恢复？普通版不保证，`sigsetjmp` 才明确。
- `longjmp` 不会调用 `atexit`、不会 flush stdio、不会释放任何资源。
- 在 `setjmp` 和 `longjmp` 之间 `malloc` 的内存，`longjmp` 后仍然泄漏（没有栈展开）。

---

## 第 9 题：并发、原子操作与内存序

### 思路

C11 内存模型的三句话：

1. 非原子对象的并发读写（至少一个写）且没有 happens-before → **data race → UB**（5.1.2.4p25）；
2. `volatile` **不是**同步原语，它只约束编译器对该对象的访问，不提供原子性/顺序性/跨线程可见性；
3. 原子操作通过 `memory_order` 建立 happens-before。

### 1. 版本 A 为什么错

```c
volatile int flag = 0;
int data = 0;
/* T1 */ data = 42; flag = 1;
/* T2 */ while (!flag) ; assert(data == 42);
```

- `flag`、`data` 都不是原子类型 → 并发访问构成 **data race** → UB。
- 编译器层面：`data = 42` 与 `flag = 1` 之间没有同步，编译器可以把 `data` 的写**推迟**到 `flag = 1` 之后（只要单线程语义不变）；`T2` 也可以把 `data` 的读**提前**到循环之前。
- CPU 层面：x86 TSO 允许 **StoreLoad** 重排（后一个 load 先于前一个 store 对其他核可见），ARM 更弱，StoreStore 都可能重排。
- `volatile` 只保证「每次都从内存读写、不缓存到寄存器」，对 CPU 的乱序/缓存一致性无能为力，也不保证 32 位以上对象的原子性。

正确写法：

```c
atomic_int flag = 0;
int data = 0;
/* T1 */ data = 42; atomic_store_explicit(&flag, 1, memory_order_release);
/* T2 */ while (!atomic_load_explicit(&flag, memory_order_acquire)) ; assert(data == 42);
```

### 2. data race 的后果

5.1.2.4p25：两个线程的冲突动作（同一内存位置、至少一个写、不是原子操作），且两者之间没有 happens-before → **UB**。不是「读到旧值」，而是**整个程序没有语义**：编译器可以假设不存在 race，从而做出任意优化（包括删除循环、把读提到写之前、把变量常量传播）。

### 3. 自旋锁

```c
#include <stdatomic.h>

typedef struct {
    atomic_flag flag;
} spinlock_t;

#define SPINLOCK_INIT { ATOMIC_FLAG_INIT }

static inline void spin_lock(spinlock_t *l)
{
    while (atomic_flag_test_and_set_explicit(&l->flag, memory_order_acquire)) {
        /* x86: __builtin_ia32_pause(); ARM: __asm__ __volatile__("yield"); */
    }
}

static inline void spin_unlock(spinlock_t *l)
{
    atomic_flag_clear_explicit(&l->flag, memory_order_release);
}
```

- **加锁 acquire**：保证临界区内的读写不会被重排到加锁之前（防止读到锁外数据）。
- **解锁 release**：保证临界区内的写对下一个获得锁的线程可见（不会被重排到解锁之后）。
- `atomic_flag_test_and_set` 的默认内存序是 `memory_order_seq_cst`；`atomic_flag_clear` 同样。显式版本更清晰。
- `pause`：告诉 CPU 这是自旋，减少流水线/总线压力、降低功耗；ARM 用 `yield`（`__asm__("yield")`）。
- `sched_yield()` 是系统调用，开销远大于几十个周期的自旋，**只适合长时间持锁**的场景；短临界区里自旋 + `pause` 更好。

### 4. 计数器

```c
int counter = 0;
counter++;   /* 多线程 */
```

x86-64 典型汇编：

```asm
mov eax, DWORD PTR [counter]
add eax, 1
mov DWORD PTR [counter], eax
```

三条指令之间可能被其他线程插入 → **丢失更新**（lost update）。

```c
atomic_int counter = 0;
atomic_fetch_add(&counter, 1);   /* x86: lock xadd [counter], 1 */
```

- `lock` 前缀锁缓存行/总线，保证读-改-写的原子性；
- 若返回值不用，编译器可用 `lock add`。
- `_Atomic int` 的 `counter++` 也是原子 RMW（seq_cst），返回旧值；`atomic_fetch_add_explicit` 可指定内存序（统计场景用 `memory_order_relaxed` 足够）。

### 5. SPSC 环形队列

```c
#include <stdatomic.h>
#include <stdbool.h>
#include <stddef.h>

typedef struct {
    _Alignas(64) atomic_size_t head;   /* 生产者写 */
    _Alignas(64) atomic_size_t tail;   /* 消费者写 */
    _Alignas(64) int *buf;
    size_t mask;                       /* capacity - 1 */
} ring_t;

bool ring_init(ring_t *r, int *storage, size_t capacity)
{
    if (capacity == 0 || (capacity & (capacity - 1)) != 0)
        return false;
    r->buf  = storage;
    r->mask = capacity - 1;
    atomic_init(&r->head, 0);
    atomic_init(&r->tail, 0);
    return true;
}

bool ring_push(ring_t *r, int v)
{
    size_t h = atomic_load_explicit(&r->head, memory_order_relaxed);
    size_t t = atomic_load_explicit(&r->tail, memory_order_acquire);
    if (h - t > r->mask)          /* h - t == capacity → 满 */
        return false;
    r->buf[h & r->mask] = v;
    atomic_store_explicit(&r->head, h + 1, memory_order_release);
    return true;
}

bool ring_pop(ring_t *r, int *v)
{
    size_t t = atomic_load_explicit(&r->tail, memory_order_relaxed);
    size_t h = atomic_load_explicit(&r->head, memory_order_acquire);
    if (h == t)                    /* 空 */
        return false;
    *v = r->buf[t & r->mask];
    atomic_store_explicit(&r->tail, t + 1, memory_order_release);
    return true;
}
```

要点：

- **head 只由生产者写，tail 只由消费者写**，所以各自 load 自己的索引可以用 `relaxed`（只有本线程写，读到的不会更旧）。
- 读对方的索引用 `acquire`，与对方写索引用 `release` 配对，保证「数据写入」先于「索引发布」。
- **满/空判定**：无符号自由运行计数器（free-running），容量 N（2 的幂）。`h - t` 在模 2^64 下始终等于「当前元素个数」，只要 N ≤ 2^63 就不会歧义。`h - t == N` 满，`h == t` 空。
- 用 `mask` 取模，因为 N 是 2 的幂。
- **伪共享**：`head` 和 `tail` 若在同一 cache line，生产者写 head 会让消费者核的 tail 缓存行失效，来回 ping-pong。`_Alignas(64)` 把两者隔到不同 cache line（64 字节是 x86 典型行大小，ARM 常见 64/128）。
- 队列本身是 SPSC；多生产者需要额外的 CAS 或锁。

### 6. ABA

无锁栈：

```c
typedef struct node { int v; struct node *next; } node_t;
_Atomic(node_t *) top;

node_t *pop(void)
{
    node_t *n, *next;
    do {
        n = atomic_load(&top);
        if (!n)
            return NULL;
        next = n->next;
    } while (!atomic_compare_exchange_weak(&top, &n, next)); /* 失败时 n 被更新 */
    free(n);
    return n;
}
```

交错：

1. 线程 A 读 `top = X`，`next = Y`；
2. 线程 B pop X、pop Y、push X（`X->next` 现在指向 Z）；
3. 线程 A 的 CAS 成功（`top` 仍是 X），把 `top` 设为 Y —— 但 Y 已经被 free，且 X 的 next 已经变了 → 栈损坏/use-after-free。

解决方案：

- **tagged pointer**：把版本号打包进指针高位（要求指针有未使用位或 16 字节对齐），CAS 同时比较指针+版本；
- **hazard pointer**：读者发布「我正在访问的指针」，回收者延迟释放；
- **epoch / RCU**：按代延迟回收，读侧几乎零开销；
- 代价：tagged pointer 受地址空间限制；hazard pointer 每次读要写共享内存；RCU 需要宽限期。

### 7. 内存序选择

- `relaxed`：只保证原子性，不保证顺序。适合**统计计数器**（`atomic_fetch_add_explicit(&cnt, 1, relaxed)`），因为不需要与其他内存操作排序。
- 不适合：发布数据（必须 release/acquire）、实现锁、Dekker 互斥。
- `seq_cst`：全局单一顺序。x86 上 load 是普通 `mov`，store 是普通 `mov`，RMW 用 `lock` 前缀；只有需要 StoreLoad 屏障时才 `mfence`。ARM 上需要 `dmb ish` / `ldar` / `stlr`。

### 追问

- `atomic_is_lock_free`：某些平台的大原子类型会用锁实现，实时性不保证。
- 自旋锁不可重入、不可睡眠；临界区不能调用可能阻塞的函数。
- `volatile` + `atomic` 一起用没有意义，`atomic` 已经包含必要语义。

---

## 第 10 题：预处理宏与元编程

### 思路

宏展开分三步：**参数替换 → `#`/`##` 处理 → 重新扫描**。所有坑都来自这三步的顺序：

- `#` 和 `##` 的操作数**不会**先展开；
- 宏参数若在体内出现多次，就会**求值多次**；
- 宏没有类型，也没有作用域。

### 1. SQUARE 的坑

```c
#define SQUARE(x) x * x
SQUARE(1 + 2)   /* 1 + 2 * 1 + 2 = 5，不是 9 */
SQUARE(i++)     /* i++ * i++ → UB */
```

四种改进：

| 方案 | 写法 | 优点 | 缺点 |
|---|---|---|---|
| 加括号 | `((x)*(x))` | 解决优先级 | **解决不了副作用**（`i++` 仍两次） |
| 内联函数 | `static inline int square(int x)` | 类型安全、单次求值、可调试 | 只支持一种类型 |
| `_Generic` | 按类型分派到不同函数 | 多类型、类型安全 | 需要为每种类型写函数 |
| GNU 语句表达式 | `({ __typeof__(x) _x = (x); _x * _x; })` | 单次求值、保留类型 | 非标准、`__typeof__` 是扩展 |

```c
#define SQUARE(x) ({ __typeof__(x) _x = (x); _x * _x; })
```

C23 有 `typeof`，但语句表达式 `({ ... })` 仍是 GNU 扩展（Clang 支持，MSVC 不支持）。

### 2. MAX 的坑

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
MAX(a++, b++)   /* 较大的那个被自增两次 */
```

类型问题：

```c
int m = MAX(-1, 1u);   /* 条件里 -1 转 unsigned → 假；结果取 1u；m = 1 */
```

更隐蔽的：

```c
unsigned u = 0;
int i = -1;
MAX(i, u)   /* 结果类型是 unsigned，i 被转成 UINT_MAX */
```

修正：

```c
#define MAX(a, b) ({ \
    __typeof__(a) _a = (a); \
    __typeof__(b) _b = (b); \
    _a > _b ? _a : _b; \
})
```

但 `_a > _b` 仍会做寻常算术转换；要真正类型安全，需要 `_Generic` 或 C++ 的模板。

### 3. do-while(0)

```c
#define LOG(x) do { fprintf(stderr, "%s\n", x); } while (0)
```

三种宏体的展开：

```c
/* 裸语句 */
#define M() a(); b()
if (c) M(); else ...;   /* 展开为 if (c) a(); b(); else ... → else 悬空/语法错误 */

/* 花括号 */
#define M() { a(); b(); }
if (c) M(); else ...;   /* 展开为 if (c) { ... }; else → 分号变成空语句，else 无匹配 if */

/* do-while(0) */
#define M() do { a(); b(); } while (0)
if (c) M(); else ...;   /* 展开为 if (c) do {...} while(0); else → 合法 */
```

`do-while(0)` 是一个**单条语句**，末尾需要分号，能安全地放在 `if/else`、`while` 体里。唯一的小坑：宏体里的 `break`/`continue` 只会作用于这个 `while(0)`，不会跳出外层循环——所以宏里不要用 `break`。

### 4. `#` 与两层宏

```c
#define STR(x) #x
STR(FOO)      /* "FOO"，不会展开 FOO */
STR(1 + 2)    /* "1 + 2" */
```

`#` 的操作数是**宏参数的原样 token 序列**，不做宏展开（6.10.3.2）。要让参数先展开：

```c
#define STRINGIFY(x) #x
#define STR(x) STRINGIFY(x)   /* STR(FOO) → STRINGIFY(42) → "42" */
```

原理：`STR` 的参数 `FOO` 在替换进 `STRINGIFY(x)` 时先被展开为 `42`，然后 `STRINGIFY` 的 `#x` 把 `42` 变成 `"42"`。

### 5. `##` 与两层 CAT

```c
#define CAT(a, b) a##b
CAT(FOO, BAR)   /* FOOBAR，即使 FOO/BAR 有定义也不展开 */
```

6.10.3.3：紧邻 `##` 的参数**不展开**，直接粘贴。正确写法：

```c
#define CAT_(a, b) a##b
#define CAT(a, b) CAT_(a, b)

#define FOO 1
#define BAR 2
CAT(FOO, BAR)   /* → CAT_(1, 2) → 12 */
```

`CAT(1, 2)` → `12`，是一个合法的 **pp-number**（预处理数字），可以当整数用。
`CAT(FOO, BAR)` 在两者未定义时 → 标识符 `FOOBAR`（未声明，编译期报错或当外部符号）。

### 6. offsetof / container_of / BUILD_BUG_ON

```c
/* 标准库已有；这是教学实现 */
#define OFFSETOF(type, member) ((size_t)&(((type *)0)->member))

#define CONTAINER_OF(ptr, type, member) \
    ((type *)((char *)(ptr) - OFFSETOF(type, member)))

/* C11 之前 */
#define BUILD_BUG_ON(cond) \
    typedef char build_bug_on_[(cond) ? -1 : 1]

/* C11 之后 */
#define BUILD_BUG_ON(cond) _Static_assert(!(cond), #cond)
```

旧式 `typedef char name[(cond) ? -1 : 1];` 的原理：条件为真时数组长度为 -1 → 约束违反 → 编译错误；为假时长度为 1 → 合法。缺点：

- 只能出现在**声明**能出现的地方（文件作用域、块作用域、结构体成员）；
- 错误信息不直观（`size of array is negative`）；
- 不能带消息；
- 同一个作用域里多次使用会重名（需要 `__LINE__` 拼接）。

`_Static_assert`（6.7.10）：标准、可带消息、可用于文件/块/结构体作用域，C11 起可用。C23 起 `static_assert` 成为关键字（不必再写 `_Static_assert`），但 `assert` 仍然是 `<assert.h>` 里的宏。

### 7. `_Generic` 与 SWAP

```c
#include <stdio.h>

static void print_int(int v)       { printf("%d\n", v); }
static void print_double(double v) { printf("%f\n", v); }
static void print_str(const char *v) { printf("%s\n", v); }

#define PRINT(x) _Generic((x), \
    int: print_int, \
    double: print_double, \
    char *: print_str, \
    const char *: print_str \
)(x)
```

注意：`_Generic` 的**所有分支都必须是合法表达式**（即使没被选中也要能通过类型检查），所以不能把不兼容的调用写进分支。

```c
#define SWAP(a, b) do { \
    __typeof__(a) _tmp = (a); \
    (a) = (b); \
    (b) = _tmp; \
} while (0)
```

- 任意类型（包括结构体），只要 `a`、`b` 类型兼容；
- 每个参数只求值一次；
- 需要 `__typeof__`（GCC/Clang 扩展，C23 标准化为 `typeof`）；
- 不能用 `memcpy` 版本替代：需要 `sizeof(a) == sizeof(b)` 且没有临时缓冲区的大小信息；
- 异或交换 `a ^= b; b ^= a; a ^= b;` 只适用于整数，且 `a`、`b` 是同一对象时会被清零。

### 8. X-macro 与变参宏

```c
#define ERROR_TABLE(X) \
    X(OK,     0) \
    X(EINVAL, 1) \
    X(ENOMEM, 2)

#define AS_ENUM(name, val) name = val,
enum err_code {
    ERROR_TABLE(AS_ENUM)
};

#define AS_STRING(name, val) [name] = #name,
static const char *const err_name[] = {
    ERROR_TABLE(AS_STRING)
};
```

新增错误码只改 `ERROR_TABLE` 一处，enum 和字符串表自动同步。指定初始化器 `[name] = ...` 保证下标与枚举值一致。

变参宏：

```c
#define LOG(fmt, ...) fprintf(stderr, fmt, __VA_ARGS__)
LOG("hello");                 /* C99：空 __VA_ARGS__ 是约束违反（尾逗号） */
#define LOG(fmt, ...) fprintf(stderr, fmt, ##__VA_ARGS__)  /* GNU 扩展：空时去掉逗号 */
#define LOG(fmt, ...) fprintf(stderr, fmt __VA_OPT__(,) __VA_ARGS__) /* C23 */
```

`#if` 中不能用 `sizeof`：预处理阶段类型系统还不存在；`#if` 的算术用 `intmax_t`/`uintmax_t`，未定义标识符按 0 处理，不能用浮点、字符串、强制转换。

### 追问

- `#include` 守卫 vs `#pragma once`：前者标准、可移植；后者非标准但几乎所有编译器支持，且对「同一文件不同路径」更鲁棒。
- 宏不能递归展开（`#define A A` 不会无限展开，预处理器会标记「不再展开」）。
- `__LINE__`、`__FILE__`、`__func__`（后者是函数内的静态数组，不是宏）。
- `offsetof` 在 C++ 里对非标准布局类型有条件限制；C 里对位域不合法。

---

## 自测评分表

| 题号 | 及格（能说清结论） | 良好（能引条款+写代码） | 优秀（能讲编译器/ABI 行为与工程取舍） |
|---|---|---|---|
| 1 | 指出 ①②③④⑤ 是 UB | 引 6.5p2，解释 UB 时间旅行 | 讲清 UBSan 边界、C/C++17 差异 |
| 2 | 算出 ①②③ | 讲清提升/转换规则 | 手写溢出安全乘法、跨数据模型 |
| 3 | 算出 sizeof | 讲清位域/FAM 限制 | container_of 的 UB 与类型检查 |
| 4 | 解析声明符 | 讲清退化与多维数组 | restrict/const 层次、函数指针表 |
| 5 | 指出 free(buf+4) 错 | 讲清 realloc/对齐 | 手写对齐分配器、effective type |
| 6 | 讲清 memcpy/memmove | 写 memmove、Duff | 字长优化 + 别名/越界分析 |
| 7 | 解释 undefined reference | 讲 C99 inline 模型 | common symbol、静态库顺序 |
| 8 | 说出 volatile 的作用 | 引 7.13.2.1 | 信号安全、TRY/CATCH 设计 |
| 9 | 说出 data race | 写自旋锁/原子加 | SPSC 队列 + 内存序 + ABA |
| 10 | 说出 SQUARE 的坑 | 写 do-while/两层宏 | _Generic、X-macro、C23 变参 |

**判定**：6 题达到「良好」≈ 大厂 C/C++ 三面通过线；8 题达到「优秀」≈ 可面基础架构/编译器/内核方向。

---

## 参考资料

- ISO/IEC 9899:2011（C11）及 N1570 草案：<https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf>
- cppreference（C 语言/原子/`setjmp`）：<https://en.cppreference.com/w/c>
- SEI CERT C Coding Standard（UB、整数、内存规则）：<https://wiki.sei.cmu.edu/confluence/display/c>
- GCC 手册（`-fno-common`、inline 语义、`-fsanitize`）：<https://gcc.gnu.org/onlinedocs/gcc/>
- Linux 内核 `container_of` 与 `include/linux/kernel.h`：<https://www.kernel.org/doc/html/latest/>
- "What Every C Programmer Should Know About Undefined Behavior"（LLVM 博客）：<https://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html>
- Bit Twiddling Hacks（haszero）：<https://graphics.stanford.edu/~seander/bithacks.html>
