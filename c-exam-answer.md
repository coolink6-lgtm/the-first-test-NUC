# C 语言程序设计 · 大一期末考核题 · 参考答案与评分要点

> 对应 `c-exam-question.md`。每题给出：**参考答案 → 完整代码 → 评分要点 → 常见错误**。
> 代码统一用 C99，输出提示用英文；所有示例都假设在 `x86-64 + GCC` 下编译（`int` 4 字节、`double` 8 字节）。
> 评分说明：编程题按“能编译（30%）+ 结果正确（50%）+ 边界与可读性（20%）”给分；不同教师可自行微调。

---

## 第 1 题：数据类型、运算符与表达式（8 分）

### 参考答案

**1. 完整输出**

```text
3 1 3.5
1 3 3
5
65 B
2.0
1
```

逐行解释：

| 输出 | 原因 |
|---|---|
| `3 1 3.5` | `7/2` 是整数除法得 3；`7%2` 得 1；`7/2.0` 中 `2.0` 是 `double`，`7` 被转成 `7.0`，结果 3.5 |
| `1 3 3` | `x = i++` 先把 1 赋给 `x`，再让 `i` 变 2；`y = ++i` 先让 `i` 变 3，再把 3 赋给 `y` |
| `5` | `n` 为 0，`&&` 短路，右边的 `++m` 根本不执行 |
| `65 B` | 字符 `'A'` 的 ASCII 码是 65；`c + 1` 是整数 66，用 `%c` 输出即 `'B'` |
| `2.0` | `5 / 2` 先做整数除法得 2，再转换成 `double` 赋给 `d` |
| `1` | 先算 `2*3=6`，`1+6=7`，`7>6` 为真（1）；`4!=3` 为真（1）；`1 && 1` 得 1 |

**2. `a / b` 与 `a / 2.0`**

`a`、`b` 都是 `int`，`/` 做整数除法，商向零取整，小数部分被丢弃；`2.0` 是 `double`，根据“寻常算术转换”，`a` 先转成 `double`，再做浮点除法。

**3. `i++` 与 `++i`**

- `i++`（后置）：**先取值，后自增**，表达式的值是自增前的值；
- `++i`（前置）：**先自增，后取值**，表达式的值是自增后的值。

本题 `x = i++` 得 1（`i` 变 2），`y = ++i` 得 3（`i` 变 3）。

**4. `m` 为什么仍是 5**

`&&` 是短路与：左操作数 `n` 为 0（假）时，整个表达式已经确定为假，右边的 `++m` 不会被求值，所以 `m` 保持 5。同理 `||` 在左边为真时也会短路。

**5. `d` 为什么是 2.0**

`5 / 2` 两边都是 `int`，先做整数除法得到 2，然后才转换成 `double` 赋给 `d`。要得到 2.5，至少把一边写成浮点数：

```c
double d = 5.0 / 2;      /* 或 5 / 2.0，或 (double)5 / 2 */
```

### 评分要点

- 输出全对得 4 分；每错一行扣 1 分，扣完为止。
- 2～5 小问各 1 分，意思对即可，不要求背术语。

### 常见错误

- 把 `7/2.0` 写成 3（忽略类型转换）；
- 认为 `i++` 和 `++i` 都先加 1；
- 认为 `&&` 两边一定都会执行；
- 把 `d` 算成 2.5（忽略 `5/2` 先做整数除法）。

---

## 第 2 题：选择结构与循环结构（8 分）

### 参考答案

**1. 输出**

```text
s=315
rev=4321
   *
  ***
 *****
*******
```

- `s`：1～100 中同时是 3 和 5 的倍数，即 15 的倍数：15+30+45+60+75+90 = **315**。
- `rev`：把 1234 的各位数字反转，得到 **4321**。
- 图形：4 行金字塔，第 `i` 行先输出 `4-i` 个空格，再输出 `2*i-1` 个 `*`。

**2. 求 1! + 2! + … + 10!**

```c
#include <stdio.h>

int main(void)
{
    int i;
    long long fact = 1, sum = 0;   /* 10! 不大，int 也够，但用 long long 更稳 */

    for (i = 1; i <= 10; i++) {
        fact *= i;                 /* fact = i! */
        sum += fact;
    }
    printf("sum = %lld\n", sum);   /* 4037913 */
    return 0;
}
```

要点：**用一个变量滚动保存当前的阶乘**，不要每次都从 1 重新乘一遍（那样也能做，但效率低，而且容易写错循环边界）。

**3. 判断素数并输出 100 以内的素数**

```c
#include <stdio.h>

int is_prime(int n)
{
    int i;
    if (n < 2)
        return 0;
    for (i = 2; i * i <= n; i++)
        if (n % i == 0)
            return 0;
    return 1;
}

int main(void)
{
    int n, cnt = 0;
    for (n = 2; n <= 100; n++) {
        if (is_prime(n)) {
            printf("%4d", n);
            if (++cnt % 10 == 0)
                printf("\n");
        }
    }
    if (cnt % 10 != 0)
        printf("\n");
    return 0;
}
```

要点：循环条件写 `i * i <= n` 比 `i < n` 快得多；`n < 2` 要单独处理（0、1 不是素数）。

**4. `break` 与 `continue`**

- `break`：**立即结束整个循环**（或 `switch`），跳到循环后面的语句；
- `continue`：**结束本次循环**，跳过循环体中剩余的语句，直接进入下一次循环的条件判断。

例子：在 1～100 中找第一个能被 7 整除的数，找到后用 `break`；输出 1～100 中所有不能被 3 整除的数，用 `continue` 跳过能被 3 整除的情况。

### 评分要点

- 输出 3 分：`s`、`rev`、图形各 1 分。
- 第 2 小问 2 分：循环正确 1 分，结果正确 1 分。
- 第 3 小问 2 分：`is_prime` 1 分，输出格式（每行 10 个）1 分。
- 第 4 小问 1 分：区别说清即可。

### 常见错误

- 金字塔的空格数或星号数算错（常见是把 `2*i-1` 写成 `2*i`）；
- 素数判断从 `i = 1` 开始，导致所有数都返回“不是素数”；
- `if (++cnt % 10 == 0)` 写成 `if (cnt % 10 == 0)`，导致第一行只输出 9 个；
- 把 `continue` 说成“结束整个循环”。

---

## 第 3 题：一维数组（10 分）

### 参考答案

**1. 最大值、最小值、平均值**

```c
#include <stdio.h>

int main(void)
{
    int a[8] = {5, 3, 9, 1, 7, 2, 8, 4};
    int i, max = a[0], min = a[0], sum = 0;

    for (i = 1; i < 8; i++) {
        if (a[i] > max) max = a[i];
        if (a[i] < min) min = a[i];
    }
    for (i = 0; i < 8; i++)
        sum += a[i];

    printf("max=%d min=%d avg=%.2f\n", max, min, sum / 8.0);
    return 0;
}
```

输出：`max=9 min=1 avg=4.88`。

注意：`sum / 8.0` 才是浮点除法；`sum / 8` 会得到 4。

**2. 原地逆置**

```c
void reverse(int a[], int n)
{
    int i, j, t;
    for (i = 0, j = n - 1; i < j; i++, j--) {
        t = a[i];
        a[i] = a[j];
        a[j] = t;
    }
}
```

调用后：`4 8 2 7 1 9 3 5`。

要点：只能交换到 `i < j` 为止，写成 `i <= j` 也不会错（中间元素自己和自己交换），但 `i < n` 就会把数组又换回去。

**3. 冒泡排序**

```c
void bubble_sort(int a[], int n)
{
    int i, j, t;
    for (i = 0; i < n - 1; i++) {
        for (j = 0; j < n - 1 - i; j++) {
            if (a[j] > a[j + 1]) {
                t = a[j];
                a[j] = a[j + 1];
                a[j + 1] = t;
            }
        }
    }
}
```

排序结果：`1 2 3 4 5 7 8 9`。

**4. 二分查找**

```c
int binary_search(int a[], int n, int key)
{
    int low = 0, high = n - 1, mid;
    while (low <= high) {
        mid = low + (high - low) / 2;   /* 防止 low + high 溢出 */
        if (a[mid] == key)
            return mid;
        else if (a[mid] < key)
            low = mid + 1;
        else
            high = mid - 1;
    }
    return -1;
}
```

对排序后的数组查找 7，返回下标 **5**（下标从 0 开始）。找不到返回 -1。

要点：二分查找的前提是**数组已经有序**；循环条件必须是 `low <= high`。

**5. 删除下标为 k 的元素**

```c
int delete_at(int a[], int n, int k)
{
    int i;
    if (k < 0 || k >= n)
        return n;                 /* 下标非法，原样返回 */
    for (i = k; i < n - 1; i++)
        a[i] = a[i + 1];          /* 后面的元素依次前移 */
    return n - 1;                 /* 有效元素个数减 1 */
}
```

删除后，数组的**有效长度**从 `n` 变成 `n-1`；最后一个位置上的旧值仍然存在，但已经不属于有效数据，不需要（也不应该）去“清空”它。

### 评分要点

- 第 1 小问 3 分：最值 2 分，平均值 1 分（`/8.0` 写错扣 1 分）。
- 第 2 小问 2 分：交换逻辑正确即可。
- 第 3 小问 2 分：两层循环边界正确 1 分，结果正确 1 分。
- 第 4 小问 2 分：`low <= high` 与 `mid` 更新正确即可。
- 第 5 小问 1 分：说出“有效长度减 1、元素前移”即可。

### 常见错误

- 求平均值时写成 `sum / 8`；
- 逆置循环写成 `for (i = 0; i < n; i++)`，交换两次等于没换；
- 冒泡内层循环写成 `j < n - 1`，导致越界访问 `a[n]`；
- 二分查找前忘记排序；
- 删除元素后忘记返回 `n-1`，或者以为数组真的“变短”了（数组长度在定义时就固定了）。

---

## 第 4 题：二维数组与矩阵（10 分）

### 参考答案

**1. 遍历输出**

```c
int i, j;
for (i = 0; i < 3; i++) {
    for (j = 0; j < 3; j++)
        printf("%d ", a[i][j]);
    printf("\n");
}
```

**2. 原地转置**

```c
void transpose(int a[3][3])
{
    int i, j, t;
    for (i = 0; i < 3; i++) {
        for (j = i + 1; j < 3; j++) {   /* 只遍历上三角 */
            t = a[i][j];
            a[i][j] = a[j][i];
            a[j][i] = t;
        }
    }
}
```

转置结果：

```text
1 4 7
2 5 8
3 6 9
```

要点：内层循环从 `j = i + 1` 开始，只交换一次；如果从 `j = 0` 开始，每个元素会被交换两次，矩阵又变回原样。

**3. 两条对角线之和**

```c
int main_diag = 0, anti_diag = 0;
for (i = 0; i < 3; i++) {
    main_diag += a[i][i];           /* 主对角线：行下标 == 列下标 */
    anti_diag += a[i][3 - 1 - i];   /* 副对角线：列下标 == n-1-行下标 */
}
printf("main=%d anti=%d\n", main_diag, anti_diag);
```

结果：`main=15 anti=15`（1+5+9 与 3+5+7）。

**4. 杨辉三角前 10 行**

```c
#include <stdio.h>

int main(void)
{
    int a[10][10] = {0};
    int i, j;

    for (i = 0; i < 10; i++) {
        a[i][0] = 1;
        a[i][i] = 1;
        for (j = 1; j < i; j++)
            a[i][j] = a[i - 1][j - 1] + a[i - 1][j];
    }

    for (i = 0; i < 10; i++) {
        for (j = 0; j <= i; j++)
            printf("%4d", a[i][j]);
        printf("\n");
    }
    return 0;
}
```

要点：每行第一个和最后一个都是 1，其余等于“上一行左边 + 上一行正上方”。

**5. 矩阵乘法**

```c
#include <stdio.h>

int main(void)
{
    int A[2][3] = {{1, 2, 3}, {4, 5, 6}};
    int B[3][2] = {{7, 8}, {9, 10}, {11, 12}};
    int C[2][2] = {0};
    int i, j, k;

    for (i = 0; i < 2; i++)
        for (j = 0; j < 2; j++)
            for (k = 0; k < 3; k++)
                C[i][j] += A[i][k] * B[k][j];

    for (i = 0; i < 2; i++) {
        for (j = 0; j < 2; j++)
            printf("%d ", C[i][j]);
        printf("\n");
    }
    return 0;
}
```

结果：

```text
58 64
139 154
```

要点：`C` 的行数等于 `A` 的行数，列数等于 `B` 的列数；三层循环的顺序可以交换，但 `C[i][j]` 必须先清零。

### 评分要点

- 第 1 小问 1 分：双重循环 + 换行正确即可。
- 第 2 小问 3 分：`j = i + 1` 2 分，输出正确 1 分。
- 第 3 小问 2 分：两个求和各 1 分。
- 第 4 小问 2 分：边界 1 和递推式各 1 分。
- 第 5 小问 2 分：三层循环 1 分，结果正确 1 分。

### 常见错误

- 转置时内层从 0 开始，交换两次；
- 副对角线写成 `a[i][i]`（和主对角线搞混）；
- 杨辉三角忘记初始化 `a[i][i] = 1`；
- 矩阵乘法忘记初始化 `C`，或者循环边界写反（`k < 2`）。

---

## 第 5 题：函数、递归与变量作用域（10 分）

### 参考答案

**1. 输出**

```text
1 2
120
1 2 3
```

**2. 为什么 `swap` 没交换**

C 语言函数参数是**值传递**：调用 `swap(a, b)` 时，把 `a`、`b` 的值复制给形参 `x`、`y`，函数里交换的是副本，`main` 中的 `a`、`b` 不受影响。

改成传地址：

```c
void swap(int *px, int *py)
{
    int t = *px;
    *px = *py;
    *py = t;
}

/* 调用 */
swap(&a, &b);
```

**3. `fact(5)` 的递归过程**

```text
fact(5) = 5 * fact(4)
        = 5 * 4 * fact(3)
        = 5 * 4 * 3 * fact(2)
        = 5 * 4 * 3 * 2 * fact(1)
        = 5 * 4 * 3 * 2 * 1
        = 120
```

终止条件：`n <= 1` 时直接返回 1。每次递归调用都把问题规模减 1，最终到达终止条件。

**4. 汉诺塔**

```c
void hanoi(int n, char from, char aux, char to)
{
    if (n == 1) {
        printf("%c -> %c\n", from, to);
        return;
    }
    hanoi(n - 1, from, to, aux);   /* 先把上面 n-1 个移到辅助柱 */
    printf("%c -> %c\n", from, to);/* 把最大的盘子移到目标柱 */
    hanoi(n - 1, aux, from, to);   /* 再把 n-1 个移到目标柱 */
}
```

移动次数：`T(n) = 2*T(n-1) + 1`，`T(1) = 1`，解得 **T(n) = 2^n - 1**。

**5. `static` 局部变量**

普通局部变量 `int c` 存放在栈上，函数每次调用都会重新创建、重新初始化；`static int c` 存放在静态存储区，**只初始化一次**，函数返回后它的值仍然保留。所以三次调用分别输出 1、2、3。

### 评分要点

- 第 1 小问 2 分：三行各占一部分，全对得 2 分。
- 第 2 小问 2 分：说清“值传递”1 分，改对代码 1 分。
- 第 3 小问 2 分：过程 1 分，终止条件 1 分。
- 第 4 小问 2 分：代码 1 分，`2^n - 1` 1 分。
- 第 5 小问 2 分：存储位置/生命周期说清即可。

### 常见错误

- 认为 `swap` 里交换了，`main` 里就会变；
- `fact` 忘记写终止条件，导致无限递归、栈溢出；
- 汉诺塔的三步顺序写错（尤其是辅助柱和目标柱的实参顺序）；
- 把 `static` 局部变量说成“全局变量”（它仍然只在函数内可见）。

---

## 第 6 题：指针基础（10 分）

### 参考答案

**1. 输出与逐步分析**

```text
10
20
21
22
10 22 30
```

| 步骤 | 表达式 | 结果 | 之后的状态 |
|---|---|---|---|
| ① | `*p++` | 10 | 先取 `*p`（10），再 `p++`，`p` 指向 `a[1]` |
| ② | `(*p)++` | 20 | 取 `a[1]` 的值 20，再把 `a[1]` 加 1 变成 21；`p` 不动 |
| ③ | `*p` | 21 | `p` 仍指向 `a[1]` |
| ④ | `++*p` | 22 | 先 `*p = *p + 1`，`a[1]` 变成 22，表达式值为 22 |

最后一行 `a[0] a[1] a[2]` 为 `10 22 30`。

记忆口诀：`*p++` 是“取星号，指针后移”；`(*p)++` 是“取星号，值加一”；`++*p` 是“值先加一，再取星号”。

**2. 指针版 swap**

```c
void swap(int *px, int *py)
{
    int t = *px;
    *px = *py;
    *py = t;
}

/* 调用 */
int a = 1, b = 2;
swap(&a, &b);        /* 传地址 */
printf("%d %d\n", a, b);   /* 2 1 */
```

**3. `a[i]` 与 `*(a + i)`**

两者完全等价：`a[i]` 在编译时就是按 `*(a + i)` 处理的。`&a[i]` 与 `a + i` 也等价，都是第 `i` 个元素的地址。

**4. 指针遍历求和**

```c
int sum_array(int *p, int n)
{
    int i, sum = 0;
    for (i = 0; i < n; i++)
        sum += *(p + i);      /* 也可写成 sum += p[i] */
    return sum;
}

/* 调用 */
int a[5] = {1, 2, 3, 4, 5};
printf("%d\n", sum_array(a, 5));   /* 15 */
```

**5. 空指针与野指针**

- **空指针**：值为 `NULL` 的指针，明确表示“不指向任何对象”；
- **野指针**：未初始化或指向已释放内存的指针，值是不确定的。

野指针的解引用是未定义行为，可能崩溃、可能读到垃圾数据。定义指针后立即初始化为 `NULL`，使用前判断 `if (p != NULL)`，可以避免大部分误用。

### 评分要点

- 第 1 小问 4 分：四行输出各 1 分（最后一行数组值包含在 4 分内也可）。
- 第 2 小问 2 分：函数 1 分，调用 1 分。
- 第 3 小问 1 分。
- 第 4 小问 2 分。
- 第 5 小问 1 分：两个概念说清即可。

### 常见错误

- 把 `*p++` 理解成 `(*p)++`；
- 以为 `(*p)++` 会移动指针；
- 函数里写 `swap(a, b)` 而不是 `swap(&a, &b)`；
- 认为 `a` 和 `&a` 完全一样（值相同，类型不同）。

---

## 第 7 题：字符数组与字符串（12 分）

### 参考答案

**1. sizeof 与 strlen**

| 表达式 | 结果 | 说明 |
|---|---|---|
| `sizeof(s1)` | 6 | 数组大小，包含结尾的 `'\0'`（h e l l o \0） |
| `sizeof(s2)` | 10 | 数组声明为 `char[10]`，后面 5 个字节补 0 |
| `strlen(s1)` | 5 | 字符串长度，不包含 `'\0'` |

`s3` 指向字符串字面量 `"hello"`，**不能修改**（例如 `s3[0] = 'H';` 是未定义行为，通常直接崩溃）。字符串字面量通常存放在只读段。

**2. 手写 `strlen` 与 `strcpy`**

```c
int my_strlen(const char *s)
{
    int n = 0;
    while (s[n] != '\0')
        n++;
    return n;
}

void my_strcpy(char *dst, const char *src)
{
    int i = 0;
    while ((dst[i] = src[i]) != '\0')
        i++;
}
```

要点：`strcpy` 的循环条件里“赋值 + 判断”合在一起，最后会把 `'\0'` 也复制过去。

**3. 原地逆置**

```c
void reverse_str(char *s)
{
    int i = 0, j = my_strlen(s) - 1;
    char t;
    while (i < j) {
        t = s[i];
        s[i] = s[j];
        s[j] = t;
        i++;
        j--;
    }
}
```

**4. 回文判断**

```c
int is_palindrome(const char *s)
{
    int i = 0, j = my_strlen(s) - 1;
    while (i < j) {
        if (s[i] != s[j])
            return 0;
        i++;
        j--;
    }
    return 1;
}
```

**5. 统计字符**

```c
#include <stdio.h>

int main(void)
{
    char s[200];
    int i, upper = 0, lower = 0, digit = 0, other = 0;

    printf("Input a string: ");
    fgets(s, sizeof s, stdin);

    for (i = 0; s[i] != '\0'; i++) {
        if (s[i] >= 'A' && s[i] <= 'Z')
            upper++;
        else if (s[i] >= 'a' && s[i] <= 'z')
            lower++;
        else if (s[i] >= '0' && s[i] <= '9')
            digit++;
        else if (s[i] != '\n')       /* 不把 fgets 读进来的换行算进去 */
            other++;
    }

    printf("upper=%d lower=%d digit=%d other=%d\n", upper, lower, digit, other);
    return 0;
}
```

注意：`fgets` 会把行末的 `'\n'` 一起读进来，统计“其他字符”时通常要把它排除。

### 评分要点

- 第 1 小问 3 分：三个结果各 1 分，能否修改说清 1 分（可包含在 3 分内）。
- 第 2 小问 3 分：两个函数各 1.5 分。
- 第 3 小问 2 分。
- 第 4 小问 2 分。
- 第 5 小问 2 分：分类正确 1 分，边界（换行）处理 1 分。

### 常见错误

- 把 `sizeof(s1)` 写成 5；
- `strlen` 把 `'\0'` 也算进去；
- `my_strcpy` 忘记复制结尾的 `'\0'`；
- 回文循环写成 `i <= j`（中间字符自己比较，不影响结果，但不是问题）；
- 用 `scanf("%s", s)` 读带空格的字符串，读到空格就停了。

---

## 第 8 题：结构体（12 分）

### 参考答案

**1. 定义与初始化**

```c
typedef struct {
    char name[20];
    int  age;
    double score;
} Student;

Student s = {"Tom", 18, 85.5};
```

**2. 输入 3 个学生，输出平均分和最高分**

```c
#include <stdio.h>

typedef struct {
    char name[20];
    int  age;
    double score;
} Student;

int main(void)
{
    Student s[3];
    int i, best = 0;
    double sum = 0;

    for (i = 0; i < 3; i++) {
        printf("Input name age score: ");
        scanf("%19s %d %lf", s[i].name, &s[i].age, &s[i].score);
    }

    for (i = 0; i < 3; i++) {
        sum += s[i].score;
        if (s[i].score > s[best].score)
            best = i;
    }

    printf("average = %.1f\n", sum / 3);
    printf("best: %s %d %.1f\n", s[best].name, s[best].age, s[best].score);
    return 0;
}
```

**3. 按成绩降序排序**

```c
void sort_by_score(Student s[], int n)
{
    int i, j;
    Student t;
    for (i = 0; i < n - 1; i++) {
        for (j = 0; j < n - 1 - i; j++) {
            if (s[j].score < s[j + 1].score) {   /* 降序 */
                t = s[j];
                s[j] = s[j + 1];
                s[j + 1] = t;
            }
        }
    }
}
```

要点：结构体变量可以直接用 `=` 整体赋值，交换时不需要逐个成员复制。

**4. 结构体指针访问**

```c
Student *p = &s[0];
printf("%s %.1f\n", p->name, p->score);   /* p->name 等价于 (*p).name */
```

**5. 加分函数与传值/传址**

```c
void add_bonus(Student *s, double bonus)
{
    s->score += bonus;
}

/* 调用 */
add_bonus(&s[0], 5.0);
```

- **传结构体变量**：把整个结构体复制一份给形参，函数内修改不影响调用者，而且结构体大时复制开销大；
- **传结构体指针**：只复制一个地址，函数可以通过指针修改原结构体，效率更高。

### 评分要点

- 第 1 小问 2 分：类型定义 1 分，初始化 1 分。
- 第 2 小问 4 分：输入 1 分，平均分 1 分，最高分 2 分。
- 第 3 小问 3 分：排序逻辑 2 分，降序方向 1 分。
- 第 4 小问 1 分。
- 第 5 小问 2 分：函数 1 分，区别 1 分。

### 常见错误

- `scanf` 读姓名不加宽度限制，导致缓冲区溢出；
- `scanf("%lf", &s[i].score)` 把 `&` 漏掉；
- 排序时只交换了 `score`，没有交换整个结构体（导致姓名和成绩对不上）；
- 用 `p.name` 访问指针（应该是 `p->name` 或 `(*p).name`）。

---

## 第 9 题：文件操作（10 分）

### 参考答案

**1. `"r"`、`"w"`、`"a"` 的区别**

| 模式 | 含义 | 文件不存在时 | 文件已存在时 |
|---|---|---|---|
| `"r"` | 只读 | `fopen` 返回 `NULL` | 从开头读 |
| `"w"` | 只写 | 创建新文件 | **清空**原内容，再写 |
| `"a"` | 追加 | 创建新文件 | 保留原内容，在末尾追加 |

**2. 写入 3 行**

```c
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("data.txt", "w");
    if (fp == NULL) {
        printf("Cannot open file.\n");
        return 1;
    }

    fprintf(fp, "1 Tom 85.5\n");
    fprintf(fp, "2 Jerry 92.0\n");
    fprintf(fp, "3 Alice 78.5\n");

    fclose(fp);
    return 0;
}
```

**3. 逐行读取并统计**

```c
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("data.txt", "r");
    char line[200];
    int lines = 0, chars = 0;

    if (fp == NULL) {
        printf("Cannot open file.\n");
        return 1;
    }

    while (fgets(line, sizeof line, fp) != NULL) {
        fputs(line, stdout);
        lines++;
    }

    /* 重新打开，逐字符统计（包括换行符） */
    fclose(fp);
    fp = fopen("data.txt", "r");
    if (fp == NULL)
        return 1;
    while (fgetc(fp) != EOF)
        chars++;
    fclose(fp);

    printf("lines=%d chars=%d\n", lines, chars);
    return 0;
}
```

说明：也可以在第一次循环里用 `strlen(line)` 累加字符数，但要注意最后一行可能没有换行符。上面的写法更直观，但打开了两次文件；两种都可以。

**4. 复制文件**

```c
#include <stdio.h>

int main(void)
{
    FILE *in = fopen("data.txt", "r");
    FILE *out = fopen("backup.txt", "w");
    int ch;

    if (in == NULL || out == NULL) {
        printf("Cannot open file.\n");
        if (in)  fclose(in);
        if (out) fclose(out);
        return 1;
    }

    while ((ch = fgetc(in)) != EOF)
        fputc(ch, out);

    fclose(in);
    fclose(out);
    return 0;
}
```

要点：`fgetc` 返回 `int`，必须用 `int` 接收，才能正确判断 `EOF`（-1）。

**5. 为什么要检查 `fopen` 和 `fclose`**

- `fopen` 可能因为路径错误、权限不足、磁盘满等原因失败，返回 `NULL`；不检查就直接使用会导致程序崩溃。
- `fclose` 会把缓冲区里尚未写入的数据刷到磁盘并释放文件资源；不关闭可能导致数据丢失（尤其是写完立刻断电/退出）或文件句柄泄漏。

### 评分要点

- 第 1 小问 2 分：三种模式各说清一点即可。
- 第 2 小问 2 分：`fopen` 检查 1 分，写入与关闭 1 分。
- 第 3 小问 3 分：逐行读取 1 分，输出 1 分，统计 1 分。
- 第 4 小问 2 分：循环复制 1 分，关闭文件 1 分。
- 第 5 小问 1 分。

### 常见错误

- 用 `"w"` 打开想读的文件，把内容清空了；
- `fgetc` 的返回值用 `char` 接收，导致 `EOF` 判断失效；
- 忘记 `fclose`；
- 不检查 `fopen` 返回值，文件不存在时直接 `fscanf` 崩溃。

---

## 第 10 题：综合应用——学生成绩管理系统（10 分）

### 参考答案

**1. 数据结构**

```c
#include <stdio.h>
#include <string.h>

#define MAX_STUDENTS 100
#define FILE_NAME    "students.txt"

typedef struct {
    int    id;
    char   name[20];
    double score;
} Student;

static Student students[MAX_STUDENTS];
static int count = 0;          /* 当前学生人数 */
```

用“数组 + 计数器”管理：数组容量固定，`count` 记录当前有效元素个数。优点是实现简单、内存连续、便于遍历和排序；缺点是容量固定，超过 100 人需要扩容（可以改成动态分配，但本题不要求）。

**2. 添加与显示**

```c
static void clear_input(void)
{
    int c;
    while ((c = getchar()) != '\n' && c != EOF)
        ;
}

static void add_student(void)
{
    if (count >= MAX_STUDENTS) {
        printf("The list is full.\n");
        return;
    }

    Student *s = &students[count];
    printf("Input id name score: ");
    if (scanf("%d %19s %lf", &s->id, s->name, &s->score) != 3) {
        printf("Invalid input.\n");
        clear_input();
        return;
    }

    count++;
    printf("Added.\n");
}

static void show_all(void)
{
    int i;
    printf("%-8s %-12s %6s\n", "ID", "NAME", "SCORE");
    for (i = 0; i < count; i++)
        printf("%-8d %-12s %6.1f\n",
               students[i].id, students[i].name, students[i].score);
}
```

**3. 查找与排序**

```c
static int find_by_id(int id)
{
    int i;
    for (i = 0; i < count; i++)
        if (students[i].id == id)
            return i;
    return -1;
}

static void sort_by_score(void)
{
    int i, j;
    Student t;
    for (i = 0; i < count - 1; i++) {
        for (j = 0; j < count - 1 - i; j++) {
            if (students[j].score < students[j + 1].score) {
                t = students[j];
                students[j] = students[j + 1];
                students[j + 1] = t;
            }
        }
    }
    printf("Sorted.\n");
}
```

**4. 保存与读取**

```c
static void save_to_file(void)
{
    FILE *fp = fopen(FILE_NAME, "w");
    int i;

    if (fp == NULL) {
        printf("Cannot open %s for writing.\n", FILE_NAME);
        return;
    }

    for (i = 0; i < count; i++)
        fprintf(fp, "%d %s %.1f\n",
                students[i].id, students[i].name, students[i].score);

    fclose(fp);
    printf("Saved %d record(s).\n", count);
}

static void load_from_file(void)
{
    FILE *fp = fopen(FILE_NAME, "r");

    if (fp == NULL) {
        printf("Cannot open %s for reading.\n", FILE_NAME);
        return;
    }

    count = 0;
    while (count < MAX_STUDENTS &&
           fscanf(fp, "%d %19s %lf",
                  &students[count].id,
                  students[count].name,
                  &students[count].score) == 3) {
        count++;
    }

    fclose(fp);
    printf("Loaded %d record(s).\n", count);
}
```

**5. `main` 与菜单循环**

```c
int main(void)
{
    int choice;

    for (;;) {
        printf("\n1.Add 2.Show 3.Search 4.Sort 5.Save 6.Load 0.Exit\n");
        printf("Choice: ");

        if (scanf("%d", &choice) != 1) {   /* 输入的不是数字 */
            printf("Invalid choice.\n");
            clear_input();
            continue;
        }

        switch (choice) {
        case 1:
            add_student();
            break;
        case 2:
            show_all();
            break;
        case 3: {
            int id, k;
            printf("Input id: ");
            if (scanf("%d", &id) != 1) {
                clear_input();
                break;
            }
            k = find_by_id(id);
            if (k < 0)
                printf("Not found.\n");
            else
                printf("%d %s %.1f\n",
                       students[k].id, students[k].name, students[k].score);
            break;
        }
        case 4:
            sort_by_score();
            break;
        case 5:
            save_to_file();
            break;
        case 6:
            load_from_file();
            break;
        case 0:
            printf("Bye.\n");
            return 0;
        default:
            printf("Invalid choice.\n");
        }
    }
}
```

输入非数字时，`scanf` 返回 0，失败字符会留在缓冲区里；如果不清理，下一次循环会立刻再次失败，形成“死循环刷屏”。所以要用 `clear_input()` 把这一行剩下的字符读掉。

### 评分要点

- 第 1 小问 2 分：结构体 1 分，数组 + 计数器 1 分。
- 第 2 小问 3 分：添加 2 分（含满/非法输入处理），显示 1 分。
- 第 3 小问 2 分：查找 1 分，排序 1 分。
- 第 4 小问 2 分：保存 1 分，读取 1 分（含 `fopen` 检查）。
- 第 5 小问 1 分：菜单循环 + 非法输入处理。

### 常见错误

- 数组越界：`count` 已经等于 100 还继续写；
- `scanf("%s", name)` 不限制长度，姓名超过 19 个字符时溢出；
- 排序只交换 `score`，导致姓名和成绩错位；
- `load_from_file` 不把 `count` 清零，导致重复追加；
- `scanf` 失败后不清理缓冲区，菜单死循环；
- 文件路径写死成绝对路径，换一台机器就找不到文件。

---

## 评分总表（教师用）

| 题号 | 分值 | 核心考点 | 建议给分点 |
|---|---|---|---|
| 1 | 8 | 运算符、类型转换、自增、短路 | 输出 4，简答 4×1 |
| 2 | 8 | 循环、素数、图形、break/continue | 输出 3，编程 4，简答 1 |
| 3 | 10 | 数组遍历、逆置、排序、二分、删除 | 3+2+2+2+1 |
| 4 | 10 | 二维数组、转置、对角线、杨辉、矩阵乘 | 1+3+2+2+2 |
| 5 | 10 | 值传递、递归、Hanoi、static | 2+2+2+2+2 |
| 6 | 10 | 指针运算、指针参数、数组等价 | 4+2+1+2+1 |
| 7 | 12 | 字符串长度、复制、逆置、回文、统计 | 3+3+2+2+2 |
| 8 | 12 | 结构体定义、数组、排序、指针 | 2+4+3+1+2 |
| 9 | 10 | 文件模式、读写、复制、关闭 | 2+2+3+2+1 |
| 10 | 10 | 综合设计、菜单、文件持久化 | 2+3+2+2+1 |
| **合计** | **100** | | |

### 给分尺度建议

- **能编译**：即使结果不对，也给该小问 30% 左右，鼓励学生写出完整结构；
- **结果正确**：核心给分点；
- **边界处理**：数组越界、空指针、`fopen` 失败、输入非法，做到一项加一点；
- **可读性**：变量命名、缩进、注释，可在总分 5 分以内浮动。

### 常见“整题崩”的原因（阅卷提醒）

1. `main` 写成 `void main()`（部分编译器不报错，但不规范）；
2. `scanf` 忘记 `&`；
3. 数组下标从 1 开始，导致越界或漏掉第一个元素；
4. 字符串用 `=` 赋值（`char s[10]; s = "abc";` 编译错误）；
5. 文件操作忘记 `fclose` 或拼错文件名；
6. 结构体排序只交换部分成员。

---

## 附：与“高阶 10 题”的区别

| 维度 | 本套（大一期末） | `question.md`（高阶） |
|---|---|---|
| 目标 | 检验语法、循环、数组、函数、指针、结构体、文件 | 考察标准条款、ABI、UB、并发、链接 |
| 代码规模 | 十几行到一百多行 | 片段为主，强调分析 |
| 答案 | 有唯一正确输出与参考实现 | 需要讨论“标准怎么说、编译器怎么做” |
| 难度 | 课程及格到良好 | 大厂三面及以上 |

> 两套题可以配合使用：本套用于期末考核，高阶那套用于拔尖学生培训或考研/面试拓展。
