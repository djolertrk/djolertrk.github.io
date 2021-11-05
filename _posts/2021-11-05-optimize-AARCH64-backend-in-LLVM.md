##  Optimizing the AARCH64 backend in LLVM

For those who are not very familiar with ARM processors, it is a CPU architecture based on the RISC (Reduced instruction Set Computing) family, which means that it supports a reduced/simplified set of instructions. Most PCs are based on an Intel (x86) variant of processors, which are designed according to CISC (Complex Instruction Set Computing) architecture rules, where we have various complex instructions, where the most interesting are the ones that can access memory directly. It is not possible with an ARM-based processor, since the only instructions that operate with memory are load/store instructions, so whatever we do with memory, it needs to be loaded into a register, observed and returned to memory. ARM processors are getting even more popular these days, not only in the IoT and mobile devices but also in the PC industry. For example, the Apple M1 (an ARM-based SoC) shows very promising benchmark data. The data sometimes cannot be a guarantee that the end-users will see a lot of benefits out of it, but I am hearing a lot of satisfied users/developers. For example, the LLVM debug build is away easier/faster accomplished on the M1 chip than on X86-based platforms.
In the following content, we’ll be focusing on the AARCH64 target, which is a 64-bit variant of ARM CPU Target, which has very good support in LLVM. Also, it is assumed that the readers are familiar with the LLVM project since this article is rather an LLVM specific proposal/description of some optimizations. The text above was very general, but I am sorry if you are not a compiler developer, since this article is not meant to be a tutorial about LLVM details (even though I’ve put some basics about ARM itself). Therefore, to emphasize, I’ll present specific optimizations and patches to LLVM. Also, there will be some benchmark data gotten in a “static” way by using the llvm-mca [0] tool, it measures the performance of the code statically, which is very useful when you want some quick numbers when dealing with non-native targets. 


<center><img width="350" alt="LLVM-logo1" src="https://user-images.githubusercontent.com/16275603/140522586-fb501104-7c1a-4c01-88ba-507582cd94f6.png"></center>

### Optimizations

Here are some problems (potentially) fixed.

####	Optimize complex GEPs

There is an LLVM Pass initially implemented for the purpose of improving AARCH64 target – SeparateConstOffsetFromGEP. Its mission is to optimize complex GEP instructions. Passes such as GVN, EarlyCSE (and similar) cannot optimize similar GEPs that have common subparts, so the SeparateConstOffsetFromGEP Pass was implemented to address that issue. In general, let us consider an example from documentation.

Let’s consider an example in C:

      // Global variable.
      float a[32][32];
      …
      fn() {
        ...
        for (int i = 0; i < 2; ++i) {
          for (int j = 0; j < 2; ++j) {
            ...
            ... = a[x + i][y + j];
            ...
          }
        }
        ...
      }

The Loop Unrolling will probably create a sequence of IR as:

      ...
      gep %a, 0, %x, %y;
      load
      gep %a, 0, %x, %y + 1;
      load
      gep %a, 0, %x + 1, %y;
      load
      gep %a, 0, %x + 1, %y + 1;
      load
      ...

As we can see the `gep %a, 0, %x, %y` is similar part here that could be optimized, and if we miss to optimize this, architectures with limited addressing modes will suffer. The idea of the SeparateConstOffsetFromGEP Pass is to merge these common parts of the GEPs, so we can calculate each pointer address by adding an offset onto the common base. For the example above, we transform those instructions into:

      base = gep a, 0, x, y
      load base
      laod base + 1 * sizeof(float)
      load base + 32 * sizeof(float)
      load base + 33 * sizeof(float)

Suddenly, the optimization was disabled by default for the AARCH64 (with the cd2334 commit), since “it was pessimising some code patterns”. I am not sure what were the exact cases, but the targets other than AARCH64 that use this LLVM Pass, run Straight Line Strength Reduce, GVN and Nary Reassociate Passes, before invoking the EarlyCSE, to gain full benefits out of this. Please also find the proposals for returning this into -O3 pipeline: PATCH1 PATCH2

This improves/fixes the problem reported at [5].

####	LICM: Hoist LOAD without STORE

The LICM LLVM Pass performs loop invariant code motion. It means that it tries to remove as much as possible code from the loop’s body. It is done by either hoisting some invariants into loop’s preheader (it is a basic block where there is only one entering block, and its only edge is to the loop’s header; more about LLVM’s loop terminology [2]), or by sinking such code into the exit block. All this is done IFF it is safe. What does it mean? It mostly means that it is not safe to promote a load/store from the loop if the load/store is conditional.  For example:

      for () { if (c) *P += 1; }

representing this as:

      tmp = *P;  for () { if (c) tmp +=1; } *P = tmp;

is not safe, because `*P` may only be valid to access if `c` is true.

There are some safety properties that should be satisfied, but it looks like if LICM cannot sink store it does not hoist load as well, even though the memory is dereferenceable on entry to the loop. Please take a look into [3], but it looks like hoisting the load (with proper PHIs) without stores can be a good optimization (the PATCH3 is proposal for that).

####	Recognize table-based ctz

GCC (I’ve tried with the GCC 10 for the aarch64) recognizes && optimizes the table-based ctz for this case:

     int f(unsigned x)
     {
         static const char table[32] = {0, 1, 28, 2, 29, 14, 24, 3, 30, 22, 20, 15, 25, 17, 4, 8, 31, 27, 13, 23, 21, 19, 16, 7, 26, 12, 18, 6, 11, 5, 10, 9};
         return table[((unsigned)((x & -x) * 0x077CB531U)) >> 27];
     }

should be represented as `__builtin_ctz(x);`

Please take a look into the bug report at [4]. The patch PATCH5 implementes/proposes a new LLVM Pass that implements this idea.

### Conclusion && Benchmark

By applying these patches, we have seen good improvements, so binaries for AARCH64 should be faster for a bit. I don't have access to any AARCH64 board to run the SPEC. I could have used the `QEMU` or the `llvm-mca` on a bigger project, but I`ll leave it for the future work.



[0] https://llvm.org/docs/CommandGuide/llvm-mca.html


[1] https://llvm.org/devmtg/2014-10/Slides/Molloy-LLVM-Performant-As-GCC-llvm-dev-2014.pdf


[2] https://llvm.org/docs/LoopTerminology.html


[3] https://bugs.llvm.org/show_bug.cgi?id=51193


[4] https://bugs.llvm.org/show_bug.cgi?id=46434


[5]	https://bugs.llvm.org/show_bug.cgi?id=51184