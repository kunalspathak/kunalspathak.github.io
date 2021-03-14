---
layout: post
title: Loop alignment in .NET Runtime
subtitle: Performance and stability improvement
# cover-img: /assets/img/path.jpg
tags: [life]
comments: true
---

## Introduction

When writing a software, developers try their best to maximize the performance they could get out of the code they have baked into the product. Often times, there are various tools available to the developers to find that last opportunity they can sneeze in to their code to make their software run faster. But sometimes they might notice slowness in the product because of a totally unrelated change they have made. Even worse, when measured the performance of a feature in a lab, it might show instable performance results that looks like the following **BubbleSort** graph<sup>1</sup>. What could possibly be introducing such flakiness in the performance?

<img align="center" width="80%" height="80%" src="/assets/img/loop-alignment/instable-bubble.PNG" />

## Code alignment

 To understand such behavior, you might need to understand how "code alignment" plays a role in software's performance. A target machine code is executed by the CPU by first fetching 16-bytes (16B), 32-bytes (32B) or 64-bytes (64B) instructions (depending on the processors and their generations) in each clock cycle using the instruction fetcher, feeding it to the instruction decoder, converting them to micro-operations (`μops`) and finally executing those `μops`. Code alignment is a technique in which one or more `NOP` instructions are added by the compiler in the generated machine code so that the hot region of the code is placed at a address that is at 16B or 32B aligned chunk of memory so that maximum fetching of the hot code can happen in fewer clock cycles. In other words, if there is a hot region like a loop in the code, the compiler would add sequence of `NOP` instructions (if needed) to make that hot region start at a memory address which is mod(16) or mod(32). Study shows that by performing such alignments, the code can benefit immensely. Additionally, the performance of such code is stable since it is not affected by the placement of code at misaligned address location. If you want to understand in details and see some examples, I would highly encourage to watch the [Causes of Performance Swings due to Code Placement in IA](https://www.youtube.com/watch?v=IX16gcX4vDQ&ab_channel=LLVM) talk given by Intel's engineer Zia Ansari at 2016 LLVM Developer's Meeting.

Since last year, we [align methods at 32B boundary](https://github.com/dotnet/runtime/pull/42909). Recently, we have added a feature in .NET Runtime to [perform adaptive loop alignment](https://github.com/dotnet/runtime/pull/44370) that adds dynamic `NOP` padding instructions in a method having loops such that the loop code  starts at 16B or 32B memory address. In this blog, I will describe the design choices we made, various heuristics that we accounted for and the analysis and implication we studied on 100+ benchmarks that led us to believe that this algorithm will be beneficial in stabilizing and improving the performance of .NET code.

## Heuristics

When we started working on this feature, we wanted to accomplish the following things:
- Identify hot inner most loop(s) that executes very frequently.
- Add `NOP` instructions before the loop code such that the first instruction within the loop falls on 32B boundary.

Below is an example of a loop `IG04~IG05` that is aligned by adding 6-bytes of `align` instruction. In this blog post, in the disassembly, although I will represent padding as `align [X bytes]`, we actually emit [multi-byte NOP](https://www.felixcloutier.com/x86/nop) for actual padding.

```armasm
...
00007ff9a59ecff6        test     edx, edx
00007ff9a59ecff8        jle      SHORT G_M22313_IG06
00007ff9a59ecffa        align    [6 bytes]
; ............................... 32B boundary ...............................
G_M22313_IG04:
00007ff9a59ed000        movsxd   r8, eax
00007ff9a59ed003        mov      r8d, dword ptr [rcx+4*r8+16]
00007ff9a59ed008        cmp      r8d, esi
00007ff9a59ed00b        jge      SHORT G_M22313_IG14

G_M22313_IG05:
00007ff9a59ed00d        inc      eax
00007ff9a59ed00f        cmp      edx, eax
00007ff9a59ed011        jg       SHORT G_M22313_IG04
```

#### Alignment boundary

Different processors are designed such that the software running on them will benefit more if loop is aligned at `16B` boundary while other processors will show benefits by having `32B` alignment boundary. While the alignment should be in multiples of `16` and most recommended boundary for major hardware manufacturers like Intel, AMD and Arm is `32 byte`, we went ahead and had that as our default. With adaptive alignment (controlled using `COMPlus_JitAlignLoopAdaptive` environment variable and is set to be `1` by default), we will try to align a loop at `32 byte` boundary. But if we do not see that it is profitable to align a loop on `32 byte` boundary (for reasons listed below), we will try to align a loop at `16 byte` boundary. With non-adaptive alignment (`COMPlus_JitAlignLoopAdaptive=0`), we will try to align a loop to `32 byte` alignment by default. The alignment boundary can be changed using `COMPlus_JitAlignLoopBoundary` environment variable.   

#### Loop selection

There is a cost associated with a padding instruction. Although `NOP` instruction is cheap, it takes few cycles to fetch and decode it. So having too many `NOP` or `NOP` instructions in hot code path can adversely affect the performance of the code. Hence, it will not be appropriate to align every possible loop in a method. That is the reason, LLVM has `-align-all-*` or gcc has `-falign-loops` flags to give the control to developers, to let them decide which loops should be aligned. Hence, the foremost thing that we wanted to do is to identify the loops in the method that will be most beneficial with the alignment. To start with, we decided to align just the non-nested loops whose block-weight meets a certain weight threshold (controlled by `COMPlus_JitAlignLoopMinBlockWeight`). In below example, `j-loop` and `k-loop` are marked as loop alignment candidates, provided they get executed more often to satisfy the block weight criteria. This is done in [optIdentifyLoopsForAlignment](https://github.com/dotnet/runtime/blob/d39ad7a8df5a7dc22e042a392919d181df34392d/src/coreclr/jit/optimizer.cpp#L2587) method of JIT.

If a loop has a call, the instructions of caller method will be flushed and those of callee will be loaded. In such case, there is no benefit in aligning the loop present in caller. Hence, we decided to not align loops that contains a method call. Below `l-loop`, although is non-nested, it has a call and hence we won't align it. This happens in [AddContainsCallAllContainingLoops](https://github.com/dotnet/runtime/blob/cb606ad46457dd2e9e11f1d9b954f239b2d9a98a/src/coreclr/jit/optimizer.cpp#L8153-L8159).

```c#
void SomeMethod(int N, int M) {
    for (int i = 0; i < N; i++) {

        // j-loop is alignment candidate
        for (int j = 0; j < M; j++) {
            // body
        }
    }

    if (condition) {
        return;
    }

    // k-loop is alignment candidate
    for (int k = 0; k < M + N; k++) {
        // body
    }

    for (int l = 0; l < M; l++) {
        // body
        OtherMethod();
    }
}
```

Once loops are identified in early phase, we proceed forward with advanced checks to see if padding is really beneficial and if yes, what should be the padding amount. All those calculations happen in [emitCalculatePaddingForLoopAlignment](https://github.com/dotnet/runtime/blob/d39ad7a8df5a7dc22e042a392919d181df34392d/src/coreclr/jit/emit.cpp#L4919).

#### Loop size
 
Aligning a loop is beneficial if the loop is small. As the loop size grows, the effect of padding disappears because there is already lot of instruction fetching, decoding and control flow happening that it doesn't matter the address at which the first instruction of a loop is present. We have defaulted the loop size to `96 bytes` which is 3 X 32B chunks. In other words, any inner loop that is small enough to fit in 3 chunks of `32B` each, will be considered for alignment. For experimentation, that limit can be changed using `COMPlus_JitAlignLoopMaxCodeSize` environment variable.

#### Aligned loop

Next, we check if the loop is already aligned at the desired alignment boundary (`32 byte` or `16 byte` for adaptive alignment and `32 byte` for non-adaptive alignment). In such cases, no extra padding is needed. Below, the loop at `IG10` starts at address `0x00007ff9a91f5980` which is already mod(32) address and so no extra padding is added to align this loop.

```armasm
00007ff9a91f597a        cmp      dword ptr [rbp+8], r8d
00007ff9a91f597e        jl       SHORT G_M24050_IG12
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ (jl: 0) 32B boundary ...............................
00007ff9a91f5980        align    [0 bytes]

G_M24050_IG10:
00007ff9a91f5980        movsxd   rdx, ecx
00007ff9a91f5983        mov      r9, qword ptr [rbp+8*rdx+16]
00007ff9a91f5988        mov      qword ptr [rsi+8*rdx+16], r9
00007ff9a91f598d        inc      ecx
00007ff9a91f598f        cmp      r8d, ecx
00007ff9a91f5992        jg       SHORT G_M24050_IG10
```

We also added a "nearly aligned loop" guard. There can be loops that don't start exactly at `32B` boundary, but they are small enough to still fit in `32B` chunk such that the entire loop body can be fetched in a single request by the instruction decoder. There is no point aligning such loops. In below example, the instructions between the borders fit in single `32B` chunk. The loop `IG04` is part of that chunk and its performance won't improve if we add extra padding to it to make the loop start at `32B` boundary. No padding is added in such loops.

```armasm
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ (mov: 3) 32B boundary ...............................
00007ff9a921a903        call     CORINFO_HELP_NEWARR_1_VC
00007ff9a921a908        xor      ecx, ecx
00007ff9a921a90a        mov      edx, dword ptr [rax+8]
00007ff9a921a90d        test     edx, edx
00007ff9a921a90f        jle      SHORT G_M24257_IG05
00007ff9a921a911        align    [0 bytes]

G_M24257_IG04:
00007ff9a921a911        movsxd   r8, ecx
00007ff9a921a914        mov      qword ptr [rax+8*r8+16], rsi
00007ff9a921a919        inc      ecx
00007ff9a921a91b        cmp      edx, ecx
00007ff9a921a91d        jg       SHORT G_M24257_IG04

G_M24257_IG05:
00007ff9a921a91f        add      rsp, 40
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ (add: 3) 32B boundary ...............................
```

This was an important guard that we added in our loop alignment logic. Without this, imagine a loop of size `20 bytes` that starts at offset `mod(32) + 1`. To align this loop, it needed padding of `31 bytes` which might not be beneficial in certain scenarios where `31 byte` `NOP` instructions are on hot code path. The "nearly aligned loop" protects us from such scenarios.

The "nearly aligned loop" check is not restrictive to just small loop that fits in single `32B` chunk. For any loop, we calculate the minimum blocks it need to fit its code in. Now, if the loop is already aligned such that it occupies those many chunks, then we can safely ignore padding the loop further.

In below example, the loop `IG04` is `37 bytes` long (`00007ff9a921c690 - 00007ff9a921c66b = 37`). It needs minimum 2 blocks of `32B` chunk to fit. If the loop starts anywhere between `mod(32)` and `mod(32) + (64 - 37)`, we can safely skip the padding because the loop is already placed such that its body will be fetched in 2 request (`32 bytes` in 1st request and `5 bytes` in next request). Adding extra padding won't make it any better.

```armasm
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ (xor: 2) 32B boundary ...............................
00007ff9a921c662        mov      r12d, dword ptr [r14+8]
00007ff9a921c666        test     r12d, r12d
00007ff9a921c669        jle      SHORT G_M11250_IG07
00007ff9a921c66b        align    [0 bytes]

G_M11250_IG04:
00007ff9a921c66b        cmp      r15d, ebx
00007ff9a921c66e        jae      G_M11250_IG19
00007ff9a921c674        movsxd   rax, r15d
00007ff9a921c677        shl      rax, 5
00007ff9a921c67b        vmovupd  ymm0, ymmword ptr[rsi+rax+16]
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ (movupd: 1) 32B boundary ...............................
00007ff9a921c681        vmovupd  ymmword ptr[r14+rax+16], ymm0
00007ff9a921c688        inc      r15d
00007ff9a921c68b        cmp      r12d, r15d
00007ff9a921c68e        jg       SHORT G_M11250_IG04

G_M11250_IG05:
00007ff9a921c690        jmp      SHORT G_M11250_IG07
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ (xor: 1) 32B boundary ...............................
```

To recap, so far, we have identified the hot nested loops in a method that needs padding, filtered the ones that has calls, filtered the ones that are big than our threshold and verified if the loop are placed such that padding will improve its performance.

#### Padding amount

To align a loop, NOP instructions (in other words, padding) need to be inserted before the loop starts so that the first instruction of the loop starts at an address which is mod(32) or mod(16). It can be a design choice on how much padding we need to add to align a loop. E.g., for aligning a loop to 32B boundary, we can choose to add maximum padding of 31 bytes or can have a limitation on the padding amount. Since padding or NOP instructions are not free, they will get executed and so careful choice of how much padding should be added is very crucial. 

<< diagram and comparison of 32 non-adaptive??>>

#### Padding placement

If it is decided that padding is needed and we calculate the padding amount, the important design choice to make is where to place the padding instructions. A naïve way is to place it just before the loop starts, but as described above, that can adversely affect the performance because the padding instructions can get executed. A smarter way is to detect some blind spots in the code before the loop and place it at such places so that they do not get executed or are executed rarely. E.g. If we have unconditional jump somewhere in the method code, we could add padding instruction after the unconditional jump. By doing this, we will make sure that the padding instruction is never executed but we still get the loop aligned at right boundary. Another place where such padding can be added is in code block or a block that executes rarely (based on PGO data). Of course, the blind spot that we select should be lexically before the loop that we are trying to align.

#### Memory cost

Lastly, there is a need to evaluate how much extra memory is allocated by performing code alignment. If compiler can evaluate above heuristics before allocating memory for the generated code, then the extra memory allocated will be equal to the padding bytes we add. If the compilers align all the loops instead of aligning just hot loops, the memory cost can go up. On the other hand, if there is no limit to the padding amount and if compiler decide to pad at 32B boundary, then that too will consume lot of memory because the padding could be between 0 ~ 31 bytes. Hence, a right balance of loop size and padding limit is very important to have. One more problem that can arise is if a compiler cannot evaluate accurately above heuristics until we emit the instruction. In that case, a compiler will have to pessimistically insert an align instruction considering the largest possible padding. When it finds out during emitting that the alignment was not needed at all or the padding amount is just couple of bytes, it would have wasted the extra bytes it allocated for that alignment instruction. 

## Impact

Code size / Allocation size

## Edge cases

Loop unroll, loop clone, compacting blocks, bogus loop, etc.

## Benchmarks 

## Future work

- relative weight of outer loop vs. inner loop

```c#
for (int i = 0; i < 1000; i++) {
    for (int j = 0; j < 2; j++) {
        // body
    }
}
```


## Loop alignment in other compilers

For native or ahead of time compilers, it is hard to predict which loop will need alignment because the target address where loop will be placed can only be known during runtime and not during ahead of time compilation. However, certain native runtimes at least give an option to the user to let them specify the alignment.

### GCC

GCC provides `-falign-functions` attribute that the user can add on top of a function. More documentation can be seen [here](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#Common-Function-Attributes) under "aligned" section. This will align the first instruction of every function at the specified boundary. It also provides [options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html3) for `-falign-loops`, `-falign-labels` and `-falign-jumps` that will align all loops, labels or jumps in the entire code getting compiled. I did not inspect the GCC code, but looking at these options, it has several limitations. First, the padding amount is fixed and can anywhere between 0 and (N – 1) bytes. Second, the alignment will happen for the entire code base and cannot be restricted to a portion of files, methods, loops or hot regions. 

### LLVM

Same as GCC, dynamic alignment during runtime is not possible so LLVM too exposes an option of alignment choice to the user. [This blog](https://easyperf.net/blog/2018/01/25/Code_alignment_options_in_llvm) gives a good overview of various options available. One of the option that it gives is `align-all-nofallthru-blocks` which will not add padding instructions if the previous block can reach the current block by falling through because that would mean that we are adding NOPs in the execution path. Instead it tries to add the padding at blocks that ends with unconditional jumps. This is similar to what I mentioned above under "Padding placement". To look for alignment handling in LLVM code base, good starting point is [Alignment.h](https://github.com/llvm/llvm-project/blob/9ddb464d37b05b993734c62511576a947b4542df/llvm/include/llvm/Support/Alignment.h#L9).

### Javascript (Chakra)
Chakra marks the inner most loop as the candidate for loop alignment. This happens for both [i386](https://github.com/v8/v8/blob/4b9b23521e6fd42373ebbcb20ebe03bf445494f9/src/compiler/backend/instruction.cc#L792-L800) and [amd64](https://github.com/microsoft/ChakraCore/blob/8527dc23954bdf77b17d25e472857f58a3b03f8a/lib/Backend/amd64/EncoderMD.cpp#L589) architecture. Later, during emitting, when it sees such loops, it aligns them with 16 byte padding as seen for [i386](https://github.com/microsoft/ChakraCore/blob/8527dc23954bdf77b17d25e472857f58a3b03f8a/lib/Backend/i386/EncoderMD.cpp#L1329) and [amd64](https://github.com/microsoft/ChakraCore/blob/8527dc23954bdf77b17d25e472857f58a3b03f8a/lib/Backend/amd64/EncoderMD.cpp#L1498). Thus, the max padding it will add is 15 bytes and it will do this for all inner loops irrespective of the loop size.

## References

1. BubbleSort2 benchmark is part of .NET's BDN framework and the source code is [here](https://github.com/dotnet/performance/blob/master/src/benchmarks/micro/runtime/Benchstones/BenchI/BubbleSort2.cs). Results taken in .NET perf lab can be seen [here](https://pvscmdupload.blob.core.windows.net/reports/allTestHistory%2frefs%2fheads%2fmaster_x64_Windows%2010.0.18362%2fBenchstone.BenchI.BubbleSort2.Test.html).

Talk about the problem and link to the video of alignment issue
- inner loop detection
- adding pads depending on loop size
- figuring out how to add pad of > 15 bytes
- concept of minBlocks to see if alignment is needed or not.
- adjust the maxPads depending on the loop size


- other compilers
