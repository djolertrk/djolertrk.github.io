##  Debug Info in LLVM keeps improving!

Everyone wants an ideal debugging user experience, that includes seeing no `optimized_out` variables during the debugging of an optimized binary. It is not an easy task for compiler developers to achive 100% coverage for all the variables, but LLVM community is doing a great job! I've been following the improvements with the respect to the location coverage over the releases [0] (not only following, I also worked on improving the Debug Info :) ) and the recent numbers look amazing. In the link I mentioned [0], we can see that LLVM 6.0 generated the binary with only 45% of PC coverage, but now LLVM 16.0.4 generates binary with 71% of PC coverage.

First of all, let me mention that I used the `binutils-gdb` as testbed, compiled with `-g -O2`. By using the [1], the results look as:

### LLVM 15.0.7

```bash
    $ llvm-locstats gdb-built-with-clang-15-0-7 
     =================================================
            Debug Location Statistics       
     =================================================
     cov%           samples         percentage(~)  
     -------------------------------------------------
       0%                 9352                1%
       (0%,10%)          10743                1%
       [10%,20%)         17696                3%
       [20%,30%)         11934                2%
       [30%,40%)          9973                1%
       [40%,50%)          8946                1%
       [50%,60%)         10828                1%
       [60%,70%)         10696                1%
       [70%,80%)         13399                2%
       [80%,90%)         15424                2%
       [90%,100%)        14290                2%
       100%             412840               75%
     =================================================
     -the number of debug variables processed: 546121
     -PC ranges covered: 70%
     -------------------------------------------------
     -total availability: 62%
     =================================================
```

### LLVM 16.0.4

```bash
    $ llvm-locstats gdb-built-with-clang-16-0-4 
      =================================================
            Debug Location Statistics       
      =================================================
      cov%           samples         percentage(~)  
      -------------------------------------------------
        0%                 8816                1%
        (0%,10%)          10153                1%
        [10%,20%)         13014                2%
        [20%,30%)         15861                2%
        [30%,40%)         10036                1%
        [40%,50%)          9203                1%
        [50%,60%)         10651                1%
        [60%,70%)         11061                2%
        [70%,80%)         13719                2%
        [80%,90%)         14158                2%
        [90%,100%)        13783                2%
        100%             416758               76%
      =================================================
      -the number of debug variables processed: 547213
      -PC ranges covered: 71%
      -------------------------------------------------
      -total availability: 65%
      =================================================
```

### Compare llvm 16.0.4 vs llvm 15.0.7

 ![locstats](https://github.com/djolertrk/djolertrk.github.io/assets/16275603/ecca7105-b237-4896-a670-2c415dfd0979)

[0] [Debug Info Stats Report](https://djolertrk.github.io/llvm-debug-loc-stats/)

[1] [llvm-locstats](https://llvm.org/docs/CommandGuide/llvm-locstats.html)
