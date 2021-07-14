##  Test original debug info preservation in LLVM optimizations

Compilers are doing an awesome job these days in terms of minimizing both running time and the final size of executable programs. LLVM/Clang [0] represents cutting-edge technology for this area. A lot of companies from the industry use that compiler (or a compiler based on the LLVM for other/non-C-like programming languages) as a primary compiler for their products.

### Debug Info and optimizations

To ship products with high performance, release versions of the program are usually built with -O2 or -O3 levels of optimizations. To avoid the machine level of debugging, it is practical to add the -g option, to create Debug Info sections, so we can use a debugger (GDB or LLDB for example) for source-level debugging. Usually, developers make a "debug" version of the project (which means turn off the optimizations and include -g). It is not practical in every situation, since sometimes it is not possible (e.g., due to lack of space on the disk), but more important thing is that the generated code we are debugging, in that case, isn't the same (as the release one) anymore.

In the case we have chosen to build our release version of the project with Debug Info, we need to pay some price. Throughout the whole LLVM pipeline, the Debug Info should stay valid, but some optimizations cannot keep the promise, so they just invalidate some kind of Debug Info (for example, the Pass cannot be sure how to connect some LLVM IR instruction with the source code line (&& column), and it just invalidates that Debug Info; as a consequence, we won't be able to put breakpoint for that program point (line, column)). For some optimizations DWARF [1] has been extended to keep things valid, e.g., due to inlining, we have some special Debug Info to record that behavior, so debuggers can pretend as if the code was not inlined at all when printing backtrace for example. The most famous && annoying "problem" when debugging optimized code is when we try to print a variable value, but our debugger prints the special "optimized_out" value. What does it mean? It means that for that specific program point compiler hasn't generated location information for that variable. OK, some of these variables are really optimized away, but some of them are not -- (at least in C-like languages) we should have value for the places where the variable is alive (from its definition up to its last use). It is a very challenging problem. A lot of knowledge about compiler technologies (from end to end in the pipeline) is needed for improving that area. There is no “magic bullet” that will solve all the problems regarding missing locations. There are some features improving it with a bigger impact (for example the debug-entry-values feature, or support multi SSA values in llvm.dbg.value() [2], etc), but in a lot of cases, there is a small/isolated problem in a Pass that doesn’t increase (when fixed) overall location coverage. So, it is rather a long journey.

### Debugging of Debug Info

There is a small cabal of compiler developers (including me) that work on fixing && improving LLVM in terms of Debug Info (especially for optimized code). There is documentation (at [3]) of techniques that could help you if you are writing a new LLVM Pass with how-to-properly-care-abut-debug-info. Please consider using this link, but I will try to demonstrate a use case for the [4], and how this can be useful. Basically, it has been implemented as an extension of existing debugify utility Pass(es) that aim to check the preservation of Debug Info in optimizations. The old school debugify attaches artificial debug info to everything before an LLVM Pass, and after the Pass is finished, it checks whether it preserved the Debug Info (and it is very useful inside the testing framework). The [4] aims to deal with original, -g generated Debug Info, so the options from that feature can be used from the front end level as well (for example: *$ clang -Xclang -fverify-debuginfo-preserve -g -O2 sample.c*). On a very high level, the idea is to detect critical spots in the LLVM pipeline that cause the debugger to print your variable (from your project) as optimized-out, but it shouldn’t have been optimized away. It detects other problems as well, for example, where we have dropped (line, column) connection with LLVM instructions, etc.

### Find Debug Info issues by inspecting GNU GDB project

I might be a bit selfish, but whenever I am testing a change in the LLVM compiler, I am using the GNU GDB project as a testbed (but sometimes I use the LLVM project itself as well). I am usually fixing Debug info-related issues on the compiler side, but sometimes I end up working on the debugger side as well, therefore if any of the compiler improvements have a positive impact on the debugging experience on the GDB project, I am happy with that. Yes - I am using GDB debugger to debug the GDB debugger. You can activate these compiler options from your project build (e.g. from Chromium) and find out what LLVM Pass is to blame for some concrete optimized_out variables in your project.

For now, the [4] can find issues with dropping the line, column mapping (within LLVM, it is called Location) between source code and corresponding LLVM instructions, and issues with dropping variables values/locations (please do note that there is a term Variable Location as well). There is an idea for finding issues for more kinds of the Debug Info. Also, we are seeing some false positives at the moment, but there are some ideas on how to improve it. Anyhow, please find the steps below on how to use it.

First, I am using the GNU GDB 7.11 for this experiment, from [GNU GDB](https://www.gnu.org/software/gdb/).
Also, I will set up a path to a JSON file that will be used by the compiler for writing down the issues that have been found. The JSON file will be translated (by the script from llvm-project/llvm/utils/) into the HTML page, with tables that contain info about the file, LLVM Pass, etc., where the Debug Info issue has been found.

    $ mkdir build && cd build
    $ ../gdb-source/configure CC=$PATH_TO_LLVM_BUILD/bin/clang CXX=$PATH_TO_LLVM_BUILD/bin/clang++ CFLAGS="-g -O2 -Wno-error -Xclang -fverify-debuginfo-preserve -Xclang -fverify-debuginfo-preserve-export=~/gdb-report-bugs.json" CXXFLAGS="-g -O2 -Wno-error -Xclang -fverify-debuginfo-preserve -Xclang -fverify-debuginfo-preserve-export=~/gdb-report-bugs.json" --enable-werror=no
    $ make -j1

After the build process is done, the ~/gdb-report-bugs.json file will contain all the information collected by the compiler. As a next step we want to generate the HTML out of these JSON objects:

    $ llvm-original-di-preservation.py ~/gdb-report-bugs.json before-the-fix.html

The HTML can be found at [Debug Info Issues Before The Compiler Fix](https://djolertrk.github.io/di-check-before-adce-fix/).

After the analysis of some “Variable Location” issues, I’ve found out that the Aggressive Dead Code Elimination Pass (ADCE) drops some variable location information (more precisely, the variable “vendor” from the bfd/elf-attrs.c). After some investigation within the compiler, I’ve come up with the patch [5] that fixes that problem. The new HTML page, generated from the JSON file produced from the fixed version of the compiler, shows a reduced number of Debug Info related issues [Debug Info Issues After The Compiler Fix](https://djolertrk.github.io/di-check-after-adce-fix/).

If you find this useful, please run these options on your build, and please do share your impressions.

[0] [LLVM](https://llvm.org/)

[1] [DWARF](http://dwarfstd.org/)

[2] [llvm-dbg-value](https://llvm.org/docs/SourceLevelDebugging.html#llvm-dbg-value)

[3] [HowToUpdateDebugInfo](https://llvm.org/docs/HowToUpdateDebugInfo.html)

[4] [test-original-debug-info-preservation-in-optimizations](https://llvm.org/docs/HowToUpdateDebugInfo.html#test-original-debug-info-preservation-in-optimizations)

[5] [LLVM Fix](https://reviews.llvm.org/D100844)

