ICS Lab1: DataLab 实验报告
一、实验目的

1. 熟悉 C 语言中的位级运算（`! ~ & ^ | + << >>`）。
2. 深入理解二进制的补码表示、算术/逻辑移位、溢出等概念。
3. 理解 IEEE 754 单精度浮点数的位级表示，实现整数与浮点数之间的位级转换。
4. 学会使用 `dlc`（操作数检查器）与 `btest`（测试框架）验证程序的正确性与合法性。

二、实验环境

- 操作系统：Ubuntu（WSL）
- 编译器：gcc（`-m32`，32 位编译，`-fwrapv`）
- 测试工具：`btest`、`dlc`、`check_ops.py`
- 验证命令：
  ```bash
  make clean && make all
  ./btest            # 功能正确性测试
  ./check_ops.py     # 操作数上限检查
  ```

三、实验结果

`./btest` 运行结果（总分 110/110）：

![alt text](<屏幕截图 2026-10-09 083239.png>)

`./check_ops.py bits.c` 运行结果（19 个函数全部通过操作数检查）：

![alt text](<屏幕截图 2026-10-09 083318.png>)

四、各题思路与实现

P1 signMask

要求：返回只有最高位为 1 的掩码 `0x80000000`。

思路：把整数 1 左移 31 位，正好落在符号位。

```c
int signMask(void) {
  return 1 << 31;
}
```
P2 bitXor

要求：只用 `~` 和 `&` 实现异或 `x ^ y`。

思路：异或的布尔展开为 `x^y = (x & ~y) | (~x & y)`。由于不能使用 `|`，用德摩根律把"或"转成"与非"：`a | b = ~(~a & ~b)`，于是
`x^y = ~(~(x & ~y) & ~(~x & y))`。

```c
int bitXor(int x, int y) {
	return ~(~(x & ~y) & ~(~x & y));
}
```
P3 negativePart

要求：若 `x < 0` 返回 `-x`，否则返回 0。

思路：`mask = x >> 31` 在负数时为全 1（`-1`），非负时为全 0。补码取负是 `~x + 1`。用 `mask` 与它相与，负数时保留 `-x`，非负时清零。

```c
int negativePart(int x){
  int mask = x >> 31;
  return mask & (~x + 1);
}
```
P4 copyByteWithin

要求：把 `x` 的第 `src` 字节复制到第 `dst` 字节，其余字节不变。

思路：字节号转成位数要 `<<3`（乘 8）。先右移取出源字节并 `& 0xFF`；再把目标字节清零（`x & ~(0xFF << (dst<<3))`），最后把源字节移到目标位置或进去。

```c
int copyByteWithin(int x, int src, int dst) {
  int byte = (x >> (src << 3)) & 0xFF;
  return (x & ~(0xFF << (dst << 3))) | (byte << (dst << 3));
}
```
P5 logicalShift

要求：实现逻辑右移（高位补 0）。

思路：`x >> n` 默认是算术右移（负数高位补 1）。因此只需把结果最高的 `n` 位清零。构造掩码 `~(((1 << 31) >> n) << 1)`：`(1<<31)>>n` 得到高 `n+1` 位为 1，再左移 1 位得到高 `n` 位为 1，取反后低 `32-n` 位为 1。相与即得逻辑右移结果。

```c
int logicalShift(int x, int n) {
  return (x >> n) & ~(((1 << 31) >> n) << 1);
}
```
P6 swapNibblePairs

要求：把每个字节内的高 4 位与低 4 位互换。

思路：构造掩码 `0x0F0F0F0F`（保留每个字节的低 4 位）。`(x & mask) << 4` 把低半字节移到高半字节；`(x >> 4) & mask` 把高半字节移到低半字节；两者相或。

```c
int swapNibblePairs(int x) {
  int mask = 0x0F;
  mask = mask | (mask << 8);
  mask = mask | (mask << 16);
  return ((x & mask) << 4) | ((x >> 4) & mask);
}
```
P7 secondLowestZeroBit

要求：返回一个只在"倒数第二个 0 位"位置为 1 的掩码。

思路：先用经典技巧找到最低的 0 位：`~x & (x + 1)`（`x+1` 会把最低一段连续 1 翻成 0、最低 0 翻成 1，与 `~x` 相与正好只留下那个最低 0 位）。把它并进 `x`（即把它变成 1），再对结果求一次最低 0 位，就得到原来的第二个 0 位。

```c
int secondLowestZeroBit(int x) {
  int y = x | (~x & (x + 1));
  return ~y & (y + 1);
}
```
P8 oddParity

要求：二进制中 1 的个数为偶数时返回 1，奇数时返回 0。

思路：用"折半异或"把所有位折叠到最低位：连续异或 `x>>16`、`x>>8`、`x>>4`、`x>>2`、`x>>1` 后，最低位就是 32 位的奇偶校验（1 的个数为奇数时为 1）。题目要求偶数返回 1，所以最后再 `^ 1`。

```c
int oddParity(int x) {
  x = x ^ (x >> 16);
  x = x ^ (x >> 8);
  x = x ^ (x >> 4);
  x = x ^ (x >> 2);
  x = x ^ (x >> 1);
  return (x & 1) ^ 1;
}
```
P9 rotateRightBits

要求：把 `x` 循环右移 `n` 位（`n` 可能很大）。

思路：先 `s = n & 31` 把移位数规约到 0~31。循环右移 = 逻辑右移得到的"低位部分" + 把最高 `s` 位搬到最低位的"左移部分"。左移量应为 `32 - s`，用 `(32 + ~s + 1) & 31` 计算（`~s+1` 即 `-s`，`&31` 处理 `s=0` 时退化为移 0 位）。

```c
int rotateRightBits(int x, int n) {
  int s = n & 31;
  int rightPart = (x >> s) & ~(((1 << 31) >> s) << 1);
  int leftPart = x << ((32 + ~s + 1) & 31);
  return rightPart | leftPart;
}
```
P10 roundEvenPow2

要求：把非负 `x` 舍入到最接近的 `2^n` 倍数，恰好一半时取"商为偶数"的那一个。

思路：`q = x >> n` 是向下整除的商，`r = x & (2^n - 1)` 是余数。令 `d = r - 2^(n-1)`：
- `d > 0`（余数过半）→ 进位；
- `d == 0`（恰好一半）→ 当 `q` 为奇数时进位（凑成偶数商）；
- 其余不进位。
最后 `(q + roundUp) << n` 还原。

```c
int roundEvenPow2(int x, int n) {
  int block = 1 << n;
  int q = x >> n;
  int r = x & (block + ~0);
  int half = block >> 1;
  int d = r + ~half + 1;
  int gt = ~(d >> 31) & !!d;
  int eq = !d;
  int roundUp = gt | (eq & (q & 1));
  return (q + roundUp) << n;
}
```
P11 midpointTowardFirst

要求：不溢出的情况下求 `(x+y)/2`，若恰为 .5 则取更靠近第一个参数 `x` 的那个整数。

思路：先算向下取整的中点 `mid = (x & y) + ((x ^ y) >> 1)`（公共位 + 不同位之和的一半，全程不溢出）。当 `x+y` 为奇数（`x^y` 最低位为 1）时，需要向 `x` 方向调整：若 `x > y` 则 `mid` 加 1。`x > y` 的判断要防溢出：符号不同时直接看 `x` 非负；符号相同时看 `x - y` 的符号。

```c
int midpointTowardFirst(int x, int y) {
  int t = x ^ y;
  int mid = (x & y) + (t >> 1);
  int odd = t & 1;
  int sx = x >> 31;
  int sy = y >> 31;
  int d = x + ~y + 1;
  int dn = d >> 31;
  int differ = sx ^ sy;
  int same = ~differ;
  int gt = (same & (~dn & !!d)) | (differ & ~sx);
  return mid + (gt & odd);
}
```
P12 isBetweenEitherOrder

要求：判断 `x` 是否落在以 `a`、`b` 为端点的闭区间内（端点顺序任意）。

思路：先做两个防溢出的有符号比较 `x < a`、`x < b`（得到掩码 `pmask`、`qmask`）。`x` 在区间内当且仅当这两个比较结果不同（一个真一个假），即 `(pmask ^ qmask) & 1`。再单独处理 `x == a`、`x == b` 两个边界（用 `!(x^a)`、`!(x^b)` 判断相等），保证闭区间端点被包含。

```c
int isBetweenEitherOrder(int x, int a, int b) {
  int sx = x >> 31;
  int sa = a >> 31;
  int sb = b >> 31;
  int dxa = x + ~a + 1;
  int dxb = x + ~b + 1;
  int dna = dxa >> 31;
  int dnb = dxb >> 31;
  int diffa = sx ^ sa;
  int diffb = sx ^ sb;
  int samea = ~diffa;
  int sameb = ~diffb;
  int pmask = (diffa & sx) | (samea & dna);
  int qmask = (diffb & sx) | (sameb & dnb);
  int pxorq = (pmask ^ qmask) & 1;
  int eqa = !(x ^ a);
  int eqb = !(x ^ b);
  return pxorq | eqa | eqb;
}
```
P13 mul5Sat

要求：计算 `5x`，若上溢返回 `INT_MAX`，下溢返回 `INT_MIN`，否则返回正常结果。

思路：`5x = (x<<2) + x`。由于乘 5 是奇倍数，饱和阈值对称：`5x` 上溢当且仅当 `x > 429496729`（`= INT_MAX/5`，二进制 `0x19999999`，由 `0x33` 掩码右移 1 位得到）；下溢当且仅当 `x < -429496729`。用 `x - (t+1)` 的符号判断 `x > t`，用 `x + t` 的符号判断 `x < -t`，并分别与符号位配合区分正负方向。

```c
int mul5Sat(int x) {
  int m = 0x33;
  m = m | (m << 8);
  m = m | (m << 16);
  int t = m >> 1;
  int r = (x << 2) + x;
  int sx = x >> 31;
  int t1 = t + 1;
  int d1 = x + ~t1 + 1;
  int g = (~sx) & ~(d1 >> 31);
  int d2 = x + t;
  int l = sx & (d2 >> 31);
  int neg = 1 << 31;
  int pos = ~neg;
  int neither = ~(g | l);
  return (g & pos) | (l & neg) | (neither & r);
}
```
P14 classifyAdd3
要求：判断 `x+y+z` 的精确和是否大于 `INT_MAX`（返回 1）、小于 `INT_MIN`（返回 -1）或正常（返回 0），不能用更宽的整数类型。

思路：分两步加。先算 `a = x + y`，记录第一次相加是否上溢/下溢；再算 `s = a + z`，记录第二次。若"净上溢"（有一次上溢且没有下溢抵消），则精确和必大于 `INT_MAX`；若"净下溢"则小于 `INT_MIN`；若上下溢同时发生（净为零）或都没有，则精确和就等于截断后的 `s`，落在正常范围内。

```c
int classifyAdd3(int x, int y, int z) {
  int a = x + y;
  int s = a + z;
  int sx = x >> 31;
  int sy = y >> 31;
  int sz = z >> 31;
  int sa = a >> 31;
  int ss = s >> 31;
  int notSa = ~sa;
  int up1 = (~sx) & (~sy) & sa;
  int dn1 = sx & sy & notSa;
  int up2 = notSa & (~sz) & ss;
  int dn2 = sa & sz & (~ss);
  int up = up1 | up2;
  int dn = dn1 | dn2;
  int upNet = up & ~dn;
  int dnNet = dn & ~up;
  return (upNet & 1) | dnNet;
}
```
P15 floatScaleThreeHalves

要求：返回 `f * 3/2` 的位级表示（round-to-nearest-even，保留 +0/-0，NaN 原样返回）。

思路：把 `uf` 拆成符号、阶码、尾数。数值写成 `M × 2^e`：
- 规格化数：`M = 2^23 + frac`，`e = exp - 150`；
- 非规格化数：`M = frac`，`e = -149`。
乘 3/2 即 `M = M*3`、`e = e-1`。若 `M < 2^23`（非规格化输入乘 3 后仍偏小）则左移归一化。接着右移到 `[2^23, 2^24)` 得到 23 位尾数并做 round-to-nearest-even（用"丢弃位翻倍 + 尾数最低位"和 `2^k` 比较实现，恰好一半时看奇偶）。最后按新阶码 `ne = e + 150` 分三种情况：`ne ≥ 255` 溢出为无穷；`ne ≤ 0` 退化为非规格化数（再舍入一次，若进位到 `2^23` 则成为最小规格化数）；否则正常拼装。

```c
unsigned floatScaleThreeHalves(unsigned uf) {
  unsigned sign = uf & 0x80000000;
  int exp = (uf >> 23) & 0xFF;
  unsigned frac = uf & 0x7FFFFF;
  unsigned M;
  int e;

  if (exp == 0xFF)
    return uf;

  if (exp == 0) {
    if (frac == 0)
      return sign;
    M = frac;
    e = -149;
  } else {
    M = frac + 0x800000;
    e = exp - 150;
  }

  M = M * 3;
  e = e - 1;

  while (M < 0x800000) {
    M = M << 1;
    e = e - 1;
  }

  unsigned mant = M;
  int k = 0;
  while (mant >= 0x1000000) {
    mant = mant >> 1;
    k = k + 1;
  }

  unsigned mask = 1 << k;
  unsigned dropped = M & (mask - 1);
  unsigned ru = ((dropped << 1) + (mant & 1)) > mask;
  if (ru)
    mant = mant + 1;

  e = e + k;

  if (mant >= 0x1000000) {
    mant = mant >> 1;
    e = e + 1;
  }

  int ne = e + 150;
  if (ne >= 255)
    return sign | 0x7F800000;
  if (ne <= 0) {
    int sh = -e - 149;
    unsigned mask2 = 1 << sh;
    unsigned dropped2 = mant & (mask2 - 1);
    unsigned F = mant >> sh;
    unsigned ru2 = ((dropped2 << 1) + (F & 1)) > mask2;
    if (ru2)
      F = F + 1;
    if (F >= 0x800000)
      return sign | 0x800000;
    return sign | F;
  }

  return sign | (ne << 23) | (mant - 0x800000);
}
```
P16 floatRoundEven

要求：把浮点数舍入到最近的整数（round-half-even），返回结果的位级表示；舍入到 0 时保留符号；NaN/无穷原样返回。

思路：
- 阶码 ≥ `0x96`（值 ≥ `2^23`）：已经是整数，直接返回；
- 阶码 < `0x7F`（值 < 1）：其中 `exp == 0x7E` 且尾数非零表示值在 (0.5, 1)，进位到 1.0；其余舍到 0（保留符号）；
- 其余：`M = 2^23 + frac`，需要右移 `sh = 150 - exp` 位丢掉小数部分，用 round-half-even 进位，再把结果左移归一化、修正阶码后拼装。

```c
unsigned floatRoundEven(unsigned uf) {
  unsigned sign = uf & 0x80000000;
  unsigned exp = (uf >> 23) & 0xFF;
  unsigned frac = uf & 0x7FFFFF;

  if (exp >= 0x96)
    return uf;
  if (exp < 0x7F) {
    if (exp == 0x7E && frac != 0)
      return sign | 0x3F800000;
    return sign;
  }
  unsigned M = frac + 0x800000;
  unsigned sh = 150 - exp;
  unsigned q = M >> sh;
  unsigned d = M & ((1 << sh) - 1);
  unsigned ru = ((d << 1) + (q & 1)) > (1 << sh);
  if (ru)
    q = q + 1;
  unsigned m = q;
  unsigned k = 0;
  while (m < 0x800000) {
    m = m << 1;
    k = k + 1;
  }
  unsigned e2 = 150 - k;
  return sign | (e2 << 23) | (m - 0x800000);
}
```
float_i2f
要求：返回 `(float)x` 的位级表示。

思路：`x == 0` 直接返回 0。取符号位和绝对值 `absx`。找到 `absx` 的最高 1 位 `msb`。把 `absx` 移动成 23 位尾数：`shift = msb - 23`，`shift ≤ 0` 时左移，否则右移 `shift` 位；右移时丢弃的低位用 round-half-even 进位，若进位到 `2^24` 则再右移一位并把阶码加 1。阶码 `exp = 127 + msb`，最后拼装符号、阶码、尾数。

```c
unsigned float_i2f(int x) {
  unsigned ux = x;
  unsigned sign;
  unsigned absx;
  unsigned mant;
  unsigned dropped;
  unsigned ru;
  int msb;
  int shift;
  int exp;

  if (ux == 0)
    return 0;

  sign = ux & 0x80000000;
  if (sign)
    absx = 0 - ux;
  else
    absx = ux;

  msb = 31;
  while (!(absx >> msb))
    msb = msb - 1;

  shift = msb - 23;
  exp = 127 + msb;

  if (shift <= 0) {
    mant = absx << (0 - shift);
  } else {
    mant = absx >> shift;
    dropped = absx & ((1 << shift) - 1);
    ru = ((dropped << 1) + (mant & 1)) > (1 << shift);
    if (ru)
      mant = mant + 1;
    if (mant >= 0x1000000) {
      mant = mant >> 1;
      exp = exp + 1;
    }
  }

  return sign | (exp << 23) | (mant & 0x7FFFFF);
}
```
P18 bitCount

要求：返回 `x` 的二进制中 1 的个数。

思路：分治法（SWAR）。先用掩码 `0x55555555` 统计每 2 位里 1 的个数，再用 `0x33333333` 把相邻 2 位组合并成每 4 位的个数，依次用 `0x0F0F0F0F`、`0x00FF00FF`、`0x0000FFFF` 逐级合并，最后 32 位总和即 1 的个数。这些大掩码都由 0~255 的小常量通过移位扩展生成。

```c
int bitCount(int x) {
  int m1 = 0x55 | (0x55 << 8);
  m1 = m1 | (m1 << 16);
  int m2 = 0x33 | (0x33 << 8);
  m2 = m2 | (m2 << 16);
  int m4 = 0x0F | (0x0F << 8);
  m4 = m4 | (m4 << 16);
  int m8 = 0xFF | (0xFF << 16);
  int m16 = 0xFF | (0xFF << 8);
  x = (x & m1) + ((x >> 1) & m1);
  x = (x & m2) + ((x >> 2) & m2);
  x = (x & m4) + ((x >> 4) & m4);
  x = (x & m8) + ((x >> 8) & m8);
  x = (x & m16) + ((x >> 16) & m16);
  return x;
}
```
P19 bitReverse

要求：把 32 位整数的所有位倒序。

思路：蝶式交换。先用 `0x55555555` 交换相邻两位，再用 `0x33333333` 交换每 2 位一组，再用 `0x0F0F0F0F` 交换每 4 位一组，最后做字节序反转。掩码用 0~255 的常量递归生成：`m4 = 0x0F0F0F0F`，`m2 = m4 ^ (m4<<2) = 0x33333333`，`m1 = m2 ^ (m2<<1) = 0x55555555`。

```c
int bitReverse(int x)
{
  int m = 0x0F;
  m = m | (m << 8);
  m = m | (m << 16);
  int m2 = m ^ (m << 2);
  int m1 = m2 ^ (m2 << 1);
  x = ((x & m1) << 1) | ((x >> 1) & m1);
  x = ((x & m2) << 2) | ((x >> 2) & m2);
  x = ((x & m) << 4) | ((x >> 4) & m);
  int ff00 = 0xFF << 8;
  x = (x << 24) | ((x & ff00) << 8) | ((x >> 8) & ff00) | ((x >> 24) & 0xFF);
  return x;
}
```
五、实验总结

1. 深入理解了补码的位级表示：`-x = ~x + 1`、`x - y = x + ~y + 1`，以及算术右移的符号扩展行为，这些是很多题（P3、P5、P11、P12）的基础。
2. 学会了"用掩码做局部操作"的套路：提取/清零/置换某个字节或半字节（P4、P6、P19），以及如何只用 0~255 的常量通过移位拼接出任意 32 位掩码。
3. 掌握了防溢出比较的技巧：比较两个有符号数时，符号不同直接看符号，符号相同才做减法看符号位（P11、P12、P14）。
4. 理解了 IEEE 754 浮点数的表示：把值写成 `M × 2^e` 统一处理规格化/非规格化数，配合 round-to-nearest-even，实现了乘 3/2、取整、int→float 三个位级转换（P15~P17）。
5. 熟悉了 `dlc` 的规则（整数题只能用 0~255 常量和规定的位运算，浮点题可用循环、分支和任意常量），并学会在操作数上限内优化表达式。

六、给助教老师的建议
希望在实验指导的文件中对每一步操作再做细致的解释
