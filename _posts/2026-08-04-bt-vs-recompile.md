---
layout: post
title: 11x 性能提升 - 反编译用于二进制翻译的 case study
date: 2026-08-04 00:00:00
description: 探索反编译用于二进制翻译的前景
tags: 反编译 二进制翻译
categories: research
---

## 1. 起因

之前有在推进反编译修复的项目，开题的时候提到反编译修复可以用于二进制翻译，然后老师就联系我讨论这个方向，分析了可能遇到的问题，结论是可以做。然后让我帮她围绕这个方向做几页 ppt，我就想到了做一个 case study。

## 2. 实验

“二进制翻译器生成的汇编代码“和“反编译源码“，把它们放在一起对比性能，已知范围内没有人调研过。我以“利用编译器的向量化能力去做优化“为出发点询问 GPT 使用什么程序，GPT 给我推荐的是 clamp 算子，源码如下：

```cpp
__attribute__((noinline))
void add_bias_clamp(
    const uint8_t *src,
    uint8_t *dst,
    int n,
    int bias)
{
    for (int i = 0; i < n; ++i) {
        int value = src[i] + bias;

        if (value > 255) {
            value = 255;
        } else if (value < 0) {
            value = 0;
        }

        dst[i] = (uint8_t)value;
    }
}
```

该算子并不复杂，它是图像处理中常用的一个算子：

- 对于一个大小为 `n` 的数组，先从内存读出来并加 bias
- 然后按照 `[0,255]` 区间进行 clamp
- 最后写回内存

### 2.1 编译成二进制作为输入

为了准确地拿到算子的反编译代码，编译时禁止了 inline；并且为了给编译器留出优化空间，编译时只开启了 `-O2` 优化选项；编译的目标平台为 x86-64。

编译命令：

```sh
gcc -O2 \
    -fno-tree-vectorize \
    -fno-unroll-loops \
    -fno-inline \
    -g \
    -o clamp_x86 clamp.c
```

### 2.2 执行静态二进制翻译

使用组内的 [RVBT](https://crad.ict.ac.cn/article/doi/10.7544/issn1000-1239.202550135) 静态二进制翻译器，它可以将 x86-64 程序端到端地翻译成 RISC-V 的 ELF 文件，最终通过动态二进制翻译器加载该 ELF 文件并在 RISC-V 平台执行。

得到的结果文件：

```text
clamp_x86.s2d: ELF 64-bit LSB relocatable, UCB RISC-V, double-float ABI, version 1 (SYSV), with debug_info, not stripped
```

使用 RISC-V 的反汇编器查看汇编代码：

```sh
riscv64-linux-gnu-objdump -d -S clamp_x86.s2d > clamp_x86.RISCV.asm
```

- `-d`：表示反汇编 `.text` 段
- `-S`: 之前写过附加调试信息的功能，可以把生成的 RISC-V 汇编对应的源 x86 汇编通过 DebugInfo 记录到 ELF 文件中，反汇编的时候加上 `-S` 选项会读取该调试信息一起输出。其对应的源代码文件为二进制翻译器自动生成，如下所示：

```sh
$ head -n 5 clamp_x86.SBT.S
TB-0x4001230:
; X86 Addr: 0x4001230   ENDBR64
; X86 Addr: 0x4001234   MOV     r9, rdi
; X86 Addr: 0x4001237   TEST    edx, edx
; X86 Addr: 0x4001239   JLE     0x400126a
```

#### 2.2.1 二进制翻译结果解析

根据二进制翻译器解析的结果，函数 `add_bias_clamp()` 对应四个基本块 (Basic Block, BB)：

- `0x4001230`：prologue、如果 n 小于 0 函数直接返回
- `0x400123b`：初始化循环变量 i 等于 0
- `0x4001240`：循环 body
- `0x400126a`：epilogue

这里分析其中具有代表性的循环 body `0x4001240`。先看原来的 x86 汇编：

```asm
    1240:       41 0f b6 04 39          movzx  eax,BYTE PTR [r9+rdi*1] # 读取 src[i]
    1245:       41 b8 00 00 00 00       mov    r8d,0x0
    124b:       01 c8                   add    eax,ecx # value=src[i]+bias
    124d:       41 0f 48 c0             cmovs  eax,r8d # add 后如果结果为负数把 0 放到 eax
    1251:       41 b8 ff 00 00 00       mov    r8d,0xff # 加载255
    1257:       44 39 c0                cmp    eax,r8d
    125a:       41 0f 4f c0             cmovg  eax,r8d # 如果 value>255 则写入 255
    125e:       88 04 3e                mov    BYTE PTR [rsi+rdi*1],al # 把 value 的低 8 位写到dst[i]
    1261:       48 83 c7 01             add    rdi,0x1 # i+=1
    1265:       48 39 fa                cmp    rdx,rdi # i!=n
    1268:       75 d6                   jne    1240 
```

- `1240-124b`: 计算 `value = src[i] + bias`，它是要 clamp 的对象
- `124d`: 根据 EFLAGS 的 `SF` 位判断 value 是否小于 0，如果小于 0 则赋值 `eax = r8d = 0`；即 clamp 的下界
- `1251-125a`: 根据 EFLAGS 的 `ZF == 0 && SF == OF` 判断 value 是否大于 255，如果大于 255 则赋值 `eax = r8d = 255`；即 clamp 的上界
- `125e`: value 写入结果数组 `dst[i]`
- `1261-1268`: 循环的自身代码。`i+=1;`、如果 `i!=n` 回到 body 开头

---

然后看二进制翻译器生成的 RISC-V 汇编：

```asm
; X86 Addr: 0x4001240 MOVZX eax, byte ptr [r9 + rdi]
 5a0: 00a78b33           add s6,a5,a0
 5a4: 000b4883           lbu a7,0(s6)
; X86 Addr: 0x4001245 MOV r8d, 0
 5a8: 00000713           li a4,0
; X86 Addr: 0x400124b ADD eax, ecx
 5ac: 02089b13           slli s6,a7,0x20
 5b0: 02069b13           slli s6,a3,0x20
 5b4: 00d888b3           add a7,a7,a3
 5b8: 02089b13           slli s6,a7,0x20
 5bc: 000b5663           bgez s6,5c8 <4656._Z14add_bias_clampPKhPhii.6+0x60>
; X86 Addr: 0x400124d CMOVS eax, r8d
 5c0: 000708b3           add a7,a4,zero
 5c4: 0080006f           j 5cc <4656._Z14add_bias_clampPKhPhii.6+0x64>
 5c8: 7c08b88b           th.extu a7,a7,31,0
; X86 Addr: 0x4001251 MOV r8d, 0xff
 5cc: 0ff00713           li a4,255
; X86 Addr: 0x4001257 CMP eax, r8d
 5d0: 02089b13           slli s6,a7,0x20
 5d4: 02071e13           slli t3,a4,0x20
 5d8: 016e5463           bge t3,s6,5e0 <4656._Z14add_bias_clampPKhPhii.6+0x78>
; X86 Addr: 0x400125a CMOVG eax, r8d
 5dc: 000708b3           add a7,a4,zero
 5e0: 7c08b88b           th.extu a7,a7,31,0
; X86 Addr: 0x400125e MOV byte ptr [rsi + rdi], al
 5e4: 00a58b33           add s6,a1,a0
 5e8: 011b0023           sb a7,0(s6)
; X86 Addr: 0x4001261 ADD rdi, 1
 5ec: 00150513           addi a0,a0,1
; X86 Addr: 0x4001265 CMP rdx, rdi
; X86 Addr: 0x4001268 JNE 0x4001240
 5f0: 04a61063           bne a2,a0,630 <4656._Z14add_bias_clampPKhPhii.6+0xc8>
```

|x86 汇编地址|RISC-V 汇编地址|操作描述|x86 指令数量|RISC-V 指令数量|指令膨胀率|
|-|-|-|-|-|-|
|`1240-124b`|`5a0-5bc`|计算 `value = src[i] + bias`，它是要 clamp 的对象|3|8|2.67x|
|`124d`|`5c0-5c8`|根据 EFLAGS 的 `SF` 位判断 value 是否小于 0，如果小于 0 则赋值 `eax = r8d = 0`；即 clamp 的下界|1|3|3x|
|`1251-125a`|`5cc-5e0`|根据 EFLAGS 的 `ZF == 0 && SF == OF` 判断 value 是否大于 255，如果大于 255 则赋值 `eax = r8d = 255`；即 clamp 的上界|3|6|2x|
|`125e`|`5e4-5e8`|value 写入结果数组 `dst[i]`|1|2|2x|
|`1261-1268`|`5ec-5f0`|循环的自身代码。`i+=1;`、如果 `i!=n` 回到 body 开头|3|2|0.67x|

> `5f0: bne a2,a0,630` 跳到的代码 `630-654` 和 `5a0-5c4` 完全相同，对应的是二进制翻译器反编译的 bug，即 `TB-0x400123b` 覆盖了 `TB-0x4001240` 的代码，所以 `TB-0x4001240` 的代码会出现两份。

可以看出二进制翻译虽然有两条指令翻译成一条的优化案例 (`5f0`)，但是在翻译 x86 的 EFLAGS 计算时绝大多数都引入了 1 倍以上的开销。根据这个观察得到第一个优化点：

**优化 1**：如果从源码直接编译成 RISC-V 程序，翻译 EFLAGS 带来的约 1 倍开销预期可以被优化掉。

### 2.3 执行反编译+反编译修复

同样的 x86 程序，作为输入给反编译器 Hex Rays (IDA)，它会给出 Pseudo-C 代码：

```c
void __fastcall add_bias_clamp(const uint8_t *src, uint8_t *dst, int n, int bias)
{
  __int64 i; // rdi
  int v6; // eax

  if ( n > 0 )
  {
    for ( i = 0; i != n; dst[i++] = v6 )
    {
      v6 = bias + src[i];
      if ( v6 < 0 )
        v6 = 0;
      if ( v6 > 255 )
        LOBYTE(v6) = -1;
    }
  }
}
```

首先，可以看出这个函数无法直接进行编译

- `__fastcall` 在 gcc linux 上无法识别
- `__int64` 类型未定义
- `LOBYTE` 宏没有被定义

其次，可以发现 Hex Rays 没有像二进制翻译器一样逐条汇编指令的翻译成 Pseudo-C，而是进行了**语义的提升**，即进行了循环的还原。根据这个观察可以得到第二个优化点：

**优化 2**：利用反编译器还原的循环语义，对代码进行向量化优化提升程序的计算吞吐。

---

反编译修复包含两层含义：

- 可通过编译
- 反编译函数的语义和原来的函数一致

修复后的反编译代码如下：

```c
#include <stdint.h>

#define __fastcall
#define __int64 int64_t
void __fastcall add_bias_clamp(const uint8_t *src, uint8_t *dst, int n, int bias)
{
  __int64 i; // rdi
  int v6; // eax

  if ( n > 0 )
  {
    for ( i = 0; i != n; dst[i++] = v6 )
    {
      v6 = bias + src[i];
      if ( v6 < 0 )
        v6 = 0;
      if ( v6 > 255 )
        v6 = 255;
    }
  }
}
```

针对三个问题分别进行了修复：
- 定义 `__fastcall` 为空
- 定义 `__int64` 为 `<stdint.h>` 中的 `int64_t`
- 将 `LOBYTE(v6) = -1` 改写成语义等价的 `v6 = 255`

### 2.4 汇编/编译

为了测试性能，将二进制翻译的 RISC-V 汇编和反编译的 C 代码都汇编/编译成 `.o` 文件，然后和测试框架链接到一起得到不同的可执行程序进行性能测试。

RISC-V 汇编文件如下：

```asm
    .text
    .align 2
    .globl add_bias_clamp
    .type add_bias_clamp, @function
add_bias_clamp:
    blez a2, .Ldone
    li t0, 0
.Lloop:
    add t1, a0, t0
    lbu a7, 0(t1)
    li a4, 0
    slli t4, a7, 32
    slli t4, a3, 32
    add a7, a7, a3
    slli t4, a7, 32
    bgez t4, .Lnonnegative
    add a7, a4, zero
    j .Lclamp_high
.Lnonnegative:
    .4byte 0x7c08b88b
.Lclamp_high:
    li a4, 255
    slli t4, a7, 32
    slli t3, a4, 32
    bge t3, t4, .Lstore
    add a7, a4, zero
.Lstore:
    .4byte 0x7c08b88b
    add t1, a1, t0
    sb a7, 0(t1)
    addi t0, t0, 1
    bne a2, t0, .Lloop
.Ldone:
    ret
    .size add_bias_clamp, .-add_bias_clamp
```

玄铁拓展指令通过手写机器码进行汇编，即 `.4byte 0x7c08b88b`。

执行汇编得到 `clamp_bt.o`：

```sh
riscv64-linux-gnu-gcc \
    -O3 -march=rv64gcv -mabi=lp64d -fno-lto \
    -c clamp_x86_bt.S -o clamp_bt.o
```

---

编译反编译代码分别使用了 gcc-13.3.0 和 clang-18.1.8：

```text
gcc version 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04.1)
Target: riscv64-linux-gnu

clang version 18.1.8 (https://github.com/llvm/llvm-project.git 3b5b5c1ec4a3095ab096dd780e84d7ab81f3d7ff)
Target: riscv64-unknown-linux-gnu
```

编译命令如下（针对 RVV1.0 扩展）：

```sh
# gcc
riscv64-linux-gnu-gcc \
    -O3 -march=rv64gcv1p0 -mabi=lp64d -fno-lto \
    -c clamp_x86_ida_fixed.c -o clamp_gcc.o

# clang
/opt/llvm-rv64gcv/bin/clang \
    -O3 -march=rv64gcv1p0 -mabi=lp64d -fno-lto \
    -c clamp_x86_ida_fixed.c -o clamp_clang.o
```

观察发现：尽管都开启了 `-march=rv64gcv1p0`，但是 gcc 没有发射向量指令，而 clang 发射了向量指令。

clang 发射的向量指令如下：

```asm
0000000000000072 <.LBB0_11>:
  72:   22870407                vl2r.v  v8,(a4)
  76:   0d3077d7                vsetvli a5,zero,e32,m8,ta,ma
  7a:   4a822857                vzext.vf4       v16,v8
  7e:   0306c457                vadd.vx v8,v16,a3
  82:   1e804457                vmax.vx v8,v8,zero
  86:   1288c457                vminu.vx        v8,v8,a7
  8a:   0ca07057                vsetvli zero,zero,e16,m4,ta,ma
  8e:   b2803857                vnsrl.wi        v16,v8,0
  92:   0c107057                vsetvli zero,zero,e8,m2,ta,ma
  96:   b3003457                vnsrl.wi        v8,v16,0
  9a:   22838427                vs2r.v  v8,(t2)
  9e:   9716                    add     a4,a4,t0
  a0:   40530333                sub     t1,t1,t0
  a4:   9396                    add     t2,t2,t0
  a6:   00031063                bnez    t1,a6 <.LBB0_11+0x34>
  aa:   00c81063                bne     a6,a2,aa <.LBB0_11+0x38>
```

- `72`：加载 src，`s = src[i:i+VL]`，当前模式为 E8、LMUL=2，假设 VLEN=1024， VL=`1024*2/8=256`
- `76`：设置 E32、LMUL=8，将 8 个 RISC-V 向量寄存器合成一组，假设 VLEN=1024，下面向量计算的 VL=`1024*8/32=256`，即一条指令计算 256 个元素
  - `7a`：将 8 bit 的元素零扩展到 32 bit，`v16-23 = v8`
  - `7e`：加 bias，`v = s + bias`
  - `82`：取 clamp 下界 0，`v_bz = max(v, 0)`
  - `86`：取 clamp 上界 255，`v_m255 = min(v_bz, 255)`
- `8a-96`：把 32 位的数据再调整回 8 位的数据，`v_mask = (v_m255 >> 24)`
- `9a`：存回结果，`dst[i:i+VL] = v_mask`
- `9e-aa`：循环自身代码，更新 `src`、`dst`、`i`

---

gcc 虽然没有像 clang 一样实现向量化，但是编译得到的程序相比二进制翻译器也简洁很多：

```asm
0000000000000000 <add_bias_clamp>:
   0:   02c05863                blez    a2,30 <.L1>
   4:   962a                    add     a2,a2,a0
   6:   0ff00893                li      a7,255

000000000000000a <.L5>:
   a:   00054783                lbu     a5,0(a0)
   e:   9fb5                    addw    a5,a5,a3
  10:   0007881b                sext.w  a6,a5
  14:   fff84713                not     a4,a6
  18:   977d                    srai    a4,a4,0x3f
  1a:   8ff9                    and     a5,a5,a4
  1c:   0108d463                bge     a7,a6,24 <.L4>
  20:   0ff00793                li      a5,255

0000000000000024 <.L4>:
  24:   00f58023                sb      a5,0(a1)
  28:   0505                    addi    a0,a0,1
  2a:   0585                    addi    a1,a1,1
  2c:   fca61fe3                bne     a2,a0,a <.L5>

0000000000000030 <.L1>:
  30:   8082                    ret
```

循环 body BT 使用了 21 条指令，而 gcc 只用了 12 条指令，节省了 42.85%。

### 2.5 性能测试

<img src="{{ '/assets/img/blog/rv_benchmark_v1.png' | relative_url }}" style="max-width: 100%;">

性能测试的结果如上图所示。反编译代码经过修复和 gcc 编译后相对 BT 取得了 ~1.4x 的性能提升，而经过 clang 编译后在 n>=256 的情况下，取得了 ~11.5x 的性能提升。256 对应了 VL=256，即一次操作 256 个元素，只有 n>=256 才能打满向量的计算吞吐。

## 3. 总结

二进制翻译器在翻译时没有很好地提升程序的语义，在处理 EFLAGS 模拟时尽管经过优化也造成了接近 2 倍的指令膨胀；而反编译器经过 10 余年的发展，已经不仅仅可以用于逆向工程 (Reverse Engineering)，同时也能用于进行二进制翻译；反编译器还原的语义信息结合编译器的优化技术，二者可以给二进制翻译带来巨大的性能提升。

本文通过 clamp 算子这一 case study，分别对比了 BT、decomp+gcc 和 decomp+clang 三组由 x86 二进制作为输入得到的 RISC-V 二进制输出的性能，发现在未向量化的情况下 (gcc) 取得了 1.4x 的性能提升；而在向量化的情况下 (clang) 取得了 11.5x 的性能提升，这一提升效果是可观的。

但是在实现路径上，关键的技术难点是反编译修复，该问题可以从两方面着手解决：

1. 函数级别的正确修复：不要求修复正确所有函数，将二进制翻译器修改成 hybrid 的架构，将能够修复正确的函数链接到生成的静态翻译结果中，从而实现性能提升。 (该想法启发自 [PRD](https://arxiv.org/abs/2202.12336))
2. 程序级别的正确修复：提升反编译修复的能力上限，目前的 [SOTA](https://arxiv.org/abs/2603.14855) 仍然不能将 GNU Coreutils 这样小型的程序修复正确，且在 stripped 情况下修复正确率仅为 25%，因此如果希望实现程序级别的优化，仍然需要拓宽反编译修复的能力边界。
