---
layout: post
title: Loop alignment in .NET Runtime
subtitle: Performance and stability improvement
# cover-img: /assets/img/path.jpg
tags: [life]
comments: true
---

## Introduction

When writing a software, developers try hard to maximize the performance they could get out of the code they have baked into the product. Often times, there are various tools available to the developers to find that last thing they can squeeze out from their code to make their software run faster. But sometimes they might notice slowness in the product because of a totally unrelated change they have made. Even worse, when measured the performance of a feature in a lab, it might show instable performance results that looks like the following **BubbleSort** graph<sup>1</sup>. What could possibly be introducing such flakiness in the performance?

<img align="center" width="80%" height="80%" src="/assets/img/loop-alignment/instable-bubble.PNG" />

## Code alignment

 To understand such behavior, you might need to understand how "code alignment" plays a role in performance. A target machine code is executed by the CPU by first fetching 16-bytes (16B), 32-bytes (32B) or 64-bytes (64B) instructions (depending on the processors and their generations) in each clock cycle using the instruction fetcher, feeding it to the instruction decoder, converting them to micro-operations (`μops`) and finally executing those `μops`. Code alignment is a technique in which one or more `NOP`s are added by the compiler in the generated machine code so that the hot region of the code is placed at a address that is at 16B or 32B aligned chunk of memory so that maximum fetching of the hot code can happen in fewer clock cycles. In other words, if there is a hot region like a loop in the code, the compiler would add sequence of `NOP`s (if needed) to make that hot region start at an address which is mod(16) or mod(32). Study shows that by performing such alignments, the code can benefit immensely. Additionally, the performance of such code is stable since it is not affected by the placement of code at misaligned address location. If you want to understand in details and see some examples, I would highly encourage to watch the [Causes of Performance Swings due to Code Placement in IA](https://www.youtube.com/watch?v=IX16gcX4vDQ&ab_channel=LLVM) talk given by Intel's engineer Zia Ansari at 2016 LLVM Developer's Meeting.

Since last year, we [align methods at 32B boundary](https://github.com/dotnet/runtime/pull/42909). Recently, we have added a feature in .NET Runtime to [perform adaptive loop alignment](https://github.com/dotnet/runtime/pull/44370) that adds dynamic `NOP` padding instructions in a method so that the loop starts at 16B or 32B memory address. In this blog, I will describe the design choices we made, various heuristics that we accounted for and the analysis and implication we studied on 100+ benchmarks that led us believe that this algorithm will be beneficial in stabilizing and improving the performance of .NET code.

## Heuristics

When we started working on this feature, we wanted to accomplish following things:
- Identify inner most loop(s) that is hot (the loop code executes very frequently).
- Add `NOP` instructions before the loop code such that the first instruction within the loop falls on 32B boundary.

Below is an example of a loop `a` that is aligned by adding TODO-byte `NOP` instructions.

```asm
```

#### Loop selection

The foremost criteria for doing a loop alignment is to shortlist the loops in the method that will benefit the most with the alignment. Since there is a cost associated with alignment (more in `Padding amount` section below), we wanted to ensure that we only pad loops that are hottest and perform lot of iterations. In our first phase, we decided to align just the non-nested loops whose block weight meets a certain weight threshold criteria. In below example, `j-loop` and `k-loop` are marked as loop alignment candidates, provided they get executed more often to satisfy our block weight criteria.

If a loop has a call, the instructions of caller method will be flushed and those of callee will be loaded. In such case, there is no benefit in aligning the loop present in caller. Hence, we decided to not align loop that contains a method call. Below, `l-loop` although is non-nested, it has call, and hence we won't align it. 

```c#
void SomeMethod(int N, int M) {
    for (int i = 0; i < N; i++) {

        // j-loop is alignmnet candidate
        for (int j = 0; j < M; j++) {
            // body
        }
    }

    if (condition) {
        return;
    }

    // k-loop is alignmnet candidate
    for (int k = 0; k < M + N; k++) {
        // body
    }

    for (int l = 0; l < M; l++) {
        // body
        OtherMethod();
    }
}
```

#### Aligned loop

3. Aligned loops: It can happen that the first instruction of the loop is already at the 32B boundary and in such cases, no extra padding would be needed.

4. Slightly aligned loops: There can be cases where although the first instruction of the loop is not at 32B boundary, but it is at offset such that the entire loop body still fits in the same 32B chunk. E.g., if the loop size is 24 bytes, and if it starts anywhere between mod(32)+0 ~ mod(32)+7 , it will still fit inside a single 32B chunk and in such cases, we can skip aligning the loop. If this heuristics is not taken in account, we can run into cases where a loop starts from mod(32)+1, and since it is not at the mod(32) address, we add extra 31 bytes to align it to next 32B chunk.

#### Padding amount



To align a loop, NOP instructions (in other words, padding) need to be inserted before the loop starts so that the first instruction of the loop starts at an address which is mod(32) or mod(16). It can be a design choice on how much padding we need to add to align a loop. E.g., for aligning a loop to 32B boundary, we can choose to add maximum padding of 31 bytes or can have a limitation on the padding amount. Since padding or NOP instructions are not free, they will get executed and so careful choice of how much padding should be added is very crucial. 

<< diagram and comparison of 32 non-adaptive??>>

#### Loop size
 
Aligning the loop is beneficial if the loop is small. As the loop size grows, the effect of padding disappears. Imagine a loop that is 62 bytes long and need at least 2 32B chunks to fit. It might not be sensible to add 31 bytes padding for such loop since fetching two chunks of loop might hide the gain obtained from padding. Additionally, the CPU will spend cycles on 31 bytes NOP if it falls on the code path and that can adversely affect the performance of the code.


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
