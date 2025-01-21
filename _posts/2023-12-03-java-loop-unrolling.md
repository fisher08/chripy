---
title: Loop Unrolling in Java - JVM JIT
description: >-
  How JVM evaluate loops.
author: kyrylrybkin
date: 2023-12-03 20:55:00 +0800
categories: [trading infrastructure]
tags: [java]
pin: true
media_subpath: '/assets/posts/loop-unrolling'
---
## The difference between source code and how code is executed
As we know, compilers optimise the source code. If the code is executed as it is written, it will not be efficient. If the compiler did not apply optimisations, it would execute a loop body per iteration, check for loop termination and jump to the beginning of the loop body. Such execution is not efficient because modern processors perform several operations in one clock cycle: selecting the next instruction, decoding, executing, writing. This type of execution is called a pipeline. The number of pipeline stages can vary depending on the processor, e.g. the Intel Pentium 4 processor had 20 pipeline stages, and in the Prescott modification it got a pipeline of 31 stages. In the case of executing one instruction per loop iteration, the execution is not linear and each transition to the beginning of the loop body invalidates the pipeline. The performance penalty in this case is comparable to a cache miss.
Let's see how loops compile in C without optimisation and how Hotspot JIT optimises assembly in java11 and java17. For the purpose of the experiment, let's allocate 1000_000 long in memory and fill it with random values.

## Loops in C code
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

unsigned long xorshift(unsigned long state[static 1]) {
    unsigned long x = state[0];
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    state[0] = x;
    return x;
}

long random_long(long min, long max) {
    int urandom = open("/dev/urandom", O_RDONLY);
    unsigned long state[1];
    read(urandom, state, sizeof(state));
    close(urandom);
    unsigned long range = (unsigned long) max - min + 1;
    unsigned long random_value = xorshift(state) % range;
    return (long) (random_value + min);
}


int main(int argv, char** argc) {
    int MAX = 1000000;

    long* data = (long*)calloc(MAX, sizeof(long));

    for (int i = 0; i < MAX; i++) {
        data[i] = random_long(0,MAX);
    }
}
```
```terminal
gcc -S loopunrolling.c 
```
Let's consider only a part of the assembly code, calling the main method. As we can see, there is only one call of the `call random_long` function per loop iteration, which is expected.

```terminal
main:
.LFB8:
	.cfi_startproc
	endbr64
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	pushq	%rbx
	subq	$40, %rsp
	.cfi_offset 3, -24
	movl	%edi, -36(%rbp)
	movq	%rsi, -48(%rbp)
	movl	$1000000, -28(%rbp)
	movl	-28(%rbp), %eax
	cltq
	movl	$8, %esi
	movq	%rax, %rdi
	call	calloc@PLT
	movq	%rax, -24(%rbp)
	movl	$0, -32(%rbp)
	jmp	.L7
.L8:
	movl	-28(%rbp), %eax
	cltq
	movl	-32(%rbp), %edx
	movslq	%edx, %rdx
	leaq	0(,%rdx,8), %rcx
	movq	-24(%rbp), %rdx
	leaq	(%rcx,%rdx), %rbx
	movq	%rax, %rsi
	movl	$0, %edi
	call	random_long
	movq	%rax, (%rbx)
	addl	$1, -32(%rbp)
.L7:
	movl	-32(%rbp), %eax
	cmpl	-28(%rbp), %eax
	jl	.L8
	movl	$0, %eax
	movq	-8(%rbp), %rbx
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE8:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0"
	.section	.note.GNU-stack,"",@progbits
	.section	.note.gnu.property,"a"
	.align 8
	.long	1f - 0f
	.long	4f - 1f
	.long	5
	
```

## Loops in Java
Now let's fill in long[] in Java. Java code is different from C, we need to add an `intStride1` method which will compile JIT since the minimum compilation unit is a method.
```java
public class LoopUnroll {
    private  static int MAX = 1000000;
    private static long[] data = new long[MAX];

    public static void main(String[] args) {
        java.util.Random random = new java.util.Random();

        for (int i = 0; i < MAX; i++) {
            data[i] = random.nextLong();
        }
        final long sum = intStride1();

        System.out.println("Out");
        System.out.println(sum);
    }

    private static long intStride1()
    {
        int sum = 0;
        for (int i = 0; i < MAX; i += 1)
        {
            sum += data[i];
        }
        return sum;
    }
}
```
### Bytecode
In Bytecode we are interested in the `private static long intStride1();` method. As we can see, the bytecode has two `ladd` operations per iteration, one for dealing with the array `data[]` (`20: ladd`) and the other for counter `i` (`24: ladd`), which corresponds to one operation per loop iteration. As we can see, there is no runtime optimisation in the bytecode.
```console
javap -p -v LoopUnroll.class
// -- omitted
  private static long intStride1();
    descriptor: ()J
    flags: (0x000a) ACC_PRIVATE, ACC_STATIC
    Code:
      stack=5, locals=4, args_size=0
         0: lconst_0
         1: lstore_0
         2: lconst_0
         3: lstore_2
         4: lload_2
         5: getstatic     #10                 // Field MAX:I
         8: i2l
         9: lcmp
        10: ifge          29
        13: lload_0
        14: getstatic     #16                 // Field data:[J
        17: lload_2
        18: l2i
        19: laload
        20: ladd
        21: lstore_0
        22: lload_2
        23: lconst_1
        24: ladd
        25: lstore_2
        26: goto          4
        29: lload_0
        30: lreturn
      LineNumberTable:
        line 21: 0
        line 22: 2
        line 24: 13
        line 22: 22
        line 26: 29
      StackMapTable: number_of_entries = 2
        frame_type = 253 /* append */
          offset_delta = 4
          locals = [ long, long ]
        frame_type = 250 /* chop */
          offset_delta = 24
// -- omitted
SourceFile: "LoopUnroll.java"
```

## Benchmark
Let's consider several loop variants with different counter type: one int and the other long and see how the counter type affects the generated JIT copmiler code, unrolling and safepoint placement in the loop.
We add `@CompilerControl(CompilerControl.Mode.DONT_INLINE)` so that the method is not embedded in benchmark. We will run benchmark with different VM keys to control code generation:
- `-XX:+UseCountedLoopSafepoints` - controls the presence of safepoints in the loop.
- `-XX:LoopStripMiningIter=number_of_iterations` - Controls the number of iterations in the inner loop. The safepoint will be stored in the outer loop and the inner loop will not contain a safepoint. The default is 1,000.
- `-XX:LoopStripStripMiningIterShortLoop=number_of_iterations` - Loops with less than the specified number of iterations will not have a safepoint.
### Код
```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Thread)
@Fork(value = 1, jvmArgsPrepend = {"-XX:+UnlockDiagnosticVMOptions", "-XX:-UseCompressedOops", "-XX:PrintAssemblyOptions=intel", "-XX:LoopStripMiningIter=10000", "-XX:-UseCountedLoopSafepoints"})
public class LoopUnrollBenchmark {

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public void baseline() {
    }

    private static final int MAX = 1_000_000;

    private long[] data = new long[MAX];

    @Setup
    public void createData()
    {
        java.util.Random random = new java.util.Random();

        for (int i = 0; i < MAX; i++)
        {
            data[i] = random.nextLong();
        }
    }

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public long intStride1()
    {
        long sum = 0;
        for (int i = 0; i < MAX; i++)
        {
            sum += data[i];
        }
        return sum;
    }

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public long longStride1()
    {
        long sum = 0;
        for (long l = 0; l < MAX; l++)
        {
            sum += data[(int) l];
        }
        return sum;
    }
}
```
```console
java  -jar target/benchmarks.jar -prof perfasm
```

### Java 11 counter loop
Build with the following vm arguments.
```shell
@Fork(value = 1, jvmArgsPrepend = {"-XX:+UnlockDiagnosticVMOptions", "-XX:-UseCompressedOops", "-XX:PrintAssemblyOptions=intel"})
```

```shell
c2, level 4, com.rkdeep.LoopUnrollBenchmark::intStride1, version 3, compile id 646 

              0x00007fdee83d10d0: cmp    r10d,0xf423f
              0x00007fdee83d10d7: jbe    0x00007fdee83d115d
              0x00007fdee83d10dd: mov    rax,QWORD PTR [r9+0x18]  ;*laload {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@16 (line 72)
              0x00007fdee83d10e1: mov    r10d,0x1           ;*goto {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
              0x00007fdee83d10e7: mov    r8d,0xfa0
           ↗  0x00007fdee83d10ed: mov    ecx,0xf423d
           │  0x00007fdee83d10f2: sub    ecx,r10d
           │  0x00007fdee83d10f5: cmp    ecx,r8d
   0.02%   │  0x00007fdee83d10f8: cmovg  ecx,r8d
           │  0x00007fdee83d10fc: add    ecx,r10d
           │  0x00007fdee83d10ff: nop                       ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
           │                                                ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 72)
   0.06%  ↗│  0x00007fdee83d1100: add    rax,QWORD PTR [r9+r10*8+0x18]
  31.11%  ││  0x00007fdee83d1105: add    rax,QWORD PTR [r9+r10*8+0x20]
  22.35%  ││  0x00007fdee83d110a: add    rax,QWORD PTR [r9+r10*8+0x28]
  22.42%  ││  0x00007fdee83d110f: add    rax,QWORD PTR [r9+r10*8+0x30]
          ││                                                ;*ladd {reexecute=0 rethrow=0 return_oop=0}
          ││                                                ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 72)
  22.14%  ││  0x00007fdee83d1114: add    r10d,0x4           ;*iinc {reexecute=0 rethrow=0 return_oop=0}
          ││                                                ; - com.rkdeep.LoopUnrollBenchmark::intStride1@19 (line 70)
   0.02%  ││  0x00007fdee83d1118: cmp    r10d,ecx
          ╰│  0x00007fdee83d111b: jl     0x00007fdee83d1100  ;*if_icmpge {reexecute=0 rethrow=0 return_oop=0}
           │                                                ; - com.rkdeep.LoopUnrollBenchmark::intStride1@7 (line 70)
           │  0x00007fdee83d111d: mov    r14,QWORD PTR [r15+0x108]
           │                                                ; ImmutableOopMap{r11=Oop r9=Oop }
           │                                                ;*goto {reexecute=1 rethrow=0 return_oop=0}
           │                                                ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
   0.01%   │  0x00007fdee83d1124: test   DWORD PTR [r14],eax  ;*goto {reexecute=0 rethrow=0 return_oop=0}
           │                                                ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
           │                                                ;   {poll}
   0.12%   │  0x00007fdee83d1127: cmp    r10d,0xf423d
           ╰  0x00007fdee83d112e: jl     0x00007fdee83d10ed
              0x00007fdee83d1130: cmp    r10d,0xf4240
              0x00007fdee83d1137: jge    0x00007fdee83d114d
              0x00007fdee83d1139: data16 xchg ax,ax         ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 72)
              0x00007fdee83d113c: add    rax,QWORD PTR [r9+r10*8+0x18]
                                                            ;*ladd {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 72)
....................................................................................................
  98.23%  <total for region 1>

....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::longStride1, version 3, compile id 630 

               0x00007f8cc43cf124: jne    0x00007f8cbc84b080  ;   {runtime_call ic_miss_stub}
               0x00007f8cc43cf12a: xchg   ax,ax
               0x00007f8cc43cf12c: nop    DWORD PTR [rax+0x0]
             [Verified Entry Point]
               0x00007f8cc43cf130: mov    DWORD PTR [rsp-0x14000],eax
               0x00007f8cc43cf137: push   rbp
               0x00007f8cc43cf138: sub    rsp,0x30           ;*synchronization entry
                                                             ; - com.rkdeep.LoopUnrollBenchmark::longStride1@-1 (line 81)
               0x00007f8cc43cf13c: mov    r10,QWORD PTR [rsi+0x10]  ;*getfield data {reexecute=0 rethrow=0 return_oop=0}
                                                             ; - com.rkdeep.LoopUnrollBenchmark::longStride1@14 (line 84)
   0.00%       0x00007f8cc43cf140: mov    r9d,DWORD PTR [r10+0x10]  ;*laload {reexecute=0 rethrow=0 return_oop=0}
                                                             ; - com.rkdeep.LoopUnrollBenchmark::longStride1@19 (line 84)
                                                             ; implicit exception: dispatches to 0x00007f8cc43cf1a4
   0.01%       0x00007f8cc43cf144: xor    eax,eax            ;*goto {reexecute=0 rethrow=0 return_oop=0}
                                                             ; - com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
               0x00007f8cc43cf146: xor    r11d,r11d
               0x00007f8cc43cf149: xor    r8d,r8d
          ╭    0x00007f8cc43cf14c: jmp    0x00007f8cc43cf153
          │    0x00007f8cc43cf14e: xchg   ax,ax
  12.00%  │ ↗  0x00007f8cc43cf150: mov    r11d,r8d           ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                ; - com.rkdeep.LoopUnrollBenchmark::longStride1@12 (line 84)
  10.85%  ↘ │  0x00007f8cc43cf153: cmp    r11d,r9d
           ╭│  0x00007f8cc43cf156: jae    0x00007f8cc43cf184
   9.98%   ││  0x00007f8cc43cf158: add    rax,QWORD PTR [r10+r11*8+0x18]
           ││                                                ;*ladd {reexecute=0 rethrow=0 return_oop=0}
           ││                                                ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 84)
  24.21%   ││  0x00007f8cc43cf15d: mov    r11,QWORD PTR [r15+0x108]
  11.56%   ││  0x00007f8cc43cf164: add    r8,0x1             ; ImmutableOopMap{r10=Oop rsi=Oop }
           ││                                                ;*goto {reexecute=1 rethrow=0 return_oop=0}
           ││                                                ; - com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
  10.62%   ││  0x00007f8cc43cf168: test   DWORD PTR [r11],eax  ;*goto {reexecute=0 rethrow=0 return_oop=0}
           ││                                                ; - com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
           ││                                                ;   {poll}
  18.83%   ││  0x00007f8cc43cf16b: cmp    r8,0xf4240
           │╰  0x00007f8cc43cf172: jl     0x00007f8cc43cf150  ;*ifge {reexecute=0 rethrow=0 return_oop=0}
           │                                                 ; - com.rkdeep.LoopUnrollBenchmark::longStride1@9 (line 82)
           │   0x00007f8cc43cf174: add    rsp,0x30
           │   0x00007f8cc43cf178: pop    rbp
   0.01%   │   0x00007f8cc43cf179: mov    r10,QWORD PTR [r15+0x108]
           │   0x00007f8cc43cf180: test   DWORD PTR [r10],eax  ;   {poll_return}
           │   0x00007f8cc43cf183: ret    
           ↘   0x00007f8cc43cf184: mov    rbp,rsi
               0x00007f8cc43cf187: mov    QWORD PTR [rsp],r8
               0x00007f8cc43cf18b: mov    QWORD PTR [rsp+0x8],rax
               0x00007f8cc43cf190: mov    QWORD PTR [rsp+0x10],r10
               0x00007f8cc43cf195: mov    DWORD PTR [rsp+0x18],r11d
               0x00007f8cc43cf19a: mov    esi,0xffffffe4
               0x00007f8cc43cf19f: call   0x00007f8cbc849e00  ; ImmutableOopMap{rbp=Oop [16]=Oop }
                                                             ;*laload {reexecute=0 rethrow=0 return_oop=0}
....................................................................................................
  98.08%  <total for region 1>

Benchmark                             Mode  Cnt          Score          Error  Units
LoopUnrollBenchmark.baseline         thrpt    5  420136389.339 ± 61698598.658  ops/s
LoopUnrollBenchmark.baseline:asm     thrpt                 NaN                   ---
LoopUnrollBenchmark.intStride1       thrpt    5       2457.647 ±      176.800  ops/s
LoopUnrollBenchmark.intStride1:asm   thrpt                 NaN                   ---
LoopUnrollBenchmark.longStride1      thrpt    5       1391.287 ±       85.554  ops/s
LoopUnrollBenchmark.longStride1:asm  thrpt                 NaN                   ---
```
You can see that with the counter type int the loop consists of 2 inner and outer loops. The body of the inner loop is repeated 4 times, i.e. the loop is expanded by 4. After the inner loop, safepoints are added.  Safepoint - points in the code where data are consistent and threads can be safely stopped for stacktrace removal or GC work. The loop that is executed for clarity can be represented as in the listing below.
```java
        for (int j = 0; j < 250; j++) {
            for (int i = 0; i < 4_000; i = i+4) {
                sum += data[i];
                sum += data[i+1];
                sum += data[i+2];
                sum += data[i+3];
            }
            // safepoint
        }
```
Unlike a loop with int counter, when using long counter the loop is compiled without using loopunrolling optimisation and safepoint is checked in each iteration of the loop. It is possible to represent with pseudocode as in the listing below
```java
        for (int i = 0; i < 1_000_000; i++) {
            sum += data[i];
            // safepoint
        }
```
Let's take the results of java 11 as a baseline.
### Java 17 counter loop saftpoints control
#### Benchmark without safepoints -XX:-UseCountedLoopSafepoints
Remove safepoints from the loop and add an inner loop with 10000 iterations `"-XX:LoopStripMiningIter=10000", "-XX:-UseCountedLoopSafepoints"`.
```shell
@Fork(value = 1, jvmArgsPrepend = {"-XX:+UnlockDiagnosticVMOptions", "-XX:-UseCompressedOops", "-XX:+UseSuperWord", "-XX:PrintAssemblyOptions=intel", "-XX:LoopStripMiningIter=10000", "-XX:-UseCountedLoopSafepoints"})
```
```shell
Result "com.rkdeep.LoopUnrollBenchmark.intStride1":
  2581.171 ±(99.9%) 14.527 ops/s [Average]
  (min, avg, max) = (2575.700, 2581.171, 2585.076), stdev = 3.773
  CI (99.9%): [2566.645, 2595.698] (assumes normal distribution)

Secondary result "com.rkdeep.LoopUnrollBenchmark.intStride1:asm":
PrintAssembly processed: 166164 total address lines.
Perf output processed (skipped 59.009 seconds):
 Column 1: cycles (49732 events)

Hottest code regions (>10.00% "cycles" events):
 Event counts are percents of total event count.

....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::intStride1, version 3, compile id 721 

   0.01%      0x00007f3ee4fd7453:   mov    r8d,DWORD PTR [r10+0xc]      ; implicit exception: dispatches to 0x00007f3ee4fd751c
   0.01%      0x00007f3ee4fd7457:   test   r8d,r8d
              0x00007f3ee4fd745a:   jbe    0x00007f3ee4fd751c
              0x00007f3ee4fd7460:   cmp    r8d,0xf423f
              0x00007f3ee4fd7467:   jbe    0x00007f3ee4fd751c
              0x00007f3ee4fd746d:   mov    rax,QWORD PTR [r10+0x10]     ;*laload {reexecute=0 rethrow=0 return_oop=0}
                                                                        ; - com.rkdeep.LoopUnrollBenchmark::intStride1@16 (line 73)
              0x00007f3ee4fd7471:   mov    r11d,0x1
          ╭   0x00007f3ee4fd7477:   jmp    0x00007f3ee4fd7483
          │   0x00007f3ee4fd7479:   nop    DWORD PTR [rax+0x0]
   0.01%  │↗  0x00007f3ee4fd7480:   mov    r11d,r9d                     ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
          ││                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 73)
   6.03%  ↘│  0x00007f3ee4fd7483:   add    rax,QWORD PTR [r10+r11*8+0x10]
   0.01%   │  0x00007f3ee4fd7488:   add    rax,QWORD PTR [r10+r11*8+0x18]
   5.98%   │  0x00007f3ee4fd748d:   add    rax,QWORD PTR [r10+r11*8+0x20]
   5.69%   │  0x00007f3ee4fd7492:   add    rax,QWORD PTR [r10+r11*8+0x28]
   5.73%   │  0x00007f3ee4fd7497:   add    rax,QWORD PTR [r10+r11*8+0x30]
   5.89%   │  0x00007f3ee4fd749c:   add    rax,QWORD PTR [r10+r11*8+0x38]
   7.75%   │  0x00007f3ee4fd74a1:   add    rax,QWORD PTR [r10+r11*8+0x40]
   5.88%   │  0x00007f3ee4fd74a6:   add    rax,QWORD PTR [r10+r11*8+0x48]
   5.69%   │  0x00007f3ee4fd74ab:   add    rax,QWORD PTR [r10+r11*8+0x50]
   5.94%   │  0x00007f3ee4fd74b0:   add    rax,QWORD PTR [r10+r11*8+0x58]
   6.17%   │  0x00007f3ee4fd74b5:   add    rax,QWORD PTR [r10+r11*8+0x60]
   5.94%   │  0x00007f3ee4fd74ba:   add    rax,QWORD PTR [r10+r11*8+0x68]
   5.84%   │  0x00007f3ee4fd74bf:   add    rax,QWORD PTR [r10+r11*8+0x70]
   5.71%   │  0x00007f3ee4fd74c4:   add    rax,QWORD PTR [r10+r11*8+0x78]
   8.30%   │  0x00007f3ee4fd74c9:   add    rax,QWORD PTR [r10+r11*8+0x80]
   6.04%   │  0x00007f3ee4fd74d1:   add    rax,QWORD PTR [r10+r11*8+0x88];*ladd {reexecute=0 rethrow=0 return_oop=0}
           │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 73)
   5.82%   │  0x00007f3ee4fd74d9:   mov    r9d,r11d
   0.00%   │  0x00007f3ee4fd74dc:   add    r9d,0x10                     ;*iinc {reexecute=0 rethrow=0 return_oop=0}
           │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@19 (line 71)
           │  0x00007f3ee4fd74e0:   cmp    r9d,0xf4231
           ╰  0x00007f3ee4fd74e7:   jl     0x00007f3ee4fd7480           ;*if_icmpge {reexecute=0 rethrow=0 return_oop=0}
                                                                        ; - com.rkdeep.LoopUnrollBenchmark::intStride1@7 (line 71)
              0x00007f3ee4fd74e9:   cmp    r9d,0xf4240
              0x00007f3ee4fd74f0:   jge    0x00007f3ee4fd7509
              0x00007f3ee4fd74f2:   add    r11d,0x10
              0x00007f3ee4fd74f6:   xchg   ax,ax                        ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                                        ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 73)
              0x00007f3ee4fd74f8:   add    rax,QWORD PTR [r10+r11*8+0x10];*ladd {reexecute=0 rethrow=0 return_oop=0}
....................................................................................................
  98.44%  <total for region 1>


....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::longStride1, version 3, compile id 719 

   0.00%     0x00007f72e4fd549a:   cmp    edx,r11d
             0x00007f72e4fd549d:   mov    r10d,0x80000000
             0x00007f72e4fd54a3:   cmovl  r11d,r10d
             0x00007f72e4fd54a7:   movsxd r10,r11d
             0x00007f72e4fd54aa:   cmp    r10,rbp
             0x00007f72e4fd54ad:   cmovg  r11d,edi
             0x00007f72e4fd54b1:   cmp    r11d,0x2
             0x00007f72e4fd54b5:   jle    0x00007f72e4fd55ad
             0x00007f72e4fd54bb:   mov    r10d,0x2                     ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                                       ; - com.rkdeep.LoopUnrollBenchmark::longStride1@12 (line 85)
  10.74%  ↗  0x00007f72e4fd54c1:   cmp    r12d,DWORD PTR [rsp]
          │  0x00007f72e4fd54c5:   jae    0x00007f72e4fd5575
   0.02%  │  0x00007f72e4fd54cb:   add    rax,QWORD PTR [rcx+r12*8+0x10]
   0.31%  │  0x00007f72e4fd54d0:   movsxd rbx,r10d
   0.02%  │  0x00007f72e4fd54d3:   mov    r8,r9
  10.86%  │  0x00007f72e4fd54d6:   add    r8,rbx
          │  0x00007f72e4fd54d9:   mov    r12,rsi
   0.08%  │  0x00007f72e4fd54dc:   add    r12,rbx
   0.02%  │  0x00007f72e4fd54df:   mov    rbx,QWORD PTR [rcx+r12*8+0x48]
  20.31%  │  0x00007f72e4fd54e4:   mov    rdi,QWORD PTR [rcx+r12*8+0x40]
   0.31%  │  0x00007f72e4fd54e9:   mov    rdx,QWORD PTR [rcx+r12*8+0x38]
   0.44%  │  0x00007f72e4fd54ee:   mov    rbp,QWORD PTR [rcx+r12*8+0x30]
   0.19%  │  0x00007f72e4fd54f3:   mov    r13,QWORD PTR [rcx+r12*8+0x28]
  10.46%  │  0x00007f72e4fd54f8:   mov    r14,QWORD PTR [rcx+r12*8+0x20]
   0.03%  │  0x00007f72e4fd54fd:   mov    r12,QWORD PTR [rcx+r12*8+0x18];*laload {reexecute=0 rethrow=0 return_oop=0}
          │                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@19 (line 85)
   0.41%  │  0x00007f72e4fd5502:   add    rax,r12
   0.18%  │  0x00007f72e4fd5505:   add    rax,r14
  10.29%  │  0x00007f72e4fd5508:   add    rax,r13
   0.10%  │  0x00007f72e4fd550b:   add    rax,rbp
   0.47%  │  0x00007f72e4fd550e:   add    rax,rdx
  10.77%  │  0x00007f72e4fd5511:   add    rax,rdi
  11.09%  │  0x00007f72e4fd5514:   add    rax,rbx                      ;*ladd {reexecute=0 rethrow=0 return_oop=0}
          │                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 85)
  11.06%  │  0x00007f72e4fd5517:   add    r8,0x8
   0.02%  │  0x00007f72e4fd551b:   mov    r12d,r8d                     ;*l2i {reexecute=0 rethrow=0 return_oop=0}
          │                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 85)
          │  0x00007f72e4fd551e:   add    r10d,0x8
   0.01%  │  0x00007f72e4fd5522:   cmp    r10d,r11d
          ╰  0x00007f72e4fd5525:   jl     0x00007f72e4fd54c1           ;*ifge {reexecute=0 rethrow=0 return_oop=0}
                                                                       ; - com.rkdeep.LoopUnrollBenchmark::longStride1@9 (line 83)
             0x00007f72e4fd5527:   cmp    r10d,DWORD PTR [rsp+0x4]
             0x00007f72e4fd552c:   jge    0x00007f72e4fd5556
             0x00007f72e4fd552e:   xchg   ax,ax                        ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                                       ; - com.rkdeep.LoopUnrollBenchmark::longStride1@12 (line 85)
             0x00007f72e4fd5530:   cmp    r12d,DWORD PTR [rsp]
             0x00007f72e4fd5534:   jae    0x00007f72e4fd55cb
             0x00007f72e4fd553a:   add    rax,QWORD PTR [rcx+r12*8+0x10];*ladd {reexecute=0 rethrow=0 return_oop=0}
                                                                       ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 85)
....................................................................................................
  98.19%  <total for region 1>

Benchmark                             Mode  Cnt          Score          Error  Units
LoopUnrollBenchmark.baseline         thrpt    5  413725733.066 ± 20385130.808  ops/s
LoopUnrollBenchmark.baseline:asm     thrpt                 NaN                   ---
LoopUnrollBenchmark.intStride1       thrpt    5       2581.171 ±       14.527  ops/s
LoopUnrollBenchmark.intStride1:asm   thrpt                 NaN                   ---
LoopUnrollBenchmark.longStride1      thrpt    5       2427.188 ±        9.450  ops/s
LoopUnrollBenchmark.longStride1:asm  thrpt                 NaN                   ---
```
As we can see in assembly, the inner loop is not added to the code because there is no sense in it if the safepoint is removed. With int counter the loop is expanded for 16 iterations. The resulting code can be represented as in the listing below:
```java
        for (int j = 0; j < 1_000_000; j++) {
                sum += data[i];
                sum += data[i+1];
                sum += data[i+2];
                sum += data[i+3];
                sum += data[i+4];
                sum += data[i+5];
                sum += data[i+6];
                sum += data[i+7];
                sum += data[i+8];
                sum += data[i+9];
                sum += data[i+10];
                sum += data[i+11];
                sum += data[i+12];
                sum += data[i+13];
                sum += data[i+14];
                sum += data[i+15];
            }
        }
```
For a long counter, the loop is unrolled by 8 iterations of the loop body. You can also notice that with the long type, the registers `rbx, rdi, rdx, rbp, r13, r14, r12` are filled first and then the summation is performed. The resulting code can be represented as in the listing below:
```java
        for (long j = 0; j < 1_000_000; j++) {
                sum += data[i];
                sum += data[i+1];
                sum += data[i+2];
                sum += data[i+3];
                sum += data[i+4];
                sum += data[i+5];
                sum += data[i+6];
                sum += data[i+7];
            }
        }
```
As you can see, java 17 has significantly improved the handling of counter loops with long counter. The number of operations compared to int counter loop has increased from 56% in java 11 to 94% in java 17.

#### Benchmark with safepoints -XX:+UseCountedLoopSafepoints and -XX:LoopStripMiningIter=1000

We will run benchmark with the parameters
```java
@Fork(value = 1, jvmArgsPrepend = {"-XX:+UnlockDiagnosticVMOptions", "-XX:-UseCompressedOops", "-XX:PrintAssemblyOptions=intel", "-XX:LoopStripMiningIter=1000"})
```

```shell
....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::intStride1, version 3, compile id 720 

                0x00007f27f8fd76da:   jbe    0x00007f27f8fd77d8
                0x00007f27f8fd76e0:   cmp    r10d,0xf423f
                0x00007f27f8fd76e7:   jbe    0x00007f27f8fd77d8
                0x00007f27f8fd76ed:   mov    rax,QWORD PTR [r8+0x10]      ;*laload {reexecute=0 rethrow=0 return_oop=0}
                                                                          ; - com.rkdeep.LoopUnrollBenchmark::intStride1@16 (line 72)
                0x00007f27f8fd76f1:   mov    r12d,0x1                     ;*goto {reexecute=0 rethrow=0 return_oop=0}
                                                                          ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
                0x00007f27f8fd76f7:   mov    ebx,0x3e80
                0x00007f27f8fd76fc:   xor    ecx,ecx
          ╭     0x00007f27f8fd76fe:   jmp    0x00007f27f8fd777e
   0.02%  │↗    0x00007f27f8fd7703:   mov    r12d,r11d                    ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
          ││                                                              ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 72)
   5.60%  ││ ↗  0x00007f27f8fd7706:   add    rax,QWORD PTR [r8+r12*8+0x10]
   0.01%  ││ │  0x00007f27f8fd770b:   add    rax,QWORD PTR [r8+r12*8+0x18]
   5.73%  ││ │  0x00007f27f8fd7710:   add    rax,QWORD PTR [r8+r12*8+0x20]
   5.75%  ││ │  0x00007f27f8fd7715:   add    rax,QWORD PTR [r8+r12*8+0x28]
   5.48%  ││ │  0x00007f27f8fd771a:   add    rax,QWORD PTR [r8+r12*8+0x30]
   5.59%  ││ │  0x00007f27f8fd771f:   add    rax,QWORD PTR [r8+r12*8+0x38]
   9.69%  ││ │  0x00007f27f8fd7724:   add    rax,QWORD PTR [r8+r12*8+0x40]
   5.59%  ││ │  0x00007f27f8fd7729:   add    rax,QWORD PTR [r8+r12*8+0x48]
   5.77%  ││ │  0x00007f27f8fd772e:   add    rax,QWORD PTR [r8+r12*8+0x50]
   5.38%  ││ │  0x00007f27f8fd7733:   add    rax,QWORD PTR [r8+r12*8+0x58]
   6.13%  ││ │  0x00007f27f8fd7738:   add    rax,QWORD PTR [r8+r12*8+0x60]
   5.56%  ││ │  0x00007f27f8fd773d:   add    rax,QWORD PTR [r8+r12*8+0x68]
   5.58%  ││ │  0x00007f27f8fd7742:   add    rax,QWORD PTR [r8+r12*8+0x70]
   5.56%  ││ │  0x00007f27f8fd7747:   add    rax,QWORD PTR [r8+r12*8+0x78]
   9.13%  ││ │  0x00007f27f8fd774c:   add    rax,QWORD PTR [r8+r12*8+0x80]
   5.83%  ││ │  0x00007f27f8fd7754:   add    rax,QWORD PTR [r8+r12*8+0x88];*ladd {reexecute=0 rethrow=0 return_oop=0}
          ││ │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 72)
   5.61%  ││ │  0x00007f27f8fd775c:   mov    r11d,r12d
   0.00%  ││ │  0x00007f27f8fd775f:   add    r11d,0x10                    ;*iinc {reexecute=0 rethrow=0 return_oop=0}
          ││ │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@19 (line 70)
   0.00%  ││ │  0x00007f27f8fd7763:   cmp    r11d,r10d
          │╰ │  0x00007f27f8fd7766:   jl     0x00007f27f8fd7703           ;*if_icmpge {reexecute=0 rethrow=0 return_oop=0}
          │  │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@7 (line 70)
          │  │  0x00007f27f8fd7768:   mov    r9,QWORD PTR [r15+0x350]     ; ImmutableOopMap {r8=Oop rdi=Oop }
          │  │                                                            ;*goto {reexecute=1 rethrow=0 return_oop=0}
          │  │                                                            ; - (reexecute) com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
   0.01%  │  │  0x00007f27f8fd776f:   test   DWORD PTR [r9],eax           ;*goto {reexecute=0 rethrow=0 return_oop=0}
          │  │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
          │  │                                                            ;   {poll}
   0.02%  │  │  0x00007f27f8fd7772:   cmp    r11d,0xf4231
          │ ╭│  0x00007f27f8fd7779:   jge    0x00007f27f8fd77a5
          │ ││  0x00007f27f8fd777b:   mov    r12d,r11d
          ↘ ││  0x00007f27f8fd777e:   mov    r10d,0xf4231
   0.01%    ││  0x00007f27f8fd7784:   sub    r10d,r12d
            ││  0x00007f27f8fd7787:   cmp    r12d,0xf4231
            ││  0x00007f27f8fd778e:   cmovg  r10d,ecx
   0.00%    ││  0x00007f27f8fd7792:   cmp    r10d,0x3e80
            ││  0x00007f27f8fd7799:   cmova  r10d,ebx
   0.00%    ││  0x00007f27f8fd779d:   add    r10d,r12d
   0.00%    │╰  0x00007f27f8fd77a0:   jmp    0x00007f27f8fd7706
            ↘   0x00007f27f8fd77a5:   cmp    r11d,0xf4240
                0x00007f27f8fd77ac:   jge    0x00007f27f8fd77c5
                0x00007f27f8fd77ae:   add    r12d,0x10
                0x00007f27f8fd77b2:   xchg   ax,ax                        ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                                          ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 72)
                0x00007f27f8fd77b4:   add    rax,QWORD PTR [r8+r12*8+0x10];*ladd {reexecute=0 rethrow=0 return_oop=0}
                                                                          ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 72)
                0x00007f27f8fd77b9:   inc    r12d                         ;*iinc {reexecute=0 rethrow=0 return_oop=0}
                                                                          ; - com.rkdeep.LoopUnrollBenchmark::intStride1@19 (line 70)
                0x00007f27f8fd77bc:   cmp    r12d,0xf4240
....................................................................................................
  98.07%  <total for region 1>

....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::longStride1, version 3, compile id 717 

                 0x00007fe980fd651f:   cmovl  r10d,esi
                 0x00007fe980fd6523:   movsxd r9,r10d
                 0x00007fe980fd6526:   cmp    r9,r11
                 0x00007fe980fd6529:   cmovg  r10d,edi
                 0x00007fe980fd652d:   mov    DWORD PTR [rsp+0x8],r10d
                 0x00007fe980fd6532:   cmp    r10d,0x2
          ╭      0x00007fe980fd6536:   jle    0x00007fe980fd65e7
          │ ↗    0x00007fe980fd653c:   mov    r10d,DWORD PTR [rsp+0x8]
          │ │    0x00007fe980fd6541:   sub    r10d,ecx
          │ │    0x00007fe980fd6544:   mov    r9d,DWORD PTR [rsp+0x8]
   0.01%  │ │    0x00007fe980fd6549:   xor    r11d,r11d
          │ │    0x00007fe980fd654c:   cmp    r9d,ecx
   0.00%  │ │    0x00007fe980fd654f:   cmovl  r10d,r11d
   0.01%  │ │    0x00007fe980fd6553:   cmp    r10d,0x1f40
   0.00%  │ │    0x00007fe980fd655a:   mov    r9d,0x1f40
          │ │    0x00007fe980fd6560:   cmova  r10d,r9d
   0.01%  │ │    0x00007fe980fd6564:   add    r10d,ecx
   0.00%  │ │    0x00007fe980fd6567:   nop    WORD PTR [rax+rax*1+0x0]     ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                              ; - com.rkdeep.LoopUnrollBenchmark::longStride1@12 (line 84)
  11.22%  │↗│    0x00007fe980fd6570:   cmp    ebx,DWORD PTR [rsp]
          │││    0x00007fe980fd6573:   jae    0x00007fe980fd6628
   0.03%  │││    0x00007fe980fd6579:   add    rax,QWORD PTR [r8+rbx*8+0x10]
   0.40%  │││    0x00007fe980fd657e:   movsxd r9,ecx
   0.04%  │││    0x00007fe980fd6581:   mov    rdx,r14
  10.88%  │││    0x00007fe980fd6584:   add    rdx,r9
   0.01%  │││    0x00007fe980fd6587:   mov    r11,rbp
   0.10%  │││    0x00007fe980fd658a:   add    r11,r9
   0.05%  │││    0x00007fe980fd658d:   mov    r9,QWORD PTR [r8+r11*8+0x48]
  18.17%  │││    0x00007fe980fd6592:   mov    r12,QWORD PTR [r8+r11*8+0x40]
   0.30%  │││    0x00007fe980fd6597:   mov    rbx,QWORD PTR [r8+r11*8+0x38]
   0.47%  │││    0x00007fe980fd659c:   mov    rdi,QWORD PTR [r8+r11*8+0x30]
   0.20%  │││    0x00007fe980fd65a1:   mov    rsi,QWORD PTR [r8+r11*8+0x28]
  10.59%  │││    0x00007fe980fd65a6:   mov    r13,QWORD PTR [r8+r11*8+0x20]
   0.09%  │││    0x00007fe980fd65ab:   mov    r11,QWORD PTR [r8+r11*8+0x18];*laload {reexecute=0 rethrow=0 return_oop=0}
          │││                                                              ; - com.rkdeep.LoopUnrollBenchmark::longStride1@19 (line 84)
   0.44%  │││    0x00007fe980fd65b0:   add    rax,r11
   0.22%  │││    0x00007fe980fd65b3:   add    rax,r13
  10.65%  │││    0x00007fe980fd65b6:   add    rax,rsi
   0.15%  │││    0x00007fe980fd65b9:   add    rax,rdi
   0.57%  │││    0x00007fe980fd65bc:   add    rax,rbx
  10.83%  │││    0x00007fe980fd65bf:   add    rax,r12
  11.50%  │││    0x00007fe980fd65c2:   add    rax,r9                       ;*ladd {reexecute=0 rethrow=0 return_oop=0}
          │││                                                              ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 84)
  11.28%  │││    0x00007fe980fd65c5:   add    rdx,0x8
   0.01%  │││    0x00007fe980fd65c9:   mov    ebx,edx                      ;*l2i {reexecute=0 rethrow=0 return_oop=0}
          │││                                                              ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 84)
   0.02%  │││    0x00007fe980fd65cb:   add    ecx,0x8
   0.04%  │││    0x00007fe980fd65ce:   cmp    ecx,r10d
          │╰│    0x00007fe980fd65d1:   jl     0x00007fe980fd6570           ;*ifge {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                              ; - com.rkdeep.LoopUnrollBenchmark::longStride1@9 (line 82)
   0.02%  │ │    0x00007fe980fd65d3:   mov    r10,QWORD PTR [r15+0x350]    ; ImmutableOopMap {r8=Oop xmm0=Oop }
          │ │                                                              ;*goto {reexecute=1 rethrow=0 return_oop=0}
          │ │                                                              ; - (reexecute) com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
   0.01%  │ │    0x00007fe980fd65da:   test   DWORD PTR [r10],eax          ;*goto {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                              ; - com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
          │ │                                                              ;   {poll}
   0.11%  │ │    0x00007fe980fd65dd:   cmp    ecx,DWORD PTR [rsp+0x8]
          │ ╰    0x00007fe980fd65e1:   jl     0x00007fe980fd653c
          ↘      0x00007fe980fd65e7:   cmp    ecx,DWORD PTR [rsp+0x4]
             ╭   0x00007fe980fd65eb:   jge    0x00007fe980fd660e
   0.00%     │   0x00007fe980fd65ed:   data16 xchg ax,ax                   ;*l2i {reexecute=0 rethrow=0 return_oop=0}
             │                                                             ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 84)
             │↗  0x00007fe980fd65f0:   cmp    ebx,DWORD PTR [rsp]
             ││  0x00007fe980fd65f3:   jae    0x00007fe980fd666f
             ││  0x00007fe980fd65f5:   add    rax,QWORD PTR [r8+rbx*8+0x10];*ladd {reexecute=0 rethrow=0 return_oop=0}
             ││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 84)
   0.00%     ││  0x00007fe980fd65fa:   movsxd rdx,ecx
             ││  0x00007fe980fd65fd:   add    rdx,r14
             ││  0x00007fe980fd6600:   add    rdx,0x1
             ││  0x00007fe980fd6604:   mov    ebx,edx                      ;*l2i {reexecute=0 rethrow=0 return_oop=0}
             ││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 84)
   0.00%     ││  0x00007fe980fd6606:   inc    ecx
             ││  0x00007fe980fd6608:   cmp    ecx,DWORD PTR [rsp+0x4]
             │╰  0x00007fe980fd660c:   jl     0x00007fe980fd65f0           ;*ifge {reexecute=0 rethrow=0 return_oop=0}
             │                                                             ; - com.rkdeep.LoopUnrollBenchmark::longStride1@9 (line 82)
             ↘   0x00007fe980fd660e:   vmovq  r11,xmm0
                 0x00007fe980fd6613:   mov    r10d,DWORD PTR [rsp]
                 0x00007fe980fd6617:   cmp    rdx,0xf4240
                 0x00007fe980fd661e:   jge    0x00007fe980fd665c
                 0x00007fe980fd6620:   mov    r14,rdx
                 0x00007fe980fd6623:   jmp    0x00007fe980fd645e
....................................................................................................
  98.42%  <total for region 1>

Benchmark                             Mode  Cnt          Score          Error  Units
LoopUnrollBenchmark.baseline         thrpt    5  419882701.472 ± 13085589.188  ops/s
LoopUnrollBenchmark.baseline:asm     thrpt                 NaN                   ---
LoopUnrollBenchmark.intStride1       thrpt    5       2493.944 ±      102.343  ops/s
LoopUnrollBenchmark.intStride1:asm   thrpt                 NaN                   ---
LoopUnrollBenchmark.longStride1      thrpt    5       2446.834 ±      231.035  ops/s
LoopUnrollBenchmark.longStride1:asm  thrpt                 NaN                   ---
```
As you can see in the assembly code, both benchmarks unroll the loop in long for 8, int for 16 iterations respectively. An inner loop and a safepoint after it are added.

### java 21
```terminal
# JMH version: 1.37
# VM version: JDK 21.0.1, OpenJDK 64-Bit Server VM, 21.0.1+12-LTS
# VM invoker: /home/kirill/.sdkman/candidates/java/21.0.1-tem/bin/java
# VM options: -XX:+UnlockDiagnosticVMOptions -XX:-UseCompressedOops -XX:PrintAssemblyOptions=intel -XX:LoopStripMiningIter=1000

....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::intStride1, version 3, compile id 764 

   0.01%         0x0000747cd43da1c5:   test   r10d,r10d
                 0x0000747cd43da1c8:   jbe    0x0000747cd43da2ac
                 0x0000747cd43da1ce:   cmp    r10d,0xf423f
                 0x0000747cd43da1d5:   jbe    0x0000747cd43da2ac
                 0x0000747cd43da1db:   mov    rax,QWORD PTR [r9+0x10]      ;*laload {reexecute=0 rethrow=0 return_oop=0}
                                                                           ; - com.rkdeep.LoopUnrollBenchmark::intStride1@16 (line 72)
                 0x0000747cd43da1df:   mov    r10d,0x1                     ;*goto {reexecute=0 rethrow=0 return_oop=0}
                                                                           ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
                 0x0000747cd43da1e5:   mov    r8d,0x3e80
          ╭      0x0000747cd43da1eb:   jmp    0x0000747cd43da268
   0.00%  │↗     0x0000747cd43da1f0:   mov    r10d,ebx                     ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
          ││                                                               ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 72)
   0.00%  ││ ↗   0x0000747cd43da1f3:   add    rax,QWORD PTR [r9+r10*8+0x10]
   5.74%  ││ │   0x0000747cd43da1f8:   add    rax,QWORD PTR [r9+r10*8+0x18]
   6.09%  ││ │   0x0000747cd43da1fd:   add    rax,QWORD PTR [r9+r10*8+0x20]
   5.56%  ││ │   0x0000747cd43da202:   add    rax,QWORD PTR [r9+r10*8+0x28]
   5.94%  ││ │   0x0000747cd43da207:   add    rax,QWORD PTR [r9+r10*8+0x30]
   5.88%  ││ │   0x0000747cd43da20c:   add    rax,QWORD PTR [r9+r10*8+0x38]
   9.17%  ││ │   0x0000747cd43da211:   add    rax,QWORD PTR [r9+r10*8+0x40]
   5.87%  ││ │   0x0000747cd43da216:   add    rax,QWORD PTR [r9+r10*8+0x48]
   5.66%  ││ │   0x0000747cd43da21b:   add    rax,QWORD PTR [r9+r10*8+0x50]
   5.61%  ││ │   0x0000747cd43da220:   add    rax,QWORD PTR [r9+r10*8+0x58]
   5.83%  ││ │   0x0000747cd43da225:   add    rax,QWORD PTR [r9+r10*8+0x60]
   5.73%  ││ │   0x0000747cd43da22a:   add    rax,QWORD PTR [r9+r10*8+0x68]
   5.56%  ││ │   0x0000747cd43da22f:   add    rax,QWORD PTR [r9+r10*8+0x70]
   5.67%  ││ │   0x0000747cd43da234:   add    rax,QWORD PTR [r9+r10*8+0x78]
   9.08%  ││ │   0x0000747cd43da239:   add    rax,QWORD PTR [r9+r10*8+0x80]
   5.86%  ││ │   0x0000747cd43da241:   add    rax,QWORD PTR [r9+r10*8+0x88];*ladd {reexecute=0 rethrow=0 return_oop=0}
          ││ │                                                             ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 72)
   5.65%  ││ │   0x0000747cd43da249:   lea    ebx,[r10+0x10]
   0.00%  ││ │   0x0000747cd43da24d:   cmp    ebx,r12d
          │╰ │   0x0000747cd43da250:   jl     0x0000747cd43da1f0           ;*if_icmpge {reexecute=0 rethrow=0 return_oop=0}
          │  │                                                             ; - com.rkdeep.LoopUnrollBenchmark::intStride1@7 (line 70)
          │  │   0x0000747cd43da252:   mov    r12,QWORD PTR [r15+0x458]    ; ImmutableOopMap {r11=Oop r9=Oop }
          │  │                                                             ;*goto {reexecute=1 rethrow=0 return_oop=0}
          │  │                                                             ; - (reexecute) com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
   0.01%  │  │   0x0000747cd43da259:   test   DWORD PTR [r12],eax          ;*goto {reexecute=0 rethrow=0 return_oop=0}
          │  │                                                             ; - com.rkdeep.LoopUnrollBenchmark::intStride1@22 (line 70)
          │  │                                                             ;   {poll}
   0.01%  │  │   0x0000747cd43da25d:   cmp    ebx,0xf4231
          │ ╭│   0x0000747cd43da263:   jge    0x0000747cd43da284
          │ ││   0x0000747cd43da265:   mov    r10d,ebx
          ↘ ││   0x0000747cd43da268:   mov    r12d,0xf4231
            ││   0x0000747cd43da26e:   sub    r12d,r10d
   0.01%    ││   0x0000747cd43da271:   cmp    r12d,0x3e80
            ││   0x0000747cd43da278:   cmova  r12d,r8d
   0.01%    ││   0x0000747cd43da27c:   add    r12d,r10d
            │╰   0x0000747cd43da27f:   jmp    0x0000747cd43da1f3
            ↘    0x0000747cd43da284:   add    r10d,0x10                    ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                                           ; - com.rkdeep.LoopUnrollBenchmark::intStride1@10 (line 72)
              ↗  0x0000747cd43da288:   add    rax,QWORD PTR [r9+r10*8+0x10];*ladd {reexecute=0 rethrow=0 return_oop=0}
              │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@17 (line 72)
              │  0x0000747cd43da28d:   inc    r10d                         ;*iinc {reexecute=0 rethrow=0 return_oop=0}
              │                                                            ; - com.rkdeep.LoopUnrollBenchmark::intStride1@19 (line 70)
              │  0x0000747cd43da290:   cmp    r10d,0xf4240
              ╰  0x0000747cd43da297:   jl     0x0000747cd43da288
                 0x0000747cd43da299:   add    rsp,0x10
....................................................................................................
  98.97%  <total for region 1>


....[Hottest Region 1]..............................................................................
c2, level 4, com.rkdeep.LoopUnrollBenchmark::longStride1, version 3, compile id 764 

               0x00007f7c7c3d94e0:   lea    r14d,[r13-0x4]
               0x00007f7c7c3d94e4:   mov    r8d,0x2
               0x00007f7c7c3d94ea:   cmp    r14d,0x2
          ╭    0x00007f7c7c3d94ee:   jle    0x00007f7c7c3d95cf
          │    0x00007f7c7c3d94f4:   vmovd  xmm1,r12d
          │    0x00007f7c7c3d94f9:   mov    rdi,rdx
          │    0x00007f7c7c3d94fc:   add    rdi,rbx
          │    0x00007f7c7c3d94ff:   mov    r12d,esi
          │    0x00007f7c7c3d9502:   vmovq  xmm0,rcx
          │ ↗  0x00007f7c7c3d9507:   mov    ebx,r13d
   0.00%  │ │  0x00007f7c7c3d950a:   sub    ebx,r8d
          │ │  0x00007f7c7c3d950d:   add    ebx,0xfffffffc
   0.01%  │ │  0x00007f7c7c3d9510:   xor    r9d,r9d
          │ │  0x00007f7c7c3d9513:   cmp    r14d,r8d
          │ │  0x00007f7c7c3d9516:   cmovl  ebx,r9d
   0.00%  │ │  0x00007f7c7c3d951a:   cmp    ebx,0x1f40
          │ │  0x00007f7c7c3d9520:   mov    r9d,0x1f40
   0.00%  │ │  0x00007f7c7c3d9526:   cmova  ebx,r9d
   0.01%  │ │  0x00007f7c7c3d952a:   add    ebx,r8d
   0.00%  │ │  0x00007f7c7c3d952d:   data16 xchg ax,ax                   ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@12 (line 84)
  10.97%  │↗│  0x00007f7c7c3d9530:   cmp    r11d,r10d
          │││  0x00007f7c7c3d9533:   jae    0x00007f7c7c3d95fd
   0.03%  │││  0x00007f7c7c3d9539:   add    rax,QWORD PTR [rdx+r11*8+0x10]
   0.44%  │││  0x00007f7c7c3d953e:   lea    r11d,[r12+r8*1]
   0.03%  │││  0x00007f7c7c3d9542:   movsxd rcx,r8d
  10.59%  │││  0x00007f7c7c3d9545:   mov    r9,QWORD PTR [rdi+rcx*8+0x18]
   0.06%  │││  0x00007f7c7c3d954a:   mov    rbp,QWORD PTR [rdi+rcx*8+0x20];*laload {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@19 (line 84)
   0.21%  │││  0x00007f7c7c3d954f:   add    rax,r9
   0.11%  │││  0x00007f7c7c3d9552:   add    rax,rbp
  10.87%  │││  0x00007f7c7c3d9555:   lea    r9d,[r11+0x3]
   0.02%  │││  0x00007f7c7c3d9559:   cmp    r9d,r10d
          │││  0x00007f7c7c3d955c:   jae    0x00007f7c7c3d9606
   0.01%  │││  0x00007f7c7c3d9562:   add    rax,QWORD PTR [rdi+rcx*8+0x28];*ladd {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 84)
  11.07%  │││  0x00007f7c7c3d9567:   add    r11d,0x4                     ;*l2i {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 84)
   0.04%  │││  0x00007f7c7c3d956b:   cmp    r11d,r10d
          │││  0x00007f7c7c3d956e:   jae    0x00007f7c7c3d95f9
   0.04%  │││  0x00007f7c7c3d9574:   add    rax,QWORD PTR [rdi+rcx*8+0x30]
  19.82%  │││  0x00007f7c7c3d9579:   mov    r11,QWORD PTR [rdi+rcx*8+0x40]
   0.16%  │││  0x00007f7c7c3d957e:   mov    r9,QWORD PTR [rdi+rcx*8+0x38];*laload {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@19 (line 84)
   0.09%  │││  0x00007f7c7c3d9583:   lea    rbp,[rsi+rcx*1]
   0.02%  │││  0x00007f7c7c3d9587:   add    rax,r9
  11.11%  │││  0x00007f7c7c3d958a:   add    rax,r11                      ;*ladd {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 84)
  11.23%  │││  0x00007f7c7c3d958d:   mov    r9d,ebp                      ;   {no_reloc}
   0.02%  │││  0x00007f7c7c3d9590:   add    r9d,0x7                      ;*l2i {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 84)
   0.02%  │││  0x00007f7c7c3d9594:   cmp    r9d,r10d
          │││  0x00007f7c7c3d9597:   jae    0x00007f7c7c3d9602
   0.02%  │││  0x00007f7c7c3d9599:   add    rax,QWORD PTR [rdi+rcx*8+0x48];*ladd {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@20 (line 84)
  10.89%  │││  0x00007f7c7c3d959e:   add    rbp,0x8
   0.02%  │││  0x00007f7c7c3d95a2:   mov    r11d,ebp                     ;*l2i {reexecute=0 rethrow=0 return_oop=0}
          │││                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@18 (line 84)
   0.02%  │││  0x00007f7c7c3d95a5:   add    r8d,0x8
   0.04%  │││  0x00007f7c7c3d95a9:   cmp    r8d,ebx
          │╰│  0x00007f7c7c3d95ac:   jl     0x00007f7c7c3d9530           ;*ifge {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@9 (line 82)
   0.02%  │ │  0x00007f7c7c3d95b2:   mov    r9,QWORD PTR [r15+0x458]     ; ImmutableOopMap {rdx=Oop rdi=Derived_oop_rdx xmm0=Oop }
          │ │                                                            ;*goto {reexecute=1 rethrow=0 return_oop=0}
          │ │                                                            ; - (reexecute) com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
   0.07%  │ │  0x00007f7c7c3d95b9:   test   DWORD PTR [r9],eax           ;*goto {reexecute=0 rethrow=0 return_oop=0}
          │ │                                                            ; - com.rkdeep.LoopUnrollBenchmark::longStride1@26 (line 82)
          │ │                                                            ;   {poll}
   0.15%  │ │  0x00007f7c7c3d95bc:   cmp    r8d,r14d
          │ ╰  0x00007f7c7c3d95bf:   jl     0x00007f7c7c3d9507
          │    0x00007f7c7c3d95c5:   vmovq  rcx,xmm0
          │    0x00007f7c7c3d95ca:   vmovd  r12d,xmm1
          ↘    0x00007f7c7c3d95cf:   cmp    r8d,r12d
               0x00007f7c7c3d95d2:   jge    0x00007f7c7c3d9647           ;*lload_1 {reexecute=0 rethrow=0 return_oop=0}
                                                                         ; - com.rkdeep.LoopUnrollBenchmark::longStride1@12 (line 84)
               0x00007f7c7c3d95d4:   cmp    r11d,r10d
               0x00007f7c7c3d95d7:   jae    0x00007f7c7c3d967b
....................................................................................................
  98.25%  <total for region 1>

Benchmark                             Mode  Cnt          Score          Error  Units
LoopUnrollBenchmark.baseline         thrpt    5  435780715.553 ± 18841125.750  ops/s
LoopUnrollBenchmark.baseline:asm     thrpt                 NaN                   ---
LoopUnrollBenchmark.intStride1       thrpt    5       2605.923 ±      224.247  ops/s
LoopUnrollBenchmark.intStride1:asm   thrpt                 NaN                   ---
LoopUnrollBenchmark.longStride1      thrpt    5       2404.129 ±       66.965  ops/s
LoopUnrollBenchmark.longStride1:asm  thrpt                 NaN                   ---
```

As we could see in Java 21 in loops with int counter unrolled similar to Java 17. As expected with int counter loop unrolled by 16 and with long counter unrolled by 8.

## Conclusion
Java 11 applies optimisations differently to loops with int and long counters. In java 11, a loop with long counter is ~2 times slower to execute than with int. In java 17, loop strip mining and safepoint control optimisations have been added, and as a consequence, an inner loop with a safepoint after it has been added. This makes it possible to control the frequency of safepoint checking when executing loops.

## Memory hierarchy
Let's look at the cache hierarchy and data access time. The time depends on the hardware and is given to illustrate how many times each memory is faster or slower than the other.
### Processor registers
Processor registers have the fastest access (typically 1 processor cycle) and can be as small as a few bytes.
### L1 cache
L1 caches (the instruction and data caches) are typically 128 kilobytes in size. Access time is around 0.5 nanoseconds for the data cache and 5 nanoseconds for a branch misprediction for the instruction cache.
### L2 cache
L2 caches - instruction and data (shared) - vary in size but can range from 256 kilobytes to 8 megabytes. L2 cache access time is around 5-7 nanoseconds.
### L3 Cache
L3 is a shared cache. L3 caches can vary in size from 32 to 64 megabytes. The L3 cache is the largest, but also the slowest, with access times of around 12 nanoseconds. The L3 cache can be on the CPU itself, but there are L1 and L2 caches for each core, while the L3 cache is more of a shared cache for all the cores on the chip.
### Main memory
Main memory (primary storage) – this varies in size from 16 to 256 gigabytes. Access times are around 60 nanoseconds.
### Disk storage
Disk storage (secondary storage) - This goes up to terabytes in size. Access speeds are around 100 microseconds for solid-state storage and around 300 microseconds for non-solid-state storage.

Access times are given above for imaging, how many times each memory is faster or slower than the other.

Let's define an array with 64 * 1024 * 1024 integer elements. In the code below we define two loops, the first iterates and multiplies each element by 4. The second loop iterates and multiplies each element by 16. Take a benchmark for two implementations. Which implementation is faster and how often? The second loop is faster in 1/16 of the time, isn't it?

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 2, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 2, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(2)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class CpuCache {
    private static final int ARRAY_SIZE = 64 * 1024 * 1024;
    public int[] array;

    @Setup(Level.Iteration)
    public void setUp() {
        array = new int[ARRAY_SIZE];
        Arrays.fill(array, 1);
    }

    @Benchmark
    public void baseline() {

    }

    @Benchmark
    public void accessfieldby_16(Blackhole bh) {
        for (int i = 0, n = array.length; i < n; i+=16) {
            array[i] *= 3;
        }
    }

    @Benchmark
    public void accessfieldby_1(Blackhole bh) {
        for (int i = 0, n = array.length; i < n; i++) {
            array[i] *= 3;
        }
    }
}
```
Consider the result of the benchmarks.

```terminal
Benchmark                  Mode  Cnt   Score    Error  Units
CpuCache.accessfieldby_1   avgt    4  18.675 ±  2.091  ms/op
CpuCache.accessfieldby_16  avgt    4  18.702 ±  0.447  ms/op
CpuCache.baseline          avgt    4  ≈ 10⁻⁶           ms/op
```

Why does it take the same time to multiply every 16th element round as it does to multiply every single element?
The second loop only does a fraction of the work, so how is it possible for the first loop to run at the same speed? The answer lies in how the CPU uses the cache and why we use every 16th element. On my laptop, I got an L1 data and instruction cache of 32 kB per core, with a cache line size of 64 bytes.
```terminal
kyryl-rybkin@pc:~$ lscpu | grep "cache"
L1d cache:                       192 KiB (6 instances)
L1i cache:                       192 KiB (6 instances)
```

```terminal
kyryl-rybkin@pc:~$ getconf -a | grep CACHE
LEVEL1_ICACHE_SIZE                 32768
LEVEL1_ICACHE_ASSOC                
LEVEL1_ICACHE_LINESIZE             64
LEVEL1_DCACHE_SIZE                 32768
LEVEL1_DCACHE_ASSOC                8
LEVEL1_DCACHE_LINESIZE             64
```

If we iterate over the data sequentially or per cache line, we can get roughly the same number of operations per second because L1 is closer to the core and fast enough.
## Data size and L1, L2, and L3 caches
Let's consider whether the size of a data structure affects the runtime performance of software. We can do a little experiment. First, define the size of the array as params for the benchmark. Second, define a benchmark loop that could evaluate over the array and do the trivial operations, e.g. multiply by 3 on each cache line.

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class ArraySize {
    private static final int ARRAY_CONTENT = 777;
    @Param({"1024", "2048", "4096", "8192", "16384", "32768", "65536", "131072", "262144", "524288", "1048576", "2097152", "4194304", "8388608", "16777216", "33554432", "67108864", "134217728", "268435456", "536870912"})
    public int size;
    public int[] array;
    public int counter;
    public int mask;

    @Setup(Level.Iteration)
    public void setUp() {
        final int elements = size / 4;
        final int indexes = elements / 16;
        mask = indexes - 1;
        array = new int[elements];
        Arrays.fill(array, ARRAY_CONTENT);
    }

    @Benchmark
    public void benchLoop() {
        array[16 * counter] *= 3;
        counter = (counter + 1) & mask;
    }
}
```

```terminal
Benchmark               (size)  Mode  Cnt  Score   Error  Units
ArraySize.benchLoop       1024  avgt   25  1.296 ± 0.001  ns/op
ArraySize.benchLoop       4096  avgt   25  1.303 ± 0.009  ns/op
ArraySize.benchLoop       8192  avgt   25  1.297 ± 0.021  ns/op
ArraySize.benchLoop      32768  avgt   25  1.357 ± 0.058  ns/op
ArraySize.benchLoop      49152  avgt   25  1.345 ± 0.036  ns/op
ArraySize.benchLoop      65536  avgt   25  1.417 ± 0.039  ns/op
ArraySize.benchLoop      98304  avgt   25  1.320 ± 0.026  ns/op
ArraySize.benchLoop     131072  avgt   25  1.415 ± 0.019  ns/op
ArraySize.benchLoop     196608  avgt   25  1.415 ± 0.037  ns/op
ArraySize.benchLoop     229376  avgt   25  1.398 ± 0.027  ns/op
ArraySize.benchLoop     262144  avgt   25  1.542 ± 0.029  ns/op
ArraySize.benchLoop     393216  avgt   25  1.411 ± 0.037  ns/op
ArraySize.benchLoop     524288  avgt   25  1.610 ± 0.034  ns/op
ArraySize.benchLoop    1048576  avgt   25  1.636 ± 0.025  ns/op
ArraySize.benchLoop    1572864  avgt   25  1.668 ± 0.063  ns/op
ArraySize.benchLoop    2097152  avgt   25  1.920 ± 0.038  ns/op
ArraySize.benchLoop    6291456  avgt   25  1.917 ± 0.036  ns/op
ArraySize.benchLoop    8388608  avgt   25  3.327 ± 0.109  ns/op
ArraySize.benchLoop   10485760  avgt   25  1.905 ± 0.078  ns/op
ArraySize.benchLoop   16777216  avgt   25  4.451 ± 0.287  ns/op
ArraySize.benchLoop   33554432  avgt   25  4.932 ± 0.030  ns/op
ArraySize.benchLoop   67108864  avgt   25  5.125 ± 0.041  ns/op
ArraySize.benchLoop  134217728  avgt   25  5.177 ± 0.020  ns/op
ArraySize.benchLoop  268435456  avgt   25  5.201 ± 0.033  ns/op
ArraySize.benchLoop  536870912  avgt   25  5.320 ± 0.274  ns/op
```
As we know, accessing a single element takes a constant time in Big O notation, O(1). But on the hardware we get a different cache level which has a different access time for stored data. If we iterate over the L1-L3 cache, it takes less than 2 ns/op to operate. But when we load data from main memory, it takes more time. If you look closely, you can even see the small jump between L1, L2 and L3 cache, with the access time increasing from L1 to L3 cache size.
![Array-size](./img/array-size.png)

Size matters. The smaller the amount of data you use for a particular data structure, the more likely it is to fit in the cache, which can lead to significantly better performance.

## Data access patterns. Random or Sequential memory access
Does the order in which the data is accessed have an impact? Instead of just running sequentially through the array, let's create a second array to store the order in which the data is accessed. In the first case, access the array sequentially as before, and then access it in any order and measure the difference. Take a look at the listing below.
```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 5, time = 1)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class RandomAccess {

    private static final int ARRAY_CONTENT = 777;
    @Param({"1024", "2048", "4096", "8192", "16384", "32768", "65536", "131072", "262144", "524288", "1048576", "2097152", "4194304", "8388608", "16777216", "33554432", "67108864", "134217728", "268435456", "536870912"})
    public int size;
    public int[] array;
    public int seqcounter;
    public int randomcounter;
    public int mask;
    public int[] rndIndex;
    public int[] seqIndex;

    @Setup(Level.Iteration)
    public void setUp() {
        final int elements = size / 4;
        final int indexes = elements / 16;
        mask = indexes - 1;
        array = new int[elements];
        Arrays.fill(array, ARRAY_CONTENT);
        rndIndex = new int[indexes];
        seqIndex = new int[indexes];
        seqcounter = 0;
        randomcounter = 0;
        final List<Integer> list = new ArrayList<>(indexes);
        for (int i=0; i<indexes; i++) {
            list.add(16 * i);
        }                          
        Collections.shuffle(list);
        for (int i=0; i<indexes; i++) {
            rndIndex[i] = list.get(i);
        }

        for (int i = 0; i < indexes; i++) {
            seqIndex[i] = i*16;
        }
    }

    @Benchmark
    public void randomAccess(Blackhole bh) {
        array[rndIndex[randomcounter]] *= 3;
        randomcounter = (randomcounter + 1) & mask;
    }

    @Benchmark
    public void seqAccess(Blackhole bh) {
        array[seqIndex[seqcounter]] *= 3;
        seqcounter = (seqcounter + 1) & mask;
    }
}
```
```terminal
Benchmark                     (size)  Mode  Cnt   Score   Error  Units
RandomAccess.randomAccess       1024  avgt   25   1.511 ± 0.146  ns/op
RandomAccess.randomAccess       2048  avgt   25   1.472 ± 0.034  ns/op
RandomAccess.randomAccess       4096  avgt   25   1.448 ± 0.063  ns/op
RandomAccess.randomAccess       8192  avgt   25   1.429 ± 0.010  ns/op
RandomAccess.randomAccess      16384  avgt   25   1.477 ± 0.028  ns/op
RandomAccess.randomAccess      32768  avgt   25   1.588 ± 0.031  ns/op
RandomAccess.randomAccess      65536  avgt   25   1.671 ± 0.108  ns/op
RandomAccess.randomAccess     131072  avgt   25   1.736 ± 0.112  ns/op
RandomAccess.randomAccess     262144  avgt   25   2.256 ± 0.047  ns/op
RandomAccess.randomAccess     524288  avgt   25   2.559 ± 0.104  ns/op
RandomAccess.randomAccess    1048576  avgt   25   2.708 ± 0.084  ns/op
RandomAccess.randomAccess    2097152  avgt   25   2.746 ± 0.051  ns/op
RandomAccess.randomAccess    4194304  avgt   25   2.762 ± 0.089  ns/op
RandomAccess.randomAccess    8388608  avgt   25   5.918 ± 1.037  ns/op
RandomAccess.randomAccess   16777216  avgt   25  13.802 ± 0.815  ns/op
RandomAccess.randomAccess   33554432  avgt   25  14.856 ± 0.449  ns/op
RandomAccess.randomAccess   67108864  avgt   25  15.767 ± 0.647  ns/op
RandomAccess.randomAccess  134217728  avgt   25  16.016 ± 0.488  ns/op
RandomAccess.randomAccess  268435456  avgt   25  16.138 ± 0.322  ns/op
RandomAccess.randomAccess  536870912  avgt   25  16.286 ± 0.264  ns/op
RandomAccess.seqAccess          1024  avgt   25   1.480 ± 0.026  ns/op
RandomAccess.seqAccess          2048  avgt   25   1.462 ± 0.024  ns/op
RandomAccess.seqAccess          4096  avgt   25   1.449 ± 0.018  ns/op
RandomAccess.seqAccess          8192  avgt   25   1.452 ± 0.041  ns/op
RandomAccess.seqAccess         16384  avgt   25   1.518 ± 0.113  ns/op
RandomAccess.seqAccess         32768  avgt   25   1.521 ± 0.011  ns/op
RandomAccess.seqAccess         65536  avgt   25   1.642 ± 0.054  ns/op
RandomAccess.seqAccess        131072  avgt   25   1.664 ± 0.044  ns/op
RandomAccess.seqAccess        262144  avgt   25   1.785 ± 0.030  ns/op
RandomAccess.seqAccess        524288  avgt   25   1.928 ± 0.041  ns/op
RandomAccess.seqAccess       1048576  avgt   25   1.989 ± 0.074  ns/op
RandomAccess.seqAccess       2097152  avgt   25   2.029 ± 0.100  ns/op
RandomAccess.seqAccess       4194304  avgt   25   2.077 ± 0.097  ns/op
RandomAccess.seqAccess       8388608  avgt   25   2.827 ± 0.328  ns/op
RandomAccess.seqAccess      16777216  avgt   25   5.133 ± 0.480  ns/op
RandomAccess.seqAccess      33554432  avgt   25   5.680 ± 0.352  ns/op
RandomAccess.seqAccess      67108864  avgt   25   5.599 ± 0.374  ns/op
RandomAccess.seqAccess     134217728  avgt   25   5.353 ± 0.441  ns/op
RandomAccess.seqAccess     268435456  avgt   25   5.236 ± 0.330  ns/op
RandomAccess.seqAccess     536870912  avgt   25   5.053 ± 0.145  ns/op
```
As a result, both curves show almost the same access time with caches L1, L2 and L3.
The performance of the sequential access scheme is noticeably higher than in the random access scheme, especially noticeable when loading data from memory.
![Impact random access on latency](./img/random-access.png)

![Cache misses level impact of random access](./img/cache-messes-random-access.png)

The ratio between successful loads and cache misses is very different for type data accesses. When the array is accessed sequentially, only about 6% of all memory loads result in cache misses. When data is accessed randomly, the number of cache misses increases. The high number of cache misses is quite expected, since the array does not fit in the cache and you have to load everything from RAM. But why are there almost no cache misses when accessing the array sequentially?
Because there is a prefetcher that loads the data of the next cache line when interacting with the current line. That's why we can observe a low level of cache misses when sequentially accessing a small-sized array.

## Performance penalty with False Sharing
```java

@State(Scope.Group)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(2)
@Threads(2)
public class FalseSharing {
    public final int[] array = new int[17];

    @Benchmark
    @Group("near")
    @GroupThreads(1)
    public void modifyNearA() {
        array[0]++;
    }
    @Benchmark
    @Group("near")
    @GroupThreads(1)
    public void modifyNearB() {
        array[1]++;
    }
    @Benchmark
    @Group("far")
    @GroupThreads(1)
    public void modifyFarA() {
        array[0]++;
    }
    @Benchmark
    @Group("far")
    @GroupThreads(1)
    public void modifyFarB() {
        array[16]++;
    }
}
```
```terminal
Benchmark                       Mode  Cnt     Score     Error   Units
FalseSharing.baseline          thrpt   25  1648.375 ± 531.950  ops/us
FalseSharing.baseline:reader   thrpt   25   801.301 ± 522.013  ops/us
FalseSharing.baseline:writer   thrpt   25   847.073 ±  14.666  ops/us
FalseSharing.far               thrpt   25   536.287 ±  66.998  ops/us
FalseSharing.far:modifyFarA    thrpt   25   372.084 ±  45.881  ops/us
FalseSharing.far:modifyFarB    thrpt   25   164.203 ±  52.845  ops/us
FalseSharing.near              thrpt   25   407.426 ±  48.644  ops/us
FalseSharing.near:modifyNearA  thrpt   25   230.663 ±  43.739  ops/us
FalseSharing.near:modifyNearB  thrpt   25   176.763 ±  41.006  ops/us
```
